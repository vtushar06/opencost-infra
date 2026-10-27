# vLLM Inference Implementation Summary

## What Was Done

This implementation adds a vLLM inference server to the OpenCost demo cluster for validating OpenCost's new vLLM metrics support (opencost/opencost#3915).

## Files Added/Modified

### New Files

1. **`argocd/apps/demo/vllm-inference/deployment.yaml`** (4 resources)
   - ConfigMap: Model configuration (model name, context length, GPU settings)
   - Deployment: vLLM server container with GPU scheduling
   - Service: ClusterIP service on port 8000
   - PodMonitor: Prometheus metrics scraping with pod_uid relabel

2. **`argocd/apps/demo/vllm-inference/haiku-harness.yaml`** (3 resources)
   - ConfigMap: Bash script that generates haiku requests
   - CronJob: Runs load harness every 15 minutes
   - Job: Manual trigger for immediate testing

3. **`argocd/appsets/vllm-inference-argo.yaml`**
   - ApplicationSet that follows repo conventions
   - Uses git directory generator on `argocd/apps/*/vllm-inference`
   - Configured for auto-sync, prune, and self-heal

### Modified Files

4. **`argocd/apps/demo/rbac/ci-user-rbac-and-vap.yaml`**
   - Added `vllm-inference` to two namespace allowlists (lines ~147 and ~199)
   - Protects namespace from CI user modifications

5. **`.github/workflows/nightly-clean-up-stacks.yaml`**
   - Added `vllm-inference` to namespace allowlist (line ~47)
   - Prevents nightly cleanup from deleting the namespace

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│ vllm-inference namespace                                     │
│                                                              │
│  ┌──────────────────────────────────────────┐               │
│  │ vLLM Deployment                          │               │
│  │ - Image: vllm/vllm-openai:v0.6.4.post1  │               │
│  │ - Model: Qwen/Qwen2.5-1.5B-Instruct     │               │
│  │ - GPU: nvidia.com/gpu: 1 (A10 24GB)     │               │
│  │ - Port: 8000 (OpenAI-compatible API)    │               │
│  │ - Metrics: /metrics (vLLM scheduler)    │               │
│  │ - Label: llm-d.ai/model                 │               │
│  └──────────────────────────────────────────┘               │
│                    ↑                                         │
│                    │ scrapes every 15s                       │
│                    │                                         │
│  ┌──────────────────────────────────────────┐               │
│  │ PodMonitor                               │               │
│  │ - Relabels pod_uid, pod, namespace       │               │
│  └──────────────────────────────────────────┘               │
│                                                              │
│  ┌──────────────────────────────────────────┐               │
│  │ Haiku Load Harness (CronJob)             │               │
│  │ - Runs: */15 * * * * (every 15 min)     │               │
│  │ - Concurrency: 3 parallel requests       │               │
│  │ - Iterations: 20 per run                 │               │
│  │ - Creates queue depth & KV cache usage   │               │
│  └──────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────┘
                     ↓
       ┌──────────────────────────────┐
       │ prometheus-system namespace  │
       │ - kube-prometheus-stack      │
       │ - Stores vLLM metrics with   │
       │   pod_uid label              │
       └──────────────────────────────┘
                     ↓
       ┌──────────────────────────────┐
       │ opencost namespace           │
       │ - OpenCost collector reads   │
       │   vLLM metrics by pod_uid    │
       │ - Correlates with pod costs  │
       └──────────────────────────────┘
