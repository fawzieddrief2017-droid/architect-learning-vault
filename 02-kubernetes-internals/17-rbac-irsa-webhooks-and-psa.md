# RBAC, Workload Identity (IRSA vs. EKS Pod Identity) & Admission Governance

## 1. Architectural Mental Model (The "Why" & "What")

In production Kubernetes and AWS cloud architecture, security is governed by **cryptographic identity, strict API boundary enforcement, and zero-trust blast-radius containment**. Unlike traditional virtual machines where an application inherits the host's EC2 Instance Profile, multi-tenant clusters host dozens of microservices on the same worker node. If a single compromised container can query the EC2 Instance Metadata Service (`169.254.169.254`), or if a misconfigured Admission Webhook stalls `kube-system`, the entire AWS account or cluster is compromised.

Every request entering the `kube-apiserver` passes through a deterministic **3-Stage Security Gate** before persistence in `etcd`, while runtime cloud access is governed by **short-lived, audience-bound projected JWTs** exchanged with AWS STS.

```
+─────────────────────────────────────────────────────────────────────────────────────────────────────────────+
|                                 KUBE-APISERVER REQUEST SECURITY PIPELINE                                    |
+─────────────────────────────────────────────────────────────────────────────────────────────────────────────+

  CLIENT (kubectl / kubelet / Pod App)
        │  HTTPS (TLS 1.3 + Client Cert OR Bearer JWT)
        ▼
  ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────┐
  │ STAGE 1: AUTHENTICATION (Who are you?)                                                                  │
  │   • X.509 Client Certs: kubelet (CN=system:node:node-1), cluster-admin                                  │
  │   • OIDC / AWS IAM Token: Human operators (aws eks get-token)                                           │
  │   • Bound ServiceAccount JWT: Pods (/var/run/secrets/kubernetes.io/serviceaccount/token)                │
  │   ──► Failure: HTTP 401 Unauthorized                                                                    │
  └─────────────────────────────────────────────────────────┬───────────────────────────────────────────────┘
                                                            │ Identity Attached (User + Groups)
                                                            ▼
  ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────┐
  │ STAGE 2: AUTHORIZATION — RBAC (Are you allowed to perform this verb on this resource?)                  │
  │   • Evaluates: Subject (User/Group/ServiceAccount) + Verb (get, list, watch, create) + Resource         │
  │   • Purely ADDITIVE (Default Deny; NO explicit Deny rules exist in Kubernetes RBAC!)                     │
  │   ──► Failure: HTTP 403 Forbidden                                                                       │
  └─────────────────────────────────────────────────────────┬───────────────────────────────────────────────┘
                                                            │ Only for mutating verbs (CREATE/UPDATE/DELETE)
                                                            ▼
  ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────┐
  │ STAGE 3: ADMISSION CONTROL (Is the payload safe and policy-compliant?)                                  │
  │   • Mutating Webhooks (Serial): Inject sidecars, IRSA env vars, default security contexts               │
  │   • Schema Validation: OpenAPI structural checks                                                        │
  │   • Validating Webhooks & PSA (Parallel): Reject privileged containers, enforce Pod Security Standards  │
  │   ──► Failure: HTTP 403 Forbidden / HTTP 422 Unprocessable Entity                                       │
  └─────────────────────────────────────────────────────────┬───────────────────────────────────────────────┘
                                                            │ Approved!
                                                            ▼
                                                 [ COMMITTED TO ETCD ]
```

---

## 2. Under the Hood: Mechanics & Internals

### A. RBAC Scoping & Hidden Privilege Escalation Vectors
Kubernetes has no internal human user database; human identities are always federated externally (OIDC, AWS IAM, client certificates), while workloads authenticate via `ServiceAccount` objects.

| Resource | Scope | Architectural Use Case |
| :--- | :--- | :--- |
| **`Role` + `RoleBinding`** | Single Namespace | Microservice reading ConfigMaps in `payments`; CI/CD runner deploying to `staging`. |
| **`ClusterRole` + `RoleBinding`** | Reusable template bound to 1 Namespace | Define a standard `team-developer` `ClusterRole` once, then bind it per team namespace. |
| **`ClusterRole` + `ClusterRoleBinding`** | Cluster-wide (All Namespaces + Nodes/PVs) | `aws-load-balancer-controller` watching `EndpointSlices` and `Nodes` cluster-wide. |

