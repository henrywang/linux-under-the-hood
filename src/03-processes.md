# Linux Deep Dive #3: Processes — fork, exec, and the Process Table

*Target: Fedora 43, kernel 6.19.11. Every command in this post is something you can run yourself.*

---

You've been using processes since you first typed a command. But what *is* a process, really?

Not "a running program" — that's the textbook answer, and it's too vague to be useful. A process is a specific data structure in kernel memory, a virtual address space, a set of open file descriptors, and a position in a tree that traces back to PID 1. Understanding what the kernel actually stores — and what happens when you call `fork()` or `exec()` — gives you a mental model that makes everything else click: why forking is fast, how exec replaces a program without changing its PID, why zombie processes exist, and what threads actually are under the hood.

This post traces the lifecycle of a process from `fork()` to `exit()`, using the kernel source and `/proc` as our specimen jars.

---

## The Process at a Glance

Before drilling in, here's the lifecycle we'll trace:

```
Parent process calls fork()
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  fork()                                                      │
│  Kernel creates a new task_struct for the child             │
│  CoW: parent and child share physical pages (read-only)     │
│  Child gets a new PID; parent gets child's PID back         │
└─────────────────────────────────────────────────────────────┘
    │
    │  (child continues here — fork() returned 0)
    ▼
┌─────────────────────────────────────────────────────────────┐
│  exec()                                                      │
│  Child replaces its virtual address space with new program  │
│  Old code/data/stack gone; new ELF loaded                   │
│  PID unchanged, some file descriptors inherited             │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Process runs                                                │
│  Scheduled on CPU, makes syscalls, handles signals          │
│  task_struct tracks state: R, S, D, T, Z                   │
└─────────────────────────────────────────────────────────────┘
    │
    │  process calls exit() or is killed
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Zombie state                                                │
│  Memory freed, but task_struct remains                      │
│  Parent calls wait() → kernel reaps the zombie              │
└─────────────────────────────────────────────────────────────┘
```

---

## The task_struct: What the Kernel Stores

Every process and thread on a Linux system has exactly one **`task_struct`** — the kernel's complete record of that execution context. It lives in kernel memory, allocated when the process is created and freed when the parent reaps it with `wait()`.

The `task_struct` is defined in `include/linux/sched.h`. It's large — hundreds of fields. The ones that matter most:

```c
struct task_struct {
    /* State */
    unsigned int        __state;     /* TASK_RUNNING, TASK_INTERRUPTIBLE, ... */

    /* Identity */
    pid_t               pid;         /* process ID */
    pid_t               tgid;        /* thread group ID (= pid for single-threaded) */

    /* Family */
    struct task_struct *parent;      /* parent process */
    struct list_head    children;    /* list of child processes */
    struct list_head    sibling;     /* position in parent's children list */

    /* Memory */
    struct mm_struct   *mm;          /* virtual address space descriptor */

    /* Files */
    struct files_struct *files;      /* open file descriptor table */
    struct fs_struct    *fs;         /* filesystem context: cwd, root, umask */

    /* Credentials */
    const struct cred  *cred;        /* UID, GID, capabilities */

    /* Signals */
    struct signal_struct *signal;    /* signal handlers, pending signals */

    /* Scheduling */
    int                 prio;        /* scheduling priority */
    u64                 se.vruntime; /* CFS virtual runtime (more in Chapter 5) */
};
```

This is the nucleus of the entire process model. When you run `ps`, it reads `/proc/[pid]/stat`, which is synthesized from the `task_struct`. When the scheduler picks the next process to run, it's choosing between `task_struct`s. When you send a signal with `kill`, the kernel sets a bit in the target's `task_struct`.

The `mm` pointer (memory descriptor) points to `mm_struct`, which describes the entire virtual address space. The `files` pointer points to the open file descriptor table. Both of these can be *shared* between `task_struct`s — that's what threads are, as we'll see.

---

## Exploring Processes via /proc

The kernel exposes every `task_struct` through the **`/proc` filesystem**. For every running process, there's a directory `/proc/[pid]/`:

```bash
$ ls /proc/1/
attr/     cmdline  environ  fd/   maps  mountstats  ns/      root
cgroup    comm     exe      io    mem   net/         oom_adj  smaps
coredump_filter    gid_map  ...   mounts             pagemap  stat  status
```

