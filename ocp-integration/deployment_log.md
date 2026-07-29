# NVCF on OpenShift - Deployment Log

Shira Alfasi, Ecosystem Engineering, Red Hat | July 2026

This document tracks each step during NVCF deployment on OpenShift, noting whether each step is required for all platforms or is OCP-specific.

## Step 1: Gateway API CRDs
- Platform: All platforms (NVCF prerequisite)
- OCP note: Standard CRDs already on cluster (v1.4.1, standard channel). Missing TCPRoute (experimental/alpha).
- OCP ISSUE: OpenShift Ingress Operator owns Gateway API CRDs and blocks installation of experimental CRDs (TCPRoute) via ValidatingAdmissionPolicy. Manually installing would cause Degraded operator status and block cluster upgrades.
- Investigation: Checked three alternatives:
  - NodePort for gRPC: works immediately, no chart or CRD changes, but bypasses OpenShift routing layer
  - GRPCRoute (supported in OCP 4.20+): operates at L7, but NVCF gRPC proxy expects L4 passthrough. Would need new chart template + Gateway listener changes. Too risky.
  - OpenShift passthrough Route: most OCP-native for gRPC, but requires TLS on the backend. NVCF gRPC proxy serves plaintext (no TLS support). Would need code/config change in NVCF.
- Decision: Use NodePort for gRPC (port 10081) for PoC. Disable gRPC TCPRoute in chart values (routes.grpc.enabled: false). NATS TCPRoute already disabled by default.
- Integration proposal note: NVCF gRPC proxy should add TLS support so it can work with OpenShift passthrough Routes. This is the proper OCP-native pattern for gRPC.
- Result: No experimental CRDs needed. Existing standard CRDs (HTTPRoute, GatewayClass, Gateway) are sufficient for HTTP traffic.
- Status: Done (no CRD install needed)

## Step 2: Install Gateway API controller
- Platform: All platforms (NVCF prerequisite). On vanilla K8s you install Envoy Gateway. On OpenShift we use OSSM.
- OCP ISSUE: Envoy Gateway v1.1.3 cannot run on OpenShift. It bundles Gateway API CRDs (blocked by Ingress Operator admission policy), its certgen job fails SCC, and it requires experimental CRDs (TLSRoute, TCPRoute) that don't exist on OCP. Controller crashes on startup.
- OCP SOLUTION: Fabien Dupont suggested using Red Hat OpenShift Service Mesh (OSSM) 3.4 instead. OSSM is Red Hat's supported Istio distribution, installed via OperatorHub. It provides the same Gateway API controller functionality without CRD conflicts.
- Installation steps (OCP-specific):
  1. Installed servicemeshoperator3 v3.4.0 via OLM (stable-3.4 channel)
  2. Created IstioCNI in istio-cni namespace (v1.30.1, 2 pods running)
  3. Created Istio control plane in istio-system namespace (istiod running, status: Healthy)
- Status: Done

## Step 3: Create Gateway resource
- Platform: All platforms (but configuration differs)
- OCP note: Using Istio GatewayClass instead of Envoy Gateway's. Service type is LoadBalancer but gets NodePort since no LB provisioner on bare-metal.
- Created Gateway in envoy-gateway namespace (kept namespace name for NVCF chart compatibility)
- Gateway has HTTP listener on port 80, exposed as NodePort 30162
- Access: http://10.6.60.125:30162 (returns 404, expected -- no routes configured yet)
- NVCF namespaces created and labeled: nvcf, api-keys, ess, sis, nats-system, vault-system, cassandra-system
- Status: Done

## Step 4: Obtain NGC API key and create pull secrets
- Platform: All platforms (NVCF prerequisite)
- NGC account created: salfasi@redhat.com
- Got access to nvcf-onprem org and selfhosted-ga team
- Container images found across two NGC paths:
  - nvcr.io/nvidia/nvcf/ -- core services (20 images)
  - nvcr.io/0833294136851237/selfhosted-ga/ -- infrastructure images (7 images)
- Helm charts found on traditional Helm repo (not OCI):
  - https://helm.ngc.nvidia.com/nvidia/nvcf -- all 16 charts
  - OCP note: the helmfile expects OCI registry for charts, but NGC publishes them on a traditional https Helm repo. Had to mirror to quay.io as OCI packages.
