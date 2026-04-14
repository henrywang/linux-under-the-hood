# Linux Under the Hood

A developer's deep dive into Linux internals — from boot sequence to kernel scheduler, memory management, filesystems, networking, and containers.

**Read the book:** https://henrywang.github.io/linux-under-the-hood

## Contents

| # | Chapter | Topics |
|---|---------|--------|
| 1 | [The Boot Sequence](src/01-boot-sequence.md) | UEFI, GRUB2, kernel init, initramfs, systemd |
| 2 | Processes & Scheduling | task_struct, fork/exec, CFS |
| 3 | Memory Management | Virtual memory, paging, mmap, OOM killer |
| 4 | The Filesystem | VFS, ext4/Btrfs internals, /proc and /sys |
| 5 | System Calls & I/O | The syscall mechanism, io_uring |
| 6 | The Network Stack | TCP/IP in the kernel, eBPF |
| 7 | Security & Containers | Namespaces, cgroups, how Docker works |

## Building locally

Requires [mdBook](https://rust-lang.github.io/mdBook/):

```bash
cargo install mdbook
mdbook serve --open
```

## License

The book prose is licensed under [CC BY 4.0](LICENSE-CC).
Code examples are licensed under [MIT](LICENSE-MIT).
