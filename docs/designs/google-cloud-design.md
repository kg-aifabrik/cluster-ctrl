# Google Cloud design — the hardened GKE cluster

*Status: draft for review · Date: 2026-06-06*
*Companion to [requirements.md](../requirements.md) and [technology-choices.md](../technology-choices.md). The security controls and evidence live in [security-requirements.md](../security-requirements.md).*

---

## 1. Scope

How one functioning, hardened **Google Kubernetes Engine (GKE)** cluster — Google's
managed Kubernetes — is assembled from Google Cloud resources: the pieces to create,
how they depend on each other, and why each choice. One cluster per environment, each
in its own project. These pieces are what the build recipe (Terraform) creates.

## 2. The shape in one breath

A custom **Virtual Private Cloud (VPC)** network holds a regional, private cluster with
**no public endpoint**. Operators and automation reach it only through **Connect
Gateway**. Nodes have no public address and reach Google services over a private path.
Secrets and disks are encrypted with our own key. Pods get cloud access through their
own identity, never the node's. Only signed images from our registry are admitted.

---

## 3. Foundation (per environment)

### Project
- One Google Cloud project per environment (dev / stage / prod) — the boundary for
  isolation, quotas, identity, and cost.
- Created and owned by the keyless automation identity built in Milestone 0.

### Enabled services (and why each)
APIs are off until enabled; we turn on only what the cluster uses. Enabling GKE and
Compute is also what brings the Google-managed service agents (below) into existence.
- **container** — the GKE service itself (clusters and node pools).
- **compute** — the network, subnets, nodes (virtual machines), and disks underneath.
- **iam** / **iamcredentials** — service accounts and short-lived credentials.
- **cloudkms** — the encryption key for secrets and disks.
- **artifactregistry** + **containeranalysis** — store images and scan them.
- **binaryauthorization** — admit only trusted images.
- **gkehub** + **gkeconnect** + **connectgateway** — fleet membership and private API access.
- **logging** + **monitoring** — where logs and metrics go (incl. managed Prometheus).
- **secretmanager** — workload secrets (technology-choices TC-13).
- **certificatemanager** / **privateca** — TLS certificates (TC-7).
- **dns** — private DNS zones so private nodes resolve Google service names.

### Customer-managed encryption key (CMEK)
- **What it is** — our own key in **Cloud Key Management Service (KMS)**, used to encrypt
  the cluster's Kubernetes secrets (in etcd) and the nodes' disks. "Customer-managed"
  means we, not Google alone, control and audit it.
- **Requirements** — a symmetric *encrypt/decrypt* key, in a key ring whose **location
  matches the cluster's region** (there is no global option for this use); automatic
  rotation enabled.
- **Created once, then kept** — the key resource is created once and lives for as long as
  the cluster does. Rotation periodically adds a *new key version* (old data stays
  readable), so the key changes over time but is never recreated. **Never destroy a key
  version still in use** — that makes the cluster's secrets unreadable.
- **How and where** — in Cloud KMS: create a key ring, then the key. Then grant *use* of
  the key (the "encrypter/decrypter" role) to two Google-managed identities (next), and
  point the cluster at the key in two places: its secret-encryption setting and its node
  disk-encryption setting.

### Service agents and the two key grants
- Google-managed identities that run parts of the cluster on our behalf. They don't exist
  until the APIs are enabled — and must be created before we grant them key access, or the
  build races.
- **Two separate grants** (a common trap — both are required):
  - the **GKE service agent** gets use of the key for **secret (etcd) encryption**;
  - the **Compute service agent** gets use of the key for **node and disk encryption**.

### Node service account
- A dedicated, least-privilege identity attached to the nodes, replacing Google's broad
  default. It can do only what nodes need: write logs and metrics, and pull images.
- Pods do **not** use it — they get their own identity (below), so this stays minimal.

### Workload Identity
- Maps a Kubernetes service account to a Google identity, so each **pod** gets its own
  scoped cloud access instead of borrowing the node's credentials.
- Set on the cluster (identity pool `<project>.svc.id.goog`) and on each node pool
  (metadata mode), so the node's credentials aren't reachable from pods.

---

## 4. Network

### Custom VPC (not auto mode)
- **Auto mode** makes Google create one subnet per region automatically, with fixed preset
  ranges. We use **custom mode** instead so we choose every range deliberately — avoiding
  overlaps with other networks and surprise address space.