- All 27 container images and 16 Helm charts mirrored to quay.io/salfasi-redhat
- Environment file created: environments/ocp-nvcf.yaml (registry: quay.io, repository: salfasi-redhat)
- Secrets file created: secrets/ocp-nvcf-secrets.yaml
- Pull secrets (quay-pull-secret) created in all NVCF namespaces
- 4 chart versions updated in helmfile to match available versions (nats 0.7.1, api 1.23.6, grpc-proxy 1.6.7, gateway-routes 1.14.0)
- Status: Done

### Note on distribution complexity
NVCF was open-sourced in April 2026 (~3 months ago). The self-hosted distribution is still fragmented across three channels:
- GitHub repo: source code + Helm charts for open-source services only
- OCI registry (nvcr.io): container images + charts, gated by NGC org access
- Traditional Helm repo (helm.ngc.nvidia.com): charts with different access controls than OCI

5 core services (api, sis, ess-api, notary-service, nvct-api) remain closed source -- no source code, no local Helm charts, only available as pre-built artifacts on NGC. GitHub issue #12 tracks the request to open-source these. This is a key gap for OpenShift productization, where Red Hat would need to rebuild all images from source with UBI base.

## Step 5: Deploy NVCF control plane (helmfile sync)

On vanilla K8s, this entire step is a single command: `helmfile sync`. It deploys all three phases in order automatically. On OpenShift, we are deploying phase by phase because each service hits OCP-specific issues (SCCs, storage, image tags) that need fixing along the way.