The most useful files:

```bash
# What command is PID 1?
$ cat /proc/1/cmdline | tr '\0' ' '
/usr/lib/systemd/systemd --switched-root --system --deserialize 31

# Human-readable status
$ cat /proc/1/status
Name:   systemd
Umask:  0000
State:  S (sleeping)
Tgid:   1
Pid:    1
PPid:   0
...
VmRSS:  15348 kB
Threads:        1

# All open file descriptors of NetworkManager
$ ls -la /proc/1618/fd/ | head -8
lrwx------. 1 root root 64 Apr 23 09:12 0 -> /dev/null
lrwx------. 1 root root 64 Apr 23 09:12 1 -> /dev/null
lrwx------. 1 root root 64 Apr 23 09:12 2 -> /dev/null
lrwx------. 1 root root 64 Apr 23 09:12 4 -> 'socket:[28431]'
lrwx------. 1 root root 64 Apr 23 09:12 5 -> 'socket:[28432]'
```

(PID 1618 is NetworkManager on this system, as established in Chapter 2.)

The `/proc/[pid]/maps` file shows the full virtual address space:

```bash
$ cat /proc/1618/maps | head -12
55f8e4d20000-55f8e4d7a000 r--p 00000000 fd:01 524302  /usr/bin/NetworkManager
55f8e4d7a000-55f8e5012000 r-xp 0005a000 fd:01 524302  /usr/bin/NetworkManager
55f8e5012000-55f8e5134000 r--p 002f2000 fd:01 524302  /usr/bin/NetworkManager
55f8e5134000-55f8e5141000 rw-p 00413000 fd:01 524302  /usr/bin/NetworkManager
55f8e67a3000-55f8e6a1b000 rw-p 00000000 00:00 0       [heap]
7f9b3c000000-7f9b3c021000 rw-p 00000000 00:00 0
7ffce2a3c000-7ffce2a5e000 rw-p 00000000 00:00 0       [stack]
7ffce2b76000-7ffce2b78000 r-xp 00000000 00:00 0       [vdso]
```

Each line is a **VMA** (Virtual Memory Area) — a contiguous range of virtual addresses with a single set of permissions. `r-xp` means read + execute + private (not shared). The executable has separate VMAs for the read-only text segment, the writable data segment, the heap, and the stack. Chapter 4 goes deep on this.

---

## The Process Tree

Every process except PID 1 has a parent. The parent-child relationship forms a tree rooted at systemd:

```bash
$ pstree -p | head -20
systemd(1)─┬─ModemManager(1203)─┬─{ModemManager}(1221)
           │                     └─{ModemManager}(1222)
           ├─NetworkManager(1618)─┬─{NetworkManager}(1636)
           │                      └─{NetworkManager}(1637)
           ├─dbus-broker(1432)
           ├─gdm(1801)─┬─gdm-session-wor(2015)
           │            └─{gdm}(1803)
           ├─kthreadd(2)─┬─kworker/0:0H(12)
           │              ├─ksoftirqd/0(9)
           │              └─...
           └─...
```

The `{...}` entries — like `{NetworkManager}(1636)` — are threads. Multiple `task_struct`s sharing the same memory space appear under their process as thread entries.

Notice PID 2 (`kthreadd`). Kernel threads — `kworker`, `ksoftirqd`, `ksystemd-udevd` — all descend from PID 2 and never enter user space. Their `mm` pointer is NULL; they only ever run kernel code.

You can walk the parent chain manually through `/proc`:

```bash
# Who is NetworkManager's parent?
$ cat /proc/1618/status | grep PPid
PPid:   1

# Shell in a shell: check your current PID and its parent
$ echo $$
47234

$ cat /proc/47234/status | grep -E '^(Pid|PPid)'
Pid:    47234
PPid:   47198

$ cat /proc/47198/status | grep -E '^(Name|Pid)'
Name:   bash
Pid:    47198
```

---

## fork(): Creating a New Process

`fork()` is the only way to create a new process in Linux. Every process you've ever seen — every shell command, every daemon, every application — was created by a `fork()` call somewhere in its ancestry.

### The syscall

At the C library level, `fork()` is a function that returns twice: once in the parent, once in the child:

