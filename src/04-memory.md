# Linux Deep Dive #4: Memory Management — Virtual Memory and the Page Cache

*Target: Fedora 43, kernel 6.19.11. Every command in this post is something you can run yourself.*

---

Run `free -h` on any Linux system that's been running for a while:

```
$ free -h
              total        used        free      shared  buff/cache   available
Mem:            30Gi        22Gi       200Mi       500Mi       7.8Gi        7.3Gi
Swap:          8.0Gi          0B       8.0Gi
```

Most people see the `free` column — 200 MB — and worry. Only 200 MB of RAM left? But the system is perfectly healthy. `available` is 7.3 GiB. Something is using 7.8 GiB under `buff/cache`, and it can be reclaimed on demand.

This is the kernel doing its job. Idle RAM is wasted RAM. The kernel fills unused memory with file data it might need again, making disk reads fast. When a process needs memory, the kernel reclaims the cache. The `free` column tells you almost nothing useful. `available` is the number that matters.

This chapter explains why — by tracing the full memory architecture from virtual addresses through page tables to physical frames, through the page cache and back. We'll cover how the kernel allocates physical memory, what really happens on a page fault, how the page cache works, and how the kernel reclaims memory under pressure.

---

## Memory at a Glance

```
User process (virtual address space)
    │
    │  Virtual address → MMU → page table walk
    ▼
┌──────────────────────────────────────────────────────┐
│  4-Level Page Tables                                  │
│  PGD → PUD → PMD → PTE → physical frame              │
│  TLB caches recent translations                      │
└──────────────────────────────────────────────────────┘
    │
    │  physical frame number
    ▼
┌──────────────────────────────────────────────────────┐
│  Physical Memory                                      │
│  ┌────────────────────┐  ┌──────────────────────┐   │
│  │  Anonymous pages   │  │  File-backed pages   │   │
│  │  heap, stack, CoW  │  │  (the page cache)    │   │
│  │  must swap to evict│  │  can drop for free   │   │
│  └────────────────────┘  └──────────────────────┘   │
│                                                       │
│  Buddy allocator manages frames (4KB to 4MB blocks)  │
│  SLUB allocator carves pages into kernel objects     │
└──────────────────────────────────────────────────────┘
    │
    │  when free memory drops below watermarks
    ▼
┌──────────────────────────────────────────────────────┐
│  Reclaim                                              │
│  kswapd scans LRU lists                              │
│  ├── file pages: drop (clean) or writeback + drop    │
│  └── anon pages: write to swap, then free frame      │
│                                                       │
│  Last resort: OOM killer                             │
└──────────────────────────────────────────────────────┘
```

---

## Virtual Addresses and Page Tables

Chapter 3 showed that every process has an `mm_struct` containing a set of VMAs — contiguous virtual address regions with permissions like `r-xp` or `rw-p`. But how does a virtual address in a VMA actually become a physical address the CPU can use?

The answer is the page table.

### The 4-Level Page Table on x86-64

On a 64-bit x86 system, virtual addresses are 48 bits wide. The CPU uses a 4-level page table structure to translate them:

```
Virtual address (48-bit):
  ┌───────┬───────┬───────┬───────┬──────────────┐
  │  PGD  │  PUD  │  PMD  │  PTE  │ Page offset  │
  │ 9 bit │ 9 bit │ 9 bit │ 9 bit │   12 bit     │
  └───────┴───────┴───────┴───────┴──────────────┘

PGD = Page Global Directory   (top-level)
PUD = Page Upper Directory
PMD = Page Middle Directory
PTE = Page Table Entry        (leaf: physical frame number + flags)
```

The translation works like a 4-level lookup:

```
CR3 register → PGD table
    │  index with bits 47–39
    ▼
PGD entry → PUD table
    │  index with bits 38–30
    ▼
PUD entry → PMD table
    │  index with bits 29–21
    ▼
PMD entry → PTE table
    │  index with bits 20–12
    ▼
PTE entry: physical frame number + R/W/X/U flags
    │  + page offset (bits 11–0)
    ▼
Physical address
```

