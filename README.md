# corral

A container runtime built from scratch in C, to understand what actually separates a container from the host.

Not a Docker replacement. A working answer to the question "what *is* a container, really?" — built by implementing one.

> **Status:** in progress. Started September 2026. See [Progress](#progress) for what works today.

---

## The idea

A container is not a thing the Linux kernel has. There is no `container` struct, no container syscall. A container is an ordinary process that has been lied to about the world it lives in, and `corral` is a program that constructs the lie.

The lie has four parts:

| Mechanism | What the process is told |
| --- | --- |
| **Namespaces** | These are the only processes, this is the hostname, these are the mounts, this is the network |
| **pivot_root** | This is the entire filesystem |
| **cgroups v2** | (told to the kernel) This process gets *this much* memory and *this many* PIDs |
| **Capabilities** | You are root, but you may not do root things |

Take away any one of them and the isolation leaks somewhere specific and demonstrable — which is what the escape suite is for.

## Why containment, not detection

`corral` never inspects the code it runs. It does not scan for anything, does not decide whether a program is safe, and does not care whether it was handed a fork bomb or `print("hello")`.

That is deliberate. Deciding in advance what an arbitrary program will do is undecidable in the general case, and even ignoring that, there are infinitely many ways to write a fork bomb — a detector just gets handed variant 48. So `corral` asks a different question: not *"is this program dangerous?"* but *"can any program, however dangerous, reach anything from inside here?"* The second question is finite, and it has a checkable answer.

Everyone gets let into the room. The room is built so that it doesn't matter who they are.

## Progress

- [ ] **1 · chroot + exec** — run a process against a different root filesystem
- [ ] **2 · Namespaces** — PID, mount, UTS, IPC; process sees itself as PID 1
- [ ] **3 · pivot_root** — real root switching, `/proc` `/sys` `/dev` mounted correctly
- [ ] **4 · cgroups v2** — enforced memory and PID limits
- [ ] **5 · Networking** — network namespace, veth pair, NAT to host
- [ ] **6 · CLI** — argument parsing, image unpacking, cleanup on exit
- [ ] **7 · Escape suite** — ~20 programs attempting to break out, results documented
- [ ] **8 · Writeup** — architecture, findings, benchmarks

## Usage

```
corral run <rootfs> <command> [args...]

  -m <bytes>    memory limit          (default 128M)
  -p <n>        max processes         (default 64)
  -t <seconds>  wall-clock timeout    (default 10)
  -n            enable networking     (default off)
```

Example:

```
$ sudo corral run ./alpine /bin/sh
/ # hostname
corral
/ # ps aux
PID   USER     COMMAND
    1 root     /bin/sh
    2 root     ps aux
```

Two processes. The host's several hundred are not hidden — from in here, they do not exist.

## Escape suite

Each case is a program written specifically to break out. The table records what the sandbox actually did, not what it was supposed to do.

| # | Attack | Expected | Result |
| --- | --- | --- | --- |
| 1 | Fork bomb | Capped by `pids.max` | — |
| 2 | Unbounded allocation | OOM-killed by `memory.max` | — |
| 3 | Infinite loop | Killed at timeout | — |
| 4 | Outbound network call | Fails, no route | — |
| 5 | Write outside root | Fails, read-only mount | — |
| 6 | `chroot` escape via open fd | Blocked by `pivot_root` | — |
| 7 | Read host `/proc` | Sees only own namespace | — |
| 8 | Mount a filesystem | Blocked, `CAP_SYS_ADMIN` dropped | — |

*Filled in during week 7. Cases that succeed stay in the table — those are the interesting ones.*

## Building

```
make
sudo ./corral run ./rootfs /bin/sh
```

Requires Linux (namespaces and cgroups v2 are kernel features — this cannot run on macOS or Windows). Developed on Ubuntu 24.04. Root is required to create namespaces and write cgroup limits.

## Notes

Written to learn, so the code favors clarity over completeness: one mechanism per file, comments explaining *why* a flag exists rather than what it does. Where `corral` differs from a production runtime like `runc`, the difference is noted in the source.

## References

- `man 7 namespaces`, `man 2 clone`, `man 2 pivot_root`, `man 7 cgroups`, `man 7 capabilities`
- Liz Rice, *Containers From Scratch*

---

Built by [Huong Nguyen](https://github.com/huongnnguyen) · CS @ UT Austin
