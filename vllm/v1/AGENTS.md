# vLLM V1 Knowledge Base

This guide applies only under `vllm/v1`. The repository root `AGENTS.md`
still controls contribution policy, environment setup, and global test rules.

## Overview

`vllm/v1` is the re-architected runtime path for scheduling, KV cache
management, worker execution, sampling, and serving integration. It is organized
by runtime responsibility, not by a flat public API layer.

## Structure

```text
vllm/v1/
|-- engine/            # front-end clients, EngineCore loop, IPC, DP coordination
|-- core/              # scheduler, KV cache manager, block pool, cache groups
|-- worker/            # device workers, GPU model runners, input batches
|-- attention/         # backend contracts, selection, metadata, kernels
|-- spec_decode/       # EAGLE, ngram, medusa, draft-model proposers
|-- kv_offload/        # scheduler/worker offload managers and policies
|-- simple_kv_offload/ # eager-mode CPU offload path
|-- structured_output/ # grammar backends and bitmask management
|-- metrics/           # scheduler/cache/perf stats and loggers
|-- sample/            # sampler, rejection sampler, logits processors
`-- executor/          # uniproc, multiproc, Ray executor boundary
```

## Where To Look

| Task | Location | Notes |
| --- | --- | --- |
| Follow a request end to end | `engine/llm_engine.py`, `engine/async_llm.py`, `engine/core.py` | Front-end entry, core loop, scheduler handoff |
| Study scheduler/KV cache | `core/sched/scheduler.py`, `core/kv_cache_manager.py`, `core/kv_cache_coordinator.py`, `core/block_pool.py` | See `core/AGENTS.md` first |
| Trace worker execution | `worker/gpu_worker.py`, `worker/gpu_model_runner.py`, `worker/gpu/model_runner.py` | See `worker/AGENTS.md`; `worker/gpu/` is experimental V2 |
| Check attention backend behavior | `attention/selector.py`, `attention/backend.py`, `attention/backends/` | See `attention/AGENTS.md`; selector may set global KV layout |
| Follow distributed startup | `engine/utils.py`, `engine/core_client.py`, `engine/coordinator.py`, `executor/` | V1 is multi-process by default |
| Follow structured output | `structured_output/__init__.py`, `structured_output/backend_types.py` | One backend is initialized and reused in V1 |
| Follow spec decode | `spec_decode/eagle.py`, `spec_decode/metadata.py`, `worker/gpu_model_runner.py` | EAGLE couples runner buffers, attention metadata, and scheduler slots |
| Follow KV offload | `kv_offload/`, `simple_kv_offload/`, `core/sched/scheduler.py` | Offload paths bridge scheduler-side state and worker transfer handlers |
| Follow KV transfer connectors | `core/sched/scheduler.py`, `worker/kv_connector_model_runner_mixin.py`, `vllm/distributed/kv_transfer/` | Connector-backed external tokens, invalid block recovery, and disaggregated-prefill flows cross the v1 boundary |
| Follow metrics | `metrics/stats.py`, `metrics/perf.py`, `metrics/loggers.py` | Cache metrics preserve newest non-empty samples |

## Code Map

| Symbol | Type | Location | Role |
| --- | --- | --- | --- |
| `LLMEngine` | class | `engine/llm_engine.py` | Synchronous v1 engine facade |
| `AsyncLLM` | class | `engine/async_llm.py` | Async facade and output handler lifecycle |
| `EngineCore` | class | `engine/core.py` | Inner schedule/execute/update loop |
| `EngineCoreProc` | class | `engine/core.py` | Process + socket wrapper around `EngineCore` |
| `Scheduler` | class | `core/sched/scheduler.py` | Admission, batching, preemption, connector updates |
| `KVCacheManager` | class | `core/kv_cache_manager.py` | Scheduler-facing KV allocation facade |
| `KVCacheCoordinator` | class | `core/kv_cache_coordinator.py` | Fans KV work across cache groups |
| `BlockPool` | class | `core/block_pool.py` | Physical block ownership, prefix-cache table, eviction |
| `Worker` | class | `worker/gpu_worker.py` | GPU worker lifecycle, profiling, KV cache allocation |
| `GPUModelRunner` | class | `worker/gpu_model_runner.py` | Main V1 GPU execution path |
| `AttentionBackend` | class | `attention/backend.py` | Attention implementation contract |
| `AttentionBackendEnum` | enum | `attention/backends/registry.py` | Backend name to import path registry |

## Conventions

- `engine/__init__.py` is a wire/data-model module, not just package glue.
- `KVCacheBlocks.blocks` is grouped by KV cache group, not by token position.
- `BlockPool.null_block` is block `0`; it stands for skipped/out-of-window slots and is not normal free-list state.
- Prefix cache hits are block-aligned; full prompt cache hits intentionally recompute the last token for logits.
- Backend selection can call `set_kv_cache_layout()` as a process-global side effect.
- Data carriers use `*Metadata`, `*Stats`, `*Spec`, `*Config`, and `*Output` naming; avoid inventing parallel names.
- Tests mirror source areas mostly under `tests/v1/<area>/`, with additional kernel/e2e coverage for attention, worker, and serving paths.

## Anti-Patterns

- Do not port V0-only behavior into V1 without checking `docs/usage/v1_guide.md`; V0 is deprecated and some features were removed.
- Do not assume prompt and decode are separate scheduler phases; V1's scheduler budgets prompt and output tokens uniformly.
- Do not add request-level structured-output backend switching; V1 initializes one backend path and reuses it.
- Do not treat `worker/gpu/` as a stable replacement for `worker/gpu_model_runner.py`; it is experimental Model Runner V2.
- Do not change attention backend layout or stride assumptions without checking worker KV tensor allocation and attention tests.
- Do not rely on LSP diagnostics until the relevant server is available; fall back to `rg` and focused tests when tooling is unavailable.

## Commands

```bash
# KV cache and scheduler focus
.venv/bin/python -m pytest tests/v1/core/test_kv_cache_utils.py tests/v1/core/test_single_type_kv_cache_manager.py tests/v1/core/test_prefix_caching.py tests/v1/core/test_scheduler.py -v

# Worker and attention focus
.venv/bin/python -m pytest tests/v1/worker/test_gpu_model_runner.py tests/v1/worker/test_gpu_input_batch.py tests/v1/attention/test_attention_backends.py tests/v1/attention/test_attention_splitting.py -v

# Engine focus
.venv/bin/python -m pytest tests/v1/engine/test_engine_core.py tests/v1/engine/test_engine_core_client.py tests/v1/engine/test_async_llm.py -v
```

## Notes

- Process architecture: API server processes talk to engine core processes over ZMQ; each engine core owns scheduler/KV cache and dispatches to GPU workers.
- With DP, there is one engine core per DP rank and a coordinator process when `data_parallel_size > 1`.
- CUDA graph capture and memory profiling affect KV cache sizing; worker profiling intentionally runs after distributed initialization.
- Connector-backed KV cache flows depend on `vllm/distributed/kv_transfer` even though scheduler/worker integration points live under `vllm/v1`.
