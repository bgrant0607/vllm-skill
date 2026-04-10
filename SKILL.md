---
name: vllm-deploy-confighub
description: Deploy vLLM production stack to Kubernetes via ConfigHub. Creates plain Kubernetes YAML resources as ConfigHub units — supports multi-model serving, request routing, LMCache, autoscaling, monitoring, LoRA adapters, and more.
trigger: Use whenever user wants to deploy, set up, configure, or manage vLLM on Kubernetes using ConfigHub, or when they want to create a vLLM production stack.
---

# vLLM Production Stack — ConfigHub Deployment Skill

This skill deploys a comprehensive vLLM production stack as plain Kubernetes YAML resources stored in ConfigHub units. It does NOT use Helm, Kustomize, or the vLLM operator.

## Overview

The vLLM production stack consists of these components:

| Component | Purpose | Templates |
|-----------|---------|-----------|
| **Namespace** | Kubernetes namespace | `namespace.yaml` |
| **Serving Engine** | Runs vLLM model inference | `engine-deployment.yaml`, `engine-service.yaml` |
| **Router** | Load-balances across engines | `router-deployment.yaml`, `router-service.yaml`, `router-rbac.yaml` |
| **Cache Server** | LMCache KV cache offloading | `cache-server-deployment.yaml`, `cache-server-service.yaml` |
| **Storage** | Model weight persistence | `engine-pvc.yaml` |
| **Secrets** | API keys and HF tokens | `secrets.yaml` |
| **Autoscaling** | Engine and router scaling | `engine-keda-scaledobject.yaml`, `router-hpa.yaml` |
| **Reliability** | Disruption budgets | `engine-pdb.yaml`, `router-pdb.yaml` |
| **Networking** | External access | `router-ingress.yaml` |
| **Monitoring** | Prometheus metrics | `engine-servicemonitor.yaml`, `router-servicemonitor.yaml` |
| **Config** | Environment variables | `engine-configmap.yaml` |

All templates are in the `templates/` subdirectory relative to this SKILL.md file.

## Deployment Procedure

### Step 1: Gather Requirements

Ask the user for their deployment requirements. Use these questions as a guide — skip questions that aren't relevant based on context:

**Required:**
- What model(s) do you want to serve? (e.g., `meta-llama/Llama-3.1-8B-Instruct`)
- What namespace should the resources be deployed to?

**Optional (offer sensible defaults):**
- Space name for ConfigHub (default: `vllm-stack`)
- Number of GPUs per model (default: 1)
- Replica count (default: 1)
- Do you need a request router? (default: yes, if multiple models or production use)
- Do you need persistent storage for model weights? (default: no)
- Do you need a Hugging Face token for gated models?
- Do you need autoscaling (KEDA for engines, HPA for router)?
- Do you need monitoring (ServiceMonitor)?
- Do you need an Ingress for external access?
- Do you need an LMCache server?
- Routing strategy: roundrobin (default), session, prefixaware, or kvaware?

### Step 2: Create ConfigHub Space

```bash
cub space create SPACE_SLUG
```

Use the space name from Step 1 (default: `vllm-stack`). Record the space slug for all subsequent commands.

### Step 3: Create Units and Customize with Functions

For each component, create a ConfigHub unit from the template YAML, then use `cub function do` to customize it. Do NOT use sed or local file editing for placeholder replacement — upload the template as-is and use ConfigHub functions to replace placeholders and customize values.

The general workflow for each unit:
1. Create the unit from the template: `cub unit create --space SPACE_SLUG UNIT_SLUG templates/TEMPLATE.yaml`
2. Replace the `MODELNAME` placeholder: `cub function do --space SPACE --unit UNIT_SLUG search-replace MODELNAME actual-model-slug`
3. Replace the `MODELURL` placeholder (engine deployment only): `cub function do --space SPACE --unit UNIT_SLUG search-replace MODELURL actual-model-url`
4. Link the unit to the namespace unit: `cub link create --space SPACE - UNIT_SLUG vllm-namespace` — this automatically sets `metadata.namespace`
5. Use `set-image`, `set-replicas`, `set-container-flag`, `yq-i`, and other functions for further customization

