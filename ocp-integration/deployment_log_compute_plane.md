# NVCF on OpenShift - Compute Plane Deployment Log

Shira Alfasi, Ecosystem Engineering, Red Hat | August 2026

Jira: NVIDIA-1080

This document tracks each step during NVCF compute plane deployment on OpenShift, noting whether each step is required for all platforms or is OCP-specific. The compute plane is the GPU worker layer: the NVCA operator and agent run on GPU nodes, receive work from the control plane via NATS, and execute inference containers on GPUs.

Prerequisite: the NVCF control plane must be fully deployed and healthy. See deployment_log_control_plane.md for that process.

Cluster: nvsentinel-ocp (2 nodes, 2x Tesla T4 GPUs, OCP 4.18, GPU Operator v26.3.3)
Control plane API: http://10.6.60.125:30162 (NodePort via OSSM Gateway)

## Step 1: Fix account bootstrap

- Platform: All platforms (the bootstrap job is part of the standard NVCF deployment)
- Dependency: Required before any CLI or compute plane operations. The bootstrap creates the initial NVCF account that all API operations authenticate against.

The account bootstrap job (a post-install/post-upgrade Helm hook in the api chart) failed during control plane deployment. It was listed as non-blocking for the control plane (NVIDIA-1006) but is blocking for the compute plane because all CLI operations require a valid NVCF account.

**Problem:** The bootstrap job creates an NVCF account and registers container registry credentials with the API. Our secrets file configured `quay.io` as the registry. The NVCF API rejected it with HTTP 400: "CONTAINER registry with hostname quay.io is not yet recognized." The API has a hardcoded list of supported registries: NGC (nvcr.io), ECR, ACR, VolcEngine, JFrog, and Harbor. quay.io is not on this list.

This is not OCP-specific. It would happen on any deployment that uses a registry outside the supported list.

**Background -- why quay.io was configured in the first place:** During control plane deployment (NVIDIA-1006), we mirrored all NVCF container images from NGC to quay.io because the chart-specified image tags don't match what's published on NGC's selfhosted-ga registry (see deployment_log_control_plane.md, Phase 2, issues 2 and 7). We re-tagged images on quay.io to match what the charts expect. So quay.io is our infrastructure image registry -- where NVCF services (api, sis, invocation-service, etc.) run from, pulled via standard Kubernetes imagePullSecrets. That mechanism works fine regardless of the registry.

The bootstrap registry credentials are a separate concept. They tell the NVCF API which registries to use when pulling function container images (the inference workloads that users deploy, like TinyLlama). These are stored in the NVCF account and used by NVCA when launching function pods. The original secrets file incorrectly used quay.io for these credentials too, but function images will come from NGC (nvcr.io), not from our mirror.

**Investigation:** Confirmed the account was never created (not partially created). The bootstrap script authenticates to the API internally via OpenBao JWT, creates the account, then registers registry credentials. The script sends registryCredentials as part of the account creation payload. If the credentials include an unrecognized registry, the entire call fails and no account is created.

Verified by generating an admin token via nvcf-cli init (port-forwarding to admin-token-issuer-proxy since its HTTPRoute was disabled) and calling `nvcf-cli function list`. Got: "Unknown client_id 'ncp' when finding account id."

**Options considered:**
  a. Set registryCredentials to empty (`[]`) -- simplest, the script omits the field entirely when the array is empty. Add credentials later via CLI. Downside: no registry credentials registered for function images.
  b. Change registryCredentials to use `nvcr.io` with NGC credentials -- matches the standard secrets template, nvcr.io is a recognized registry, and function images will come from NGC anyway.
  c. Try adding quay.io to the API's recognized registries via env vars (e.g., `NVCF_REGISTRIES_RECOGNIZED_CONTAINER_QUAY_HOSTNAME`) -- speculative, the API backend is closed-source and might not support arbitrary registry env vars.
  d. Skip the bootstrap and create the account manually via internal API call -- complex, would need to replicate the OpenBao JWT auth flow that the bootstrap script uses.

