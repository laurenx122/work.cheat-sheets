# AWS EKS Cheatsheet

> Companion to your CKAD cheatsheet. Same idea: big picture first, then drill into every layer with concept tables before commands, and worked examples instead of blank templates.

---

## 0. The Big Picture — How Everything Connects

```
AWS Account
└─ VPC (network boundary for the whole cluster)
    ├─ Subnets (public + private, spread across ≥2 AZs)      -> where ENIs/nodes/pods live
    ├─ Internet Gateway (IGW)                                 -> public subnet internet access
    ├─ NAT Gateway (in public subnet)                         -> private subnet outbound internet
    ├─ Route Tables                                           -> public rt -> IGW, private rt -> NAT
    └─ Security Groups                                        -> firewall at ENI/node level

EKS Cluster (control plane, AWS-managed)
    ├─ API Server         -> fronts kubectl/eksctl/console, HA across AZs, AWS-managed & patched
    ├─ etcd                -> AWS-managed, encrypted, multi-AZ, you never touch it directly
    ├─ Scheduler / Controller Manager -> AWS-managed
    ├─ OIDC Provider        -> issued per-cluster -> backbone of IRSA / Pod Identity
    └─ Cluster Endpoint     -> public, private, or public+private (controls where API is reachable from)
        └─ aws-auth ConfigMap / EKS Access Entries -> maps IAM identities -> Kubernetes RBAC

    Data Plane (where your workloads actually run) — one cluster can mix all three:
    ├─ Self-Managed Nodes  -> you create/manage EC2 + ASG yourself, full control
    ├─ Managed Node Groups -> AWS manages EC2 lifecycle (create/update/drain/terminate) via ASG
    └─ Fargate Profiles    -> serverless, no nodes to manage, 1 pod = 1 micro-VM

    Namespace (isolation boundary, same as vanilla K8s)
    └─ Deployment / StatefulSet / DaemonSet / Job / CronJob
        └─ Pod(s)
            ├─ Container(s)      -> images, env, resources
            ├─ ENI + IP           -> from VPC CNI, pod gets a real VPC-routable IP
            ├─ ServiceAccount     -> identity; annotated with IAM role (IRSA) or linked via Pod Identity
            └─ EBS/EFS volume     -> via CSI driver, backed by real AWS storage

    Networking layer
    ├─ VPC CNI (amazon-vpc-cni-k8s)  -> assigns pods real VPC IPs from ENIs (ENI + prefix delegation)
    ├─ CoreDNS                       -> in-cluster DNS, runs as a Deployment (EKS add-on)
    ├─ kube-proxy                    -> Service IP routing (EKS add-on)
    ├─ Service (ClusterIP/NodePort/LoadBalancer) -> stable virtual IP -> routes to pod IPs by label
    ├─ AWS Load Balancer Controller  -> watches Ingress/Service -> provisions real ALB/NLB
    │   ├─ Ingress  -> creates ALB (Layer 7, HTTP/S)
    │   └─ Service type=LoadBalancer -> creates NLB (Layer 4, TCP/UDP)
    ├─ Gateway API (newer) -> Gateway + HTTPRoute -> also provisioned by LB Controller
    ├─ VPC Lattice          -> app-to-app networking/mesh across VPCs & accounts, no peering needed
    └─ NetworkPolicy         -> firewall rules BETWEEN pods (enforced by CNI or Calico)

    Storage layer
    ├─ EBS CSI Driver -> PVC -> dynamic EBS volume -> ReadWriteOnce, single-AZ, block storage
    ├─ EFS CSI Driver -> PVC -> EFS access point -> ReadWriteMany, multi-AZ, NFS storage
    └─ StorageClass    -> defines which CSI driver + parameters (volume type, filesystem, etc.)

    Secrets layer
    ├─ Kubernetes Secrets (base64, not encrypted at rest by default)
    ├─ EKS Envelope Encryption (KMS)  -> encrypts etcd secrets with your own CMK
    └─ External Secrets / Secrets Manager CSI driver -> pulls from AWS Secrets Manager / SSM Parameter Store

    Compute scaling layer
    ├─ Cluster Autoscaler   -> scales EC2 Node Group ASGs based on unschedulable pods
    ├─ Karpenter            -> faster, node-type-aware, provisions right-sized nodes directly (no ASG needed)
    └─ HPA / VPA             -> scales pod replicas / pod resource requests

    Identity layer
    ├─ IAM Roles for Service Accounts (IRSA) -> ServiceAccount annotation + OIDC trust -> pod assumes IAM role
    └─ EKS Pod Identity (newer, simpler)      -> Pod Identity Association -> no OIDC annotation dance needed

    Observability / lifecycle layer
    ├─ CloudWatch Container Insights -> metrics/logs for cluster, nodes, pods
    ├─ EKS Add-ons                    -> AWS-managed lifecycle for CNI, CoreDNS, kube-proxy, EBS/EFS CSI, etc.
    └─ Upgrade cycle                  -> control plane first, then add-ons, then node groups (in that order)

RBAC layer (who can do what, same as vanilla K8s, but identity now comes from IAM)
    IAM User/Role -> aws-auth ConfigMap or Access Entry -> maps to K8s Group -> RoleBinding/ClusterRoleBinding -> Role/ClusterRole
```