Each level is a 4KB page holding 512 8-byte entries (9 bits of index → 2^9 = 512). The final 12 bits of the virtual address index into the 4KB page. This gives 48-bit coverage: 9+9+9+9+12 = 48.

The flags in each PTE are what drive many kernel mechanisms:

| PTE flag | Effect |
|----------|--------|
| Present | Page is in RAM; if clear, triggers a page fault |
| Writable | Write permission; if clear on a mapped page, triggers CoW fault |
| User | Accessible from user space; kernel-only pages have this clear |
| Accessed | CPU sets this when the page is read (used by reclaim) |
| Dirty | CPU sets this on write (used to detect pages needing writeback) |
| NX (No-Execute) | Prevents execution of data pages |

You can see the page table walk in action through `/proc/[pid]/pagemap`, though that requires root. The more accessible view is `/proc/[pid]/maps`, which shows the VMAs:

```bash
$ cat /proc/$$/maps | head -8
5592b7c38000-5592b7c3e000 r--p 00000000 fd:01 530753  /usr/bin/bash
5592b7c3e000-5592b7c7e000 r-xp 00006000 fd:01 530753  /usr/bin/bash
5592b7c7e000-5592b7cad000 r--p 00046000 fd:01 530753  /usr/bin/bash
5592b7cae000-5592b7cb2000 rw-p 00075000 fd:01 530753  /usr/bin/bash
5592b9040000-5592b90ae000 rw-p 00000000 00:00 0       [heap]
7f2345a00000-7f2345c00000 r--p 00000000 fd:01 398912  /usr/lib/locale/locale-archive
...
7ffe8e3b0000-7ffe8e3d2000 rw-p 00000000 00:00 0       [stack]
7ffe8e3f8000-7ffe8e3fc000 r--p 00000000 00:00 0       [vvar]
```

Each row is a VMA. The permissions `r-xp`, `rw-p`, `r--p` map directly to PTE flags. The page table for this process contains entries for each page within each VMA — but only for pages that have actually been touched (page tables are themselves demand-paged).

### The TLB: Caching Page Table Lookups

Four memory accesses to translate one address would be crippling. The CPU keeps a **Translation Lookaside Buffer (TLB)** — a small hardware cache of recent virtual→physical translations. A TLB hit returns the physical address in a single cycle. A miss triggers the full 4-level walk.

TLB misses are part of why context switches have measurable cost: switching processes means switching page tables (loading a new value into `CR3`), which flushes the TLB. The next few hundred memory accesses in the new process are all TLB misses until the cache warms up.

The kernel minimizes TLB pressure through **huge pages**: instead of 4KB pages (needing many TLB entries), the kernel can use 2MB pages (PMD-level mappings), covering 512× more memory with a single TLB entry. On this system, transparent huge pages (THP) are in `madvise` mode — the kernel uses 2MB pages for anonymous memory when the application explicitly requests them:

```bash
$ cat /sys/kernel/mm/transparent_hugepage/enabled
always [madvise] never
```

---

## Physical Memory — Zones, the Buddy Allocator, and Slab

### Memory Zones

Not all physical RAM is equivalent. Legacy ISA devices could only DMA to the first 16 MB. Older PCI devices can only address below 4 GB. The kernel divides physical memory into **zones** to track these constraints:

```bash
$ cat /proc/buddyinfo
Node 0, zone      DMA      0      0      0      0      0      0      0      0      1      1      2
Node 0, zone    DMA32     17     13     13     12      9      9     14     12      9      9    768
Node 0, zone   Normal  65656  39923  16923   8715   2995    987    367    127     53     27   1876
```

| Zone | Physical range | Purpose |
|------|----------------|---------|
| DMA | 0–16 MB | ISA/legacy DMA devices |
| DMA32 | 0–4 GB | 32-bit PCI devices |
| Normal | > 4 GB | All other allocations |

On this 30 GB machine, the `Normal` zone holds almost all RAM.

### The Buddy Allocator

The kernel's fundamental physical memory allocator is the **buddy allocator**. It maintains 11 free lists, indexed by order 0 through 10, where order N holds blocks of 2^N contiguous pages:

