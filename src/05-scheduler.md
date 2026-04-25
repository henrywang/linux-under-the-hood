# Linux Deep Dive #5: The Scheduler — How the Kernel Decides What Runs

*Target: Fedora 43, kernel 6.19.11. Every command in this post is something you can run yourself.*

---

Right now, `/proc/loadavg` on this machine shows:

```
0.24 0.38 0.37 2/1951 75458
```

The `2/1951` means 2 processes are running at this exact moment out of 1,951 that exist. The system has 12 CPUs, so it could run 12 simultaneously — but only 2 have something to do right now. The rest are waiting.

That "waiting" is almost never passive. Processes sleep voluntarily (waiting for I/O, a timer, a lock), then wake up demanding CPU time. Thousands of times per second, the kernel must answer the same question: out of everyone who wants to run right now, who goes next?

That decision belongs to the scheduler.

---

## The Big Picture

The Linux scheduler is not a single algorithm. It's a layered system with multiple scheduling classes, each handling different process types. At the top level:

```
┌─────────────────────────────────────────────────────────┐
│                    Scheduling Classes                    │
│  (highest priority checked first, runs if runnable)      │
│                                                          │
│  SCHED_DEADLINE  ← real-time: explicit deadline/period  │
│  SCHED_FIFO      ← real-time: first in, never preempted │
│  SCHED_RR        ← real-time: round-robin with slice    │
│  SCHED_NORMAL    ← normal processes (you, bash, chrome) │
│  SCHED_BATCH     ← CPU-bound background work            │
│  SCHED_IDLE      ← lower than nice +19                  │
└─────────────────────────────────────────────────────────┘
```

Real-time classes (`DEADLINE`, `FIFO`, `RR`) always preempt normal processes. Within the normal class, the scheduler runs **EEVDF** — Earliest Eligible Virtual Deadline First — the algorithm that replaced CFS in Linux 6.6.

---

## From CFS to EEVDF

To understand EEVDF, you need to understand what it replaced and why.

### The CFS mental model

**CFS (Completely Fair Scheduler)**, introduced in Linux 2.6.23, was built around one idea: every process should get exactly its fair share of CPU time. Not just approximately — exactly. CFS tracks this precisely.

The mechanism: each process has a **vruntime** (virtual runtime) counter, measured in nanoseconds. Every nanosecond a process runs, its vruntime increases. But the rate of increase depends on the process's priority — a high-priority process's vruntime grows more slowly, so it gets selected to run more often.

CFS stored all runnable processes in a **red-black tree** keyed by vruntime. The scheduler always picked the process with the smallest vruntime — the one that has received the least CPU time. After running for one time slice, its vruntime increased, and it was reinserted at its new position. Whoever now had the smallest vruntime ran next.

This made the scheduler O(log n) for most operations (red-black tree insertion/lookup), with the property that no process ever accumulated more than one time slice of "debt" relative to any other.

### The EEVDF refinement

CFS had a problem: it picked the "most deserving" process but had no concept of *when* that process needed to run. A low-latency audio thread and a CPU-hungry compiler could have the same vruntime, but they have wildly different timing requirements.

**EEVDF** (Earliest Eligible Virtual Deadline First, landed in Linux 6.6) adds an explicit **virtual deadline** alongside vruntime. Each process gets a time slice — a requested service quantum. Its virtual deadline is computed as:

```
virtual_deadline = vruntime + time_slice / weight
```

The scheduler picks the runnable process whose virtual deadline is earliest — not just the one with the smallest vruntime. This gives the scheduler a way to prefer short-running, latency-sensitive tasks over long-running ones even when they have similar accumulated CPU time.

### What you can see

On this machine (kernel 6.19.11), every `SCHED_NORMAL` process has an explicit time slice of **2.8 ms**:

```bash
$ chrt -p $$
pid 75352's current scheduling policy: SCHED_OTHER
pid 75352's current scheduling priority: 0
pid 75352's current runtime parameter: 2800000
```

That `2800000` is nanoseconds — 2.8 ms. That's the EEVDF slice. CFS had no such fixed slice; it calculated a "target latency" divided by the number of runnable processes. EEVDF makes it explicit.

You can see each process's vruntime directly:

```bash
$ cat /proc/$$/sched | head -10
bash (75352, #threads: 1)
-------------------------------------------------------------------
se.exec_start                                :      47483595.680258
se.vruntime                                  :     102684.252917
se.sum_exec_runtime                          :          5.248837
se.nr_migrations                             :                 0
nr_switches                                  :                62
nr_voluntary_switches                        :                26
nr_involuntary_switches                      :                 0
se.load.weight                               :           1048576
```