The templates use `confighubplaceholder` for the Kubernetes namespace (set automatically when linked to the namespace unit) and distinct placeholders `MODELNAME` and `MODELURL` for model-specific values (handled by `search-replace`).

**Important function usage notes:**
- `yq` is **read-only** (displays output only). Use `yq-i` to **mutate** config data.
- `search-replace` works on all string values including inside arrays.
- `set-container-flag` sets `--flag=value` style args. Templates use this format.
- Linking a unit to the namespace unit sets `metadata.namespace` automatically, but does NOT change namespace references inside container args — use `set-container-flag` for those (e.g., the router's `--k8s-namespace`).
- Use `--unit UNIT_SLUG` instead of `--where "Slug = 'UNIT_SLUG'"` for targeting a single unit — it's simpler.

#### 3a: Namespace

Always create the namespace unit first. All other units will be linked to it.

Template: `namespace.yaml`

```bash
cub unit create --space SPACE vllm-namespace templates/namespace.yaml
cub function do --space SPACE --unit vllm-namespace search-replace confighubplaceholder NAMESPACE
```

**Unit slug:** `vllm-namespace`

#### 3b: Secrets (if needed)

Create this unit if the user needs HF tokens or a vLLM API key.

Template: `secrets.yaml`

To add secret data after creating the unit:
```bash
cub function do --space SPACE --unit vllm-secrets yq-i '.data.HF_TOKEN = "BASE64_ENCODED_VALUE"'
```

For HF tokens per model, use key names like `hf_token_MODELNAME`.

**Unit slug:** `vllm-secrets`

#### 3c: Serving Engine (per model)

For EACH model, create a Deployment and Service. Replace `MODELNAME` with a short slug for the model (e.g., `llama3-8b`).

**Deployment — Template:** `engine-deployment.yaml`

Create the unit, then customize with functions:
```bash
cub unit create --space SPACE vllm-MODELSLUG-engine-deployment templates/engine-deployment.yaml
cub function do --space SPACE --unit vllm-MODELSLUG-engine-deployment search-replace MODELNAME MODELSLUG
cub function do --space SPACE --unit vllm-MODELSLUG-engine-deployment search-replace MODELURL 'MODEL_URL'
```

Where `MODELSLUG` is a short name (e.g., `llama3-8b`) and `MODEL_URL` is the full model path (e.g., `meta-llama/Llama-3.1-8B-Instruct`). The first `search-replace` handles all label/name references; the second replaces the model URL placeholder in the vllm serve command. Namespace placeholders (`confighubplaceholder`) are handled later by `set-namespace`.

**vLLM configuration flags** — Use `set-container-flag` to add or change `--flag=value` args on the unit after creation, or use `yq-i` to append boolean flags:

```bash
# Example: set tensor parallelism
cub function do --space SPACE --unit UNIT set-container-flag vllm tensor-parallel-size 2

# Example: add a boolean flag (no value)
cub function do --space SPACE --unit UNIT yq-i '.spec.template.spec.containers[0].command += ["--enable-chunked-prefill"]'
```

| Feature | Flag | Example value |
|---------|------|---------------|
| Tensor parallelism | `tensor-parallel-size` | `2` |
| Max model length | `max-model-len` | `16384` |
| Data type | `dtype` | `bfloat16` |
| Max sequences | `max-num-seqs` | `32` |
| GPU memory utilization | `gpu_memory_utilization` | `0.95` |
| Max LoRAs | `max_loras` | `4` |
| Tool call parser | `tool-call-parser` | `hermes` |
| Runner type | `runner` | `pooling` |
| Boolean flags (use yq-i) | `--enable-chunked-prefill`, `--enable-prefix-caching`, `--enable-lora`, `--enable-auto-tool-choice` | |
| vLLM v0 mode | Add env var `VLLM_USE_V1=0` instead of `PROMETHEUS_MULTIPROC_DIR` | |

**LMCache configuration** — If LMCache is enabled, add these to the container:

1. Add arg via yq-i: `--kv-transfer-config={"kv_connector":"LMCacheConnectorV1","kv_role":"kv_both"}`
2. Add env vars:
   - `LMCACHE_USE_EXPERIMENTAL: "True"`
   - `VLLM_RPC_TIMEOUT: "1000000"`
   - `LMCACHE_LOG_LEVEL: "INFO"` (or DEBUG, WARNING, ERROR)
3. For CPU offloading, add: `LMCACHE_LOCAL_CPU: "True"`, `LMCACHE_MAX_LOCAL_CPU_SIZE: "30"` (GB)
4. For disk offloading, add: `LMCACHE_MAX_LOCAL_DISK_SIZE: "N"` (GB)
5. For remote cache server, add: `LMCACHE_REMOTE_URL: "lmcache://vllm-cache-server-service:PORT"`, `LMCACHE_REMOTE_SERDE: "naive"`
6. For KV-aware routing controller integration, add: `LMCACHE_ENABLE_CONTROLLER: "True"`, `LMCACHE_LMCACHE_INSTANCE_ID` (from `metadata.name` fieldRef), `LMCACHE_CONTROLLER_PULL_URL`, `LMCACHE_LMCACHE_WORKER_PORTS`
7. For NIXL (disaggregated prefill/decode), add: `LMCACHE_ENABLE_NIXL`, `LMCACHE_NIXL_ROLE`, `LMCACHE_NIXL_RECEIVER_HOST`, `LMCACHE_NIXL_RECEIVER_PORT`, `LMCACHE_NIXL_BUFFER_SIZE`, `LMCACHE_NIXL_BUFFER_DEVICE`, `LMCACHE_NIXL_ENABLE_GC`
8. For PD mode, also set `hostIPC: true`, `hostPID: true` on the pod spec, and additional PD-specific env vars

**HF token** — If the model requires a Hugging Face token, add an env var to the container:
```yaml
- name: HF_TOKEN
  valueFrom:
    secretKeyRef:
      name: vllm-secrets
      key: hf_token_MODELNAME
```
Also change `HF_HOME` to `/data` if using PVC storage.

**vLLM API key** — If securing the API, add:
```yaml
- name: VLLM_API_KEY
  valueFrom:
    secretKeyRef:
      name: vllm-secrets
      key: vllmApiKey
```

**Persistent storage** — If using PVC storage, add to the container:
```yaml
volumeMounts:
  - name: model-storage
    mountPath: /data
```
And to the pod spec:
```yaml
volumes:
  - name: model-storage
    persistentVolumeClaim:
      claimName: vllm-MODELNAME-storage-claim
```
Also change `HF_HOME` env var to `/data`.

**Shared memory** — If using tensor parallelism, add:
```yaml
volumeMounts:
  - name: shm
    mountPath: /dev/shm
```
And:
```yaml
volumes:
  - name: shm
    emptyDir:
      medium: Memory
      sizeLimit: 20Gi  # adjust as needed
```

**Resources** — Adjust CPU, memory, and GPU requests/limits based on the model and hardware:
- Small models (1-3B): 4 CPU, 8Gi memory, 1 GPU
- Medium models (7-13B): 6 CPU, 16Gi memory, 1 GPU
- Large models (30-70B): 12 CPU, 64Gi memory, 2-4 GPUs (with tensor parallelism)

For HAMi GPU scheduling, use custom resource names like `nvidia.com/gpumem`, `nvidia.com/gpumem-percentage`, `nvidia.com/gpucores`.

**Init containers** — If the user needs init containers (e.g., for model preparation), add an `initContainers` section to the pod spec.

**Sidecar for LoRA** — If LoRA is enabled with PVC storage, add a sidecar container:
```yaml
- name: sidecar
  image: lmcache/lmstack-sidecar:latest
  imagePullPolicy: Always
  env:
    - name: PORT
      value: "30090"
    - name: LORA_DOWNLOAD_BASE_DIR
      value: /data/lora-adapters
  volumeMounts:
    - name: model-storage
      mountPath: /data
```

**Chat templates** — If the user needs a custom chat template, create a ConfigMap with the template content and mount it:
```yaml
volumes:
  - name: chat-templates
    configMap:
      name: vllm-MODELNAME-chat-templates
volumeMounts:
  - name: chat-templates
    mountPath: /templates
```
Add arg via set-container-flag: `cub function do --space SPACE --unit UNIT set-container-flag vllm chat-template /templates/TEMPLATE_FILENAME`

**Node placement** — Add affinity, nodeSelector, tolerations, nodeName, priorityClassName, or schedulerName to the pod spec as needed.

**Security contexts** — The template defaults to `runAsNonRoot: false`. Adjust pod-level `securityContext` and container-level `securityContext` as needed.

**Unit slug:** `vllm-MODELSLUG-engine-deployment`

**Service — Template:** `engine-service.yaml`

```bash
cub unit create --space SPACE vllm-MODELSLUG-engine-service templates/engine-service.yaml
cub function do --space SPACE --unit vllm-MODELSLUG-engine-service search-replace MODELNAME MODELSLUG
```

**Unit slug:** `vllm-MODELSLUG-engine-service`

#### 3d: Engine PVC (optional, per model)

Template: `engine-pvc.yaml`

Replace `MODELNAME`. Customize storage size, access modes, and storage class.

**Unit slug:** `vllm-MODELNAME-engine-pvc`

#### 3e: Router (optional but recommended)

Create these units if the user wants a router (recommended for production or multi-model deployments):

1. **RBAC (ServiceAccount + Role + RoleBinding)** — Template: `router-rbac.yaml` — Unit slug: `vllm-router-rbac`
   - If using `service-name` discovery type instead of `pod-ip`, change the Role resources to `["pods", "services", "endpoints"]`
2. **Deployment** — Template: `router-deployment.yaml` — Unit slug: `vllm-router-deployment`
3. **Service** — Template: `router-service.yaml` — Unit slug: `vllm-router-service`

**Router deployment customization:**

The template uses `--flag=value` format, so `set-container-flag` can modify any arg:

```bash
# Set the namespace for k8s service discovery
cub function do --space SPACE --unit vllm-router-deployment set-container-flag router-container k8s-namespace NAMESPACE

# Change routing strategy
cub function do --space SPACE --unit vllm-router-deployment set-container-flag router-container routing-logic session

# Add session key
cub function do --space SPACE --unit vllm-router-deployment set-container-flag router-container session-key SESSION_KEY_NAME

# Change label selector
cub function do --space SPACE --unit vllm-router-deployment set-container-flag router-container k8s-label-selector "environment=production"
```

The `--k8s-label-selector` arg must match the labels on the engine pods. The default template uses `environment=production`.

Routing strategies — set `routing-logic` to one of:
- `roundrobin` (default) — even distribution
- `session` — sticky sessions; also set `session-key`
- `prefixaware` — route by prompt prefix similarity
- `kvaware` — KV-cache-aware; also set `lmcache-controller-port`

For static service discovery (no k8s API), change service-discovery and add static args:
```bash
cub function do --space SPACE --unit vllm-router-deployment set-container-flag router-container service-discovery static
cub function do --space SPACE --unit vllm-router-deployment set-container-flag router-container static-backends 'http://backend1:8000,http://backend2:8000'
cub function do --space SPACE --unit vllm-router-deployment set-container-flag router-container static-models 'model1,model2'
```

For OpenTelemetry tracing:
```bash
cub function do --space SPACE --unit vllm-router-deployment set-container-flag router-container otel-endpoint HOST:PORT
cub function do --space SPACE --unit vllm-router-deployment set-container-flag router-container otel-service-name vllm-router
```

For the vLLM API key, add the same `VLLM_API_KEY` env var as the engine.

**Service type** — Change `spec.type` from `ClusterIP` to `NodePort` or `LoadBalancer` if needed. For `NodePort`, add `nodePort: PORT` to the port spec.

#### 3f: Router HPA (optional)

Template: `router-hpa.yaml`

If the user wants router autoscaling based on CPU. Do NOT set `spec.replicas` in the router Deployment when using HPA.

**Unit slug:** `vllm-router-hpa`

#### 3g: Router Ingress (optional)

Template: `router-ingress.yaml`

Customize host, paths, TLS, and ingress class.

**Unit slug:** `vllm-router-ingress`

#### 3h: Cache Server (optional)

Template: `cache-server-deployment.yaml`, `cache-server-service.yaml`

Deploy if the user wants remote KV cache offloading. Configure resources for RDMA if needed.

**Unit slugs:** `vllm-cache-server-deployment`, `vllm-cache-server-service`

#### 3i: Pod Disruption Budgets (optional)

Templates: `engine-pdb.yaml`, `router-pdb.yaml`

For production deployments with multiple replicas.

**Unit slugs:** `vllm-MODELNAME-engine-pdb`, `vllm-router-pdb`

#### 3j: KEDA ScaledObject (optional, per model)

Template: `engine-keda-scaledobject.yaml`

For autoscaling engines based on Prometheus metrics (requires KEDA installed). Replace `MODELNAME`. Customize min/max replicas, triggers, and Prometheus query.

Supports scale-to-zero with `idleReplicaCount: 0`.

**Unit slug:** `vllm-MODELNAME-keda-scaledobject`

#### 3k: ServiceMonitors (optional)

Templates: `engine-servicemonitor.yaml`, `router-servicemonitor.yaml`

For Prometheus metrics collection (requires prometheus-operator CRDs).

**Unit slugs:** `vllm-engine-servicemonitor`, `vllm-router-servicemonitor`

#### 3l: ConfigMap (optional)

Template: `engine-configmap.yaml`

For bulk environment variable configuration across all engines.

**Unit slug:** `vllm-configs`

### Step 4: Create Links

Create links between units whose resources reference each other. Use `cub link create --space SPACE "-" FROM_UNIT TO_UNIT` where FROM_UNIT is the unit that contains the reference and TO_UNIT is the unit being referred to.

**Namespace links (always create these for every unit except the namespace itself):**

Link every unit to the namespace unit. This automatically sets `metadata.namespace` on each unit's resources.

```bash
# Link all units to namespace (repeat for each unit created)
cub link create --space SPACE - vllm-MODELSLUG-engine-deployment vllm-namespace
cub link create --space SPACE - vllm-MODELSLUG-engine-service vllm-namespace
cub link create --space SPACE - vllm-router-rbac vllm-namespace
cub link create --space SPACE - vllm-router-deployment vllm-namespace
cub link create --space SPACE - vllm-router-service vllm-namespace
cub link create --space SPACE - vllm-cache-server-deployment vllm-namespace
cub link create --space SPACE - vllm-cache-server-service vllm-namespace
# ... and any other units (secrets, PVC, PDB, HPA, ingress, etc.)
```

**Resource reference links (create based on which components were deployed):**

Engine links (per model):
```bash
# Engine service selects engine deployment pods
cub link create --space SPACE - vllm-MODELSLUG-engine-service vllm-MODELSLUG-engine-deployment
```

Router links (if router is enabled):
```bash
# Router deployment references the RBAC service account and discovers engine pods
cub link create --space SPACE - vllm-router-deployment vllm-router-rbac
cub link create --space SPACE - vllm-router-deployment vllm-MODELSLUG-engine-deployment

# Router service selects router deployment pods
cub link create --space SPACE - vllm-router-service vllm-router-deployment
```

If serving multiple models, create a link from the router deployment to each engine deployment.

Cache server links (if cache server is enabled):
```bash
# Cache server service selects cache server deployment pods
cub link create --space SPACE - vllm-cache-server-service vllm-cache-server-deployment
```

Optional component links:
```bash
# HPA targets the router deployment
cub link create --space SPACE - vllm-router-hpa vllm-router-deployment

# Ingress routes to the router service
cub link create --space SPACE - vllm-router-ingress vllm-router-service

# KEDA ScaledObject targets the engine deployment
cub link create --space SPACE - vllm-MODELSLUG-keda-scaledobject vllm-MODELSLUG-engine-deployment

# PDB selects engine/router deployment pods
cub link create --space SPACE - vllm-MODELSLUG-engine-pdb vllm-MODELSLUG-engine-deployment
cub link create --space SPACE - vllm-router-pdb vllm-router-deployment

# ServiceMonitors select services
cub link create --space SPACE - vllm-engine-servicemonitor vllm-MODELSLUG-engine-service
cub link create --space SPACE - vllm-router-servicemonitor vllm-router-service

# Secrets referenced by engine deployments
cub link create --space SPACE - vllm-MODELSLUG-engine-deployment vllm-secrets

# PVC referenced by engine deployment
cub link create --space SPACE - vllm-MODELSLUG-engine-deployment vllm-MODELSLUG-engine-pvc

# ConfigMap referenced by engine deployment
cub link create --space SPACE - vllm-MODELSLUG-engine-deployment vllm-configs
```

### Step 5: Verify Namespaces

All units linked to the namespace unit in Step 4 should have their `metadata.namespace` set automatically. The router's `--k8s-namespace` container arg must still be set separately via `set-container-flag` (see Step 3e).

### Step 6: Verify

Check for remaining placeholders:

```bash
cub function do --space SPACE get-placeholders
```

If any placeholders remain (`confighubplaceholder`, `MODELNAME`, or `MODELURL`), fix them with the appropriate function:

```bash
# Namespace placeholders — should have been handled by set-namespace
cub function do --space SPACE set-namespace NAMESPACE

# Model-specific placeholders
cub function do --space SPACE --unit UNIT_SLUG search-replace MODELNAME actual-model-slug
cub function do --space SPACE --unit UNIT_SLUG search-replace MODELURL actual-model-url
```

List all created units:

```bash
cub unit list --space SPACE
```

### Step 7: Report Summary

Print a summary of what was created:
- Space name
- List of units created and their resource types
- Key configuration decisions (model, GPU count, routing strategy, etc.)
- Next steps: `cub unit apply --space SPACE UNIT_SLUG` to deploy to a cluster

## Configuration Reference

### Engine Container Image

| Repository | When to use |
|-----------|-------------|
| `vllm/vllm-openai` | Standard vLLM (use `vllm` command) |
| `lmcache/vllm-openai` | LMCache-integrated vLLM (use `/opt/venv/bin/vllm` command) |

When using `lmcache/vllm-openai`, change the container command from `vllm` to `/opt/venv/bin/vllm`.

### Label Conventions

Engine pods use these labels for service discovery and selection:
- `model: MODELNAME` — identifies the model
- `app.kubernetes.io/name: serving-engine`
- `app.kubernetes.io/instance: vllm`
- `app.kubernetes.io/component: serving-engine`
- `app.kubernetes.io/part-of: vllm-stack`
- `environment: production` — used by router's `--k8s-label-selector`

Router pods use:
- `app.kubernetes.io/name: router`
- `app.kubernetes.io/instance: vllm`
- `app.kubernetes.io/component: router`
- `app.kubernetes.io/part-of: vllm-stack`

### Ports

| Component | Port | Purpose |
|-----------|------|---------|
| Engine | 8000 | vLLM API (OpenAI-compatible) |
| Engine | 55555 | ZMQ (internal) |
| Engine | 9999 | UCX (internal) |
| Router | 8000 | Router API |
| Router | 9000 | LMCache controller |
| Cache Server | 8000 | LMCache server |

### GPU Resource Types

| Type | Resource key |
|------|-------------|
| Standard NVIDIA | `nvidia.com/gpu` |
| NVIDIA MIG | `nvidia.com/mig-4g.71gb` (etc.) |
| HAMi GPU memory | `nvidia.com/gpumem` |
| HAMi GPU memory % | `nvidia.com/gpumem-percentage` |
| HAMi GPU cores | `nvidia.com/gpucores` |

## Troubleshooting

Common issues when deploying vLLM:

- **Pod stuck in Pending**: Check GPU availability (`kubectl describe node`), resource requests, node selectors/affinity
- **OOMKilled**: Increase memory limits or reduce `--gpu_memory_utilization`
- **CrashLoopBackOff**: Check logs (`kubectl logs`), verify model URL, check HF token for gated models
- **Startup probe failures**: vLLM model loading can take 5-10+ minutes for large models; increase `failureThreshold` on startupProbe
- **Router can't find backends**: Verify label selectors match between router args and engine pod labels; check RBAC permissions