```
Order 0:  1 page   = 4KB
Order 1:  2 pages  = 8KB
Order 2:  4 pages  = 16KB
...
Order 10: 1024 pages = 4MB
```

The numbers in `buddyinfo` are the count of free blocks at each order. In the `Normal` zone above, there are 65,656 order-0 blocks (4KB each), 39,923 order-1 blocks (8KB each), and so on.

When the kernel needs N pages:
1. Find the smallest order ≥ N with a free block
2. If that order is larger than needed, **split** the block: one half goes to the next lower order's free list, the other half is used
3. When pages are freed, the kernel checks if the adjacent "buddy" block is also free; if so, they **merge** back to the higher order

Splitting and merging keep the allocator efficient and minimize fragmentation. You can watch allocation pressure by checking how many high-order blocks remain — if order-10 is empty and order-0 is full, the system is fragmented and can't satisfy large contiguous allocations.

```bash
# Watch buddy allocator state live
watch -d cat /proc/buddyinfo
```

### The SLUB Allocator

The buddy allocator deals in full pages. But the kernel constantly allocates objects much smaller than 4KB: `task_struct` (~7KB but padded), `file` (~300 bytes), `dentry` (~200 bytes), `inode` (~600 bytes), socket buffers (~200 bytes each). Handing out a full 4KB page for each would waste enormous amounts of RAM.

The **SLUB allocator** (Simplified Unqueued Layer) sits on top of the buddy allocator. It maintains per-CPU **slabs** — pages filled with pre-allocated objects of a single size. Allocating a small kernel object takes a free slot from a slab; freeing it returns the slot. No buddyallocator call needed.

```bash
# See the largest slab caches by memory usage
$ sudo slabtop -o | head -15
 Active / Total Objects (% used)    : 2344037 / 2429115 (96.5%)
 Active / Total Slabs (% used)      : 82684 / 82684 (100.0%)
 Active / Total Caches (% used)     : 193 / 244 (79.1%)
 Active / Total Size (% used)       : 622408.38K / 655006.31K (95.0%)

  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
253056 253056 100%    0.19K   6016       42     24064K dentry
198400 186988  94%    0.06K   3100       64      6200K kmalloc-64
110916  95897  86%    1.06K   9243       12    147888K inode_cache
 37248  37248 100%    0.50K   1164       32     18624K kmalloc-512
```

`dentry` (directory entry cache) and `inode_cache` are typically the largest slab consumers — the kernel caches filesystem metadata aggressively because lookups are frequent and disk access is slow.

---

## Page Faults

When the CPU walks the page table and something is wrong, it raises a **page fault** — a hardware exception that transfers control to the kernel's fault handler (`handle_mm_fault()` in `mm/memory.c`). There are several distinct kinds:

| Fault type | Cause | Kernel action | Disk I/O? |
|------------|-------|---------------|-----------|
| **Minor** | Page in VMA but PTE not present (first access) | Allocate frame, fill PTE | No |
| **Major** | Page was evicted to disk; PTE marked not-present | Read page from disk, fill PTE | Yes |
| **CoW** | Write to a read-only shared page (after fork) | Copy page, update PTE to writable | No |
| **Invalid** | Access outside any VMA | Deliver SIGSEGV | No |

Minor faults are normal and constant — they happen every time a program accesses a new page of its heap or stack. The kernel allocates a physical frame, zeros it (to prevent information leaks), updates the PTE, and the program continues. No disk I/O, just a brief detour through the kernel.

Major faults are a symptom of memory pressure. They mean the kernel previously evicted this page to disk to free RAM, and now the program needs it back. Major faults cause visible latency — the application stalls waiting for the disk read to complete.

```bash
# Page fault totals since boot
$ grep -E '^pgfault|^pgmajfault' /proc/vmstat
pgfault       39670010   # minor faults — normal, healthy
pgmajfault        3007   # major faults — disk reads to recover evicted pages
```

39 million minor faults since boot is expected. 3007 major faults is low — this system hasn't been under enough memory pressure to evict much. On a memory-constrained system, `pgmajfault` climbs steadily and latency spikes whenever a major fault stalls a thread.

