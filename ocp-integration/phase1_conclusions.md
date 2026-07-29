# Phase 1 Research Conclusions: NVCF on OpenShift

## Security & SCCs

Most NVCF components run as non-root (uid 1000) and fit within OpenShift's default `restricted` SCC. The NVCA Operator already has explicit OpenShift SCC awareness -- its RBAC references `security.openshift.io` and the `nonroot` SCC.

The only problematic component is the **Container Cache DaemonSet**, which requires `privileged` SCC due to `hostPID`, `hostIPC`, `hostNetwork`, and `privileged: true`. It modifies containerd/CRI-O config on nodes, which also conflicts with CoreOS immutable nodes.

NVCF Unbound (DNS) needs the `SYS_RESOURCE` capability but is otherwise standard.

## Networking

NVCF uses **Gateway API with Envoy Gateway** -- not traditional K8s Ingress or OpenShift Routes.

- HTTPRoute resources handle HTTP traffic (API, invocation, LLM gateway)
- TCPRoute resources handle gRPC (port 10081) and NATS (port 4222)
- OpenShift Routes have no TCPRoute equivalent

Three options identified:
- **Option A**: Install Envoy Gateway on OpenShift (recommended for PoC, least changes to NVCF)
- **Option B**: Rewrite HTTPRoutes as OpenShift Routes (more native, loses TCP routing)
- **Option C**: Hybrid -- Routes for HTTP, NodePort/LoadBalancer for gRPC and NATS

## Operator & OLM

NVCF deploys via Helm/Helmfile with strict ordering (infra deps -> core services -> ingress -> NVCA). It is not an OLM operator. GPU Operator and NFD are already on OperatorHub and are prerequisites, not conflicts. Wrapping NVCF in an OLM operator is a possible future PR proposal.

## Image Registries

NVCF pulls from NGC (nvcr.io) using per-namespace `docker-registry` secrets. This is compatible with OpenShift's pull secret mechanism. The Container Cache's direct CRI-O configuration is the exception -- may conflict with CoreOS.

## Summary

| Area | OpenShift Readiness | Effort |
|------|-------------------|--------|
| Security/SCCs | Most components fit restricted SCC | Low |
| NVCA Operator | Already has SCC awareness | Low |
| GPU Operator + NFD | Already on OperatorHub | None |
| Image pulling (NGC) | Standard K8s pull secrets | Low |
| Networking (HTTP) | Needs Gateway API controller or Route rewrite | Medium |
| Networking (gRPC/NATS) | No Route equivalent for TCP | Medium-High |
| Container Cache DaemonSet | Needs privileged SCC + CoreOS workaround | High |
| OLM operator wrapper | Not needed for PoC | Future |
