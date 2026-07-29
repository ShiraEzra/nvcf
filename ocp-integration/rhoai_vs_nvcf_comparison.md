# RHOAI Model Serving vs NVCF: Comparison

## The Short Version

Both platforms let you deploy AI models without deep Kubernetes knowledge. But they sit at different layers:

- **RHOAI** = the data science platform. It covers the whole journey: you develop in a notebook, train with pipelines, register your model, serve it, monitor it with guardrails.
- **NVCF** = the GPU runtime optimizer. It doesn't care how you trained the model. It takes a container and makes the GPU infrastructure work smarter — scale-to-zero, queue requests, cache containers, manage multiple clusters.

Think of it like: RHOAI is the "what" (what model, what data, what workflow), NVCF is the "how" (how to run it efficiently on GPUs).

The overlaps are real — both can deploy a model and give you an API endpoint. Both support OpenAI-compatible APIs. Both have auth. Both autoscale. So on the surface they look like competitors.

But the gaps tell the real story:

**NVCF does things RHOAI doesn't (or does better):**
- Scale-to-zero — managed GPU memory unload with optimized warm-start (RHOAI can scale-to-zero via Knative mode, but it tears down the whole pod — slower, coarser)
- Multi-cluster — one API across many GPU clusters
- Request queuing — instead of dropping requests when busy
- Container caching — pre-pull 50GB+ images so cold starts are fast
- Rate limiting — built in, per-function
- Multi-protocol — HTTP, gRPC, and streaming (WebRTC) out of the box
- Multi-tenancy — purpose-built isolation between teams/orgs
- Secrets management — OpenBao (Vault-compatible) built in

**RHOAI does things NVCF can't:**
- Notebooks/workbenches — develop and experiment
- Training — Ray, distributed training
- Pipelines — automate ML workflows
- Model registry — version and track models
- TrustyAI — guardrails, bias detection
- OpenShift-native — Routes, OAuth, OLM, SCCs
- Dashboard UI — point-and-click, no CLI needed
- Pre-built runtimes — validated vLLM, OpenVINO, MLServer, NIM
- Canary deployments — KServe supports traffic splitting between model versions
- Lighter footprint — no extra databases or messaging systems needed

They're complementary, not competing. The integration vision: RHOAI handles develop/train/register, NVCF handles the GPU serving layer with scale-to-zero and multi-cluster routing.

## Detailed Comparison

### Overview Table

| | RHOAI | NVCF |
|---|---|---|
| **Built by** | Red Hat (OpenShift-native) | NVIDIA (Kubernetes-native) |
| **Primary focus** | Full ML lifecycle (notebooks, training, serving, monitoring) | GPU workload orchestration (serving, scaling, multi-cluster) |
| **Target users** | Data scientists, ML engineers | Platform teams, GPU cloud providers |
| **Model serving engine** | KServe + vLLM (or OpenVINO, MLServer) | Custom invocation plane + any container |
| **Deployment interface** | Dashboard UI or Kubernetes YAML | CLI (`nvcf-cli`) or REST API |
| **API format** | OpenAI-compatible (via vLLM) | OpenAI-compatible (via LLM Gateway) |
| **GPU management** | GPU Operator + hardware profiles | GPU-type-aware placement + NVCA agent per cluster |
| **Protocol support** | REST only (raw deployment mode) | HTTP, gRPC, streaming (WebRTC) |
| **Infrastructure deps** | Uses what's already in OpenShift | Requires NATS, Cassandra, OpenBao |
| **License** | Proprietary (Red Hat subscription) | Apache 2.0 (open source, April 2026) |

## Where They Overlap

### Self-Service Model Deployment
Both platforms abstract away Kubernetes complexity:
- **RHOAI**: Data scientist uses the dashboard to pick a model, select a runtime (vLLM), choose a hardware profile (GPU), and click deploy.
- **NVCF**: Data scientist uses the CLI or API to specify a container image, GPU type, and scaling policy. One command deploys.

Both produce the same end result: a running model with an API endpoint.

