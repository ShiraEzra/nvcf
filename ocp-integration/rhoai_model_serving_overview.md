# RHOAI Model Serving Overview

## What We Deployed

We deployed TinyLlama-1.1B-Chat as a self-hosted AI inference endpoint on OpenShift using RHOAI 3.4. The stack:

- **RHOAI Dashboard** -- self-service UI for deploying and managing models
- **KServe** (v0.17.0) -- Kubernetes-native model serving framework
- **vLLM** (v0.18.0) -- high-throughput inference engine for LLMs
- **Raw Deployment mode** -- uses standard Kubernetes Deployments (no Knative/Istio/ServiceMesh required)

The deployed model exposes an **OpenAI-compatible API** (`/v1/chat/completions`), meaning any application built for the OpenAI/ChatGPT API can point to this endpoint instead.

## Deployment Details

| Setting | Value |
|---------|-------|
| Model | TinyLlama-1.1B-Chat-v1.0 (from HuggingFace) |
| Runtime | vLLM NVIDIA GPU ServingRuntime for KServe |
| GPU | 1x Tesla T4 (16 GiB VRAM) |
| VRAM usage | 2.05 GiB model + 10.86 GiB KV cache |
| Max context | 2,048 tokens |
| Concurrent capacity | ~253 requests at max context |
| Precision | FP16 (T4 does not support BF16) |
| Endpoint | `https://tinyllama-1b-chat-test-model-serving.apps.nvsentinel-ocp.salfasi.okoyl.xyz` |
| Auth | Bearer token (from SA secret `default-token-tinyllama-1b-chat-sa`) |

## Why Deploy Models In-Cluster?

### 1. Data Privacy & Sovereignty
The model runs entirely on your hardware. No data leaves the cluster. Critical for:
- Healthcare (patient data, HIPAA)
- Finance (trading strategies, customer PII)
- Government / defense (classified or sensitive data)
- Any org with data residency requirements (GDPR, etc.)

### 2. Cost Control at Scale
Cloud API pricing (OpenAI, Anthropic, etc.) charges per-token. At scale (millions of requests/day), self-hosting on owned GPU infrastructure is significantly cheaper. The cost model shifts from variable (per-request) to fixed (infrastructure).

### 3. No Rate Limits or Vendor Lock-In
- No API quotas or throttling
- No dependency on third-party availability
- No surprise pricing changes
- Swap models without changing application code

### 4. Customization
- Fine-tune models on proprietary data
- Deploy specialized models per team or use case
- Control model versions, rollbacks, A/B testing
- Run models not available through cloud APIs

### 5. Low Latency
In-cluster inference avoids round-trips to external APIs. For latency-sensitive applications (real-time chat, code completion, autonomous systems), this matters.

## What RHOAI Adds

Without RHOAI, deploying a model on Kubernetes requires:
- Writing ServingRuntime YAML
- Configuring InferenceService resources
- Managing GPU scheduling and resource limits
- Setting up authentication, routing, TLS
- Monitoring and scaling

RHOAI provides a **self-service platform layer** on top of OpenShift:
- **Dashboard UI** -- data scientists deploy models without knowing Kubernetes
- **Pre-built serving runtimes** -- vLLM, OpenVINO, MLServer, ready to use
- **Hardware profiles** -- abstract GPU/CPU/memory allocation
- **Model registry** -- track and version models
- **NIM integration** -- deploy NVIDIA NIM models directly
- **Token authentication** -- built-in API security
- **Monitoring** -- metrics, health checks, auto-scaling (HPA)

## Available Serving Runtimes (RHOAI 3.4)

| Runtime | Use Case |
|---------|----------|
| vLLM NVIDIA GPU | LLMs on NVIDIA GPUs (our setup) |
| vLLM CPU (x86) | LLM inference without GPUs |
| vLLM AMD GPU (ROCm) | LLMs on AMD GPUs |
| vLLM Intel Gaudi | LLMs on Habana Gaudi accelerators |
| vLLM Multi-node | Large models across multiple nodes |
| OpenVINO Model Server | Optimized inference on Intel hardware |
| MLServer | General ML models (scikit-learn, XGBoost, etc.) |
| Hugging Face Detector | TrustyAI guardrails for content safety |

## RHOAI Dashboard Navigation

| Section | Purpose |
|---------|---------|
| **Projects** | Namespace-scoped workspaces grouping models, workbenches, pipelines, storage |
| **AI hub > Models > Catalog** | Browse Red Hat validated models and community models |
| **AI hub > Models > Registry** | Track registered model versions |
| **AI hub > Models > Deployments** | View and manage deployed models across all projects |
| **Develop & train** | Workbenches (Jupyter), pipelines, experiments (MLflow), distributed training |
| **Applications > Enabled** | Quick-launch enabled applications (MLflow, workbenches) |
| **Settings > Serving runtimes** | Under "Model resources and operations" -- manage available runtimes |
| **Settings > Model registry settings** | Configure model registry backends |

## Deployment Flow (via Dashboard)

1. **Create a project** (Projects > Create a project) -- creates a namespace
2. **Deploy model** (from project page > Serve models > Deploy model):
   - Step 1: Model details -- source URI, model type (Generative AI / Predictive)
   - Step 2: Model deployment -- name, hardware profile, serving runtime, replicas
   - Step 3: Advanced settings -- external route, token auth, timeout, env vars
   - Step 4: Review and deploy

## Gotchas Encountered

1. **GPU hardware profile required**: The default hardware profile has no GPU. vLLM crashes with `RuntimeError: Failed to infer device type`. We created a custom `nvidia-gpu-small` profile with `nvidia.com/gpu: 1`.
2. **Leftover secrets on re-deploy**: When deleting and re-creating a deployment with the same name, the token secret from the previous deployment may persist and cause `secrets "X" already exists` errors. Delete the secret manually before re-deploying.
3. **T4 precision**: Tesla T4 (compute capability 7.5) does not support BF16. vLLM automatically falls back to FP16.
4. **Flash Attention 2 unsupported on T4**: FA2 requires compute capability >= 8.0. vLLM falls back to FlashInfer attention backend.

## Connection to NVCF

NVCF (NVIDIA Cloud Functions) is NVIDIA's platform for deploying and managing GPU workloads. The key question: **how do RHOAI and NVCF complement or overlap?**

- RHOAI provides the OpenShift-native model serving platform (KServe + vLLM)
- NVCF provides function orchestration, container caching, and GPU cluster management
- Both handle model deployment, scaling, and inference routing
- Potential complementary areas: NVCF's container caching could accelerate cold starts; NVCF's function chaining could orchestrate multi-model pipelines on top of RHOAI-served models

This comparison is the focus of the next phase.
