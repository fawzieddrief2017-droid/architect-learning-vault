# CSI Storage Architecture, Volume Lifecycle & Two-Phase Mount Engine

## 1. Architectural Mental Model (The "Why" & "What")

In cloud-native systems, compute instances are ephemeral, but persistent state must survive pod recreation, node drains, and physical hardware failures. The **Container Storage Interface (CSI)** standardizes storage operations into an out-of-tree, gRPC-driven state machine that cleanly isolates the Kubernetes control plane from cloud provider vendor APIs.

To guarantee crash consistency, prevent split-brain write collisions, and enforce security isolation, Kubernetes handles persistent block storage (such as AWS EBS) through a strict **Two-Phase Commit Architecture**:

1. **Phase 1: Control Plane Attachment (Cloud Fabric Level):** Handled by `AttachDetachController` inside `kube-controller-manager` via the out-of-tree `csi-attacher` sidecar. It calls the cloud API (`ec2:AttachVolume`) to attach the raw block device to the virtual machine hypervisor (AWS Nitro PCIe bus). The node's operating system does not touch the filesystem yet.
2. **Phase 2: Node-Level Staging & Mounting (Linux Kernel VFS Level):** Handled locally on the worker node by `kubelet`'s **`VolumeManager`** reconciliation loop. It invokes the local CSI Node daemon via UNIX domain sockets to format the raw block device (`NodeStageVolume`), mount it to a global host directory, and execute an atomic Linux VFS bind mount (`NodePublishVolume`) directly into the Pod's isolated mount namespace (`CLONE_NEWNS`).

```
+───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────+
|                                    CSI TWO-PHASE VOLUME ATTACH & MOUNT STATE MACHINE                                  |
+───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────+

  KUBERNETES CONTROL PLANE                                          AWS CLOUD FABRIC (NITRO)
  ┌────────────────────────────────────────────────────────┐        ┌─────────────────────────────────────────────────┐
  │ 1. PVC Created & Bound to PV (StorageClass: gp3)       │        │ AWS EC2 & EBS Infrastructure                    │
  │    Pod scheduled to Node-A (kube-scheduler)            │        │                                                 │
  └──────────────────────────┬─────────────────────────────┘        │                                                 │
                             │ Watch Event (Pod Scheduled)          │                                                 │
                             ▼                                      │                                                 │
  ┌────────────────────────────────────────────────────────┐        │                                                 │
  │ 2. AttachDetachController (kube-controller-manager)     │        │                                                 │
  │    - Creates VolumeAttachment API Object in etcd       │        │                                                 │
  └──────────────────────────┬─────────────────────────────┘        │                                                 │
                             │ HTTP/2 Watch                         │                                                 │
                             ▼                                      │                                                 │
  ┌────────────────────────────────────────────────────────┐        │                                                 │
  │ 3. csi-attacher (Controller Sidecar Pod)               │        │                                                 │
  │    - Issues gRPC: ControllerPublishVolume()            ├───────►│ Calls EC2 API: AttachVolume                     │
  │      (VolumeId: vol-0abc, NodeId: i-0123NodeA)         │        │ Nitro attaches block device to PCI bus!         │
  └────────────────────────────────────────────────────────┘        └────────────────────────┬────────────────────────┘
                                                                                             │
=============================================================================================│===========================
  WORKER NODE (Node-A: Linux Kernel & Kubelet)                                               │ Device /dev/nvme1n1 appears
=============================================================================================│===========================
                                                                                             ▼
  ┌───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
  │ 4. Kubelet VolumeManager (Reconciliation Loop on Node-A)                                                          │
  │    - Detects block device /dev/nvme1n1 populated on PCIe bus via udev                                             │
  │    - Issues node-local gRPC over UNIX domain socket (/var/lib/kubelet/plugins/ebs.csi.aws.com/csi.sock):          │
  │                                                                                                                   │
  │    [ STEP A: NodeStageVolume() ] (Global Host Format & Mount)                                                     │
  │    ├─ Probes block device for filesystem header (blkid /dev/nvme1n1)                                              │
  │    ├─ If raw: runs mkfs.ext4 -F -m0 /dev/nvme1n1                                                                  │
  │    └─ Mounts to global host staging directory:                                                                    │
  │       /var/lib/kubelet/plugins/kubernetes.io/csi/ebs.csi.aws.com/<vol-id>/globalmount                             │
  │                                                                                                                   │
  │    [ STEP B: NodePublishVolume() ] (Pod Namespace Injection)                                                      │
  │    ├─ Creates Pod-specific volume target directory:                                                               │
  │    │  /var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~csi/<vol-name>/mount                                  │
  │    └─ Executes Linux VFS bind mount:                                                                              │
  │       mount -o bind /var/lib/kubelet/.../globalmount /var/lib/kubelet/pods/.../mount                               │
  └────────────────────────────────────────┬──────────────────────────────────────────────────────────────────────────┘
                                           │
                                           │ Container Creation (CRI / containerd)
                                           ▼
  ┌───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
  │ 5. Container Mount Namespace (CLONE_NEWNS)                                                                        │
  │    - Container process accesses data at target path: /data (Backed directly by NVMe EBS block device)             │
  └───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Under the Hood: Mechanics & Internals

### A. The Declarative API Ladder & Dynamic Provisioning Loop

Kubernetes decouples storage administration from developer consumption through three declarative API layers:

1. **`StorageClass` (Infrastructure Blueprint):** Defined by the Cloud Architect. Specifies the CSI provisioner plugin (`ebs.csi.aws.com`), volume type (`gp3`), baseline IOPS/throughput, KMS encryption key ARN, and topology binding rules (`volumeBindingMode`).
2. **`PersistentVolumeClaim` (Developer Request):** Defined by application manifests. Requests capacity (`100Gi`) and access mode (`ReadWriteOncePod`), agnostic of the underlying cloud provider.
3. **`PersistentVolume` (Bound Cloud Asset):** Instantiated dynamically by the `csi-provisioner` sidecar. Stores the unique AWS volume handle (`csi.volumeHandle: vol-0987654321fedcba0`) and node affinity topology constraints (`topology.ebs.csi.aws.com/zone: us-east-1a`).

#### The Dynamic Provisioning Sequence:
```text
PVC Created (Pending) ──> csi-provisioner watches API ──> Calls AWS EC2 CreateVolume 
                      ──> AWS allocates EBS disk ──> csi-provisioner creates PV 
                      ──> PersistentVolumeController binds PVC to PV (Bound)
