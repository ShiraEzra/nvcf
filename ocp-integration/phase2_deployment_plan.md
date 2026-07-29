# Phase 2: NVCF Deployment Plan for OpenShift

## Cluster State

- Cluster: nvsentinel (OpenShift 4.22, Kubernetes v1.35.5)
- Nodes: 2 (1 control-plane+worker, 1 worker), both RHCOS
- GPUs: 1x Tesla T4 per node (2 total), GPU Operator v26.3.3 installed
- Storage: `local-path` StorageClass (default)
- Container runtime: CRI-O 1.35
- Existing workloads: RHOAI 3.4, TinyLlama-1.1B-Chat serving in test-model-serving namespace
- Kubeconfig: ~/.kube/nvsentinel-ocp.kubeconfig

## What NVCF Deploys

NVCF self-hosted is a split-stack architecture: a control plane and a compute plane.

### Control Plane (9 namespaces)

Phase 1 -- Infrastructure dependencies:
- **NATS** (nats-system) -- message broker, StatefulSet, 3 replicas default
- **Cassandra** (cassandra-system) -- database, StatefulSet, 3 replicas default
- **OpenBao** (vault-system) -- secrets management (Vault-compatible), StatefulSet, 3 replicas default
- **cert-manager** (cert-manager) -- certificate lifecycle

Phase 2 -- Core services:
- **api-keys** (api-keys) -- API key management
- **admin-issuer-proxy** (api-keys) -- admin token issuance
- **sis** (sis) -- system integration service
- **ess-api** (ess) -- execution space service
- **nvcf-api** (nvcf) -- core API, depends on ess-api
- **invocation-service** (nvcf) -- request routing, depends on api
- **grpc-proxy** (nvcf) -- gRPC support, depends on api
- **nvct-api** (nvcf) -- task control, depends on api
- **notary-service** (nvcf) -- image signature verification
- **reval** (nvcf) -- re-evaluation/bootstrap
- **nats-auth-callout-service** (nats-system) -- NATS authentication

Phase 3 -- Ingress:
- **nvcf-gateway-routes** (envoy-gateway-system) -- HTTPRoute/TCPRoute definitions

Phase 4 -- Observability (optional):
- **state-metrics** (nvcf) -- Prometheus metrics

### Compute Plane (separate install after control plane)

- **nvca-operator** -- NVIDIA Cluster Agent operator
- **function-autoscaler** -- autoscaling controller

### Dependency Graph

```
cert-manager ─┐
NATS ─────────┤
              ├── Core Services ── Gateway Routes ── (Observability)
OpenBao ──────┤     (api, sis, ess, invocation, grpc, notary, etc.)
Cassandra ────┘
```

OpenBao depends on NATS for initialization. All other Phase 1 components are independent.

## OpenShift Adaptation Decisions

### 1. Networking: Gateway API vs Routes

NVCF uses Gateway API with Envoy Gateway. It requires:
- HTTPRoute for HTTP traffic (API, invocation, LLM gateway)
- TCPRoute for gRPC (port 10081) and NATS (port 4222)

**Decision: Install Envoy Gateway on OpenShift (Option A from Phase 1)**

Rationale:
- Smallest change to NVCF (zero modification to charts or routes)
- Gateway API is a Kubernetes standard, not a vendor lock-in
- OpenShift 4.22 supports Gateway API CRDs
- OpenShift Routes have no TCPRoute equivalent, so rewriting would require separate LoadBalancer/NodePort services for gRPC and NATS
- Envoy Gateway runs as a regular deployment, no special privileges needed

Cluster-specific networking facts:
- This is a bare-metal cluster (Dell R740xd servers in a Red Hat lab)
- No MetalLB or other LoadBalancer provisioner is installed
- OpenShift Router uses HostNetwork (ports 80/443 bound on nodes)
- Gateway API CRDs are already installed (HTTPRoute, GatewayClass, GRPCRoute) but TCPRoute is missing
- Node IPs: 10.6.60.125, 10.6.60.126

**Networking plan:**
1. Install experimental Gateway API CRDs (adds TCPRoute support)
2. Install Envoy Gateway as the Gateway API controller
3. Patch the Envoy Gateway service to use NodePort (since no LB provisioner exists)
4. Access NVCF via node IP + NodePort (e.g., 10.6.60.125:30080 for HTTP, :30081 for gRPC, :34222 for NATS)
5. Cannot use HostNetwork ports 80/443 since the OpenShift Router already occupies them

### 2. cert-manager

