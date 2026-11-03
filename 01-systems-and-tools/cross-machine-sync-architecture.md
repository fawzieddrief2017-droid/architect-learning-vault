# Systems Architecture: Cross-Machine Learning State Synchronization

## 1. Architectural Mental Model

Maintaining a continuous, reproducible systems engineering and cloud architecture learning environment across two distinct physical machines (Home PC and Enterprise Work PC) requires solving a **distributed state synchronization problem across heterogeneous network trust boundaries**.

The architecture decouples **deterministic curriculum state** (architectural specifications, custom tooling, runbooks, and textbook chapters) from **ephemeral host runtimes** (local locks, process IDs, and temporary sockets) using a version-controlled Git repository and atomic filesystem structures.

```
+-------------------------------------------------------------------------------------------------------+
|                                    DISTRIBUTED LEARNING STATE TOPOLOGY                                 |
+-------------------------------------------------------------------------------------------------------+

        HOME PC (Fedora Linux)                                     WORK PC (Enterprise Linux)
   +-------------------------------+                          +-------------------------------+
   | ~/architect-learning-vault/   |                          | ~/architect-learning-vault/   |
   |   ├── 01-systems-and-tools/   |                          |   ├── 01-systems-and-tools/   |
   |   ├── 02-kubernetes-internals/|                          |   ├── 02-kubernetes-internals/|
   |   └── README.md               |                          |   └── README.md               |
   +-------------------------------+                          +-------------------------------+
                   |                                                          ^
                   | [git commit & push]                                      | [git pull --rebase]
                   v                                                          |
+-------------------------------------------------------------------------------------------------------+
|                                 CENTRAL VAULT (GitHub Repository)                                     |
|                                                                                                       |
|  Repository: `architect-learning-vault` (Author: Fawzi Eddrief)                                       |
|   ├── README.md                           (Curriculum index, timeline & topic maps)                   |
|   ├── 01-systems-and-tools/               (Synchronization, debugging pipelines, tooling)             |
|   └── 02-kubernetes-internals/            (Step-by-step kernel and systems engineering textbooks)     |
+-------------------------------------------------------------------------------------------------------+
```

---

## 2. Under the Hood (Mechanics & Boundaries)

### A. State Classification Matrix

| Directory / File | Classification | Synchronization Strategy | Architectural Justification |
| :--- | :--- | :--- | :--- |
| `01-systems-and-tools/` | **Deterministic Content** | Git Tracked | Systems architecture, diagnostics, sync pipelines. |
| `02-kubernetes-internals/` | **Deterministic Content** | Git Tracked | Core engineering masterclass chapters and packet walks. |
| `README.md` | **Index & Roadmap** | Git Tracked | Master curriculum map and progress tracking. |
| Local build artifacts | **Ephemeral Runtime** | **Excluded (`.gitignore`)** | Sockets, temporary files, and local build caches. |

### B. Enterprise Egress Traversal (Port 443 SSH Multiplexing)
In restricted environments where outbound egress on standard SSH port 22 is blocked by enterprise perimeter firewalls, Git operations are routed over port 443 via `~/.ssh/config`:

```sshconfig
Host github.com
    Hostname ssh.github.com
    Port 443
    User git
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 30
```

---

## 3. Operational Runbook

1. **At the end of a session on Machine A:**
   ```bash
   cd ~/architect-learning-vault
   git add -A
   git commit -m "docs: session $(date +'%Y-%m-%d') learning delta"
   git push origin main
   ```
2. **At the start of a session on Machine B:**
   ```bash
   cd ~/architect-learning-vault
   git pull --rebase origin main
   ```