```

#### The Multi-AZ Trap: `volumeBindingMode`
* **`Immediate` (Anti-pattern in multi-AZ):** Volume creation is triggered the microsecond the PVC is committed to `etcd`. Because the Pod has not been scheduled yet, the CSI driver picks an AZ at random or uses a default. When the Pod is subsequently scheduled, if compute nodes with available CPU/memory exist only in a **different AZ**, the Pod remains permanently unschedulable (`volume node-affinity conflict`).
* **`WaitForFirstConsumer` (Production Standard):** Postpones volume provisioning until `kube-scheduler` selects a specific worker node based on compute resources and pod anti-affinity. The scheduler extracts the node's zone topology label (`topology.kubernetes.io/zone=us-east-1b`) and injects it into the provisioning request. The EBS volume is guaranteed to be created in the exact AZ where the pod will run.

---

### B. The Controller vs. Node Plugin Separation

The AWS EBS CSI Driver architecture runs two decoupled components across privilege boundaries:

```
+──────────────────────────────────────────────────────────────────────────────────────────────────+
| 1. CSI CONTROLLER DEPLOYMENT (Cluster-wide, Leader-Elected)                                      |
|    - Runs in kube-system as a Deployment with multiple sidecars:                                 |
|      * csi-provisioner: Calls AWS CreateVolume / DeleteVolume APIs                               |
|      * csi-attacher: Watches VolumeAttachment objects -> Calls AWS AttachVolume / DetachVolume   |
|      * csi-resizer: Watches PVC size updates -> Calls AWS ModifyVolume (online expansion)        |
+──────────────────────────────────────────────────────────────────────────────────────────────────+
                                           │
                                           │ Control Plane communicates via AWS IAM (IRSA)
                                           ▼
