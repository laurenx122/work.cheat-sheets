# CKAD / Kubernetes Cheatsheet

Certified Kubernetes Application Developer (CKAD) 
A single reference tying together every topic in your course list: what each object does, how it connects to the others, what to look for in YAML, and the commands to inspect/troubleshoot it.

---

## 0. The Big Picture — How Everything Connects

```
Namespace (isolation boundary)
  └─ Deployment
       └─ ReplicaSet (auto-created, auto-managed)
            └─ Pod(s)
                 ├─ Container(s)      -> images, commands/args, env vars, resources
                 ├─ Volumes           -> ConfigMap / Secret / PVC / emptyDir mounted in
                 ├─ ServiceAccount    -> identity used to talk to the API server
                 ├─ SecurityContext   -> user/group/permissions the container runs as
                 └─ Probes            -> liveness / readiness / startup

  └─ Service                          -> stable IP/DNS, routes to Pods via label selector
       └─ Endpoints (auto-created)    -> the actual Pod IPs currently matching the selector

  └─ Ingress                          -> HTTP(S) routing INTO Services (host/path based)
  └─ NetworkPolicy                    -> firewall rules BETWEEN Pods

  └─ PersistentVolumeClaim -> PersistentVolume -> StorageClass (dynamic provisioning)

  └─ Job / CronJob                    -> run-to-completion / scheduled Pods (not "always on")
  └─ StatefulSet                      -> like a Deployment, but for Pods needing stable
                                          identity + storage (databases, etc.)
  └─ DaemonSet                        -> one Pod per node (agents, log collectors)

RBAC layer (who can do what):
  ServiceAccount / User -> RoleBinding -> Role (namespaced) or ClusterRoleBinding -> ClusterRole (cluster-wide)
```

**Mental model:** Deployment manages ReplicaSet manages Pods. Service just *watches* for Pods matching a label — it has no idea a Deployment even exists. That's why deleting a Deployment doesn't touch the Service, and why a Service can silently break (blank endpoints) if labels drift.

---

## 1. Core Workload Objects

| Object | Create (`kubectl apply`) | Update | Delete |
|---|---|---|---|
| **Pod** | Spins up one naked container instance | Not supported in-place — delete & recreate | `kubectl delete pod <name>` — gone, no replacement |
| **ReplicaSet** | Spins up the RS + target number of Pods | Partial — editing the template updates the RS definition, but *existing* Pods don't change until deleted | `kubectl delete rs <name>` — deletes RS and all its Pods |
| **Deployment** | Creates Deployment → ReplicaSet → Pods | Fully automated — change image/config, re-apply, K8s does a zero-downtime rolling update | `kubectl delete deployment <name>` — deletes Deployment, RS, and Pods |
| **Service** | Opens a stable virtual IP + DNS name, maps via selector to matching Pod labels | In-place — re-apply YAML or `kubectl edit svc` | `kubectl delete svc <name>` — removes routing only; Pods keep running |

**Golden rules:**
- Never manage naked Pods in production — always wrap in a Deployment.
- Never edit a standalone ReplicaSet to roll out a new app version — use a Deployment so rolling updates are automatic.
- Services are independent of Deployments — deleting a Deployment kills your app; the Service just sits there with empty endpoints until matching Pods exist again.

### YAML skeleton — where to look

```yaml
apiVersion: apps/v1        # Pod uses "v1", RS/Deploy use "apps/v1"
kind: Deployment
metadata:
  name: myapp
  labels: {app: myapp}     # labels ON the Deployment object itself (rarely matched against)
spec:
  replicas: 3
  selector:
    matchLabels: {app: myapp}     # <-- MUST match template.metadata.labels exactly
  template:                       # <-- this is the "Pod template" — a Pod spec nested inside
    metadata:
      labels: {app: myapp}        # <-- these are the labels Pods actually get
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
```

⚠️ **Most common exam bug:** `spec.selector.matchLabels` not matching `spec.template.metadata.labels`. Deployment/RS refuses to create or acts weird if these mismatch.

### Key commands

```bash
kubectl run mypod --image=nginx --dry-run=client -o yaml > pod.yaml     # generate Pod YAML fast
kubectl create deployment myapp --image=nginx --replicas=3 --dry-run=client -o yaml
kubectl scale deployment myapp --replicas=5
kubectl set image deployment/myapp myapp=nginx:1.25
kubectl rollout status deployment/myapp
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp                 # rollback to previous
kubectl rollout undo deployment/myapp --to-revision=2
```

---

## 2. Namespaces

Logical isolation within one cluster — most objects are namespaced, some (Nodes, PV, ClusterRole, StorageClass) are **not**.

```bash
kubectl create namespace dev
kubectl get pods -n dev
kubectl get pods --all-namespaces          # or -A
kubectl config set-context --current --namespace=dev   # switch default ns
```

In YAML: `metadata.namespace: dev`. If omitted, `default` is used.

---

## 3. Configuration

### 3.1 Images, Commands & Arguments

| Docker | Kubernetes YAML field | Purpose |
|---|---|---|
| `ENTRYPOINT` | `command:` | The executable that runs |
| `CMD` | `args:` | Default arguments (overridable) |

```yaml
containers:
- name: myapp
  image: myapp:1.0
  command: ["sleep"]     # overrides ENTRYPOINT
  args: ["3600"]         # overrides CMD
```
`kubectl exec <pod> -- <cmd>` runs a one-off command; doesn't change the container's own command.

### 3.2 Environment Variables

```yaml
env:
- name: APP_COLOR
  value: "blue"
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef: {name: db-secret, key: password}
- name: LOG_LEVEL
  valueFrom:
    configMapKeyRef: {name: app-config, key: log_level}
```

### 3.3 ConfigMaps — non-sensitive config

```bash
kubectl create configmap app-config --from-literal=LOG_LEVEL=debug
kubectl create configmap app-config --from-file=app.properties
kubectl get configmap app-config -o yaml
```
Consume as env vars (above), `envFrom`, or mounted as a **volume** (each key becomes a file).

