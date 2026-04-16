# Linux Deep Dive #2: systemd — The Init System and Beyond

*Target: Fedora 43, kernel 6.19.11. Every command in this post is something you can run yourself.*

---

The boot sequence chapter ended with systemd becoming PID 1. But that was just the handoff. What does systemd actually *do* once it's in charge?

The short answer: everything. systemd is not just an init system — it's a suite of components that manages services, mounts filesystems, handles logging, schedules jobs, tracks login sessions, and enforces resource limits. Understanding it is understanding how a modern Linux system is organized.

---

## Why PID 1 Is Special

Before diving into systemd, it's worth understanding why PID 1 matters at all.

In Linux, every process has a parent. When a parent process dies, its children are **reparented** — adopted by PID 1. This makes PID 1 the universal parent of all orphaned processes. It's also the only process that cannot be killed with `SIGKILL`. If PID 1 dies, the kernel panics. The whole system depends on it staying alive and responsive.

The traditional Unix init (SysV init) was simple: a shell script runner that started services sequentially from `/etc/rc.d/`. Each service started only after the previous one finished. On modern hardware with many services, this was slow. It also had no standard way to track whether a service was actually running, restart it if it crashed, or collect its logs.

systemd was designed to fix all of this.

---

## The Unit: systemd's Fundamental Building Block

Everything in systemd is described by a **unit file** — a declarative INI-style text file that tells systemd what something is, what it needs, and how to manage it.

### Anatomy of a unit file

Here's the real unit file for `sshd` on this Fedora machine:

```ini
[Unit]
Description=OpenSSH server daemon
Documentation=man:sshd(8) man:sshd_config(5)
After=network.target sshd-keygen.target
Wants=sshd-keygen.target

[Service]
Type=notify
EnvironmentFile=-/etc/sysconfig/sshd
ExecStart=/usr/sbin/sshd -D $OPTIONS
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
```

Every unit file has the same three-section structure:

**`[Unit]`** — metadata and dependencies. The most important directives:

| Directive | Meaning |
|-----------|---------|
| `Description=` | Human-readable name shown in `systemctl status` |
| `After=` | Start this unit only *after* these units are active |
| `Before=` | Start this unit *before* these units |
| `Wants=` | Weak dependency: try to start these, but don't fail if they don't start |
| `Requires=` | Hard dependency: if these fail, this unit fails too |
| `BindsTo=` | Like `Requires=`, but also stop this unit if the dependency stops |

**`[Service]`** (or `[Timer]`, `[Socket]`, etc.) — how to run it:

| Directive | Meaning |
|-----------|---------|
| `Type=notify` | The process signals systemd when it's fully ready (via `sd_notify()`) |
| `ExecStart=` | The command to run |
| `ExecReload=` | How to reload config without restarting |
| `Restart=on-failure` | Automatically restart if the process exits with a non-zero status |
| `RestartSec=42s` | Wait 42 seconds before restarting |
| `KillMode=process` | Only kill the main process on stop, not its children |

**`[Install]`** — when to activate this unit:

| Directive | Meaning |
|-----------|---------|
| `WantedBy=multi-user.target` | Enable this unit when `multi-user.target` is enabled |

The `[Install]` section only matters for `systemctl enable` — it's how systemd knows which target to hook a unit into.

### Where unit files live

systemd loads unit files from several locations, in priority order:

```
/etc/systemd/system/        ← administrator overrides (highest priority)
/run/systemd/system/        ← runtime-generated units
/usr/lib/systemd/system/    ← vendor/package-provided units (lowest priority)
```

If you want to override a vendor unit, don't edit `/usr/lib/` directly — create a **drop-in** file:

```bash
# Create a drop-in directory for the sshd service
mkdir -p /etc/systemd/system/sshd.service.d/

# Write an override — only the directives you want to change
cat > /etc/systemd/system/sshd.service.d/override.conf << 'EOF'
[Service]
RestartSec=10s
EOF

# Tell systemd to reload its config
systemctl daemon-reload
```