```c
pid_t pid = fork();
if (pid == 0) {
    // We're in the child — fork() returned 0
    printf("child PID: %d\n", getpid());
    exit(0);
} else if (pid > 0) {
    // We're in the parent — pid is the child's PID
    printf("parent sees child PID: %d\n", pid);
    wait(NULL);  // wait for child to exit
} else {
    // pid == -1: fork failed (ENOMEM, EAGAIN, etc.)
    perror("fork");
}
```

Under the hood, glibc's `fork()` calls the **`clone()`** syscall — the real workhorse. `fork()` is `clone()` with a specific set of flags:

```c
// What fork() does internally (simplified):
clone(SIGCHLD, 0);
```

`clone()` gives fine-grained control over what the child shares with the parent. `fork()` shares nothing (new memory space, new FD table copy). `pthread_create()` shares almost everything. Same syscall, different flags.

### What the kernel does on fork()

The kernel executes `copy_process()` in `kernel/fork.c`. The steps:

```
clone() syscall
    │
    ├── Allocate a new task_struct for the child
    │
    ├── Copy the parent's task_struct fields
    │   ├── pid  ← new PID, assigned from the PID namespace
    │   ├── ppid ← parent's PID
    │   └── state ← TASK_RUNNING
    │
    ├── copy_mm() ── duplicate the virtual address space (CoW — see below)
    │
    ├── copy_files() ── copy the file descriptor table
    │   (child gets the same FD numbers, pointing to same open file entries)
    │
    ├── copy_fs() ── copy the filesystem context
    │   (child inherits parent's cwd, root, umask)
    │
    ├── copy_sighand() ── copy signal handlers
    │
    └── Place child on the scheduler's run queue
```

After `copy_process()` returns, the kernel returns the child's PID to the parent, and the child is placed on the run queue. Both parent and child are now runnable; the scheduler decides which goes first.

### The child starts where fork() was called

The child doesn't start at `main()`. It starts at the exact instruction after `fork()` returned, with a copy of the parent's entire register state — stack pointer, instruction pointer, everything. This is how fork can "return twice": from the child's perspective it woke up having just returned from a syscall with 0 in `rax`, so `fork()` returns 0.

---

## Copy-on-Write: Why fork() Is Fast

Duplicating a process's virtual address space sounds expensive — a process might have gigabytes of memory mapped. Copying it all on every `fork()` would make shells unusable.

The solution is **copy-on-write (CoW)**.

When `fork()` calls `copy_mm()`, it doesn't copy physical memory pages. Instead:

1. The parent's page table entries are duplicated — the child gets its own page table pointing to the *same physical pages*.
2. Both the parent's and child's PTEs for those pages are marked **read-only**.
3. If either process writes to a shared page, a **page fault** fires.
4. The page fault handler sees this is a CoW fault: it allocates a new physical page, copies the content, updates the faulting process's PTE to point to the new page (with write permission restored), and resumes execution.

The result: `fork()` costs roughly the time to copy the page table structures — not the pages themselves. You pay only for pages that actually get written after the fork.

```
Before fork():
Parent VMAs                  Physical RAM
  PTE: 0x1000 → frame A ←── [frame A: code/data]
  PTE: 0x2000 → frame B ←── [frame B: data]

After fork() — CoW setup:
Parent VMAs                  Physical RAM
  PTE: 0x1000 → frame A (ro) ←── [frame A: code/data]   ← same frame
  PTE: 0x2000 → frame B (ro) ←── [frame B: data]         ← same frame
Child VMAs                           ↑
  PTE: 0x1000 → frame A (ro) ────────┘
  PTE: 0x2000 → frame B (ro) ─────────────────────────────┘

After child writes to 0x2000 (page fault → CoW copy):
Parent VMAs                  Physical RAM
  PTE: 0x1000 → frame A (ro) ←── [frame A: code/data]   (unchanged)
  PTE: 0x2000 → frame B (ro) ←── [frame B: original]    (unchanged)
Child VMAs
  PTE: 0x1000 → frame A (ro) ←── [frame A: code/data]   (still shared)
  PTE: 0x2000 → frame C (rw) ←── [frame C: modified]    (new copy)
```

If you immediately call `exec()` after `fork()` — the common case for shells — the child's address space is thrown away before it writes anything. CoW means this common path allocates almost no physical memory at all.

---

## exec(): Replacing the Process Image

`fork()` creates a copy. `exec()` replaces a process's program with a different one.

### The syscall

