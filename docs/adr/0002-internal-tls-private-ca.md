# ADR-0002: Internal endpoint TLS uses a private CA in Certificate Authority Service

- **Status:** Accepted
- **Date:** 2026-06-08
- **Deciders:** SRE
- **References:** [google-cloud-design.md §8](../designs/google-cloud-design.md); technology-choices TC-7; [security-requirements.md SEC-10](../security-requirements.md)

## Context

The internal gateway terminates TLS for internal-tool hostnames. Internal hostnames must not
appear in public Certificate Transparency (CT) logs. The clients are humans using browsers.

## Decision drivers

- Internal hostnames stay private — no public CT disclosure.
- Automated issuance and rotation.
- Forward compatibility with a possible future service mesh.
- Cost is not a constraint.

## Considered options

1. **Public Certificate Manager certificate for an owned-domain internal hostname**
   (DNS-authorized). Rejected: publicly trusted certificates are logged in CT, which discloses
   internal hostnames; a wildcard certificate was also rejected.
2. **Self-managed in-cluster CA** (cert-manager self-signed root). Rejected: the operator must
   protect the root key, audit is weaker, and a future mesh would replace it.
3. **Private CA in Certificate Authority Service (CAS), issued via cert-manager +
   `google-cas-issuer`.** Chosen.
4. **Service-mesh CA.** Deferred (see ADR-0003); the mesh is a security-phase decision.

## Decision

Internal endpoint certificates are issued by a private CA in CAS: a long-lived root with
per-environment subordinate CAs. cert-manager with `google-cas-issuer` requests leaf
certificates and writes them to Secrets the gateway references (`tls.certificateRefs`). The
CAS root reaches human/browser clients through MDM and in-cluster service clients through
trust-manager.

## Consequences

- **Good:** internal hostnames never enter public CT; a managed CA with IAM and audit;
  automated leaf rotation; forward-compatible with a mesh (ADR-0003).
- **Bad:** CAS carries a recurring cost; trust distribution to browsers (MDM) is operator-owned
  work.
