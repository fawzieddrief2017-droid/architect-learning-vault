# Kubernetes Internals: Linux IPC, Shared Memory & The Pod Boundary

## 1. Architectural Mental Model (The "Why" & "What")

In operating systems engineering, **IPC (Inter-Process Communication)** refers to the kernel subsystems that allow separate, memory-isolated processes to exchange data, synchronize execution, and share state.

By default, the Linux Virtual Memory Manager (VMM) enforces hardware-assisted page table isolation: Process A cannot read or write to Process B's virtual memory space. IPC mechanisms bypass this boundary without the serialization overhead of network sockets:
1. **Shared Memory (`shm`):** The kernel maps the identical physical RAM pages into the page tables of multiple processes. Data exchange occurs at memory bus speeds with **zero syscalls and zero memory copies**.
2. **POSIX / SysV Message Queues (`mq`):** Priority queues stored in kernel memory.
3. **Semaphores (`sem`):** Kernel counters used for synchronization and mutual exclusion (`mutex`).

In Kubernetes, the **Linux IPC Namespace (`CLONE_NEWIPC`)** is one of the core isolation boundaries allocated to the Pod's sandbox.

```
+---------------------------------------------------------------------------------------------------------------+
|                                     LINUX INTER-PROCESS COMMUNICATION (IPC)                                   |
+---------------------------------------------------------------------------------------------------------------+

  PROCESS A (e.g., Python Web Server)                       PROCESS B (e.g., C++ Analytics Worker)
   Virtual Address: 0x7fff1000                               Virtual Address: 0x7fff9000
          |                                                         |
          | Page Table Pointer (PTE)                                | Page Table Pointer (PTE)
          v                                                         v
  +-----------------------------------------------------------------------+
  | KERNEL MEMORY / PHYSICAL RAM: SHARED MEMORY SEGMENT (`/dev/shm`)      |
  |  - Allocated via `shmget()` / `mmap(MAP_SHARED)`                       |
  |  - Zero copy: Process A writes -> Process B reads instantly           |
  |  - Synchronized via POSIX Semaphores (`sem_wait` / `sem_post`)        |
  +-----------------------------------------------------------------------+

=================================================================================================================
                                   KUBERNETES IPC NAMESPACE BOUNDARY
=================================================================================================================

 [ NODE PHYSICAL HOST: ipc:[4026531839] ]
   ├── Host processes have their own SysV/POSIX IPC tables (`ipcs -a`)
   |
   +---> [ POD 1: ipc:[4026532891] (Pinned by /pause) ]
   |       ├── Container 1 (App)   \__ Bound to the SAME IPC namespace!
   |       └── Container 2 (Sidecar)/   They can access the same shared memory segments & message queues.
   |
   +---> [ POD 2: ipc:[4026534500] (Isolated Sandbox) ]
           └── Container 3 cannot discover, attach to, or corrupt Pod 1's shared memory segments.
```

---

## 2. Under the Hood (Mechanics & Internals)

### A. SysV IPC vs. POSIX IPC

| Feature / Metric | System V (SysV) IPC | POSIX IPC |
| :--- | :--- | :--- |
| **Identifiers** | Numeric keys (`key_t`) and IDs | Named file-like paths (`/dev/shm/<name>`) |
| **Interface** | `shmget`, `shmat`, `shmdt`, `msgget`, `semget` | `shm_open`, `mmap`, `mq_open`, `sem_open` |
| **Filesystem Visibility** | Virtual memory only; inspected via `/proc/sysvipc/` | First-class files on `tmpfs` mounted at `/dev/shm` |
| **Lifecycle** | Persists until explicit `IPC_RMID` or reboot | Persists until `shm_unlink()` or reboot |

### B. What Does `CLONE_NEWIPC` Isolate?
- System V IPC message queues, semaphore sets, and shared memory segments.
- POSIX message queue virtual filesystem mounts (`/dev/mqueue`).
- Prevents unprivileged processes in one container from scanning or modifying shared memory segments owned by other workloads on the same node.

---

## 3. Production Implementation: High-Performance `/dev/shm` Allocation

Containers default to a **64 MiB limit on `/dev/shm`**. High-performance data workloads (PostgreSQL, PyTorch, Chromium) will crash if this limit is exceeded. 

To expand `/dev/shm` and share it across containers in a Pod, mount an in-memory `emptyDir`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-memory-pipeline
  namespace: analytics
spec:
  containers:
    - name: data-producer
      image: registry.internal.net/analytics/producer:v1.0
      volumeMounts:
        - name: dshm
          mountPath: /dev/shm
      resources:
        limits:
          memory: 4Gi
        requests:
          memory: 2Gi

    - name: data-consumer
      image: registry.internal.net/analytics/consumer:v1.0
      volumeMounts:
        - name: dshm
          mountPath: /dev/shm
      resources:
        limits:
          memory: 4Gi
        requests:
          memory: 2Gi

  volumes:
    - name: dshm
      emptyDir:
        medium: Memory # Allocates a tmpfs instance directly in RAM
        sizeLimit: 2Gi # Overrides the restrictive 64MiB default limit
```

---

## 4. Failure Modes, Bottlenecks & Troubleshooting

### Failure Mode: Orphaned Shared Memory Leaks (`OOMKilled`)
- **Symptom:** Pod memory usage gradually climbs until terminated with exit code `137` (`OOMKilled`), but heap memory is stable.
- **Root Cause:** A crashed process failed to invoke `shmctl(shmid, IPC_RMID)` or `shm_unlink()`. Unlike heap allocations, shared memory segments persist in memory as long as the IPC namespace or container lives.
- **Diagnostics:**
  ```bash
  # Enter the Pod IPC namespace from the host node
  PID=$(crictl inspect $(crictl ps --name data-producer -q) | jq .info.pid)
  sudo nsenter -t $PID -i ipcs -m

  # Clean up orphaned shared memory segments
  sudo nsenter -t $PID -i ipcrm -m <shmid>
  ```
