# Dynamo on GCP g4 (RTX PRO 6000 Blackwell) — Kimi K2.5 NVFP4

NVIDIA Dynamo + SGLang reference deployment for **Kimi K2.5 NVFP4** on Google Cloud's `g4-standard-384` instance (8× RTX PRO 6000 Blackwell, SM_120). 2-node GKE topology, TP=8 + PP=2.

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
2× g4-standard-384  (16 GPUs total, RTX PRO 6000 Blackwell SM_120)
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

## Performance Benchmarks

Workload: ISL=1024, OSL=8192, conc=512, 1,536 prompts. **bold** = directly comparable column.

| Variant | SGLang version | Benchmark / OSL | Output Throughput (tok/s) | Total Throughput (tok/s) | ITL P50 |
|---|---|---|---|---|---|
| Google Standalone (published reference) | `lmsysorg/sglang:dev-cu13` | bench_serving, variable OSL | 3,237 | 3,632 | 121 ms |
| **NVIDIA Standalone** (matches Google methodology) | `lmsysorg/sglang:dev-cu13` | bench_serving, variable OSL | **3,374** (+4.2%) | **~3,786** | **118 ms** (-2.5%) |
| **NVIDIA Standalone (aiperf fixed-OSL baseline)** ← Dynamo apples-to-apples reference | `lmsysorg/sglang:v0.5.10.post1` | aiperf, **locked OSL=8192** | **3,971** | **4,480** | 122.6 ms |
| Dynamo parity — completed | `v0.5.10.post1` (bundled in `sglang-runtime:1.1.0`) | aiperf, locked OSL=8192 | 3,723 | 4,200 | 128.0 ms |
| Dynamo optimized — work in progress | `v0.5.10.post1` (bundled in `sglang-runtime:1.1.0`) | aiperf, locked OSL=8192, shared-prefix workload | — | — | — |

*SGLang version note*: The Standalone fixed-OSL baseline (row 3) is intentionally pinned to `v0.5.10.post1` — the same SGLang version bundled in Dynamo's certified `sglang-runtime:1.1.0` image — so the Dynamo parity comparison (row 4) holds the SGLang code constant and isolates the wrapper effect from upstream SGLang version drift. Rows 1-2 use the rolling `dev-cu13` tag to match Google's published methodology.

**Reading guide:**
- **Goal 1** (NVIDIA Standalone vs Google): direct apples-to-apples — both use `bench_serving` + variable OSL. NVIDIA Standalone matches and slightly exceeds Google's reference on every metric.
- **Goal 2** (Dynamo parity vs NVIDIA Standalone fixed-OSL baseline): completed. Both runs use `aiperf` + locked OSL + the same SGLang version that's bundled in Dynamo's certified runtime image, so the comparison isolates the Dynamo wrapper from engine config differences. Throughput within ~6% of the baseline.
- **Goal 3** (Dynamo optimized): work in progress. Dynamo's primary value-add is the KV-aware router + radix cache on shared-prefix workloads, which is the next focus.