OCP-specific values set in our environment file: certManager disabled (already installed by RHOAI), gRPC TCPRoute disabled (OCP doesn't support TCPRoute), OSSM gateway references (instead of Envoy Gateway). Replica counts adjusted per service based on requirements (NATS: 1, OpenBao: 3, Cassandra: 1).

### Phase 1: Infrastructure dependencies (3 services)

These are the foundation that NVCF core services depend on. All three must be running before Phase 2 can start.

**NATS** -- Message broker. All NVCF services communicate through NATS (event-driven architecture). It is the nervous system of NVCF. Without it, nothing else can start.
- OCP issues encountered:
  1. Image tag mismatch: selfhosted-ga images had "-ea" suffix (e.g., 0.23.0-ea) but charts expect tags without suffix (0.23.0). Fix: re-tagged and pushed images.
  2. Replicas: chart defaults to 3 replicas. Env file override didn't reach the subchart. Fix: inline values override in helmfile.
  3. PersistentVolumes: local-path requires manual PV creation. Old PVs stuck in "Failed" after cleanup. Fix: recreated PVs.
  4. Helm timeout: timed out waiting but pods were running. Fix: re-ran sync.
- Status: Done

**OpenBao** -- Secrets vault (Vault-compatible). Stores database passwords, API keys, registry credentials. Services authenticate to OpenBao to get their secrets at startup. Without it, services can't access their configuration. After initialization, a migrations job runs to set up schemas, authentication methods, and write the initial secrets (Cassandra password, API credentials, registry pull secrets).

OCP issues encountered and decisions:

1. SCC / UID mismatch:
   - Problem: OpenBao runs as UID 100 with fsGroup 1000. OpenShift assigns UID range 1000940000+ per namespace and blocks pods running outside this range.
   - Options considered:
     a. Grant anyuid SCC to OpenBao service accounts -- opens security permissions, against OCP principles.
     b. Override uid/gid in chart values to use OCP-compatible UIDs -- chart supports this, but the global template doesn't pass these values through.
     c. Set openshift: true in the chart -- the chart has a built-in OpenShift flag that removes the hardcoded securityContext entirely, letting OpenShift assign UIDs automatically.
   - Decision: Option c (openshift: true). Most OCP-native -- no SCC changes, OpenShift controls UID assignment. Added as inline values override in helmfile because the global template doesn't pass this through. This is a small fix NVIDIA could make: wire the openshift flag through the global template.

2. ConfigMap file permissions:
   - Problem: The init job mounts scripts from ConfigMaps with defaultMode: 0500 (owner-only execute). The file owner is root (UID 0), but the container runs as an OpenShift-assigned UID. The container can't execute the script because it's not the owner.
   - Options considered:
     a. Grant anyuid SCC so the container runs as the expected UID -- opens permissions unnecessarily.
     b. Patch the chart template to change defaultMode from 0500 to 0555 (world-executable) -- correct fix, script is not sensitive.
     c. Use a helmfile post-renderer to patch the rendered manifest -- more complex, no chart changes.
     d. Change the command from execute to source -- also requires chart change, may have side effects.
   - Decision: Option b (patch defaultMode to 0555). One-character change in the chart template, no security implications (the script is a deployment helper, not a secret). Re-packaged the patched chart and pushed to quay.io. Good PR candidate for NVIDIA.

3. Webhook circular dependency:
   - Problem: OpenBao's agent injector deploys a MutatingAdmissionWebhook that intercepts all pod creation in the namespace. The init and migration jobs need to create pods, but the webhook times out trying to communicate with the not-yet-initialized vault. Circular dependency: vault can't initialize because the webhook blocks the init job, and the webhook can't work because vault isn't initialized.
   - This is not OCP-specific -- it would happen on any platform where the webhook is slow or the vault initialization takes time.
   - Decision: Disable the injector during initial deployment (injector.enabled: false in helmfile values). The injector can be re-enabled after initialization if needed for runtime secret injection.

4. Replica count:
   - Problem: We initially tried 1 replica (to match our 2-node cluster), then 2. Both failed because the init script hardcodes waiting for pods 0, 1, and 2 (expects exactly 3 replicas).
   - Decision: Use 3 replicas (the default). Created 3 PVs across both nodes (2 on worker, 1 on control-plane). The init script doesn't support fewer than 3 replicas -- this is a limitation worth reporting to NVIDIA.

- Final configuration: openshift: true, injector disabled, 3 replicas, 3 PVs, patched defaultMode 0555
- Status: Done (3/3 pods running, init job completed, migrations completed)

**Cassandra** -- Database. Stores function definitions, deployment state, API keys, invocation history. All NVCF state lives here.

OCP issues encountered and decisions:

1. Privileged init container (dynamic-seed-discovery):
   - Problem: The NVCF wrapper chart adds a dynamic-seed-discovery init container that runs as root (UID 0) with privileged: true. It installs system packages (netcat, dnsutils) at startup to do DNS lookups for other Cassandra pods. OpenShift blocks root + privileged by default.
   - Note: The Bitnami subchart has a built-in OpenShift compatibility flag (adaptSecurityContext: auto) but it only applies to Bitnami's own containers, not to the init container added by NVCF's wrapper.
   - Decision: Disabled seed discovery entirely (dynamicSeedDiscovery.enabled: false). With 1 replica there are no other pods to discover, so the init container has nothing useful to do.
   - Also set podSecurityContext.enabled: false, containerSecurityContext.enabled: false, and adaptSecurityContext: force to remove all hardcoded UIDs and let OpenShift assign them.
   - No SCC permissions changed -- fully OCP-native.
   - Integration proposal note: For multi-replica deployments, seed discovery would need to be fixed -- either pre-bake netcat/dnsutils into the image (so no package install at runtime) or rewrite the script to run as non-root. This would be a PR proposal for NVIDIA.

2. ConfigMap file permissions (same issue as OpenBao):
   - Problem: Init-cluster job mounts script from ConfigMap with defaultMode: 0500 (owner-only execute). Container runs as OpenShift-assigned UID, can't execute root-owned file.
   - Decision: Same fix as OpenBao -- patched defaultMode from 0500 to 0555. Repackaged chart and pushed to quay.io. This is a systemic issue in NVCF charts (both OpenBao and Cassandra have it). Single PR to NVIDIA could fix all charts.

3. Helmfile values nesting:
   - Problem: Getting values overrides to reach the Bitnami subchart through the NVCF wrapper required the correct YAML nesting. Several attempts needed because the template rendered correctly but the deployed chart used different values.
   - Not OCP-specific -- helm subchart value plumbing.

- Final configuration: seed discovery disabled, all security contexts disabled (OCP assigns UIDs), adaptSecurityContext: force, patched defaultMode 0555, 1 replica
- Status: Done (1/1 pod running, init-cluster completed, migrations completed)

### Phase 2: Core NVCF services (~12 services)

These ARE NVCF -- the API, invocation routing, key management, etc. They deploy as one group once Phase 1 is healthy. Mostly stateless deployments (no PVs), so fewer OCP issues expected.

Services include: nvcf-api (core API), api-keys (key management), sis (system integration), ess-api (execution space), invocation-service (request routing), grpc-proxy (gRPC support), notary-service (image verification), reval (re-evaluation), nvct-api (task control), nats-auth-callout (NATS authentication), admin-issuer-proxy (admin auth).

- Status: Pending (depends on Phase 1)

### Phase 3: Gateway routes (1 chart)

HTTPRoute definitions that connect the OSSM Gateway to the core services. This is what makes NVCF accessible from outside the cluster.

- Status: Pending (depends on Phase 2)

## Step 6: Register GPU cluster and install NVCA operator
- Platform: All platforms
- Status: Pending (depends on step 5)
