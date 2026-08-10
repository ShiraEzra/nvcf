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

### Phase 2: Core NVCF services (11 active, 4 disabled)

These ARE NVCF -- the API, invocation routing, key management, etc. They deploy as one group once Phase 1 is healthy. All are stateless deployments (no PVs needed). Deployed via: `helmfile sync --selector release-group=services`

Helmfile layer: `02-core.yaml.gotmpl`. Contains 15 releases total. 4 are disabled by conditions in the OCP environment. The remaining 11 deploy across 5 namespaces (nvcf, api-keys, ess, sis, nats-system).

#### All 15 releases in the helmfile

| # | Release | Chart | Version | Namespace | Condition | Status in OCP |
|---|---------|-------|---------|-----------|-----------|---------------|
| 1 | api-keys | helm-nvcf-api-keys | 1.5.1 | api-keys | (always) | Active |
| 2 | sis | helm-nvcf-sis | 1.17.0 | sis | (always) | Active |
| 3 | api | helm-nvcf-api | 1.23.6 | nvcf | (always) | Active |
| 4 | nvct-api | helm-nvcf-nvct-api | 1.4.2 | nvcf | (always) | Active |
| 5 | invocation-service | helm-nvcf-invocation-service | 1.5.4 | nvcf | (always) | Active |
| 6 | grpc-proxy | helm-nvcf-grpc-proxy | 1.6.7 | nvcf | (always) | Active |
| 7 | ratelimiter | helm-nvcf-rate-limiter | 1.0.3 | nvcf | rateLimiter.enabled | Disabled |
| 8 | ess-api | helm-nvcf-ess-api | 1.6.1 | ess | (always) | Active |
| 9 | notary-service | helm-nvcf-notary-service | 1.4.1 | nvcf | (always) | Active |
| 10 | admin-issuer-proxy | helm-admin-token-issuer-proxy | 1.4.3 | api-keys | (always) | Active |
| 11 | reval | helm-reval | 1.3.8 | nvcf | (always) | Active |
| 12 | nats-auth-callout-service | helm-nvcf-nats-auth-callout-service | 1.1.3 | nats-system | (always) | Active |
| 13 | llm-request-router | helm-nvcf-llm-request-router | 1.6.3 | nvcf | addons.llm.enabled | Disabled |
| 14 | llm-api-gateway | helm-nvcf-llm-api-gateway | 1.2.0 | nvcf | addons.llm.enabled | Disabled |
| 15 | vanity-gateway | helm-nvcf-vanity-gateway | 0.1.0-nvcf-10204.1 | nvcf | addons.vanityGateway.enabled | Disabled |

Disabled releases (set in environments/ocp-nvcf.yaml): ratelimiter (not needed for PoC), llm-request-router and llm-api-gateway (LLM routing addon, not needed), vanity-gateway (custom domain routing, not needed).

#### Chart source availability

6 of the 11 active charts have source code in this monorepo (under deploy/helm/). These can be audited and patched locally:
- api-keys, invocation-service, grpc-proxy, admin-issuer-proxy, reval, nats-auth-callout-service

5 charts are upstream OCI-only (no source in this repo). They are pulled from the registry at install time and cannot be audited locally. Issues will be discovered at deploy time:
- sis, api, nvct-api, ess-api, notary-service

#### Pre-deployment OCP compatibility analysis

Analyzed all 6 local chart templates for OCP restricted-v2 SCC compatibility. Key findings:

**Charts with hardcoded UIDs (will fail restricted-v2 SCC on OCP):**

1. api-keys: `runAsUser: 1000` hardcoded directly in the deployment template (line 90). Not overridable via Helm values. Must patch the template to make it values-driven, then repackage the chart.

2. grpc-proxy: `runAsUser: 1000` in values.yaml. Overridable via helmfile inline values. Also has `runAsNonRoot` and `readOnlyRootFilesystem` commented out in defaults.

