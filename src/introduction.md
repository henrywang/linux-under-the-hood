# Linux Under the Hood

Most developers use Linux every day — as their workstation, their servers, their containers. But the system beneath the shell prompt remains a black box.

This book changes that.

*Linux Under the Hood* is a deep-dive series for developers who want to understand what Linux is actually doing: how it boots, how it runs your programs, how it manages memory, how the filesystem works, how the network stack processes your packets, and how security primitives like containers actually work under the hood.

Each chapter combines:
- **Narrative explanation** — the why before the how
- **Hands-on experiments** — real commands you can run on your own Fedora system
- **Source-level details** — pointers into the kernel source when it matters

No prior kernel development experience required. If you can write a program and deploy it on Linux, you're ready.

## How to read this book

The chapters are designed to be read in order — each one builds on the last. But if you already understand processes, you can skip ahead to memory management without being lost.

All examples are tested on **Fedora 43**, kernel **6.19**. The concepts apply to any Linux distribution; the specific commands and paths may vary slightly.

## The chapters

| # | Chapter | What you'll learn |
|---|---------|-------------------|
| 1 | [The Boot Sequence](01-boot-sequence.md) | UEFI → GRUB2 → kernel → initramfs → systemd |
| 2 | Processes & Scheduling | task_struct, fork/exec, the CFS scheduler |
| 3 | Memory Management | Virtual memory, paging, mmap, OOM killer |
| 4 | The Filesystem | VFS, ext4/Btrfs internals, /proc and /sys |
| 5 | System Calls & I/O | The syscall mechanism, io_uring |
| 6 | The Network Stack | TCP/IP in the kernel, eBPF |
| 7 | Security & Containers | Namespaces, cgroups, how Docker works |

Let's start at the very beginning — the moment you press the power button.