### 3.4 Secrets — sensitive config (base64-encoded, NOT encrypted by default)

```bash
kubectl create secret generic db-secret --from-literal=password=mypass
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d   # decode to check
```
Encrypt at rest by configuring an `EncryptionConfiguration` on the API server (`--encryption-provider-config`).

### 3.5 Security Contexts — who the container runs as

```yaml
spec:
  securityContext:            # Pod-level — applies to all containers
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: myapp
    securityContext:          # Container-level — overrides Pod-level for this container
      runAsUser: 2000
      capabilities:
        add: ["NET_ADMIN"]
```
Pod-level and container-level can both exist — container-level wins for that container.

### 3.6 Resource Requirements

```yaml
resources:
  requests: {cpu: "250m", memory: "64Mi"}   # guaranteed / used for scheduling
  limits:   {cpu: "500m", memory: "128Mi"}  # hard ceiling
```
- Exceed **memory limit** → container OOMKilled.
- Exceed **CPU limit** → throttled, not killed.
- No `requests` set → scheduler treats it as "needs nothing," can overcommit a node.

**Inspect:** `kubectl top pod`, `kubectl describe pod <name>` (look for `Reason: OOMKilled` in Last State).

### 3.7 Service Accounts — identity for talking to the API

```bash
kubectl create serviceaccount myapp-sa
kubectl get sa
```
```yaml
spec:
  serviceAccountName: myapp-sa   # Pod field — default is "default" SA if omitted
```
Mounted automatically at `/var/run/secrets/kubernetes.io/serviceaccount/` inside the Pod (token, ca.crt, namespace).

### 3.8 Taints & Tolerations — repel Pods from Nodes (unless they tolerate it)

**Taint effects** (`spec.effect` on the taint / toleration):

| Effect | Behavior |
|---|---|
| `NoSchedule` | New Pods without a matching toleration won't be scheduled here; existing Pods are unaffected |
| `PreferNoSchedule` | Soft version — scheduler tries to avoid it, but will place Pods here if there's no other option |
| `NoExecute` | New Pods without a toleration won't schedule, **and existing Pods already running here get evicted** |

```bash
kubectl taint nodes node1 key=value:NoSchedule
```
```yaml
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"       # see effects table above
```
Toleration doesn't *force* placement onto that node — it just permits it. For that, combine with node affinity.

### 3.9 Node Selectors & Node Affinity — attract Pods to Nodes

```yaml
spec:
  nodeSelector:                     # simple, exact match only
    disktype: ssd
```
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:   # hard rule
        nodeSelectorTerms:
        - matchExpressions:
          - {key: disktype, operator: In, values: ["ssd"]}
      # OR preferredDuringSchedulingIgnoredDuringExecution: (soft rule, best-effort)
```

**Taints/Tolerations vs Node Affinity:** taints *repel* everything except tolerating Pods (node's perspective); affinity *attracts* Pods to specific nodes (Pod's perspective). Use both together to fully dedicate a node to specific workloads.

---

## 4. Multi-Container Pods

Containers in the same Pod share network namespace (localhost) and can share volumes.

| Pattern | What it does |
|---|---|
| **Sidecar** | Helper running alongside main container the whole time (e.g., log shipper) |
| **Ambassador** | Proxies network connections for the main container |
| **Adapter** | Standardizes/transforms the main container's output |
| **Init Container** | Runs **before** app containers, must complete successfully first; used for setup/wait-for-dependency |

```yaml
spec:
  initContainers:
  - name: init-wait-db
    image: busybox
    command: ['sh', '-c', 'until nslookup db; do sleep 2; done']
  containers:
  - name: myapp
    image: myapp:1.0
```
Multiple init containers run **sequentially**; regular containers run in parallel.

**Inspect:** `kubectl logs <pod> -c <container-name>` (container name required when there's more than one), `kubectl logs <pod> -c <initcontainer-name>`.

---

## 5. Observability

### 5.1 Probes

| Probe | Question it answers | Failure result |
|---|---|---|
| **Liveness** | "Is this container still alive/healthy?" | Container is **restarted** |
| **Readiness** | "Is this container ready to receive traffic?" | Pod is pulled **out of Service endpoints** (not restarted) |
| **Startup** | "Has this slow-starting app finished booting?" | Blocks liveness/readiness checks until it passes |

```yaml
containers:
- name: myapp
  livenessProbe:
    httpGet: {path: /healthz, port: 8080}
    initialDelaySeconds: 15
    periodSeconds: 10
  readinessProbe:
    exec:
      command: ["cat", "/tmp/ready"]
    periodSeconds: 5
  # other probe types: tcpSocket: {port: 3306}
```

**This is the #1 SRE-relevant topic.** A missing/misconfigured readiness probe = traffic sent to a Pod that isn't actually ready = errors. A missing liveness probe = a hung container never gets restarted.

### 5.2 Logging

```bash
kubectl logs <pod>
kubectl logs <pod> -c <container>       # multi-container pod
kubectl logs <pod> --previous           # logs from before last crash/restart
kubectl logs -f <pod>                   # follow/stream
```

### 5.3 Monitoring

Kubernetes has no built-in full monitoring stack — commonly Metrics Server (for `kubectl top`) + Prometheus/Grafana.

```bash
kubectl top node
kubectl top pod
```

---

## 6. Pod Design

### 6.1 Labels, Selectors, Annotations

- **Labels** — key/value pairs used for *identification and grouping* (`app: frontend`, `env: prod`). Selectors match on these.
- **Annotations** — key/value metadata *not* used for selection (build info, changelog notes, tool config).

```bash
kubectl get pods --show-labels
kubectl get pods -l app=frontend,env=prod          # AND condition
kubectl label pod mypod tier=backend
kubectl annotate pod mypod description="test pod"
```

### 6.2 Rolling Updates & Rollbacks

Default `Deployment` strategy is `RollingUpdate` — replaces Pods gradually with zero downtime.

```yaml
spec:
  strategy:
    type: RollingUpdate     # or Recreate (kills all old, then creates all new — downtime)
    rollingUpdate:
      maxUnavailable: 1     # how many old Pods can be down at once
      maxSurge: 1           # how many extra new Pods can exist during rollout
