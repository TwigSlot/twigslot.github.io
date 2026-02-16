---
title: "Init Systems, Daemon Spawning, and the Performance Cost of Process Management"
date: 2026-02-17
draft: false
tags:
  - performance-engineering
  - linux
  - systems-programming
  - systemd
  - init-systems
---

Most of us type `systemctl start myservice` and move on. The service starts, it runs in the background, it restarts if it crashes. We don't think about what's happening underneath because systemd has made daemon management feel trivial. But it wasn't always this way, and the machinery required to correctly spawn a Unix daemon is surprisingly involved. Understanding it matters if you care about performance — because the init system sits in the critical path of every service lifecycle event, and its design choices ripple into CPU affinity, cgroup placement, socket activation latency, and process scheduling.

This post covers how daemons actually work at the kernel level, how different init systems handle them, and why the systemd debate is more nuanced than either side admits.

---

## The Double Fork: What Daemons Actually Are

A daemon is a process that runs in the background, detached from any controlling terminal. Getting there from a normal forked child is more involved than it sounds. The traditional recipe, codified by W. Richard Stevens in *Advanced Programming in the UNIX Environment*, goes like this:

```c
#include <unistd.h>
#include <stdlib.h>
#include <sys/stat.h>

void daemonize(void) {
    pid_t pid;

    // First fork: exit parent, child continues
    pid = fork();
    if (pid < 0) exit(1);
    if (pid > 0) exit(0);  // Parent exits

    // Create new session — become session leader
    if (setsid() < 0) exit(1);

    // Second fork: exit session leader, grandchild continues
    pid = fork();
    if (pid < 0) exit(1);
    if (pid > 0) exit(0);  // Session leader exits

    // Now we're a daemon:
    // - Not a session leader (can't acquire controlling tty)
    // - No controlling terminal
    // - Orphaned — init adopts us

    umask(0);
    chdir("/");

    // Close all file descriptors
    for (int fd = sysconf(_SC_OPEN_MAX); fd >= 0; fd--)
        close(fd);
}
```

Why do you need *two* forks? The answer lies in how Linux organizes processes into sessions and process groups.