3. admin-issuer-proxy: `runAsUser: 1000, fsGroup: 1000` in values.yaml. Best-hardened chart overall (has allowPrivilegeEscalation: false, readOnlyRootFilesystem: true, seccompProfile: RuntimeDefault). UIDs are overridable via helmfile inline values.

4. reval: `runAsUser: 65532` set in global.yaml.gotmpl (not in the chart itself). UID 65532 is the distroless nonroot user. Still a hardcoded UID that OCP may reject. Needs override.

**Charts with no security context at all:**
- invocation-service: zero securityContext in template or values. OCP will inject defaults from the SCC, which should work (restricted-v2 assigns a random UID, sets runAsNonRoot, drops capabilities). May or may not work depending on whether the image expects a specific UID.
- nats-auth-callout-service: same as invocation-service -- no securityContext at all.

**Other issues:**
- nats-auth-callout-service: has `ingress.enabled: true` with nginx IngressClassName by default. Will create an Ingress resource that fails on OCP (no nginx ingress controller). Must disable via helmfile values.
- invocation-service: has an Ingress template but disabled by default (`ingress.enabled: false`). No action needed.

**No issues expected:**
- No ConfigMap defaultMode issues (all use Kubernetes default 0644, unlike the 0500 problem in Phase 1)
- No hostNetwork/hostPID/privileged in any service chart
- No PVC/PV requirements (all stateless)
- No init containers or Jobs in local charts (api chart may have an accountBootstrap Job, but that's upstream)

**5 upstream charts (sis, api, nvct-api, ess-api, notary-service):** Cannot audit. No securityContext overrides are passed for any of them in global.yaml.gotmpl. If they have hardcoded UIDs in their templates, they will fail on OCP and need fixes at deploy time.

#### Deployment approach

Deploy-then-fix, iteratively. Apply known fixes first, then attempt deployment and fix whatever breaks from the upstream charts. OCP SCC violations surface immediately as pod admission failures.

Steps:
1. Patch api-keys chart template to make securityContext values-driven. Repackage and push to quay.io.
2. Add inline OCP values overrides to 02-core.yaml.gotmpl for grpc-proxy, admin-issuer-proxy, reval, and nats-auth-callout-service.
3. Run helmfile sync --selector release-group=services.
4. Diagnose and fix failures from upstream charts.
5. Verify all 11 services are running.

- Status: Done

#### OCP issues encountered and decisions

**1. SCC / runAsUser rejection (all services):**

- Problem: All NVCF service charts set `runAsUser: 1000` (or `65532` for reval and nats-auth-callout). OpenShift's restricted-v2 SCC only allows UIDs from the namespace's assigned range (e.g., 1000890000-1000899999 for the nvcf namespace). Every pod was rejected with "runAsUser: Invalid value: 1000: must be in the ranges [...]".
- This is a systemic issue across all NVCF charts. Every chart hardcodes `runAsUser: 1000` in its values.yaml. The api-keys chart went further: it hardcoded the value directly in the deployment template, not even in values.
- Options considered:
  a. Grant nonroot-v2 SCC to all NVCF service accounts -- allows any non-root UID, bypasses OCP's UID assignment. Quick but not OCP-native.
  b. Override securityContext via helmfile inline values for each release, setting `runAsUser: null` to remove the hardcoded UID and let OCP assign one. Keep `runAsNonRoot: true` and `capabilities.drop: ALL`.
  c. Patch each chart source to remove `runAsUser` from values.yaml defaults and repackage. Most work, cleanest result.
- Decision: Option b (helmfile inline values overrides). Added `runAsUser: null` overrides for all 10 services (invocation-service doesn't set runAsUser). For api-keys, also patched the chart template to make securityContext values-driven (it was hardcoded in the template), repackaged, and pushed to quay.io.
- Key learning: Helm deep-merges values. Setting `runAsNonRoot: true` alone does not remove the existing `runAsUser: 1000` from the chart defaults. Must explicitly set `runAsUser: null` to remove it.
- Services needing NET_BIND_SERVICE capability (sis, nvct-api, notary-service): kept the `add: [NET_BIND_SERVICE]` while removing runAsUser.
- Integration proposal note: NVCF charts should not hardcode `runAsUser: 1000`. Either remove it from defaults (let the platform assign UIDs) or add an `openshift` flag that removes it, similar to what the OpenBao chart already does. Single PR could fix all charts.

**2. Image tag mismatches (notary-service, nvct-api, sis):**

- Problem: Three services failed with ImagePullBackOff. The chart-specified image tags don't match the tags available on NGC or quay.io:
  - notary-service: chart expects 1.9.4, only 1.8.1 exists on NGC
  - nvct-service-oss (nvct-api): chart expects 1.5.5, only 1.5.9-hotfix.1 exists
  - spot (sis): chart expects 1.563.1, only 1.563.1-hotfix.1 exists
- Not OCP-specific -- the charts reference image tags that don't exist on the selfhosted-ga NGC registry. Likely built against internal NVIDIA CI artifacts.
- Decision: Re-tagged the available images on quay.io to match the expected tags. The images are close enough in version for a PoC.
- Integration proposal note: NVCF selfhosted chart versions and image tags are out of sync. The charts reference image versions that are not published to NGC. This needs to be fixed for any self-hosted deployment, not just OpenShift.

**3. OpenBao vault agent JWT authentication failure:**

- Problem: After re-enabling the OpenBao injector, every service pod starts with a vault-agent-init container that tries to authenticate to OpenBao using the pod's Kubernetes service account token. Authentication fails with "no known key successfully validated the token signature." The vault-agent retries with exponential backoff, keeping pods stuck in Init:0/1 forever.
- Root cause: OpenBao's JWT auth was configured during initialization with a single static public key. This OCP cluster has 4 signing keys (OCP rotates service account signing keys). Tokens signed by any of the other 3 keys are rejected.
- On vanilla K8s, the init script fetches the cluster's JWKS public key and configures OpenBao with it. On this OCP cluster, either the init script grabbed only one of the 4 keys, or OCP rotated keys after initialization.
- Options considered:
  a. Add all 4 static public keys to OpenBao's `jwt_validation_pubkeys` config via `bao write auth/jwt/config`. Quick fix, but keys rotate over time and would need manual updates after each rotation. Not sustainable.
  b. Configure OIDC discovery URL: set `oidc_discovery_url=https://kubernetes.default.svc` so OpenBao auto-fetches signing keys from the K8s API. Auto-refreshes on rotation. Attempted this, but OpenBao server gets 403 Forbidden accessing the JWKS endpoint because OCP restricts anonymous access to `/openid/v1/jwks`. Would need a ClusterRoleBinding for the server SA. Works, but bypasses the NVCF-native mechanism.
  c. Enable NVCF's built-in `issuerDiscovery` migration: the OpenBao chart already has an `issuerDiscovery` feature (disabled by default) that configures JWT auth using OIDC discovery. The chart includes a pre-install RBAC hook that grants the migration service account access to `system:service-account-issuer-discovery` (a built-in K8s ClusterRole). On OCP, also need a ClusterRoleBinding for the OpenBao server SA so it can fetch JWKS at runtime.
  d. Switch from JWT auth to Kubernetes auth method: OpenBao has a native `auth/kubernetes` backend that validates tokens via the K8s TokenReview API -- no key management at all. Most robust, but NVCF charts are hardcoded to use `auth/jwt` path in vault annotations. Would require changing every service chart.
- Decision: Option c (enable issuerDiscovery). This is the NVCF-native mechanism for exactly this problem. It uses the chart's built-in OIDC discovery + migration logic, and only requires one additional ClusterRoleBinding for the OpenBao server SA (standard OCP pattern -- same thing cert-manager and RHOAI need). It handles key rotation automatically and documents a clear integration requirement.
- Steps taken:
  1. Created ClusterRoleBinding granting OpenBao server SA access to `system:service-account-issuer-discovery`.
  2. Set `issuerDiscovery.enabled: true` in ocp-nvcf.yaml.
  3. Re-ran OpenBao helm upgrade. The migration ran but fell back to the static public key -- the migration script tries anonymous JWKS fetch, which fails on OCP, and falls back to the mounted static key from the `cluster-jwt` secret.
  4. Manually configured OIDC discovery via `bao write auth/jwt/config oidc_discovery_url=https://kubernetes.default.svc oidc_discovery_ca_pem=<ca-cert>`. But OpenBao's OIDC client fetches JWKS as an anonymous HTTP client (no bearer token), so it still got 403 Forbidden on OCP.
  5. Created ClusterRoleBinding granting `system:unauthenticated` access to `system:service-account-issuer-discovery`. This exposes only the JWKS endpoint (public key material, not sensitive) to unauthenticated requests -- matching vanilla K8s behavior where this is the default.
  6. Vault authentication now works. Services pass Init and start successfully.
- Integration proposal note: On OCP, NVCF requires two RBAC changes for OpenBao JWT auth: (1) enable `issuerDiscovery` in the chart values, and (2) grant `system:unauthenticated` access to `system:service-account-issuer-discovery` so OpenBao can fetch JWKS keys. Alternatively, NVIDIA could modify the OpenBao OIDC discovery to use a service account token for JWKS fetching instead of anonymous access. On vanilla K8s, the OIDC endpoints are already accessible to unauthenticated users by default.

**4. Missing vault secrets (caused by issue 3):**

- Problem: Services like nats-auth-callout-service and ess-api crash with "failed to read /etc/secrets/secrets.json" because the vault-agent init container can't authenticate and therefore never writes the secrets file.
- Not a separate issue -- it's a consequence of the JWT authentication failure (issue 3). Once vault auth is fixed, the vault-agent will inject secrets and services will start.

**5. admin-issuer-proxy HTTPRoute hostname validation:**

- Problem: admin-issuer-proxy chart creates an HTTPRoute with hostname `api-keys.{{ .Values.global.domain }}`. Our OCP domain is `10.6.60.125:30162` (IP + NodePort), resulting in hostname `api-keys.10.6.60.125:30162` which is invalid per the Gateway API spec (hostnames cannot contain ports).
- Not OCP-specific -- the domain includes a port because we're using NodePort on bare-metal without a DNS name or LoadBalancer.
- Decision: Disabled the HTTPRoute for admin-issuer-proxy via `gateway.enabled: false` in the helmfile inline values. The admin-issuer-proxy is internal-only and doesn't need external routing for the PoC.
- Integration proposal note: For production, a proper DNS domain should be configured instead of IP:port. This would resolve the hostname validation issue for all HTTPRoutes.

**6. NATS JetStream replicas (resolved):**

- Problem: nvcf-api fails to start with `JetStreamApiException: replicas > 1 not supported in non-clustered mode`. The API creates JetStream streams with multiple replicas for durability, but NATS was running as a single node without clustering.
- Not OCP-specific -- caused by running NATS without clustering on our small cluster.
- Options considered:
  a. Override stream replicas to 1 via env var -- unknown property name (API is closed-source), attempt with `NVCF_NATS_STREAM_REPLICAS=1` didn't work due to helmfile value merging.
  b. Enable NATS clustering with 1 replica -- NATS gets stuck "Waiting for routing to be established" because it expects cluster peers that don't exist.
  c. Scale NATS to 3 replicas with clustering enabled -- same pattern as OpenBao (3 replicas across 2 nodes). Matches what NVCF expects.
- Decision: Option c (3 replicas). Created 2 additional PVs (1 on worker, 1 on control-plane), set `cluster.enabled: true, replicas: 3`.
- Additional issue after enabling clustering: API failed with `no suitable peers for placement`. The API requests streams with a placement tag `dc` (`NVCF_NATS_REGION_PLACEMENT_TAG: "dc"`) and NATS nodes have `server_tags: ["dc:ncp"]` in the chart defaults, but the running pods hadn't picked up the tag. A `kubectl rollout restart` of the NATS StatefulSet resolved it.

**7. Image tag mismatches (additional, resolved):**

- Two more images needed re-tagging on quay.io when deploying invocation-service and api-keys:
  - nvcf-invocation-service: chart expects 0.5.2, available 0.8.5
  - nvcf-api-keys-service: chart expects 1.2.14, available 1.5.0
- Same issue as finding #2 -- chart-specified image tags don't match what's published on NGC.

**8. Account bootstrap job failure (non-blocking):**

- Problem: The api chart includes a helm hook Job (`nvcf-api-account-bootstrap`) that creates the initial NVCF account. It fails with HTTP 400: "CONTAINER registry with hostname quay.io is not yet recognized." The API only recognizes nvcr.io as a container registry.
- Not OCP-specific -- caused by using quay.io instead of nvcr.io for container images.
- Non-blocking for the PoC. All 11 services run without it. Will need to be addressed when deploying functions (NVIDIA-1043).

**Final status (2026-08-05):**
All 11 Phase 2 core services running on OpenShift:

| # | Service | Namespace | Pods | Status |
|---|---------|-----------|------|--------|
| 1 | api-keys | api-keys | 2/2 | Running |
| 2 | admin-token-issuer-proxy | api-keys | 2/2 | Running |
| 3 | ess-api | ess | 2/2 | Running |
| 4 | nats-auth-callout-service | nats-system | 2/2 | Running |
| 5 | nvcf-api | nvcf | 2/2 | Running |
| 6 | invocation-service | nvcf | 2/2 | Running |
| 7 | grpc-proxy | nvcf | 2/2 | Running |
| 8 | notary-service | nvcf | 2/2 | Running |
| 9 | nvct-api | nvcf | 2/2 | Running |
| 10 | reval | nvcf | 1/1 | Running |
| 11 | spot-instance-service (sis) | sis | 2/2 | Running |

- Status: Done

### Phase 3: Gateway routes (1 chart)

HTTPRoute definitions that connect the OSSM Gateway to the core services. This is what makes NVCF accessible from outside the cluster. Deployed via: `helmfile sync --selector release-group=ingress`

OCP issues encountered and decisions:

**1. Domain format (resolved):**

- Problem: `global.domain` was set to `10.6.60.125:30162` (IP + NodePort). The gateway-routes chart uses this to build HTTPRoute hostnames like `api.10.6.60.125:30162`, which is invalid per the Gateway API spec (hostnames cannot contain ports).
- Decision: Changed domain to `10.6.60.125` (without port). The port is handled by the Gateway's NodePort listener, not the hostname. Clients access via `http://10.6.60.125:30162` with the appropriate Host header.

**2. gRPC TCPRoute CRD missing (resolved):**

- Problem: The gateway-routes chart includes a TCPRoute template for gRPC traffic, gated by `nvcfGatewayRoutes.routes.grpc.enabled` (defaults to `true`). Our OCP environment sets `ingress.gatewayApi.routes.grpc.enabled: false`, but `global.yaml.gotmpl` has a gap -- it passes through `grpcWorker.enabled` and `nats.enabled` but not `grpc.enabled`. The chart never sees our `false` and tries to render a TCPRoute, which fails because OCP doesn't have the TCPRoute CRD (experimental Gateway API, blocked by OCP Ingress Operator).
- Decision: Added inline value override in the ingress release to set `nvcfGatewayRoutes.routes.grpc.enabled: false`. Same pattern as the securityContext overrides.
- Integration proposal note: `global.yaml.gotmpl` should pass through `grpc.enabled` to the gateway-routes chart, same as it does for `grpcWorker` and `nats`.

**Result:**

7 HTTPRoutes created: nvcf-api, api-keys, invocation-service, sis, nvct-api, reval, llm-api-gateway.

API connectivity verified:
- `curl -H "Host: api.10.6.60.125" http://10.6.60.125:30162/health` returns `{"status":"UP"}`
- Authenticated endpoints return 401 (expected -- needs NVCF account setup)

- Status: Done

## Step 6: Register GPU cluster and install NVCA operator
- Platform: All platforms
- Status: Pending (depends on step 5)
