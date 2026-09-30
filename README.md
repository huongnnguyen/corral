# corral

**Can you escape the container?**

`corral` is a container runtime written from scratch in C, and an escape room built on top of it. You get a shell inside a container. A flag sits on the host, outside it. Every level is a container with a defense deliberately switched off. Your job is to get the flag, and in the process you learn what a container actually is.

It is not a Docker replacement. It is a working answer to "what *is* a container, really?", built by implementing one and then trying to break it.

> **Status:** in progress. Started September 2026. Weekend 1 is done; see [Progress](#progress) for what works today. Everything under [Play](#play) describes the plan, not something you can run yet.

---

## Who this is for

- You have heard "containers are just processes" and nodded without being able to explain it.
- You are a CS student who has used Docker but never looked underneath.
- You like CTF-style puzzles and want one that teaches real Linux internals.
- You just want to break out of something. That is a fine reason.

No prior kernel knowledge needed. Each level points you at the right man page instead of the answer.

## The big idea

A container is not a thing the Linux kernel has. There is no `container` struct, no container syscall. A container is an ordinary process that has been lied to about the world it lives in, and `corral` is a program that builds the lie.

The lie has four parts:

| Mechanism | What the process is told |
| --- | --- |
| **Namespaces** | These are the only processes, this is the hostname, these are the mounts, this is the network |
| **pivot_root** | This is the entire filesystem |
| **cgroups v2** | (told to the kernel) This process gets *this much* memory and *this many* PIDs |
| **Capabilities** | You are root, but you may not do root things |

Remove any one of them and the isolation leaks somewhere specific and demonstrable. That is the game: each level removes one, and you find the leak.

## Play

```
corral play <level>
```

Each level gives you a briefing, a shell inside the container, and optional hints. A flag file lives on the host. Read it and you escaped.

| Lvl | Name | What's missing | What you'll learn |
| --- | --- | --- | --- |
| 1 | **Chroot Jail** | `pivot_root` (only `chroot`) | Why chroot is a path change, not isolation |
| 2 | **Open Skies** | PID namespace | What namespaces hide, and what you can reach when they don't |
| 3 | **Leaky Mounts** | Mount hygiene | Isolation also means what you forgot to close or unmount |
| 4 | **Root Is Not Root** | Capability drop | Why root inside must not hold root's powers |
| 5 | **Fork Bomb Survival** | cgroup limits | Not an escape: crash the VM, then watch the limits contain it |
| 6 | **Open Door** | Network namespace | What a container sharing the host's network can reach |
| 7 | **The Vault** | Nothing. Fully hardened. | The final boss |

If anyone gets out of level 7, that is a real finding and goes straight into the escape table below.

**Hints, not spoilers.** Hints point at man pages (`man 7 capabilities`, `man 2 chroot`) rather than giving away the move. Solution write-ups will live in a separate folder, so no accidental spoilers while you browse the repo.

### Run it somewhere disposable

The weak levels are weak on purpose. A real escape gives the player root on whatever machine is running them. So:

- Use a throwaway VM, never your laptop or a shared machine.
- Do not expose it to the public internet unless it is a disposable cloud VM with nothing else on it.
- Snapshot first. Breaking out is supposed to be possible.

## Why containment, not detection

`corral` never inspects the code it runs. It does not scan for anything and does not decide whether a program is safe. It does not care whether it was handed a fork bomb or `print("hello")`.

That is deliberate. Deciding in advance what an arbitrary program will do is undecidable in general, and even ignoring that, there are infinitely many ways to write a fork bomb: a detector just gets handed variant 48. So `corral` asks a different question. Not *"is this program dangerous?"* but *"can any program, however dangerous, reach anything from inside here?"* The second question is finite, and it has a checkable answer.

Everyone gets let into the room. The room is built so that it doesn't matter who they are. Hence the game: if you can get out, the room has a bug.

## Progress

Each step adds a defense, and with it a level you can break.

- [x] **1 · chroot + exec**: run a process against a different root filesystem *(Level 1)*
- [ ] **2 · Namespaces**: PID, mount, UTS, IPC; the process sees itself as PID 1 *(Level 2)*
- [ ] **3 · pivot_root**: real root switching, `/proc` `/sys` `/dev` mounted correctly *(Levels 1, 3)*
- [ ] **4 · cgroups v2 + capabilities**: enforced memory and PID limits, capability drop *(Levels 4, 5)*
- [ ] **5 · Networking**: network namespace, veth pair, NAT to host *(Level 6)*
- [ ] **6 · CLI + game runner**: argument parsing, image unpacking, cleanup on exit, `corral play`
- [ ] **7 · Escape suite**: ~20 programs attempting to break out, results documented *(Level 7)*
- [ ] **8 · Writeup**: architecture, findings, demo recording

## Using it as a plain runtime

The game runs on the same code as the runtime, so you can also just run containers:

```
corral run <rootfs> <command> [args...]

  -m <bytes>    memory limit          (default 128M)
  -p <n>        max processes         (default 64)
  -t <seconds>  wall-clock timeout    (default 10)
  -n            enable networking     (default off)
```

Example of where this is headed:

```
$ sudo corral run ./alpine /bin/sh
/ # hostname
corral
/ # ps aux
PID   USER     COMMAND
    1 root     /bin/sh
    2 root     ps aux
```

Two processes. The host's several hundred are not hidden: from in here, they do not exist.

Under the hood, each level is just a runtime config with some defenses turned off (think `--no-pidns`, `--keep-caps`, `--no-cgroup`). Playing level 2 and running a container with `--no-pidns` are the same thing.

## Escape suite

Separate from the levels: a set of programs written specifically to break out of the fully hardened container. The table records what the sandbox actually did, not what it was supposed to do.

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

*Filled in during weekend 7. Cases that succeed stay in the table: those are the interesting ones.*

## Building

```
make
sudo ./corral run ./rootfs /bin/sh
```

Requires Linux (namespaces and cgroups v2 are kernel features, so this cannot run natively on macOS or Windows). Developed on Ubuntu 24.04. Root is required to create namespaces and write cgroup limits. WSL2, a local VM, or a cheap cloud VM all work.

## Notes

Written to learn, so the code favors clarity over completeness: one mechanism per file, comments explaining *why* a flag exists rather than what it does. Where `corral` differs from a production runtime like `runc`, the difference is noted in the source.

## References

- `man 7 namespaces`, `man 2 clone`, `man 2 pivot_root`, `man 7 cgroups`, `man 7 capabilities`
- Liz Rice, *Containers From Scratch*

---

Built by [Huong Nguyen](https://github.com/huongnnguyen) · CS @ UT Austin