### Subnet and the Pod/Service ranges
- **A subnet is regional** — it spans every zone in the region. One subnet serves the nodes
  in all three availability zones; subnets are *not* per-zone.
- The cluster is **VPC-native** ("alias IP"): the node subnet has its **primary range** (node
  addresses) plus two **secondary ranges** on the same subnet — one for **Pods**, one for
  **Services**. Pods and Services are *secondary ranges*, not separate subnets.
- **Sizing** — GKE gives each node a `/24` (room for ~110 pods), so the Pod range must cover
  `max nodes × /24` and dominates consumption; the Services range defaults to a `/20`. Pull
  the Pod range from the `100.64.0.0/10` block (reserved for exactly this) to avoid burning
  ordinary private address space. All ranges must be non-overlapping. Plan generously now.
- **Growing later** — the **primary (node) range can be expanded** in place; **extra Pod
  ranges can be added** to node pools as the cluster grows. The **Services range is fixed at
  creation** and cannot change — which is why it's sized up front.
- **Proxy-only subnet** — a separate subnet used only by *internal* load balancers / internal
  Gateways (the Envoy proxies that front internal HTTPS endpoints). The cluster itself does not
  need one; we add it if and when we expose internal endpoints.

### Reaching Google services privately
- **Private Google Access** on the subnet lets nodes with no public address reach Google
  APIs and Artifact Registry over Google's internal network, with private DNS pointing those
  names at Google's restricted virtual IP.

### Outbound internet for pods
- Private nodes have no public address, so a pod that needs the public internet egresses
  through **Cloud NAT** (a managed network-address-translation service on a Cloud Router),
  which gives shared outbound IPs. Without Cloud NAT, pods can reach only Google services.
- It's **off by default** (least exposure) and added only when a workload needs it. Egress is
  also gated by the namespace's default-deny network policy, so a pod needs *both* the Cloud
  NAT path *and* an explicit egress allow.

### Dataplane V2 (Cilium)
- The cluster's data plane is **Dataplane V2**, Google's managed networking built on
  **Cilium / eBPF** (extended Berkeley Packet Filter — programmable in the Linux kernel). It
  enforces NetworkPolicy in the kernel and powers network flow logging.
- We pair it with a **default-deny** network policy in every namespace.

---

## 5. Cluster and nodes

### Regional cluster
- `location` is a **region**, which gives a control plane replicated across three zones
  automatically; `node_locations` pins nodes across three zones too — the cluster survives
  losing one zone.

### Default node pool removed
- Creating a cluster auto-creates one "default" node pool using Google's default settings
  (including the broad default identity). We delete it (`remove_default_node_pool`) and
  create our own pools instead.
- **Why manage pools explicitly** — every pool then uses *our* hardened settings (our node
  identity, shielded nodes, Container-Optimized OS, machine type, labels/taints), and pools
  can be added, resized, or removed independently as day-2 operations.

### Versions and upgrades (per environment)
- **Dev** — enrolled in a **release channel** (Regular, the balanced one). Google auto-upgrades
  the control plane and nodes to tested versions during the maintenance window and auto-repairs
  nodes. Low stakes, and it surfaces version issues early.
- **Staging and production** — upgrades are **deliberate, scheduled Day-2 operations, never
  automatic**: surprise upgrades risk developer productivity (staging) and customer impact
  (production). Automatic upgrades are held off (maintenance exclusions) and each upgrade is
  triggered on a planned schedule through the normal reviewed-change path.
- **Validate first** — before upgrading staging or production, we build a throwaway **test
  cluster** on the new version with the same automation, confirm our workloads run correctly,
  then upgrade staging, then production.
- The build recipe is identical across environments; the upgrade cadence is an environment
  parameter, not a different cluster.

### Hardening, built in
- Verified-boot ("shielded") nodes, **Container-Optimized OS (COS)** nodes (minimal,
  auto-patched), secret encryption with our key, Workload Identity, and Dataplane V2
  default-deny — together clearing the **Center for Internet Security (CIS) Level 2** floor.

### Where logs and metrics go
- **Logs** — system and workload logs flow to **Cloud Logging**.
- **Metrics** — to **Cloud Monitoring**; we additionally enable **Managed Service for
  Prometheus**, so Prometheus-format workload metrics are collected and queryable without us
  running a Prometheus server.