```c
execve("/usr/bin/ls", (char *[]){"/usr/bin/ls", "-la", NULL},
       environ);
// If execve returns, something went wrong
perror("execve");
exit(127);
```

`execve()` takes three arguments: the path to the new program, the argument array (`argv`), and the environment array (`envp`). If it succeeds, it **never returns** — the calling process's virtual address space is completely replaced.

### What the kernel does on exec()

The kernel's `do_execve()` in `fs/exec.c`:

```
execve() syscall
    │
    ├── Open and read the target file
    │   Starts with #! (shebang)? → redirect to the interpreter
    │   Looks like an ELF binary? → call the ELF loader
    │
    ├── Flush the old virtual address space
    │   (all VMAs unmapped, page tables cleared, mm_struct reset)
    │
    ├── Load the new ELF binary:
    │   ├── Map text segment (r-xp) into virtual memory
    │   ├── Map data segment (rw-p) into virtual memory
    │   ├── Set up a new stack
    │   └── Dynamically linked? → map ld.so first; it loads libraries
    │
    ├── Push argc, argv, envp onto the new stack
    │
    ├── Set instruction pointer to the ELF entry point
    │
    └── Return to user space at _start()
```

The key: the PID does not change. The `task_struct` is the same one. What changed is the `mm` field — the old memory descriptor is gone, replaced with one describing the new program's address space.

```bash
# exec() replaces the program but keeps the PID
$ echo $$
47234

$ exec bash       # replace this bash with a new bash
$ echo $$
47234             # same PID — exec() doesn't create a new process
```

### What exec() preserves

Despite replacing everything, exec carries forward some state:

| Preserved | Discarded |
|-----------|-----------|
| PID, PPID | Virtual address space |
| Open file descriptors (unless `O_CLOEXEC`) | Memory mappings |
| Real UID/GID | Signal handlers (reset to default) |
| Ignored signals | |
| cwd, root directory | |
| Resource limits (`ulimit`) | |

The `O_CLOEXEC` flag tells the kernel to close an FD when exec happens. Modern code sets it on almost every FD that shouldn't leak into child programs — sockets, pipe ends, anything sensitive.

---

## The fork + exec Pattern

The standard Unix pattern for running a program is the **fork-exec idiom**:

```c
pid_t pid = fork();
if (pid == 0) {
    // Child: set up redirections/pipes, then exec
    dup2(pipe_fd[1], STDOUT_FILENO);  // redirect stdout to pipe
    close(pipe_fd[0]);
    close(pipe_fd[1]);
    execvp("ls", argv);
    exit(127);  // only reached if exec fails
} else {
    // Parent: manages the child
    close(pipe_fd[1]);
    read(pipe_fd[0], buf, sizeof(buf));
    wait(NULL);
}
```

This is exactly how your shell works. When you type `ls -la`:

```
bash (PID 47234)
    │  calls fork()
    ├──────────────────────────────────┐
    │                                  ▼
    │                         bash (PID 47235)  ← child copy
    │                                  │  sets up redirections
    │                                  │  calls execve("/usr/bin/ls", ...)
    │                                  ▼
    │                         ls (PID 47235)    ← same PID, new program
    │                                  │  runs, writes output
    │                                  │  exits(0)
    │                                  │
    │  wait() returns ─────────────────┘
    ▼
bash (PID 47234)  ← resumes with exit status
```

The genius of this design: all the setup work (redirections, pipe wiring, environment changes) happens in the child *after* fork but *before* exec. The parent doesn't have to know about any of it.

You can watch every `execve()` call a command makes:

```bash
$ strace -e execve bash -c 'ls /tmp' 2>&1
execve("/bin/bash", ["bash", "-c", "ls /tmp"], 0x... /* environ */)
...
[pid 47235] execve("/usr/bin/ls", ["ls", "/tmp"], 0x... /* environ */)
```

Two `execve` calls: one for bash itself at startup, one for `ls` inside bash.

---

## Process States

Every process is always in one of a handful of states. The `__state` field in `task_struct` tracks this:

```
TASK_RUNNING (R)
    The process is either running on a CPU right now,
    or sitting in a run queue waiting for a CPU.
    Both cases show as 'R' in ps.

TASK_INTERRUPTIBLE (S)
    Sleeping, waiting for an event: I/O complete, timer,
    mutex, network data, etc. CAN be woken by a signal.

TASK_UNINTERRUPTIBLE (D)
    Sleeping, waiting for an event. CANNOT be interrupted
    by signals — not even SIGKILL. Typically waiting on
    disk I/O, kernel locks, or NFS. A process stuck in D
    state usually means a device or mount isn't responding.

__TASK_STOPPED (T)
    Stopped by SIGSTOP or job control (Ctrl+Z).
    Resumes on SIGCONT.

EXIT_ZOMBIE (Z)
    Has exited, but parent hasn't called wait() yet.
    No memory, no CPU time — just the task_struct remains
    to hold the exit status.
```

You see these in `ps` output:

```bash
$ ps aux | head -12
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0 173936 15348 ?        Ss   Apr22   0:01 /usr/lib/systemd/systemd
root           2  0.0  0.0      0     0 ?        S    Apr22   0:00 [kthreadd]
root           9  0.0  0.0      0     0 ?        S    Apr22   0:00 [ksoftirqd/0]
root        1432  0.0  0.0  10604  5124 ?        Ss   Apr22   0:00 /usr/bin/dbus-broker
root        1618  0.0  0.1  54680 18204 ?        Ss   Apr22   0:01 /usr/bin/NetworkManager
xiaofeng   47234  0.0  0.0  14228  8316 pts/0    Ss   09:15   0:00 -bash
```

The `STAT` column encodes state plus modifiers:

| Letter | Meaning |
|--------|---------|
| `R` | Running or runnable |
| `S` | Interruptible sleep |
| `D` | Uninterruptible sleep (disk/kernel wait) |
| `Z` | Zombie |
| `T` | Stopped |
| `I` | Idle kernel thread |
| `s` | Session leader |
| `l` | Multi-threaded |
| `+` | Foreground process group |
| `N` | Low priority (nice > 0) |

`D` state is worth understanding on its own. A process in D state **cannot be killed** — not even with `SIGKILL`. The signal is queued, but the process can't check it until the kernel operation it's waiting for completes. If a disk or NFS mount stops responding, processes waiting on I/O pile up in D state and stay there until the device responds or the mount is force-unmounted.

```bash
# Find D-state processes (usually none; if you see them, investigate the mount or device):
$ ps aux | awk '$8 ~ /^D/'
```

---

## Threads: Processes That Share Memory

So far, "process" and "task_struct" have been interchangeable. But what about threads?

In Linux, there's no fundamental distinction between a process and a thread at the kernel level. **A thread is just a `task_struct` that shares its `mm` (virtual address space) with another `task_struct`.**

This sharing is controlled by the flags passed to `clone()`:

```c
// fork(): new address space, new FD table, new everything
clone(SIGCHLD, 0);

// pthread_create(): shared address space, shared FDs, shared signal handlers
clone(CLONE_VM | CLONE_FILES | CLONE_FS | CLONE_SIGHAND |
      CLONE_THREAD | CLONE_SETTLS, stack_ptr);
```

When you call `pthread_create()`, glibc calls `clone()` with the flags above. The result is a new `task_struct` — its own kernel entity, its own PID, schedulable independently — but with the `mm` pointer pointing to the *same* `mm_struct` as the creating thread. They share one virtual address space.

From the kernel's perspective, threads and processes are the same thing. Both are `task_struct`s. Both get scheduled by the same CFS scheduler. Both appear in `/proc`. The `tgid` field (thread group ID) ties them together: all threads in a process share the same `tgid`, which equals the PID of the first thread.

```bash
# NetworkManager is multi-threaded
$ cat /proc/1618/status | grep -E '(Tgid|Pid|Threads)'
Tgid:   1618
Pid:    1618
Threads:        3

# The threads are visible under /proc/1618/task/
$ ls /proc/1618/task/
1618  1636  1637

# Thread 1636 has its own PID but the same TGID
$ cat /proc/1636/status | grep -E '(Tgid|Pid)'
Tgid:   1618
Pid:    1636
```

`/proc/1618/task/` lists all threads in the thread group. Thread 1618 is the main thread (its PID equals the TGID); 1636 and 1637 are worker threads with their own PIDs but sharing the same memory space.

This architecture — threads as `clone()`d task_structs — is why Linux doesn't need separate thread scheduling code. The CFS scheduler handles both.

---