The CoW fault is the mechanism from Chapter 3: after `fork()`, parent and child share physical pages marked read-only. The first write from either side triggers a fault, the kernel copies the page, and both now have their own writable copy.

You can measure per-process fault rates in real time:

```bash
# Fault rates for a running process
$ cat /proc/1/status | grep -i fault
voluntary_ctxt_switches: 31248
nonvoluntary_ctxt_switches: 1156

# For cumulative fault count, smaps_rollup is faster than summing smaps:
$ sudo cat /proc/1/smaps_rollup | grep -E 'Rss|Pss'
Rss:               21432 kB
Pss:               13891 kB
```

---

## The Page Cache

This is the mechanism behind the `buff/cache` column in `free`. It's also the single most important thing to understand about Linux memory management.

### The Basic Idea

When you `read()` from a file, the kernel doesn't transfer bytes directly from disk to your process. It maintains a **page cache** — a pool of physical pages indexed by `(filesystem inode, page offset)`. The read path is:

```
read() syscall
    │
    ├── Is (inode, offset) already in the page cache?
    │   ├── Yes: copy from cache to user buffer → done (no disk I/O)
    │   └── No:  read from disk into a new cache page,
    │             then copy to user buffer
    │
    └── Cache page stays in RAM after the read
```

The page cache entry persists after your `read()` returns. The *next* `read()` of the same region — whether by you, another process, or another program — finds it in cache and pays no disk cost.

This is why a fresh `make` that compiles 10,000 source files goes faster on the second run: all those file reads hit the cache. The first run was I/O-bound; the second is CPU-bound.

### File-Backed vs Anonymous Memory

Physical memory pages fall into two categories:

```
Physical pages
    │
    ├── File-backed (page cache)
    │   ├── Clean: identical to disk → kernel can drop for free
    │   └── Dirty: modified, not yet written back
    │           → kernel must flush to disk before dropping
    │
    └── Anonymous (no file)
        ├── Heap, stack, CoW copies, anonymous mmap
        └── To free: must write to swap device first
                      (or kill the process)
```

File-backed pages are cheap to evict — the kernel just drops the page and re-reads it from disk if needed again. Anonymous pages are expensive: they have no file backing, so eviction requires writing to swap and reading back later.

This distinction drives the entire reclaim strategy.

### mmap() and the Page Cache

`mmap()` is the page cache seen from the process side. When your program opens a file and calls:

```c
void *ptr = mmap(NULL, file_size, PROT_READ, MAP_SHARED, fd, 0);
```

The kernel creates a VMA that represents a window into the file's pages in the page cache. No data is loaded yet. When you dereference `ptr`, a minor page fault fires; the kernel checks the page cache (reading from disk if necessary), and installs a PTE pointing to that cache page. **The process's PTE and the page cache entry point to the same physical frame.**

This has an important consequence: every process running the same program shares its text segment.

```bash
# How many processes share bash's text pages?
$ cat /proc/$(pgrep -n bash)/smaps | grep -A 10 'bash$' | head -12
5592b7c3e000-5592b7c7e000 r-xp 00006000 fd:01 530753  /usr/bin/bash
Size:                256 kB
KernelPageSize:        4 kB
Rss:                 252 kB
Pss:                  14 kB    ← proportional share — 252KB / ~18 processes
Shared_Clean:        252 kB    ← all of it is shared
Private_Clean:         0 kB
```

`Rss` (Resident Set Size) is the total physical memory mapped. `Pss` (Proportional Set Size) divides shared pages by the number of processes sharing them. The `r-xp` text segment shows `Shared_Clean: 252 kB` and `Pss: 14 kB` — 18 bash processes share the same physical pages. One copy in RAM, many processes using it.

### Reading the Page Cache Statistics

```bash
$ grep -E '^(Cached|Buffers|Mapped|Dirty|Writeback)' /proc/meminfo
Buffers:            4208 kB
Cached:          6564188 kB
Mapped:          1189268 kB
Dirty:              1336 kB
Writeback:             0 kB
```

