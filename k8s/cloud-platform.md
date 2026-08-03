# Cloud Platform Prep — GCP-first, AWS reference

> For the EKS Platform Engineer role. Learn the GCP concept (already known), then map to AWS.
> Companion to k8s-internals-notes.md + cloud-vpc-notes.md.

## Block A — IAM + Workload Identity

### GCP IAM model
- **Triangle:** WHO (principal/member) → granted WHAT (role = permission bundle) → ON WHERE (resource).
- **Principal:** user, group, **service account**, or WIF principal (`principalSet://...`).
- **Role** = named bundle of permissions. Basic (Owner/Editor/Viewer, avoid) | **Predefined** (use) | Custom.
- **IAM policy (binding)** = attaches `{member → role}` on a resource.
- **Hierarchy:** Organization → Folder → Project → Resource. Policies **inherit DOWNWARD + additive**. Grant at narrowest scope.
- ⭐ **Service account = identity AND resource → the two-policy split:**
  | Side | Question | GCP mechanism |
  |------|----------|---------------|
  | **Permission** | what can it DO? | roles granted **TO** the SA (e.g. storage.objectViewer) |
  | **Trust** | who can ACT AS it? | IAM policy **ON** the SA: `roles/iam.serviceAccountUser` (impersonate) or `roles/iam.workloadIdentityUser` (federate = WIF) |
- **Security ladder:** SA key (❌ long-lived) < impersonation (✅ short-lived) < **WIF** (✅✅ no key, external federation).

> GCP soundbite: "IAM is principal→role→resource, inheriting down org/folder/project. A service account is both identity and resource: roles granted TO it = permission; the IAM policy ON it (serviceAccountUser/workloadIdentityUser) = trust (who can impersonate/federate). Prefer federation over keys."

### 🔁 AWS reference map
| GCP | AWS | Note |
|-----|-----|------|
| principal/member | IAM principal | same |
| **role = permission bundle** | **policy** (JSON) | ⚠️ WORDS CROSS OVER |
| **service account** (assumable identity) | **IAM Role** (assumable identity) | ⚠️ AWS "Role" = identity, NOT permissions |
| IAM policy ON the SA (who impersonates) | **trust policy** (on the Role) | who can assume |
| roles granted TO the SA | **permission policy** (on the Role) | what it can do |
| WIF (workloadIdentityUser + pool + provider) | **IRSA** (IAM OIDC provider + AssumeRoleWithWebIdentity) | same OIDC triple |
| impersonation | `sts:AssumeRole` | temp creds |
| Org→Folder→Project | Org→OU→Account (+ **SCPs**) | AWS isolates at account level |
| predefined role | AWS managed policy | curated |
| custom role | customer-managed policy | your own |
- ⚠️ **THE trap:** GCP "Role" = permission bundle (≈ AWS policy). AWS "Role" = assumable identity (≈ GCP service account). Different meaning of the same word.

### IRSA (AWS pod→AWS auth) = WIF with AWS names
- Same issuer+subject+audience triple:
  | WIF | IRSA |
  |-----|------|
  | issuer = serviceAccountIssuer URL | EKS cluster **OIDC provider URL** (registered as IAM OIDC provider) |
  | subject = system:serviceaccount:ns:sa | same |
  | audience = pool | **sts.amazonaws.com** |
- **Flow:** register EKS OIDC issuer as IAM OIDC provider → create IAM Role {trust policy: OIDC provider + condition sub=system:serviceaccount:ns:sa, aud=sts.amazonaws.com; permission policy: s3:GetObject ...} → annotate SA `eks.amazonaws.com/role-arn: arn:...` → pod-identity webhook injects projected token (aud=sts.amazonaws.com) + env `AWS_ROLE_ARN`, `AWS_WEB_IDENTITY_TOKEN_FILE` → SDK calls **`AssumeRoleWithWebIdentity`** at sts.amazonaws.com → validates token vs cluster JWKS + trust policy → short-lived AWS creds.
- AWS env twins of GCP: `AWS_WEB_IDENTITY_TOKEN_FILE` + `AWS_ROLE_ARN` ≈ `GOOGLE_APPLICATION_CREDENTIALS` (external_account).
- **EKS Pod Identity** (2023) = simpler alt: EKS add-on agent + `PodIdentityAssociation` (SA↔role), trust = `pods.eks.amazonaws.com`; no per-cluster OIDC provider juggling. IRSA still most common.