#### Critical RBAC Escalation Vectors:
1. **`list` / `watch` on `secrets`:** Grants ability to dump **every** Secret in the namespace in one API response—even if `resourceNames` filtering is intended.
2. **`create` on `pods/exec`:** Equivalent to interactive shell access inside every container in that namespace, exposing mounted tokens and in-memory secrets.
3. **`escalate`, `bind`, `impersonate`:** Bypass Kubernetes' built-in privilege-escalation check (which normally prevents a user from creating a Role with more permissions than they already hold).

---

### B. Bound Projected ServiceAccount Tokens & AWS IRSA

In legacy Kubernetes (< v1.21), `ServiceAccount` tokens were stored as permanent, non-expiring `Secret` objects in `etcd`. Modern Kubernetes uses the **`TokenRequest` API** to project short-lived, cryptographically bound JWTs into `tmpfs`:

* **`iss` (Issuer):** Cluster OIDC discovery URL.
* **`sub` (Subject):** `system:serviceaccount:<namespace>:<serviceaccount-name>`.
* **`aud` (Audience):** Explicitly scoped (e.g., `sts.amazonaws.com` for AWS STS, preventing token replay against `kube-apiserver`).
* **`kubernetes.io` Pod Binding:** Tied directly to the Pod's `uid`; invalidated the instant the Pod is deleted, and auto-rotated by `kubelet` at 80% of its TTL.

```
+─────────────────────────────────────────────────────────────────────────────────────────────────────────────+
|                              AWS IRSA: OIDC FEDERATED TOKEN EXCHANGE FLOW                                   |
+─────────────────────────────────────────────────────────────────────────────────────────────────────────────+

  Pod YAML (serviceAccountName: checkout-sa)
        │
        ▼
  [ EKS Pod Identity Mutating Webhook ]
  Injects:
    • AWS_ROLE_ARN=arn:aws:iam::123456789012:role/checkout-s3-role
    • AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
    • Projected tmpfs token volume (aud: sts.amazonaws.com)
        │
        ▼
  App AWS SDK calls: sts:AssumeRoleWithWebIdentity(RoleArn, WebIdentityToken=JWT)
        │
        ▼
  AWS STS fetches public JWKS from cluster OIDC Discovery URL, verifies RSA signature + 'sub'/'aud'
  against IAM Role Trust Policy, and returns temporary STS credentials (AccessKey, SecretKey, SessionToken).
```

---

### C. EKS Pod Identity vs. IRSA (Multi-Cluster Scale)

While IRSA works well for small cluster counts, it hits a hard wall in large enterprises:
* Every EKS cluster has a **unique OIDC Provider URL**.
* Sharing a single IAM Role across 50 EKS clusters requires listing 50 OIDC ARNs in the IAM Role Trust Policy—rapidly hitting AWS's **4,096-character (4KB) Trust Policy hard limit**.

**EKS Pod Identity** eliminates OIDC trust-policy sprawl completely:

| Architectural Dimension | IRSA (OIDC Federation) | EKS Pod Identity (Modern Standard) |
| :--- | :--- | :--- |
| **IAM Trust Policy Principal** | `Federated`: `arn:aws:iam::...:oidc-provider/oidc.eks...` (Unique per cluster) | `Service`: `pods.eks.amazonaws.com` (**Identical across all clusters!**) |
| **4KB Trust Policy Ceiling** | Hits limit at ~15–25 clusters sharing one role | **Unlimited clusters** (1 trust policy) |
| **Credential Exchange Path** | Pod SDK $\rightarrow$ HTTPS directly to AWS STS (`AssumeRoleWithWebIdentity`) | Pod SDK $\rightarrow$ Node-local `eks-pod-identity-agent` (`169.254.170.23:80`) $\rightarrow$ `sts:AssumeRoleForPodIdentity` |
| **ABAC Session Tags** | Not automatically injected | Automatically injects `eks-cluster-name`, `kubernetes-namespace`, `kubernetes-service-account` |

---

### D. Admission Control, Webhooks & Pod Security Admission (PSA)

After RBAC approves *who* is calling, Admission Control inspects *what* is being submitted:
1. **Mutating Webhooks (Run Serially):** Can rewrite the object (e.g., Istio sidecar injection, EKS IRSA env var injection).
2. **Object Schema Validation:** Validates against OpenAPI spec.
3. **Validating Webhooks & PSA (Run in Parallel):** Read-only Allow/Deny gate.

#### Pod Security Admission (PSA) Profiles:
Applied via namespace labels (`pod-security.kubernetes.io/enforce`, `audit`, `warn`):
* **`privileged`:** Unrestricted (reserved for `kube-system` CNI/CSI DaemonSets).
* **`baseline`:** Blocks known host escalations (`hostNetwork`, `hostPID`, `privileged: true`, `hostPath`).
* **`restricted`:** Enterprise standard (`runAsNonRoot: true`, `allowPrivilegeEscalation: false`, `capabilities.drop: ["ALL"]`, `seccompProfile.type: RuntimeDefault`).

