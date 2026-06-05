# cluster-ctrl — Requirements

*Status: draft for review · Date: 2026-06-05*

This document states **what** we need. The technology chosen to meet each requirement is
recorded in **[technology-choices.md](technology-choices.md)**; security and hardening
requirements are in **[security-requirements.md](security-requirements.md)**.

---

## 1. Overview

**cluster-ctrl** builds and runs hardened **Google Kubernetes Engine (GKE)** clusters
— Google's managed Kubernetes service — on Google Cloud, for the AiFabrik operations
team. Operators work with it two ways:

- **Cluster build** — a reviewed change in Git is turned into a fully built, hardened
  cluster by automation. No cluster is hand-built.
- **Operator Console** — a small web application for day-to-day administration of
  existing clusters: infrastructure changes and namespace/quota setup.

Every cluster is built to one standard, hardened at build, and kept secure
continuously. Every cluster change is reviewed, approved, and applied by automation;
Git is the record of what every cluster is. Application teams deploy their own workloads
through a separate, GitOps-based pathway owned by the SRE team — not through the console.

Starting scope: three environments (development, staging, production) × two purposes =
at most six clusters. Built simple, designed to extend.

---

## 2. Goals

1. Build a hardened cluster reliably and repeatably, with no manual steps after a
   one-time setup.
2. Make everyday operations safe and simple for the Site Reliability Engineering (SRE)
   team.
3. Make every cluster secure by default, and prove it with evidence.
4. Make every change reviewable, approved, and traceable to a person.

---

## 3. Principles

1. ***Simplicity*** — prefer the simplest approach that meets the need.
2. ***Audit-ready*** — every change traces to a recorded, approved decision; nothing
   changes by hand.
3. ***Secure by design*** — one security standard for every cluster, kept true
   continuously.

---

## 4. Key terms

| Term | Meaning |
|---|---|
| **Environment** | An isolated stage — **development (dev)**, **staging (stage)**, or **production (prod)** — each its own Google Cloud project. |
| **Purpose** | What a cluster is for: the **Fleet Operations Plane (FOP)** (runs our own operations tooling) or the **Management Plane (MGMT)** (the end-user product surface). |
| **Cluster** | One regional GKE cluster, identified by environment, purpose, and name. |
| **Configuration ("shape")** | What a cluster is made of: node pools, machine types, Kubernetes version, and options. |
| **Namespace / tenant** | A walled-off slice of a cluster for one team, individual, or purpose — the unit of isolation between tenants. |
| **Day-2 operations** | Ongoing administration of a cluster after it is built (e.g. resizing a node pool, creating a namespace), as opposed to building it (Day-1). |
| **Operator** | An SRE team member who builds and runs clusters — the only user of the console and build path. |
| **Approver** | One of a defined list of approved SRE GitHub users who may approve a change before it is applied. |
| **Infrastructure repository** | The Git repository holding cluster configuration; separate from the repositories that hold application workloads. |

---

## 5. Roles

- **SRE operators** — the only users of the console and build path. They build and
  operate clusters, set up namespaces and quotas (via the console), and own the
  GitOps pathway teams use to deploy workloads.