**First fork**: The parent exits. The child is now an orphan, so init (PID 1) adopts it. More importantly, the child is not a process group leader (it has a new PID but inherited the parent's PGID), so it can call `setsid()`.

**`setsid()`**: Creates a new session. The calling process becomes: (1) the session leader, (2) the process group leader, and (3) has no controlling terminal. This is the key step — it detaches from the original terminal session.

**Second fork**: The session leader exits. The grandchild inherits the session but is *not* the session leader. This matters because on System V derivatives, a session leader that opens a terminal device automatically acquires it as a controlling terminal. The second fork prevents this. The grandchild can never accidentally reattach to a tty.

This double-fork dance also handles zombie prevention. The intermediate process exits immediately, so init reaps it. The final daemon process is a direct child of init, which always calls `wait()` on its children.

If you've ever wondered why so many daemons had subtle bugs around terminal handling, signal delivery, or zombie accumulation — this is why. Getting daemonization right requires understanding sessions, process groups, controlling terminals, and the difference between System V and BSD tty semantics. Every step in the sequence exists to handle a specific edge case.

---

## SysVinit: Shell Scripts All the Way Down

The original Linux init system, SysVinit, manages daemons with shell scripts in `/etc/init.d/`. A typical service script looks like:

```bash
#!/bin/sh
case "$1" in
  start)
    echo "Starting mydaemon..."
    /usr/sbin/mydaemon --config /etc/mydaemon.conf &
    echo $! > /var/run/mydaemon.pid
    ;;
  stop)
    kill $(cat /var/run/mydaemon.pid)
    rm -f /var/run/mydaemon.pid
    ;;
  restart)
    $0 stop
    $0 start
    ;;
esac
```

The daemon is expected to daemonize itself (the double fork). The init script just launches it and records the PID. This has several problems:

**PID file races**: Between writing the PID file and the daemon completing initialization, the PID might be recycled. You end up sending `SIGTERM` to an unrelated process. This isn't theoretical — it happened regularly on busy systems.

**No process tracking**: If the daemon forks children, SysVinit has no way to track them. `kill $(cat pidfile)` only kills the main process. Orphaned children linger.

**Sequential startup**: Services start one at a time, in a dependency order encoded by symlink naming (`S01foo`, `S02bar`). On a machine with 50 services, boot time is the sum of all startup times. No parallelism.

**No resource control**: SysVinit predates cgroups. There's no built-in mechanism for CPU affinity, memory limits, or I/O bandwidth control. You'd add `taskset` or `nice` calls to the shell script manually.

**Restart handling**: If a daemon crashes, SysVinit doesn't notice. You'd run a separate watchdog or use `inittab` respawn entries, which have their own quirks.

### Performance implications

From a performance engineering perspective, SysVinit's model is hostile. Shell script interpretation for every service operation adds latency. The sequential boot model wastes parallelism on multi-core machines. The lack of cgroup integration means you can't do meaningful resource isolation — a misbehaving daemon can consume unbounded CPU, starve other services of I/O, or exhaust memory without any init-level intervention.

The PID tracking problem is particularly nasty for performance-sensitive workloads. If your monitoring can't reliably identify which process is your daemon, you can't reliably pin it to cores, assign it to NUMA nodes, or place it in the right cgroup hierarchy.

---

## Upstart: Event-Driven, But Still Daemonizing

Ubuntu's Upstart (2006–2015) introduced event-driven service management. Instead of sequential runlevel scripts, services declared what events they depended on:

```
# /etc/init/mydaemon.conf
start on filesystem and net-device-up IFACE=eth0
stop on shutdown

expect daemon
respawn

exec /usr/sbin/mydaemon
```

The `expect daemon` directive told Upstart that the service would double-fork, so Upstart would track the grandchild PID. This was an improvement — Upstart could actually follow the double fork and know which PID to monitor.

But `expect` was fragile. If you said `expect daemon` and the process only forked once, Upstart would track the wrong PID. If you said `expect fork` and it double-forked, same problem. Getting this wrong produced services that Upstart thought were running when they'd crashed, or services that Upstart kept killing because it was tracking the wrong process.

Upstart's event model did enable some parallelism — services that didn't depend on each other could start concurrently. But the dependency graph was implicit (encoded in event names) rather than explicit, which made reasoning about boot order difficult.

Upstart also lacked cgroup integration in its early versions. CPU pinning, NUMA awareness, and memory limits were still manual.

---

## systemd: Supervision Without the Fork Dance

systemd (2010–present) took a fundamentally different approach: **daemons don't daemonize themselves**. Instead of the double-fork ritual, a service just runs in the foreground:

```ini
[Unit]
Description=My Daemon
After=network.target

[Service]
Type=simple
ExecStart=/usr/sbin/mydaemon --config /etc/mydaemon.conf
Restart=on-failure

CPUAffinity=2 3
MemoryMax=512M
IOWeight=100

[Install]
WantedBy=multi-user.target
```

systemd forks the service process itself and manages it directly. No PID files. No double fork. No shell script wrapper. The service process is a direct child of PID 1 (or more precisely, of the systemd service manager), tracked in a cgroup that systemd creates for it.

### Why this matters for performance

**cgroup-based tracking**: Every service gets its own cgroup. This means systemd can:
- Track all processes spawned by a service, including children and grandchildren
- Kill the entire service tree with a single cgroup operation (no zombie stragglers)
- Apply resource limits (CPU, memory, I/O) at the cgroup level

**CPU affinity**: `CPUAffinity=` directly maps to `sched_setaffinity()`. You can pin services to specific cores without wrapper scripts. For latency-sensitive workloads, this eliminates the scheduler bouncing your process between cores, destroying L1/L2 cache locality.

**NUMA policy**: `NUMAPolicy=` and `NUMAMask=` control memory allocation policy. For databases or in-memory caches, binding to a specific NUMA node can reduce memory access latency by 40–100ns per access (local vs. remote NUMA on a typical dual-socket system).

**Memory limits**: `MemoryMax=` and `MemoryHigh=` use the cgroup memory controller. `MemoryHigh` is a soft limit that triggers reclaim pressure; `MemoryMax` is a hard limit that triggers the OOM killer. This prevents a runaway service from pushing the entire system into swap death.

**I/O control**: `IOWeight=` and `IODeviceWeight=` use the cgroup I/O controller (blk-cgroup). You can ensure your latency-sensitive database gets I/O priority over batch jobs.

**Parallel startup**: systemd builds an explicit dependency graph and starts services in parallel wherever possible. Socket activation (more on this below) further increases parallelism by removing ordering dependencies.

**Socket activation**: systemd can create listening sockets *before* starting the service. The service inherits the socket via file descriptor passing:

```c
#include <systemd/sd-daemon.h>

int main(void) {
    int n = sd_listen_fds(0);
    if (n > 0) {
        int listen_fd = SD_LISTEN_FDS_START;  // fd 3
        // listen_fd is already bound and listening
    }
}
```

This has two performance implications:
1. Services can start on-demand (first connection triggers startup), reducing idle resource usage.
2. Boot parallelism increases because services don't need to wait for their dependencies to finish binding sockets — systemd holds the socket and buffers connections until the service is ready.

### The notify protocol

For services that need initialization time, `Type=notify` lets the service signal readiness:

```c
sd_notify(0, "READY=1");
sd_notify(0, "STATUS=Processing requests...");
sd_notify(0, "MAINPID=1234");
```

This replaces the old pattern of "fork, do setup, signal parent, parent exits" with an explicit readiness notification. systemd doesn't consider the service "up" until it receives `READY=1`, so dependent services wait for actual readiness rather than guessing based on fork timing.

---

## OpenRC: The Middle Ground

Gentoo's OpenRC (and Alpine Linux's default) takes a pragmatic middle path. It uses shell-script service definitions but with better process management than SysVinit:

```bash
#!/sbin/openrc-run

command="/usr/sbin/mydaemon"
command_args="--config /etc/mydaemon.conf"
command_background=true
pidfile="/var/run/mydaemon.pid"

depend() {
    need net
    after firewall
}
```

OpenRC manages the backgrounding itself when `command_background=true` — the daemon doesn't need to double-fork. It uses `start-stop-daemon`, which handles PID tracking more reliably than raw shell scripts.

### What OpenRC lacks

OpenRC doesn't use cgroups by default (though it can optionally integrate with them). This means:
- No automatic process tree tracking
- No built-in CPU affinity or NUMA policy
- No memory or I/O limits without external tools
- Service cleanup on stop relies on PID tracking, not cgroup-based kill

For performance-critical deployments, this means you're back to manually managing `taskset`, `numactl`, `cgroups`, and `ulimit` in your service scripts or wrapper scripts.

OpenRC does support parallel startup and has explicit dependency declarations, so boot time is reasonable. Its footprint is also much smaller than systemd — the init process itself uses minimal memory and CPU, which matters on embedded systems or containers.

---

## runit and s6: Supervision Trees

runit and s6 (by Laurent Bercot) represent the "Unix philosophy" approach to init: small, focused tools composed together.

A runit service is a directory with a `run` script:

```bash
#!/bin/sh
exec /usr/sbin/mydaemon --foreground --config /etc/mydaemon.conf
```

That's it. No daemonizing. The service runs in the foreground, and the supervisor (`runsv`) manages it. If it exits, `runsv` restarts it. Logging is handled by a paired `log/run` script that reads the service's stdout/stderr.

