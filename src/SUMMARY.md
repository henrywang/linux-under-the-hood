# Linux Under the Hood

[Introduction](introduction.md)

---

# Part I — The System Starts

- [The Boot Sequence](01-boot-sequence.md)
- [systemd Deep Dive](02-systemd.md)

# Part II — Processes & Memory

- [Processes: fork, exec, and the Process Table](03-processes.md)
- [Memory Management: Virtual Memory and the Page Cache](04-memory.md)
- [The Scheduler: How the Kernel Decides What Runs](05-scheduler.md)

# Part III — Filesystem & I/O

- [The VFS Layer: How Linux Abstracts Filesystems](06-vfs.md)
- [Btrfs: Snapshots, Subvolumes, and CoW](07-btrfs.md)
- [Block I/O: From read() to the Disk](08-block-io.md)

# Part IV — Image-Based Systems

- [composefs: Composing Read-Only Filesystems](09-composefs.md)
- [ostree: Immutable Trees and Atomic Updates](10-ostree.md)
- [rpm-ostree: Layering Packages on an Immutable Base](11-rpm-ostree.md)
- [bootc: The OS as a Container Image](12-bootc.md)

# Part V — Networking & Security

- [NetworkManager: From Interface Detection to IP Address](13-networkmanager.md)