- **Approvers** — a defined list of approved SRE GitHub users; the same list gates
  infrastructure-change pull requests and staging/production deployments
  ([TC-2](technology-choices.md#tc-2-source-of-truth-change-flow-and-approvals)).
- **Application teams** — own their workloads (manifests in their own repositories,
  deployed via GitOps; self-service in development). They do not build, operate, or
  change clusters, and do not use the console.

---

## 6. Scope

**In scope:** building hardened GKE clusters through automation; hardening them at
build and keeping them secure continuously; Day-2 infrastructure administration and
namespace/quota setup through the Operator Console; providing the GitOps pathway the
SRE team uses to deploy workloads; at most six clusters across three environments and
two purposes.

**Out of scope:**

- On-premises or edge clusters (a separate system); more than one cloud.
- **Self-service cluster management** — only SRE builds/operates/changes clusters.
- **Workload deployment tooling and specifics** — the deployment pathway is owned by the
  SRE team and operated via GitOps; this project provides a GitOps-ready cluster, not the
  deployment mechanism (see **WLD-6**). Tenant deployments never go through the console.
- **Cost reporting features** — usage and cost by cluster and by namespace are viewed
  through GKE's native cost tooling
  ([TC-14](technology-choices.md#tc-14-cost-visibility)), not built into the console.
- **Graphics-processing-unit (GPU) node pools** — not planned for these clusters.
- Database/storage operations beyond provisioning.

---

## 7. Requirements

Seven themes, ordered within each by the order things happen in a cluster's life.
Where a requirement is met by a specific technology, the choice is linked in
[technology-choices.md](technology-choices.md).

### 7.1 Foundation & Platform (FND)

*What a human puts in place once, before automation can run, and how it is checked.*

- **FND-1 — One-time setup enables automation.** A human completes a one-time setup
  that creates the projects, identities, and permissions automation needs; after it,
  building clusters is fully automated.
- **FND-2 — The setup is verified.** A programmatic check confirms every authorization
  automation depends on is correctly in place — cloud permissions and identities,
  repository trust, storage and access wiring, console sign-in and approval wiring —
  right after setup and again after any change to it, before automation is relied on.
- **FND-3 — One project per environment.** Each environment is its own Google Cloud
  project — the boundary for isolation and cost.
- **FND-4 — Keyless, per-environment automation identity.** Automation uses short-lived,
  keyless credentials. Each environment has its own identity, so a compromise in one
  environment cannot reach another.
  ([TC-3](technology-choices.md#tc-3-keyless-cloud-authentication))
- **FND-5 — Private access only.** Clusters have no public control-plane endpoint; all
  operator and automation access is private and identity-controlled — no public
  addresses, allow-lists, or virtual private network (VPN).
  ([TC-4](technology-choices.md#tc-4-private-cluster-access))
- **FND-6 — An always-on platform capability.** A long-running platform component,
  created at setup, hosts the console and runs the continuous security reconciliation and
  scheduled scans (see [security-requirements.md](security-requirements.md)). It lives in
  its own dedicated project, is hardened like any cluster, and is reached only over the
  private access path.

### 7.2 Cluster Build (BLD)

*How a cluster comes into being — proposed in Git, built from one standard recipe.*

- **BLD-1 — Built from a reviewed change in Git.** An operator builds a cluster by
  proposing a reviewed change (a **pull request**) to the infrastructure repository; once
  approved, automation builds it. Creation is not done in the console.
  ([TC-1](technology-choices.md#tc-1-provisioning-and-automation-engine),
  [TC-2](technology-choices.md#tc-2-source-of-truth-change-flow-and-approvals))
- **BLD-2 — Two dimensions, one cluster per choice.** A build is one environment × one
  purpose → one cluster; the 3 × 2 grid gives the at-most-six clusters.
- **BLD-3 — One standard build recipe.** Every cluster is built from the same recipe —
  no per-cluster bespoke construction.
- **BLD-4 — Purpose does not change the build.** Clusters are built identically; purpose
  may shape later Day-2 choices (node pools, machine types).
- **BLD-5 — Highly available by default.** Every cluster is regional — control plane and
  nodes across availability zones — and survives the loss of one zone.
- **BLD-6 — Configuration lives in Git.** A cluster's configuration is recorded in Git
  and changed only through reviewed changes.
- **BLD-7 — The recipe can be extended.** It can grow to new cluster capabilities and,
  later, new kinds of cloud resource.

### 7.3 Security & Hardening (SEC)

*Every cluster is hardened during the build to one standard with a baseline report,
then kept secure by continuous reconciliation, scheduled scans, and alerting.*

Detailed requirements are maintained separately in
**[security-requirements.md](security-requirements.md)** and will be iterated there.

### 7.4 Tenancy & Workloads (WLD)

*How teams get space on a cluster, get images and secrets in safely, and how workloads
reach the cluster.*

- **WLD-1 — Tenancy model.** A cluster hosts one or more teams, each in its own
  namespace. Dev has many; stage and prod have few (as few as one), all SRE-managed.
- **WLD-2 — Ready to host workloads out of the box.** Every cluster ships with what
  teams need for typical workloads — pods, deployments, internal/external services,
  **Transport Layer Security (TLS)**, and persistent storage. The components that provide
  this are chosen in [technology-choices.md](technology-choices.md) (TC-6 through TC-11).
- **WLD-3 — Image supply chain and registry.** Clusters pull container images only from a
  single trusted internal registry; public base images are mirrored through it, never
  pulled directly from the internet. Images are built and signed in the pipeline with
  build provenance, and only signed, provenance-bearing images from the trusted path are
  admitted. Admission is permissive (audit) in development and enforced (blocking) in
  staging and production.
  ([TC-12](technology-choices.md#tc-12-image-supply-chain-and-admission))
- **WLD-4 — Workload secrets.** Every cluster provides a sanctioned way for workloads to
  read secrets — a managed secrets service surfaced to workloads with per-workload
  identity — so teams never bake secrets into images or commit them. (Cluster secret
  encryption is part of the security standard.)
  ([TC-13](technology-choices.md#tc-13-workload-secrets))
- **WLD-5 — Namespace and quota setup.** Creating a namespace, with its resource quota
  and the standard security stamp applied, is a Day-2 console action after the first
  release (OPS-4); it submits a reviewed change and is done by SRE.
- **WLD-6 — A GitOps pathway to deploy workloads.** Each cluster is GitOps-ready and
  provides the pathway the SRE team uses to deploy workloads, from repositories separate
  from the infrastructure repository. Development deployments are self-service; staging
  and production deployments require approval by the SRE approver list (GOV-3). The
  deployment tooling and specifics are owned by the SRE team and are out of scope for
  this project; deployments never go through the console.
- **WLD-7 — Workload recipes are reference examples.** Markdown reference examples show
  how to deploy common workload categories — including a known-good Deployment that
  complies with the security standard, and guidance on scheduling resilience
  ([TC-11](technology-choices.md#tc-11-workload-scheduling-and-autoscaling)).

### 7.5 Day-2 Operations & Console (OPS)

*How operators run a cluster day to day, through a console that never changes a cluster
directly. The console is used only for infrastructure changes and namespace/quota setup
— never for workload deployment (WLD-6).*

- **OPS-1 — Operations run through the Operator Console.**
- **OPS-2 — Sign-in with single sign-on.** Operators sign in with enterprise single
  sign-on (SSO). ([TC-5](technology-choices.md#tc-5-console-authentication))
- **OPS-3 — The console never changes a cluster directly.** Every action submits a
  reviewed change that automation applies; the console has no direct write access.
- **OPS-4 — First-release capabilities:**
  - **OPS-4a — View what is provisioned** at the cluster level — node pools, machine
    types, versions, configuration, status — showing both the Git-recorded intent and
    the live state so an operator can see whether they match. Read-only.
  - **OPS-4b — Manage node pool size** — add or remove nodes; add or remove a pool.
- **OPS-5 — The console is designed to be extended.** Later capabilities include
  namespace and quota setup, cluster and node-pool upgrades, maintenance windows, node
  repair, and lifecycle actions — added without reworking the foundations.

### 7.6 Change Governance & Audit (GOV)

*How every change is controlled and recorded.*
([TC-2](technology-choices.md#tc-2-source-of-truth-change-flow-and-approvals))

- **GOV-1 — Git is the single source of truth.** Nothing changes by hand or bypasses Git.
- **GOV-2 — Every cluster change is reviewed and approved.** Every change to a cluster
  (build, configuration, security) is a reviewed change requiring SRE approval before
  automation applies it — no automatic apply anywhere, including dev. (Workload deployment
  follows WLD-6.)
- **GOV-3 — Approvals by a defined SRE list.** A single list of approved SRE GitHub users
  are the only ones who may approve infrastructure-change pull requests and
  staging/production deployments.
- **GOV-4 — Preview, then apply exactly what was approved.** A change is previewed before
  approval and applied exactly as approved; applies are serialized per cluster/environment,
  and a change that has gone stale fails safely and is previewed again.
- **GOV-5 — A complete audit trail.** For any change, the record shows who requested it,
  who approved it, and what changed; Git history, approvals, logs, and reports
  reconstruct any cluster's past.

### 7.7 Reliability & Ways of Working (REL)

*The engineering qualities the whole system holds itself to.*

- **REL-1 — Observability.** Console and automation emit structured logs, metrics, and
  traces with a correlation identifier and expose health checks; the continuously running
  parts report their own health; secrets and personal data are never logged.
- **REL-2 — Resilience.** Networked calls use timeouts, retries with backoff, and
  idempotency, and degrade gracefully.
- **REL-3 — Reproducibility.** Versions are pinned and lock files committed.
- **REL-4 — Least privilege and secret hygiene.** Every identity runs with least
  privilege; secrets come from a secrets manager or the environment, never hard-coded or
  logged.
- **REL-5 — Availability target.** The console targets 99.5% monthly availability — an
  internal target, not a contractual service-level agreement. No further service-level
  targets are set at this time. (The console is Day-2-only and never changes a cluster
  directly, so its downtime blocks operator convenience, not cluster operation.)

---

## 8. Open questions

1. **Secrets and TLS certificates** — the approach is under active discussion; it will
   refine WLD-2, WLD-4, and technology choices
   [TC-7](technology-choices.md#tc-7-tls-certificates) and
   [TC-13](technology-choices.md#tc-13-workload-secrets).
2. **Namespace security stamp specifics** — external egress model and quota sizing
   (tracked in [security-requirements.md](security-requirements.md)).
