# Kubernetes Internals — Interview Prep Notes

> Hands-on tutorial notes. Each lesson: concept → command → what to observe → interview soundbite.

## Table of Contents
1. [Cluster Anatomy — the control plane](#lesson-1--cluster-anatomy)
2. [Reconciliation loop & controllers](#lesson-2--the-reconciliation-loop)
3. [API server & etcd](#etcd-storage-internals)
4. [Scheduling internals](#lesson--scheduling)
5. [Networking internals](#lesson--networking)
6. [Pod lifecycle & rolling updates](#lesson--pod-lifecycle)
7. [Observability internals](#lesson--observability)

---

## Lesson 1 — Cluster Anatomy

### The mental model
Kubernetes = **control plane** (the brain) + **worker nodes** (the muscle).

```
        CONTROL PLANE (the brain)                WORKER NODES (the muscle)
 ┌───────────────────────────────────┐      ┌──────────────────────────┐
 │  kube-apiserver  ← front door      │      │  kubelet   ← node agent  │
 │  etcd            ← source of truth │◄────►│  kube-proxy← networking  │
 │  scheduler       ← places pods     │      │  container runtime (CRI) │
 │  controller-mgr  ← reconciles      │      │  your Pods / containers  │
 └───────────────────────────────────┘      └──────────────────────────┘
```

Key rule: **Only the API server talks to etcd.** Everything else talks to the API server.

### Component cheat-sheet
| Component | Runs where | One-line job |
|-----------|-----------|--------------|
| `kube-apiserver` | control plane | REST front door; authN/authZ/admission; only one that reads/writes etcd |
| `etcd` | control plane | Distributed key-value store; cluster source of truth; Raft consensus |
| `kube-scheduler` | control plane | Decides which node each new Pod runs on |
| `kube-controller-manager` | control plane | Runs control loops (Deployment, ReplicaSet, Node, etc.) |
| `kubelet` | every node | Ensures its node's containers match desired spec; reports status |
| `kube-proxy` | every node | Implements Service networking (iptables/IPVS) |
| `CoreDNS` | as pods | Cluster DNS — resolves service names |

### Commands run
```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -n kube-system -o wide
```

### Observations (my sandbox = OpenShift, not vanilla K8s!)
- Cluster is **OpenShift** on GCP: `api.sb0120.caas.gcp.ford.com:6443`, runtime `cri-o`, OS `RHCOS`.
- **16 nodes**: 3 control-plane/master + 13 workers.
- Worker roles are specialized via **node labels**: `worker` (general), `infra` (routers/monitoring/registry), `el` (edge/egress), `kata` (VM-isolated pods).
- **Mixed architecture**: masters are `aarch64` (ARM), workers are `x86_64`. Scheduling respects the `kubernetes.io/arch` label.
- **`kube-system` is EMPTY** — OpenShift moves control-plane pods into dedicated Operator-managed namespaces:
  - `openshift-kube-apiserver`, `openshift-etcd`, `openshift-kube-scheduler`, `openshift-kube-controller-manager`
  - Networking: `openshift-ovn-kubernetes` (OVN-Kubernetes replaces plain kube-proxy).
- **Scheduler** runs on control-plane only (singleton brain, uses **leader election** — 3 replicas, 1 active).
- **Networking proxy** runs on **every node** (DaemonSet) because each node programs its own local routing rules.

### Interview soundbite
> "The control plane is the brain: API server is the only front door and the only
> component that talks to etcd, the source of truth. The scheduler assigns pods to
> nodes, and controllers reconcile desired vs actual state. Each worker node runs a
> kubelet that keeps its containers matching spec, plus kube-proxy for service networking."
### OpenShift vs vanilla K8s (know this — you're on OpenShift)
| Concept | Vanilla K8s | OpenShift |
|---------|-------------|-----------|
| Control-plane namespace | `kube-system` | `openshift-*` (per-component, Operator-managed) |
| Container runtime | containerd (common) | CRI-O |
| Node OS | any Linux | RHCOS (immutable, managed by MCO) |
| Networking | kube-proxy + CNI plugin | OVN-Kubernetes (kube-proxy built in) |
| "Namespace" UX | Namespace | Project (Namespace + extra metadata) |
| Who manages upgrades | you / kubeadm | Cluster Version Operator + Machine Config Operator |
| Security default | permissive | SCCs (Security Context Constraints) enforced |

### Deep dive: what the control-plane namespaces revealed
- **Static pods**: apiserver/etcd/scheduler pods are named `<component>-<nodename>`. The **kubelet** runs them directly from `/etc/kubernetes/manifests/` — **no scheduler involved**. Solves the bootstrap chicken-and-egg (API server can't schedule itself).
- **Multi-container pods**: `kube-apiserver` and `etcd` show `5/5` = 5 containers (main + sidecars for certs, health, metrics, logs). Proves a Pod can hold multiple containers sharing net/storage.
- **Revisioned rollouts** (OpenShift): `installer-<N>` and `revision-pruner-<N>` pods. Each config change bumps a revision number; installer pods rewrite static-pod manifests **one master at a time**; pruners clean old revisions. Saw a live rollout: `installer-113` completed sequentially across 3 masters and apiserver pods restarted in matching order = zero-downtime control-plane update.
- **`guard` pods**: enforce quorum safety (PDB-style) so rolling updates never drop etcd/apiserver below majority.

### etcd quorum (why 3 masters, odd numbers)
etcd uses **Raft**; every write needs a **majority** = `floor(N/2)+1`.
| Nodes | Quorum | Failures tolerated |
|-------|--------|--------------------|
| 3 | 2 | 1 |
| 4 | 3 | 1 (no gain over 3!) |
| 5 | 3 | 2 |
Even counts add cost + failure surface without extra resilience → always use **odd** (3/5/7).

---

## Lesson 2 — The Reconciliation Loop

### Core idea
Everything in K8s is a control loop: **`observe → diff → act`**, forever. You declare *desired* state; controllers drive *actual* → desired.

### Ownership chain (each layer reconciles only the one below it)
- **Deployment controller** → owns **ReplicaSets** (manages rollouts, rollback, revision history).
- **ReplicaSet controller** → owns **Pods** (keeps count = `.spec.replicas`; recreates deleted pods).
- Link is recorded in each object's **`ownerReferences`** → also drives **garbage collection**.

### How controllers "notice" changes (the real internals) ⭐
```
API Server ──(watch stream)──► Informer ──► Local Cache
                                  │
                                  ▼  event handlers
                              Work Queue (key = namespace/name)
                                  │
                                  ▼
                        worker: reconcile(key)
```
- **LIST + WATCH, not polling.** Informer lists once, then opens a long-lived **watch**; API server streams ADD/UPDATE/DELETE events (backed by etcd watch, tracked by **`resourceVersion`**).
- **Local cache** → controllers read cache, not API server (near-zero load). Periodic **re-list** self-heals missed events.
- **Work queue** = dedup + rate-limit + exponential backoff. Pushes object **keys**, not events.
- **Level-triggered, not edge-triggered** → acts on *current observed state*, not the specific event. This is WHY K8s is self-healing + eventually consistent.

### Commands
```bash
kubectl -n learn-k8s get events --sort-by=.lastTimestamp | tail -20
kubectl -n learn-k8s get pod -l app=web -o jsonpath='{range .items[*]}{.metadata.name}{"  rv="}{.metadata.resourceVersion}{"\n"}{end}'
kubectl -n learn-k8s get rs -o jsonpath='{.items[0].metadata.ownerReferences}' | python3 -m json.tool
```

### Interview soundbite
> "Controllers use an informer that does a LIST+WATCH against the API server, caches
> objects locally, and pushes changed keys onto a rate-limited work queue. Reconciliation
> is **level-triggered** — it acts on the observed current state, not the event itself —
> which is what makes Kubernetes self-healing and eventually consistent."

### Cascading deletion & the Garbage Collector
- `ownerReferences` chain is **layered**: Pod → owns ReplicaSet? No — Pod.ownerRef = ReplicaSet; RS.ownerRef = Deployment. Chain: **Deployment → RS → Pod**.
- `kubectl delete deployment` → API server → etcd marks it deleted → **garbage collector controller** (in kube-controller-manager) sees orphaned RS → deletes it → Pods orphaned → deleted. This is **cascading deletion**.
- **Propagation policies** (`--cascade`):
  - **Background** (default): delete owner now, GC cleans children async.
  - **Foreground**: owner stuck "deleting" (via `foregroundDeletion` **finalizer**) until children gone first.
  - **Orphan**: delete owner, keep children (strip their ownerRef).
- **Finalizers** = keys in `metadata.finalizers` that block removal from etcd until a controller does cleanup and removes the key. Cause of "stuck Terminating" namespaces/pods.

### Deep dive: resourceVersion (RV)
- **Opaque string** on every object's `metadata`. API contract: treat as opaque, don't do math/compare yourself.
- Backed by etcd's **global, cluster-wide, monotonically-increasing revision counter** — EVERY write to ANY key bumps it. So RV is a **global logical clock**, not per-object change count. Can't meaningfully compare RVs of two different objects.
- **Two jobs:**
  1. **Optimistic concurrency (CAS):** update sends object + its RV; API server accepts only if RV still matches etcd, else **409 Conflict** ("object has been modified"). Prevents lost updates without locks.
  2. **Watch resume token / bookmark:** a watch says "stream changes since RV=X"; the global ordered counter lets the API server resume exactly there.

### Deep dive: how controllers notice changes so fast (full pipeline)
```
etcd ──watch(push)──► kube-apiserver (1 watch/type, in-mem watch-cache)
     ──fan-out HTTP stream──► Reflector (LIST+WATCH) ──► DeltaFIFO
     ──► Indexer (local indexed cache) ──► informer handlers ──► Workqueue(key)
     ──► worker: reconcile(key) reads CURRENT cache state
```
1. **etcd watch**: native push over gRPC on the global revision; no polling; ms latency.
2. **API server = multiplexer**: keeps ONE etcd watch per resource type + in-memory **watch-cache (cacher)**; fans events out to thousands of clients over long-lived **HTTP chunked / HTTP2 stream** (`?watch=true&resourceVersion=N`), filtered by ns/label/field.
3. **Reflector** (in controller): LIST once (get collection RV bookmark) → WATCH from that RV → on drop or **410 Gone** (RV too old, history window passed) re-LIST + resume.
4. **Indexer / local cache**: in-memory, indexed by ns/label. Controllers read THIS (microseconds, zero API load) — the speed + scale trick.
5. **Handlers → workqueue**: OnAdd/Update/Delete just compute key (`ns/name`) and enqueue onto a **rate-limited, dedup, backoff** queue. Handlers do NO real work.
6. **Workers**: pop key → read CURRENT object from cache (not event payload) → diff → act → re-enqueue w/ backoff on error.

**Key nuance:** handlers are edge-triggered (tell you WHICH key), but reconcile is **level-triggered** (reads latest full state). 10 piled-up events → reconcile once against newest state. Workqueue decoupling = the genius.

### Interview soundbite (watch/RV)
> "etcd supports native watch on a global revision counter. The API server holds one watch
> per resource and fans events from an in-memory watch-cache to all clients over a streaming
> connection. Each controller's reflector does LIST+WATCH into a local indexed cache; handlers
> enqueue the object key onto a rate-limited workqueue, and workers reconcile by reading the
> current cached state. resourceVersion is the global bookmark powering resumable watches and
> optimistic-concurrency (CAS) on writes."

---

## API Server Request Flow

Every request (kubectl, controllers, kubelets) goes through the SAME pipeline. API server = only component that talks to etcd.

```
TLS → AuthN → AuthZ → Mutating admission → Schema validation → Validating admission → Persist(etcd, CAS on RV) → Response
```

1. **TLS / transport** — HTTPS `:6443`, mTLS (client cert or bearer token), connection terminated; downstream = chain of in-process handlers.
2. **AuthN (who are you?)** — chain of authenticators: client certs (CN=user, O=groups), bearer tokens (ServiceAccount JWT, OIDC/SSO), webhook. Output = `{username, uid, groups, extra}`. Fail → **401**. Only identity, no permissions.
3. **AuthZ (allowed?)** — chain, first explicit allow/deny wins: **RBAC** (verb+resource+namespace via Role/ClusterRole bindings), Node authorizer, Webhook, (OpenShift **SCC**). Fail → **403**.
4. **Mutating admission** — runs after authZ, before persist; **can modify** object. Built-ins: ServiceAccount (token volume), LimitRanger (default requests), DefaultStorageClass. **MutatingAdmissionWebhooks**: sidecar injection (Istio), SCC defaults.
5. **Schema validation** — OpenAPI schema: required fields, types, immutability. Bad → **422**.
6. **Validating admission** — last word, **can reject not modify**. Built-ins: ResourceQuota, PodSecurity. **ValidatingAdmissionWebhooks / ValidatingAdmissionPolicy (CEL)**: OPA/Gatekeeper, Kyverno. Reject → **403**.
7. **Persist to etcd** — serialize (protobuf internally), write `/registry/<res>/<ns>/<name>`, **CAS on resourceVersion**; global revision bumps → **watchers streamed the change**.
8. **Response** — persisted object + new RV/uid/defaults; `201`/`200`.

**Order mnemonic:** AuthN → AuthZ → Mutate → Validate-schema → Validate-admission → Persist. Mutating ALWAYS before validating (validate the final object).

**Extras worth citing:**
- **Server-Side Apply** (`kubectl apply`): sends only your fields; merges via **`managedFields`** (field ownership + conflict detection).
- **Aggregation layer**: some APIs (`metrics.k8s.io`) served by other servers via `APIService` proxy.
- **CRDs**: same pipeline; stored in etcd, served by CRD handler.
- **`--dry-run=server`**: runs stages 1–6 (incl. admission) but SKIPS persist. Great for policy testing.
- Webhook `failurePolicy: Fail|Ignore`.

### Commands
```bash
kubectl -n learn-k8s get pods -v=8 2>&1 | grep -E 'GET|POST|Response Status' | head
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations
kubectl -n learn-k8s create deployment dryrun-test --image=nginx --dry-run=server -o yaml | head
kubectl auth can-i create pods -n learn-k8s
```

### Interview soundbite (API server flow)
> "Every request hits the same pipeline: TLS → authentication (who) → authorization/RBAC
> (allowed?) → mutating admission (may modify, e.g. sidecar injection) → schema validation →
> validating admission (may reject, e.g. quotas/policies) → persist to etcd with a CAS on
> resourceVersion, which bumps the global revision and streams the change to all watchers.
> Mutating always precedes validating so policies see the final object."

### OpenShift SCC (Security Context Constraints) — extra admission gate
- RBAC says *whether* you can create a pod; **SCC** says *what kind* of pod (runAsUser/root, privileged, hostNetwork/hostPath, capabilities, SELinux, fsGroup ranges).
- Enforced at **admission**, so `kubectl auth can-i create pods` = yes (RBAC) but pod still rejected if it violates the SCC bound to its ServiceAccount.
- Classic error: `unable to validate against any security context constraint`. #1 reason a "valid" pod won't start on OpenShift. Default restrictive SCC = `restricted-v2`.

### Kubernetes extensibility (the 3 mechanisms) — cluster is a living example
1. **CRDs** — define new object types (e.g. CNPG `Cluster`, Tekton `Pipeline`, Knative `Service`).
2. **Admission webhooks** — intercept requests at stages 4/6 (Kyverno policy, StackRox/ACS security, OTel sidecar injection, pod-identity token injection, Multus multi-NIC, Kata config).
3. **Controllers/Operators** — reconcile the CRs (same LIST+WATCH loop as core controllers).
> Interview: "How is K8s extensible?" → CRDs (new types) + admission webhooks (intercept) + operators (reconcile). All plug into the SAME api-server pipeline.

### How operators get installed — OLM (Operator Lifecycle Manager)
**Two layers:** (1) OLM installs the OPERATOR; (2) the operator installs the APP.
- You DON'T create Loki/OTel pods directly. Install the **operator** (via OLM) → create a **CR** (`LokiStack`, `OpenTelemetryCollector`) → operator reconciles it into the pods.
- **OLM objects:**
  | Object | Role | Analogy |
  |--------|------|---------|
  | **CatalogSource** | repo of available operators (OperatorHub) | apt/yum repo |
  | **PackageManifest** | an operator's catalog entry (channels/versions) | package listing |
  | **Subscription** | "I want operator X, channel stable, auto-update" | `apt install X` |
  | **InstallPlan** | concrete install/upgrade plan (manual/auto approve) | "proceed? y/N" |
  | **ClusterServiceVersion (CSV)** | installed operator manifest: Deployment+RBAC+owned CRDs+version | installed pkg metadata |
  | **OperatorGroup** | which namespaces the operator watches | install scope |
- **Flow:** CatalogSource → Subscription → InstallPlan(approve) → CSV → deploys operator Deployment + RBAC + CRDs → operator watches → you create CR → app pods appear.
- **Reconciliation all the way down:** OLM reconciles Subscription→CSV→operator; operator reconciles CR→pods.
- **Core vs add-on:** add-ons (Loki, OTel, cert-manager, CNPG, Tekton) = **OLM** (Subscriptions). Core cluster operators (apiserver, etcd, dns, ingress, **monitoring/Prometheus**, OVN) = **Cluster Version Operator (CVO)** as **ClusterOperator** objects, NOT OLM.
```bash
kubectl get catalogsource -n openshift-marketplace
kubectl get csv -A | grep -iE 'loki|opentelemetry|tempo'
kubectl get subscription -A ; kubectl get installplan -A | head
kubectl get clusteroperators | head          # CVO-managed core operators
kubectl get lokistack -A ; kubectl get opentelemetrycollector -A
```
> Soundbite: "OpenShift add-ons install via OLM: a Subscription to an operator from a CatalogSource → OLM makes an InstallPlan and applies a ClusterServiceVersion that deploys the operator + CRDs. Then you create a CR (LokiStack/OpenTelemetryCollector) and the operator reconciles it into pods. Core operators are CVO-managed ClusterOperators. Reconciliation at every layer."

### CRD vs CR vs Controller vs Operator (clear these up)
| Term | What it is | OOP analogy |
|------|-----------|-------------|
| **CRD** | defines a new **type** (registers `LokiStack` kind in API) | class/blueprint |
| **CR** | an **instance** of that type (your config) | object |
| **Controller** | running **pod** with reconcile logic that watches CRs | the code/methods |
| **Operator** | = **Controller + the CRD(s) it owns**, packaged | class+methods bundled |
- **CRD alone does NOTHING** — a CR just sits in etcd until a **controller** watches + reconciles it. Operator = both together.

### Where the controller runs + how a CR becomes pods
- Controller pod runs ONCE in an **operator namespace** (e.g. `openshift-operators-redhat`), watches CRs **cluster-wide**.
- App pods run in the namespace where **you created the CR** (e.g. `openshift-logging`). Different namespaces — controller has cluster RBAC to reach across.
- **Chain (reconciliation feeding reconciliation):**
  ```
  create LokiStack CR → operator reconciles → creates StatefulSets/Deployments (NOT pods directly)
    → built-in controllers (StatefulSet ctrl, Deployment→ReplicaSet→Pod) → pods
    → scheduler binds → kubelet runs
  ```
- Operator sets **ownerReferences** on all it creates → back to the CR. Delete the CR → **GC cascades** → whole stack's pods deleted. (Same cascading deletion as Lesson 2.)
```bash
kubectl get pods -A | grep -i loki-operator                  # controller (operator ns)
kubectl -n openshift-logging get statefulset,deployment | grep loki   # child objects it created
kubectl -n openshift-logging get statefulset logging-loki-ingester -o jsonpath='{.metadata.ownerReferences}'  # owned by LokiStack CR
```

---

## etcd Storage Internals

### What etcd is
- Distributed **consistent key-value store**, **Raft** consensus, **CP** in CAP (rejects rather than serve stale).
- **MVCC**: keeps revision history (powers resourceVersion + watch). Storage = **bbolt B+tree** on disk + in-mem btree index (key→revisions).

### Key layout
- Every object at **`/registry/<resource>/<namespace>/<name>`**. Core types serialized as **protobuf**; CRDs as JSON.
- "List pods in ns X" = **range scan** over `/registry/pods/<ns>/` prefix.
- **Is `/registry/...` an etcd path?** It's the literal etcd **key**, but etcd is a **FLAT** kv store — no real dirs; the `/` are just characters. The **API server storage layer** invents the `/registry/` prefix (configurable via `--etcd-prefix`); etcd blindly stores `(key string)→(bytes)`, sorted lexicographically.
- Keys are **sorted** → LIST = etcd **range query** `[/registry/pods/ns/ , /registry/pods/ns0)`. The `/` just makes related keys sort adjacent so hierarchical-looking queries are efficient range scans.

### ⭐ Verified on OpenShift: prefix is `/kubernetes.io`, NOT `/registry`
- Vanilla/kubeadm uses `/registry/...`; **OpenShift sets `--etcd-prefix=/kubernetes.io`** → keys like `/kubernetes.io/pods/<ns>/<name>`. Live proof the prefix is API-server-owned + configurable.
- **CONFIRMED LIVE** (etcdctl inside etcd pod):
  ```
  /kubernetes.io/pods/cloudtooling-fuzzyhub/fuzzyhub-resolver-5f5ffdd7cb-62jch
  /kubernetes.io/pods/cloudtooling/psql-client
  ```
  Same object via API server REST (JSON, note resourceVersion + OVN annotation):
  ```
  kubectl get --raw "/api/v1/namespaces/cloudtooling/pods/psql-client"
  → metadata.resourceVersion=1550838611 (huge → global counter, cluster-wide writes)
  → annotation k8s.ovn.org/pod-networks = pod IP 172.24.15.214/24, mac, gateway (OVN-assigned; see networking lesson)
  ```
- **Exec into etcd** (OpenShift etcdctl container has certs+endpoints as env vars):
  ```bash
  oc -n openshift-etcd rsh -c etcdctl etcd-<master>
  etcdctl member list -w table
  etcdctl get / --prefix --keys-only | head -50
  etcdctl get /kubernetes.io/pods/learn-k8s/ --prefix --keys-only
  etcdctl get /kubernetes.io/pods/learn-k8s/ --prefix --limit=1 | strings | head   # protobuf → readable bits
  etcdctl endpoint status -w table --cluster   # shows global revision = RV source
  ```
- **Three addressing layers for the SAME object:**
  | Layer | Address |
  |-------|---------|
  | kubectl | `kubectl -n learn-k8s get pod web-xxx` |
  | API server REST (JSON) | `/api/v1/namespaces/learn-k8s/pods/web-xxx` |
  | etcd key OpenShift (protobuf) | `/kubernetes.io/pods/learn-k8s/web-xxx` |
  | etcd key vanilla | `/registry/pods/learn-k8s/web-xxx` |
  - apiserver translates REST path ↔ etcd key, and JSON ↔ protobuf.
- ⚠️ **etcd is READ-ONLY for humans.** Never `put`/`del` directly → bypasses validation, admission, and correct watch events → cluster corruption. Always go through the API server.

### MVCC / revisions / compaction (ties to RV + 410 Gone)
- Every write = new global revision (= RV). History kept for watch replay.
- **Compaction** (~every 5 min): drop revisions older than a point. Watch resuming from compacted rev → `mvcc: required revision has been compacted` → apiserver returns **410 Gone** → reflector re-LISTs. THIS is why 410 happens.
- **Defrag**: compaction frees revisions logically but leaves holes in bbolt; defrag physically reclaims disk. Run per-member, one at a time (briefly blocks that member).

### etcd disk lifecycle: compaction → defrag → quota (operational)
**Problem:** MVCC keeps every revision → busy cluster (heartbeats/leases/updates) writes thousands of revs/min → DB only grows → hits space quota → outage. Two cleanup stages:

**1. Compaction (logical):**
- Drops **historical revisions** < N; keeps CURRENT values (+ recent history for watches).
- Trigger: apiserver auto-compacts ~5 min (`--etcd-compaction-interval`); etcd `--auto-compaction` too.
- Frees pages **INSIDE** the bbolt file → file does NOT shrink on disk; space reused by future writes.
- Side effect = the `410 Gone` story.

**2. Defragmentation (physical):**
- Compaction leaves "holes"; `db size` (physical) stays big while `db size in use` (live) drops = fragmentation.
- Defrag rewrites bbolt compactly → returns space to **OS** → physical file shrinks.
- ⚠️ **Stop-the-world: blocks that member** while running → do **one member at a time**, leader last.
- OpenShift **cluster-etcd-operator auto-defrags** past a fragmentation threshold — rarely manual.
```bash
etcdctl endpoint status -w table --cluster   # DB SIZE vs DB SIZE IN USE = fragmentation gap
# etcdctl defrag --command-timeout=30s        # manual, blocks member (usually operator-managed)
```

**Space quota / NOSPACE alarm (the incident):**
- `--quota-backend-bytes` = max DB (default 2 GiB; OpenShift ~8 GiB).
- Exceed → etcd raises **`NOSPACE` alarm** → **rejects all writes (read-only)** → cluster can't schedule/update.
- Recovery: compact → defrag → **disarm** (`etcdctl alarm disarm`).
```bash
etcdctl alarm list   # empty = healthy; NOSPACE/CORRUPT = problem
```

### ⚠️ Retention window: ~5 MINUTES, not days (common misconception)
- Compaction removes only **old revision HISTORY** (superseded versions of keys) — NOT the current objects.
- **Current objects are NEVER auto-deleted** by etcd/compaction; they live until you `kubectl delete` (through the API).
- Kubernetes apiserver drives compaction via `--etcd-compaction-interval` = **default 5 min**. So etcd keeps only ~5 min of change history for watch replay.
- **Rule:** a watch disconnected > ~5 min → its bookmark revision is compacted → `410 Gone` → reflector re-LISTs. Short blips (seconds) resume from the apiserver **watch-cache** without hitting etcd.
- "History gone" ≠ "object gone." Data is safe; only the undo history is trimmed.

### 410 → relist: EXPECTED behavior, NOT a "gap"
- `410 Gone` → reflector **re-LISTs** (fresh bookmark) → resumes WATCH. This is normal/healthy **relist/resync**, by design — not an error.
- On relist: reflector gets full current state via LIST → **diffs vs local cache** (`Replace()` in DeltaFIFO) → synthesizes needed add/update/**delete** events so cache matches reality.
- Consequence: **intermediate transitions can be collapsed/missed**, but **final state is always correct**.
  ```
  Disconnect window: Pod X created→updated→deleted (3 events).
  After relist: X not in LIST → reflector never sees X.
  Net: correct final state (X gone); the intermediate "X briefly existed" is LOST — and that's FINE.
  ```
- Correct terms: "relist/resync" (expected), "missed/collapsed events" (✅), NOT "gap" (❌ no correctness gap).
- **This is exactly WHY controllers are level-triggered** — they reconcile observed current state, so missing intermediate events is harmless. Edge-triggered would break; level-triggered just re-reads reality.

### Backup (snapshot) — detail
- Point-in-time **consistent copy** of the whole keyspace → 1 file. `etcdctl snapshot save snap.db` (captures a consistent revision from one member).
- **OpenShift:** `/usr/local/bin/cluster-backup.sh <dir>` on a master → saves etcd snapshot **+ static-pod resources/certs metadata** needed for clean restore.
- Back up FREQUENTLY — anything written after the snapshot is lost on restore.

### Restore (disaster recovery flow) — detail
```
1. Pick ONE surviving master → /usr/local/bin/cluster-restore.sh <backup-dir>
   → wipes its etcd, restores snapshot as a NEW single-member cluster (quorum of 1).
2. That master = sole source of truth.
3. cluster-etcd-operator re-adds the other 2 masters: they wipe old etcd data,
   join as fresh followers, sync full keyspace from the restored member.
4. Quorum of 3 rebuilt.
```
- **Why the dance?** Naively restarting old members → conflicting Raft logs → etcd won't form consistent cluster. Restore forces ONE authoritative copy, others rebuild from it.
- Point-in-time loss: anything between snapshot & disaster is gone. OpenShift also handles cert regen + revision rollout.

| Operation | Layer | Frequency | Effect |
|-----------|-------|-----------|--------|
| Compaction | logical | auto ~5 min | drops old revs; frees pages in file; causes `410 Gone` |
| Defrag | physical | when fragmented (operator) | shrinks file on disk; blocks member |
| Snapshot | backup | scheduled | point-in-time copy of keyspace |
| Restore | recovery | disaster only | single-member → re-add others → quorum |

### Interview soundbite (etcd ops)
> "etcd is MVCC so its DB only grows: compaction runs every few minutes to drop old revision
> history (which is what makes stale watches get 410 Gone), but that only frees space inside
> the file — defrag physically shrinks it and blocks the member, so you do one at a time. If
> the DB hits its quota, etcd raises NOSPACE and goes read-only. DR is a point-in-time etcd
> snapshot; restore brings up one member as the source of truth, then re-adds the others so
> they rebuild quorum from it."

### Encryption at rest
- OFF by default → Secrets sit in **plaintext** in etcd files. Enable via `EncryptionConfiguration` (aescbc/aesgcm/**KMS** envelope). OpenShift: set `encryption.type` on `APIServer` CR → encrypts + auto-rotates keys.

### Write path / quorum (ties to 3 masters)
- Writes forwarded to **leader**; Raft replicates log to followers; **committed only after majority (2/3) acks** → applied → ack client.
- Lose 2/3 → no quorum → etcd read-only/unavailable (stays consistent over available).
- **Leases** = TTL keys → node heartbeats (`kube-node-lease`) + **leader election** (only 1 active scheduler/controller-manager among 3).

### Leader election (active/passive HA) — seen live via Lease objects
- 3 scheduler pods run (`3/3`) but only ONE is active — the one holding the `kube-scheduler` **Lease**. Others = hot standbys retrying to acquire. Failover in seconds.
- Independent per component (scheduler election ≠ controller-manager election); one node can coincidentally hold several.
- **HOLDER string** = `<nodename>_<processUUID>`; UUID regenerates on restart → distinguishes "same holder renewing" from "new claimant".
- **Mechanism** (built on Lease + CAS on resourceVersion):
  1. ACQUIRE: read Lease; if empty/expired → CAS-write self as holder.
  2. RENEW: holder updates `renewTime` every ~10s (CAS write).
  3. Followers watch; if `now - renewTime > leaseDurationSeconds` → expired → race to acquire.
  - Two simultaneous claims → both CAS → only one wins, other gets **409 Conflict**. etcd optimistic concurrency IS the lock (no external lock service).
- Lease fields: `holderIdentity`, `leaseDurationSeconds`, `renewTime`, `acquireTime`, `leaseTransitions`.
- **Why active/passive?** Scheduler/controllers make GLOBAL mutating decisions → 2 active could bind same pod to different nodes. Exactly-one-writer + warm spares. (vs kube-proxy/OVN = active on every node, only program LOCAL state, no conflict.)
```bash
kubectl -n openshift-kube-scheduler get lease kube-scheduler -o jsonpath='{.spec.holderIdentity}{"  renew="}{.spec.renewTime}{"\n"}'
```

### Backup & restore (DR question)
- Backup = etcd **snapshot** (`etcdctl snapshot save`; OpenShift `cluster-backup.sh`). Point-in-time.
- Restore = restore single member → rebuild quorum. Anything after snapshot is LOST → snapshot often.
- **Golden line (precise):** "etcd is the source of truth for all **API objects** (spec+status); restoring it restores the cluster's **configuration/desired state**."

### ⚠️ What is NOT in etcd (nuance: "restore = whole cluster" is a simplification)
- **IN etcd:** all API objects (pods/deploys/svc/secrets/RBAC/CRDs/PV+PVC *objects*), spec AND status.
- **NOT in etcd:**
  | Thing | Actually lives | Restored by etcd restore? |
  |-------|----------------|---------------------------|
  | running container processes | node (kubelet + CRI-O) | ❌ |
  | container images | node local disk | ❌ |
  | **PersistentVolume DATA (bytes)** | storage backend (cloud disk/Ceph/NFS) | ❌ (etcd only has the PV/PVC *object*) |
  | pod/container logs | node filesystem | ❌ |
  | metrics | Prometheus | ❌ |
  | events | etcd but TTL ~1h | partial |
- **After restore:** apiserver serves OLD desired state → kubelets report actual (their pods never stopped) → **controllers reconcile** actual→desired (re-pull images, recreate pods) → **PV data untouched** (was never in etcd) so DBs survive & re-mount same volume.
- Interview: etcd restore rolls back *config/desired state*; data plane (containers/images/PV bytes) is NOT in etcd and reconverges via reconciliation.

### "Manual etcd cleanup" — the 3 meanings
| If they mean... | It's really... | Tool |
|-----------------|----------------|------|
| "DB is huge, reclaim space" | compact + defrag + disarm alarm | `etcdctl` (in pod) |
| "delete piled-up objects clogging etcd" | **object** cleanup | `kubectl`/API (**safe**) |
| "a member is broken" | member remove + re-add | `etcdctl` / operator |

**1. Storage maintenance** (reclaim disk):
```bash
etcdctl compact <rev>                 # drop history (logical)
etcdctl defrag --command-timeout=60s  # shrink db file (per member, one at a time)
etcdctl alarm disarm                  # clear NOSPACE after freeing space
```
**2. Object cleanup** (most common day-to-day) — junk that bloats etcd: completed/failed Pods (Jobs/CronJobs), old Events, finished Jobs, orphaned ConfigMaps/Secrets (Helm release secrets, cert-manager), stale CRs.
```bash
kubectl delete pods --field-selector=status.phase=Succeeded -A
kubectl delete jobs --field-selector=status.successful=1 -A
kubectl get --raw=/metrics | grep apiserver_storage_objects | sort -t'"' -k2 -n | tail -20  # what's bloating etcd
```
⚠️ ALWAYS via API/kubectl, NEVER raw `etcdctl del` (API deletion fires watch events, runs finalizers, keeps cache consistent).
**3. Member cleanup**: `etcdctl member remove <id>` then re-add fresh → re-syncs from healthy members. OpenShift cluster-etcd-operator usually automates.

> Soundbite: "Manual etcd cleanup = reclaim disk (compact/defrag/disarm) OR delete accumulated objects (completed pods, finished jobs, orphaned secrets) that bloat etcd — object deletion always through the API server, never editing etcd keys directly."

### Commands
```bash
kubectl get leases -A | head
kubectl -n openshift-kube-scheduler get lease           # who's the active leader
kubectl get apiserver cluster -o jsonpath='{.spec.encryption}{"\n"}'   # encryption-at-rest?
kubectl get --raw='/metrics' | grep apiserver_storage_objects | head   # etcd object footprint
```

### Interview soundbite (etcd)
> "etcd is a Raft-based consistent key-value store; the API server keys objects under
> /registry/<resource>/<ns>/<name> as protobuf. It's MVCC — every write bumps a global
> revision (that's resourceVersion), and old revisions are compacted every few minutes,
> which is exactly why a stale watch gets 410 Gone. Writes commit only after quorum, so
> you run an odd number of members, and disaster recovery is periodic etcd snapshots."

---

## Lesson — Networking

### The 3 network ranges (verified on this cluster)
| Network | CIDR (this cluster) | Purpose |
|---------|---------------------|---------|
| **Pod network** | `172.24.0.0/14` | every Pod gets a unique real IP (OVN-assigned) |
| **Service network** | `172.30.0.0/16` | stable virtual ClusterIPs for Services |
| **Node network** | `10.0.145.x` | actual node NICs |
- **Flat network model:** every Pod has its own IP; every pod reaches every other pod directly, **no NAT between pods**. OVN-Kubernetes implements it here.

### ⭐ IP ranges decoder (never be confused again)
| IP starts with… | It's a… | Real / Virtual | Examples seen |
|-----------------|---------|----------------|---------------|
| **`10.0.x`** | **Node** (VM NIC) — or a **hostNetwork** pod using the node IP | real | `10.0.145.30`, `10.0.145.84` |
| **`172.24.x`** | **Pod** (a normal pod's own IP) | real | web pod `172.24.15.204`, psql `172.24.15.214` |
| **`172.30.x`** | **Service** (virtual ClusterIP, no NIC owns it) | **virtual** (NAT rule only) | web svc `172.30.21.229`, DNS `172.30.0.10` |
| **`100.64.x`** | OVN internal/masquerade | internal | in pod route annotation |
- Shortcut: **`.30` = Service, `.24` = Pod, `10.0` = Node.**
- **Why some pods show `10.0.x` and others `172.24.x`:** normal pod → its OWN pod IP (`172.24.x`); **`hostNetwork: true`** pod → shares the NODE's IP (`10.0.x`). hostNetwork examples: `kube-apiserver`, `etcd`, `ovnkube-node` (they ARE the node's plumbing). Same "Pod" type, different net namespace. (That's why apiserver static pod = `10.0.145.84` but its `installer`/`guard` sidecapods = `172.24.x`.)
- Node & Pod IPs are **real** (routable/pingable). Service IPs are **virtual** — can't ping, but connecting works because OVN DNATs to a real pod IP.

### Services solve pod ephemerality
- Pods die/respawn with NEW IPs → never rely on pod IP. **Service = stable ClusterIP** (never changes) that load-balances to current ready pods.
- **Service selector** (`app=web`) → matches pods → **EndpointSlice controller** (in kube-controller-manager) keeps the **EndpointSlice** = list of current pod IPs + ready flags.
- **EndpointSlice** replaced the older flat `Endpoints` object (scales better: sharded, ~100 endpoints per slice).
- ⭐ Not-ready pods STILL appear in the EndpointSlice but with `conditions.ready: false`. **Only `ready:true` endpoints receive traffic.** → this is WHY readiness probes gate Service traffic. (Saw live: crashing pods listed but ready:false → ClusterIP had 0 usable backends.)

### Live mapping observed
```
Service web  ClusterIP 172.30.46.62   (service net 172.30/16)
EndpointSlice endpoints 172.24.x.x     (pod net 172.24/14) — matches pod IPs exactly
```

### The dataplane: how a request actually flows
```
curl web.learn-k8s.svc.cluster.local
 1. DNS (CoreDNS): name -> 172.30.46.62 (ClusterIP)
 2. connect 172.30.46.62:80  (VIRTUAL IP — no NIC owns it!)
 3. node dataplane DNATs ClusterIP -> one READY endpoint (e.g. 172.24.0.220:8080)
 4. routed over OVN overlay to that pod
 5. server responds 200
```
**A Service is NOT a running proxy** — it's a **virtual IP implemented as NAT/flow rules on every node**:
- Vanilla K8s: `kube-proxy` programs **iptables** (or IPVS) → DNAT ClusterIP → random ready endpoint.
- **This cluster (OVN-Kubernetes):** no kube-proxy iptables; **OVN load-balancers** in the virtual switch do the DNAT. Same idea.
- No single proxy in the path → fast, no SPOF. Distributed L4 load-balancing via NAT.

### CNI vs kube-proxy (two separate jobs OVN combines)
- **Vanilla K8s has NO default CNI** — Kubernetes defines the CNI spec; you must install a plugin or pods stay NotReady. `kubenet` = deprecated basic option. Real: **Calico** (common), **Flannel** (simple), **Cilium** (eBPF), cloud CNIs (AWS VPC/Azure/GKE). OpenShift = **OVN-Kubernetes**.
- **Two jobs:**
  | Job | Vanilla K8s | This cluster (OVN-K8s) |
  |-----|-------------|------------------------|
  | Pod networking (assign IPs, flat net) = CNI | a CNI plugin | OVN-Kubernetes |
  | Service routing (ClusterIP→pod DNAT) = kube-proxy | `kube-proxy` DaemonSet (iptables/IPVS) | **OVN-Kubernetes (built-in, NO kube-proxy)** |
- **OVN-K8s & Cilium REPLACE kube-proxy** → no kube-proxy DaemonSet, no iptables Service chains; DNAT via OVN load-balancers in the dataplane. Flannel does only pod-net → still needs separate kube-proxy.
- Confirm: `kubectl get ds -A | grep kube-proxy` → empty on OVN; `kubectl -n openshift-ovn-kubernetes get pods` = the DaemonSet doing BOTH.

### DNS (CoreDNS)
- Every pod's `/etc/resolv.conf` → cluster DNS (CoreDNS, `openshift-dns` ns on OpenShift).
- Name pattern: `<service>.<namespace>.svc.cluster.local`. `search` domains let you shortcut (`web`, `web.learn-k8s`).
- CoreDNS watches Services/EndpointSlices → answers name → ClusterIP (and headless svc → pod IPs).
- ⭐ **DNS is itself a Service:** `nameserver 172.30.0.10` is in the SERVICE net → CoreDNS reached via its own ClusterIP (`dns-default` svc in `openshift-dns`), which OVN DNATs to a CoreDNS pod. Nice recursion.
- ⚠️ **`options ndots:5`** = names with <5 dots try **search domains FIRST** before treating as absolute. A 4-dot FQDN like `web.learn-k8s.svc.cluster.local` triggers several wasted NXDOMAIN lookups (append each search domain) before resolving. Classic K8s DNS-latency footgun. **Fix:** trailing dot (`...cluster.local.` = absolute) or use short intra-ns names.
- ClusterIP is stable for the **life of the Service object**; delete+recreate the Service → NEW ClusterIP (saw 172.30.46.62 → 172.30.21.229). Pods behind it churn freely without changing the VIP.

### Why `172.30.0.10` is in every pod's resolv.conf (bootstrap)
- **kubelet writes it** using its `clusterDNS` setting (`--cluster-dns=172.30.0.10`) when pod `dnsPolicy: ClusterFirst` (default).
- That IP = **ClusterIP of the DNS Service** (`dns-default`/`kube-dns`), a **pinned well-known address** (conventionally 10th IP of service CIDR).
- **Must be static** → bootstrap chicken-and-egg: a pod can't discover the DNS server via DNS, so the address is hardcoded, not resolved.
- Traffic to `172.30.0.10:53` is DNATed by OVN/kube-proxy → a ready CoreDNS pod, like any Service.
- **dnsPolicy:** ClusterFirst (default, uses cluster DNS) | Default (node's resolv.conf) | None (custom `dnsConfig`) | ClusterFirstWithHostNet (cluster DNS even for hostNetwork pods).

### Service types (quick)
- **ClusterIP** (default): internal VIP only.
- **NodePort**: opens a port on every node → ClusterIP.
- **LoadBalancer**: external LB → NodePort → ClusterIP.
- **Headless** (`clusterIP: None`): no VIP; DNS returns pod IPs directly (StatefulSets).
- OpenShift **Route** / K8s **Ingress**: L7 HTTP routing on top of Services.

### Commands
```bash
kubectl -n learn-k8s get svc web
kubectl -n learn-k8s get endpointslice -l kubernetes.io/service-name=web -o wide
kubectl -n learn-k8s get endpointslice -l kubernetes.io/service-name=web -o yaml | grep -A3 conditions
kubectl get network.operator cluster -o jsonpath='{.spec.defaultNetwork.type}{"\n"}'  # OVNKubernetes
kubectl -n openshift-dns get pods -o wide | head
```

### Interview soundbite (networking)
> "Every pod gets a real IP on a flat network — no NAT between pods. A Service is a stable
> virtual ClusterIP, not a proxy process: it's NAT rules (iptables/IPVS via kube-proxy, or
> OVN load-balancers) programmed on every node that DNAT the ClusterIP to a ready endpoint
> pod IP. The EndpointSlice controller keeps the backend list in sync with ready pods, and
> CoreDNS resolves the service name to the ClusterIP. Only ready endpoints get traffic, which
> is why readiness probes gate Service membership."

### NetworkPolicy (the pod firewall) — demoed live on OVN
- **Default: wide open** — any pod → any pod (flat net). NetworkPolicy restricts it.
- ⭐ **Whitelist + "default-deny once selected":** if NO policy selects a pod → allow all. The moment ANY policy selects it for a direction (Ingress/Egress) → that direction becomes **deny-all except explicit allow rules**.
- **Enforced by the CNI, not core K8s.** OVN/Calico/Cilium enforce; **Flannel ignores** (policies silently do nothing). Object exists regardless.
- Namespaced; selects pods via `podSelector`; allow rules via `from`/`to` (podSelector, namespaceSelector, ipBlock) + ports.
- **Live demo results** (learn-k8s, `web` pods):
  | Test | Result | Why |
  |------|--------|-----|
  | baseline (no policy) | 200 | default open |
  | `web-deny-all` (Ingress, no rules) | 000 | pod now default-deny ingress |
  | from `role=client` | 200 | matched allow rule |
  | from `role=other` | 000 | not whitelisted |
  | delete both policies | 200 | no policy selects pod → default open again |
- `000` = OVN drops the packet (connection times out), NOT a server error.
```yaml
# default-deny ingress for app=web:
spec: { podSelector: { matchLabels: { app: web } }, policyTypes: [Ingress] }
# allow only role=client:
spec: { podSelector: {matchLabels: {app: web}}, policyTypes: [Ingress],
        ingress: [ { from: [ { podSelector: { matchLabels: { role: client } } } ] } ] }
```
> Soundbite: "NetworkPolicy is a whitelist firewall enforced by the CNI. Pods are open by default; once any policy selects a pod for a direction, it's deny-all except your explicit allow rules. OVN/Calico/Cilium enforce it; Flannel ignores it."

---

## Lesson — Observability

### 3 pillars
| Pillar | Answers | Tools on this cluster |
|--------|---------|------------------------|
| **Metrics** (numbers) | how much/many? CPU, mem, req rate, p99 | Prometheus (rhobs), Grafana, Dynatrace |
| **Logs** (text) | what did it say? | Loki (LogQL) |
| **Traces** (spans) | where did the request go? latency across services | Tempo, OpenTelemetry, Jaeger, Dynatrace |
- **OpenTelemetry (OTel)** = vendor-neutral standard to produce+ship all three (collector: receive→process→export).

### The pull model + `/metrics`
- Almost every K8s component exposes a **`/metrics`** HTTP endpoint in **Prometheus text format** (`name{labels} value`).
- **Prometheus SCRAPES** (pull, not push): HTTP-GETs each target's /metrics every ~30s → stores as time-series (TSDB) with labels.
- **Target discovery = LIST+WATCH the API** (same informer pattern as controllers) via **ServiceMonitor/PodMonitor** CRs (Prometheus Operator reconciles them). New matching pod → auto-scraped, no config change/restart. (CRD+operator+reconcile pattern again.)

### ⭐ node-exporter vs kube-state-metrics (classic interview trap)
| | node-exporter | kube-state-metrics |
|---|---------------|---------------------|
| Runs as | **DaemonSet** (per node) | Deployment (watches API) |
| Metrics about | the **host/OS**: CPU/RAM/disk/net of the machine | **K8s objects**: #replicas, pod phase, deploy status |
| Source | `/proc`, `/sys` | LIST+WATCH API server |
| Example | `node_cpu_seconds_total` | `kube_deployment_status_replicas` |
- Memorize: node-exporter = "how's the **hardware**?"; kube-state-metrics = "what's the **cluster state**?"

### ⭐ metrics-server vs Prometheus (TWO separate metric paths)
| | metrics-server | Prometheus |
|---|----------------|------------|
| API | **metrics.k8s.io** (aggregation layer) | PromQL/HTTP |
| Data | *current* CPU/mem only, NO history | full time-series history |
| Powers | **`kubectl top`, HPA autoscaling** | dashboards, alerts |
| Storage | in-memory ~15s | TSDB on disk |
- HPA gets pod CPU from **metrics-server via metrics.k8s.io aggregation API**, NOT Prometheus.

### Supporting components (seen live)
- **Thanos-querier**: unifies multiple Prometheus + **long-term storage** + global query (Prometheus local data is short-term).
- **Alertmanager** (HA, gossip): Prometheus fires rules → Alertmanager does **dedup/group/silence/route** to Slack/PagerDuty/email.
- **telemeter-client**: ships metrics subset to Red Hat (phone-home).
- **Loki**: logs, microservice arch (distributor→ingester→querier→query-frontend→**compactor** [compacts index/chunks, like etcd compaction for logs]).
- **OTel collector**: receive→process→export telemetry (traces/metrics/logs) to backends (Tempo/Loki/Dynatrace).

### `openshift-monitoring` pods — full component reference
| Pod | Role |
|-----|------|
| `prometheus-k8s-0/1` | Scraper + **TSDB** (time-series DB). 2 replicas = HA, each scrapes independently. |
| `prometheus-operator` | Reconciles **ServiceMonitor/PodMonitor** CRs → Prometheus scrape config. |
| `prometheus-operator-admission-webhook` | Validates PrometheusRule/monitor CRs at admission. |
| `node-exporter` (DaemonSet, 1/node) | **HOST/OS** metrics: CPU/mem/disk/net from `/proc`,`/sys`. |
| `kube-state-metrics` | **K8s OBJECT** metrics (replicas, pod phase, deploy status) via LIST+WATCH. |
| `openshift-state-metrics` | OpenShift-specific object metrics (routes, builds, etc.). |
| `metrics-server` (2) | Serves **metrics.k8s.io** → powers `kubectl top` + **HPA**. Current CPU/mem only, no history. |
| `alertmanager-main-0/1` | Receives fired alerts → dedup/group/silence/**route** to receivers. HA gossip. |
| `thanos-querier` | Global/unified query across Prometheus + long-term store. |
| `cluster-monitoring-operator` (CMO) | Top-level operator managing the whole monitoring stack. |
| `openshift-monitoring-cr-controller-manager` | Reconciles monitoring CRs (user-workload config). |
| `telemeter-client` | Sends anonymized metrics subset to Red Hat. |
| `monitoring-plugin` | Console UI plugin for the monitoring dashboards. |

### Loki pods — full component reference
| Pod | Path | Role |
|-----|------|------|
| `logging-loki-gateway` | both | Auth + routing front door (nginx-like). |
| `logging-loki-distributor` | write | Receives pushed logs, validates, shards to ingesters. |
| `logging-loki-ingester` | write | Batches logs into **chunks**, flushes to object storage. |
| `logging-loki-query-frontend` | read | Splits big queries, queues, caches. |
| `logging-loki-querier` | read | Executes LogQL: merges recent (ingester) + old (storage). |
| `logging-loki-index-gateway` | read | Serves the **index** to queriers. |
| `logging-loki-compactor` | maint | Compacts index + enforces **retention/deletion** (like etcd compaction). |
| `vector` / `collector` (DaemonSet) | ingest | Per-node agent: tails `/var/log/pods`, **pushes** logs to distributor. |

### Commands
```bash
kubectl get pods -n openshift-monitoring
kubectl get servicemonitors -A | head
kubectl top nodes ; kubectl top pods -n learn-k8s   # metrics-server path
kubectl get opentelemetrycollectors -A
kubectl get pods -A | grep -iE 'tempo|loki|grafana|otel|dynatrace'
```

### Interview soundbite (observability)
> "K8s metrics are pull-based: components expose /metrics in Prometheus format and Prometheus
> scrapes them, discovering targets by watching the API via ServiceMonitors. node-exporter gives
> host metrics, kube-state-metrics gives object metrics. Separately, metrics-server serves
> metrics.k8s.io for kubectl top and HPA. Thanos adds long-term/global query, Alertmanager routes
> alerts, Loki handles logs, and OTel/Tempo handle traces."

### Loki (logs) — "Prometheus for logs"
- ⭐ **Key design:** indexes only **LABELS** (metadata), NOT full log text. Log content = compressed chunks in **object storage** (GCS/S3), scanned at query time. → cheap + scalable vs Elasticsearch (which indexes everything).
- A "stream" = a label set (like a Prometheus series). Query language **LogQL** mirrors PromQL: `{ns="x",app="web"} |= "error"` (label selector indexed, `|=` filter scans chunks).
- **Logs PUSH, metrics PULL:** a per-node **collector DaemonSet (Vector**; older: Fluentd/Promtail) tails `/var/log/pods/...`, adds K8s labels, **pushes** to Loki. (Prometheus scrapes/pulls.)
- **Microservices (write vs read):**
  - Write: Vector → gateway → **distributor** (validate/shard) → **ingester** (batch→chunks→object storage)
  - Read: LogQL → gateway → **query-frontend** (split/cache/queue) → **querier** (merge ingester recent + storage old) → **index-gateway** serves index
  - **compactor**: compacts index + enforces **retention/deletion** — analogous to **etcd compaction**. (This cluster's `logging-loki-compactor-0` crash-looping 4481x → compaction+retention broken.)
- Two instances seen: `logging-loki` (app logs) + `workflow-loki` (pipeline logs) — isolated for scale/retention.
```bash
kubectl get pods -n openshift-logging -o wide | grep -iE 'collector|vector'
kubectl get lokistack -A
kubectl -n openshift-logging logs logging-loki-compactor-0 --tail=20   # debug the crashloop
```
> Soundbite: "Loki indexes only labels, not log text, storing compressed chunks in object storage — cheap and scalable. A per-node Vector collector pushes container logs to distributor→ingester→storage; LogQL queries go query-frontend→querier. Logs push, metrics pull. The compactor compacts the index and enforces retention like etcd compaction."

### Distributed tracing / APM (OTel, Tempo, Dynatrace)
- **Trace** = one request's end-to-end journey; **Span** = one timed unit (function/HTTP/DB call) with start/end + attributes. Spans sharing a **trace-id** form a tree → shows WHERE latency is (metrics/logs can't).
- **Context propagation:** trace context travels in HTTP header **`traceparent`** (W3C): `00-<trace-id>-<span-id>-<flags>`. Service injects it on outgoing calls; next service makes its span a child (same trace-id). Backend reconstructs the tree.
- **OpenTelemetry (OTel)** = vendor-neutral standard. Pipeline:
  ```
  app (OTel SDK/agent) → OTel COLLECTOR [receivers → processors(batch/filter) → exporters] → backends (Tempo/Prometheus/Loki/Dynatrace)
  ```
  - Collector = the `otel`/`clustermetrics` statefulset pods. **OTLP** = the wire protocol.
  - Emit once, export to many backends → no lock-in.
- ⭐ **Auto-instrumentation = MUTATING ADMISSION WEBHOOK** (same mechanism as Istio sidecar!). Pod annotation `instrumentation.opentelemetry.io/inject-java: "true"` → OTel operator webhook (admission stage 4) injects an **init-container** (copies agent) + env (`JAVA_TOOL_OPTIONS=-javaagent:...`, OTLP endpoint) → app starts instrumented, no code change. (This is the `mpod.kb.io` webhook seen earlier.)
- **Dynatrace**: commercial APM. `DynaKube` CR + operator. **OneAgent DaemonSet** (per node) auto-instruments at OS/process level; optional app-level injection via mutating webhook. Ingests OTLP (acts as OTel backend). Extras: Smartscape (dependency map), Davis (AI root-cause). *(NOT on this cluster — this cluster uses **Tempo** as the trace backend.)*
- **Pull vs push (why):** metrics = sampled **state** → pull (Prometheus scrape; doubles as health-check + discovery). logs/traces = ephemeral **events** (gone if not captured) → push the instant they happen.
- **Full picture:** metrics=pull(/metrics→Prometheus); logs=push(Vector→Loki); traces=push(OTel SDK→collector→Tempo/Dynatrace).
```bash
kubectl -n openshift-ford-cluster-collectors get opentelemetrycollector otel -o yaml | grep -A30 'config:'
kubectl get instrumentations -A ; kubectl get dynakube -A 2>/dev/null
kubectl get pods -A | grep -iE 'tempo|dynatrace'
```
> Soundbite: "Tracing propagates a W3C traceparent header so each service's span links to one trace-id, reconstructing the request tree. OpenTelemetry standardizes it: apps emit OTLP to a Collector (receive→process→export) that fans out to Tempo, Dynatrace, etc. Auto-instrumentation uses a mutating admission webhook to inject the agent — same mechanism as Istio sidecars. Dynatrace adds a OneAgent DaemonSet and AI root-cause on top."

---

## Lesson — Scheduling

### Two-phase pipeline
Pod created with empty `.spec.nodeName` → scheduler picks a node:
```
All nodes → FILTER (predicates): eliminate nodes that CAN'T run it → SCORE (priorities): rank 0-100 → BIND: write .spec.nodeName
```
- **Filter examples:** NodeResourcesFit (cpu/mem), TaintToleration, nodeSelector/affinity, arch (arm64/amd64), ports, volume zone.
- **Score examples:** LeastAllocated (spread), PodTopologySpread (zones), ImageLocality (image cached), InterPodAffinity.
- **Scheduler does NOT run the pod** — only sets `.spec.nodeName`. kubelet on that node watches → starts it. **Scheduler = matchmaker; kubelet = executor.**

### 3 placement mechanisms
| Mechanism | Direction | Set on | Meaning |
|-----------|-----------|--------|---------|
| **nodeSelector / nodeAffinity** | Pod picks nodes | Pod | "I WANT nodes with label X" (attraction) |
| **Taint** | Node repels pods | Node | "keep pods OFF unless they tolerate this" |
| **Toleration** | Pod accepts taint | Pod | "I can tolerate taint X" (permission slip) |
- Taint = node's "keep out" sign; toleration = pod's "visitor pass". Toleration only ALLOWS, doesn't ATTRACT → pair with nodeSelector/affinity to actually pin.

### Taint effects (interview favorite)
| Effect | Blocks new pods? | Evicts running pods? | Use |
|--------|------------------|----------------------|-----|
| **NoSchedule** | ✅ | ❌ (stay) | node isolation |
| **PreferNoSchedule** | soft (avoid) | ❌ | soft preference |
| **NoExecute** | ✅ | ✅ (deleted) | drain/evict on node problems |
- **NoExecute nuance:** toleration can have **`tolerationSeconds`** (grace period). Node goes unreachable → node controller adds `node.kubernetes.io/unreachable:NoExecute` → pods evicted after tolerationSeconds (default 300s).
- Evicted pod recreated ONLY if owned by a controller (Deploy/RS/STS); bare pod just deleted.

### This cluster's isolation (verified)
```
infra/kata/el/master nodes → NoSchedule taint node-role.kubernetes.io/<role>
worker nodes → NO taint → general workloads (web pods landed here)
```
- OpenShift pairs the taint (repel) with a **nodeSelector** (`node-role.kubernetes.io/infra: ""`) to pull infra components ONTO infra nodes. Need BOTH.

### Affinity types (quick)
- **nodeAffinity**: pod → node labels (required=hard filter, preferred=soft score). Modern nodeSelector.
- **podAffinity**: co-locate with pods matching a label ("run near the cache").
- **podAntiAffinity**: spread away from pods matching a label ("don't put 2 replicas on same node/zone").
- **topologyKey**: the domain (node/zone) the affinity applies over.
- **PodTopologySpread**: even spread across zones/nodes (`maxSkew`).

### Commands
```bash
kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints' | grep -v '<none>'
kubectl -n learn-k8s describe pod -l app=web | grep -A12 Events   # scheduler decision (events TTL ~1h)
# force fail:
kubectl -n learn-k8s run stuck --image=... --overrides='{"spec":{"nodeSelector":{"disktype":"unobtanium"}}}'
kubectl -n learn-k8s describe pod stuck | grep -A10 Events   # "0/16 nodes available: didn't match node selector"
```

### Interview soundbite (scheduling)
> "The scheduler filters nodes that can't run the pod (resources, taints, affinity, arch), scores
> the rest, and binds by writing .spec.nodeName — the kubelet then runs it. NoSchedule blocks new
> pods without a toleration but leaves running pods alone; NoExecute also evicts running pods, with
> an optional tolerationSeconds grace period (how pods leave unreachable nodes). Taints only repel;
> pair a toleration with a nodeSelector/affinity to actually pin a workload to specific nodes."

---

## Lesson — Pod Lifecycle

### Probes (kubelet runs them per container)
| Probe | Question | On failure |
|-------|----------|-----------|
| **liveness** | alive or hung? | **restart** the container |
| **readiness** | can it serve traffic? | **remove from EndpointSlice** (no restart, keeps running) |
| **startup** | finished booting? | holds off liveness/readiness until it passes (slow-start apps) |
- ⭐ readiness fail = leaves Service endpoints (gates traffic, ties to networking lesson); liveness fail = killed+restarted. Too-aggressive liveness on slow app → **restart loops** (classic mistake).

### QoS classes (eviction order under memory pressure)
| QoS | Condition | Evicted |
|-----|-----------|---------|
| **Guaranteed** | requests == limits (all containers, cpu+mem) | **last** (protected) |
| **Burstable** | has requests < limits | middle |
| **BestEffort** | no requests/limits | **first** |
- Order: BestEffort → Burstable → Guaranteed. Set requests==limits for critical pods.
- **OOMKill vs eviction:** container exceeds its mem LIMIT → kernel **OOM-kills just that container** (`OOMKilled`). NODE low overall → kubelet **evicts whole pods** by QoS.
- (web pods = BestEffort — no requests/limits set.)

### PodDisruptionBudget (PDB)
- "Keep ≥ N (or X%) pods available." Blocks **VOLUNTARY** disruptions (drain, upgrade) that would drop below budget.
- Voluntary (drain/upgrade) → PDB **respected** (eviction API refuses until replacement Ready).
- Involuntary (node crash, OOM) → PDB **can't help** (hardware doesn't ask permission).
- The `etcd-guard`/`apiserver-guard` pods enforce PDB-like control-plane safety.

### Graceful shutdown (pod termination sequence)
```
1. Pod → Terminating; removed from EndpointSlices (stops new traffic)
2. preStop hook runs (if defined)
3. SIGTERM to container PID 1  ← app starts clean shutdown
4. terminationGracePeriodSeconds countdown (default 30s)
5. still alive after grace → SIGKILL (forced)
```
- ⚠️ Steps 1 & 3 are concurrent → apps must **handle SIGTERM** (drain in-flight); often add **preStop sleep** so endpoint removal propagates before exit (avoids dropped requests during rolling updates).

### Commands
```bash
kubectl -n learn-k8s get pod -l app=web -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.status.qosClass}{"\n"}{end}'
kubectl get pdb -A | head
kubectl -n openshift-monitoring get pod prometheus-k8s-0 -o yaml | grep -A6 -iE 'livenessProbe|readinessProbe|terminationGracePeriod'
```

### Interview soundbite (pod lifecycle)
> "Liveness failure restarts the container; readiness failure just pulls it from Service endpoints
> without restarting. QoS (Guaranteed/Burstable/BestEffort) sets eviction order under node memory
> pressure — BestEffort dies first. A PDB protects availability during voluntary disruptions like
> drains but can't help an involuntary node crash. On termination the pod leaves endpoints, runs
> preStop, gets SIGTERM, then SIGKILL after the grace period — so apps should handle SIGTERM."

---

## Lesson — kubelet + CRI / CNI / CSI (node execution path)

### kubelet = the node agent
- Runs on every node; watches API for pods where `.spec.nodeName == my node`; reconciles them into running containers.
- **Delegates** everything to 3 pluggable interfaces (doesn't run containers or do networking itself).

### The 3 CxI interfaces (top node-level interview Q)
| Interface | Governs | This cluster | Alternatives |
|-----------|---------|--------------|--------------|
| **CRI** (Container Runtime Interface) | start/stop containers | **CRI-O** | containerd, (dockershim removed) |
| **CNI** (Container Network Interface) | pod networking (IP, routes) | **OVN-Kubernetes** | Calico, Cilium, Flannel |
| **CSI** (Container Storage Interface) | mount volumes | **GCP PD CSI** | EBS, Ceph, NFS |
- Why: K8s defines a standard API, vendors plug in → runs everywhere.

### Full pod → running sequence (connects all lessons)
```
1. Scheduler sets .spec.nodeName → API → etcd
2. kubelet (watching) sees pod assigned to it
3. CRI (CRI-O): create POD SANDBOX = "pause" container (holds netns + cgroup)
4. CNI (OVN): assign pod IP (172.24.x), veth, routes → writes k8s.ovn.org/pod-networks annotation
5. CSI: attach + mount PVs (if any)
6. CRI: pull images → init containers → app containers (JOIN sandbox netns → share pod IP)
7. kubelet runs probes → Ready → EndpointSlice controller adds to Service
8. Pod Running & serving
```

### ⭐ The "pause" (sandbox) container
- Hidden per-pod container (not in `kubectl`). Holds the pod's Linux **namespaces** (network/IPC).
- All real containers **join pause's netns** → that's WHY they share `localhost` + one pod IP.
- App container crashes/restarts → pause STAYS → **pod IP unchanged**.

### OpenShift specifics
- **CRI-O** (saw `cri-o://1.35.5` in get nodes). Uses **runc**; **kata** nodes use `kata-runtime` (VM isolation) via a **RuntimeClass**.
- **crictl** = CRI debug CLI (like `docker ps` at CRI level), run on node via `oc debug node/<n>` → `chroot /host crictl ps`.

### Commands
```bash
kubectl get nodes -o wide | awk '{print $1, $NF}'   # runtime per node
kubectl get csidrivers ; kubectl get csinodes | head
kubectl get runtimeclass                            # kata etc.
# oc debug node/<n> → chroot /host crictl ps / crictl pods (see pause sandboxes)
```

### Interview soundbite (kubelet/CRI/CNI/CSI)
> "The kubelet watches the API for pods on its node and reconciles them into containers by
> delegating to three pluggable interfaces: CRI (runtime, CRI-O) to run containers, CNI (OVN) to
> set up networking and assign the pod IP, and CSI to mount volumes. Each pod gets a pause sandbox
> container that holds the network namespace, so all containers share one pod IP that survives
> container restarts."
