# cluster-ctrl — Technology Choices

*Status: draft for review · Date: 2026-06-05*

The **how** companion to [requirements.md](requirements.md). The requirements state
*what* we need; this document records the technology chosen to meet each, and why.
Each entry follows one format:

> **Requirement** → **Considerations** → **Choice** → **Rationale**

Security-specific tooling (benchmark scanners, secret encryption, admission-policy
detail) will be added here as the [security requirements](security-requirements.md) are
iterated. Choices marked **Open** still have a sub-decision to confirm.

---

## TC-1: Provisioning and automation engine

- **Requirement:** FND-1, BLD-1, BLD-3, GOV-2 — automated, reviewed, repeatable cluster
  builds and changes.
- **Considerations:** a continuously-reconciling cloud control plane (Crossplane, Config
  Connector); a cluster product (Rancher, Cluster API); or infrastructure-as-code run by
  a pipeline. For the operator interface: buy an internal developer platform, or build a
  small purpose-made console.
- **Choice:** Terraform for provisioning, run by a continuous-integration pipeline, with
  a purpose-built operator console.
- **Rationale:** simplest path for ≤6 clusters; credentials are short-lived per run;
  every change is previewed and approved before apply; reuses Google's hardened modules
  and common skills. Control-plane tools need an always-on, highly-privileged component (a
  standing attack surface); cluster-only products can't serve future non-cluster resources.

## TC-2: Source of truth, change flow, and approvals

- **Requirement:** GOV-1, GOV-2, GOV-3, GOV-4, BLD-1, OPS-3 — a version-controlled source
  of truth; every change reviewed, approved, previewed, and applied by automation.
- **Considerations:** direct cloud changes vs. reviewed change requests; self-hosted vs.
  hosted Git; where approval gating and the pipeline live.
- **Choice:** Git hosted on **GitHub**. Changes are **pull requests**; automation runs in
  **GitHub Actions**; applies are gated by **GitHub Environments**; approval authority is
  set by **GitHub organization team** membership.
- **Rationale:** one platform for source of truth, review, pipeline, and approval gating;
  native required-reviewer controls satisfy "no auto-apply"; team membership is an
  existing, auditable authority model.

## TC-3: Keyless cloud authentication

- **Requirement:** FND-4 — short-lived, keyless automation credentials, scoped per
  repository and environment.
- **Considerations:** downloadable service-account keys vs. federated short-lived tokens.
- **Choice:** **Workload Identity Federation** between GitHub Actions and Google Cloud;
  **Workload Identity** for in-cluster workloads.
- **Rationale:** no long-lived keys to leak or rotate; tokens last about an hour and are
  scoped to the specific identity; per-environment identities contain blast radius.

## TC-4: Private cluster access

- **Requirement:** FND-5, FND-6 — private, identity-controlled access with no public
  endpoints.
- **Considerations:** a public control-plane endpoint with authorized networks; a
  VPN/bastion; Google's fleet gateway.
- **Choice:** **GKE Connect Gateway** (via fleet membership).
- **Rationale:** no public endpoint and no network allow-lists or VPN to operate; access
  is identity-controlled and logged through Google's managed service.

## TC-5: Console authentication

- **Requirement:** OPS-2 — operators sign in with enterprise single sign-on.
- **Considerations:** local accounts; an external identity provider; the organization's
  existing directory.
- **Choice:** **Google Workspace single sign-on (SSO)**.
- **Rationale:** the team's existing identity; no separate account store; central
  deprovisioning.

## TC-6: Ingress and external traffic

- **Requirement:** WLD-2 — a single, TLS-terminating front door for traffic into a cluster.
- **Considerations:** the NGINX ingress controller; **GKE Gateway API**; other conformant
  controllers (Envoy Gateway, Istio, Cilium).
- **Choice:** **GKE Gateway API** — a global external gateway for internet-facing
  services, a regional internal gateway for internal services.
- **Rationale:** ingress-nginx is being retired (best-effort maintenance only to ~March
  2026, after the 2025 "IngressNightmare" critical vulnerabilities); GKE Gateway is
  Generally Available and Google's recommended direction, and removes a self-hosted,
  internet-facing admission controller.
- **Trade-offs accepted:** no native rate limiting / Web Application Firewall / external
  auth (use **Cloud Armor** and **Identity-Aware Proxy**); no regex path matching; ~16
  rules per route; advanced behaviour rides on GCP-specific policy resources (not
  portable). Acceptable here (GCP-only, ≤6 clusters). If hard multi-cloud or on-premises
  portability ever appears, switch to a portable conformant controller on GKE — not back
  to ingress-nginx.

