# Linux Deep Dive #1: The Boot Sequence — From Power Button to Shell Prompt

*Target: Fedora 43, kernel 6.19.11. Every command in this post is something you can run yourself.*

---

You press the power button. About 10–30 seconds later, a shell prompt appears. What happened in between?

Most developers treat this as a black box. That's a shame — the Linux boot sequence is one of the most elegant pieces of engineering in the entire system. It's a relay race where each stage does just enough work to hand off to the next. Miss any baton and the system halts.

This post tears the black box open. We'll trace every handoff from firmware to your first interactive shell, using a real Fedora 43 machine as the specimen.

---

## The Relay Race at a Glance

```
Power on
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 1 — UEFI Firmware                       ~6s          │
│  POST → find ESP → load shim.efi → load grubx64.efi        │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 2 — GRUB2 Bootloader                    ~3s          │
│  Read grub.cfg → show menu → load vmlinuz + initramfs       │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 3 — Kernel Initialization               ~2s          │
│  Decompress → setup memory → init subsystems → mount initrd │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 4 — initramfs                           ~15s         │
│  Unlock LUKS → find root filesystem → switch_root           │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 5 — systemd (PID 1)                     ~12s         │
│  Parse units → mount filesystems → start services           │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 6 — Login / Shell                                    │
│  getty → login → PAM → your shell                           │
└─────────────────────────────────────────────────────────────┘
```

Those phase timings aren't invented — they come directly from `systemd-analyze` on this machine:

```
$ systemd-analyze
Startup finished in 5.884s (firmware) + 2.696s (loader) + 1.614s (kernel)
                 + 15.109s (initrd) + 12.194s (userspace) = 37.501s
```

How does `systemd-analyze` know these numbers? Each phase boundary is a measured handoff, not an estimate:

- **Kernel phase**: When the kernel finishes its initialization and mounts the initramfs as the root filesystem, it records a timestamp to the monotonic clock. This marks the end of the "kernel" phase and the start of the "initrd" phase.
- **initrd and userspace phases**: When systemd starts running (first inside initramfs, then as the real PID 1 after `switch_root`), it records its own timestamps. The difference between these timestamps gives you the initrd and userspace durations.

We'll use this data throughout.

---

## Phase 1 — UEFI Firmware

### What BIOS used to do

The old BIOS (Basic Input/Output System) was a 16-bit program burned into a chip. At power-on it ran **POST** (Power-On Self Test) — checking RAM, CPU, devices — then looked for a bootable disk by reading the first 512 bytes (the **Master Boot Record**). The first 446 bytes were executable bootloader code, 64 bytes were a partition table, and the last 2 bytes were a magic signature `0x55 0xAA`. This is why old bootloaders had to fit in 446 bytes. Absurd.

Modern systems use **UEFI** (Unified Extensible Firmware Interface), which is essentially a small operating system. It understands GPT partition tables, reads FAT32 filesystems, and can load proper `.efi` executables. No 446-byte constraint.

### The EFI System Partition (ESP)

UEFI stores bootloaders in a dedicated FAT32 partition called the **EFI System Partition (ESP)**. On this Fedora machine:

```
$ efibootmgr -v
BootCurrent: 0005
BootOrder: 0005,0004,0002,0001,0000,0003,0006,0007

Boot0005* Fedora  HD(1,GPT,e7f9e70d-...)/\EFI\FEDORA\SHIM.EFI
Boot0001  Fedora  HD(1,GPT,e7f9e70d-...)/\EFI\FEDORA\SHIMX64.EFI
Boot0000  Windows Boot Manager  ...
```

`BootCurrent: 0005` — we booted entry 0005, which points to `\EFI\FEDORA\SHIM.EFI` on partition 1 (the ESP).

The UEFI firmware reads this table, opens the ESP, loads `SHIM.EFI`, and jumps to it.

### Secure Boot and the Shim

Why `SHIM.EFI` and not `grubx64.efi` directly?

**Secure Boot** is a UEFI feature that refuses to execute unsigned binaries. Only code signed by a key in the firmware's database is allowed to run. Microsoft controls the keys in most consumer firmware — which creates a problem for Linux distributions: they can't ship a GRUB2 signed by Microsoft for every distro release.