s6 is architecturally similar but more rigorous. It provides:
- `s6-svc` for service control
- `s6-svstat` for status queries
- `s6-log` for reliable logging with automatic rotation
- Readiness notification (similar to systemd's `Type=notify`)
- Proper dependency management via `s6-rc`

### Performance characteristics

The supervision model (runit, s6) has interesting performance properties:

**Low overhead**: The supervisor processes are tiny. `runsv` is a single small C program that calls `fork`, `exec`, and `wait` in a loop. There's almost no overhead beyond the kernel's process management.

**Fast restarts**: When a service crashes, the supervisor notices immediately (the child process exited) and restarts it. No polling, no PID file checking. The restart latency is essentially `fork()` + `exec()` time.

**No cgroup integration**: Like OpenRC, runit and s6 don't natively manage cgroups. You can wrap services with `cgexec` or use cgroup delegation, but it's manual.

**No built-in CPU affinity**: Same story. You'd add `taskset` or `numactl` to the `run` script.

s6 has one notable advantage: `s6-linux-init` can serve as PID 1, giving you a complete init system with proper signal handling, zombie reaping, and shutdown sequencing — all in a few hundred kilobytes of statically-linked C.

---

## The systemd Controversy: Unpacking the Arguments

The debate around systemd is one of the most persistent in the Linux community. Here's what both sides actually argue, stripped of tribalism.

### Arguments against systemd

**Complexity and attack surface**: systemd is large. It includes an init system, a logging daemon (journald), a network manager (networkd), a DNS resolver (resolved), a time synchronizer (timesyncd), a login manager (logind), a container manager (machined), a boot loader manager (bootctl), and more. Critics argue this violates the Unix philosophy of small, composable tools and increases the attack surface. A vulnerability in `systemd-resolved` compromises PID 1's security boundary.

**Binary logging**: journald stores logs in a binary format rather than plain text. You need `journalctl` to read them. If the journal corrupts (which happens), you lose logs. Plain text logs are universally readable, greppable, and resilient.

**Debugging opacity**: When a service fails to start under systemd, the error path can be difficult to trace. The interaction between unit file directives, cgroup setup, namespace creation, and socket activation creates a large state space. Compare this to a shell script where you can `bash -x` the startup sequence.

**Portability**: systemd is Linux-only. It uses cgroups, `epoll`, `signalfd`, `timerfd`, and other Linux-specific interfaces. BSD systems, Illumos, and other Unix variants can't run it. Software that hard-depends on systemd's interfaces (socket activation, `sd_notify`) becomes Linux-only.

**Scope creep**: systemd absorbs functionality that previously lived in separate projects. `udev` was merged into the systemd repository. `logind` replaced `ConsoleKit`. `resolved` competes with `unbound` and `dnsmasq`. Critics see this as ecosystem consolidation that reduces choice and increases coupling.

### Arguments for systemd

**Correctness**: The double-fork daemonization pattern is error-prone. PID files are unreliable. Shell script init systems have subtle bugs around quoting, signal handling, and error propagation. systemd replaces all of this with declarative configuration and cgroup-based tracking. Services that "just work" under systemd often had intermittent bugs under SysVinit.

**Performance features**: As detailed above, systemd provides CPU affinity, NUMA policy, memory limits, I/O weight, and other performance controls declaratively. Doing this with shell scripts is possible but fragile and non-standard.

**Dependency management**: systemd's dependency graph, combined with socket activation, enables aggressive parallelism during boot. Systems that took 30+ seconds to boot under SysVinit often boot in under 5 seconds with systemd.

**Standardization**: Before systemd, every distribution had its own init scripts with different conventions. A service packaged for Debian wouldn't work on Red Hat without modification. systemd unit files are portable across distributions. This is a real engineering benefit for anyone shipping software.

**Security features**: Unit files can declaratively enable `ProtectSystem=`, `PrivateTmp=`, `NoNewPrivileges=`, `SeccompFilter=`, and namespace isolation. These are available without writing any code — just configuration. Getting equivalent isolation with shell scripts requires expertise in Linux namespaces and seccomp.

**Journal correlation**: journald tags log entries with metadata (service name, PID, cgroup, boot ID, invocation ID). You can filter logs by service, by time range, by priority, across reboots. `grep` on `/var/log/syslog` can't do this.

### The performance engineering perspective

From a pure performance standpoint, systemd is the most capable init system available on Linux. No alternative provides the same level of integrated cgroup management, CPU affinity control, NUMA policy, I/O scheduling, and resource limiting without bolting on external tools.

But capability isn't free. systemd itself consumes memory (typically 10–30 MB resident for PID 1 plus journald, timesyncd, etc.). On a server running a handful of performance-critical services, this overhead is irrelevant. On an embedded system with 64 MB of RAM, it matters.

The journal's binary format also has performance implications. Writing structured binary log entries is faster than formatting and writing text lines (no `sprintf`, no text encoding). But reading and searching binary logs requires `journalctl`, which loads journal files into memory. For high-volume logging, the journal can consume significant I/O and memory.

For latency-sensitive services specifically, the relevant systemd features are:

| Feature | systemd | OpenRC | runit/s6 |
|---------|---------|--------|----------|
| CPU affinity | `CPUAffinity=` | Manual `taskset` | Manual `taskset` |
| NUMA binding | `NUMAPolicy=` | Manual `numactl` | Manual `numactl` |
| Core isolation | `AllowedCPUs=` | Not built-in | Not built-in |
| Memory limits | `MemoryMax=` | Not built-in | Not built-in |
| I/O priority | `IOWeight=` | Manual `ionice` | Manual `ionice` |
| Cgroup tracking | Automatic | Optional | Not built-in |
| Restart latency | ~ms | ~ms | ~ms |
| PID 1 memory | ~10-30 MB | ~1-5 MB | ~1-2 MB |

---

## Core Pinning: Why Init Systems Matter for Latency

To understand why init system integration with CPU affinity matters, consider what happens without it.

On a typical Linux system, the CFS (Completely Fair Scheduler) can migrate processes between cores freely. When your latency-sensitive process migrates from core 2 to core 5:

1. **L1 cache**: ~32 KB of hot data, completely cold on the new core. Refilling takes hundreds of nanoseconds per cache line.
2. **L2 cache**: ~256 KB–1 MB, also cold. Refill cost is higher (lower bandwidth from L3).
3. **L3 cache**: Shared across cores in a socket, so data might still be there — but access latency increases if the new core is in a different L3 slice.
4. **TLB**: Page table entries are per-core. A migration flushes the TLB, causing page table walks for every memory access until entries are repopulated.
5. **Branch predictor state**: Per-core, completely lost on migration.

The total cost of a core migration is typically 10–50 microseconds of degraded performance, depending on working set size. For a service processing requests in single-digit microseconds, that's catastrophic.

With systemd, you write `CPUAffinity=2 3` and the problem is solved at service startup. The kernel won't migrate the process off those cores. Combined with `isolcpus=2,3` on the kernel command line (which removes those cores from the general scheduler), you get a dedicated execution environment with no interference from other processes.

Going further, `AllowedCPUs=` in systemd works with cpusets (the cgroup v2 CPU controller). This is more flexible than `CPUAffinity` — it controls which CPUs the *entire cgroup* (all processes in the service) can use, and it's hierarchical.

For the most demanding workloads, you'd combine:
- `isolcpus=2,3,4,5` (kernel parameter) — removes cores from general scheduling
- `CPUAffinity=2 3 4 5` (systemd) — pins your service to those cores
- `NUMAPolicy=bind` + `NUMAMask=0` (systemd) — ensures memory allocations come from the local NUMA node
- `IRQAffinity` (via `/proc/irq/*/smp_affinity`) — moves interrupt handling off your isolated cores
- `nohz_full=2,3,4,5` (kernel parameter) — disables timer ticks on those cores when only one task is running

This kind of configuration is possible with any init system using wrapper scripts, but systemd makes it declarative and consistent. When you have 50 services on a machine and each needs specific CPU, memory, and I/O constraints, declarative configuration in unit files beats 50 hand-crafted shell scripts.

---

## What Actually Uses What

A quick survey of which distributions use which init system, as of 2026:

| Distribution | Init System | Notes |
|-------------|-------------|-------|
| Debian 12+ | systemd | Default since Debian 8 (Jessie) |
| Ubuntu 15.04+ | systemd | Migrated from Upstart |
| Fedora 15+ | systemd | One of the earliest adopters |
| Arch Linux | systemd | Default since 2012 |
| RHEL / CentOS / Rocky | systemd | Default since RHEL 7 |
| SUSE / openSUSE | systemd | Default since openSUSE 12.2 |
| Gentoo | OpenRC | systemd available as option |
| Alpine Linux | OpenRC | Musl-based, minimal footprint |
| Void Linux | runit | Explicitly anti-systemd |
| Artix Linux | OpenRC/runit/s6 | Arch without systemd |
| Devuan | sysvinit | Debian without systemd |
| Chimera Linux | dinit | New init system, service manager in C++ |
| Slackware | sysvinit | BSD-style init scripts |
| GuixSD | GNU Shepherd | Scheme-based service management |

The trend is clear: systemd dominates mainstream distributions. The alternatives survive in niches — embedded (Alpine), minimalist (Void), ideological (Devuan), and experimental (Chimera with dinit).

---

## Newer Alternatives: dinit and Beyond

It's worth mentioning dinit, a newer init system written in C++ that's gaining traction. It provides:
- Dependency-based service management (like systemd)
- Foreground service supervision (like runit/s6)
- Readiness notification
- Optional cgroup integration
- Small codebase (~15k lines of C++)

dinit occupies an interesting position: it provides the supervision and dependency management of systemd without the scope expansion. It doesn't include a logging daemon, DNS resolver, network manager, or any of systemd's auxiliary components. For users who want modern service management without the full systemd ecosystem, it's a credible option.

However, dinit currently lacks the deep cgroup integration that makes systemd compelling for performance engineering — you won't find equivalents of `CPUAffinity=`, `NUMAPolicy=`, or `MemoryMax=` as built-in service directives. You can still achieve these with wrapper scripts or cgroup delegation.

---

## Closing Thoughts

The double-fork daemonization pattern is a relic of a simpler time — when Unix systems ran a handful of services and terminals were physical devices. The fact that we needed a specific fork sequence to prevent a process from accidentally acquiring a controlling terminal tells you something about the accidental complexity that accumulated in Unix process management.

systemd's core insight was that the init system should *own* the process lifecycle rather than asking each daemon to implement it independently. This eliminated an entire class of bugs and enabled cgroup-based resource management that would be impractical to bolt onto a shell-script init system.

Whether the additional components bundled with systemd (journald, resolved, networkd, etc.) are a good idea is a separate question from whether its service management model is sound. You can disagree with the scope while acknowledging that `Type=simple` + cgroup tracking + declarative resource limits is a strictly better daemon management model than double-fork + PID files + shell scripts.

For performance engineering specifically, the question is practical: do you need integrated CPU affinity, NUMA policy, memory limits, and I/O control for your services? If yes, systemd gives you that declaratively. If your services are simple and your performance requirements are modest, OpenRC or runit will serve you fine with less overhead.

The kernel doesn't care which init system starts your process. It cares about which cgroup the process is in, which cores it's pinned to, which NUMA node its memory comes from, and whether anything else is competing for its cache lines. The init system is just the interface to those kernel mechanisms — and right now, systemd provides the most complete interface.

---

## References

- W. Richard Stevens, *Advanced Programming in the UNIX Environment*, Chapter 13: Daemon Processes
- [The Linux Kernel — Processes, sessions, and process groups](https://aeb.win.tue.nl/linux/lk/lk-10.html)
- [Double fork explanation (Stack Overflow)](https://stackoverflow.com/questions/881388/what-is-the-reason-for-performing-a-double-fork-when-creating-a-daemon)
- [systemd documentation](https://www.freedesktop.org/software/systemd/man/)
- [s6 documentation](https://skarnet.org/software/s6/)
- [dinit documentation](https://davmac.github.io/dinit/)
- [OpenRC documentation](https://wiki.gentoo.org/wiki/OpenRC)
- Lennart Poettering, [systemd for Administrators blog series](http://0pointer.de/blog/projects/systemd-for-admins-1.html)