| Field | Meaning |
|-------|---------|
| `Cached` | Total page cache size |
| `Buffers` | Block device metadata cache (separate from file data cache) |
| `Mapped` | Page cache pages currently mapped into at least one process's address space |
| `Dirty` | Modified pages not yet written to disk |
| `Writeback` | Pages currently being written to disk |

On this machine, `Cached` is 6.4 GB — 6.4 GB of file data sitting in RAM, ready for instant re-use. `Dirty` is only 1.3 MB, meaning the kernel's writeback threads are keeping up with writes. `Writeback` is 0, so no I/O is in flight right now.

---

## Reclaim — How the Kernel Manages Memory Pressure

### LRU Lists

The kernel tracks which pages are worth keeping with four LRU (Least Recently Used) lists:

```
LRU lists
    ├── active_file    — recently used file-backed pages
    ├── inactive_file  — older file-backed pages (reclaim candidates)
    ├── active_anon    — recently used anonymous pages
    └── inactive_anon  — older anonymous pages (swap candidates)
```

Pages move between these lists based on access patterns. The hardware **Accessed** bit in PTEs tells the kernel when a page was last used. Pages migrate from active to inactive when they haven't been accessed for a while; from inactive, they become reclaim candidates.

```bash
$ grep -E '^(Active|Inactive)' /proc/meminfo
Active:          7844660 kB
Inactive:        4958656 kB
Active(anon):    4685204 kB
Inactive(anon):        0 kB
Active(file):    3159456 kB
Inactive(file):  4958656 kB
```

On this system: 4.7 GB of anonymous memory is active (process heaps/stacks in use), 3.2 GB of file cache is active (recently read files). The 4.9 GB of inactive file cache is the first thing kswapd will drop if memory gets tight.

### kswapd and Watermarks

**kswapd** is a kernel thread that maintains free memory within a healthy range. The kernel defines three watermarks for each zone:

```
Zone free pages
    │
    ├── High watermark — kswapd stops here (plenty of free memory)
    │
    ├── Low watermark  — kswapd wakes up and starts reclaiming
    │
    └── Min watermark  — direct reclaim kicks in; allocations stall
                         until kswapd frees enough pages
```

When free memory drops below the low watermark, kswapd wakes and scans the inactive lists:

1. **Inactive file pages** (clean): drop immediately — they're exact copies of disk data
2. **Inactive file pages** (dirty): schedule writeback, then drop after write completes
3. **Inactive anon pages**: write to swap device, then free the frame

```bash
# Is kswapd doing any work?
$ grep -E 'pgsteal_kswapd|pgscan_kswapd|kswapd_low_wmark' /proc/vmstat
pgsteal_kswapd             0    # pages reclaimed by kswapd (0 = no pressure)
pgscan_kswapd              0    # pages scanned during reclaim
kswapd_low_wmark_hit_quickly   0
```

All zeros: this system is not under memory pressure. kswapd is idle. On a memory-constrained system, `pgsteal_kswapd` climbs continuously.

### Swappiness

The `vm.swappiness` sysctl (0–200, default 60) controls the balance between evicting file cache and swapping anonymous pages. It's a weight in the reclaim cost calculation — higher values make the kernel more willing to swap anonymous pages alongside evicting file cache.

```bash
$ cat /proc/sys/vm/swappiness
60
```

Contrary to common belief, swappiness is not a memory-use threshold. `swappiness=60` does not mean "start swapping at 60% memory use." It means: when choosing between evicting a file page and swapping an anonymous page, weight the decision so that both are considered in roughly a 60:100 ratio.

Setting `swappiness=1` makes the kernel almost never swap — it will exhaust file cache before touching anonymous memory. Setting it to `200` makes the kernel aggressively swap, preferring to keep file cache hot. Neither extreme is universally right.

```bash
# Check swap usage
$ grep -E '^(SwapTotal|SwapFree|SwapCached|Pswpin|Pswpout)' /proc/meminfo
SwapTotal:       8388604 kB
SwapFree:        8388604 kB
SwapCached:            0 kB

# Swap activity since boot
$ grep -E '^pswp' /proc/vmstat
pswpin    0     # pages swapped in (major faults from swap)
pswpout   0     # pages swapped out
```