### OpenAI-Compatible API
Both expose models via the `/v1/chat/completions` endpoint format. Applications can target either platform without code changes.

### Authentication & Access Control
- **RHOAI**: Token authentication via Kubernetes service accounts.
- **NVCF**: Built-in API key management with scoped permissions (per-function, per-action).

Both work, but NVCF's is more granular — you can create keys limited to specific functions with specific actions (invoke only, manage, admin).

### Autoscaling
- **RHOAI**: HPA-based autoscaling (CPU/memory metrics). KServe provides request-based scaling with Knative (if enabled).
- **NVCF**: GPU-aware autoscaling based on inference queue depth — specifically designed for inference workloads.

NVCF's autoscaling is smarter for GPU inference because it looks at queue depth (how many requests are waiting), not CPU/memory which don't reflect GPU inference load accurately.

### NIM Integration
Both platforms support NVIDIA NIM (optimized model containers), but differently:
- **RHOAI**: NIM is a serving runtime option in the dashboard — you select it when deploying a model.
- **NVCF**: NIM containers are deployed as functions — NVCF orchestrates them like any other container.

Same NIM containers underneath, different management layer on top.

## Where NVCF Goes Further

### Scale-to-Zero (GPU Memory Release)
This is NVCF's biggest differentiator. When a model is idle, NVCF unloads it from GPU memory using a managed, optimized process.

**Important nuance**: RHOAI *can* scale-to-zero if you enable Knative/Serverless mode (instead of the raw deployment mode we used). But there's a quality difference:
- **RHOAI (Knative mode)**: Tears down the entire pod when idle. On wake-up, the pod is recreated from scratch — pull image, load model, warm up. For a large model (70B), this can take 60+ seconds.
- **NVCF**: Managed GPU memory unload with optimized warm-start and container caching. The recovery is faster because NVCF pre-caches images and model weights on the node.

Also: RHOAI's Knative mode requires installing OpenShift Serverless and Service Mesh operators (heavier stack). Raw deployment mode is simpler but has no scale-to-zero at all.

For enterprises with many models and limited GPUs, NVCF's approach is transformative — 5-10x more models on the same hardware, with faster cold starts.

### Multi-Cluster Management
NVCF manages multiple GPU clusters from a single control plane:
- Different regions, GPU types, on-prem vs cloud
- Automatic request routing to the right cluster
- Single API and single set of credentials

RHOAI manages one OpenShift cluster at a time. Multi-cluster requires ACM (Advanced Cluster Management) as a separate product, which doesn't provide GPU-aware routing.

### Request Queuing
When all GPU workers are busy, NVCF queues requests instead of rejecting them. Callers can check their queue position. This is critical for GPU inference where processing can take seconds to minutes.

RHOAI/KServe returns errors or relies on standard Kubernetes service behavior when pods are overloaded.

### Container Caching
NVCF's Container Cache DaemonSet pre-pulls and caches container images and model weights on GPU nodes, dramatically reducing cold start times. Without this, pulling a 50GB+ model image can take minutes.

RHOAI has no equivalent — cold starts depend on standard image pull speed and model download time. (NIM Operator's NIMCache feature helps by pre-caching NIM model weights, but it's limited to NIM containers only.)

### Multi-Protocol Support
NVCF supports HTTP, gRPC, and streaming (including WebRTC for video) out of the box. This matters for:
- Real-time video processing (WebRTC)
- Speech-to-text streaming
- High-performance model-to-model communication (gRPC)

RHOAI's KServe in raw deployment mode only supports REST. Knative mode adds gRPC support, but not WebRTC.

### Multi-Tenancy
NVCF has purpose-built multi-tenant isolation:
- Team A can't see or invoke Team B's functions unless explicitly granted access
- Scoped API keys per team/org
- Separate resource quotas per tenant

RHOAI uses OpenShift's standard namespace-level RBAC for isolation. It works, but isn't purpose-built for AI workloads — there's no concept of "Team A's functions" at the platform level.