```

## Why These Choices

### 1. Standalone vLLM (Not Full llm-d)

**Decision:** Deploy vLLM directly without llm-d router/gateway infrastructure.

**Reasoning:**
- **Single GPU:** llm-d's router optimizes multi-replica load balancing—not needed here
- **Simplicity:** vLLM provides full OpenAI API and Prometheus metrics natively
- **No dependencies:** Avoids Gateway API Inference Extension CRDs
- **Faster:** One Deployment vs. router + gateway + CRDs
- **Sufficient:** OpenCost only needs vLLM metrics endpoint, not routing features

**Tradeoff:** If we later need multi-GPU or advanced routing, we'd refactor to llm-d. For metrics validation, standalone is cleaner.

### 2. Model: Qwen/Qwen2.5-1.5B-Instruct

**Decision:** Use Qwen 2.5 1.5B instruction-tuned model.

**Reasoning:**
- **Size:** ~3.1GB FP16 fits easily on A10 24GB with ~20GB left for KV cache
- **Ungated:** No Hugging Face token required (Apache-2.0 license)
- **Quality:** Modern model from Alibaba, good instruction following
- **Fast download:** Small model means quick pod startup
- **Realistic:** Still exercises vLLM scheduler (queue, preemption, KV cache)

**Alternatives considered:**
- **Llama-3-8B** (~16GB): More realistic production size, but leaves less KV cache headroom and requires HF token (gated)
- **Larger models** (70B+): Don't fit single A10
- **Quantized models** (AWQ/GPTQ): Save VRAM but add complexity; unnecessary for validation

### 3. Load Harness: Haiku Generation

**Decision:** Simple bash script in ConfigMap that calls `/v1/chat/completions` asking for haikus.

**Reasoning:**
- **No custom image:** Uses public `curlimages/curl:8.10.1` base
- **Simple maintenance:** Bash script in ConfigMap, easy to tune concurrency/rate
- **Creates load:** 3 concurrent requests × 20 iterations = realistic queue depth
- **Small responses:** Haikus are short (~50 tokens), fast iteration
- **Variety:** Rotates through 10 topics to avoid cache hits

**Why haikus specifically:**
- Short, well-defined output format
- Easy to verify in logs
- Fast generation (seconds, not minutes)
- Fun debugging output 🙂

**Alternatives considered:**
- **Locust/k6:** Overkill, needs custom image
- **Python script:** Would need Python base image
- **curl one-shot:** Too simple, no concurrency control

### 4. Prometheus Scraping: PodMonitor

**Decision:** Use `monitoring.coreos.com/v1` PodMonitor with relabel configs.

**Reasoning:**
- **Follows repo pattern:** Matches how DCGM exporter is scraped
- **Automatic discovery:** Prometheus Operator watches PodMonitors
- **Clean relabels:** Adds `pod_uid`, `pod`, `namespace` in one place
- **Scoped:** Only scrapes vllm-inference pods

**Why not additionalScrapeConfigs:**
- PodMonitor is more Kubernetes-native
- Lives with the app manifests, not global Prometheus config
- Easier to delete when done

### 5. Configuration via ConfigMap

**Decision:** Store model name, context length, GPU settings in ConfigMap, reference via env vars.

**Reasoning:**
- **Easy tuning:** Change model without editing Deployment
- **Clear documentation:** Settings visible at top of manifest
- **GitOps-friendly:** Config changes are atomic commits

**Example to change model:**
```yaml
data:
  model: "meta-llama/Llama-3.1-8B-Instruct"  # Change here
  max_model_len: "8192"                       # Adjust context
```

## How to Verify After Deployment

### 1. Check vLLM Pod Status

```bash
# Pod should be Running
kubectl get pods -n vllm-inference
# NAME                              READY   STATUS    RESTARTS   AGE
# vllm-inference-xxxxxxxxxx-xxxxx   1/1     Running   0          5m

# Check logs for "Uvicorn running" and model loading
kubectl logs -n vllm-inference deployment/vllm-inference --tail=50
```

**Expected log snippets:**
- `INFO: Downloading model Qwen/Qwen2.5-1.5B-Instruct...`
- `INFO: Using model weights format ['*.safetensors']`
- `INFO: Uvicorn running on http://0.0.0.0:8000`

**Common issues:**
- **Pending (Insufficient GPU):** Folding@home still running; check opencost/opencost-infra#528 status
- **ImagePullBackOff:** Network/registry issue; check `kubectl describe pod`
- **CrashLoopBackOff:** OOM (unlikely on A10) or bad args; check logs

### 2. Verify vLLM API Responds

```bash
# Health check
kubectl exec -n vllm-inference deployment/vllm-inference -- \
  curl -s http://localhost:8000/health
# Expected: {"status":"ok"} or similar

# List models endpoint
kubectl exec -n vllm-inference deployment/vllm-inference -- \
  curl -s http://localhost:8000/v1/models | jq
# Expected: {"object":"list","data":[{"id":"Qwen/Qwen2.5-1.5B-Instruct",...}]}

# Generate a test completion (from within cluster)
kubectl run -it --rm debug --image=curlimages/curl:8.10.1 --restart=Never -- \
  curl -X POST http://vllm-inference.vllm-inference:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen2.5-1.5B-Instruct",
    "messages": [{"role": "user", "content": "Say hello"}],
    "max_tokens": 20
  }'
# Expected: JSON with "choices"[0]."message"."content"
```