> IRSA soundbite: "IRSA is EKS keyless pod→AWS auth — same OIDC federation as GCP WIF. The cluster OIDC issuer is an IAM OIDC provider; an IAM Role's trust policy allows it for a specific system:serviceaccount:ns:sa with aud sts.amazonaws.com, and its permission policy grants the AWS access. Annotate the SA with the role ARN; the SDK calls AssumeRoleWithWebIdentity to swap the projected token for temp creds. AWS Pod Identity is the newer simpler alternative."

---

## Block F — Node Autoscaling

### ⚠️ First: two DIFFERENT autoscalers (interview trap)
- **POD** autoscaling → HPA (replica count) / VPA (pod size).
- **NODE** autoscaling → Cluster Autoscaler / Karpenter (how many VMs exist).
- They **compose**: HPA adds replicas → pods go `Pending` (no room) → node autoscaler adds a node.

### GCP first — GKE Cluster Autoscaler (CA)
- **Node pool** = group of *identical* GCE VMs = a **Managed Instance Group (MIG)** + instance template. CA runs per pool with **min/max**.
- **Core loop:**
  | Direction | Trigger | Action |
  |-----------|---------|--------|
  | **Scale UP** | pod `Pending` — no node has room for its **requests** | simulate "add node from pool X, does it fit?" → bump MIG |
  | **Scale DOWN** | node <50% of requests for ~10min **and** pods can move | cordon → drain → delete VM |
- ⭐ **CA reasons on requests, NOT usage.** `requests: 0` → CA thinks nodes are empty → **never scales up** even while pods OOM. Requests = currency of scheduler + autoscaler.
- **Scale-DOWN blockers** ("why won't it scale down?"):
  - bare pod (no controller) · local storage (emptyDir/hostPath) · **PDB** would break · annotation `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"` · kube-system pod w/o PDB.
- **GKE extras:** **Node Auto-Provisioning (NAP)** = CA can create *new* node pools with a better machine type. **Autopilot** = nodes fully hidden/managed.
- **Observe:** `kubectl describe pod` → Events "insufficient cpu"; `kubectl get nodes -w` new node NotReady→Ready; CA logs "Scale-up: setting group size to N"; `kubectl get events | grep TriggeredScaleUp`.

> GCP soundbite: "Cluster Autoscaler grows a node pool's MIG when pods can't fit and reaps nodes under 50% utilization that can be safely drained. It reasons on resource requests, not live usage, and respects PDBs, taints, and affinity. Node pools are fixed-shape; NAP can mint new pools with a better machine type."

### 🔁 AWS reference — two choices
**A) Cluster Autoscaler on EKS** (direct port): node-pool = **Auto Scaling Group (ASG)** = AWS's MIG; CA changes ASG desired capacity. One managed node group ↔ one ASG ↔ one fixed shape. **AZ gotcha:** EBS is zonal → run **one ASG per AZ** so rescheduled pods land where their volume lives.

**B) ⭐ Karpenter** (AWS-native 2021 — the cost lever): **no node groups, no ASGs.** Talks **directly to EC2 Fleet API**, provisions a **right-sized node JIT** for the exact pending pods.
| | Cluster Autoscaler | **Karpenter** |
|---|---|---|
| unit | node group / ASG (**fixed shape**) | **any EC2 instance**, chosen per pods |
| how | bump ASG desired count | EC2 Fleet API directly |
| speed | slower (ASG indirection) | faster |
| instance choice | pre-decided per group | **dynamic bin-pack** (cheapest fit) |
| consolidation | weak | **active** (repacks onto fewer/cheaper) |
| spot | via ASG mixed policy | **native** |
| config | ASG + flags | **`NodePool`** + **`EC2NodeClass`** CRDs |
| portability | multi-cloud | **AWS-only** |
- **`NodePool`** CRD = constraints: instance families, `capacity-type: [spot,on-demand]`, arch (amd64/arm64/Graviton), limits, consolidation policy.
- **`EC2NodeClass`** CRD = AWS specifics: AMI family, subnets, SGs, IAM role.
- **Flow:** pod Pending → Karpenter reads requests+affinity+taints → picks cheapest instance that fits (merges many pods onto one node) → launches via Fleet in seconds → **consolidation** later repacks onto fewer/cheaper nodes.
- **Cloud map:** GKE node pool (MIG) ≈ EKS managed node group (ASG) — both fixed-shape, both use CA. **Karpenter has no GCP equivalent** — groupless, dynamic, JIT.