### Batch/Async Tasks
NVCF supports two workload types under one unified API:
- **Functions**: Long-running inference services (like what we deployed)
- **Tasks**: Run-to-completion batch jobs (fine-tuning, batch inference, data prep)

RHOAI handles batch workloads through separate components (Pipelines, Ray, Trainer) rather than a unified task API. It's more capable for complex workflows, but less unified.

### Built-In Rate Limiting
NVCF includes per-function rate limiting out of the box (e.g., "this function allows 100 requests per second"). RHOAI requires external solutions (e.g., API gateway, Istio rate limiting).

### Secrets Management
NVCF includes OpenBao (Vault-compatible) for secrets management:
- Encrypts secrets at rest
- Injects secrets into model pods automatically
- Manages registry credentials

RHOAI uses standard Kubernetes secrets and OpenShift's built-in secret management. Functional, but less sophisticated.

## Where RHOAI Goes Further

### Full ML Lifecycle
RHOAI covers the entire workflow, not just serving:
- **Workbenches**: Jupyter notebooks for development and experimentation
- **Pipelines**: Kubeflow Pipelines for automated ML workflows
- **Training**: Distributed training with Ray and Trainer
- **Model Registry**: Version and track models across their lifecycle
- **TrustyAI**: Model explainability, bias detection, guardrails
- **MLflow**: Experiment tracking and comparison
- **Feature Store**: Feast integration for feature management

NVCF focuses narrowly on deployment and invocation — it assumes models are already trained and containerized.

### OpenShift Integration
RHOAI is deeply integrated with OpenShift:
- Uses OpenShift Routes for ingress (no Gateway API dependency)
- Leverages OpenShift OAuth for user authentication
- Works with OpenShift monitoring (Prometheus, Grafana built-in)
- Respects OpenShift SCCs and RBAC
- Managed by OLM (Operator Lifecycle Manager)

NVCF requires Gateway API + Envoy Gateway for networking, which conflicts with OpenShift's Route-based model. This is one of the main integration challenges identified in Phase 1.

### Pre-Built Serving Runtimes
RHOAI ships with validated, Red Hat-supported runtimes:
- vLLM for NVIDIA, AMD, Intel, CPU
- OpenVINO for Intel-optimized inference
- MLServer for traditional ML models
- NIM integration for NVIDIA's optimized model containers

NVCF is runtime-agnostic — you bring any container, but you configure it yourself. More flexible, but more work.

### Dashboard UI
RHOAI's dashboard provides a visual, point-and-click experience for the entire workflow. NVCF is CLI/API-first with no built-in UI. For data scientists who don't want to use a terminal, this is a significant difference.

### Canary Deployments
KServe (used by RHOAI) supports traffic splitting between model versions — e.g., send 90% of traffic to v1 and 10% to v2. This enables canary testing of new model versions. NVCF supports deployment versions but is less explicit about traffic splitting.

### Lighter Infrastructure Footprint
RHOAI uses what's already in OpenShift — no extra databases or messaging systems. NVCF requires three additional infrastructure components (NATS, Cassandra, OpenBao) running in the cluster, which adds operational overhead.

### Observability
Both have monitoring, but different approaches:
- **RHOAI**: Leverages OpenShift's built-in Prometheus and Grafana. General-purpose metrics.
- **NVCF**: Ships its own Prometheus metrics, OpenTelemetry tracing, and reference Grafana dashboards purpose-built for GPU inference (queue depth, per-function GPU utilization, latency histograms).

NVCF's observability is more inference-specific. RHOAI's is more general but already integrated with what platform teams know.

### Multi-Container Deployments
NVCF lets you deploy a single container or a Helm chart (multi-container deployments). This is useful for complex inference setups (model + preprocessing + postprocessing in one deployment). RHOAI's KServe is single-container per InferenceService.

## Complementary, Not Competing

The key insight: **RHOAI and NVCF are complementary rather than competing**.

