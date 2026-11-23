# Kubernetes Internals: Pod Anatomy & Linux Namespaces

## 1. Architectural Mental Model

In Kubernetes, a **Pod is not a virtual machine or a monolithic process**; it is a **shared Linux kernel isolation boundary**. While a standalone container runtime isolates every process into private namespaces, a Pod intentionally binds multiple containers together into shared namespaces while preserving distinct filesystem trees.

The runtime bootstraps this via the **Pod Sandbox Pattern** using the `/pause` container.

```
+---------------------------------------------------------------------------------------------------------------+
|                                      POD SANDBOX ARCHITECTURE & BOUNDARIES                                    |
+---------------------------------------------------------------------------------------------------------------+

 [ HOST NODE KERNEL SPACE ]
  Path: /sys/fs/cgroup/kubepods.slice/pod<UUID>/
       |
       +--------------------------------------------------------------------------------------------------------+
       | SHARED LINUX NAMESPACES (Pinned by /pause)                                                             |
       |                                                                                                        |
       |  [Network Namespace: net:[4026532890]]                                                                 |
       |   ├── eth0: IP 10.244.1.25 (Single shared IP for the Pod)                                             |
       |   └── lo:   127.0.0.1      (Shared loopback interface for local IPC communication)                     |
       |                                                                                                        |
       |  [IPC Namespace: ipc:[4026532891]]                                                                     |
       |   └── Shared memory segments (/dev/shm) and POSIX message queues                                       |
       |                                                                                                        |
       |  [UTS Namespace: uts:[4026532892]]                                                                     |
       |   └── Hostname: pod-payment-api-67d4f9b8c-z9x2                                                         |
       +--------------------------------------------------------------------------------------------------------+
             ^                                     ^                                     ^
             | join netns, ipc, uts                | join netns, ipc, uts                | anchor namespaces
             |                                     |                                     |
+--------------------------+          +--------------------------+          +-----------------------------------+
| APPLICATION CONTAINER    |          | LOGGING / SIDECAR        |          | PAUSE CONTAINER                   |
|                          |          |                          |          |                                   |
| Private Mount Namespace: |          | Private Mount Namespace: |          | Process:                          |
|  rootfs -> App image     |          |  rootfs -> Sidecar image |          |  pid = 1                          |
| Private PID Namespace:   |          | Private PID Namespace:   |          |  pause() syscall (sleeps)         |
|  pid 1 = app binary      |          |  pid 1 = sidecar binary  |          |  Reaps orphaned zombie processes  |
+--------------------------+          +--------------------------+          +-----------------------------------+
```

---

## 2. Under the Hood (Linux Namespaces Breakdown)

Every containerized process inside a Pod is isolated by specific Linux kernel namespaces:

| Namespace | Linux Flag | Kernel Boundary | In Pod Context |
| :--- | :--- | :--- | :--- |
| **Mount (`MNT`)** | `CLONE_NEWNS` | File system tree (`/`) | **Private per container**: Container A cannot see Container B's filesystem unless explicitly shared via a `Volume`. |
| **Network (`NET`)** | `CLONE_NEWNET` | IP, routing table, ports, `lo` | **Shared across Pod**: All containers share `eth0`, port space, and `127.0.0.1`. |
| **IPC** | `CLONE_NEWIPC` | POSIX message queues, SysV shared memory | **Shared across Pod**: Containers can share memory blocks directly. |
| **UTS** | `CLONE_NEWUTS` | Hostname and NIS domain | **Shared across Pod**: All containers share the Pod's hostname. |
| **PID** | `CLONE_NEWPID` | Process ID tree | **Private by default**: Can be shared by configuring `shareProcessNamespace: true`. |
| **User** | `CLONE_NEWUSER` | UID and GID mapping | Translates container root (UID 0) to unprivileged host UID. |

### The `/pause` Program
The pause container executes a minimalist C program that performs two critical tasks:
1. Calls `pause()` or `sigsuspend()` to yield CPU back to the kernel while holding open the namespace file descriptors under `/proc/<pid>/ns/`.
2. Serves as PID 1 to reap orphaned child processes (`zombies`) when processes die before their sub-threads.

---

## 3. Production Manifest: Inter-Container Localhost Communication

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-and-logger
  namespace: default
spec:
  containers:
    - name: web-app
      image: nginx:1.25-alpine
      ports:
        - containerPort: 80
          name: http
    - name: sidecar-logger
      image: curlimages/curl:8.4.0
      command: ["/bin/sh", "-c"]
      # Communicates directly via localhost because both containers share netns
      args:
        - while true; do curl -s http://127.0.0.1:80/ > /dev/null; sleep 5; done
```

---

## 4. Diagnostics: Entering the Pod Network Namespace from the Host Node

To inspect or debug a container running a scratch image without any tools installed:

```bash
# 1. Identify the container PID on the worker node
CONTAINER_ID=$(crictl ps --name web-app -q)
PID=$(crictl inspect $CONTAINER_ID | jq .info.pid)

# 2. Attach host-native diagnostic tools directly to the pod's network namespace
sudo nsenter -t $PID -n ip addr show
sudo nsenter -t $PID -n ss -tulpen
sudo nsenter -t $PID -n tcpdump -i eth0 -nn -vv
```