> Karpenter soundbite: "Karpenter replaces Cluster Autoscaler AND node groups. Instead of resizing fixed ASGs, it reads pending pods and provisions right-sized EC2 instances just-in-time via the Fleet API — faster, better bin-packing. Consolidation actively repacks workloads onto fewer, cheaper nodes and it uses Spot natively — the big cost lever on EKS. Config is NodePool + EC2NodeClass CRDs, not ASGs."

---

## Block G — Load Balancers (GKE | OpenShift | EKS)

### ⚠️ First: GKE ≠ OpenShift-on-GCP
| | **GKE** | **OpenShift-on-GCP (my cluster)** |
|---|---|---|
| Google gives | **managed K8s** (Google runs control plane, no masters visible) | just **IaaS** (raw GCE VMs, VPC, disks) |
| Own the cluster | Google runs masters/etcd/upgrades | **I own everything** — 3 master Machines, etcd, HAProxy router on GCE VMs |
| Install | `gcloud container clusters create` | **`openshift-install create cluster` (IPI)** |
| Node scaling | GKE Cluster Autoscaler | **MachineSet** replica bump → Machine API → GCE VM |
- **IPI install:** `openshift-install` + GCP SA cred → Terraform provisions VPC/subnets/firewall → **GCP L4 LB for API** (:6443) → bootstrap VM → **3 masters + workers via Machine API/MachineSets** → RHCOS → Cloud DNS records → bootstrap self-destructs. GCP = pure infra; OpenShift runs on top.

### 3 ways traffic enters a cluster
- `Service ClusterIP` = internal only · `NodePort` = port 30000–32767 on every node · `LoadBalancer` = real cloud LB (one Svc→one LB→one IP) · **Ingress/Route/Gateway** = ONE L7 entry, host/path routing to MANY Services.
- ⭐ **Who creates the cloud LB?** The **Cloud Controller Manager (CCM)** translates `type=LoadBalancer` → real cloud LB via cloud API.

### L4 vs L7 (GCP terms)
| | **L4 (network)** | **L7 (application)** |
|---|---|---|
| GCP name | Network LB / passthrough | HTTP(S) LB |
| layer | 4 (TCP/UDP) | 7 (HTTP) |
| sees | IP+port | URL/host/path/header/cookie |
| K8s trigger | `Service type=LoadBalancer` | Ingress (GKE) |
| TLS | passthrough | terminates at LB |
- **L4 decode chain:** `forwarding rule (IP:port) → backend service → instances/NEG`.
- **L7 decode chain:** `forwarding rule → target proxy (TLS) → URL map (host/path) → backend service → NEG` — URL map = the "smart" host/path routing. (⚠️ this chain is L7/Ingress ONLY, not the L4 service LB.)
- ⭐ **NEG / container-native LB** = pod IPs registered directly in backend → LB hits pods, no node hop.

### ⭐ THE big architectural difference
```
GKE  & EKS:  L7 routing lives IN THE CLOUD LB (HTTP(S) LB / ALB)
OpenShift:   L7 routing lives INSIDE THE CLUSTER (HAProxy "router" pods)
             → cloud only gives an L4 LB to reach the router pods
```

### 3-way comparison
| Concept | **GKE** | **OpenShift-on-GCP** | **EKS** |
|---|---|---|---|
| L7 ingress mechanism | GCP L7 HTTP(S) LB (cloud) | **Route + HAProxy router (in-cluster)** | ALB (cloud) |
| triggered by | `Ingress` | **`Route`** (Ingress→Route auto) | `Ingress` |
| where L7 routing happens | cloud LB | **HAProxy pods (openshift-ingress ns)** | cloud LB (ALB) |
| cloud LB job for ingress | L7 (routes) | just **L4** → reach router | L7 (routes) |
| L4 Service LB | GCP Network LB | GCP Network LB | **NLB** |
| pod-direct dataplane | NEG | router → Service → pods | **target group `ip` mode** |
| TLS terminated by | cloud L7 LB | **HAProxy router** | ALB (ACM cert) |
| LB controller | GKE ingress ctlr | OpenShift Ingress Operator | **AWS Load Balancer Controller** |