Zero swap activity on this machine — there's enough RAM that kswapd hasn't needed to swap anything out.

---

## The OOM Killer

When free memory hits zero, all reclaim options are exhausted, and a memory allocation is still failing, the kernel invokes the **OOM killer** (Out Of Memory killer) as a last resort.

The OOM killer selects a process to terminate based on `oom_score` — a value from 0 to 1000 calculated from:

- RSS (how much RAM the process is actually using)
- Runtime (longer-running processes are slightly protected)
- Nice value (low-priority processes score higher)
- Whether the process is privileged (root processes score slightly lower)

The process with the highest score gets killed with SIGKILL.

```bash
# OOM score for each running process
$ cat /proc/1/oom_score          # systemd — should be 0 (protected)
0

$ cat /proc/$$/oom_score         # your shell
5

# Processes can adjust their score (-1000 to +1000)
$ cat /proc/$$/oom_score_adj
0
```

Setting `oom_score_adj` to `-1000` makes a process immune to OOM killing. Systemd sets itself to -1000 for exactly this reason. Critical system daemons often do the same. Setting it to `+1000` means "kill me first in an OOM."

```bash
# Which process would the OOM killer target right now?
$ ps -eo pid,oom_score,rss,comm --sort=-oom_score | head -10
    PID OOM_SCORE    RSS COMMAND
   2847       182 2345200 firefox
   3012        89  854320 slack
   3201        45  412800 gnome-shell
```

The OOM killer fires rarely on a well-sized system. If it fires often, the right fix is more RAM, fixing a memory leak, or adding limits with memory cgroups — not tuning OOM parameters.

---

## Why `free` Shows Almost No "Free" Memory

Now we have all the pieces. Return to the output from a memory-loaded system:

```
$ free -h
              total        used        free      shared  buff/cache   available
Mem:            30Gi        22Gi       200Mi       500Mi       7.8Gi        7.3Gi
Swap:          8.0Gi          0B       8.0Gi
```

What does each column actually mean?

| Column | What it counts |
|--------|----------------|
| `total` | Total physical RAM |
| `used` | Process memory + kernel memory not reclaimable |
| `free` | Completely idle frames — holding nothing |
| `shared` | tmpfs and shared memory |
| `buff/cache` | Page cache + buffer cache — **reclaimable on demand** |
| `available` | Estimated memory a new allocation could get ≈ free + reclaimable buff/cache |

The `free` column being 200 MB does not mean the system is nearly out of memory. It means the kernel has filled almost all idle frames with page cache — which it should. Those 7.8 GB of `buff/cache` are serving file reads instantly.

When a process needs more memory, the kernel reclaims from `buff/cache` first (by dropping clean file pages). The new allocation succeeds, and `buff/cache` shrinks. `available` — not `free` — is what tells you whether a large new allocation will succeed.

The actual machine for this series has 30 GB of RAM and was not heavily loaded at the time this chapter was written:

```bash
$ free -h
              total        used        free      shared  buff/cache   available
Mem:            30Gi        11Gi        12Gi       153Mi       6.6Gi        19Gi
Swap:          8.0Gi          0B       8.0Gi
```

Here `free` is 12 GB — plenty. But the pattern is the same: `buff/cache` is 6.6 GB of page cache that the kernel built up from disk reads. It's not waste. It's usable memory being put to work.

The rule: **the system is healthy as long as `available` is above zero and `Swap:used` is not climbing rapidly.**

```bash
# One-liner to check memory health
$ free -h && grep -E '^(SwapFree|Dirty|Writeback)' /proc/meminfo
```

---

## Try It Yourself

### 1. The big picture

```bash
free -h
cat /proc/meminfo
```

### 2. Where is memory going, by process?

```bash
# Sort by RSS (resident, actual physical pages)
ps -eo pid,rss,vsz,comm --sort=-rss | head -15

# VSZ is virtual size (mapped but not necessarily in RAM)
# RSS is what's actually resident in physical memory
```

### 3. Per-VMA detail for a process

