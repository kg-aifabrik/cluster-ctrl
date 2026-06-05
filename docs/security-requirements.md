# cluster-ctrl — Security & Hardening Requirements

*Status: draft for review · Date: 2026-06-05*
*Companion to [requirements.md](requirements.md). Iterated separately.*

---

## Summary

Every cluster is hardened during the build to one standard, and a baseline report is
produced. From then on it is continuously reconciled to its committed security, scanned
on a schedule, and alerted on when something drifts or a check goes silent. All evidence
is archived and tamper-evident. The same standard applies to every cluster and every
namespace. Secrets are managed centrally and never committed, and all communication is
encrypted in transit.

---

## Requirements (SEC)

- **SEC-1 — Hardened at build, by automation.** Hardening is applied during the build,
  not added afterwards, by automation that continuously reconciles each cluster to the
  configuration committed in Git.
- **SEC-2 — The hardening standard.** Every cluster meets the same standard, in three
  parts.

  *Cloud and cluster controls:*
  - a private control-plane endpoint (no public address);
  - verified-boot ("shielded") nodes;
  - per-workload cloud identity, so pods never use the node's credentials;
  - secrets encrypted with a **customer-managed encryption key (CMEK)** in Cloud **Key
    Management Service (KMS)**;
  - pod-to-pod traffic denied by default unless explicitly allowed;
  - only signed, provenance-bearing container images from the trusted Artifact Registry
    path admitted, enforced by **Binary Authorization** — audit mode (logged, not
    blocked) until the image-signing pipeline exists, then enforced with no exception
    path. Admission is permissive in development and enforced in staging and production
    (see the image supply chain, WLD-3 in [requirements.md](requirements.md), and the
    open questions below).

  *In-cluster guardrails:*
  - **Pod Security Standards (PSS)** (built-in Kubernetes pod restrictions) at the
    *restricted* level;
  - admission policies blocking unsafe pod settings (privileged containers, host access,
    privilege escalation, and similar);
  - per-namespace **role-based access control (RBAC)** with no broad wildcards;
  - a default-deny network policy in every namespace;
  - no automatic mounting of default service-account tokens;
  - required resource limits on workloads.

  *The floor:* the **Center for Internet Security (CIS)** Kubernetes Benchmark at
  **Level 2**.
- **SEC-3 — One standard for every cluster and namespace.** The standard is identical
  for every cluster, and every namespace receives the identical security stamp — a single
  template applied the same way everywhere; only the namespace's own name and ownership
  label are substituted.
- **SEC-4 — A baseline report at build.** A baseline posture report is generated and
  archived as a cluster's starting evidence.
- **SEC-5 — Security is a closed loop.** The live cluster is continuously reconciled to
  its committed security and self-heals drift — no person is in the steady-state loop.
- **SEC-6 — Day-2 posture reports.** Posture reports run on a schedule and on demand
  against every cluster; each run is archived as evidence.
- **SEC-7 — Verification tooling and evidence retention.** Posture is measured with
  **kube-bench** (CIS benchmark checker) and **kubescape** (posture scanner). Reports are
  archived, retained for a defined period, and kept append-only and tamper-evident — no
  archived report is silently overwritten or removed.
- **SEC-8 — Alert on anomalies and on silence.** Anomalies (corrected drift, benchmark
  regressions) raise a **Slack** alert; so does a silence — unhealthy enforcement or a
  scheduled scan that fails to run.
- **SEC-9 — Secret management.** Secrets are never baked into container images, committed
  to Git, or left as long-lived plaintext. Workloads read secrets from a managed secrets
  service through per-workload identity, **mounted at runtime rather than held as standing
  Kubernetes Secret objects by default**. Git holds only *references* to secrets; the
  values live in the managed secrets service, encrypted with a customer-managed encryption
  key (CMEK). Platform and automation credentials — for example the console's GitHub App
  key and the alerting webhook — are stored the same way and read via workload identity,
  never hard-coded or logged. (How: [technology-choices.md](technology-choices.md) TC-13
  and TC-2.)
- **SEC-10 — Encryption in transit.** All communication is encrypted with **Transport
  Layer Security (TLS)** — at the public edge, from the edge into the cluster, and
  service-to-service. No plaintext on the wire. The *mechanism* for enforcing
  service-to-service mutual TLS is an open technology choice (see open questions).

---

## The per-namespace security stamp (for discussion)

The *existence* of a single, identical stamp on every namespace is settled (**SEC-3**);
the *contents* below are proposals to confirm.

| Element | Proposed default | To decide |
|---|---|---|
| **Pod Security Standards** | *restricted* level. | Confirm. |
| **Network policy** | Default-deny ingress/egress, then allow: cluster DNS, the Kubernetes API server, same-namespace traffic, named platform services, and Google APIs / Artifact Registry. | External egress: allow-with-logging vs. deny-with-opt-in. (DNS-only egress would break normal workloads.) |
| **RBAC + service account** | A scoped namespace role and a namespace service account; no default token mounting. | Confirm. |
| **Resource quota + limit range** | A quota and limit range, with default requests/limits injected. | One fixed size applied identically, or named sizes (small/medium/large) at creation. |
| **Ownership label** | An owner/team label for audit. | Confirm. |

---

## Open questions

1. **Image-signing pipeline** — establishing it is the trigger for moving signed-image
   admission from audit to enforcement (SEC-2). When and how?
2. **Namespace stamp specifics** — external egress (allow-with-logging vs. deny-with-opt-in)
   and quota sizing (one fixed size vs. named sizes).
3. **Image supply-chain enforcement** — the Binary Authorization policy (signature,
   provenance/SLSA, trusted registry path, image freshness, vulnerability threshold),
   vulnerability scanning via Artifact Analysis, and the dev (audit) vs. staging/production
   (enforce) split. Pairs with WLD-3 in [requirements.md](requirements.md).
4. **Service-to-service mutual TLS mechanism (SEC-10).** All communication must be TLS
   (firm); *how* east-west mutual TLS is enforced is open. Leading candidate: a managed
   service mesh (Cloud Service Mesh) with a mesh-wide `STRICT` policy and its built-in
   certificate authority, which would also issue internal/workload certificates and
   supersede the private-CA approach in TC-7. Kept open for now.