- **OpenShift flow:** `Client → DNS *.apps.<cluster> → GCP L4 LB → HAProxy router pods → Service → pods`. Route configures HAProxy; TLS modes = **edge / reencrypt / passthrough**. In-cluster router = why OpenShift is cloud-portable.
- **EKS target group modes:** `instance` = targets are nodes (NodePort, extra hop) · ⭐ `ip` = pod IPs directly (VPC-CNI routable IPs) = twin of GKE NEG. Preferred.
- **Cloud map:** GKE L7 HTTP(S) LB ≈ AWS **ALB** (`Ingress`) · GKE L4 Network LB ≈ AWS **NLB** (`type=LoadBalancer`) · NEG ≈ AWS **target group ip mode** · GKE ingress ctlr ≈ **AWS Load Balancer Controller** · OpenShift Route/HAProxy = *no direct cloud twin* (routing is in-cluster).

> 3-way soundbite: "GKE and EKS push L7 routing out to a cloud load balancer — HTTP(S) LB or ALB — programmed by an Ingress. OpenShift keeps L7 inside the cluster: HAProxy router pods (the Ingress Controller) do host/path routing and TLS, configured by Route objects, and the cloud only supplies an L4 LB to reach those router pods. That in-cluster router is why OpenShift is portable across clouds and on-prem."

