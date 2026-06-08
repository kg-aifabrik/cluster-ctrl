# ADR-0003: Service mesh deferred to the security phase; design kept forward-compatible

- **Status:** Accepted
- **Date:** 2026-06-08
- **Deciders:** SRE
- **References:** [google-cloud-design.md §8](../designs/google-cloud-design.md); [security-requirements.md SEC-10](../security-requirements.md); requirements §8; iac-gke issues #15, #17

## Context

East-west, pod-to-pod mutual TLS (SEC-10) could be delivered by a service mesh. The mesh
decision is interrelated with namespace authorization and posture, which belong to the
security milestone. The ingress milestone must neither block on the mesh nor foreclose it.

## Decision drivers

- Keep the ingress milestone tight and shippable.
- Avoid building a mesh subsystem (sidecar rollout, staged `STRICT` enforcement, authorization
  policy) ahead of its sibling security work.
- Avoid rework when a mesh is later adopted.

## Considered options

1. **Build the mesh inside the ingress milestone.** Rejected: a large, staged effort coupled to
   namespace authorization, with sidecar resource overhead on small dev nodes, that pulls a
   security requirement into the build milestone.
2. **Defer the mesh to the security phase and keep the ingress/CA design forward-compatible.**
   Chosen.

## Decision

East-west mTLS and any service mesh are built in the security phase. The ingress design is
forward-compatible: CAS is the single private root (Cloud Service Mesh, if adopted, uses the
same CAS as its certificate authority); the GKE gateway remains the north-south entry point and
no mesh ingress gateway replaces it; the gateway→pod hop is secured later with a GKE
`BackendTLSPolicy` (re-encryption) or sidecar permissive mode.

## Consequences

- **Good:** the ingress milestone stays focused and shippable; the CAS and gateway work is
  reused, not replaced, when a mesh lands.
- **Bad:** east-west pod-to-pod traffic (including the gateway→pod hop) is not yet mTLS-encrypted
  until the security phase.
