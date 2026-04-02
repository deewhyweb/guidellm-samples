# gpt-oss-20b Benchmark: vLLM vs llm-d

A/B benchmark comparing standard vLLM (round-robin routing) against llm-d (intelligent prefix-cache-aware routing) using GuideLLM.

## Directory Structure

```
vllm/           # Baseline vLLM deployment (round-robin)
llm-d/          # llm-d deployment (intelligent scheduling)
guidellm/       # GuideLLM benchmark jobs + data PVC
```

## Prerequisites

- OpenShift cluster with GPU nodes
- `oc` CLI logged in with cluster-admin
- Access to the `gpt-oss-20b` model
- The prompts file at `test-data-generator/prefix/prompts.csv` (from llm-d-playbooks)

## Step 1: Deploy vLLM (Baseline)

```bash
oc apply -f vllm/serving-runtime.yaml
oc apply -f vllm/inference-service.yaml
oc apply -f vllm/service.yaml
```

Wait for all 4 replicas to be ready:

```bash
oc get pods -n gpt-oss-20b-benchmark -l serving.kserve.io/inferenceservice=gpt-oss-20b-vllm -w
```

## Step 2: Deploy llm-d (Intelligent Scheduler)

```bash
oc apply -f llm-d/gateway.yaml
oc apply -f llm-d/hardware-profile.yaml
oc apply -f llm-d/llm-inference-service.yaml
```

Wait for all 4 replicas to be ready:

```bash
oc get pods -n gpt-oss-20b-benchmark -l llm-d.ai/inferenceservice=gpt-oss-20b -w
```

## Step 3: Upload Prompts to PVC

Create the benchmark namespace and PVC, then upload the prompts file:

```bash
oc apply -f guidellm/pvc.yaml

# Start the data loader pod
oc apply -f guidellm/data-loader-pod.yaml
oc wait --for=condition=Ready pod/benchmark-data-loader -n gpt-oss-20b-benchmarks --timeout=120s

# Copy prompts.csv into the PVC
oc cp /Users/phayes/projects/llm-d-playbooks/08-benchmarking/intelligent-inference-scheduler/test-data-generator/prefix/prompts.csv \
  gpt-oss-20b-benchmarks/benchmark-data-loader:/data/prompts.csv

# Verify the file was copied
oc exec -n gpt-oss-20b-benchmarks benchmark-data-loader -- ls -la /data/prompts.csv

# Delete the loader pod (PVC data persists)
oc delete pod benchmark-data-loader -n gpt-oss-20b-benchmarks
```

## Step 4: Run GuideLLM Benchmarks

Run benchmarks against vLLM and llm-d (can run sequentially or in parallel):

```bash
# Benchmark against vLLM baseline
oc apply -f guidellm/guidellm-vllm.yaml

# Benchmark against llm-d
oc apply -f guidellm/guidellm-llm-d.yaml
```

Monitor progress:

```bash
oc logs -f job/guidellm-benchmark-vllm -n gpt-oss-20b-benchmarks
oc logs -f job/guidellm-benchmark-llm-d -n gpt-oss-20b-benchmarks
```

## Step 5: Collect Results

Results are written to the job pods. Copy them out before the TTL (24h) expires:

```bash
# Get the pod names
VLLM_POD=$(oc get pods -n gpt-oss-20b-benchmarks -l job-name=guidellm-benchmark-vllm -o jsonpath='{.items[0].metadata.name}')
LLMD_POD=$(oc get pods -n gpt-oss-20b-benchmarks -l job-name=guidellm-benchmark-llm-d -o jsonpath='{.items[0].metadata.name}')

# Copy results
oc cp gpt-oss-20b-benchmarks/$VLLM_POD:/results ./results-vllm
oc cp gpt-oss-20b-benchmarks/$LLMD_POD:/results ./results-llm-d
```

## Benchmark Configuration

| Parameter | Value |
|-----------|-------|
| Model | gpt-oss-20b |
| Replicas | 4 |
| Concurrency levels | 32, 64 |
| Max duration | 3000s (50 min) |
| Data source | prompts.csv (prefix-cache workload) |

## Key Metrics to Compare

| Metric | Description |
|--------|-------------|
| TTFT | Time to first token (lower is better) |
| ITL | Inter-token latency (lower is better) |
| TPOT | Time per output token (lower is better) |
| KV Cache Hit Rate | Higher means more prefix reuse |

## Cleanup

```bash
oc delete -f guidellm/guidellm-vllm.yaml
oc delete -f guidellm/guidellm-llm-d.yaml
oc delete -f guidellm/pvc.yaml
oc delete -f llm-d/llm-inference-service.yaml
oc delete -f llm-d/hardware-profile.yaml
oc delete -f llm-d/gateway.yaml
oc delete -f vllm/service.yaml
oc delete -f vllm/inference-service.yaml
oc delete -f vllm/serving-runtime.yaml
```