+──────────────────────────────────────────────────────────────────────────────────────────────────+
| 2. CSI NODE DAEMONSET (Host-Local, Runs on EVERY Worker Node)                                    |
|    - Listens on UNIX domain socket: /var/lib/kubelet/plugins/ebs.csi.aws.com/csi.sock            |
|    - Kubelet communicates directly over this socket via gRPC:                                    |
|      * NodeStageVolume(): Formats block device (mkfs) & mounts to host globalmount               |
|      * NodePublishVolume(): Binds host globalmount into pod mount namespace (mount --bind)       |
|      * NodeUnpublishVolume(): Unbinds pod volume directory                                       |
|      * NodeUnstageVolume(): Unmounts device from host globalmount                                |
+──────────────────────────────────────────────────────────────────────────────────────────────────+
```

---

### C. Linux VFS Bind Mounts & Mount Namespaces

When `kubelet` sets up storage for a container, it relies on two Linux kernel primitives:

1. **Global Staging (`NodeStageVolume`):** Formats the raw block device with an `ext4` or `XFS` filesystem if unformatted, and mounts it to a single host directory:
   ```bash
   mount -t ext4 /dev/nvme1n1 /var/lib/kubelet/plugins/kubernetes.io/csi/ebs.csi.aws.com/vol-0123/globalmount
   ```
   *Why this exists:* A block device must never be formatted or mounted multiple times simultaneously. Global staging guarantees that formatting occurs exactly once, even if multiple containers within the same Pod share the volume.

2. **Pod Namespace Projection (`NodePublishVolume`):** Uses an atomic Linux VFS bind mount (`mount --bind`) to project the global mount directory into the Pod's private volume target directory:
   ```bash
   mount --bind \
     /var/lib/kubelet/plugins/kubernetes.io/csi/ebs.csi.aws.com/vol-0123/globalmount \
     /var/lib/kubelet/pods/550e8400-e29b-41d4-a716-446655440000/volumes/kubernetes.io~csi/pvc-data/mount
   ```
3. **Container Runtime Namespace Isolation (`CLONE_NEWNS`):** When `containerd` invokes `runc` to spawn the container, it calls `pivot_root` or `chroot`. The target path inside the container (e.g., `/var/lib/postgresql/data`) points directly to the mount table inode created by the bind mount, providing raw, unencumbered block I/O performance directly to the underlying NVMe controller.

---

### D. Concurrency Models: `ReadWriteOnce` (RWO) vs. `ReadWriteOncePod` (RWOP)

Understanding the blast radius of access modes is critical for zero-downtime architecture:

| Access Mode | Locking Boundary | Multi-Node Mount? | Multi-Pod on Same Node? | Safe for Rolling Deployments? |
| :--- | :--- | :--- | :--- | :--- |
| **`ReadWriteOnce` (RWO)** | **Single Compute Node** | ❌ Blocked by AWS Hypervisor | ✅ **Allowed by Kubernetes!** | ❌ **No** (Causes `Multi-Attach error`) |
| **`ReadWriteOncePod` (RWOP)** | **Single Container Pod** | ❌ Blocked by Scheduler | ❌ **Blocked by Admission Control!** | ✅ **Yes** (Enforces strict 1:1 exclusivity) |
| **`ReadWriteMany` (RWX)** | **Distributed Cluster** | ✅ Allowed (NFS / EFS) | ✅ Allowed | ✅ Yes (Requires distributed lock manager) |

#### The Fatal RWO Misconception:
In Kubernetes, `ReadWriteOnce` does **not** mean "Single Pod". It means **"Single Worker Node"**.
* **Same-Node Risk (Silent Data Corruption):** If two pods mount the same RWO volume and land on the same worker node (e.g., single-node test environments or misconfigured replica sets), both pods bind-mount the same global ext4 filesystem simultaneously. Because standard filesystems (`ext4`/`XFS`) lack distributed lock managers, simultaneous writes corrupt inode journals and destroy database files.
* **Multi-Node Risk (The Rolling Update Hang):** During a rolling update, a Deployment defaults to `maxSurge: 25%`. The controller schedules Pod-v2 on Node-B before terminating Pod-v1 on Node-A. Since EBS volumes cannot attach to two EC2 hypervisors concurrently, AWS rejects the `AttachVolume` API call, hanging Pod-v2 in `ContainerCreating` indefinitely.

#### The RWOP Solution (Kubernetes 1.27+ GA):
`ReadWriteOncePod` enforces strict, cluster-wide single-pod exclusivity at the admission controller and scheduler levels. If a workload attempts to mount an RWOP PVC that is already consumed by another pod (even on the same worker node), admission is immediately rejected.

---

## 3. Production Implementation: Hardened AWS EBS CSI Architecture

### A. Production `StorageClass` (AWS EKS gp3)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3-production
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer        # ──► Prevents cross-AZ scheduling deadlocks
reclaimPolicy: Retain                          # ──► Prevents accidental data deletion on PVC removal
allowVolumeExpansion: true                     # ──► Enables online filesystem resizing without downtime
parameters:
  type: gp3                                    # ──► General Purpose SSD (baseline 3,000 IOPS, 125 MB/s)
  iops: "3000"
  throughput: "125"
  encrypted: "true"                            # ──► Enforces AWS KMS envelope encryption at rest
  kmsKeyId: "arn:aws:kms:us-east-1:123456789012:key/production-storage-key"
```