### 3. Verify Prometheus Metrics Scraping

```bash
# Check raw vLLM metrics endpoint
kubectl exec -n vllm-inference deployment/vllm-inference -- \
  curl -s http://localhost:8000/metrics | grep ^vllm:
# Expected output (sample):
# vllm:num_requests_running 0.0
# vllm:num_requests_waiting 0.0
# vllm:kv_cache_usage_perc 0.0
# vllm:num_preemptions_total 0.0
# vllm:prompt_tokens_total 0.0
# vllm:generation_tokens_total 0.0

# Check PodMonitor exists
kubectl get podmonitor -n vllm-inference
# NAME             AGE
# vllm-inference   10m

# Verify Prometheus is scraping (requires prometheus port-forward or exec)
kubectl port-forward -n prometheus-system svc/prometheus-kube-prometheus-prometheus 9090:9090 &
curl -s 'http://localhost:9090/api/v1/query?query=vllm:num_requests_running' | jq

# CRITICAL: Verify pod_uid label is present
curl -s 'http://localhost:9090/api/v1/query?query=vllm:num_requests_running' \
  | jq '.data.result[] | select(.metric.pod_uid) | {pod: .metric.pod, pod_uid: .metric.pod_uid}'
# Expected: {"pod":"vllm-inference-xxx","pod_uid":"<uuid>"}
```

**If pod_uid is missing:**
- Check PodMonitor relabel config syntax
- Check Prometheus Operator logs: `kubectl logs -n prometheus-system deployment/prometheus-kube-prometheus-operator`
- Verify ServiceMonitorSelectorNilUsesHelmValues is false in prometheus values.yaml

### 4. Run Load Harness and Check Metrics Movement

```bash
# Trigger manual job
kubectl create job -n vllm-inference --from=cronjob/haiku-harness test-$(date +%s)
kubectl logs -n vllm-inference job/test-<timestamp> -f

# Expected output:
# Starting haiku load harness
# Endpoint: http://vllm-inference:8000/v1/chat/completions
# Concurrency: 3
# ...
# [12:34:56] Request 1: SUCCESS
# <haiku output>
# ---

# While load is running, check metrics move
watch 'curl -s "http://localhost:9090/api/v1/query?query=vllm:num_requests_running" | jq ".data.result[0].value[1]"'
# Should see non-zero values during load

# Check queue depth
curl -s 'http://localhost:9090/api/v1/query?query=vllm:num_requests_waiting' | jq

# Check KV cache usage
curl -s 'http://localhost:9090/api/v1/query?query=vllm:kv_cache_usage_perc' | jq
```

### 5. Verify OpenCost Integration (After opencost/opencost#3915 Merges)

```bash
# Check OpenCost is scraping vLLM metrics
kubectl logs -n opencost deployment/opencost-demo | grep -i vllm

# Query OpenCost API for vllm-inference namespace costs
kubectl port-forward -n opencost svc/opencost-demo 9003:9003 &
curl -s 'http://localhost:9003/allocation/compute?window=1h&aggregate=namespace' \
  | jq '.data[] | select(.name == "vllm-inference")'

# Check if vLLM metrics appear in OpenCost internal metrics
# (Exact query depends on opencost/opencost#3915 implementation)
```

## Kubectl Commands Reference

### Quick Status Check
```bash
# All vLLM resources
kubectl get all -n vllm-inference

# Current load harness jobs
kubectl get jobs -n vllm-inference

# PodMonitor status
kubectl get podmonitor -n vllm-inference -o yaml
```

### Debugging

```bash
# Describe vLLM pod (events, status)
kubectl describe pod -n vllm-inference -l app=vllm-inference

# Full logs
kubectl logs -n vllm-inference deployment/vllm-inference --all-containers

# Check GPU allocation
kubectl describe node <gpu-node-name> | grep -A10 Allocatable

# Exec into vLLM pod
kubectl exec -it -n vllm-inference deployment/vllm-inference -- /bin/bash

# Check vLLM process
kubectl exec -n vllm-inference deployment/vllm-inference -- ps aux | grep vllm
```

### Manual Testing