Key fields:
- `se.vruntime` — 102,684 ms accumulated virtual time. This is the scheduler's currency.
- `se.sum_exec_runtime` — only 5.2 ms of actual CPU time. This process barely runs.
- `nr_voluntary_switches` — 26 times this process gave up the CPU willingly (waiting for I/O or input).
- `nr_involuntary_switches` — 0. The scheduler never had to yank the CPU away from it. It always yielded on its own.

Compare that to a busier process, systemd (PID 1):

```bash
$ cat /proc/1/sched | head -10
systemd (1, #threads: 1)
-------------------------------------------------------------------
se.exec_start                                :      47483479.684417
se.vruntime                                  :        106.342126
se.sum_exec_runtime                          :       2724.968813
nr_switches                                  :              11033
nr_voluntary_switches                        :              10422
nr_involuntary_switches                      :                611
se.load.weight                               :           1048576
```

systemd has `nr_involuntary_switches: 611` — 611 times the scheduler decided "time's up" and switched to someone else. That's normal for a long-running process handling many short requests.

---

## Nice Values and Weights

Not all SCHED_NORMAL processes are equal. The **nice value** controls how fast a process's vruntime accumulates — and therefore how much CPU time it gets.

Nice values range from **-20** (highest priority) to **+19** (lowest). Default is 0. The name comes from "being nice" to other processes: a process with a high nice value voluntarily gives up CPU time.

The kernel converts nice values to **weights** using a fixed table where each step is approximately 1.25×:

| Nice | Weight | Relative to nice-0 |
|------|--------|---------------------|
| -20  | 88761  | ~86× more           |
| -10  | 9548   | ~9.3×               |
| -5   | 3121   | ~3.1×               |
| 0    | 1024   | baseline            |
| +5   | 335    | ~0.33×              |
| +10  | 110    | ~0.11×              |
| +19  | 15     | ~68× less           |

The vruntime delta per nanosecond of real time is:

```
delta_vruntime = delta_exec × (1024 / weight)
```

A nice -20 process (weight 88761) accumulates vruntime 86× *slower* than a nice 0 process. Since the scheduler picks based on earliest deadline (derived from vruntime), the nice -20 process runs 86× more often. A nice +19 process (weight 15) accumulates vruntime 68× *faster* — it falls behind quickly and runs rarely.

You can see nice values live on this machine:

```bash
$ ps -eo pid,ni,comm --sort=ni | head -5
    PID  NI COMMAND
      9 -20 rcu_tasks_kthread
     11 -20 rcu_tasks_rude_
     12 -20 rcu_tasks_trace
   5297 -11 pipewire
```

PipeWire (the audio server) runs at nice -11. That's why audio doesn't stutter when the CPU is loaded. The kernel scheduler ensures it gets CPU time well ahead of normal-priority processes.

```bash
$ ps -eo pid,ni,comm --sort=-ni | head -3
    PID  NI COMMAND
 136282 +19 khugepaged
  75352   0 bash
```

`khugepaged` — the background thread that collapses small pages into huge pages — runs at nice +19. It gets CPU time only when nothing else wants it. Perfect for a background maintenance task.

You can change nice values:

```bash
# Start a process at nice +10
nice -n 10 make -j12

# Change a running process's nice value (requires root to go more negative)
renice -n 5 -p <pid>
```

---

## Scheduling Classes in Detail

When the scheduler runs, it checks each class from highest to lowest priority. If a real-time process is runnable, it runs — the normal class doesn't get a look in.

### SCHED_DEADLINE

The highest priority class. A process declares its needs explicitly:

```
period=100ms, runtime=10ms, deadline=50ms
```

"Every 100ms, give me 10ms of CPU time, and deliver it within 50ms of each period's start."

The kernel enforces this using **CBS (Constant Bandwidth Server)** — it tracks how much runtime each deadline task has consumed and throttles it if it overruns. Admission control rejects new DEADLINE tasks if they'd make the system unschedulable.

Real-time audio, video capture, and industrial control systems use this. On this machine, the RT bandwidth pool is configured conservatively:

```bash
$ cat /proc/sys/kernel/sched_rt_period_us
1000000
$ cat /proc/sys/kernel/sched_rt_runtime_us
950000
```