**How to read this diagram top to bottom:** AWS builds the *network* (VPC) → AWS builds the *control plane* (EKS cluster) → you attach *compute* (nodes/Fargate) → Kubernetes objects run *on* that compute → CNI/LB Controller/CSI drivers are the glue that make Kubernetes networking and storage actually manifest as real AWS resources (ENIs, ALBs, EBS volumes) → IAM (via IRSA/Pod Identity) is how pods get real AWS permissions → aws-auth/Access Entries is how real AWS identities get Kubernetes permissions.

---

## 1. Introduction

### Course Introduction (concept, not a technical topic)

EKS (Elastic Kubernetes Service) is AWS's **managed Kubernetes control plane**. AWS runs and patches the API server, etcd, scheduler, and controller-manager for you across multiple AZs. You are still responsible for the **data plane** (worker nodes/Fargate), **cluster configuration**, **networking (VPC/CNI)**, **IAM integration**, and **everything above the Kubernetes API** (workloads, Helm charts, add-ons).

**Shared responsibility split, memorize this:**

| AWS manages | You manage |
|---|---|
| API server, etcd, scheduler, controller-manager | Worker nodes (unless full Fargate) |
| Control plane HA across AZs | Node OS patching (self-managed nodes) |
| Control plane upgrades (you trigger, AWS executes) | Application workloads, Helm charts |
| Control plane logging plumbing (you enable it) | VPC, subnets, security groups design |
| etcd backups & encryption infra | IAM roles, RBAC, aws-auth/Access Entries |
| | Add-on versions (unless "auto" mode) |
| | Cluster & node group version upgrades (you trigger) |

---

## 2. EKS Fundamentals

### 2.1 What is EKS

- EKS = AWS's managed control plane for **upstream, CNCF-conformant Kubernetes**. Any standard `kubectl`/Helm/K8s manifest works unmodified.
- You pay: (1) a flat **per-cluster hourly control plane fee**, (2) compute costs for whatever data plane you choose (EC2 nodes, Fargate vCPU/memory), (3) any add-on-specific costs (e.g., NLB/ALB hourly + LCU charges).
- Control plane runs across **a minimum of 2 (usually 3) AZs** for HA automatically — you don't configure this, AWS does it.
- EKS Distro (EKS-D) — the same Kubernetes distribution AWS uses for EKS, but you can run it **anywhere** (on-prem, other clouds) yourself, unmanaged.
- EKS Anywhere — AWS-supported tooling to run EKS-D **on your own infrastructure** with AWS support contracts, for on-prem/hybrid needs.

### 2.2 Common Use Cases

| Use case | Why EKS fits |
|---|---|
| Microservices platforms | Native Service Discovery, Ingress, autoscaling |
| Batch / ML workloads | Job/CronJob + Karpenter for fast burst-scaling GPU nodes |
| Hybrid/multi-cloud | EKS Anywhere + EKS Connector to unify management |
| Cost-sensitive spiky workloads | Fargate (pay per pod) or Spot-backed node groups |
| Regulated workloads | Private endpoint + KMS envelope encryption + VPC-only networking |
| CI/CD & dev/test clusters | Fast to spin up/down, IaC-friendly (Terraform/eksctl/CDK) |

### 2.3 Architecture (deep dive)

**Control plane components (all AWS-managed, run in an AWS-owned VPC, not yours):**

| Component | Role |
|---|---|
| kube-apiserver | Front door for all API calls; HA, load-balanced, multi-AZ |
| etcd | Cluster state store; AWS-managed, encrypted, automatically backed up, multi-AZ |
| kube-scheduler | Assigns pods to nodes |
| kube-controller-manager | Runs core controllers (ReplicaSet, Node, etc.) |
| cloud-controller-manager | AWS-specific controller logic (ELB provisioning triggers, node lifecycle) |

**Data plane components (run in YOUR VPC, on YOUR nodes or Fargate):**

| Component | Role |
|---|---|
| kubelet | Node agent; talks to API server, runs containers |
| kube-proxy | Programs iptables/ipvs rules for Service routing (an EKS add-on) |
| VPC CNI (aws-node) | Assigns real VPC IPs to pods (an EKS add-on) |
| CoreDNS | Cluster DNS (an EKS add-on) |
| Container runtime | containerd (default since EKS 1.24+, dockershim removed) |

**Connectivity:** Nodes reach the control plane over the **cluster endpoint** (public and/or private). The control plane reaches back into your VPC via **cross-account ENIs** that EKS creates in your subnets during cluster creation — this is why EKS needs specific subnet tags and IAM permissions at cluster-creation time.

### 2.4 Deployment Options