```bash
# Immediate haiku generation (5 iterations)
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: test-load-$(date +%s)
  namespace: vllm-inference
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: test
        image: curlimages/curl:8.10.1
        command:
        - sh
        - -c
        - |
          for i in 1 2 3 4 5; do
            curl -X POST http://vllm-inference:8000/v1/chat/completions \
              -H "Content-Type: application/json" \
              -d '{"model":"Qwen/Qwen2.5-1.5B-Instruct","messages":[{"role":"user","content":"Write a haiku about clouds"}],"max_tokens":50}' &
          done
          wait
EOF

# Check Prometheus targets
kubectl port-forward -n prometheus-system svc/prometheus-kube-prometheus-prometheus 9090:9090 &
# Open http://localhost:9090/targets and search for "vllm"
```

## Open Questions for Alex Meijer

### 1. Model Selection
**Current:** `Qwen/Qwen2.5-1.5B-Instruct` (1.5B params, ~3.1GB, Apache-2.0, ungated)

**Question:** Is this model appropriate for validation, or would you prefer:
- **Llama-3.1-8B-Instruct** for more realistic production size (requires HF token)?
- **Larger context** (current: 4096 tokens; model supports up to 32K)?
- **Different model family** (e.g., Mistral, Gemma)?

**Recommendation:** Stay with Qwen 2.5 1.5B unless you need specific model behavior for testing.

### 2. Load Pattern
**Current:** CronJob every 15 min, 3 concurrent requests × 20 iterations = 60 total requests per run

**Question:** Is this load sufficient, or should we:
- **Increase frequency** (e.g., every 5 minutes)?
- **More concurrency** (e.g., 10 parallel requests)?
- **Longer context** (current: short prompts; could use RAG-style long context)?
- **Continuous load** (Deployment instead of CronJob)?

**Recommendation:** Start with current pattern, increase if metrics aren't moving enough.

### 3. llm-d Router/Gateway
**Current:** Standalone vLLM deployment (no llm-d router)

**Question:** Should this eventually use full llm-d infrastructure (router + InferencePool + Gateway)?

**Tradeoffs:**
- **Full llm-d:** Better load balancing across replicas, prefix cache awareness, but adds CRDs and complexity
- **Standalone vLLM:** Simpler, faster to deploy, sufficient for single-GPU validation

**Recommendation:** Stay standalone unless you plan multi-GPU or need llm-d routing features.

### 4. GPU Contention & Timing
**Current:** Assumes single A10 GPU available after Folding@home removal

**Question:** Should this PR:
- **Wait** for opencost/opencost-infra#528 (Folding@home removal) to merge first?
- **Coexist** temporarily (vLLM pod will Pending until GPU free)?
- **Scale Folding@home** to 0 replicas manually?

**Recommendation:** Merge opencost/opencost-infra#528 first to avoid confusion.

### 5. Metrics Retention
**Current:** Prometheus in demo cluster has 360d retention (from values.yaml)

**Question:** After validation completes, should we:
- **Keep running** indefinitely for ongoing monitoring?
- **Remove** after opencost/opencost#3915 is verified?
- **Scale to 0** but leave manifests for future testing?

**Recommendation:** Keep running; GPU cost is minor, and it's useful for demos.

## Risk Assessment

### 🔴 High Risk: GPU Availability

**Issue:** Single A10 GPU node (`np1`) may be occupied by Folding@home.

**Impact:** vLLM pod stays `Pending` with `Insufficient nvidia.com/gpu`.

**Mitigation:**
1. Merge opencost/opencost-infra#528 first (removes Folding@home)
2. OR manually scale Folding@home: `kubectl scale -n folding-at-home deployment/fah-gpu-cpu --replicas=0`
3. OR add second A10 to `np1` pool (cost: ~$1.52/hr extra)

**Timeline:** Non-blocking for PR review; blocks actual deployment.

### 🟡 Medium Risk: Model Download Time

**Issue:** First pod startup downloads ~3.1GB model from Hugging Face.

**Impact:** Pod takes 3-5 minutes to become Ready (vs. 30s for cached).

**Mitigation:**
- Model is cached in `emptyDir` volume (persists across restarts, not recreates)
- Use `imagePullPolicy: IfNotPresent` (default) to cache vLLM image
- Consider pre-warming: deploy once, let it download, then leave running

**Timeline:** One-time delay on first deploy per node.

### 🟢 Low Risk: Prometheus Metrics Lag

**Issue:** PodMonitor scrapes every 15s; new metrics may not appear immediately.