RT tasks (FIFO, RR, DEADLINE combined) can use at most 95% of each 1-second period. The remaining 5% is reserved for normal processes — a safety valve to prevent a runaway RT process from completely starving the system.

### SCHED_FIFO and SCHED_RR

Classic POSIX real-time policies. Both run at a fixed priority (1–99, higher is better):

- **FIFO**: Runs until it blocks or explicitly yields. No time slicing. Preempts any lower-priority process immediately.
- **RR** (Round Robin): Like FIFO, but with a time slice. After the slice expires, it moves to the back of its priority queue. Same priority means round-robin; higher priority always wins.

The kernel threads that migrate processes between CPUs use FIFO:

```bash
$ ps -eo pid,cls,rtprio,comm | grep migration | head -4
   30 FF      99 migration/0
   37 FF      99 migration/1
   44 FF      99 migration/2
   51 FF      99 migration/3
```

`FF` = FIFO, rtprio 99 — the highest possible RT priority. These threads must run instantly when a CPU balance decision is made; nothing should ever preempt them.

### SCHED_NORMAL (EEVDF)

The class that runs the rest: your shell, your browser, your compiler. This is where EEVDF operates. The `se.load.weight: 1048576` in the `/proc/$$/sched` output is the internal representation of nice 0 (1024 × 1024 for precision scaling).

### SCHED_BATCH and SCHED_IDLE

- **BATCH**: Like NORMAL, but the scheduler assumes it's a CPU-bound job and avoids waking it up aggressively. Useful for batch processing that shouldn't interrupt interactive tasks.
- **IDLE**: Runs only when no other task — not even SCHED_NORMAL — is runnable. Lower than nice +19.

---

## Preemption: Who Can Interrupt Whom

A process runs until something better comes along. But "something better" can mean different things depending on the kernel's preemption configuration.

This machine's kernel is compiled with `PREEMPT_DYNAMIC`:

```bash
$ uname -a
Linux hostname 6.19.11-200.fc43.x86_64 #1 SMP PREEMPT_DYNAMIC ...
```

`PREEMPT_DYNAMIC` means the preemption model can be changed at runtime via boot parameter. The possible modes:

| Mode | When preemption happens |
|------|------------------------|
| `none` | Only at explicit yield points (voluntary) |
| `voluntary` | At yield points + explicit preemption checks |
| `full` | At almost any kernel code point |

`full` preemption gives the best latency — a high-priority process can interrupt even kernel code (with appropriate locking). `none` gives the best throughput on server workloads.

Regardless of preemption mode, certain events always trigger a reschedule check:
- A timer interrupt fires (every 1ms by default with `CONFIG_HZ=1000`)
- A process unblocks (wakes from sleep)
- A process's time slice expires
- A higher-priority process becomes runnable

When a reschedule is needed, the kernel sets the `TIF_NEED_RESCHED` flag on the current process. The scheduler runs at the next safe preemption point.

---

## SMP: Per-CPU Run Queues and Load Balancing

With 12 CPUs, the scheduler doesn't have one global queue — it has **one run queue per CPU**. This eliminates a massive bottleneck: CPUs don't fight over a single lock to pick their next task.

```bash
$ nproc
12
```

Each CPU has its own:
- **EEVDF run queue**: the red-black tree of normal-priority runnable tasks
- **RT run queue**: lists of RT tasks per priority level
- **DL run queue**: deadline tasks with their CBS accounting

When you fork a new process, the kernel picks which CPU to assign it to. When a process wakes up, it returns to the CPU it last ran on — its data is still warm in that CPU's L1/L2 cache. The scheduler tries to keep processes on the same CPU as long as the load stays balanced.

### Load balancing

**Load balancing** runs periodically, triggered by the scheduler tick. The process:

