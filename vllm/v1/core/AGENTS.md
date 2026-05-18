# vLLM V1 Core Guide

Use this before changing `vllm/v1/core` or `vllm/v1/core/sched`. This is the
highest-value area for studying the V1 KV cache manager.

## Overview

`core` owns scheduling policy plus KV/encoder cache bookkeeping. The scheduler
decides what to run; the KV cache stack decides which token slots and blocks back
that decision.

## Where To Look

| Task | Location | Notes |
| --- | --- | --- |
| Main scheduling loop | `sched/scheduler.py` | Admission, queues, token budget, preemption, connector updates |
| Async scheduling | `sched/async_scheduler.py` | Output placeholders and async spec-token behavior |
| Scheduler payloads | `sched/output.py` | New/cached request data sent to workers |
| Queue policy | `sched/request_queue.py` | FCFS and priority queue implementations |
| KV facade | `kv_cache_manager.py` | Scheduler-facing computed-block lookup and allocation |
| Group coordination | `kv_cache_coordinator.py` | No-prefix, unitary, and hybrid group fan-out |
| Per-type managers | `single_type_kv_cache_manager.py` | Full, sliding, chunked local, Mamba, cross-attn rules |
| Physical blocks | `block_pool.py`, `kv_cache_utils.py` | Block hashes, free queue, prefix-cache table, sizing |
| Encoder cache | `encoder_cache_manager.py` | Multimodal encoder output cache, not token KV cache |

## Core Flow

1. `Scheduler.schedule()` forms `SchedulerOutput` and asks `KVCacheManager` for computed blocks and slots.
2. `KVCacheManager` delegates group-specific decisions to `KVCacheCoordinator`.
3. Coordinators call `SingleTypeKVCacheManager` implementations for block math and cache hits.
4. `BlockPool` owns `KVCacheBlock` instances, prefix-cache hashes, free-list order, and KV events.
5. `Scheduler.update_from_output()` consumes worker output, frees or updates request/cache state, then emits `EngineCoreOutputs`.

## Conventions

- `KVCacheBlocks.blocks` outer dimension is KV cache group. Do not index it as token-major.
- `KVCacheManager.empty_kv_cache_blocks` is intentionally reused to reduce GC churn.
- `BlockPool.null_block` is block `0`; do not free it or treat it as a normal allocation.
- Free request blocks in reverse order so tail blocks become earliest eviction candidates.
- Prefix cache hits are block-aligned; full prompt hits recompute the last token so logits can be produced.
- `UnitaryKVCacheCoordinator` assumes one KV cache group and, when caching, `hash_block_size == block_size` after DCP/PCP scaling.
- `HybridKVCacheCoordinator` requires every group block size to be divisible by `hash_block_size` and currently rejects DCP/PCP > 1.
- Hybrid cache lookup checks full attention first and aligns to the LCM of group block sizes.
- `CrossAttentionManager` is for encoder-decoder cross-attention allocation; do not add prefix-caching semantics there without tests.
- `SchedulerOutput.new_block_ids_to_zero` is consumed by workers so newly allocated GPU blocks are zeroed before reuse.

## Anti-Patterns

- Do not bypass `KVCacheManager` from scheduler code to mutate `BlockPool` directly.
- Do not cache invalid connector-reported blocks; block IDs from workers must be scheduler-allocated.
- Do not analyze connector-backed KV flows only inside `vllm/v1`; connector interfaces and implementations live under `vllm/distributed/kv_transfer`.
- Do not assume encoder-decoder models can use KV connectors or chunked prefill; tests guard disabled paths.
- Do not weaken block hash checks to support mixed specs; same-group layers must have compatible specs.
- Do not add metrics by reusing one-time scheduling snapshots after preemption unless the metric tracks first-time values separately.

## Tests

```bash
.venv/bin/python -m pytest tests/v1/core/test_kv_cache_utils.py tests/v1/core/test_single_type_kv_cache_manager.py tests/v1/core/test_prefix_caching.py tests/v1/core/test_kv_cache_metrics.py -v
.venv/bin/python -m pytest tests/v1/core/test_scheduler.py tests/v1/core/test_async_scheduler.py tests/v1/core/test_priority_scheduler_random.py -v
```