cert-manager is already running on the cluster (installed by RHOAI/OLM). Installing a second instance would conflict.

**Decision**: Skip NVCF's cert-manager chart. Set `certManager.enabled: false` (or equivalent) in the helmfile environment, or use `--selector` to exclude it during deployment. The existing cert-manager will handle certificate operations for NVCF.

### 3. Security Context Constraints

From Phase 1 research:
- Most NVCF services run as non-root (uid 1000) and fit the `restricted` SCC
- NVCA Operator already has OpenShift SCC awareness in its RBAC
- Container Cache DaemonSet needs `privileged` SCC (skip for PoC)
- NVCF Unbound (DNS) needs `SYS_RESOURCE` capability

**Decision for PoC**: Use the default `restricted` SCC. Create a `nonroot` or custom SCC only if specific services fail to schedule. Skip Container Cache entirely.

If OpenShift's restricted SCC blocks services (e.g., due to UID range enforcement via namespace annotations), we may need to:
- Create service accounts with `nonroot-v2` SCC binding
- Or use `anyuid` SCC for specific service accounts

This is a "try and fix" situation -- deploy, see what gets blocked, add the minimal SCC grant needed.

### 4. Storage

The cluster has `local-path` as the default StorageClass. This is fine for PoC but not HA-safe (no replication, no dynamic provisioning across nodes).

For Cassandra (3 replicas default), OpenBao (3 replicas default), and NATS (3 replicas default), we need to reduce replica counts to 1 each since we only have 2 nodes and local-path does not support multi-node access.

**Decision**: Set `storageClass: local-path` and reduce all StatefulSet replicas to 1. This is acceptable for PoC/demo tier per NVCF's own sizing guide.

### 5. Image Registry

NVCF images are on NGC (nvcr.io). We need:
- NGC API key with access to the NVCF image org
- docker-registry secrets in each NVCF namespace

Standard Kubernetes pull secret mechanism, compatible with OpenShift.

### 6. Node Selectors

With only 2 nodes, dedicated node pools are not feasible.

**Decision**: Disable node selectors (`nodeSelectors.enabled: false`). All workloads schedule wherever they fit. The existing TinyLlama deployment uses 1 GPU on one node; NVCF control plane services will spread across both nodes.

### 7. Cassandra Resource Preset

The cluster has modest resources. Use `large` preset (1 CPU, 2048Mi request / 1.5 CPU, 3072Mi limit) instead of default `xlarge`, since we only run 1 replica.

## How NVCF Installation Works

In a standard (non-OpenShift) environment, NVCF installation is a single command:

```bash
HELMFILE_ENV=my-env helmfile sync
```

You configure an environment file (registry, domain, secrets) and Helmfile reads
the deployment definitions and installs everything in the correct order:

1. Phase 1 -- Dependencies (NATS, Cassandra, OpenBao, cert-manager) -- deploy in parallel, 5-10 min
2. Phase 2 -- Core services (API, invocation, gRPC proxy, etc.) -- deploy with dependency ordering, 5-10 min
3. Phase 3 -- Gateway routes (HTTPRoute/TCPRoute definitions) -- 1-2 min
4. Phase 4 -- Observability (optional metrics) -- 1-2 min

Then separately, you register a GPU cluster and install the NVCA operator on it.

Container Cache is NOT part of any of these phases. It is listed under "Optional
Enhancements" in the NVCF docs (docs/user/optional-enhancements.md). It is a
separate Helm chart you install after everything else is running, only if you
want faster image pulls on GPU nodes. You can run NVCF indefinitely without it.

Skipping it does not mean we are modifying the installation or running a degraded
mode. It means we do the standard install and choose not to add an optional
optimization on top. The NVCF control plane, API, invocation, autoscaling,
scale-to-zero -- all of that works the same with or without Container Cache. The
only user-visible difference: when NVCF deploys a function for the first time on
a node, the container image pulls from the registry at normal speed instead of
being pre-cached. For TinyLlama (~2GB) that is maybe 30-60 seconds extra on
first deploy. For a 50GB production model the difference would be more
significant, but that is a production optimization, not a PoC concern.

## Installation Order

### Prerequisites (before NVCF)

1. GPU Operator -- done (v26.3.3)
2. cert-manager -- done (installed by RHOAI)
3. Gateway API CRDs -- partially done (HTTPRoute exists, need experimental CRDs for TCPRoute)
4. Install Envoy Gateway controller
5. Create Gateway resource with NodePort service type
6. Configure NVCF gateway routes to use NodePort endpoints
7. Get NGC API key and verify access to NVCF image org (nvcf-onprem or equivalent)

