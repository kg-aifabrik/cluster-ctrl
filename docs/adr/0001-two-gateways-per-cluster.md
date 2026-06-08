# ADR-0001: Two gateways per cluster (internal + external), shared via ReferenceGrant

- **Status:** Accepted
- **Date:** 2026-06-08
- **Deciders:** SRE
- **References:** [google-cloud-design.md §8](../designs/google-cloud-design.md); technology-choices TC-6, TC-8

## Context

Ingress serves two exposure classes: internal tools (no customer traffic, private) and
public, internet-facing services. A cluster has more than one namespace, and the number of
namespaces taking ingress is small but not fixed.

## Decision drivers

- A consistent, centrally enforced edge (WAF, TLS, HTTP→HTTPS redirect).
- Cost — each load balancer and reserved IP is a recurring charge.
- Isolation between the internal and public exposure classes.
- Operability for a small SRE team.

## Considered options

1. **One shared gateway for all traffic.** Rejected: internal-only and internet-facing
   endpoints cannot share one gateway without exposing internal tools at the edge.
2. **One gateway per namespace.** Rejected: the gateway and load-balancer count grows with
   namespaces, the security configuration drifts per team, and there is no cost saving because
   the two exposure classes already force at least two gateways.
3. **Two gateways per cluster, one per exposure class, shared across namespaces via HTTPRoute
   attachment (cross-namespace attachment via `ReferenceGrant`).** Chosen.

## Decision

Each cluster runs exactly two platform-owned gateways — an internal `gke-l7-rilb` and an
external `gke-l7-global-external-managed`. Workload namespaces attach HTTPRoutes; a route
outside the gateway's namespace attaches through a `ReferenceGrant` the platform issues per
namespace. The security baseline (Cloud Armor, SSL policy, redirect, certificates) lives in
Terraform-owned Google Cloud resources, instantiated for both gateways from one reusable
module, so the posture is uniform regardless of gateway or namespace count.

## Consequences

- **Good:** a fixed two-load-balancer footprint regardless of namespace count; a uniform
  security posture by construction; clean isolation between internal and public traffic.
- **Bad:** cross-namespace attachment requires a `ReferenceGrant` per namespace, and hostname
  and path allocation on a shared gateway must be coordinated.