Drop-ins are merged with the original unit file. Your changes survive package updates.

---

## Unit Types

systemd has several unit types, each with its own suffix and section:

| Type | Suffix | Purpose |
|------|--------|---------|
| Service | `.service` | A daemon or one-shot process |
| Socket | `.socket` | A socket for socket activation |
| Timer | `.timer` | A scheduled job (cron replacement) |
| Mount | `.mount` | A filesystem mount point |
| Device | `.device` | A kernel device (auto-generated from udev) |
| Target | `.target` | A synchronization milestone |
| Path | `.path` | Trigger a service when a path changes |
| Slice | `.slice` | A cgroup resource boundary |
| Scope | `.scope` | A group of externally-started processes |

---

## Dependencies and Targets

### The dependency graph

When systemd starts, it doesn't just run services one by one. It builds a **dependency graph** from all unit files, then walks the graph in parallel — starting everything it can simultaneously, waiting only where ordering (`After=`/`Before=`) requires it.

The key insight: `Wants=`/`Requires=` express *what* must exist, while `After=`/`Before=` express *when* to start it. These are orthogonal. A unit can require another without ordering itself after it (they'd start in parallel), or be ordered after something it doesn't strictly require.

### Targets as milestones

**Targets** are units with no `[Service]` section — they're pure synchronization points. Other units declare `WantedBy=some.target`, and systemd activates those units when pulling in that target.

This replaces the old SysV concept of **runlevels** (single-user, multi-user, graphical, etc.). The mapping:

| SysV runlevel | systemd target |
|---------------|----------------|
| 1 | `rescue.target` |
| 3 | `multi-user.target` |
| 5 | `graphical.target` |
| 6 | `reboot.target` |

On this machine, `graphical.target` is the default — it pulls in `multi-user.target`, which pulls in hundreds of other units:

```bash
$ systemctl get-default
graphical.target

$ systemctl list-dependencies graphical.target | head -20
graphical.target
● ├─accounts-daemon.service
● ├─gdm.service
● ├─rtkit-daemon.service
● ├─systemd-update-utmp-runlevel.service
● └─multi-user.target
●   ├─atd.service
●   ├─auditd.service
●   ├─avahi-daemon.service
●   ├─chronyd.service
●   ├─crond.service
●   ├─dbus-broker.service
...
```

To switch to a different target temporarily (without rebooting):

```bash
# Drop to multi-user (no graphical session)
systemctl isolate multi-user.target

# Enter emergency shell
systemctl isolate emergency.target
```

---

## Service Lifecycle

A service unit can be in several states. The full lifecycle:

```
inactive
    │
    │  systemctl start
    ▼
activating   ← ExecStartPre= commands running
    │
    │  ExecStart= process running, waiting for ready signal
    ▼
active (running)
    │
    │  systemctl stop  OR  process exits unexpectedly
    ▼
deactivating  ← ExecStopPost= commands running
    │
    ▼
inactive  (or → failed if it crashed and Restart= doesn't apply)
```

You can inspect this at any time:

```bash
$ systemctl status NetworkManager.service
● NetworkManager.service - Network Manager
     Loaded: loaded (/usr/lib/systemd/system/NetworkManager.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-04-16 13:44:57 CST; 1h 20min ago
   Main PID: 1618 (NetworkManager)
      Tasks: 4 (limit: 37682)
     Memory: 12.2M
        CPU: 1.107s
     CGroup: /system.slice/NetworkManager.service
             └─1618 /usr/bin/NetworkManager --no-daemon
```

The key fields: `Loaded` (was the unit file found and parsed?), `Active` (is it running?), `Main PID` (what process is it?), `CGroup` (where in the cgroup hierarchy does it live?).

### Service types

The `Type=` directive controls how systemd knows a service is "ready":

| Type | How systemd knows it's ready |
|------|------------------------------|
| `simple` | Immediately after `ExecStart=` forks (default) |
| `notify` | The process calls `sd_notify(READY=1)` |
| `dbus` | The process acquires a D-Bus name |
| `forking` | The process forks and the parent exits (old-style daemons) |
| `oneshot` | The process runs to completion and exits |
| `idle` | Like `simple`, but delays start until all jobs are dispatched |

`notify` is preferred for modern daemons — it means systemd actually *knows* the service is ready to accept connections, not just that it started.

---

## Socket Activation

One of systemd's most useful features: **socket activation**. Instead of starting a service immediately and waiting for it to be ready, systemd can:

1. Create and bind the socket *before* the service starts (instant)
2. Queue any incoming connections
3. Start the service only when something actually connects
4. Hand the socket's file descriptor to the service process

From a client's perspective, the socket is always there. The service only actually starts when needed.

The D-Bus socket is the canonical example. Here's its socket unit:

```ini
# /usr/lib/systemd/system/dbus.socket
[Unit]
Description=D-Bus System Message Bus Socket

[Socket]
ListenStream=/run/dbus/system_bus_socket

[Install]
WantedBy=sockets.target
```

systemd creates `/run/dbus/system_bus_socket` at boot (part of `sockets.target`, which is reached very early). The actual `dbus-broker.service` only starts when something tries to connect. This is why D-Bus is available the moment any service needs it, without serializing on `dbus-broker.service` startup.

Socket activation also enables **parallel startup without races**: two services that both need D-Bus can start simultaneously, because they'll both see the socket immediately and block until dbus-broker is ready.

---

## Timers: The cron Replacement

systemd timers replace `cron` for scheduling recurring jobs. They have two parts: a `.timer` unit (the schedule) and a matching `.service` unit (what to run).

Here's a real timer on this system:

```bash
$ systemctl list-timers
UNIT                         NEXT                            LEFT
dnf-makecache.timer          Thu 2026-04-16 22:12:09 CST     7h left
fstrim.timer                 Mon 2026-04-20 00:00:00 CST     3 days left
logrotate.timer              Thu 2026-04-17 00:00:00 CST     9h left
plocate-updatedb.timer       Thu 2026-04-17 00:00:00 CST     9h left
systemd-tmpfiles-clean.timer Thu 2026-04-17 00:23:36 CST     9h left
```

The `systemd-tmpfiles-clean.timer` unit:

```ini
[Unit]
Description=Daily Cleanup of Temporary Directories

[Timer]
OnBootSec=15min
OnUnitActiveSec=1d
```

`OnBootSec=15min` — run 15 minutes after boot. `OnUnitActiveSec=1d` — run again every 24 hours after that. No crontab syntax, no minute/hour/day columns to memorize.

Timer directives:

| Directive | Meaning |
|-----------|---------|
| `OnBootSec=` | Time after system boot |
| `OnUnitActiveSec=` | Time after this timer last activated |
| `OnCalendar=` | A calendar expression (like cron, but readable: `daily`, `weekly`, `Mon *-*-* 02:00:00`) |
| `Persistent=true` | If the timer was missed (system was off), run immediately on next boot |

The advantage over cron: the corresponding `.service` unit gets full systemd treatment — logging to the journal, resource limits, failure tracking.

---

## journald: Structured Logging

Traditional syslog wrote plain text to files. systemd-journald writes **structured, binary logs** with metadata attached to every message:

- Timestamp (microsecond precision)
- Unit name
- PID, UID, GID
- SELinux context
- Kernel boot ID (so you can distinguish logs from different boots)

```bash
# All logs from this boot
journalctl -b 0

# Logs from last boot
journalctl -b -1

# Follow logs in real time (like tail -f)
journalctl -f

# Only from one service
journalctl -u NetworkManager.service

# Only errors and worse
journalctl -p err

# Since a specific time
journalctl --since "2026-04-16 13:00:00"

# Kernel messages only (like dmesg)
journalctl -k
```

The journal is queryable in ways plain text files aren't. You can filter by any metadata field:

```bash
# All messages from PID 1618
journalctl _PID=1618

# All messages with a specific syslog identifier
journalctl SYSLOG_IDENTIFIER=NetworkManager
```

Journal files live in `/var/log/journal/` (persistent across reboots) or `/run/log/journal/` (volatile, lost on reboot). On this machine:

```bash
$ journalctl --disk-usage
Archived and active journals take up 264.0M in the file system.
```

---

## cgroups: Resource Isolation

Every service systemd manages runs inside a **cgroup** (control group) — a kernel mechanism for grouping processes and enforcing resource limits.

You can see the cgroup hierarchy right now:

```bash
$ systemd-cgls
CGroup /:
-.slice
├─system.slice
│ ├─NetworkManager.service
│ │ └─1618 /usr/bin/NetworkManager --no-daemon
│ ├─dbus-broker.service
│ │ └─1432 /usr/bin/dbus-broker ...
│ └─...
├─user.slice
│ └─user-1000.slice
│   └─user@1000.service
│     └─...
└─init.scope
  └─1 /usr/lib/systemd/systemd
```

The tree reflects the systemd unit hierarchy:
- `system.slice` — all system services
- `user.slice` — all user sessions
- `init.scope` — PID 1 itself

Because each service has its own cgroup, systemd can:
- **Track all processes in a service**, even ones the main process forked — no more daemons escaping their service
- **Kill a service cleanly** — `SIGTERM` then `SIGKILL` to every process in the cgroup, not just the main PID
- **Enforce resource limits** set in the unit file

You can add resource limits directly in a unit's `[Service]` section:

```ini
[Service]
MemoryMax=512M
CPUQuota=25%
TasksMax=100
```

This tells the kernel: this service gets at most 512MB RAM, 25% of one CPU core, and 100 processes/threads. No external tooling needed.

---

## Try It Yourself

### 1. Inspect any service

```bash
systemctl status <service-name>

# Examples:
systemctl status NetworkManager.service
systemctl status gdm.service
```

### 2. Follow what systemd is doing right now

```bash
journalctl -f
```

### 3. See all running services and their memory/CPU

```bash
systemctl list-units --type=service --state=running
```

### 4. Find the slowest services to start

```bash
systemd-analyze blame
```

### 5. Trace why a service started

```bash
systemd-analyze critical-chain <service>
# Example:
systemd-analyze critical-chain NetworkManager.service
```

### 6. List all active timers and when they next fire

```bash
systemctl list-timers
```

### 7. Read a unit file

```bash
systemctl cat sshd.service
# Shows the file as loaded (including drop-ins)
```

### 8. Create a drop-in override

```bash
systemctl edit sshd.service
# Opens $EDITOR with the right drop-in path pre-created
```

### 9. See the full cgroup tree

```bash
systemd-cgls
```

### 10. Query structured journal fields

```bash
# List all journal fields you can filter on
journalctl -F _SYSTEMD_UNIT

# All log entries from NetworkManager, only errors
journalctl -u NetworkManager.service -p err
```

---

## Putting It All Together

systemd's design reflects a single philosophy: **everything is a unit, everything is tracked, everything is declared**. Rather than shell scripts with implicit ordering, you write unit files that explicitly state dependencies. Rather than ad-hoc logging, every process writes to the journal with metadata. Rather than hoping services stay running, you declare `Restart=on-failure` and systemd handles it.

```
systemd (PID 1)
    │
    ├── reads all unit files from /usr/lib/systemd/system/ and /etc/systemd/system/
    │
    ├── builds dependency graph (Wants/Requires + After/Before)
    │
    ├── activates sockets early (socket activation)
    │
    ├── walks graph in parallel toward default.target
    │   ├── sysinit.target   → mounts, udev, journal, crypto
    │   ├── basic.target     → sockets, timers, paths
    │   ├── multi-user.target → all system services
    │   └── graphical.target → display manager
    │
    ├── places each service in its own cgroup
    │
    ├── collects all stdout/stderr into journald
    │
    └── monitors services, restarts on failure
```

The next chapter goes one level deeper: what is a process, what does the kernel actually store for each one, and how does `fork()` work under the hood?

---

*Part of the [Linux Deep Dive](./README.md) series.*