### NVCF Control Plane Install

Using the Helmfile installation path (not quickstart, which is k3d-only):

1. Prepare the environment file (environments/nvsentinel.yaml)
2. Prepare the secrets file (secrets/nvsentinel-secrets.yaml)
3. Create namespaces and image pull secrets
4. Deploy Phase 1: dependencies (NATS, Cassandra, OpenBao -- skip cert-manager, already installed)
5. Deploy Phase 2: core services
6. Deploy Phase 3: gateway routes
7. Verify control plane health

### NVCF Compute Plane Install

Since control plane and compute plane are on the same cluster:

1. Register the cluster with the control plane API
2. Install NVCA operator
3. Verify NVCFBackend health

### Validation

1. Deploy a test function (load_tester_supreme or similar)
2. Invoke it via the API
3. Deploy TinyLlama via NVCF and compare with the RHOAI deployment

## Required Tools

On the deployment machine:
- kubectl (v1.34.1, already installed)
- helm >= 3.12
- helmfile 1.1.x (avoid 1.2.x, use 1.3.0+ with --sequential-helmfiles)
- helm-diff plugin >= 3.11
- nvcf-cli (build from repo or download from NGC)
- NGC API key

## OpenShift-Specific Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| No cloud LB on bare-metal | Gateway cannot get external IP | Use MetalLB or NodePort |
| SCC blocks service startup | Pods stuck in CreateContainerError | Grant nonroot-v2 or anyuid SCC per service account |
| local-path storage limits | No HA, no dynamic cross-node provisioning | Accept for PoC, reduce replicas to 1 |
| CRI-O vs containerd differences | Container Cache DaemonSet incompatible | Skip Container Cache for PoC |
| UID range enforcement | OpenShift assigns UID ranges per namespace | May need to override with SCC |
| Resource pressure (2 nodes) | OOM or scheduling failures | Reduce replica counts and resource presets |
| cert-manager conflict | RHOAI already installed cert-manager | Skip NVCF's cert-manager chart (confirmed running) |

## Image Distribution Gap

All NVCF container images (~25+) are hosted on NVIDIA's private registry
(nvcr.io) and require an NGC API key tied to a specific NVIDIA organization.
This is not compatible with how OpenShift solutions are distributed.

In the OpenShift ecosystem, container images come from trusted, certified
sources:
- registry.redhat.io (Red Hat certified images, requires Red Hat subscription)
- quay.io (Red Hat's registry platform)
- OperatorHub (certified operators with validated images via OLM)

Pulling from a private NVIDIA registry with a special API key is acceptable for
a PoC but not for a production OpenShift solution. For comparison, RHOAI images
come from registry.redhat.io, the GPU Operator images come from certified
sources on OperatorHub -- users never need a separate vendor API key.

Only ~8 of the ~25+ NVCF services have Dockerfiles in the open-source repo.
The rest are custom NVIDIA builds of open-source software (e.g.,
bitnami-cassandra:5.0.6-nv-1, nvcf-openbao:2.5.4-nv-1.3.0) that cannot be
reproduced from the repo source alone. Building from source is not a practical
alternative.

**For the PoC:** We use NGC images to prove NVCF works on OpenShift. This
requires an NGC API key with access to the NVCF image org.

**For the integration proposal**, this is a gap that needs to be addressed.
Options for NVIDIA:
1. Publish NVCF images to quay.io or another publicly accessible registry
2. Get Red Hat container certification for NVCF images
3. Distribute NVCF as a certified operator through OperatorHub (with images
   hosted on a certified registry)
4. At minimum, make images available without requiring a private org membership

This is one of the key differences between "NVCF works on OCP" and "NVCF is an
OCP-native solution."

## Cluster Checks Completed

- No MetalLB or LB provisioner -- will use NodePort for Envoy Gateway
- cert-manager already running (skip NVCF's chart)
- Gateway API CRDs partially installed (need experimental CRDs for TCPRoute)
- OpenShift Router uses HostNetwork on ports 80/443 (cannot reuse those ports)
- Node IPs: 10.6.60.125, 10.6.60.126

## Immediate Next Steps

1. Get NGC API key and verify access to NVCF image registry
2. Install required CLI tools (helm, helmfile, helm-diff, nvcf-cli)
3. Install experimental Gateway API CRDs (adds TCPRoute)
4. Install Envoy Gateway and create the Gateway resource with NodePort
5. Prepare the NVCF helmfile environment file for nvsentinel