| Option | Nodes managed by | Best for |
|---|---|---|
| **Self-managed nodes** | You (EC2 + your own ASG + AMI + userdata) | Max customization (custom AMI, kernel tuning, GPU drivers) |
| **Managed Node Groups (MNG)** | AWS (creates/updates/drains/terminates EC2 via ASG) | Default choice for most teams; rolling upgrades handled for you |
| **Fargate** | AWS (no EC2 visible to you at all) | Simplicity, security isolation per-pod, spiky/unpredictable workloads |
| **EKS Auto Mode** (newer) | AWS (fully automates node provisioning, scaling, AMI, security patching, plus built-in networking/storage/LB) | "Just run workloads" — minimal ops overhead, opinionated defaults |
| **EKS Anywhere** | You, on your own infra (on-prem/bare metal/vSphere) | Air-gapped/hybrid/regulatory requirements |

**Rule of thumb:** MNG for general workloads, Fargate for isolated/serverless/bursty workloads, both can coexist in the same cluster via different namespaces/Fargate profiles.

### 2.5 Tools needed for EKS

| Tool | Purpose |
|---|---|
| `aws` CLI (v2) | Talks to AWS APIs; generates the kubeconfig token |
| `kubectl` | Talks to the Kubernetes API |
| `eksctl` | Opinionated CLI that wraps CloudFormation to create clusters/node groups/Fargate profiles fast |
| `helm` | Package manager for installing add-ons like LB Controller, metrics-server, Karpenter |
| Terraform (`terraform-aws-modules/eks`) | IaC-managed clusters, most common in production |
| AWS Console | GUI, good for inspection, not ideal for repeatable provisioning |
| `aws-iam-authenticator` (legacy) / built into `aws eks get-token` now | Converts IAM identity into a K8s auth token |

```bash
# install/verify tools (Git Bash / Linux)
aws --version
kubectl version --client
eksctl version
helm version

# configure kubectl to talk to an existing cluster
aws eks update-kubeconfig --region us-east-1 --name my-cluster

# quick smoke test
kubectl get nodes
kubectl get svc -A
```

### 2.6 Networking (fundamentals level — see section 3 for deep dive)

- Every pod gets a **real, routable VPC IP address** (not an overlay network) — this is the single biggest architectural difference from most other K8s networking setups (Calico overlay, Flannel, etc.).
- This is enabled by the **Amazon VPC CNI plugin**, which attaches ENIs (Elastic Network Interfaces) to nodes and assigns secondary IPs from those ENIs to pods.
- Consequence: your **VPC subnet size directly limits how many pods you can run** (IP exhaustion is a real, common EKS problem — solved via prefix delegation, secondary CIDRs, or custom networking).

### 2.7 Authentication

Two eras, know both because you'll see both in the wild:

**Old model — `aws-auth` ConfigMap** (still supported):
```bash
kubectl edit configmap aws-auth -n kube-system
```
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/eks-node-role
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
  mapUsers: |
    - userarn: arn:aws:iam::123456789012:user/lauren
      username: lauren
      groups:
        - system:masters
```

**New model — EKS Access Entries** (2023+, recommended now, no more editing a ConfigMap):
```bash
# grant an IAM principal cluster access
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:user/lauren \
  --type STANDARD

# attach a built-in access policy (like RBAC, but AWS-managed)
aws eks associate-access-policy \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:user/lauren \
  --access-scope type=cluster \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy

# list who has access
aws eks list-access-entries --cluster-name my-cluster
```

**Auth flow to remember (end to end):**
1. `kubectl` calls `aws eks get-token` → generates a short-lived signed token using your IAM credentials.
2. API server validates that token against IAM (via the authenticator webhook, built-in now).
3. API server checks **aws-auth ConfigMap or Access Entries** → maps that IAM identity to a Kubernetes `username`/`groups`.
4. Kubernetes RBAC (`RoleBinding`/`ClusterRoleBinding`) then decides what that `username`/`groups` can actually **do**.

> IAM authenticates ("who are you"). Kubernetes RBAC authorizes ("what can you do"). You need BOTH configured or access fails.

---

## 3. EKS Networking

### 3.1 How networking works

- **VPC CNI (`aws-node` DaemonSet)** runs on every node, manages ENIs and IP address pools.
- Each EC2 instance type has a max **ENI count** and **IPs-per-ENI** limit (bigger instance = more pod capacity). This is the #1 cause of "pod stuck in Pending with no clear reason" on EKS — you've hit the node's max pod count.
- **Warm IP pool**: CNI pre-allocates spare IPs/ENIs on each node so new pods start fast instead of waiting on an EC2 API call.
- `kube-proxy` still handles Service (ClusterIP) routing exactly like vanilla K8s (iptables/ipvs mode) — CNI only affects **pod IP assignment**, not Service routing.

```bash
# see max pods per node type math
kubectl get nodes -o custom-columns=NAME:.metadata.name,PODS:.status.allocatable.pods