## Zombies and Reaping

When a process calls `exit()` (or is killed by a signal), the kernel:

1. Frees all its memory (pages, page tables, VMAs)
2. Closes all open file descriptors
3. Detaches from its thread group
4. Changes `__state` to `EXIT_ZOMBIE`
5. Sends `SIGCHLD` to the parent

The `task_struct` is kept alive in zombie state so the parent can retrieve the exit status. The parent does this by calling `wait()` or `waitpid()`:

```c
int status;
pid_t child = wait(&status);

if (WIFEXITED(status)) {
    printf("exited normally, status %d\n", WEXITSTATUS(status));
} else if (WIFSIGNALED(status)) {
    printf("killed by signal %d\n", WTERMSIG(status));
}
```

Once `wait()` returns, the kernel removes the zombie's `task_struct`. This is called **reaping** the zombie.

### What if the parent never calls wait()?

If a parent exits without reaping its children, those children are **reparented** to PID 1 (systemd). systemd calls `waitpid()` on any process reparented to it, so they get reaped promptly. This is one of the reasons PID 1 must never crash — if it did, unreapable zombies would accumulate until the system ran out of PID space.

If the parent stays alive but never calls `wait()`, zombies accumulate. Each zombie is a `task_struct` still consuming kernel memory:

```bash
# Find zombie processes (on a healthy system, there should be none):
$ ps aux | awk '$8 == "Z"'

# The PPID points to the process that should be reaping:
$ ps -eo pid,ppid,stat,comm | awk '$3 ~ /Z/'
```

---

## Signals: Asynchronous Notifications

Signals are the kernel's mechanism for asynchronous notification. A signal can be sent by the kernel (e.g., `SIGSEGV` on a segfault, `SIGCHLD` when a child exits), by another process (`kill -9 1234`), or by the user (Ctrl+C sends `SIGINT`).

### How signals are delivered

When a signal is sent to a process:

1. The kernel sets a bit in the process's **pending signal bitmask** (`task_struct → pending`)
2. The next time the process is scheduled or returns from a syscall, the kernel checks for pending signals
3. If a signal is pending and not blocked, the kernel delivers it

Delivery means one of:
- **Default action**: terminate, terminate+core dump, ignore, stop, or continue — depending on the signal
- **Custom handler**: if the process called `sigaction()` to install a handler, the kernel redirects execution to it

The common signals:

| Signal | Default | Triggered by |
|--------|---------|--------------|
| `SIGHUP` | Terminate | Controlling terminal closed |
| `SIGINT` | Terminate | Ctrl+C |
| `SIGQUIT` | Core dump | Ctrl+\ |
| `SIGKILL` | Terminate | Cannot be caught or ignored |
| `SIGSEGV` | Core dump | Invalid memory access |
| `SIGTERM` | Terminate | Polite termination request |
| `SIGCHLD` | Ignore | Child exited or stopped |
| `SIGSTOP` | Stop | Cannot be caught or ignored |
| `SIGCONT` | Continue | Resume a stopped process |
| `SIGPIPE` | Terminate | Write to a closed pipe |

### SIGKILL and SIGSTOP cannot be caught

These two signals are special: no process can install a handler for them, block them, or ignore them. This gives the system a guaranteed way to kill or stop any process regardless of its state or what code it's running.

This is also why killing a D-state process doesn't work: `SIGKILL` is queued (the bit is set in the pending bitmask), but the process is in uninterruptible sleep and won't check pending signals until the kernel operation it's waiting for completes.

### Sending signals

```bash
# By PID:
kill -SIGTERM 1618      # ask NetworkManager to stop gracefully
kill -SIGKILL 1618      # force it

# By name (resolves PID automatically):
pkill NetworkManager
killall -SIGTERM bash

# Check what a signal number means:
kill -l 9               # prints KILL
```

### Inspecting signal state

```bash
# What signals does NetworkManager have blocked/ignored?
$ grep -E '^Sig' /proc/1618/status
SigQ:   0/59282
SigPnd: 0000000000000000
SigBlk: 0000000000001000
SigIgn: 0000000000003003
SigCgt: 0000000000010002
```

These are 64-bit bitmasks, one bit per signal. `SigBlk` (blocked) and `SigIgn` (ignored) are set by the process through `sigprocmask()` and `sigaction()`. You can decode them:

