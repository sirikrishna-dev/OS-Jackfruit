```
# Multi-Container Runtime (OS-Jackfruit)

**Course:** Operating Systems — UE24CS252B
**Team Size:** Individual

---

## Team Information
- **Name:** Siri Krishna
- **SRN:** [Your SRN here]

---

## Project Summary
A lightweight Linux container runtime in C with a long-running supervisor and a kernel-space memory monitor. Manages multiple isolated containers, coordinates concurrent logging, exposes a CLI, and includes scheduling experiments.

---

## Build, Load, and Run Instructions

### Prerequisites
- Ubuntu 22.04/24.04 VM (VirtualBox), Secure Boot OFF
- Install dependencies:
```
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### Build
```
make
```

### Load Kernel Module
```
sudo insmod monitor.ko
```

### Start Supervisor (Terminal 1)
```
sudo ./engine supervisor ./rootfs
```

### Start Containers (Terminal 2)
```
sudo ./engine start alpha ./rootfs /bin/sh
sudo ./engine start beta ./rootfs /bin/sh
```

### List Containers
```
sudo ./engine ps
```

### View Logs
```
sudo ./engine logs alpha
```

### Scheduling Experiments
```
time sudo ./engine start high ./rootfs /cpu_hog --nice -5
time sudo ./engine start low ./rootfs /cpu_hog --nice 10
```

### Stop and Cleanup
```
sudo ./engine stop alpha
sudo ./engine stop beta
sudo rmmod monitor
sudo dmesg | tail -5
```

---

## Demo Screenshots

| # | What it Shows |
|---|---------------|
| 1 | Multi-container supervision — two containers running under one supervisor |
| 2 | `ps` command showing container metadata (PID, STATE, LOG) |
| 3 | Logging pipeline — log file contents captured from container |
| 4 | CLI command issued and supervisor responding |
| 5 | dmesg showing SOFT LIMIT warning |
| 6 | dmesg showing HARD LIMIT kill + supervisor metadata updated |
| 7 | Scheduling experiment — high vs low nice value timing |
| 8 | Clean teardown — containers reaped, module unloaded, no zombies |

---

## Engineering Analysis

### 1. Isolation Mechanisms
Containers are isolated using Linux namespaces: PID namespace gives each container its own process tree, UTS namespace gives it its own hostname, and mount namespace + chroot gives it its own filesystem view using the Alpine rootfs. The host kernel is still shared — all containers use the same kernel, same system calls, and the host can see all container PIDs.

### 2. Supervisor and Process Lifecycle
A long-running supervisor is necessary to track container metadata, reap exited children (preventing zombies), and handle signals. When a container exits, SIGCHLD is sent to the supervisor which calls waitpid() to reap it and update its state to stopped. Without the supervisor, zombie processes would accumulate.

### 3. IPC, Threads, and Synchronization
Two IPC mechanisms are used: pipes for capturing container stdout/stderr into the logging pipeline, and a UNIX domain socket for CLI commands to reach the supervisor. The bounded buffer between producer (container output) and consumer (log writer thread) is protected by a mutex and condition variables to avoid race conditions, lost data, and deadlock.

### 4. Memory Management and Enforcement
RSS (Resident Set Size) measures physical memory currently used by a process. It does not measure virtual memory or shared libraries. Soft limits log a warning when first exceeded — giving the process a chance to recover. Hard limits terminate the process when exceeded. Enforcement belongs in kernel space because user-space cannot reliably monitor and kill processes — the kernel has direct access to process memory maps and can act atomically.

### 5. Scheduling Behavior
Experiments showed that a container with nice -5 (higher priority) completed CPU-bound work in ~0.9s while nice +10 (lower priority) took ~10s. This shows the Linux CFS scheduler allocates more CPU time to higher priority processes. I/O-bound processes were less affected by nice values since they spend most time waiting, not competing for CPU.

---

## Design Decisions and Tradeoffs

| Subsystem | Choice | Tradeoff | Justification |
|-----------|--------|----------|---------------|
| Namespace isolation | PID + UTS + mount | No network isolation | Sufficient for this project scope |
| Supervisor architecture | Single long-running process | Single point of failure | Simplest way to track all container state |
| IPC/Logging | Pipe + UNIX socket | Two separate channels to manage | Separates concerns cleanly |
| Kernel monitor | Periodic RSS polling | Not real-time | Simple and reliable, avoids complexity |
| Scheduling experiments | nice values | Limited to CFS tuning | Directly observable and measurable |

---

## Scheduler Experiment Results

| Container | Nice Value | Completion Time |
|-----------|------------|----------------|
| high | -5 | ~0.9 seconds |
| low | +10 | ~10.0 seconds |
| high2 | -5 | ~0.9 seconds |
| low2 | +10 | ~10.0 seconds |

The results show CFS gives significantly more CPU time to higher priority processes. Lower priority containers had to wait while higher priority ones ran, resulting in ~10x difference in completion time.

---

## References
1. Linux Namespaces — https://man7.org/linux/man-pages/man7/namespaces.7.html
2. Linux Kernel Module Programming — https://sysprog21.github.io/lkmpg/
3. Alpine Linux rootfs — https://alpinelinux.org/
```