# check CNI plugin version/logs
kubectl get daemonset aws-node -n kube-system
kubectl logs -n kube-system -l k8s-app=aws-node --tail=50
```

### 3.2 Prefix Delegation

- Default CNI mode assigns **one secondary IP per ENI slot** → wastes IPs and hits limits fast on small subnets.
- **Prefix Delegation** assigns a `/28` **IPv4 prefix (16 IPs)** per ENI slot instead of a single IP → dramatically increases pods-per-node (e.g., an `m5.large` goes from ~29 pods to 100+ pods).
- Enabled via an environment variable on the CNI add-on:

```bash
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true
kubectl set env daemonset aws-node -n kube-system WARM_PREFIX_TARGET=1
```

- Trade-off: requires a **contiguous free `/28` block** in the subnet — small/fragmented subnets can fail to allocate. Best practice: use dedicated subnets sized for prefix delegation.

### 3.3 IPv6

- EKS supports **IPv6-only pod/service networking** (cluster created as IPv6 from the start — cannot convert an existing IPv4 cluster).
- Each pod still gets a real routable IP, just from an IPv6 `/64` **prefix delegated per ENI** (prefix delegation is mandatory/automatic in IPv6 mode, not optional).
- Nodes and pods get IPv6 addresses; **NAT64/DNS64** is used for pods reaching legacy IPv4-only external endpoints.
- Why choose it: solves IP exhaustion permanently, future-proofs large clusters. Downside: your whole toolchain (LB Controller, on-prem integrations, monitoring) needs IPv6 readiness.

```bash
eksctl create cluster --name my-cluster --ip-family IPv6
```

### 3.4 Network Policies

| Layer | What it controls |
|---|---|
| Security Group | Firewall at **node/ENI** level (coarse, AWS-level) |
| NetworkPolicy | Firewall **between pods** (fine-grained, Kubernetes-level) |

- Plain EKS **VPC CNI now supports NetworkPolicy enforcement natively** (since CNI v1.14+, no Calico needed) — must be explicitly enabled.
- Still deny-by-default logic: once ANY NetworkPolicy selects a pod, all traffic not explicitly allowed is denied.

```bash
# enable network policy enforcement on the VPC CNI add-on
kubectl set env daemonset aws-node -n kube-system ENABLE_NETWORK_POLICY=true
```

```yaml
# example: only allow frontend -> backend on port 8080, deny everything else to backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

### 3.5 Network Policies Demo (scenario walkthrough)

```bash
# 1. confirm feature is on
kubectl get daemonset aws-node -n kube-system -o yaml | grep ENABLE_NETWORK_POLICY

# 2. apply a default-deny for a namespace
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: prod
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
EOF

# 3. test — this should now time out
kubectl run test --rm -it --image=busybox -n prod -- wget -qO- --timeout=3 backend-svc

# 4. apply the allow rule (from example above), retest, should succeed
kubectl apply -f backend-allow-frontend.yaml
```

---

## 4. EKS Storage

### 4.1 EBS (Elastic Block Store)

| Trait | Value |
|---|---|
| Access mode | `ReadWriteOnce` only (one node at a time) |
| Scope | Single AZ — pod must be scheduled in same AZ as the volume |
| Use case | Databases, single-writer stateful workloads (StatefulSets) |
| Driver | `ebs-csi-driver` (EKS-managed add-on) |
| Needs | IAM permissions via IRSA/Pod Identity attached to the CSI driver's ServiceAccount |

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer   # delays provisioning until pod is scheduled -> picks correct AZ
parameters:
  type: gp3
  encrypted: "true"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: ebs-gp3
  resources:
    requests:
      storage: 20Gi
```

```bash
kubectl apply -f ebs-storageclass.yaml
kubectl get pvc db-data
kubectl describe pvc db-data     # look for "Successfully provisioned volume" event
```

> `volumeBindingMode: WaitForFirstConsumer` is the #1 gotcha — without it, EBS may provision the volume in an AZ that has no available node, and the pod sticks in Pending forever.

### 4.2 EFS (Elastic File System)

| Trait | Value |
|---|---|
| Access mode | `ReadWriteMany` (many pods, many nodes, simultaneously) |
| Scope | Multi-AZ (NFS-based, regional) |
| Use case | Shared config/content across replicas, CMS uploads, shared ML datasets |
| Driver | `efs-csi-driver` (EKS-managed add-on) |
| Needs | An EFS filesystem + mount targets in each subnet/AZ pre-created in AWS first |

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap        # dynamic provisioning via access points
  fileSystemId: fs-0123456789abcdef0
  directoryPerms: "700"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-uploads
spec:
  accessModes: ["ReadWriteMany"]
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

### 4.3 EKS Other Storage

| Type | Notes |
|---|---|
| `emptyDir` | Node-local scratch space, deleted when pod dies — no AWS backing needed |
| `hostPath` | Direct node filesystem access — rarely appropriate, security risk |
| FSx for Lustre CSI driver | High-performance parallel filesystem for HPC/ML training jobs |
| FSx for NetApp ONTAP / OpenZFS | Enterprise NAS features (snapshots, dedup) via their own CSI drivers |
| S3 (not a CSI volume) | Access via SDK/`aws s3` from app code, or `s3fs`/Mountpoint for S3 CSI driver for file-like access |

**Decision table:**

| Need | Use |
|---|---|
| Single-writer, fast block storage | EBS |
| Shared read/write across many pods | EFS |
| Object storage, app-level access | S3 (SDK, not a mounted volume normally) |
| HPC/ML massive throughput | FSx for Lustre |
| Scratch space only | emptyDir |

---

## 5. EKS Secrets

### 5.1 EKS Secrets Intro

- Kubernetes `Secret` objects are **base64-encoded, NOT encrypted**, by default in etcd — base64 is encoding, not security.
- EKS lets you enable **envelope encryption**: etcd-stored secrets get encrypted at rest using a **KMS Customer Managed Key (CMK)** you control.

```bash
# enable secrets envelope encryption on an existing cluster (via console/eksctl/terraform)
eksctl utils enable-secrets-encryption \
  --cluster my-cluster \
  --key-arn arn:aws:kms:us-east-1:123456789012:key/abcd-1234
