## Weekend 1: chroot vs. container (Sep 26-27)

**Build:** `FROM scratch` Docker image built directly from the Alpine minirootfs tarball.

\`\`\`bash
docker build -t alpine-scratch .
docker run -it --rm alpine-scratch   # w/ interactive shell & clean up after container closes
\`\`\`

**Inside:**

\`\`\`
/ # ls /
bin   etc    lib    mnt   proc   run   srv   tmp   var
dev   home   media  opt   root   sbin  sys   usr

/ # cat /etc/os-release
NAME="Alpine Linux"
ID=alpine
VERSION_ID=3.24.2
PRETTY_NAME="Alpine Linux v3.24"
HOME_URL="https://alpinelinux.org/"
BUG_REPORT_URL="https://gitlab.alpinelinux.org/alpine/aports/-/issues"

/ # ps aux
PID   USER   TIME   COMMAND
1     root   0:00   /bin/sh
13    root   0:00   ps aux
\`\`\`

Bare Alpine filesystem, nothing from host. Only processes visible: the shell (PID 1) and `ps aux` (PID 13, which happens to be the next available in this namespace).

**Host side (`docker inspect`):**

\`\`\`json
"State": {
    "Status": "running",
    "Running": true,
    "Paused": false,
    "Restarting": false,
    "OOMKilled": false,
    "Dead": false,
    "Pid": 48837,
    "ExitCode": 0,
    "Error": "",
    "StartedAt": "2026-09-27T02:35:24.984232768Z",
    "FinishedAt": "0001-01-01T00:00:00Z"
},
...
\`\`\`

Think Severance, the show :P

**Isolation importance:** Security, Fault containment, Reproducibility ("works on my machine"), multi-tenancy (think large scale, you shouldn't be able to see other people's container)

**Next:** build the chroot-by-hand version (`fork` + `chroot` + `exec`) and see which of these properties it *doesn't* get for free.