### TLS termination modes (OpenShift Route) — where the encrypted tunnel ends
| mode | client→router | router→pod | cert lives on | router sees plaintext? |
|------|---------------|-----------|---------------|------------------------|
| **edge** | HTTPS | **plain HTTP** | the Route | ✅ |
| **reencrypt** | HTTPS | **new HTTPS** | Route + pod cert | ✅ (decrypt→re-encrypt) |
| **passthrough** | HTTPS | **same HTTPS (untouched)** | the **pod** only | ❌ (routes by SNI only, no path) |
- ⚠️ passthrough = end-to-end encryption but **lose L7 path routing** (router can't read encrypted HTTP, only SNI hostname).
- **What a cert IS:** X.509 cert (public: domain+public key+CA signature+validity) **+ private key** (secret). Passport analogy: cert=passport, key=secret proof. Route `tls:` fields: `certificate`, `key`, `caCertificate`, `destinationCACertificate` (reencrypt only). passthrough = no cert on Route (pod holds it).
- **TLS = protocol layer** between TCP(L4) and HTTP(L7): HTTP wrapped in TLS wrapped in TCP. "Terminate TLS" = unwrap the encryption envelope to plaintext HTTP.
- **Handshake direction (one-way TLS):** client ClientHello carries **SNI hostname (plaintext)** → router picks matching cert → **router presents ITS cert** → **client VERIFIES** (trusted CA? domain match? not expired?). Client brings **NO cert**, only the hostname. Router holds+presents; client only verifies. **mTLS** = exception where client also presents a cert.

> TLS soundbite: "TLS is an envelope around the HTTP request; the cert is the server's passport shown during the handshake, not carried in the request. The client sends the hostname as SNI, the router presents the matching Route cert, the client verifies it against a CA. edge terminates at the router (plain HTTP to pod), reencrypt re-encrypts to the pod, passthrough forwards raw TLS so the pod terminates it — losing path routing. Only in mTLS does the client also present a cert."

---

## Block D — DNS + ExternalDNS (GKE | OpenShift | EKS)

### Problem: how does myapp.apps... become an IP?
`hostname --(DNS)--> LB IP --> handshake/routing`. DNS = name→address phone book; step BEFORE the LB/TLS.

### DNS record types
| record | maps | note |
|--------|------|------|
| **A** | name → IPv4 | |
| **AAAA** | name → IPv6 | |
| **CNAME** | name → another name | alias |
| **NS** | delegate zone → nameservers | |
| **TXT** | arbitrary text | ExternalDNS ownership + ACME challenge |
- **hosted/managed zone** = container of all records for a domain; cloud runs authoritative NS.
- ⚠️ **AWS Alias record** = A record pointing at an AWS resource (ALB/NLB/CloudFront) — needed because ALB has **no static IP, only a DNS name**. Like CNAME but works at zone apex + free.

### GCP first — Cloud DNS (+ OpenShift wildcard)
- **Cloud DNS** = GCP managed DNS; create a **managed zone**, Google runs NS.
- OpenShift IPI install created: `*.apps.<cluster> A → router L4 LB IP` and `api.<cluster> A → API LB IP`.
- ⭐ **wildcard `*.apps`**: ANY hostname under `.apps` resolves to the **same router LB IP** → HAProxy dispatches by Host header via Routes → **no per-app DNS needed** on OpenShift.
- **Observe:** `oc get dns.config/cluster -o yaml`; `dig +short myapp.apps.<cluster>` → router LB IP.

### ExternalDNS — automate records from K8s objects (GKE/EKS)
- On GKE/EKS each Ingress gets its OWN hostname (no wildcard router) → need a DNS record per app. **ExternalDNS** = controller that watches Ingress/Service/Route + **creates matching DNS records** in cloud provider.
- Flow: Ingress `host: shop.example.com` + LB address from `status` → ExternalDNS → Route53/Cloud DNS record. Writes a **TXT ownership marker** (only manages records it created). It's a **reconciliation loop**: delete Ingress → record deleted.
- Auth = same federation: EKS **IRSA** (`route53:ChangeResourceRecordSets`), GKE **Workload Identity** (Cloud DNS admin).

### 🔁 3-way comparison
| | **GKE** | **OpenShift-on-GCP** | **EKS** |
|---|---|---|---|
| DNS service | Cloud DNS | Cloud DNS | **Route 53** |
| per-app DNS needed? | yes | **no — `*.apps` wildcard** | yes |
| how records made | ExternalDNS | installer wildcard (once) | ExternalDNS |
| record → LB | A / CNAME | A (wildcard → L4 LB) | **Alias** (A→ALB) |
| ExternalDNS auth | Workload Identity | n/a | **IRSA** |
| zone object | managed zone | managed zone | hosted zone |
- ⭐ OpenShift wildcard = **no ExternalDNS for normal apps**; GKE/EKS need it (distinct LB hostname per Ingress).

> 3-way soundbite: "DNS maps the hostname to the LB address. On OpenShift the installer makes a Cloud DNS wildcard `*.apps` pointing at the router LB, so every app resolves without per-app records — the Route does L7 dispatch. On GKE/EKS each Ingress gets its own LB hostname, so I run ExternalDNS: it watches Ingress objects and reconciles records into Cloud DNS or Route 53, tagging each with a TXT ownership record, authenticating to Route 53 via IRSA."

---

## Block E — cert-manager + Venafi Issuer (auto cert lifecycle)

### Core idea: cert-manager = reconciliation loop for certs
Automates request + store + **auto-renew** (default at 2/3 of lifetime). No more manual PEM paste / expiry panic.

### Four objects
| object | role |
|--------|------|
| **Issuer / ClusterIssuer** | *where* certs come from (Venafi/ACME/Vault/CA). Issuer=1 ns; ClusterIssuer=cluster-wide |
| **Certificate** | *what* you want — domain + target Secret name |
| **CertificateRequest** | internal CSR cert-manager generates |
| **Secret** (`kubernetes.io/tls`) | **result**: `tls.crt`+`tls.key`. Route/Ingress mounts this |

**Loop:** Certificate → cert-manager gen keypair+CSR → ask Issuer to sign → store cert+key in Secret → Route/Ingress references it → **auto-renew** before expiry.

### Venafi (our setup)
- **Venafi** = enterprise cert-lifecycle / machine-identity platform = **policy gateway in front of the corporate CA** (DigiCert/MS CA). Enterprises don't use public Let's Encrypt internally.
- Venafi enforces org PKI **policy**: approved CAs, allowed domains/SANs, key size/algo, validity, audit/inventory.
- **Two flavors:** **TPP** (Trust Protection Platform, on-prem — typical <NAME>) uses `url`+`zone`+token; **Venafi Cloud** (VaaS) uses `zone`+API key.
- Config (TPP): `spec.venafi.zone: "<NAME>\\Platform\\OpenShift"` (policy folder) + `tpp.url` + `credentialsRef` (Secret w/ access token). Certificate references this ClusterIssuer → cert-manager calls Venafi → policy applied → corp-CA-signed cert into Secret.

### ⭐ Venafi issuer needs NO ACME challenge
| | **ACME** (Let's Encrypt) | **Venafi** |
|---|---|---|
| prove domain ownership | **challenge** HTTP-01 / DNS-01 | **none** — pre-authorized via credentials + zone policy |
| DNS-01 | writes **TXT record** in Route53/Cloud DNS (same perm as ExternalDNS!) | n/a |
| trust model | public domain-control proof | enterprise policy + credentials |
| CA | Let's Encrypt (public) | corporate CA behind Venafi |
- ACME challenge types: **HTTP-01** (serve token at `/.well-known/acme-challenge/`) · **DNS-01** (TXT record, works for wildcards + private domains).
- Venafi skips challenges because authorization = credentials + policy zone, not domain-control proof.

### Ties to Block G (TLS)
`cert-manager+Venafi → Secret (tls.crt+key) → Route references Secret → HAProxy presents it in handshake (edge/reencrypt)`. Block E **produces** the cert Block G's router **presents**.

### 🔁 AWS reference
| | GCP/OpenShift | AWS |
|---|---|---|
| cert automation | **cert-manager** (+Venafi/ACME) | **ACM** |
| cert lives | K8s Secret | ACM (managed, key hidden) |
| presenter | HAProxy router / Ingress | **ALB** (references ACM cert ARN) |
| renewal | cert-manager auto | ACM auto |
| enterprise CA | Venafi → corp CA | ACM Private CA, or cert-manager+Venafi on EKS |
- On EKS: run cert-manager+Venafi (identical) to stay on corporate PKI, OR use ACM+ALB (cert ARN, AWS renews).

> Venafi soundbite: "We auto-issue certs with cert-manager's Venafi issuer. A Certificate triggers cert-manager to generate a keypair+CSR, call Venafi TPP under a policy zone enforcing our approved CA/domains/key rules, and store the signed cert in a TLS Secret the Route presents — auto-renewed before expiry. Unlike ACME/Let's Encrypt it needs no HTTP-01/DNS-01 challenge because authorization comes from credentials and the zone, not public domain-control proof. On AWS the equivalent is ACM referenced by the ALB, though we'd keep cert-manager+Venafi on EKS for corporate PKI."

### How it's installed on OUR OpenShift cluster (OLM)
- CRDs don't come from `kubectl apply` — an **Operator** ships them, and **OLM** installs the operator.
- **Chain:** `Subscription → OLM → InstallPlan → CSV (ClusterServiceVersion)` → CSV registers CRDs + starts controller pods. CRD ships *inside* the CSV.
- **cert-manager** = "cert-manager Operator for Red Hat OpenShift" (ns `cert-manager-operator`, controllers in `cert-manager`) → CRDs `certificates/issuers/clusterissuers.cert-manager.io`.
- **Venafi Enhanced Issuer** = SEPARATE install (own Subscription/Helm) → CRDs `venaficlusterissuers/venafiissuers.jetstack.io`. That's why `issuerRef.group: jetstack.io` ≠ native cert-manager Venafi.
- **Dependency ladder:** OLM→cert-manager operator (Certificate CRD) · OLM/Helm→Venafi issuer (VenafiClusterIssuer CRD + `standard-venafi-issuer` instance→Venafi TPP) · app repo's Certificate CR references it→Secret. Likely **Argo CD applies the Subscriptions** (GitOps).
- **Verify:** `oc get csv -A | grep -iE 'cert-manager|venafi'` · `oc get subscription -A` · `oc get crd | grep -iE 'cert-manager.io|jetstack.io'` · `oc get pods -n cert-manager`.

### Annotated real <NAME> Certificate CR (this is a CR, not the CRD)
```yaml
kind: Certificate                    # instance of certificates.cert-manager.io CRD
spec:
  secretName: tls-cert-<app>         # OUTPUT Secret (tls.crt+tls.key) the Route presents
  secretTemplate:
    annotations: {kyverno.<NAME>.com/reload: "allow"}  # Kyverno restarts pods on rotation (Reloader-style)
  duration: 8760h                    # 365d validity (Venafi zone must allow)
  renewBefore: 2160h                 # renew 90d early → auto-renew at day 275
  privateKey: {rotationPolicy: Always}   # NEW keypair each renewal (stronger than Never/reuse)
  commonName / dnsNames: patchme     # placeholders → real hostname patched by cldctl at deploy
  issuerRef: {name: standard-venafi-issuer, kind: VenafiClusterIssuer, group: jetstack.io}  # Venafi Enhanced Issuer
```
- `{{.cleanedAppName}}` etc = **Go-template rendered by <NAME> `cldctl`** (label `renderedBy: cldctlVersion`); file in Git is a template, applied object is rendered.
- `patchme` = placeholder swapped for real `*.apps` host per app.

---

## Block H — GitOps with Argo CD (JD's #1 topic)

### Core idea
⭐ **GitOps = the K8s reconciliation loop pointed at a Git repo instead of etcd.** Git = desired state; an in-cluster agent continuously reconciles cluster→Git.
- **4 principles:** declarative (YAML) · versioned in Git (audit + `git revert` rollback) · **pulled** by in-cluster agent (not CI push) · **continuously reconciled** (auto-corrects drift).
- Argo CD = **CNCF Graduated** (Argo family, donated by Intuit, graduated Dec 2022). Peer **Flux** also graduated (lighter/CLI). Argo = UI + `Application` CRD.

### Architecture (runs in-cluster)
| component | job |
|-----------|-----|
| **application-controller** | THE reconciler: diff desired(Git) vs live(cluster) → sync |
| **repo-server** | clones Git + renders Helm/Kustomize/plain YAML |
| Redis | cache · API/UI/dex | web UI + SSO |
- ⭐ **`Application` CR** = unit of GitOps: "sync this Git repo/path → this cluster/namespace."

### Sync & Drift (interview drill)
- **Sync status:** Synced (matches Git) / OutOfSync (diff exists).
- **Sync policy:** Manual (human clicks Sync) or **Automated** + toggles:
  - ⭐ **selfHeal: true** = revert manual `kubectl edit` back to Git (cluster can't diverge).
  - **prune: true** = delete from cluster when removed from Git.
- **Health:** Progressing / Healthy / Degraded (is the Deployment actually rolled out).

### Explains OUR cluster
`Git (Subscriptions, VenafiClusterIssuer, Kyverno policies…) → Argo CD Application syncs → OLM installs operators → CRDs appear`. **Argo sits ABOVE OLM.** cldctl renders templates → Git → Argo syncs.

### Scale patterns
- **App-of-Apps** = one root Application → folder of child Applications → bootstrap whole cluster.
- **ApplicationSet** = generator templating many Applications (per-cluster/per-team) → multi-tenant/multi-cluster (JD word).
- **Sync waves/hooks** (`argocd.argoproj.io/sync-wave`) = ordering (CRDs before CRs).

### 🔁 Mapping
- Cloud-agnostic: identical on GKE/OpenShift/EKS. OpenShift ships **"OpenShift GitOps"** = packaged Argo CD via OLM. On EKS = standard (JD stack).

> GitOps soundbite: "GitOps is the K8s reconciliation loop pointed at Git: the repo is the source of truth and an in-cluster agent reconciles the cluster to match it. Argo CD's Application CR maps a Git path to a cluster; the application-controller diffs desired-vs-live and syncs, selfHeal reverts manual drift, prune deletes removed resources. At scale, App-of-Apps and ApplicationSets handle multi-tenant fleets. It's how our OpenShift cluster is managed — Argo applies Subscriptions, OLM installs operators — and identical on EKS since Argo is cloud-agnostic."

### Operators on OpenShift (verify: `oc get csv -A | grep -i gitops`)
| | operator | operator CRD | controller CRDs (argoproj.io) |
|---|---|---|---|
| **Argo CD** | Red Hat OpenShift GitOps (`openshift-gitops-operator`, OLM) | `ArgoCD` (ns `openshift-gitops`) | `Application`, `ApplicationSet`, `AppProject` |
| **Argo Rollouts** | *same* GitOps operator (GitOps ≥1.9) | **`RolloutManager`** | `Rollout`, `AnalysisTemplate`, `AnalysisRun`, `Experiment` |
- Verify: `oc get argocd -A` · `oc get rolloutmanager -A` · `oc get crd | grep argoproj.io`. Upstream alt = argoproj-labs Argo Rollouts Manager / Helm.

---

## Block I — Argo Rollouts (progressive delivery)

### Gap it fills
Built-in Deployment = **RollingUpdate only**, no traffic control, no metric gate (v2 returns 500s → rolls on anyway). Argo Rollouts = **canary/blue-green + traffic shaping + auto-analysis + auto-rollback**.

### `Rollout` CRD = drop-in for Deployment
Same `replicas/selector/template` + a `strategy` block (canary or blueGreen). Point workload at a `Rollout` instead of Deployment.

### Two strategies
| | **Canary** | **Blue-Green** |
|---|---|---|
| traffic | gradual (10→50→100%, `setWeight`) | all-at-once selector flip |
| cost | ~1× (+ few canary pods) | **2×** (both full stacks) |
| blast radius | small (canary %) | full (but tested first) |
| rollback | shift weight back | flip selector back |
- Canary steps: `setWeight` + `pause {duration}` or `pause {}` (indefinite until human promote).

### ⭐ AnalysisTemplate = automated gates (the "progressive" part)
- At each step an **AnalysisRun** queries metrics → pass/fail auto-decision.
- Provider = **Prometheus** (ties to observability stack), Datadog, CloudWatch, Job.
- `successCondition: result >= 0.99` + `failureLimit: 3` → bad metric = **auto-abort + auto-rollback**. = "automated canary analysis."

### Traffic management (who enforces the %)
Ingress/**ALB (AWS LB Ctlr)**, Istio, SMI, Gateway API, or Service weighting. EKS: ALB weighted target groups or Istio (ties to Block G).

### Pairs with Argo CD
`Argo CD = WHAT runs (Git→cluster)` + `Argo Rollouts = HOW it releases safely`. Argo CD syncs the Rollout object; Rollouts executes canary w/ metric gates.

> Rollouts soundbite: "Argo Rollouts replaces the Deployment with a Rollout CRD adding canary and blue-green. Canary shifts a small traffic slice to v2 and at each step an AnalysisRun queries Prometheus for success rate/latency — fail the threshold and it auto-aborts and rolls back, so bad releases never reach full traffic. Blue-green runs two full stacks and flips the Service selector for instant switch/rollback. It pairs with Argo CD: Argo CD gets manifests onto the cluster from Git, Rollouts governs how safely v2 takes traffic via the ALB or Istio."

---

## Block J — GitHub Actions CI/CD (closes the loop)

### The full pipeline
`CI (GitHub Actions): build→test→scan→push image→bump tag in Git` → `CD (Argo CD): sees Git change→sync` → `Rollouts: canary+Prometheus analysis→live/rollback`.
- ⭐ **CI ends at Git; CD starts from Git. Git = handoff = "push CI, pull CD."**
- ⭐ **GitHub Actions NEVER touches the cluster** (no kubeconfig/creds). Only commits to Git; Argo (in-cluster) applies. = GitOps security.

### Anatomy
| concept | meaning |
|---------|---------|
| **Workflow** | YAML in `.github/workflows/` |
| **Event/trigger** | `on: push / pull_request / schedule / workflow_dispatch` |
| **Job** | steps on one runner; parallel or `needs:` chained |
| **Runner** | machine — GitHub-hosted (`ubuntu-latest`) or self-hosted |
| **Step** | `run:` (shell) or `uses:` (action) |
| **Action** | reusable pkg (`actions/checkout`, `docker/build-push-action`) |

### ⭐ OIDC to AWS (ties to Block A — no static keys)
Same OIDC federation as IRSA/WIF:
| | IRSA (pod→AWS) | GitHub Actions→AWS |
|---|---|---|
| issuer | EKS cluster OIDC | `token.actions.githubusercontent.com` |
| subject | `system:serviceaccount:ns:sa` | `repo:org/repo:ref:refs/heads/main` |
| STS call | `AssumeRoleWithWebIdentity` | **same** |
| replaces | SA keys | **stored AWS access keys** |
- Job needs `permissions: {id-token: write}` → mints OIDC JWT → IAM Role trust policy allows GitHub OIDC provider + condition on `sub` (repo/branch) → temp creds → push to ECR.
- Step: `uses: aws-actions/configure-aws-credentials@v4` with `role-to-assume` (NO key/secret).

### GitOps-correct pattern
⭐ **image build repo ≠ deploy config repo.** CI commits new image tag to the **config repo**; Argo watches config repo (app churn ≠ deploy spam; config repo = audited desired state).

### 🔁 Alternatives
GitHub Actions (JD) · GitLab CI · Jenkins · **Tekton = OpenShift Pipelines** (<NAME> may use). Registry: ECR/Quay/Artifact Registry. Cloud auth = OIDC (no keys).

> CI/CD soundbite: "CI and CD split at Git: GitHub Actions builds, tests, scans, pushes the image, then commits the new tag to the config repo — it never touches the cluster. Argo CD pulls from that repo and Rollouts does the progressive release. GitHub Actions authenticates to AWS via OIDC not stored keys — same federation as IRSA — an IAM Role trusts the GitHub OIDC provider scoped to repo and branch. That push-CI/pull-CD boundary makes GitOps auditable and keeps cluster creds out of CI."