## TC-7: TLS certificates

- **Requirement:** WLD-2 — automated TLS certificates for public and internal endpoints.
- **Considerations:** cert-manager with a public ACME authority (Let's Encrypt, Google
  Trust Services, ZeroSSL, Buypass); Certificate Manager Google-managed certs; Google
  Certificate Authority Service (CAS) for a private CA.
- **Choice:** public endpoints → **Certificate Manager Google-managed certificates** on
  the gateway; internal endpoints → a private CA in **Certificate Authority Service**,
  issued via **cert-manager**.
- **Rationale:** managed issue/renew with no ACME plumbing for the public edge (first 100
  certs free). Google-managed *public* certs cannot serve internal endpoints, so internal
  and mutual TLS need a private CA; CAS is managed and hardware-security-module-backed.
  (Classic Google-managed SSL certs aren't supported on regional/internal load balancers —
  Certificate Manager is the supported route.)
- **Open:** internal CA tier and trust distribution/rotation; confirm Certificate Manager
  attachment on the chosen gateway class.

## TC-8: Service exposure and DNS

- **Requirement:** WLD-2 — sensible default exposure and name resolution.
- **Considerations:** internal vs. external by default; in-cluster DNS; automatic public
  DNS records.
- **Choice:** **internal load balancer by default**, external opt-in; **CoreDNS/kube-dns**
  for in-cluster name resolution; **public DNS records left to the SRE team** (not in the
  baseline).
- **Rationale:** least exposure by default; in-cluster DNS is required by the namespace
  network policy; public DNS is low-volume and better owned by SRE for now.

## TC-9: Workload storage

- **Requirement:** WLD-2 — encrypted persistent storage out of the box.
- **Considerations:** block (Persistent Disk) vs. shared file (Filestore); reclaim,
  binding, and expansion defaults.
- **Choice:** default storage class on **Google Persistent Disk**, encrypted with a
  customer-managed key; `Delete` reclaim, `WaitForFirstConsumer` binding, volume expansion
  enabled. **Filestore** (shared, multi-writer) is out of scope for now.
- **Rationale:** covers the common single-writer case securely; shared storage adds cost
  and complexity and isn't needed yet.

## TC-10: In-cluster observability

- **Requirement:** WLD-2, REL-1 — workload metrics and logs without per-team setup.
- **Considerations:** cloud-native logging/monitoring only vs. also a metrics stack.
- **Choice:** **GKE managed Prometheus** plus standard logging and monitoring agents,
  enabled in the baseline.
- **Rationale:** workloads get metrics and dashboards out of the box; managed, so no
  self-run monitoring stack to operate.

## TC-11: Workload scheduling and autoscaling

- **Requirement:** WLD-7 (resilient scheduling guidance); WLD-2 (autoscaling).
- **Considerations:** templating disruption protection into every namespace vs. guidance
  in recipes; including autoscaling now vs. later.
- **Choice:** **priority classes and PodDisruptionBudgets via the workload recipes**
  (guidance, not enforced). **Autoscaling is deferred** to a later enhancement.
- **Rationale:** disruption protection is workload-specific, so guidance fits better than
  a one-size template; autoscaling isn't needed for the first cut.

## TC-12: Image supply chain and admission

- **Requirement:** WLD-3, and SEC-2 (admission of trusted images).
- **Considerations:** pulling from public registries directly vs. an internal registry;
  how images are scanned, signed, given provenance, and admitted.
- **Choice:** **Artifact Registry** as the only registry clusters pull from, with
  **remote/virtual repositories** mirroring public images; build **provenance (Supply-chain
  Levels for Software Artifacts, SLSA)** and **signing** (a build attestor and/or
  cosign/Sigstore) in the pipeline; **vulnerability scanning** (Artifact Analysis);
  admission gated by **Binary Authorization** — **audit in development, enforced (blocking)
  in staging and production**.
- **Rationale:** clusters never pull untrusted images from the internet; only signed,
  provenance-bearing, scanned images from the trusted path run; the dev/prod split keeps
  developers fast while protecting higher environments. The gate sits at the registry and
  admission boundaries, so it is uniform regardless of each team's pipeline.

## TC-13: Workload secrets

- **Requirement:** WLD-4 — a sanctioned way for workloads to consume secrets.
- **Considerations:** plain Kubernetes Secrets vs. an external managed secrets service.
- **Choice:** **Google Secret Manager** surfaced through the **Secrets Store driver**,
  accessed with **Workload Identity**.
- **Rationale:** one audited path; secrets are never baked into images or committed;
  per-workload identity scopes access.