```

```bash
kubectl rollout status deployment/myapp
kubectl rollout history deployment/myapp --revision=2
kubectl rollout undo deployment/myapp
```

### 6.3 Deployment Strategies (application-level, not native K8s objects)

| Strategy | How | Native K8s feature? |
|---|---|---|
| **Rolling Update** | Gradual replace | Yes — built into Deployment |
| **Blue-Green** | Two full environments; switch a Service's selector from old to new all at once | No — you build it manually via two Deployments + a Service selector swap |
| **Canary** | Small % of traffic to new version alongside old, before full rollout | No — build it manually: two Deployments sharing one Service's label selector, differing replica counts |

### 6.4 Jobs & CronJobs — run-to-completion, not "always on"

```yaml
apiVersion: batch/v1
kind: Job
metadata: {name: myjob}
spec:
  completions: 3        # run the Pod 3 times successfully
  parallelism: 2         # 2 at once
  backoffLimit: 4         # retries before marking Job failed
  template:
    spec:
      containers:
      - name: job
        image: busybox
        command: ["echo", "done"]
      restartPolicy: Never    # Job Pods must be Never or OnFailure (not Always)
```

```yaml
apiVersion: batch/v1
kind: CronJob
metadata: {name: mycron}
spec:
  schedule: "*/5 * * * *"     # standard cron syntax
  jobTemplate:
    spec: {template: {spec: {containers: [...], restartPolicy: Never}}}
```

```bash
kubectl get jobs
kubectl get cronjobs
kubectl create job manual-run --from=cronjob/mycron   # trigger a CronJob manually
```

---

## 7. State Persistence

### 7.1 Volumes (Pod-level, tied to Pod lifetime)

```yaml
spec:
  volumes:
  - name: data
    emptyDir: {}              # dies with the Pod — scratch space only
  containers:
  - name: myapp
    volumeMounts:
    - name: data
      mountPath: /var/www/html      # <-- this is what you were debugging (nginx/php-fpm mismatch)
```
⚠️ Your nginx/php-fpm bug: both containers must mount the **same volume name** at the path each expects — mismatched `mountPath` between containers = broken shared files even though the volume itself is fine.

### 7.2 PersistentVolume (PV) & PersistentVolumeClaim (PVC) — cluster-level storage, outlives Pods

**Why this exists:** configuring storage directly inside every Pod definition doesn't scale — updating storage means editing every single Pod YAML. PVs centralize it: an admin provisions a pool of cluster-wide storage once, and developers request slices of it via claims without needing to know the underlying infrastructure.

```
Admin/StorageClass provisions -> PersistentVolume (actual storage, cluster-scoped, NOT namespaced)
                                        ^
Developer requests storage    -> PersistentVolumeClaim  (namespace-scoped, binds 1:1 to a PV that satisfies it)
                                        ^
