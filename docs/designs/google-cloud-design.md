# Google Cloud design — the hardened GKE cluster

*Status: draft for review · Date: 2026-06-05*
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

## 3. Entities, by layer

Built bottom-up — each layer depends on the ones above it.

### 3.1 Foundation (per environment)

| Entity | What it is, and our choice |
|---|---|
| **Project** | One per environment — the isolation and cost boundary. Created and owned by the keyless automation identity from Milestone 0. |
| **Enabled services** | Turn on what the cluster touches: GKE, Compute, **Identity and Access Management (IAM)**, **Key Management Service (KMS)**, Artifact Registry + scanning, Binary Authorization, Fleet + Connect Gateway, Logging, Monitoring. Enabling GKE/Compute is also what creates the Google-managed service agents. |
| **KMS keyring + key** | One encryption key (a **customer-managed key**), in the cluster's region (no global option), with rotation on. Used for the encryption below. |
| **Service agents + two key grants** | Google-managed identities. Grant the **GKE agent** use of the key for **secret (etcd) encryption**, and the **Compute agent** use of it for **node and disk encryption** — two grants to two identities; missing either breaks the cluster. |
| **Node service account** | A dedicated, least-privilege identity for nodes (replacing the broad default): just enough to log, send metrics, and pull images. Pods do **not** use it. |
| **Workload Identity** | The cluster's identity pool (`<project>.svc.id.goog`); node pools run in metadata mode so each pod gets its own scoped cloud identity. |

*Watch out:* service agents don't exist until forced into being — create them before
granting key access, or the build races. Rotating the key does not re-encrypt existing
secrets, and destroying an in-use key version makes the cluster unusable.

### 3.2 Network

| Entity | What it is, and our choice |
|---|---|
| **VPC network** | A custom network (no auto subnets) — we place the cluster deliberately. |
| **Subnet + two secondary ranges** | One regional subnet with named ranges for **Pods** and **Services** (the cluster is alias-IP / "VPC-native"). The ranges **cannot be changed after creation**, so the IP space is planned once, up front. |
| **Private Google Access + private DNS** | Lets nodes with no public address reach Google services (registry, APIs) over Google's internal path. |
| **Cloud NAT** | Added only if workloads need outbound internet; off by default (least exposure). |
| **Dataplane V2 + default-deny** | The advanced data plane, with a default-deny network policy in every namespace. |

### 3.3 Cluster and nodes

| Entity | What it is, and our choice |
|---|---|
| **Regional cluster** | Control plane and nodes across three zones (survives one zone loss). The default node pool is removed; pools are managed explicitly. |
| **Release channel** | On a channel, for automatic, tested control-plane and node upgrades. |
| **Hardening (built in)** | Verified-boot ("shielded") nodes, **Container-Optimized OS (COS)** nodes, secret encryption with our key, Workload Identity, Dataplane V2 default-deny — clearing the **Center for Internet Security (CIS) Level 2** floor. |
| **Observability** | System and workload logs and metrics, plus managed Prometheus, on by default. |
| **Maintenance + ingress** | A set maintenance window; the Gateway interface enabled for ingress (TC-6); deletion protection on. |
| **Node pools** | A general hardened pool; a **Confidential (memory-encrypting)** pool is an optional per-cluster add, always on-demand (never spot). Autoscaling is deferred. |

### 3.4 Access — no public endpoint

| Entity | What it is, and our choice |
|---|---|
| **Private nodes + private control plane** | Nodes have no public address. The control plane uses the **DNS-based private endpoint** (the current recommended approach) with external access off — there is no public API endpoint to attack. |
| **Fleet membership** | The cluster joins a fleet — the prerequisite for Connect Gateway and later fleet features. |
| **Connect Gateway** | The only way operators and automation reach the cluster's API: by IAM plus in-cluster role-based access control, with no network allow-lists or virtual private network (VPN). |

### 3.5 Supply chain

| Entity | What it is, and our choice |
|---|---|
| **Artifact Registry** | One private image repository per project; public base images are mirrored through a proxy repository so the cluster never pulls from the internet directly. The node identity has read-only access. |
| **Binary Authorization** | An admission policy that admits only trusted images. Starts in **audit** (logs, does not block) and switches to **enforce** once the signing pipeline exists. |

## 4. Build order

1. Project → enable services → force-create the service agents.
2. KMS keyring + key → the two key grants.
3. Node service account and its roles.
4. VPC → subnet + secondary ranges → Private Google Access / DNS (Cloud NAT if needed).
5. Artifact Registry (+ reader grant) and the Binary Authorization policy.
6. Cluster (regional, private, hardened, Workload Identity, our key, Dataplane V2) → fleet membership.
7. Node pools (general; Confidential if requested).
8. Connect Gateway access (IAM + in-cluster roles).

## 5. Decisions, in brief

- **One project per environment** — clean isolation, cost, and blast-radius boundary.
- **No public control-plane endpoint; DNS-based private endpoint + Connect Gateway** — the recommended modern path; nothing reaches the API over the internet.
- **Our own key for secrets and disks** — control and auditability; two grants because Google uses two managed identities.
- **Least-privilege node identity + Workload Identity** — nodes can do little; pods get scoped identities, not node keys.
- **Regional + release channel** — high availability and automatic patching.
- **Dataplane V2 default-deny, CIS Level 2** — secure by default.
- **Images only from our registry; Binary Authorization audit → enforce** — no untrusted images.

## 6. Open items

- **Control-plane endpoint** — confirm the DNS-based endpoint on our GKE version at build (it is the recommended approach, but verify availability).
- **Binary Authorization** — the newer check-based policy is still Preview; start with the generally-available project policy.
- **Service-to-service mutual TLS / service mesh** — still open (requirements §8).
- **Autoscaling** — deferred.

## 7. Related

[requirements.md](../requirements.md) · [technology-choices.md](../technology-choices.md) · [security-requirements.md](../security-requirements.md).