```

- This does NOT change how you create/use Secrets in `kubectl` — it's transparent to the app. It only protects the etcd-at-rest copy (and anyone dumping an etcd snapshot without KMS access).
- Note: enabling this **does not retroactively encrypt already-existing secrets** — existing ones stay unencrypted until they're next written/rotated.

### 5.2 Kubernetes Secrets Options

| Option | How it works | When to use |
|---|---|---|
| Native K8s `Secret` | `kubectl create secret ...`, base64 in etcd | Simple, low-sensitivity, dev/test |
| Native + KMS envelope encryption | Adds at-rest encryption to the above | Any production cluster, minimum bar |
| AWS Secrets Manager + CSI driver | Secret lives in Secrets Manager, mounted into pod as a volume at runtime, never stored in etcd at all | Production secrets, rotation needed |
| SSM Parameter Store + CSI driver | Same idea, cheaper, less feature-rich than Secrets Manager (no automatic rotation) | Config values, cheaper secrets |
| External Secrets Operator (ESO) | Syncs external secret stores INTO native K8s Secrets automatically | GitOps-friendly, want native Secret objects still |

```bash
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password='S3cur3P@ss'

kubectl get secret db-creds -o jsonpath='{.data.username}' | base64 -d
```

```yaml
# mounting from AWS Secrets Manager via the Secrets Store CSI Driver
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: db-creds-spc
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "prod/db/creds"
        objectType: "secretsmanager"
```

---

## 6. Load Balancers

### 6.1 Load Balancers Intro

The **AWS Load Balancer Controller (LBC)** is the piece of glue software (a Deployment you install, usually via Helm) that watches Kubernetes `Ingress`/`Service`/`Gateway` objects and provisions **real AWS ALB/NLB resources** to match.

| K8s object | AWS resource created | Layer |
|---|---|---|
| `Service` type `LoadBalancer` | Network Load Balancer (NLB) | L4 (TCP/UDP) |
| `Ingress` | Application Load Balancer (ALB) | L7 (HTTP/HTTPS) |
| Gateway API `Gateway`+`HTTPRoute` | ALB or NLB depending on route type | L4 or L7 |

```bash
# install the controller (once per cluster) via Helm
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port:
                  number: 80
```

```bash
kubectl apply -f web-ingress.yaml
kubectl get ingress web-ingress   # ADDRESS column fills in with ALB DNS name once provisioned
```

**Target types, know the difference:**

| Target type | Traffic goes to | Requires |
|---|---|---|
| `instance` | Node's `NodePort`, then kube-proxy forwards to pod | Works with any CNI, extra hop |
| `ip` | Directly to the pod's VPC IP | Requires VPC CNI (pod has real VPC IP) — fewer hops, standard on EKS |

### 6.2 Gateway Ingress

- **Gateway API** is the newer, more expressive successor to `Ingress` (multi-protocol, more granular routing, role-split between infra owners and app owners).
- Objects: `GatewayClass` (defines who provisions it — e.g., LBC) → `Gateway` (the actual listener/LB) → `HTTPRoute`/`TCPRoute`/`GRPCRoute` (routing rules attached to a Gateway).
- Still provisioned by the **same AWS Load Balancer Controller**, just via a different, more flexible API surface than `Ingress`.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: aws-alb
  listeners:
    - name: http
      protocol: HTTP
      port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
spec:
  parentRefs:
    - name: my-gateway
  rules:
    - backendRefs:
        - name: web-svc
          port: 80
```

### 6.3 VPC Lattice

- A fully AWS-native **application networking service** — think "service mesh without running a mesh." Works **across VPCs, accounts, and even EKS + ECS + EC2** simultaneously.
- No VPC peering, no Transit Gateway, no CNI-level tricks needed — Lattice handles auth (IAM), routing, and observability at the application layer.
- Exposed to Kubernetes via the **Gateway API** (`GatewayClass: amazon-vpc-lattice`) — same object model as 6.2, different backing infra.
- Best for: large orgs with many accounts/VPCs needing service-to-service connectivity without networking-team tickets for every peering request.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: amazon-vpc-lattice
spec:
  controllerName: application-networking.k8s.aws/gateway-api-controller
