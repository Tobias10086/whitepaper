Raw Report

# EDA Report for Base Model and Three Chutes Models

This report is based on the data collected **up to 2026-04-07 UTC**. It prioritizes benchmarks with stable, finalized outputs for the main conclusions, and keeps incomplete or snapshot-only runs in separate sections.

Models covered:

- base_model (Qwen/Qwen3-32B-TEE)
- axon1_affine_m19_model
- leary_comos_affine_cx_model
- leary_comos_affine_cs_model

## Executive Summary

- The most reliable cross-model comparisons currently come from:
  - BrowseComp-ZH
  - MCP-Bench
  - MemoryAgentBench
- On BrowseComp-ZH, Leary CX is clearly the strongest model by final accuracy.
- On MCP-Bench, Leary CX and Leary CS form the top tier, while base_model is still strong on tool-call validity and overall tool stability.
- On MemoryAgentBench, all three Chutes models outperform base_model on retrieval quality, and they are also much faster.
- ToolSandbox and Tau2 are still best treated as directional snapshots rather than final leaderboard-grade results.
- BFCL and LoCoMo are not complete yet and should not be used for strong model ranking.

## Stable Benchmarks

### Summary Table

| Model | BrowseComp-ZH Accuracy | MCP Tool Call Success | MemoryAgentBench F1 |
| --- | --- | --- | --- |
| Base (Qwen3-32B-TEE) | 6.92% | 98.01% | 6.21 |
| Axon1 M19 | 2.77% | 87.14% | 9.30 |
| Leary CX | 9.34% | 98.21% | 9.37 |
| Leary CS | 3.11% | 96.95% | 8.73 |

!BrowseComp-ZH Accuracy

!MCP-Bench Metrics

!MemoryAgentBench Quality

!MemoryAgentBench Latency

## Benchmark-by-Benchmark Analysis

### 1. BrowseComp-ZH

This is currently the cleanest finalized reading-comprehension / retrieval QA comparison across all four models.

| Model | Samples | Correct | Accuracy | Avg Latency (s) | Avg Output Tokens | Timeout Count |
| --- | --- | --- | --- | --- | --- | --- |
| Base (Qwen3-32B-TEE) | 289 | 20 | 6.92% | 91.44 | 4050.7 | 0 |
| Axon1 M19 | 289 | 8 | 2.77% | 11.52 | 272.6 | 0 |
| Leary CX | 289 | 27 | 9.34% | 115.98 | 1871.2 | 0 |
| Leary CS | 289 | 9 | 3.11% | 44.41 | 212.7 | 0 |

Key readouts:

- Leary CX is the strongest model here in final answer accuracy.
- base_model is still competitive on Chinese long-form retrieval QA, clearly above Axon1 and Leary CS.
- Leary CX achieves the best accuracy, but it does so with the highest latency and a very large output footprint.
- Leary CS is much shorter and faster than Leary CX, but the shorter output does not translate into better accuracy.

### 2. MCP-Bench

This is currently the strongest and most stable tool-use benchmark across all four models.

| Model | Task Success Rate | Tool Call Success Rate | Valid Tool Name Rate | Task Completion Score | Tool Selection Score | Planning Score | Avg Agent Time (s) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Base (Qwen3-32B-TEE) | 100.00% | 98.01% | 100.00% | 6.78 | 7.76 | 6.69 | 193.94 |
| Axon1 M19 | 100.00% | 87.14% | 92.12% | 5.86 | 6.23 | 5.16 | 191.48 |
| Leary CX | 100.00% | 98.21% | 99.80% | 7.71 | 8.07 | 6.94 | 86.79 |
| Leary CS | 100.00% | 96.95% | 99.25% | 7.93 | 8.24 | 6.95 | 125.18 |

Key readouts:

- Leary CX and Leary CS are the clear top tier on end-to-end tool-use quality.
- base_model is surprisingly strong on tool-call hygiene: perfect valid tool names and near-top tool-call success.
- Axon1 is the weakest in tool-use stability. Its task success is still nominally perfect, but its intermediate tool-use quality is noticeably worse.
- Leary CX combines top-tier scores with the best runtime profile in this benchmark.

### 3. MemoryAgentBench

This section uses the Accurate_Retrieval / longmemeval_s* setting. All four models produced complete result files.

| Model | F1 | Substring EM | ROUGE-L F1 | ROUGE-L Recall | Avg Query Time (s) | Avg Output Tokens | Samples |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Base (Qwen3-32B-TEE) | 6.21 | 5.67 | 6.23 | 19.39 | 16.14 | 50.00 | 300 |
| Axon1 M19 | 9.30 | 9.67 | 9.55 | 24.41 | 2.89 | 37.31 | 300 |
| Leary CX | 9.37 | 11.00 | 9.71 | 22.80 | 2.60 | 26.24 | 300 |
| Leary CS | 8.73 | 8.67 | 9.33 | 20.54 | 6.57 | 23.61 | 300 |

Key readouts:

- All three Chutes models are clearly stronger than base_model on retrieval-focused long-context QA.
- Leary CX is the most balanced model here, with the strongest exact-match-like behavior and the strongest ROUGE-L F1.
- Axon1 has the highest ROUGE-L recall, which suggests it is especially good at surfacing relevant supporting content.
- base_model is not only weaker in answer quality, it is also dramatically slower on a per-query basis.

## Snapshot Benchmarks

These benchmarks are useful, but they should not be interpreted with the same confidence level as the stable section above.

### ToolSandbox Snapshot

At the time of writing, Leary CS did not yet have a stable summary file comparable to the other three, so this section only covers base, Axon1, and Leary CX.

| Model | Covered Scenarios | ALL_CATEGORIES Similarity | Avg Turn Count |
| --- | --- | --- | --- |
| Base (Qwen3-32B-TEE) | 926 | 0.0694 | 27.05 |
| Axon1 M19 | 898 | 0.4808 | 18.02 |
| Leary CX | 890 | 0.4542 | 17.12 |

!ToolSandbox Snapshot

Interpretation:

- Axon1 and Leary CX are much stronger than base_model in multi-tool interaction quality.
- base_model has slightly broader coverage at this point, but the resulting interaction quality is still much lower.
- This section should be read as a progress snapshot, not a final ranking.

### Tau2 Snapshot

| Model | Completed Simulations | Successes | Snapshot pass_hat_1 |
| --- | --- | --- | --- |
| Base (Qwen3-32B-TEE) | 115 | 50 | 0.4348 |
| Axon1 M19 | 200 | 17 | 0.0850 |
| Leary CX | 200 | 22 | 0.1100 |
| Leary CS | 107 | 14 | 0.1308 |

!Tau2 Snapshot

Interpretation:

- The base_model number is from an earlier incomplete historical run and should not be compared directly to finalized formal runs.
- Among the three Chutes models, Leary CX and Leary CS look slightly stronger than Axon1 in the current snapshot.
- Since not every run is equally complete, this section should remain directional only.

## Incomplete or Partially Available Benchmarks

### Status Table

| Benchmark | Status | Notes |
| --- | --- | --- |
| BFCL | Incomplete | Only partial *_result.json files are available |
| LoCoMo | Running / restarting | The category-5 KeyError has been fixed and runs were restarted |
| LongMemEval | Generated | Generation finished, but a unified evaluation layer has not yet been incorporated into this report |

### BFCL Raw Completion Snapshot

| Model | Materialized *_result.json Files | Interpretation |
| --- | --- | --- |
| Base (Qwen3-32B-TEE) | 1 | Too incomplete for ranking |
| Axon1 M19 | 18 | Still incomplete |
| Leary CX | 18 | Still incomplete |
| Leary CS | 4 | Still incomplete |

## Additional Raw Data

### Stable Benchmark Source Files

#### BrowseComp-ZH

- Base: summary.json
- Axon1: summary.json
- Leary CX: summary.json
- Leary CS: summary.json

#### MCP-Bench

- Base: base_model_results.json
- Axon1: axon1_affine_m19_model_results.json
- Leary CX: leary_comos_affine_cx_model_results.json
- Leary CS: leary_comos_affine_cs_model_results.json

#### MemoryAgentBench

- Base: results.json
- Axon1: results.json
- Leary CX: results.json
- Leary CS: results.json

### Snapshot Benchmark Source Files

#### ToolSandbox

- Base: result_summary.json
- Axon1: result_summary.json
- Leary CX: result_summary.json

#### Tau2

- Base snapshot: results.json
- Axon1: summary.json
- Leary CX: summary.json
- Leary CS snapshot: results.json


#### SWE-rebench

Axon1 M19 SWE-rebench 0 HumanEval 72.56 Tau2-bench 21.34
Leary CS SWE-rebench 7.02 HumanEval 88.41 Tau2-bench 21.43
Leary CX SWE-rebench 12.28 HumanEval 89.02 Tau2-bench 7.76
Qwen 32b SWE-rebench 0 HumanEval 81.71 Tau2-bench 14.05


### Generated but Not Yet Unified

#### LongMemEval

- Base: generation_logs/base_model
- Axon1: generation_logs/axon1_affine_m19_model
- Leary CX: generation_logs/leary_comos_affine_cx_model
- Leary CS: generation_logs/leary_comos_affine_cs_model

## Final Takeaways

- If the decision must be based on the most trustworthy completed benchmarks available right now, Leary CX is the strongest overall model in this group.
- Leary CS is very strong on tool-use tasks, but it is materially weaker than Leary CX on Chinese retrieval QA.
- Axon1 remains competitive on long-context memory and agent-style tasks, but it is less stable than the Leary models on tool-use execution.
- base_model is still a respectable baseline, especially in Chinese retrieval QA and MCP tool-call correctness, but it trails the Chutes models on more complex long-context and agentic workloads.

This document intentionally separates finalized evidence from in-progress signals. Snapshot benchmarks should be used for hypothesis generation, not for strong leaderboard claims.