The solution is a **shim**: a tiny, Microsoft-signed binary whose only job is to load another bootloader after verifying it against a second key database — one that Red Hat (or Canonical, or SUSE) controls.

```
UEFI firmware
    │
    │  verifies against Microsoft key database
    ▼
shim.efi          ← signed by Microsoft
    │
    │  verifies against Red Hat key database
    ▼
grubx64.efi       ← signed by Red Hat
    │
    │  verifies against Red Hat key database
    ▼
vmlinuz            ← signed by Red Hat
```

Without Secure Boot enabled, UEFI loads `grubx64.efi` directly — the shim is unnecessary and the chain is one step shorter.

You can inspect the ESP yourself (you'll need root):

```bash
# The ESP is mounted at /boot/efi
ls /boot/efi/EFI/fedora/
# shim.efi  shimx64.efi  grubx64.efi  grub.cfg  ...
```

**`/boot` vs `/boot/efi` — a common point of confusion.** These are two different things. `/boot` is a regular directory (or sometimes its own partition) on your root filesystem — it holds kernels, initramfs images, and GRUB config files, typically on ext4 or Btrfs. `/boot/efi` is where the ESP is *mounted* — it's a separate FAT32 partition (FAT32 is required by the UEFI spec) that contains the `.efi` bootloader binaries. So when you see `/boot/efi/EFI/fedora/shim.efi`, you're looking at a file on the FAT32 ESP partition, not on your root filesystem.

Most distros (Fedora, Debian, Ubuntu) mount the ESP at `/boot/efi`. Arch Linux often mounts the ESP directly at `/boot`, which works fine for single-boot setups. If you're dual-booting or sharing a machine with a Debian-based distro, use `/boot/efi` as the ESP mount point to avoid conflicts — different distros have different expectations about what lives in `/boot`.

### What UEFI hands to GRUB

UEFI doesn't just load GRUB and forget it. UEFI hands control to GRUB by locating and executing the `grubx64.efi` application (or `shimx64.efi` for Secure Boot) stored on the FAT32-formatted ESP. It passes a **handoff structure** containing:
- The memory map (what physical RAM exists and which regions are usable)
- A pointer to the UEFI runtime services (which the kernel uses later for things like `efivarfs`)
- ACPI tables

Once GRUB is running, UEFI is mostly done.

---

## Phase 2 — GRUB2 Bootloader

GRUB2 (GRand Unified Bootloader version 2) runs as a UEFI application. Its job: find the kernel and initramfs on disk, load them into RAM, set up the kernel command line, and jump to the kernel entry point.

### Reading the config

GRUB2 reads `/boot/grub2/grub.cfg`. On Fedora this file is auto-generated by `grub2-mkconfig`. You typically never edit it by hand — instead edit `/etc/default/grub` and regenerate. The config describes menu entries, timeouts, and kernel arguments.

```bash
# See the GRUB environment (saved default, etc.)
sudo grub2-editenv list
# saved_entry=...
# kernelopts=root=UUID=... ro rootflags=subvol=root ...
# boot_success=1
```

### The kernel command line

After loading the kernel, GRUB passes a command line string. You can always inspect what was actually used:

```bash
$ cat /proc/cmdline
BOOT_IMAGE=(hd0,gpt2)/vmlinuz-6.19.11-200.fc43.x86_64
  root=UUID=37e2f5e1-66e7-4a8a-a69e-57bcc9a44af2
  ro
  rootflags=subvol=root
  rd.luks.uuid=luks-5ea459ba-b7ec-439a-a3cf-7d25ff3b2889
  rhgb quiet
```

There's a lot of information here. Let's decode it:

| Parameter                           | Meaning                                                           |
| ----------------------------------- | ----------------------------------------------------------------- |
| `BOOT_IMAGE=(hd0,gpt2)/vmlinuz-...` | The kernel file GRUB loaded, from partition 2 of disk 0           |
| `root=UUID=37e2f5e1-...`            | The root filesystem to mount after boot                           |
| `ro`                                | Mount root read-only initially (fsck can run; remounted rw later) |
| `rootflags=subvol=root`             | Btrfs-specific: mount the `root` subvolume                        |
| `rd.luks.uuid=luks-5ea459ba-...`    | Tell initramfs to unlock a LUKS-encrypted device                  |
| `rhgb`                              | Red Hat Graphical Boot (Plymouth splash screen)                   |
| `quiet`                             | Suppress most kernel messages on console                          |

This one command line tells the whole story of this machine's storage setup: there's a Btrfs filesystem inside a LUKS-encrypted container, and the bootable root is a subvolume within it.

### Who actually reads the command line?

The kernel command line is a single string, but it's consumed by multiple components. You might wonder: how does `rd.luks.uuid` end up in the initramfs while `quiet` goes to the kernel? The answer is a filtering hierarchy:

1. **Known kernel parameters**: If the kernel recognizes the string (`root=`, `ro`, `quiet`, etc.), it uses it to configure itself during `start_kernel()`.

2. **Module parameters**: If the string contains a dot (e.g., `nvidia.modeset=1`), the kernel treats the part before the dot as a module name and passes the value to that module when it loads.

3. **initramfs directives**: Parameters prefixed with `rd.*` are conventions established by dracut. The kernel doesn't interpret them — dracut's scripts inside the initramfs read `/proc/cmdline` and act on the ones they recognize (like `rd.luks.uuid`).

4. **systemd parameters**: systemd also reads `/proc/cmdline` when it starts. Parameters like `systemd.unit=multi-user.target` or `systemd.log_level=debug` let you configure PID 1 from the bootloader.

5. **The unknowns**: Historically, any parameter not recognized by the kernel and not containing a dot was passed to PID 1 as an environment variable. Modern systemd is stricter about this for security reasons — it won't turn arbitrary strings into `$VARIABLES`. If you need to set an environment variable via the command line, use the explicit `systemd.setenv=VAR=VALUE` syntax.

The whole thing works because `/proc/cmdline` is readable by anyone — every component just picks out the parameters it cares about and ignores the rest.

### What GRUB loads

GRUB reads two files from `/boot` and loads them into RAM:

1. **`vmlinuz-6.19.11-200.fc43.x86_64`** — the compressed kernel image
2. **`initramfs-6.19.11-200.fc43.x86_64.img`** — a compressed archive (70MB on this system)

```bash
$ ls -lh /boot/vmlinuz-6.19.11-200.fc43.x86_64
-rwxr-xr-x. 1 root root 18M Apr  2 16:55 /boot/vmlinuz-6.19.11-200.fc43.x86_64

$ ls -lh /boot/initramfs-6.19.11-200.fc43.x86_64.img
-rw-------. 1 root root 70M Apr  2 21:22 /boot/initramfs-6.19.11-200.fc43.x86_64.img
```

Once both are in RAM, GRUB jumps to the kernel's entry point. GRUB is done.

### What about systemd-boot?

GRUB2 isn't the only bootloader in the Linux world. **systemd-boot** (formerly known as gummiboot) is a simpler alternative that's gaining adoption — Arch Linux, some Ubuntu configurations, and Fedora all support it.

The key differences from GRUB2:

- **No scripting language or config generator.** GRUB2 has its own shell, scripting, and `grub2-mkconfig`. systemd-boot has none of that — it reads simple drop-in files directly.
- **Uses the Boot Loader Specification (BLS).** Each kernel gets a small `.conf` file in the ESP (typically under `loader/entries/`) that lists the kernel path, initramfs path, and command line options. Adding a kernel means dropping a file; removing one means deleting it.
- **Lives entirely on the ESP.** GRUB2 reads from both the ESP and `/boot` (a separate partition). systemd-boot reads everything from the ESP's FAT32 filesystem.
- **No theming or interactive shell.** It shows a plain menu and boots. That's it.

This machine uses GRUB2, so that's what we trace in this post. But if you run `bootctl status` and see output instead of an error, your system is using systemd-boot — the handoff to the kernel works the same way, just with less machinery in between.

---

## Phase 3 — Kernel Initialization

### vmlinuz: what's actually in that file

The kernel image isn't a plain ELF binary. It's a **bzImage** — a self-extracting compressed archive:

```bash
$ file /boot/vmlinuz-6.19.11-200.fc43.x86_64
Linux kernel x86 boot executable, bzImage,
  version 6.19.11-200.fc43.x86_64,
  ZST compressed,
  64-bit EFI handoff entry point
```

"bzImage" stands for "big zImage" (the `b` doesn't mean bzip2 — it means "big", as in it can be loaded above 1MB). Modern kernels are compressed with **Zstandard (ZST)** for faster decompression. When the kernel starts executing, the first thing it does is decompress itself into RAM.

```
[ GRUB jumps here ]
vmlinuz (compressed)
    │  arch/x86/boot/header.S  — 16-bit setup code
    │  arch/x86/boot/compressed/head_64.S — decompress kernel
    ▼
vmlinux (uncompressed) loaded into RAM
    │  arch/x86/kernel/head_64.S — 64-bit startup
    │  start_kernel() in init/main.c
    ▼
kernel is running
```

### start_kernel(): the origin of everything

Once decompressed, the kernel calls `start_kernel()` in `init/main.c` — one of the most consequential function calls in all of software. It initializes, in roughly this order:

```
start_kernel()
├── setup_arch()          — CPU detection, parse command line, set up memory map
├── mm_init()             — memory management subsystem
├── sched_init()          — the scheduler (CFS)
├── rcu_init()            — RCU (Read-Copy-Update) — kernel's lock-free data structure mechanism
├── init_IRQ()            — interrupt controllers
├── time_init()           — timekeeping
├── softirq_init()        — deferred interrupt processing
├── console_init()        — printk to the screen
├── rest_init()           — spawn PID 1 and PID 2
```

You can see this happening in real time:

```bash
$ dmesg | head -30
[    0.000000] Linux version 6.19.11-200.fc43.x86_64 (mockbuild@...) gcc 15.2.1
[    0.000000] Command line: BOOT_IMAGE=(hd0,gpt2)/vmlinuz-6.19.11-...
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009ffff] usable
[    0.000000] BIOS-e820: [mem 0x00000000000a0000-0x00000000000fffff] reserved
...
```

The `[    0.000000]` timestamps are in seconds since kernel start. You can watch the kernel build up: memory map, then CPU features, then interrupts, then the scheduler. Every subsystem announces itself.

### The memory map (e820)

The first thing the kernel does is ask UEFI (via the data passed by GRUB): "what physical memory is available?" The answer comes as an **e820 map** (named after BIOS interrupt 0xe820, the ancient API that established the convention):

```bash
$ dmesg | grep -i 'e820\|usable\|reserved' | head -15
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009ffff] usable
[    0.000000] BIOS-e820: [mem 0x00000000000a0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x0000000009bfefff] usable
[    0.000000] BIOS-e820: [mem 0x00000000000a0000-0x00000000000fffff] reserved
...
[    0.000000] BIOS-e820: [mem 0x0000000100000000-0x000000080e2fffff] usable
```

The first 640KB (`0x00000–0x9ffff`) is "usable" — traditional DOS territory. Then `0xa0000–0xfffff` is reserved — the old video RAM and ROM region. Nothing runs there. Above 1MB, RAM is usable again. Above 4GB (`0x100000000`), the rest of RAM is available.

### Spawning PID 1 and PID 2

At the end of `start_kernel()`, `rest_init()` creates two kernel threads:

- **PID 1** — `kernel_init()`: will become the init process (eventually executes `/usr/lib/systemd/systemd`)
- **PID 2** — `kthreadd`: the kernel thread daemon, parent of all other kernel threads

The original `start_kernel()` thread becomes the **idle thread** (PID 0) — it runs whenever the CPU has nothing else to do, executing the `hlt` instruction.

Before PID 1 can exec systemd though, there's an important intermediate step.

---

## Phase 4 — initramfs: The Bootstrap Filesystem

### The chicken-and-egg problem

Here's the problem: the kernel needs to mount the root filesystem to run init. But to mount the root filesystem, you might need:

- Kernel modules for your storage controller (NVMe, SATA, etc.)
- Tools to unlock a LUKS-encrypted device
- Tools to assemble a RAID array or LVM volume
- Scripts to figure out which UUID maps to which `/dev/nvme0n1p3`

Those modules and tools live *on* the root filesystem — but the root filesystem isn't mounted yet. Classic chicken-and-egg.

The solution: **initramfs** (initial RAM filesystem). The kernel bundles a minimal temporary root filesystem, mounts it first, does the preparatory work, then switches to the real root.

### What's in initramfs

The initramfs is a **cpio archive** (a Unix archiving format, like tar) compressed with zstd. It contains a miniature Linux environment:

```bash
$ lsinitrd /boot/initramfs-6.19.11-200.fc43.x86_64.img | head -40
Image: /boot/initramfs-6.19.11-200.fc43.x86_64.img: 70M
========================================================================

# Explore the full contents:
$ lsinitrd /boot/initramfs-6.19.11-200.fc43.x86_64.img | grep -E '\.(ko|service|sh)$' | head -20
```

On this system the initramfs is 70MB — because it contains:
- A copy of systemd (yes, systemd runs *inside* initramfs first)
- LUKS tools (`cryptsetup`) — required because the root fs is encrypted
- Btrfs tools — required to mount the subvolume
- NVMe and storage drivers
- A minimal `/etc`, `/dev`, `/sys`, `/proc`

**dracut** is Fedora's tool for building initramfs images. It takes a modular approach: you declare which "dracut modules" you need, and it assembles the minimal environment for your specific machine.

dracut runs at three points in a system's life:

1. **During OS installation** — the installer calls dracut at the end to build an initramfs tailored to the hardware it just installed onto.
2. **On every kernel update** — when `dnf` installs a new kernel, a post-install hook automatically triggers dracut to build a matching initramfs. Kernel modules are version-specific, so the old initramfs won't work with the new kernel.
3. **Manually, when you change low-level storage configuration** — moving to a different machine, enabling LUKS, or installing early-boot drivers (like Nvidia) requires a fresh initramfs.

```bash
# Rebuild initramfs for the current kernel (run as root):
dracut --force

# See what dracut modules were included:
lsinitrd /boot/initramfs-$(uname -r).img | grep 'dracut module'
```

### How the kernel enters initramfs

After mounting the initramfs as its root filesystem, the kernel executes a single file: **`/init`**. Whatever that file is becomes PID 1.

- In older systems, `/init` was a shell script that sequentially unlocked storage, mounted the real root, and called `pivot_root`.
- In modern dracut-built initramfs images, `/init` is a symlink to a **stripped-down systemd**. This early systemd has one job: get to `/sysroot`. It doesn't start your network, desktop, or user services — just the storage and crypto units needed to mount the real root.

Using systemd here rather than a script enables parallelism: it can simultaneously wait for a USB device to initialize, prompt for a LUKS passphrase, and assemble an LVM/RAID array.

### LUKS unlocking in the initramfs

This machine uses full-disk encryption. Look at the kernel command line again:

```
rd.luks.uuid=luks-5ea459ba-b7ec-439a-a3cf-7d25ff3b2889
```

`rd.*` parameters are **initramfs directives** (the `rd` prefix comes from dracut: "ram disk"). During the initramfs phase, before the real root is mounted, the system must:

1. Find the LUKS container by UUID
2. Prompt you for a passphrase (or use a stored key)
3. Use `cryptsetup luksOpen` to create a decrypted device mapper device
4. Then mount *that* device as root

This is why initramfs took **15 seconds** on this boot — a significant chunk of that is the time for the user to type a passphrase (or for a TPM to unseal a key). On an unencrypted system, initramfs skips the LUKS step entirely, and this phase drops from ~15s to ~2s. The initramfs image is also much smaller since `cryptsetup` and its dependencies aren't needed.

### switch_root: handing off to the real filesystem

Once the real root filesystem is mounted at `/sysroot`, the initramfs systemd performs **`switch_root`** — a three-step operation:

1. **Pivot**: `/sysroot` becomes the new `/`. The old initramfs root is displaced.
2. **Free**: The initramfs is deleted from RAM, reclaiming its memory.
3. **Exec**: The real `/usr/lib/systemd/systemd` on disk is exec'd, taking over as PID 1.

The key distinction from the lower-level `pivot_root` syscall: `switch_root` actively frees the initramfs memory and is performed by systemd inside the initramfs — the kernel doesn't do this on its own. Notably, `switch_root` doesn't even call the `pivot_root` syscall under the hood — it uses `mount --move` (`MS_MOVE`) instead, which relocates the mount point without the namespace gymnastics that `pivot_root` requires.

---

## Phase 5 — systemd: The Init System

### PID 1's job

`systemd` is now PID 1. This is significant: PID 1 is special in Linux. It is the parent of all orphaned processes (any process whose parent dies gets reparented to PID 1). It cannot be killed with `SIGKILL`. If it crashes, the kernel panics.

systemd's core job is to **start services in the right order as fast as possible**. It reads **unit files** — declarative descriptions of what to start and what it depends on.

### Units and targets

Everything in systemd is a **unit**. The most common types:

| Unit type | Suffix | Purpose |
|-----------|--------|---------|
| Service | `.service` | A daemon or one-shot process |
| Mount | `.mount` | A filesystem to mount |
| Device | `.device` | A kernel device (auto-created from udev) |
| Socket | `.socket` | A socket to listen on (for socket activation) |
| Target | `.target` | A synchronization point (like a milestone) |

**Targets** are especially important — they replace the old concept of runlevels:

```bash
$ systemctl list-units --type=target --state=active
  UNIT                   LOAD   ACTIVE SUB    DESCRIPTION
  basic.target           loaded active active Basic System
  cryptsetup.target      loaded active active Local Encrypted Volumes
  getty.target           loaded active active Login Prompts
  graphical.target       loaded active active Graphical Interface
  local-fs.target        loaded active active Local File Systems
  multi-user.target      loaded active active Multi-User System
  network.target         loaded active active Network
  sysinit.target         loaded active active System Initialization
```

`graphical.target` is the final destination on a desktop system — it's reached when everything is ready.

### The dependency graph

systemd builds a dependency graph and walks it in parallel. The critical path to `graphical.target` on this machine:

```bash
$ systemd-analyze critical-chain
graphical.target @12.194s
└─multi-user.target @12.194s
  └─plymouth-quit-wait.service @9.496s +2.697s
    └─systemd-user-sessions.service @9.478s +11ms
      └─remote-fs.target @9.460s
        └─network.target @3.455s
          └─wpa_supplicant.service @4.376s +31ms
            └─basic.target @2.218s
              └─dbus-broker.service @2.154s +46ms
                └─sysinit.target @2.133s
                  └─systemd-resolved.service @2.075s +57ms
                    └─local-fs.target @1.927s
                      └─boot-efi.mount @1.907s +19ms
                        └─boot.mount @1.883s +23ms
```

Read this bottom-up: `boot.mount` must succeed before `local-fs.target`, which must complete before `systemd-tmpfiles-setup`, and so on up to `graphical.target`. The number after `@` is when the unit became active; the `+` number is how long it took to start.

### What sysinit.target does

`sysinit.target` is the first major milestone — basic system setup. By the time it's reached:

- `/proc`, `/sys`, `/dev` are all mounted
- `udev` has run and populated `/dev` with device nodes
- The system clock is set
- Cryptographic volumes are open
- `systemd-journald` is running (the journal)

### Socket activation: starting services lazily

One of systemd's clever tricks: **socket activation**. Instead of starting `dbus.service` immediately and waiting for it to be ready, systemd can:

1. Create and bind the D-Bus socket immediately (instant)
2. Queue any messages sent to that socket
3. Start `dbus-broker.service` only when something actually connects
4. Deliver the queued messages once the service is ready

Callers don't see a delay — from their perspective the socket was always there. This is how systemd achieves fast parallel startup: almost everything can start "at the same time" without actually racing.

---

## Phase 6 — Login and Your Shell

### getty and the TTY

Once `multi-user.target` is reached, `systemd` starts **getty** processes. A getty (get tty) opens a terminal device, prints the login prompt, and waits for input.

On a headless system you'd interact with `tty1–tty6`. On a desktop, a display manager (like GDM for GNOME) handles the graphical login instead.

```bash
# See which getty units are running
systemctl status getty@tty1.service
```

### The login chain

When you type your password at a text login:

```
getty (opens /dev/tty1, prints "login: ")
    │
    ▼
login binary
    │  calls PAM (Pluggable Authentication Modules)
    │  PAM checks /etc/passwd, /etc/shadow, optionally LDAP, fingerprint, etc.
    ▼
user session created
    │  PAM runs pam_systemd.so → registers session with systemd-logind
    │  PAM runs pam_env.so → loads environment variables
    │  login reads /etc/profile, sets HOME, SHELL, PATH
    ▼
exec $SHELL   ← your shell, finally
```

Your shell (bash, zsh, fish) sources its startup files (`~/.bashrc`, `~/.zshrc`, etc.) and you get a prompt. The boot is complete.

---

## Try It Yourself

Here are the commands that let you observe the boot sequence from inside a running system:

### 1. How long did each phase take?

```bash
systemd-analyze
# Startup finished in 5.884s (firmware) + 2.696s (loader) + 1.614s (kernel)
#              + 15.109s (initrd) + 12.194s (userspace) = 37.501s
```

### 2. What took the longest to start?

```bash
systemd-analyze blame | head -20
```

### 3. Visualize the dependency graph

```bash
# Generate an SVG of the full boot dependency graph
systemd-analyze plot > boot.svg
# Open with a browser or image viewer
```

### 4. What's in your initramfs?

```bash
lsinitrd /boot/initramfs-$(uname -r).img | less
```

### 5. What did the kernel print during boot?

```bash
# Current boot
dmesg | less

# With timestamps as wall clock time
dmesg -T | less

# Stored in the journal (survives reboots)
journalctl -b 0       # this boot
journalctl -b -1      # previous boot
journalctl -b -1 -p err   # only errors from last boot
```

### 6. What EFI boot entries exist?

```bash
efibootmgr -v
```

### 7. Examine the kernel image itself

```bash
file /boot/vmlinuz-$(uname -r)
# Linux kernel x86 boot executable, bzImage, ZST compressed ...
```

### 8. What was the actual kernel command line?

```bash
cat /proc/cmdline
```

### 9. Trace a single boot unit

```bash
# See exactly when and how a unit started
systemd-analyze critical-chain NetworkManager.service

# See the log output from a unit during boot
journalctl -b -u NetworkManager.service
```

### 10. Profile a specific unit

```bash
systemd-analyze blame | grep -i luks
# If LUKS is slow, this shows up in the blame list
```

---

## Putting It All Together

Let's revisit the original diagram, but now with the full picture filled in:

```
Power on
    │
    ▼  UEFI firmware runs POST
       Reads EFI variable: boot entry 0005 → \EFI\FEDORA\SHIM.EFI
       Loads shim.efi (Microsoft-signed), verifies signature
       shim loads grubx64.efi (Red Hat-signed), verifies signature
    │
    ▼  GRUB2 reads /boot/grub2/grub.cfg
       Presents boot menu (1 second timeout on this machine)
       Loads vmlinuz-6.19.11 and initramfs-6.19.11.img into RAM
       Sets kernel command line
       Calls EFI handoff entry point in vmlinuz
    │
    ▼  Kernel decompresses itself (ZST → vmlinux in RAM)
       Processes e820 memory map from UEFI
       start_kernel(): initializes mm, scheduler, IRQs, console
       Mounts initramfs as root filesystem
       Spawns PID 1 (kernel_init) and PID 2 (kthreadd)
    │
    ▼  initramfs systemd runs
       Finds LUKS container (UUID: 5ea459ba-...)
       Prompts for passphrase / unseals TPM key
       cryptsetup luksOpen → creates /dev/mapper/luks-...
       Mounts Btrfs filesystem, subvolume "root", at /sysroot
       switch_root: /sysroot becomes new /, initramfs freed from RAM
       Exec real /usr/lib/systemd/systemd
    │
    ▼  systemd (PID 1) reads unit files
       Builds dependency graph
       Starts sysinit.target in parallel
       Starts local-fs.target, network.target, ...
       Reaches multi-user.target, then graphical.target
    │
    ▼  GDM (display manager) starts
       OR getty opens /dev/tty1
       login → PAM authentication
       Session registered with systemd-logind
       Exec bash/zsh/fish
    │
    ▼
$ _
```

Every character in that `$ _` represents a chain of decisions made by UEFI, GRUB, the kernel, dracut, and systemd — all working together, most of it in well under a minute.

---

## What's Next

In the next post, we'll go deeper into something `start_kernel()` sets up: **processes**. What is a process really? What does the kernel actually store for each one? How does `fork()` work at the kernel level? And how does the scheduler decide which process runs next?

---

*Part of the [Linux Deep Dive](./README.md) series.*