**Decision:** Option b. Changed the secrets file to register `nvcr.io` (CONTAINER type) and `helm.ngc.nvidia.com` (HELM type) with NGC API key credentials, matching the standard NVCF secrets template format. No conflict with the existing setup: infrastructure images continue to come from quay.io (via K8s imagePullSecrets), function images will come from nvcr.io (via NVCF's registry credential system).

**Steps taken:**
1. Updated `secrets/ocp-nvcf-secrets.yaml`: replaced quay.io entries with nvcr.io and helm.ngc.nvidia.com using base64-encoded NGC Docker credential (`$oauthtoken:<NGC_API_KEY>`).
2. Ran `helmfile sync --selector name=api` to upgrade the api release. The post-upgrade hook triggered the bootstrap job with the corrected secret.
3. Bootstrap job completed successfully in 7 seconds.
4. Verified: `nvcf-cli function list` returns "No functions found" (correct, no functions deployed yet).

**Integration proposal note:** NVCF should add quay.io (and ghcr.io, Docker Hub) to the list of recognized container registries. These are widely used in the OpenShift ecosystem. Alternatively, provide an API configuration mechanism to add custom registries without modifying the backend code.

- Status: Done

## Step 2: Configure nvcf-cli

- Platform: All platforms (CLI configuration for self-hosted deployment)
- OCP note: Two issues specific to our OCP deployment affect CLI setup.

**nvcf-cli configuration:**

Created `~/.nvcf-cli.yaml` pointing at the control plane gateway:
- `base_http_url`: http://10.6.60.125:30162
- Host header overrides: `api.10.6.60.125`, `api-keys.10.6.60.125`, `invocation.10.6.60.125`
- `client_id`: ncp (matches the ADMIN_CLIENT_ID configured in secrets)
- `icms_url`: http://10.6.60.125:30162 (SIS endpoint for cluster management)

**OCP ISSUE: admin-token-issuer-proxy not routable through the gateway.**

The `nvcf-cli init` command generates an admin JWT token by calling `POST /v1/admin/keys` on the API Keys service. In the standard deployment, this goes to the admin-token-issuer-proxy via its own HTTPRoute. On our OCP deployment, the admin-issuer-proxy HTTPRoute was disabled (hostname validation fails with IP:port, see control plane deployment log, Phase 2, issue 5).

The api-keys HTTPRoute matches `PathPrefix: /` for hostname `api-keys.10.6.60.125`, which routes all requests to the api-keys service. The `/v1/admin/keys` path should go to admin-token-issuer-proxy instead, but there is no path-based routing for it.

Workaround: used `oc port-forward -n api-keys svc/admin-token-issuer-proxy 18080:8080` and set `API_KEYS_ADMIN_SERVICE_URL=http://localhost:18080` when running `nvcf-cli init`. Token generated successfully, saved to `~/.nvcf-cli.state`.

For production, this should be fixed by either:
- Configuring a proper DNS domain (eliminates the IP:port hostname issue)
- Adding a path-based HTTPRoute rule that routes `/v1/admin/*` to admin-token-issuer-proxy

**Token details:** Admin JWT with scopes including `register_function`, `deploy_function`, `cluster-management`. Expires after 12 hours. Subject: `ncp`.

- Status: Done

## Step 3: Install prerequisites

### SMB CSI driver

- Platform: All platforms (NVCF prerequisite for compute plane)
- OCP note: The node DaemonSet requires `privileged: true` for its containers. On OCP, this requires the privileged SCC. The kube-system namespace has permissive defaults, so no manual SCC grant was needed. Same pattern as the GPU Operator's device plugin DaemonSet.

Installed via Helm:
```
helm install csi-driver-smb csi-driver-smb/csi-driver-smb \
  --namespace kube-system --version v1.17.0 \
  --set windows.enabled=false --set linux.enabled=true
```

Result: controller (1 pod, 4/4 containers) + node DaemonSet (2 pods, 3/3 containers per node) all running. CSI driver registered as `smb.csi.k8s.io`.

No issues encountered.

- Status: Done

### inotify limits

- Platform: All platforms (NVCF prerequisite for compute plane)
- OCP note: On OCP CoreOS nodes, sysctl changes should use MachineConfig or TunedProfile (OCP-native), not a privileged DaemonSet (the approach in NVCF docs).

The NVCA operator and agent use Linux inotify file watchers to monitor Kubernetes manifests, model cache directories, container state, and GPU device files. The kernel limits how many watches a single user can create.

Current values on the cluster:
- `fs.inotify.max_user_instances`: 8192 (matches NVCF recommendation)
- `fs.inotify.max_user_watches`: 65,536 (NVCF recommends 524,288)

The `max_user_instances` value is already at the recommended level (likely set by the GPU Operator). The `max_user_watches` is lower than recommended but should be sufficient for a 2-GPU PoC with limited concurrent functions. The 524,288 recommendation is for production clusters running many concurrent function deployments where each deployment adds more watches.

**Decision:** Skip for PoC. The current values are likely sufficient for 2 GPUs. If NVCA fails with "no space left on device" errors when adding inotify watches, increase `max_user_watches` via a TunedProfile (the OCP-native approach).

**Production note:** For production deployments on OCP, increase `max_user_watches` to 524,288 using one of these OCP-native methods:
- TunedProfile (preferred, no reboot): create a Tuned CR in the openshift-cluster-node-tuning-operator namespace with `[sysctl] fs.inotify.max_user_watches=524288`.
- MachineConfig (persistent across reboots): write a sysctl.d conf file via MachineConfig, but this triggers a rolling node reboot.
- Do NOT use a privileged DaemonSet as suggested in the NVCF docs. While it works on vanilla K8s, it is not the OCP-native pattern and would need to be re-applied on every pod restart.

- Status: Skipped for PoC (documented for production)

### GPU Operator

- Platform: All platforms (NVCF prerequisite)
- Already installed: NVIDIA GPU Operator v26.3.3 via OLM (nvidia-gpu-operator namespace)
- 2x Tesla T4 GPUs available (1 per node)
- Device plugin DaemonSet running on both nodes
- Status: Already installed, no action needed

## Step 4: Download compute-plane-stack and configure environment

- Platform: All platforms (standard NVCF distribution)
- OCP note: Same image mirroring requirement as the control plane. The compute-plane-stack Helm chart and container images are on NGC (gated) and need to be mirrored to quay.io.

The compute-plane-stack is a separate Helmfile bundle distributed via NGC. It contains a Makefile, environment templates, and a helmfile that installs the NVCA operator chart. Unlike the control plane (which has 15+ releases across 3 phases), the compute plane is a single chart installation.

### NGC CLI installation

Installed NGC CLI v4.34.10 to download the bundle. The CLI is a Python-based binary that requires its full directory structure (not just the executable).

### Bundle download and extraction

Downloaded `nvcf-compute-plane-stack:1.0.6` from `nvidia/nvcf` on NGC. Extracted to `/home/salfasi/dev/NVCF/nvcf-compute-plane-stack/`.

Bundle contents:
```
nvcf-compute-plane-stack/
  helmfile.d/
    01-dependencies.yaml.gotmpl   # Addon operators (Grove, Dynamo) -- disabled
    02-nvca.yaml.gotmpl            # NVCA operator chart installation
  environments/
    base.yaml                      # Default values template
    ocp-nvcf.yaml                  # Our OCP environment (created below)
  global.yaml.gotmpl               # Go template for per-chart values
  registration/                    # Cluster registration output
  Makefile                         # register-cluster, install, destroy targets
```

### Image and chart mirroring

The compute-plane-stack helmfile references chart `helm-nvca-operator:1.12.7` (appVersion 3.0.3). Same distribution issue as the control plane: the chart is published on the traditional Helm repo (`helm.ngc.nvidia.com/nvidia/nvcf`), not the OCI registry the helmfile expects. Mirrored to quay.io as OCI.

Container images mirrored from NGC to quay.io:

| Image | Source | Tag | Notes |
|-------|--------|-----|-------|
| nvca-operator | nvcr.io/nvidia/nvcf | 3.0.3 | The operator deployment |
| nvca | nvcr.io/nvidia/nvcf | 3.0.3 | The agent (created by operator) |
| nvcf-image-credential-helper | nvcr.io/nvidia/nvcf | 0.10.2 | Registry auth sidecar |
| samba | nvcr.io/nvidia/nvcf-byoc | 1.0.4 -> 1.0.5 | Shared model cache sidecar. Only 1.0.4 on NGC; chart expects 1.0.5. Re-tagged during mirror. |

The samba image is under a different NGC path (`nvcf-byoc` instead of `nvcf`) and the available version (1.0.4) doesn't match the chart default (1.0.5). Same version mismatch pattern as control plane images.

### OCP environment configuration

Created `environments/ocp-nvcf.yaml` with:
- Image and chart registry: `quay.io/salfasi-redhat`
- Pull secrets: `quay-pull-secret`
- In-cluster control plane endpoints (since this is a single-cluster deployment):
  - ICMS: `http://api.sis.svc.cluster.local:8080`
  - ReVal: `http://reval.nvcf.svc.cluster.local:8080`
  - NATS: `nats://nats.nats-system.svc.cluster.local:4222`

No host header overrides needed for in-cluster endpoints -- they use direct service DNS, not the gateway.

- Status: Done

## Step 5: Register GPU cluster

- Platform: All platforms (cluster registration with ICMS/SIS)
- OCP note: OIDC identity source auto-detected as `psat` (projected service account token) from the OCP API server. No SPIRE on this cluster.

Cluster registration tells the control plane about this GPU cluster. The CLI fetches the cluster's OIDC configuration and JWKS public keys, then sends them to the ICMS (SIS) service. ICMS stores the cluster identity so it can verify the NVCA agent's service account tokens at runtime.

**OCP ISSUE: CLI sends cluster registration to wrong service via gateway.**

The Makefile's `register-cluster` target calls `nvcf-cli init` (to generate an admin token) followed by `nvcf-cli cluster register`. Both failed on our OCP deployment:

1. `nvcf-cli init` fails because admin-token-issuer-proxy is not routable through the gateway (same issue as Step 2). Workaround: port-forward and set `API_KEYS_ADMIN_SERVICE_URL`.

2. `nvcf-cli cluster register --icms-url http://10.6.60.125:30162` sends `POST /v1/accounts/ncp/clusters` to the gateway with `Host: api.10.6.60.125`. This routes to the NVCF API service, which returns 404 because the cluster registration endpoint is handled by the SIS service (routed via `Host: sis.10.6.60.125`). The CLI has no separate host header config for ICMS requests -- it always uses the `api_host` override.

Workaround: bypassed the gateway entirely by port-forwarding to SIS:
```
oc port-forward -n sis svc/api 18081:8080 &
nvcf-cli cluster register --name nvsentinel --nca-id ncp --icms-url http://localhost:18081 ...
```

This is the same class of issue as the admin-issuer-proxy: when the gateway routes by hostname and the CLI only has a single host header override, services that need different hostnames must be accessed via port-forward.

**Registration result:**
- Cluster ID: `13dd42f8-6e5e-4fe1-aa9c-b07911a021b6`
- Cluster Group ID: `3be2c8f6-4579-488c-9a54-61dbb612e851`
- OIDC Issuer: `https://kubernetes.default.svc`
- Identity source: psat
- Region: us-west-1

Saved registration values to `registration/nvsentinel-register-values.yaml` with in-cluster service URLs (replacing the localhost port-forward URLs from the CLI output).

**Integration proposal note:** The nvcf-cli should support separate host header overrides for each service (api, api-keys, sis, reval, invocation) or support a SIS-specific host header for ICMS operations. Currently a single `api_host` is used for all requests, which breaks when services are routed by hostname through a shared gateway.

- Status: Done

## Step 6: Install NVCA operator

- Platform: All platforms (Helm chart installation via helmfile)
- OCP note: Two SCC-related findings, one positive and one requiring manual fix.

Installed via the compute-plane-stack Makefile:
```
make install CLUSTER_NAME=nvsentinel NCA_ID=ncp HELMFILE_ENV=ocp-nvcf \
  KUBECONFIG_FILE=~/.kube/nvsentinel-ocp.kubeconfig
```

Created the `nvca-operator` namespace and `quay-pull-secret` image pull secret before installation.

### OCP finding 1: Operator pod accepted by nonroot SCC (positive)

The NVCA operator chart hardcodes `runAsUser: 1000, runAsGroup: 1010, fsGroup: 1010` in its deployment template. Unlike the control plane services (which had no SCC awareness and all required `runAsUser: null` overrides), the NVCA operator chart includes RBAC rules that grant `use` permission on the `nonroot` SecurityContextConstraint:

```yaml
- apiGroups: ["security.openshift.io"]
  resources: ["securitycontextconstraints"]
  resourceNames: ["nonroot"]
  verbs: ["use"]
```

This is rendered unconditionally (not behind an OpenShift flag), so on OCP the operator's service account can use the `nonroot` SCC, which allows any non-root UID. UID 1000 is accepted without any chart modifications or value overrides.

This is the correct OCP pattern. The operator pod started without SCC issues.

### OCP ISSUE 2: Agent pod rejected by SCC

The NVCA operator creates the NVCA agent as a Deployment in the `nvca-system` namespace (a separate namespace from the operator). The agent deployment also hardcodes `runAsUser: 1000`. However, the chart's SCC RBAC only grants `nonroot` SCC to the operator's own service account in the `nvca-operator` namespace. The agent's service account (`nvca` in `nvca-system`) has no SCC grants.

The agent pod was rejected by OCP admission:
```
restricted-v2: .containers[0].runAsUser: Invalid value: 1000:
must be in the ranges: [1000970000, 1000979999]
```

**Options considered:**
  a. Grant `nonroot` SCC to the agent service account -- matches the pattern the operator already uses for itself.
  b. Grant `nonroot-v2` SCC -- the modern version of nonroot on OCP 4.x, with additional seccomp profile support.
  c. Patch the operator to remove hardcoded UIDs from the agent deployment it creates -- would require modifying operator code, not just chart values.

**Decision:** Option b (`nonroot-v2`). The agent pod sets `seccompProfile: RuntimeDefault` and legacy seccomp annotations, which `nonroot-v2` handles correctly. The legacy `nonroot` SCC may reject pods with seccomp annotations.

```
oc adm policy add-scc-to-user nonroot-v2 system:serviceaccount:nvca-system:nvca
```

After granting the SCC and restarting the deployment, the agent pod started successfully (2/2 containers running).

**Integration proposal note:** The NVCA operator chart should extend its SCC RBAC to also cover the agent service account in `nvca-system`. Currently the chart grants `nonroot` SCC only to the operator's own namespace. Since the operator creates resources in `nvca-system`, the chart should include a ClusterRoleBinding granting `nonroot` (or `nonroot-v2`) to `system:serviceaccount:nvca-system:nvca`. This would make the compute plane deployment work on OCP without manual SCC grants.

- Status: Done

## Step 7: Verify NVCA agent connectivity

- Platform: All platforms
- OCP note: No OCP-specific issues at this stage.

**Operator status:**
- `nvca-operator` namespace: 1 pod (2/2 containers), Running
- NVCFBackend custom resource: created, version 3.0.3

**Agent status:**
- `nvca-system` namespace: 1 pod (2/2 containers: agent + webhook), Running
- NVCFBackend health: **healthy**

**Operator logs confirm successful reconciliation:**
```
Event: "Starting ClusterAgent 3.0.3 installation"
Event: "ClusterAgent 3.0.3 installation completed"
Event: "ClusterAgent health changed from '' to 'healthy'"
```

**Agent connectivity:**
- Connected to NATS (`nats://nats.nats-system.svc.cluster.local:4222`)
- Polling for work on JetStream queues (CreateNvcaFunctionTaskStream, TerminateNvcaStream)
- JetStream "stream not found" errors are expected -- streams are created when functions are deployed via the NVCF API

The agent is ready to receive function deployment requests from the control plane.

- Status: Done

## Final status

All 7 acceptance criteria for NVIDIA-1080 are complete. The compute plane is deployed and healthy on OpenShift.

**Pod summary:**

| Namespace | Pod | Containers | Status |
|-----------|-----|------------|--------|
| nvca-operator | nvca-operator | 2/2 | Running |
| nvca-system | nvca | 2/2 | Running |

**OCP issues encountered:**

| # | Issue | OCP-specific | Resolution | PR candidate |
|---|-------|-------------|------------|--------------|
| 1 | quay.io not recognized by NVCF API as registry | No | Use nvcr.io for function images, quay.io for infrastructure | Add quay.io to recognized registries |
| 2 | admin-issuer-proxy not routable through gateway | Partially (IP:port hostname) | Port-forward workaround | Separate host header per service in CLI |
| 3 | CLI sends ICMS requests with wrong Host header | Partially (hostname-based routing) | Port-forward to SIS directly | Add sis_host config to CLI |
| 4 | NVCA agent pod rejected by SCC | Yes | Grant nonroot-v2 SCC to agent SA | Extend chart SCC RBAC to agent namespace |

Issues 2 and 3 are related to the same root cause: hostname-based gateway routing with IP:port instead of DNS. A proper DNS domain would resolve both.

**Next task:** NVIDIA-1043 -- Deploy TinyLlama via NVCF to validate end-to-end function deployment on the compute plane.