**Impact:** First load test might show stale zeros.

**Mitigation:**
- Wait 30s after pod Ready before first load test
- Manual Job includes wait loop for vLLM health

**Timeline:** Negligible; normal scrape interval.

### 🟢 Low Risk: Namespace Cleanup

**Issue:** Nightly workflow could delete namespace if allowlist update is missed.

**Impact:** Next day, vLLM pod gone.

**Mitigation:**
- Already added to allowlist in this PR
- Workflow runs 4 AM UTC; easy to notice and redeploy

**Timeline:** Only matters if PR is partially applied.

## Cost Implications

### Current Setup
- **GPU node:** `VM.GPU.A10.1` (1x NVIDIA A10 24GB) @ ~$1.52/hr on-demand
  - Already exists for Folding@home; no new cost
  - If Folding@home removed, same node reused
- **vLLM pod:** Uses full GPU; no fractional GPU sharing
- **Load harness:** Negligible (100m CPU, 128Mi RAM)

### Monthly Cost (if dedicated)
- **24/7 on-demand:** $1.52/hr × 730hr = ~$1,110/month
- **24/7 preemptible:** $0.46/hr × 730hr = ~$336/month

### Cost Optimization Options
1. **Share with other GPU workloads:** Not practical (vLLM wants full GPU)
2. **Spot/preemptible instances:** Requires node pool config change
3. **Scale to 0 when not testing:** Manual or automated (e.g., scale down weekends)
4. **Smaller GPU:** A10 is already cheapest NVIDIA GPU on OKE

**Recommendation:** Keep running; it's a demo cluster and cost is reasonable for validation.

## Next Steps After PR Merge

1. **Wait for Folding@home removal** (opencost/opencost-infra#528)
2. **Argo CD auto-syncs** this ApplicationSet (within ~3 minutes)
3. **vLLM pod starts**, downloads model (~5 min first time)
4. **Manual verification:**
   ```bash
   kubectl get pods -n vllm-inference -w
   kubectl logs -n vllm-inference deployment/vllm-inference -f
   ```
5. **Trigger test load:**
   ```bash
   kubectl create job -n vllm-inference --from=cronjob/haiku-harness test-initial
   kubectl logs -n vllm-inference job/test-initial -f
   ```
6. **Verify metrics in Prometheus:**
   ```bash
   kubectl port-forward -n prometheus-system svc/prometheus-kube-prometheus-prometheus 9090:9090
   # Open http://localhost:9090/graph
   # Query: vllm:num_requests_running{namespace="vllm-inference"}
   ```
7. **Coordinate with OpenCost PR** (opencost/opencost#3915):
   - Share Prometheus query showing `pod_uid` label
   - Test OpenCost collector with these metrics
8. **Report back** any issues or needed tweaks

## Related Resources

- **OpenCost vLLM metrics PR:** https://github.com/opencost/opencost/pull/3915
- **PodMonitor approach:** https://github.com/opencost/opencost-helm-chart/pull/379
- **Folding@home removal:** https://github.com/opencost/opencost-infra/pull/528
- **vLLM documentation:** https://docs.vllm.ai/
- **Qwen model card:** https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct

## Support & Troubleshooting

If issues arise:

1. **Check this summary** for verification steps
2. **Review PR description** for architecture context
3. **Search issues:**
   - vLLM: https://github.com/vllm-project/vllm/issues
   - Prometheus Operator: https://github.com/prometheus-operator/prometheus-operator/issues
4. **Ask in OpenCost Slack:** #opencost-dev channel
5. **Tag @mattray or @Alex Meijer** on the PR

## Beginner-Friendly Glossary

- **vLLM:** Fast inference engine for large language models (like serving a chatbot)
- **Inference:** Using a trained AI model to generate outputs (not training it)
- **PodMonitor:** Kubernetes resource that tells Prometheus "scrape metrics from these pods"
- **Relabel:** Change metric labels (like adding `pod_uid`) during scraping
- **KV cache:** Memory storing past conversation for faster generation
- **Queue depth:** Number of requests waiting to be processed
- **Node selector:** Tell Kubernetes "only run this pod on nodes with GPUs"
- **Toleration:** Allow pod to run on nodes with special restrictions (like GPU nodes)
- **ApplicationSet:** Argo CD resource that generates Applications from templates
- **Auto-sync:** Argo CD automatically deploys changes when Git updates