| Capability | Best Provided By |
|---|---|
| ML development environment | RHOAI (workbenches, notebooks) |
| Training & pipelines | RHOAI (KFP, Ray, Trainer) |
| Model registry & versioning | RHOAI (model registry) |
| Model explainability & guardrails | RHOAI (TrustyAI) |
| Dashboard UI for data scientists | RHOAI |
| Pre-built, validated runtimes | RHOAI (vLLM, OpenVINO, NIM) |
| OpenShift-native integration | RHOAI (Routes, OAuth, OLM) |
| Canary deployments / traffic splitting | RHOAI (KServe) |
| Single-cluster model serving | Either (both work well) |
| Scale-to-zero GPU memory management | NVCF (faster, smarter than RHOAI's Knative mode) |
| Multi-cluster GPU orchestration | NVCF (single control plane) |
| Request queuing & rate limiting | NVCF (built-in) |
| Container/model caching | NVCF (Container Cache DaemonSet) |
| Multi-protocol (gRPC, WebRTC) | NVCF |
| Multi-tenant isolation | NVCF (purpose-built) |
| Secrets management | NVCF (OpenBao/Vault) |
| Multi-container deployments | NVCF (Helm chart packaging) |

In short: everything about **building, training, and managing models** → RHOAI. Everything about **running GPUs efficiently at scale** → NVCF.

### Potential Integration Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    RHOAI (OpenShift)                      │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  │
│  │Workbench │  │Pipelines │  │ Model    │  │TrustyAI│  │
│  │(Jupyter) │  │  (KFP)   │  │ Registry │  │(Guard) │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───┬────┘  │
│       │ Develop      │ Automate    │ Version     │ Guard  │
│       ▼              ▼             ▼             ▼       │
│  ┌───────────────────────────────────────────────────┐   │
│  │          NVCF (GPU Workload Manager)               │   │
│  │                                                    │   │
│  │  ┌────────────┐  ┌───────────────┐  ┌──────────┐ │   │
│  │  │ Scale-to-  │  │ Multi-cluster │  │ Request  │ │   │
│  │  │   zero     │  │   routing     │  │ queuing  │ │   │
│  │  └────────────┘  └───────────────┘  └──────────┘ │   │
│  │  ┌────────────┐  ┌───────────────┐  ┌──────────┐ │   │
│  │  │ Container  │  │ Rate limiting │  │ Multi-   │ │   │
│  │  │  caching   │  │ & API keys    │  │ tenancy  │ │   │
│  │  └────────────┘  └───────────────┘  └──────────┘ │   │
│  └───────────────────────────────────────────────────┘   │
│       │                                                  │
│       ▼                                                  │
│  ┌───────────────────────────────────────────────────┐   │
│  │       GPU Nodes (Tesla T4, A100, H100)             │   │
│  │       via NVIDIA GPU Operator                      │   │
│  └───────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

In this model:
- RHOAI handles the ML lifecycle (develop, train, register, guard models)
- NVCF handles the GPU runtime layer (deploy, scale, route, cache, isolate)
- Both share the same GPU Operator and physical GPU nodes
- The OpenAI-compatible API is the common interface

## What We Demonstrated

By deploying TinyLlama on RHOAI, we proved that:
1. RHOAI model serving works end-to-end on OpenShift with NVIDIA GPUs
2. The dashboard provides a self-service deployment experience
3. KServe + vLLM produce an OpenAI-compatible inference endpoint
4. The raw deployment mode (no Knative/Istio) keeps the infrastructure simple

What RHOAI lacks that NVCF would add:
- Smarter scale-to-zero would free the T4 GPU when TinyLlama is idle (with faster recovery than Knative)
- Container caching would speed up cold starts
- If we had multiple clusters, NVCF would unify them under one API
- Request queuing would handle traffic spikes gracefully
- Multi-tenancy would isolate different teams' workloads

## Next Steps

1. Deploy NVCF on the same OpenShift cluster (Phase 2)
2. Deploy the same model (TinyLlama) via NVCF and compare the operational experience
3. Test NVCF's scale-to-zero behavior on OpenShift
4. Identify integration points where NVCF could use RHOAI's model registry or where RHOAI could delegate serving to NVCF
5. Document a proposal for Red Hat-NVIDIA integration