```

---

## 7. Compute & Scaling

### 7.1 Fargate

| Trait | Detail |
|---|---|
| Unit of billing | Per pod (vCPU + memory requested, rounded to Fargate-supported sizes) |
| Node visibility | None — no EC2 instance to see/patch/SSH into |
| Networking | Each pod = its own micro-VM, still gets a real VPC IP via CNI |
| Not supported | DaemonSets, privileged pods, hostNetwork/hostPort, GPU workloads |
| Selection mechanism | **Fargate Profile** — matches namespace (+ optional label selectors) |

```bash
eksctl create fargateprofile \
  --cluster my-cluster \
  --name fp-default \
  --namespace serverless-ns \
  --labels env=prod
```

- Any pod created in a namespace covered by a Fargate profile gets scheduled onto Fargate automatically — no node affinity/toleration needed by default (a profile can also restrict by label).

### 7.2 EKS Node Groups

| Feature | Self-managed | Managed Node Group |
|---|---|---|
| EC2/ASG creation | You | AWS (via eksctl/console/Terraform) |
| Rolling upgrades | You script it | `eksctl upgrade nodegroup` / console handles cordon-drain-replace |
| AMI choice | Any AMI | AWS-optimized AMI (or Bottlerocket) by default, custom AMI supported via launch template |
| Spot support | Yes, manual config | Yes, native `--spot` flag |

```bash
# create a managed node group
eksctl create nodegroup \
  --cluster my-cluster \
  --name ng-general \
  --node-type m5.large \
  --nodes 3 --nodes-min 2 --nodes-max 6 \
  --managed

# create a spot-backed node group for cost savings
eksctl create nodegroup \
  --cluster my-cluster \
  --name ng-spot \
  --instance-types m5.large,m5a.large,m5n.large \
  --spot

# rolling upgrade nodes to latest AMI release for current K8s version
eksctl upgrade nodegroup --cluster my-cluster --name ng-general
```

### 7.3 Karpenter

- A **cluster autoscaler alternative** built by AWS, now CNCF-donated — provisions **right-sized EC2 instances directly** (no pre-defined ASG/Node Group needed).
- Watches for **unschedulable pods**, computes the cheapest/best-fit instance type on the fly, launches it, binds the pod, and **consolidates/terminates underutilized nodes** automatically.
- Faster than Cluster Autoscaler (skips ASG scaling delay) and more flexible (mixes instance types/sizes automatically instead of one fixed type per ASG).

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
      nodeClassRef:
        name: default
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
```

```bash
kubectl get nodepools
kubectl get nodeclaims          # Karpenter's version of "nodes it has provisioned"
kubectl logs -n karpenter deploy/karpenter
```

**Cluster Autoscaler vs Karpenter, memorize this:**

| | Cluster Autoscaler | Karpenter |
|---|---|---|
| Provisions via | Existing ASGs (Node Groups) | Directly, no ASG required |
| Instance flexibility | One type per ASG (or mixed-instance ASG config) | Fully dynamic best-fit per pod |
| Speed | Slower (ASG scaling lifecycle) | Faster (talks to EC2 fleet API directly) |
| Consolidation | Basic | Aggressive, continuous bin-packing |

### 7.4 Compute Demo (scenario walkthrough)

```bash
# scenario: deploy a workload that needs to trigger node scale-up
kubectl create deployment scale-demo --image=nginx --replicas=1
kubectl scale deployment scale-demo --replicas=50
kubectl get pods -o wide                     # watch some go Pending
kubectl get events --sort-by='.lastTimestamp' | grep -i "not trigger\|scale"

# Cluster Autoscaler path: watch ASG desired count climb
aws autoscaling describe-auto-scaling-groups \
  --query "AutoScalingGroups[?contains(AutoScalingGroupName,'ng-general')].DesiredCapacity"

# Karpenter path: watch a NodeClaim get created
kubectl get nodeclaims -w
```

---

## 8. Redundancy & Resiliency

### 8.1 Cluster Access

- **Endpoint access modes** — set at the cluster level, controls WHERE the API server can be reached from:

| Mode | Reachable from |
|---|---|
| Public only (default) | Anywhere on the internet |
| Public + Private | Internet AND inside the VPC (nodes always use private path automatically when both enabled) |
| Private only | Only inside the VPC (via VPN/Direct Connect/bastion/Cloud9-in-VPC) |

```bash
aws eks update-cluster-config \
  --name my-cluster \
  --resources-vpc-config endpointPublicAccess=false,endpointPrivateAccess=true
```

- **Public access CIDR allowlisting** — even with public access on, you can restrict WHICH IPs can reach it:
```bash
aws eks update-cluster-config \
  --name my-cluster \
  --resources-vpc-config publicAccessCidrs="203.0.113.0/24"
```