### B. Single-Replica Database Workload with RWOP & Rolling Update Budget

When deploying single-replica stateful databases via a `Deployment`, you must explicitly set `maxSurge: 0` to prevent volume attachment deadlocks:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data-pvc
  namespace: databases
spec:
  accessModes:
    - ReadWriteOncePod                         # ──► Cluster-wide single-pod lock
  storageClassName: ebs-gp3-production
  resources:
    requests:
      storage: 100Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-primary
  namespace: databases
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0                              # ──► MANDATORY: Never create Pod-v2 before Pod-v1 dies!
      maxUnavailable: 1                        # ──► Terminate Pod-v1, release EBS attachment, then start Pod-v2
  selector:
    matchLabels:
      app: postgres-primary
  template:
    metadata:
      labels:
        app: postgres-primary
    spec:
      containers:
      - name: postgres
        image: postgres:16-alpine
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: postgres-data-pvc
```

---

## 4. Failure Modes, Bottlenecks & Troubleshooting

### Failure Mode 1: The Rolling Update Multi-Attach Deadlock
* **Symptom:** New database pod remains stuck in `ContainerCreating`. `kubectl describe pod` outputs:
  ```text
  Warning  FailedAttachVolume  attachdetach-controller
  Multi-Attach error for volume "pvc-0123-4567": Volume is already exclusively attached to one node and can't be attached to another
  ```
* **Root Cause:** A `Deployment` with default rolling update settings created Pod-v2 on Node-B while Pod-v1 was still running on Node-A. AWS EBS block devices cannot be attached to two EC2 hypervisors concurrently.
* **Remedy:** Set `maxSurge: 0` and `maxUnavailable: 1` in the Deployment spec, or transition the workload to a `StatefulSet`.

### Failure Mode 2: Unresponsive Node & Hanging `VolumeAttachment` Finalizers
* **Symptom:** A worker node experiences a hardware crash or network partition. The control plane marks the node `NotReady` and evicts the pod, but the replacement pod on a healthy node hangs forever in `ContainerCreating`.
* **Root Cause:** The `VolumeAttachment` API object has a deletion timestamp, but its finalizer (`external-attacher/ebs-csi-aws-com`) prevents deletion. The `AttachDetachController` refuses to detach the EBS volume because it has not received an unmount acknowledgment from the dead node's `kubelet`. This safety mechanism prevents **split-brain corruption**.
* **Remedy (The 2-Step I/O Fencing / STONITH Procedure):**
  1. *Fence the hardware (AWS Level):* Terminate the dead EC2 instance via AWS CLI:
     ```bash
     aws ec2 terminate-instances --instance-ids i-0deadnode12345
     ```
  2. *Remove the node from Kubernetes:*
     ```bash
     kubectl delete node node-worker-az1a
     ```
     Deleting the Node object informs `AttachDetachController` that the node is gone permanently. It removes the finalizer, calls AWS `ec2:DetachVolume`, and attaches the EBS volume to the healthy node.
  3. *Automated Alternative (K8s 1.26+ GA):* Taint the dead node with the out-of-service taint:
     ```bash
     kubectl taint nodes node-worker-az1a node.kubernetes.io/out-of-service=nodeshutdown:NoExecute
     ```
     The controller automatically bypasses the unmount ACK requirement and force-detaches the volume safely.

### Failure Mode 3: AWS Nitro NVMe Asynchronous Enumeration
* **Symptom:** Pod fails mounting with `timeout waiting for device to appear` or `device or resource busy`.
* **Under the Hood:** Modern AWS EC2 Nitro instances expose EBS volumes as PCIe NVMe block devices (`/dev/nvme1n1`, `/dev/nvme2n1`). Device numbering is non-deterministic across reboots. The AWS EBS CSI driver relies on `udev` rules and `/sbin/ebsnvme-id` to map the AWS Volume ID (`vol-xxxx`) to the active `/dev/nvmeXn1` device.
* **SRE Diagnostic & Triage Commands:**
  ```bash
  # 1. Query the NVMe controller directly on the worker node
  /sbin/ebsnvme-id -v /dev/nvme1n1
  # Output: Volume ID: vol-0987654321fedcba0

  # 2. Inspect active VFS bind mounts linking globalmount to the Pod UID directory
  findmnt -t ext4,xfs | grep kubelet

  # 3. Check for open file handles holding the unmount lock (EBUSY)
  lsof +D /var/lib/kubelet/plugins/kubernetes.io/csi/ebs.csi.aws.com/

  # 4. Verify systemd-udevd status
  systemctl status systemd-udevd
  ```

---

## 5. Socratic Architectural Challenge

### Scenario:
You are the Lead Cloud Architect reviewing an automated database migration.

A development team has an active-active cluster of three web workers that process uploaded image files. They configured a `StorageClass` using AWS EBS `gp3`, provisioned a single `500Gi` PVC with `accessModes: [ ReadWriteOnce ]`, and mounted it to all three pods in their Deployment (`replicas: 3`).

During initial testing on a developer cluster, the application worked flawlessly: all three pods were running, and all three could upload and read images from `/data`.

However, the moment the terraform configuration was applied to the production multi-AZ EKS cluster, two of the three pods permanently crashed with `Multi-Attach error for volume`, and an automated failover test caused silent image file corruption on the surviving pod.

### Architectural Questions:
1. Why did this design appear to succeed in the developer's test cluster, but catastrophically fail in the production multi-AZ cluster?
2. What fundamental filesystem limitation caused silent corruption on the ext4 volume?
3. Which AWS storage service and Kubernetes CSI driver must you mandate in their architectural design to support concurrent read/write access across multiple pods and multiple Availability Zones?

### Architectural Answers:
1. **The Test Environment Illusion:** In the developer's test cluster (likely single-node or a small cluster where all pods landed on the same node), Kubernetes satisfied `ReadWriteOnce` because RWO means **"Single Node"**. The `kubelet` VolumeManager simply executed `mount --bind` three times from the same `/globalmount` directory. In production, `kube-scheduler` spread the pods across multiple nodes in multiple AZs. AWS Nitro rejected attaching the same EBS volume to multiple EC2 instances, triggering the `Multi-Attach error`.
2. **Local Non-Clustered Filesystem Corruption:** Standard filesystems (`ext4`, `XFS`) have no distributed lock manager or cluster-aware journaling. When multiple independent application processes write simultaneously to the same underlying block device without synchronization, inode tables overlap, superblock updates collide, and filesystem corruption occurs.
3. **The Production Fix:** Mandate **AWS EFS (Elastic File System)** provisioned via the **`aws-efs-csi-driver`** with `accessModes: [ ReadWriteMany ]` (RWX). For high-throughput blob/image workloads, re-architect the application to upload directly to **AWS S3** using presigned URLs, decoupling image storage from POSIX filesystem semantics entirely.