---

## 3. Production Implementation: Hardened EKS Identity & Admission Manifests

### A. EC2 IMDSv2 Hop-Limit Hardening (Terraform Launch Template)
Prevents Pods from traversing the `veth` hop to steal the worker node's EC2 Instance Profile at `169.254.169.254`:

```hcl
resource "aws_launch_template" "eks_nodes" {
  name_prefix = "eks-hardened-node-"

  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required" # ──► Enforces IMDSv2 session tokens
    http_put_response_hop_limit = 1          # ──► CRITICAL: Blocks Pods from reaching 169.254.169.254!
  }
}
```

### B. Pod Security Admission (`restricted`) & Safe Validating Webhook

```yaml
# 1. Enforce Pod Security Standards at the Namespace level
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
---
# 2. Safe ValidatingWebhookConfiguration excluding kube-system to prevent deadlocks
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: production-policy-guard
webhooks:
- name: validate.policies.corp.io
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 5                          # ──► Tight timeout prevents API server thread exhaustion
  failurePolicy: Fail
  namespaceSelector:
    matchExpressions:
    - key: kubernetes.io/metadata.name
      operator: NotIn
      values: ["kube-system", "kyverno"]     # ──► CRITICAL: Never block control plane or webhook pods!
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE", "UPDATE"]
    resources: ["pods"]
  clientConfig:
    service:
      name: policy-webhook-svc
      namespace: kyverno
      path: /validate
```

---

## 4. Failure Modes, Bottlenecks & Troubleshooting

### Failure Mode 1: The Self-Locking Admission Webhook Outage
* **Symptom:** All `kubectl apply` and HPA/Deployment scaling operations fail across the entire cluster with:
  ```text
  Error from server (InternalError): failed calling webhook "validate.policies.corp.io": context deadline exceeded
  ```
* **Root Cause:** A webhook with `failurePolicy: Fail` lacks a `namespaceSelector` excluding `kube-system` and its own namespace. When the webhook Pods crash or scale, the `kube-apiserver` blocks creation of the replacement webhook Pods because they must pass through the dead webhook first.
* **Emergency Triage:**
  ```bash
  # Inspect failing webhook configurations
  kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations

  # Emergency break-glass: delete or patch failurePolicy to Ignore
  kubectl delete validatingwebhookconfiguration production-policy-guard
  ```

### Failure Mode 2: Silent IAM Fallback to Node Instance Profile
* **Symptom:** A Pod without an IRSA annotation or Pod Identity association unexpectedly succeeds in writing to S3/DynamoDB in dev/prod.
* **Root Cause:** `http_put_response_hop_limit` on the EC2 worker node was left at AWS's default value of `2`, allowing the AWS SDK inside the Pod to cross the `veth` pair and fetch the worker node's EC2 Instance Profile from `169.254.169.254`.
* **Verification:**
  ```bash
  # Test from inside a non-privileged Pod (must time out or fail!)
  kubectl exec -it test-pod -- curl -s -m 2 -X PUT "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"
  ```

---

## 5. Socratic Architectural Challenge

### Scenario:
You harden an EKS worker node Launch Template with `http_put_response_hop_limit = 1` (IMDSv2) and deploy a payment Pod configured to use IRSA (`checkout-sa`).

During a late-night hotfix, an engineer deploys a new `payments` microservice Pod, but forgets to annotate `checkout-sa` with `eks.amazonaws.com/role-arn`.

### Architectural Questions:
1. When the Pod attempts to read an S3 bucket using the standard AWS SDK, does it silently fall back to the worker node's EC2 IAM Role, or does it fail?
2. Trace the exact network and SDK mechanism that produces this outcome.

### Architectural Answers:
1. **It fails safely (`NoCredentialProviders` / `Unable to locate credentials`).**
2. **Under the Hood:** Because the `ServiceAccount` lacks the `eks.amazonaws.com/role-arn` annotation, the EKS Mutating Webhook does not inject `AWS_ROLE_ARN` or `AWS_WEB_IDENTITY_TOKEN_FILE`. The AWS SDK walks its fallback chain down to EC2 IMDSv2 (`169.254.169.254`). Because `http_put_response_hop_limit = 1` is enforced on the EC2 instance, crossing the Pod's `veth` interface consumes 1 hop and drops the IP TTL to 0 at the hypervisor boundary—preventing silent privilege escalation to the node's IAM Role.