- **Multi-AZ resiliency** happens automatically for the control plane. For YOUR data plane, you must explicitly spread node groups across ≥2-3 AZs (subnet selection at node group creation) and use **PodDisruptionBudgets** + **topology spread constraints** so your own workloads survive an AZ outage.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web
```

### 8.2 IRSA (IAM Roles for Service Accounts)

The classic (still very common) way for a **pod** to get real AWS permissions, without hardcoding AWS keys.

**How it works end to end:**
1. Cluster has an **OIDC provider** registered in IAM (one-time setup per cluster).
2. You create an **IAM Role** with a **trust policy** that trusts that specific OIDC provider + a specific `namespace:serviceaccount`.
3. You create a Kubernetes `ServiceAccount` **annotated** with that role's ARN.
4. Pods using that ServiceAccount get env vars (`AWS_ROLE_ARN`, `AWS_WEB_IDENTITY_TOKEN_FILE`) injected automatically by a mutating webhook — the AWS SDK inside your app picks these up transparently and calls `AssumeRoleWithWebIdentity`.

```bash
# one-time: associate OIDC provider with the cluster
eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve

# create IAM role + K8s ServiceAccount + trust policy in one shot
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace prod \
  --name s3-reader-sa \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader-sa
  namespace: prod
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/s3-reader-role
---
apiVersion: v1
kind: Pod
metadata:
  name: s3-test
  namespace: prod
spec:
  serviceAccountName: s3-reader-sa
  containers:
    - name: aws-cli
      image: amazon/aws-cli
      command: ["sleep", "3600"]
```

```bash
kubectl exec -it s3-test -n prod -- aws s3 ls   # should work with no static keys anywhere
```

### 8.3 Pod Identity (the newer, simpler replacement for IRSA)

- Released 2023 — removes the OIDC-annotation dance. Uses an **EKS Pod Identity Agent** (a DaemonSet add-on) plus a simple **association** object.
- No per-pod ServiceAccount annotation needed, no per-cluster OIDC trust-policy templating — just a direct `role <-> namespace/serviceaccount` mapping via the API.

```bash
# install the Pod Identity Agent add-on (once per cluster)
aws eks create-addon --cluster-name my-cluster --addon-name eks-pod-identity-agent

# create the association (role <-> serviceaccount)
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace prod \
  --service-account s3-reader-sa \
  --role-arn arn:aws:iam::123456789012:role/s3-reader-role
```

**IRSA vs Pod Identity, memorize this:**

| | IRSA | Pod Identity |
|---|---|---|
| Setup | OIDC provider + per-role trust policy | One association API call |
| ServiceAccount changes | Needs `eks.amazonaws.com/role-arn` annotation | No annotation needed at all |
| Cross-account roles | Supported, more manual trust-policy work | Supported, simpler |
| Role reuse across clusters | Trust policy hardcodes OIDC per cluster (messy) | Association is per-cluster, cleaner to manage many clusters |
| Recommended for new clusters | Legacy/compatibility | **Yes, default choice now** |

---

## 9. Upgrades and Maintenance

### 9.1 EKS Monitoring

| Layer | Tool |
|---|---|
| Control plane logs | CloudWatch Logs (API server, audit, authenticator, scheduler, controller-manager — each toggled independently) |
| Node/pod metrics | CloudWatch Container Insights (via CloudWatch agent/Fluent Bit DaemonSet) |
| Prometheus-native | Amazon Managed Prometheus (AMP) + Amazon Managed Grafana (AMG) |
| Node-level OS metrics | CloudWatch agent DaemonSet |

```bash
# enable specific control plane log types
aws eks update-cluster-config \
  --name my-cluster \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

# tail control plane logs
aws logs tail /aws/eks/my-cluster/cluster --follow
```

### 9.2 Upgrade Cycles

- AWS supports a **rolling window of Kubernetes minor versions** (typically the last ~4 minor versions at any time, aligned with upstream K8s support + AWS's own **Standard Support / Extended Support** tiers).
- **Standard Support**: included in the base EKS price, for the newest supported versions.
- **Extended Support**: extra per-cluster-hour fee, lets you stay on an older minor version longer (up to ~26 months past standard end-of-life) instead of being forced to upgrade immediately.
- You control WHEN to upgrade (never fully automatic) — but once a version hits end-of-standard-support, AWS **auto-upgrades** clusters that haven't moved, so don't let it lapse silently.

### 9.3 EKS Upgrades (the actual mechanics — order matters, memorize this)

**Correct upgrade order, every time:**
1. **Back up / review** — check add-on compatibility matrix, deprecated API usage (`kubectl deprecations` / `pluto` / `kube-no-trouble`).
2. **Upgrade the control plane** — one minor version at a time (1.28 → 1.29, never skip versions).
3. **Upgrade EKS add-ons** (CNI, CoreDNS, kube-proxy, EBS/EFS CSI) to versions compatible with the new control plane.
4. **Upgrade node groups / Fargate** — new nodes come up on the new version; old nodes drained via rolling replacement.
5. **Verify** workloads, re-run smoke tests.

```bash
# step 2: upgrade control plane (one minor version only)
eksctl upgrade cluster --name my-cluster --version 1.29 --approve