Pod uses the claim             -> volumes: [{persistentVolumeClaim: {claimName: mypvc}}]
```

📌 **Lifecycle separation:** PVs are cluster-level (not tied to any namespace); PVCs are namespace-scoped and live inside whatever namespace your app Pods run in.

**PV access modes** (set in `spec.accessModes` on both the PV and the PVC — they must be compatible for binding):

| Mode | Short | Meaning |
|---|---|---|
| `ReadWriteOnce` | **RWO** | Mounted read-write by a single node at a time |
| `ReadOnlyMany` | **ROX** | Mounted read-only by many nodes simultaneously |
| `ReadWriteMany` | **RWX** | Mounted read-write by many nodes simultaneously |

**PV reclaim policies** (`spec.persistentVolumeReclaimPolicy` — decides what happens to the PV once its bound PVC is deleted):

| Policy | Behavior |
|---|---|
| **Retain** (default) | PV becomes `Released`; data and the volume stay intact on the backend. An admin must manually clean it up before it can be reused. |
| **Delete** | Automatically deletes both the PV object *and* the underlying storage asset (e.g. the AWS EBS volume itself) |
| **Recycle** *(deprecated on most modern drivers)* | Wipes the data (`rm -rf /thevolume/*`) and makes the PV `Available` again |

**PV status values** (`kubectl get pv`):

| Status | Meaning |
|---|---|
| `Available` | Free, not yet claimed |
| `Bound` | Linked to a PVC |
| `Released` | Claim was deleted, but data hasn't been reclaimed yet (with `Retain` policy) |
| `Failed` | Automatic reclamation failed |

**Example — PersistentVolume (local testing, `hostPath`):**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata: {name: pv-log}
spec:
  capacity: {storage: 100Mi}
  accessModes:
  - ReadWriteMany                       # see access-mode table above
  persistentVolumeReclaimPolicy: Retain # see reclaim-policy table above
  hostPath:
    path: /pv/log
```
⚠️ **`hostPath` warning:** fine for quick single-node testing, but breaks in multi-node clusters since nodes don't share local filesystems. For real environments, use a cloud backend instead:
```yaml
  awsElasticBlockStore:
    volumeID: <volume-id>
    fsType: ext4
```

**Example — PersistentVolumeClaim requesting a slice of that pool:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: claim-log-1}
spec:
  accessModes:
  - ReadWriteMany      # must match/be compatible with the PV's access mode, or it won't bind
  resources:
    requests:
      storage: 50Mi     # PVC's requested size can be SMALLER than the PV — it'll still bind
  storageClassName: standard
```

**Example — Pod consuming the PVC (not the raw volume directly):**
```yaml
apiVersion: v1
kind: Pod
metadata: {name: webapp}
spec:
  containers:
  - name: event-simulator
    image: kodekloud/event-simulator
    volumeMounts:
    - name: log-volume
      mountPath: /log
  volumes:
  - name: log-volume
    persistentVolumeClaim:
      claimName: claim-log-1     # <-- Pod references the PVC by name, never the PV directly
```

**Binding mechanics:**
- Kubernetes matches a PVC to a PV based on **access modes**, **capacity**, and **storage class** — all three must be compatible.
- A PVC requesting *less* storage (e.g. 500Mi) can bind to a *larger* PV (e.g. 1Gi) if no exact match exists and everything else lines up.
- No matching PV available → PVC sits in `Pending` until one shows up (or gets created).
- To force a PVC onto one *specific* PV, use `spec.selector.matchLabels` on the PVC matched against labels on the PV.

```bash
kubectl get pv                              # global pool: capacity, access modes, status, bound claim
kubectl get pvc                             # STATUS: Bound / Pending, bound volume, allocated size
kubectl describe pvc mypvc                  # check Events for exactly why it's not binding
kubectl apply -f pvc.yaml                   # or: kubectl create -f pvc.yaml
kubectl delete pvc claim-log-1              # triggers the PV's reclaim policy
kubectl exec webapp -- cat /log/app.log     # verify data is actually landing on the mounted volume
kubectl replace --force -f pod.yaml         # force-recreate a Pod when an in-place edit is rejected
```

**Troubleshooting:**
| Symptom | Cause |
|---|---|
| PVC stuck `Pending` | Access-mode mismatch (e.g. PVC requests RWX but the only PV offers RWO), or requested capacity exceeds every available PV |
| PVC stuck `Terminating` | An active Pod is still mounting it — delete the Pod first, then the PVC finishes deleting |
| Data survives PVC deletion | Expected with `Retain` policy — PV goes to `Released`, needs manual admin cleanup before reuse |

### 7.3 StorageClass — dynamic provisioning (no manual PV creation needed)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: {name: standard}
provisioner: kubernetes.io/aws-ebs    # cloud-specific provisioner
```
When a PVC references a StorageClass, the PV is created automatically on demand.

### 7.4 StatefulSets — for apps needing stable identity + storage (databases, Kafka, etc.)

| Deployment | StatefulSet |
|---|---|
| Pods are interchangeable, random names (`myapp-7d8f9-x2kd9`) | Pods get **stable, ordered names** (`mysql-0`, `mysql-1`, `mysql-2`) |
| Shared/no persistent identity | Each Pod keeps **its own PVC** across restarts |
| Created/deleted in any order | Created/deleted/scaled **in order**, one at a time |

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: {name: mysql}
spec:
  serviceName: mysql-headless     # <-- must reference a Headless Service
  replicas: 3
  selector: {matchLabels: {app: mysql}}
  template: {metadata: {labels: {app: mysql}}, spec: {containers: [...]}}
  volumeClaimTemplates:           # <-- each Pod gets its OWN PVC generated from this template
  - metadata: {name: data}
    spec: {accessModes: ["ReadWriteOnce"], resources: {requests: {storage: 1Gi}}}
```

### 7.5 Headless Services — needed for StatefulSet Pod-to-Pod discovery

```yaml
apiVersion: v1
kind: Service
metadata: {name: mysql-headless}
spec:
  clusterIP: None          # <-- this makes it "headless"
  selector: {app: mysql}
```
Instead of one virtual IP, DNS returns each individual Pod IP: `mysql-0.mysql-headless`, `mysql-1.mysql-headless`, etc.

---

## 8. Services & Networking

### 8.1 Service Types

| Type | Reachable from | Use case |
|---|---|---|
| **ClusterIP** (default) | Inside cluster only | Internal service-to-service traffic |
| **NodePort** | `<NodeIP>:<30000-32767>` | Quick external access, dev/testing |
| **LoadBalancer** | External cloud LB, public IP | Production external access on a cloud provider |
| **ExternalName** | Maps to a DNS CNAME | Point to something outside the cluster |

```yaml
apiVersion: v1
kind: Service
metadata: {name: myapp-svc}
spec:
  type: ClusterIP
  selector:
    app: myapp             # <-- MUST match the target Pods' labels exactly
  ports:
  - port: 80                # port the Service listens on
    targetPort: 8080         # port the container actually listens on
    # nodePort: 30080        # only used/valid if type: NodePort
```

### 8.2 🔍 Inspecting & Troubleshooting a Service (blank endpoints = broken)

This is the core debugging flow — **do it in this order:**

```bash
# 1. Check the Service's selector
kubectl get svc myapp-svc -o yaml | grep -A3 selector
# or:
kubectl describe svc myapp-svc      # shows Selector: and Endpoints: right in the output

# 2. Check the Pod's actual labels
kubectl get pods --show-labels
kubectl get pods -l app=myapp       # does this selector actually return the pods you expect?

# 3. Check the Endpoints object (this is the ground truth of "is routing actually working")
kubectl get endpoints myapp-svc
kubectl describe endpoints myapp-svc
```

| Symptom | Meaning | Likely cause |
|---|---|---|
| `kubectl get endpoints myapp-svc` shows **`<none>`** or empty | Service found **zero** matching Pods | Selector typo, label typo, or Pods not `Ready` (failing readiness probe) |
| Endpoints has entries, but app still unreachable | Routing is fine | Wrong `targetPort`, app not listening on that port inside the container, firewall/NetworkPolicy blocking it |
| Endpoints has *fewer* IPs than expected replicas | Some Pods not Ready | Check `kubectl get pods` — CrashLoopBackOff, failing readiness probe, etc. |

**Golden rule:** `Service.spec.selector` must exactly match `Pod.metadata.labels` (not the Deployment's own labels, not the Pod template's *other* fields — specifically the Pods' actual labels). A Service never looks at the Deployment at all.

```bash
# Quick end-to-end connectivity test from inside the cluster
kubectl run tmp-shell --rm -it --image=busybox -- sh
# then inside: wget -O- myapp-svc:80   or   nslookup myapp-svc
```

### 8.3 Network Policies — firewall rules between Pods

**Core concept:** By default, Kubernetes allows **all** traffic between all Pods — completely open. NetworkPolicy locks this down using labels, namespaces, and IP blocks. The moment you attach a `policyTypes: [Ingress]` (or `Egress`) to a Pod's selector, that direction of traffic becomes **deny-by-default** — only what you explicitly allow gets through. Everything else silently drops.

Typical flow this protects: `User -> Web -> App (APIs) -> Database`. You'd lock the Database down so only the App tier can reach it, not the whole cluster.

**Default Deny (block all incoming traffic to a Pod):**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: db-policy}
spec:
  podSelector:
    matchLabels: {role: db}
  policyTypes:
  - Ingress
  # leaving 'ingress:' out entirely = allow nothing in
```

**Restrict by Pod + Namespace together** (allow only `api-pod` Pods that live inside the `prod` namespace):
```yaml
spec:
  podSelector:
    matchLabels: {role: db}
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - podSelector: {matchLabels: {name: api-pod}}
      namespaceSelector: {matchLabels: {name: prod}}   # combined = AND (both must match)
    ports:
    - protocol: TCP
      port: 3306
```
⚠️ **Danger:** `namespaceSelector` **without** a companion `podSelector` allows **every Pod** in that namespace in — not just the one you meant.

**Allow an external IP (`ipBlock`)** — e.g. an outside backup server:
```yaml
spec:
  podSelector:
    matchLabels: {role: db}
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - podSelector: {matchLabels: {name: api-pod}}
      namespaceSelector: {matchLabels: {name: prod}}
    - ipBlock:
        cidr: 192.168.5.10/32
    ports:
    - protocol: TCP
      port: 3306
```
📌 Separate items **under `from:`** = **OR** logic (prod api-pods OR that external IP). Fields combined **inside the same `from:` item** = **AND** logic (as in the podSelector+namespaceSelector example above). This is the #1 place people get tripped up — indentation determines AND vs OR.

**Egress (outbound) rules** — must be declared explicitly, and needs its own `policyTypes` entry:
```yaml
spec:
  podSelector:
    matchLabels: {name: internal}
  policyTypes:
  - Egress
  egress:
  - to:
      podSelector: {matchLabels: {name: payroll}}
    ports:
    - protocol: TCP
      port: 8080
  - to:
      podSelector: {matchLabels: {name: mysql}}
    ports:
    - protocol: TCP
      port: 3306
```

**Ingress vs Egress — which side each protects:**
| Rule type | Protects | Controls |
|---|---|---|
| **Ingress** | The target Pod itself | Who is allowed to *call in* to it (e.g. lock down a database or payroll port) |
| **Egress** | Other things the Pod might reach | What the Pod itself is allowed to *call out* to (e.g. an internal app pushing data externally) |

📌 **Automatic response traffic:** once you allow Ingress traffic in, replies to that traffic flow back out automatically — you do **not** need a matching Egress rule just to let responses return.

⚠️ **CNI dependency:** NetworkPolicy requires a CNI plugin that actually enforces it — Calico, Weave Net, Cilium, KubeRouter. **Flannel (a common default) silently accepts the YAML but enforces nothing** — your "policy" does nothing and you won't get an error telling you that.

```bash
kubectl get pods
kubectl get service
kubectl get netpol                        # short alias — safer than "networkpolicies",
                                            # which can throw a server error on some clusters
kubectl describe netpol <policy_name>      # shows resolved pod selectors + allowed ports/sources
kubectl create -f internal-policy.yaml
```

### 8.4 Ingress — HTTP(S) routing into Services

**The problem it solves:** exposing every app via its own NodePort or LoadBalancer means managing a pile of external ports, firewall rules, and cloud load balancers — one per service. Expensive and messy.

**The solution:** Ingress is a Layer 7 (HTTP/HTTPS-aware) reverse proxy living *inside* the cluster. One entry point (port 80/443) handles SSL termination, host/path-based routing, and auth — instead of N separate external endpoints.

**Two separate pieces — both required:**
| Piece | What it is |
|---|---|
| **Ingress Controller** | The actual running software (NGINX, Traefik, HAProxy) that watches Ingress resources and does the routing. **Kubernetes ships with none of these by default — you must deploy one yourself.** |
| **Ingress Resource** (`kind: Ingress`) | Just the YAML *rules* — paths/hosts → which Service to send traffic to. Does nothing without a controller watching it. |

**Pattern 1 — Default backend** (everything goes to one service):
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: {name: ingress-wear}
spec:
  defaultBackend:
    service: {name: wear-service, port: {number: 80}}
```

**Pattern 2 — Path-based routing** (`/wear` vs `/watch` on the same host):
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: {name: ingress-wear-watch}
spec:
  rules:
  - http:
      paths:
      - path: /wear
        pathType: Prefix        # required in networking.k8s.io/v1 — YAML fails without it
        backend: {service: {name: wear-service, port: {number: 80}}}
      - path: /watch
        pathType: Prefix
        backend: {service: {name: watch-service, port: {number: 80}}}
```

**Pattern 3 — Host-based routing** (different domains → different services):
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: {name: ingress-host-based}
spec:
  rules:
  - host: wear.my-online-store.com
    http:
      paths: [{path: /, pathType: Prefix, backend: {service: {name: wear-service, port: {number: 80}}}}]
  - host: watch.my-online-store.com
    http:
      paths: [{path: /, pathType: Prefix, backend: {service: {name: watch-service, port: {number: 80}}}}]
```
If `host:` is omitted from a rule, the controller matches traffic from **any** hostname (wildcard).

**Production-style example — multiple paths + NGINX annotations:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear-watch
  namespace: app-space               # Ingress is namespace-scoped — must live in the
                                       # same namespace as the Services/apps it targets
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - http:
      paths:
      - {path: /wear, pathType: Prefix, backend: {service: {name: wear-service, port: {number: 8080}}}}
      - {path: /stream, pathType: Prefix, backend: {service: {name: video-service, port: {number: 8080}}}}
      - {path: /eat, pathType: Prefix, backend: {service: {name: food-service, port: {number: 8080}}}}
```

**Key annotations & why they matter:**
| Annotation | Fixes |
|---|---|
| `nginx.ingress.kubernetes.io/rewrite-target: /` | Your backend app expects requests at `/`, but the incoming path is `/pay` or `/wear` — this strips/translates the path before forwarding, so the app doesn't 404 on the prefix |
| `nginx.ingress.kubernetes.io/ssl-redirect: "false"` | Fixes the classic **HTTP 308 Permanent Redirect** bug — NGINX forces HTTPS redirection by default; disable it for plain HTTP testing |

**Deploying a controller yourself (rough shape):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: ingress-controller, namespace: ingress-space}
spec:
  replicas: 1
  selector: {matchLabels: {name: nginx-ingress}}
  template:
    metadata: {labels: {name: nginx-ingress}}
    spec:
      serviceAccountName: ingress-serviceaccount   # controller needs RBAC to watch Ingress objects via the API
      containers:
      - name: nginx-ingress-controller
        image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
        args:
        - /nginx-ingress-controller
        - --configmap=$(POD_NAMESPACE)/nginx-configuration
        - --default-backend-service=app-space/default-http-backend   # unhandled traffic -> clean 404, not a crash
        env:
        - {name: POD_NAMESPACE, valueFrom: {fieldRef: {fieldPath: metadata.namespace}}}
```
Then expose the controller itself so the outside world can actually reach it:
```bash
kubectl expose deploy ingress-controller -n ingress-space --name ingress \
  --port=80 --target-port=80 --type NodePort
```

```bash
kubectl get ingress -A
kubectl describe ingress <name> -n <namespace>     # verify paths + backends resolve, catch silent 404s
kubectl apply -f ingress.yaml

# Fast imperative creation, no YAML needed:
kubectl create ingress ingress-pay -n critical-space --rule="/pay=pay-service:8282"
kubectl create ingress ingress-wear-watch -n app-space \
  --rule="/wear=wear-service:8080" --rule="/watch=video-service:8080"
```

📌 **External exposure still required:** the Ingress controller itself needs a NodePort/LoadBalancer Service so traffic can reach it from outside the cluster in the first place — Ingress routes traffic *within* the cluster once it arrives, it doesn't get it there.

---

## 9. Security

### 9.1 Authentication vs Authorization

- **Authentication (AuthN)** — "who are you?" Users (via certs/tokens, managed outside K8s) or ServiceAccounts (managed inside K8s, have a token).
- **Authorization (AuthZ)** — "what are you allowed to do?" Handled by **RBAC** (most common), ABAC, or Webhook.

### 9.2 KubeConfig — client-side config for authenticating to a cluster

Lives at `$HOME/.kube/config` by default. Three sections, each a *list*, plus a **context** that ties one entry from each together:

```
Clusters                Contexts                     Users
(where to connect)      (which cluster + which        (who you are)
                         user, bundled as one name)
─────────────            ─────────────                ─────────────
development    <---\     admin@production   ----\     admin
production      <---+--- dev@google          ----+--- dev-user
google           <--/    mykubeadmin@         \---     prod-user
mykubeplayground  <------  mykubeplayground  --------  mykubeadmin
```
A **context** is just a shortcut name for a *(cluster, user, namespace)* triple — instead of specifying `--cluster` and `--user` on every command, you switch context once and `kubectl` uses that combo for everything after.

**Full example (matches the diagram — clusters/contexts/users as parallel lists):**
```yaml
apiVersion: v1
kind: Config

clusters:                                    # WHERE to connect — API server address + CA cert
- name: my-kube-playground                   # (values hidden — normally has server: + certificate-authority-data:)
- name: development
- name: production
- name: google

contexts:                                    # WHICH cluster + WHICH user, bundled under one name
- name: my-kube-admin@my-kube-playground     # convention: <user>@<cluster>
- name: dev-user@google
- name: prod-user@production

users:                                       # WHO you are — credentials (cert, token, etc.)
- name: my-kube-admin
- name: admin
- name: dev-user
- name: prod-user

current-context: my-kube-admin@my-kube-playground   # <-- the context kubectl uses right now
```
📌 Each `clusters[]` / `users[]` entry above is a summary name only — in a real file each also carries the actual connection details, e.g.:
```yaml
clusters:
- name: production
  cluster:
    server: https://prod.example.com:6443
    certificate-authority-data: <base64 CA cert>
users:
- name: prod-user
  user:
    client-certificate-data: <base64 cert>
    client-key-data: <base64 key>
contexts:
- name: prod-user@production
  context:
    cluster: production        # <-- references clusters[].name
    user: prod-user             # <-- references users[].name
    namespace: default          # optional — defaults the namespace for this context
```

```bash
kubectl config view                              # see the full merged config (secrets redacted)
kubectl config get-contexts                       # list all contexts, * marks the current one
kubectl config current-context
kubectl config use-context prod-user@production    # switch which cluster+user kubectl talks to
kubectl config set-context --current --namespace=dev   # change just the namespace of current context
kubectl --kubeconfig=/path/to/other/config get pods    # use a config file that isn't the default
```

### 9.3 API Groups

```
/api/v1                    -> "core" group: pods, services, configmaps, secrets, namespaces
/apis/apps/v1               -> deployments, replicasets, statefulsets, daemonsets
/apis/batch/v1               -> jobs, cronjobs
/apis/networking.k8s.io/v1    -> ingress, networkpolicy
/apis/rbac.authorization.k8s.io/v1 -> roles, rolebindings, clusterroles
```
```bash
kubectl api-resources                 # lists every resource + its API group + namespaced or not
kubectl explain deployment.spec       # docs for any field, straight from the API
```

### 9.4 RBAC — Role / RoleBinding (namespaced) vs ClusterRole / ClusterRoleBinding (cluster-wide)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role                          # ClusterRole = same shape, no namespace, can also cover cluster-scoped resources
metadata: {name: pod-reader, namespace: dev}
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding                   # ClusterRoleBinding = same shape, cluster-wide
metadata: {name: read-pods, namespace: dev}
subjects:
- kind: ServiceAccount
  name: myapp-sa
  namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

| Combo | Scope |
|---|---|
| Role + RoleBinding | One namespace only |
| ClusterRole + ClusterRoleBinding | Whole cluster |
| ClusterRole + RoleBinding | Cluster-wide permission set, but granted only within one namespace (common pattern for reusing a ClusterRole) |

```bash
kubectl auth can-i create pods --as=system:serviceaccount:dev:myapp-sa
kubectl auth can-i create pods --as=system:serviceaccount:dev:myapp-sa -n dev
kubectl get rolebindings,clusterrolebindings -A
kubectl describe role pod-reader -n dev
```

### 9.5 Admission Controllers

Plugins that intercept requests to the API server **after** AuthN/AuthZ, **before** persisting to etcd.
- **Validating** — accept or reject (e.g., `NamespaceExists`)
- **Mutating** — can modify the request (e.g., inject a default value, sidecar)

Configured on the API server itself (`--enable-admission-plugins`) — not something you typically apply as YAML like other objects, except for custom **ValidatingWebhookConfiguration** / **MutatingWebhookConfiguration**.

### 9.6 API Versions & Deprecation

```
alpha  (v1alpha1) -> may have bugs, disabled by default, can change/vanish anytime
beta   (v1beta1)  -> well-tested, enabled by default, schema may still change
stable (v1)        -> production-ready, won't be dropped without a long deprecation cycle
```
```bash
kubectl api-versions                  # see what's actually available on this cluster
kubectl convert -f old.yaml --output-version apps/v1   # (requires kubectl-convert plugin)
```

### 9.7 CRDs, Custom Controllers & Operators (extending Kubernetes itself)

```
CustomResourceDefinition (CRD)  -> teaches the API server a NEW kind (e.g., "kind: MySQLCluster")
Custom Resource (CR)             -> an actual instance of that new kind
Custom Controller                -> watches custom resources, reconciles real state to match desired state
Operator                          -> a custom controller + domain-specific operational knowledge
                                     (e.g., "how to safely upgrade a MySQL cluster")
```

**The ordering rule that trips people up:** the CRD must exist *before* you can create any Custom Resource of that kind — the API server has no idea what `kind: FlightTicket` even means until the CRD teaches it.

```bash
$ kubectl create -f flightticket.yml           # tried to create the CR FIRST
no matches for kind "FlightTicket" in version "flights.com/v1"    # <-- fails, kind doesn't exist yet
```

**Step 1 — define the CRD** (`flightticket-custom-definition.yml`):
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: flighttickets.flights.com     # convention: <plural>.<group>
spec:
  scope: Namespaced                    # Namespaced | Cluster
  group: flights.com                   # the API group this new kind lives under
  names:
    kind: FlightTicket                 # the actual `kind:` you'll write in CR YAML
    singular: flightticket
    plural: flighttickets              # used in kubectl get flighttickets
    shortNames:
    - ft                                # lets you run: kubectl get ft
  versions:
  - name: v1
    served: true                        # is this version currently reachable via the API?
    storage: true                       # is this the version actually persisted in etcd? (exactly one version must be true)
    schema:
      openAPIV3Schema:                  # validates every CR against this shape — reject bad ones
        type: object
        properties:
          spec:
            type: object
            properties:
              from:
                type: string
              to:
                type: string
              number:
                type: integer
                minimum: 1
                maximum: 10             # e.g. reject a booking request for 11+ seats
```

| Field | What it controls |
|---|---|
| `metadata.name` | Must literally be `<plural>.<group>` — not arbitrary |
| `spec.scope` | Whether instances of this CR live inside a namespace or are cluster-wide |
| `spec.group` | The API group new CRs will use in their `apiVersion` (`flights.com/v1`) |
| `spec.names.kind` | The `kind:` value people will actually write (`FlightTicket`) |
| `spec.names.plural` | What `kubectl get <plural>` uses |
| `spec.names.shortNames` | Optional short alias, e.g. `kubectl get ft` |
| `spec.versions[].served` | Whether this API version currently accepts requests |
| `spec.versions[].storage` | Which version's shape is actually stored in etcd — exactly one `true` across all versions |
| `spec.versions[].schema.openAPIV3Schema` | Validation rules — malformed CRs get rejected at creation, before ever reaching a controller |

```bash
kubectl create -f flightticket-custom-definition.yml
# customresourcedefinition "FlightTicket" created

kubectl get crd
kubectl api-resources | grep flights.com   # confirms the new kind + shortname are now registered
```

**Step 2 — now the Custom Resource itself can be created** (`flightticket.yml`):
```yaml
apiVersion: flights.com/v1        # <-- must match CRD's group + version exactly
kind: FlightTicket                 # <-- must match CRD's spec.names.kind exactly
metadata:
  name: my-flight-ticket
spec:
  from: Mumbai
  to: London
  number: 2
```

```bash
kubectl create -f flightticket.yml
# flightticket "my-flight-ticket" created

kubectl get flighttickets
kubectl get ft                      # shortName also works now
# NAME               AGE
# my-flight-ticket   24m
```

⚠️ **`STATUS: Pending` forever:** creating the CR only stores the object's *desired state* in etcd — nothing actually happens until a **Custom Controller** is running, watching that kind, and acting on it. A CRD with no controller behind it is just structured data storage; it won't book an actual flight. This is exactly the gap an **Operator** fills — a controller written with the specific operational knowledge of how to reconcile that resource for real.

```bash
kubectl explain flightticket        # works like any built-in type once the CRD is installed
kubectl explain flightticket.spec   # drills into the schema you defined
```

---

## 10. Helm — package manager for Kubernetes

```
Chart      -> a package (templated YAML + default values)
Release    -> a deployed instance of a chart
Repository -> where charts are hosted
```

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo nginx
helm install myrelease bitnami/nginx
helm upgrade myrelease bitnami/nginx --set replicaCount=3
helm rollback myrelease 1
helm list
helm uninstall myrelease
helm template mychart/                    # render YAML locally without installing
```

---

## 11. Kustomize — template-free YAML customization (built into `kubectl`)

```
base/              -> the plain, generic manifests
overlays/
  dev/             -> patches for dev (fewer replicas, dev image tag)
  prod/            -> patches for prod (more replicas, resource limits)
```

```yaml
# kustomization.yaml
resources:
- deployment.yaml
- service.yaml
namePrefix: dev-
commonLabels: {env: dev}
images:
- name: myapp
  newTag: dev-latest
patches:
- path: increase-replicas.yaml
```

```bash
kubectl kustomize ./overlays/dev
kubectl apply -k ./overlays/dev
```

**Patches** — how you actually change a field on an existing resource without rewriting the whole file. A patch targets a resource by `kind` + `name`, then applies one or more operations to it.

`api-depl.yaml` (the base resource being patched):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment      # <-- this is the field the patch below will change
spec:
  replicas: 1
  selector:
    matchLabels:
      component: api
  template:
    metadata:
      labels:
        component: api
    spec:
      containers:
      - name: nginx
        image: nginx
```

`kustomization.yaml` (the patch itself):
```yaml
patches:
- target:                         # WHICH resource to patch
    kind: Deployment
    name: api-deployment
  patch: |-                       # WHAT to change — JSON 6902 patch syntax
  - op: replace
    path: /metadata/name
    value: web-deployment
```

| Field | Meaning |
|---|---|
| `target.kind` / `target.name` | Selects the exact resource this patch applies to |
| `patch.op` | The operation: `replace`, `add`, or `remove` |
| `patch.path` | A JSON pointer to the field being changed (`/metadata/name` = `metadata.name`) |
| `patch.value` | The new value to set (omitted for `remove`) |

Result: `kubectl apply -k .` deploys the Deployment from `api-depl.yaml`, but with `metadata.name` overridden to `web-deployment` — the base file itself is never edited, so it can be reused unmodified across other overlays.

📌 Two patch styles exist: the JSON 6902 style above (`op`/`path`/`value` — precise, works well for renames or single-field tweaks) and the **strategic merge patch** style (write a partial YAML snippet shaped like the resource itself — better for adding/merging whole blocks like `env:` or `resources:`). Both are valid under `patches:`.
**Kustomize vs Helm:** Kustomize = patch plain YAML declaratively, no templating language, no logic. Helm = full templating engine with variables/conditionals/loops, better for distributing reusable packages to others.

---

## 12. Universal Troubleshooting Checklist

When *anything* is broken, work top-down:

```bash
kubectl get pods -o wide                    # is it even running? which node? what IP?
kubectl describe pod <name>                 # Events section at the bottom = the story of what went wrong
kubectl logs <pod> [-c container] [--previous]
kubectl get events --sort-by='.lastTimestamp'   # cluster-wide recent events
kubectl exec -it <pod> -- sh                # get a shell inside to check directly
```

| Symptom | Where to look first |
|---|---|
| `Pending` | `kubectl describe pod` → Events (often: no node has enough resources, or unmet affinity/taint) |
| `CrashLoopBackOff` | `kubectl logs <pod> --previous`, check probes, check command/args |
| `ImagePullBackOff` | Image name/tag typo, private registry auth (imagePullSecrets) |
| `ContainerCreating` stuck | Volume/PVC not binding — `kubectl describe pvc`, check StorageClass |
| Pod Running but Service can't reach it | Section 8.2 above — selector/labels/endpoints/readiness |
| `0/1 Ready` | Readiness probe failing — `kubectl describe pod`, check probe config vs actual app behavior |
| Deployment stuck mid-rollout | `kubectl rollout status`, check `maxUnavailable`/`maxSurge`, new Pods' probes failing |
| RBAC "forbidden" errors | `kubectl auth can-i ... --as=...`, check Role/RoleBinding subjects and verbs |

---

## 13. Quick Command Reference

```bash
# Imperative generators (fast YAML scaffolding for the exam)
kubectl run pod1 --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment d1 --image=nginx --replicas=3 --dry-run=client -o yaml > deploy.yaml
kubectl expose deployment d1 --port=80 --target-port=8080 --dry-run=client -o yaml > svc.yaml
kubectl create job j1 --image=busybox --dry-run=client -o yaml -- echo hi > job.yaml
kubectl create cronjob c1 --image=busybox --schedule="*/1 * * * *" --dry-run=client -o yaml -- echo hi
kubectl create configmap cm1 --from-literal=key=val --dry-run=client -o yaml
kubectl create secret generic s1 --from-literal=key=val --dry-run=client -o yaml
kubectl create serviceaccount sa1 --dry-run=client -o yaml
kubectl create role r1 --verb=get,list --resource=pods --dry-run=client -o yaml
kubectl create rolebinding rb1 --role=r1 --serviceaccount=default:sa1 --dry-run=client -o yaml

# General inspection
kubectl get all -n <ns>
kubectl explain <resource>.<field>          # inline docs for any YAML field
kubectl api-resources
kubectl diff -f file.yaml                    # preview changes before applying
kubectl apply -f file.yaml
kubectl delete -f file.yaml
```

---

## 14. Golden Rules

- Always check `spec.selector` vs `template.metadata.labels`/Pod labels first — mismatches break Deployments, ReplicaSets, and Services alike.
- Blank `kubectl get endpoints` output is *the* signal a Service is broken — go straight to comparing selector vs Pod labels.
- Liveness ≠ Readiness: one restarts the container, the other just pulls it out of Service rotation.
- Jobs/CronJobs need `restartPolicy: Never` or `OnFailure` — never `Always`.
- A Service never "knows about" a Deployment — it only watches Pod labels live, continuously.
- Use `--dry-run=client -o yaml` constantly — it's the fastest way to scaffold correct YAML under time pressure.
- `kubectl explain` beats trying to remember exact field names/nesting.