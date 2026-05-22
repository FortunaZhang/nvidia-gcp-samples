# Dynamo on GCP g4 (RTX PRO 6000 Blackwell) — Kimi K2.5 NVFP4

NVIDIA Dynamo + SGLang reference deployment for **Kimi K2.5 NVFP4** on Google Cloud's `g4-standard-384` instance (8× RTX PRO 6000 Blackwell, SM_120).

Tested on a 2-node `g4-dynamo-mn` GKE cluster with multi-NIC + jumbo MTU + local NVMe.

## What's here

| File | Purpose |
|---|---|
| `standalone-sglang-kimi-k25-nvfp4.yaml` | Bare SGLang StatefulSet (2 nodes, TP=8 + PP=2) — matches Google's published Kimi NVFP4 reference flag-for-flag, but pins SGLang to `v0.5.10.post1` (the version Dynamo's `sglang-runtime:1.1.0` image bundles) for apples-to-apples comparison |
| `dgd-agg-sglang-kimi-k25-nvfp4-parity.yaml` | Dynamo aggregated DGD — same engine flags as Standalone, `--router-mode random` (KV routing has no benefit at 1 replica) |
| `dgd-agg-sglang-kimi-k25-nvfp4-optimized.yaml` | Dynamo aggregated DGD with KV-aware routing, radix cache, KV events for shared-prefix workloads |
| `run-benchmark-parity.sh` | aiperf-based parity benchmark — same workload to both Standalone and Dynamo endpoints, locked OSL, deterministic sampling |
| `benchmark-kimi-k25-nvfp4-pod.yaml` | aiperf client pod (runs `run-benchmark-parity.sh` against either endpoint) |

## Topology

```
2× g4-standard-384  (16 GPUs total)
├── TP=8 × PP=2     (one distributed SGLang worker across both nodes)
├── DP=8            (DP attention on for MoE)
└── modelopt_fp4    (NVFP4 weights ~370 GB, bf16 KV cache, mem-fraction 0.82)
```

## Quick start

```bash
# 1. Standalone SGLang (matches Google reference)
kubectl apply -f standalone-sglang-kimi-k25-nvfp4.yaml

# 2. Dynamo parity DGD
kubectl apply -f dgd-agg-sglang-kimi-k25-nvfp4-parity.yaml

# 3. Dynamo optimized DGD (shared-prefix workload)
kubectl apply -f dgd-agg-sglang-kimi-k25-nvfp4-optimized.yaml

# 4. Benchmark client pod
kubectl apply -f benchmark-kimi-k25-nvfp4-pod.yaml

# Wait for engine ready (~12-15 min), then run a benchmark
kubectl cp run-benchmark-parity.sh perf-kimi-k25-nvfp4:/workspace/
kubectl exec perf-kimi-k25-nvfp4 -- chmod +x /workspace/run-benchmark-parity.sh
kubectl exec perf-kimi-k25-nvfp4 -- bash -c \
  'nohup setsid /workspace/run-benchmark-parity.sh standalone > /workspace/bench.log 2>&1 &'
```

Each YAML and script has inline comments explaining the choices.

## Reference

Google's published Kimi K2.5 NVFP4 reference: <https://github.com/shivajid/sglang-rtx-pro-6000/tree/main/models/KimiK2.5/nvfp4>

Headline numbers (workload: ISL=1024, OSL=8192, conc=512, 1,536 prompts):

| Variant | Throughput | TTFT P50 | ITL P50 |
|---|---|---|---|
| Google Standalone (reference) | 3,237 tok/s | 304 ms | 121 ms |
| NVIDIA Standalone (this repo) | **3,374 tok/s** (+4.2%) | **288 ms** (-5%) | **118 ms** (-2.5%) |
| Dynamo parity | 3,723 tok/s | (locked-OSL methodology, not directly comparable to Google's variable-OSL numbers) | 128 ms |
| Dynamo optimized (best, bench_serving, 80% shared prefix) | **3,975 tok/s** (+22.8% vs Google) | latency-focused win | — |