### Maintenance window (dev)
- A recurring window we define on the **dev** cluster for GKE's automatic upgrades, so they land
  at a predictable, low-traffic time. Staging and production have no automatic upgrades, so they
  use no window — their timing is set by the scheduled Day-2 upgrade operation.

### Ingress (Gateway API)
- We enable the **GKE Gateway** controller (our chosen ingress, TC-6) on the cluster so routing
  of traffic into services can be defined later. Enabling it now costs nothing and avoids a
  reconfigure.

### Deletion protection
- On for every cluster, guarding against accidental deletion.

### Node pools
- A **general** hardened pool (machine type, disk, on-demand), plus an optional
  **Confidential** (memory-encrypting) pool per cluster — always **on-demand**, never spot
  (spot Confidential nodes proved unreliable). **Autoscaling is deferred.**

---

## 6. Access — no public endpoint

### Private nodes + private control plane
- Nodes have no public address. The control plane uses the **DNS-based private endpoint**
  (Google's current recommended approach) with external access turned off — there is **no
  public API endpoint** to attack or allow-list.

### Fleet membership
- The cluster joins a **fleet** (a group of clusters). Today this is only the prerequisite
  for Connect Gateway.
- **Later fleet features** it unlocks (not used yet): multi-cluster Services and Gateways,
  fleet-wide configuration and policy management, team scopes for multi-tenancy, Cloud Service
  Mesh, and fleet-wide observability.

### Connect Gateway
- The single way operators and automation reach the cluster's Kubernetes interface — by
  Google **Identity and Access Management (IAM)** plus in-cluster role-based access control,
  with **no** network allow-lists or virtual private network (VPN).

---

## 7. Supply chain

### Artifact Registry
- One private image repository per project for our images, plus a proxy repository that
  mirrors public base images — so the cluster never pulls from the internet directly. The
  node identity has read-only access.

### Binary Authorization
- An admission policy that admits only trusted images. It starts in **audit** (logs, does not
  block) and switches to **enforce** once the image-signing pipeline exists.

*(Image scanning and the daily evidence reports are in [security-requirements.md](../security-requirements.md).)*

---

## 8. Build order

1. Project → enable services → force-create the service agents.
2. KMS key ring + key → the two key grants.
3. Node service account and its roles.
4. VPC → subnet + secondary ranges → Private Google Access / DNS (Cloud NAT if needed).
5. Artifact Registry (+ reader grant) and the Binary Authorization policy.
6. Cluster (regional, private, hardened, Workload Identity, our key, Dataplane V2) → fleet membership.
7. Node pools (general; Confidential if requested).
8. Connect Gateway access (IAM + in-cluster roles).

## 9. Decisions, in brief

- **One project per environment** — clean isolation, cost, and blast-radius boundary.
- **No public control-plane endpoint; DNS-based private endpoint + Connect Gateway** — nothing reaches the API over the internet.
- **Our own key for secrets and disks** — control and auditability; two grants because Google uses two managed identities.
- **Least-privilege node identity + Workload Identity** — nodes can do little; pods get scoped identities, not node keys.
- **Custom VPC, generous immutable Pod/Service ranges** — deliberate, non-overlapping addressing planned once.
- **Regional everywhere; upgrades by environment** — dev auto-upgrades on a release channel; staging and production upgrade deliberately via scheduled Day-2 operations, validated on a test cluster first.
- **Dataplane V2 default-deny, CIS Level 2** — secure by default.
- **Images only from our registry; Binary Authorization audit → enforce** — no untrusted images.

## 10. Open items

- **Control-plane endpoint** — confirm the DNS-based endpoint on our GKE version at build.
- **Binary Authorization** — the newer check-based policy is still Preview; start with the generally-available project policy.
- **Staging/production upgrade mechanism** — keep them on a channel with automatic upgrades suppressed (maintenance exclusions), or pin a specific version; settle at build.
- **Service-to-service mutual TLS / service mesh** — still open (requirements §8).
- **Autoscaling** — deferred.

## 11. Related

[requirements.md](../requirements.md) · [technology-choices.md](../technology-choices.md) · [security-requirements.md](../security-requirements.md).