# step 3: check + upgrade add-ons
aws eks describe-addon-versions --addon-name vpc-cni --kubernetes-version 1.29
eksctl update addon --cluster my-cluster --name vpc-cni --version latest

# step 4: upgrade managed node group (rolling: cordon -> drain -> replace)
eksctl upgrade nodegroup --cluster my-cluster --name ng-general

# step 4 (Fargate): no action needed, new pods automatically launch on new patch version
```

> **You cannot skip minor versions on the control plane.** Going from 1.27 to 1.29 requires two separate upgrade operations (1.27→1.28, then 1.28→1.29).

### 9.4 EKS Add-ons

- **EKS Add-ons** = AWS-managed lifecycle (install/upgrade/configure) for common cluster components, instead of you self-managing them via Helm/manifests.
- Core ones: `vpc-cni`, `coredns`, `kube-proxy`, `aws-ebs-csi-driver`, `aws-efs-csi-driver`, `eks-pod-identity-agent`, `adot` (OpenTelemetry).

| Conflict resolution mode | Behavior |
|---|---|
| `NONE` | Fail if the add-on already exists self-managed |
| `OVERWRITE` | Take over management, discard existing custom config |
| `PRESERVE` | Take over management, keep existing custom config values |

```bash
# list what's available and installed
aws eks describe-addon-versions --kubernetes-version 1.29 --query 'addons[].addonName'
aws eks list-addons --cluster-name my-cluster

# install one
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name aws-ebs-csi-driver \
  --resolve-conflicts PRESERVE

# check add-on health
aws eks describe-addon --cluster-name my-cluster --addon-name vpc-cni
```

---

## 10. Quick-Reference: Full Cluster Bootstrap (putting it all together)

```bash
# 1. create cluster with a managed node group, OIDC auto-associated
eksctl create cluster \
  --name my-cluster \
  --region us-east-1 \
  --version 1.29 \
  --nodegroup-name ng-general \
  --node-type m5.large \
  --nodes 3 --nodes-min 2 --nodes-max 6 \
  --managed \
  --with-oidc

# 2. point kubectl at it
aws eks update-kubeconfig --region us-east-1 --name my-cluster

# 3. install core add-ons
aws eks create-addon --cluster-name my-cluster --addon-name aws-ebs-csi-driver --resolve-conflicts PRESERVE
aws eks create-addon --cluster-name my-cluster --addon-name eks-pod-identity-agent

# 4. install the Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system --set clusterName=my-cluster

# 5. set up IAM for a workload (Pod Identity, the modern way)
aws eks create-pod-identity-association \
  --cluster-name my-cluster --namespace prod \
  --service-account app-sa --role-arn arn:aws:iam::123456789012:role/app-role

# 6. deploy something and expose it
kubectl create namespace prod
kubectl create deployment web --image=nginx -n prod
kubectl expose deployment web --port=80 --type=LoadBalancer -n prod
kubectl get svc web -n prod -w   # wait for EXTERNAL-IP (NLB DNS name)
```

---

## 11. Common Failure Scenarios (fast diagnosis table)

| Symptom | Likely cause | Check |
|---|---|---|
| Pod stuck `Pending`, no scheduling reason obvious | Node out of IPs (CNI limit) | `kubectl describe pod`, check `WARM_ENI_TARGET`/prefix delegation |
| Pod stuck `Pending`, cluster has 0 nodes | Autoscaler/Karpenter not triggered, or no matching NodePool/ASG | `kubectl get events`, `kubectl get nodeclaims` |
| `kubectl` gets "Unauthorized" | IAM identity not in aws-auth/Access Entries | `aws eks list-access-entries` |
| Pod can't reach AWS API (S3, DynamoDB) | Missing/misconfigured IRSA or Pod Identity | `kubectl describe sa`, check role trust policy |
| PVC stuck `Pending` | AZ mismatch (EBS is single-AZ) | Check `WaitForFirstConsumer`, check node/volume AZ |
| ALB/NLB never appears after creating Ingress/Service | LB Controller not installed, or missing subnet tags | `kubectl describe ingress`, check subnet tags `kubernetes.io/role/elb` |
| Upgrade fails or workloads break post-upgrade | Skipped a minor version, or deprecated API still in use | Run `pluto detect-helm`/`kubectl deprecations` before upgrading |
| Secrets visible in etcd snapshot | Envelope encryption never enabled | `eksctl utils enable-secrets-encryption` |

---

*Cross-reference: pair this with your `CKAD_Cheatsheet.md` and `CKAD_Command_Reference.md` for the pure-Kubernetes object-level details (Deployments, Probes, ConfigMaps, etc.) — this doc focuses on the AWS-specific layer wrapped around vanilla Kubernetes.*