1. Find the busiest CPU in the current **scheduling domain**
2. If it's significantly busier than the local CPU, pull some tasks over
3. Prefer tasks that are "cache cold" (haven't run recently) to minimize cache pollution from migration

The `migration/N` threads (FIFO priority 99) handle the actual task movement:

```bash
$ ps -eo pid,cls,rtprio,comm | grep migration
   30 FF      99 migration/0
   37 FF      99 migration/1
   44 FF      99 migration/2
   51 FF      99 migration/3
   58 FF      99 migration/4
   65 FF      99 migration/5
   72 FF      99 migration/6
   79 FF      99 migration/7
   86 FF      99 migration/8
   93 FF      99 migration/9
  100 FF      99 migration/10
  107 FF      99 migration/11
```

One per CPU. They run at the highest RT priority so they can interrupt anything to perform the migration instantly.

### Scheduling domains

The scheduler understands CPU topology — SMT siblings (hyperthreads), cores sharing an L2, NUMA nodes. Balancing is done hierarchically: first balance within a core (hyperthreads), then within a socket, then between sockets (NUMA). Crossing NUMA boundaries is expensive and done conservatively.

```bash
$ cat /sys/devices/system/cpu/cpu0/topology/core_siblings_list
0-11
```

On this machine, all 12 CPUs are in one scheduling domain (no NUMA).

### Autogroups

With `sched_autogroup_enabled` (on by default):

```bash
$ cat /proc/sys/kernel/sched_autogroup_enabled
1
```

The scheduler groups processes by session. If you run `make -j12` in a terminal, all 12 build jobs share one autogroup. A simultaneous `make -j12` in another terminal gets its own autogroup. Each group gets 50% of CPU — regardless of how many tasks each contains. This prevents a parallel build from starving an interactive shell session even when it spawns many more threads.

---

## Load Average: What It Actually Means

```bash
$ cat /proc/loadavg
0.24 0.38 0.37 2/1951 75458
```

The three numbers (0.24, 0.38, 0.37) are **exponentially weighted moving averages** (EWMA) of the number of processes in:
- **RUNNING state** — actually on a CPU or ready to run
- **UNINTERRUPTIBLE state** — D state, waiting for I/O that cannot be interrupted

The time constants are 1-minute, 5-minute, and 15-minute. The "average" doesn't mean average over that full interval — it's an EWMA that weights recent values more heavily, with a half-life roughly equal to that time constant.

A load average of 1.0 on a single-CPU machine means the CPU is perfectly saturated. On this 12-CPU machine, 12.0 would mean saturation. At 0.24, we have plenty of headroom.

### The D-state trap

The most important thing to understand about load average: **it includes uninterruptible I/O wait**, not just CPU usage.

A process blocked on a disk read is in D state. It contributes to load average even though it's not using CPU. This is why you can have a high load average with low CPU utilization:

```bash
# High load, low CPU: probably I/O-bound
$ uptime
 13:47:01 up  1:46,  2 users,  load average: 8.23, 7.91, 6.44
$ top
%Cpu(s):  3.2 us,  0.8 sy,  0.0 ni, 94.3 id,  1.5 wa,  0.0 hi,  0.2 si
```

CPUs are idle 94% of the time — but 8+ tasks are blocked on disk, driving up load average without touching CPU.

This distinction matters enormously for diagnosis. Load average alone doesn't tell you whether you're CPU-bound or I/O-bound.

---

## Diagnosing Scheduling Latency

When a process is "slow" or "laggy," the question is: is it slow because it can't get CPU, or because it's doing something slow? The scheduler exposes the data to answer this.

### Enable schedstats

Per-process scheduling statistics are disabled by default (they add overhead). Enable them:

```bash
echo 1 | sudo tee /proc/sys/kernel/sched_schedstats
```

Now `/proc/PID/schedstat` has three columns:

```bash
$ cat /proc/$$/schedstat
6814340 4857 35
```

| Column | Meaning |
|--------|---------|
| 1 | Time spent running on CPU (nanoseconds) |
| 2 | Time spent waiting in the run queue (nanoseconds) |
| 3 | Number of timeslices run |

Column 2 is the key diagnostic: **time spent runnable but not running**. If a process is slow and this number is high, it's not getting CPU — the scheduler is the bottleneck. If this number is low, the process itself is doing something slow (blocking on I/O, lock contention, computation).

For this bash shell: 4,857 ns of wait time across 35 timeslices. Negligible — the shell is interactive and almost never needs CPU.

### Voluntary vs involuntary switches

From `/proc/PID/sched`:

```
nr_voluntary_switches   ← process called schedule() itself (I/O, sleep, lock wait)
nr_involuntary_switches ← scheduler took the CPU away (time slice expired)
```

A process with many involuntary switches is CPU-hungry — it never gives up the CPU voluntarily; the timer interrupt keeps yanking it away. A process with many voluntary switches is latency-sensitive — it keeps waking up, doing a small piece of work, then blocking again.

For the bash shell: 26 voluntary, 0 involuntary — it always yields before its slice expires. For systemd: 10,422 voluntary, 611 involuntary — it does a lot of short work (voluntary) but occasionally runs long enough to get preempted.

### perf sched

For deeper analysis, `perf sched` records scheduler events at the hardware level:

```bash
# Record scheduler events for 5 seconds
sudo perf sched record -- sleep 5

# Summarize per-process scheduling latency
sudo perf sched latency | head -20
```

This shows each process's maximum and average scheduling latency — time from becoming runnable to actually running. A well-tuned interactive system should have p99 latency under a few milliseconds for normal processes.

### Run queue length

```bash
# vmstat: the 'r' column is tasks running + waiting for CPU
$ vmstat 1 3
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 19581972 308096 6964064    0    0    13     8  170  319  1  0 98  0  0
 1  0      0 19581972 308096 6964064    0    0     0     0  219  403  0  0 99  0  0
 1  0      0 19581972 308096 6964064    0    0     0     0  185  361  0  0 99  0  0
```

`r=2` means 2 tasks are currently running or waiting for a CPU. On a 12-CPU machine, this is nothing. If `r` consistently exceeds `nproc`, tasks are queuing for CPU time.

---

## Try It Yourself

### 1. See EEVDF slice for any process

```bash
chrt -p $$
chrt -p 1
```

### 2. Read the scheduler's view of any process

```bash
cat /proc/$$/sched
cat /proc/1/sched
```

Look at `se.vruntime`, `nr_voluntary_switches`, and `nr_involuntary_switches`.

### 3. See nice values for all processes

```bash
ps -eo pid,ni,comm --sort=ni | head -10
ps -eo pid,ni,comm --sort=-ni | head -10
```

### 4. Run something at a lower priority

```bash
nice -n 10 sha256sum /dev/urandom &
kill %1
```

### 5. Check real-time scheduling on kernel threads

```bash
ps -eo pid,cls,rtprio,comm | grep -v '\-$' | head -20
# FF = FIFO, RR = round-robin, TS = time-sharing (normal)
```

### 6. Enable and read per-process wait time

```bash
echo 1 | sudo tee /proc/sys/kernel/sched_schedstats
cat /proc/$$/schedstat
# col 1: ns running, col 2: ns waiting, col 3: timeslices
```

### 7. Watch the run queue in real time

```bash
vmstat 1
# 'r' column = tasks running + waiting for CPU
# consistently > nproc means CPU saturation
```

### 8. Find CPU-hungry processes

```bash
top -b -n 1 | head -20
```

### 9. Check autogroup status

```bash
cat /proc/sys/kernel/sched_autogroup_enabled
# 1 = on: parallel builds in different terminals each get a fair CPU share
```

### 10. Profile scheduling latency with perf

```bash
sudo perf sched record -- sleep 5
sudo perf sched latency | head -20
```

---

## Putting It All Together

The scheduler's job seems simple — pick the next process — but the constraints are brutal: maximize CPU utilization, minimize latency for interactive tasks, enforce real-time guarantees, scale across dozens of CPUs, and do all of this in microseconds.

The layered design handles it:

```
Timer interrupt (every 1ms)
    │
    ▼
Check scheduling classes top-to-bottom:
    │
    ├─ SCHED_DEADLINE task runnable? → run it (CBS enforcement)
    │
    ├─ SCHED_FIFO/RR task runnable? → run it (priority order)
    │
    └─ SCHED_NORMAL: EEVDF
           │
           ├── vruntime tracks accumulated CPU time
           │   (weighted by nice: higher priority → slower vruntime growth)
           │
           ├── virtual_deadline = vruntime + slice / weight
           │   (2.8ms slice, explicit since EEVDF in Linux 6.6)
           │
           └── pick runnable task with earliest virtual_deadline
    │
    ├── Per-CPU run queues: no global lock
    ├── Load balancer: pulls tasks from busy CPUs
    └── migration/N (FIFO 99): moves tasks between CPUs

Accounting you can read:
    ├── nr_involuntary_switches → process is CPU-hungry
    ├── schedstat col 2 → time waiting for CPU (diagnosis)
    └── load average → EWMA of (RUNNING + UNINTERRUPTIBLE)
                       high load + low CPU = I/O bottleneck
```

The `2/1951` in `/proc/loadavg` is the whole story compressed: out of 1,951 processes, 2 had something to compute right now. The scheduler gave them a CPU in microseconds, and the other 1,949 never noticed.

The next chapter goes one level deeper into what all those idle processes are waiting for: the filesystem. Every `read()`, `write()`, `open()` passes through the **VFS layer** — the kernel abstraction that makes ext4, btrfs, tmpfs, and a network socket look identical to userspace.

---

*Part of the [Linux Deep Dive](./README.md) series.*