```bash
# smaps shows Rss, Pss (proportional), and sharing info per mapping
cat /proc/$$/smaps | grep -E '^(Size|Rss|Pss|Shared_Clean|Private)' | head -30
```

### 4. Proportional memory use across the system

```bash
# PSS (Proportional Set Size) divides shared pages fairly between sharers
sudo cat /proc/*/smaps_rollup 2>/dev/null | grep ^Pss | awk '{s+=$2} END {print s/1024 "MB total PSS"}'
```

### 5. The buddy allocator's free list

```bash
# Higher-order blocks = less fragmentation = better large allocations
cat /proc/buddyinfo
```

### 6. Slab allocator caches

```bash
sudo slabtop -o | head -20
# OBJ SIZE × SLABS × OBJ/SLAB = total memory used by each cache
```

### 7. Page fault counts since boot

```bash
grep -E '^pgfault|^pgmajfault' /proc/vmstat
# pgmajfault counts disk reads due to evicted pages — low is good
```

### 8. Is kswapd doing any work?

```bash
grep -E 'pgsteal_kswapd|pgscan_kswapd|pswp' /proc/vmstat
# Non-zero pgsteal_kswapd means active reclaim — system is under memory pressure
```

### 9. Drop the page cache and watch `free` change

```bash
# Sync first to avoid dropping dirty data
sync

# Drop page cache, dentries, and inodes (3 = all)
# This is SAFE — it's read-only metadata and clean file data
echo 3 | sudo tee /proc/sys/vm/drop_caches

# free will jump; buff/cache will drop
free -h

# The cache refills automatically as you read files
```

### 10. OOM scores across the system

```bash
# Which process would the OOM killer target?
ps -eo pid,oom_score,rss,comm --sort=-oom_score | head -10

# Check if any process is protecting itself from OOM
grep -r -l '\-1000' /proc/*/oom_score_adj 2>/dev/null | head -5
```

---

## Putting It All Together

```
Process writes to a new heap page
    │
    │  CPU walks page table → PTE not present
    ▼
Minor page fault
    │  kernel allocates a physical frame (buddy allocator: order-0)
    │  SLUB if kernel object; direct page if user allocation
    │  zeroes the frame (security: no previous content leaks)
    │  installs PTE: virtual page → physical frame, writable
    │
    ▼ frame is now in memory, process continues

Process reads a file it hasn't opened before
    │
    │  read() → VFS → page cache lookup → miss
    ▼
Major page fault (or synchronous read)
    │  block I/O reads the file's page from disk
    │  page lands in page cache (indexed by inode + offset)
    │  PTE installed (for mmap) or data copied to user buffer (for read())
    │
    ▼ page cache entry persists

File page stays in page cache
    │
    ├── accessed again → serves from RAM, no disk I/O
    │
    └── memory pressure → kswapd
        │
        ├── page is clean → drop for free
        │   (re-read from disk if accessed again → major fault)
        │
        └── page is dirty → writeback, then drop

Anonymous page under pressure
    │
    └── kswapd writes to swap device
        │  frame is freed
        │  PTE updated: page-not-present (swap offset stored elsewhere)
        │
        └── process accesses the page again
            │  major fault: read from swap → restore frame → PTE updated
            ▼  slow, but correct

Memory completely exhausted
    │
    └── OOM killer: select highest oom_score process → SIGKILL
        frame freed, pressure relieved
```

The page cache is the unifying thread: `read()`, `write()`, `mmap()`, executable loading, and library sharing all flow through it. The kernel's job is to keep the working set of your running processes in RAM while using the rest for cache — and to reclaim that cache quietly, before you notice, whenever someone needs more memory.

---

## What's Next

In Chapter 5, we'll look at the **scheduler** — how the kernel decides which of many runnable processes gets CPU time. We've already seen `task_struct` and its `se.vruntime` field (CFS virtual runtime). Chapter 5 fills that in: what CFS virtual runtime means, how nice values translate to actual CPU share, what `load average` really measures, and how to diagnose scheduler-related latency.

---

*Part of the [Linux Deep Dive](./README.md) series.*