```bash
$ python3 -c "
blocked = 0x0000000000001000
for i in range(64):
    if blocked & (1 << i):
        print(f'  signal {i+1} is blocked')
"
  signal 13 is blocked    # SIGPIPE — NetworkManager ignores broken pipes
```

---

## Try It Yourself

### 1. Explore the full process tree

```bash
pstree -p | less
```

### 2. Inspect any process's complete state

```bash
cat /proc/1618/status
cat /proc/1618/cmdline | tr '\0' ' '
ls -la /proc/1618/fd/
cat /proc/1618/maps | head -20
```

### 3. Watch fork() and exec() at the syscall level

```bash
strace -e clone,execve,wait4 bash -c 'ls /tmp' 2>&1 | head -20
```

### 4. Observe copy-on-write in action

```bash
# Fork a process that holds 50MB but doesn't write anything
python3 -c "
import os, time
data = bytearray(50 * 1024 * 1024)
pid = os.fork()
if pid == 0:
    print('child PID:', os.getpid())
    time.sleep(30)
else:
    print('parent PID:', os.getpid())
    time.sleep(30)
" &

# While it sleeps, check VmRSS and RssAnon for both PIDs:
# Both will show ~50MB VmRSS, but they share the physical frames
grep -E '(VmRSS|RssAnon)' /proc/$BGPID/status
```

### 5. Create a zombie intentionally

```bash
python3 -c "
import os, time
pid = os.fork()
if pid == 0:
    os._exit(0)      # child exits immediately
else:
    time.sleep(30)   # parent doesn't call wait()
" &

ps aux | awk '$8 == "Z"'   # zombie appears here
```

### 6. Explore threads of a multi-threaded process

```bash
ls /proc/1618/task/            # one directory per thread
ps -eLf | grep 1618            # threads in ps format (LWP column)
```

### 7. Observe process state transitions

```bash
sleep 100 &
BGPID=$!

ps -p $BGPID -o pid,stat       # S (interruptible sleep)

kill -SIGSTOP $BGPID
ps -p $BGPID -o pid,stat       # T (stopped)

kill -SIGCONT $BGPID
ps -p $BGPID -o pid,stat       # S (sleeping again)

kill $BGPID
```

### 8. Decode your shell's signal masks

```bash
grep Sig /proc/$$/status
```

### 9. Trace the fork-exec chain for any command

```bash
strace -f -e clone,execve bash -c 'cat /proc/version' 2>&1 | grep -E '(clone|exec)'
```

### 10. Find D-state processes

```bash
ps aux | awk '$8 ~ /^D/'
# Usually empty; if not, check which device/mount is involved
```

---

## Putting It All Together

The Linux process model is built on one abstraction: everything is a `task_struct`.

```
Everything is a task_struct
    │
    ├── Created via clone() (fork = clone with no sharing)
    │   ├── New task_struct allocated
    │   ├── Virtual address space duplicated — CoW, no actual copy
    │   ├── File descriptor table copied
    │   └── Child placed on run queue; both parent and child are runnable
    │
    ├── Program replaced via execve()
    │   ├── Old address space flushed
    │   ├── New ELF mapped into memory
    │   └── PID unchanged — same task_struct, new mm_struct
    │
    ├── State tracked in task_struct.__state
    │   R (runnable) → S (waiting) ↔ D (uninterruptible) → Z (zombie) → reaped
    │
    ├── Threads = clone() with CLONE_VM
    │   ├── Same mm_struct, separate task_struct
    │   └── Same scheduler, same /proc, different PID — same TGID
    │
    └── Exits → zombie → parent calls wait() → reaped
        If parent dies first → reparented to PID 1 → systemd reaps it
```

Every shell command follows this path: bash calls `fork()`, the child sets up redirections, calls `execve()`, the program runs, exits, and bash reaps it with `wait()`. Pipes, redirections, job control, and process groups all fall out of this single model — composable primitives with no hidden machinery.

---

## What's Next

In the next chapter, we go into **memory** in depth. We've touched on virtual memory — VMAs, page tables, copy-on-write — but only as much as the process model required. Chapter 4 traces the full story: how the kernel manages physical memory, what happens on a page fault, how the page cache works, and why `free` shows almost no "free" memory on a healthy, idle system.

---

*Part of the [Linux Deep Dive](./README.md) series.*
