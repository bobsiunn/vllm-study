# vLLM V1 Worker Guide

Use this before changing `vllm/v1/worker`. Worker code is the handoff between
scheduled requests, GPU memory, model execution, sampling, and attention metadata.

## Overview

`gpu_worker.py` owns worker lifecycle and memory profiling. `gpu_model_runner.py`
is the main V1 execution path. `worker/gpu/` contains experimental Model Runner
V2; follow its README before touching it.

## Where To Look

| Task | Location | Notes |
| --- | --- | --- |
| Worker lifecycle | `gpu_worker.py`, `worker_base.py` | Device init, profiling, cache allocation, warmup |
| Main V1 runner | `gpu_model_runner.py` | Input prep, execution, sampling, spec decode, KV tensors |
| Experimental V2 runner | `gpu/model_runner.py`, `gpu/README.md` | Active development; do not treat as feature-complete |
| Batch state | `gpu_input_batch.py`, `block_table.py` | Request rows, block tables, compaction, slot mapping |
| Shared worker utilities | `utils.py`, `gpu/attn_utils.py` | KV cache binding, tensor allocation/reshape, attention groups |
| Platform variants | `cpu_worker.py`, `xpu_worker.py`, `tpu_input_batch.py` | Device-specific worker constraints |

## Conventions

- Initialize distributed/NCCL state before memory snapshots; profiling assumes no other process releases memory during the profile window.
- `Worker.initialize_from_config()` initializes KV transfer before model runner KV cache setup so connector state sees the scheduler config.
- V1 sampler warmup runs after CUDA graph capture to avoid losing buffers to `torch.accelerator.empty_cache()`.
- `InputBatch`, `CachedRequestState`, block tables, and runner request state must stay synchronized across add, finish, preempt, and compact operations.
- Persistent buffers and async copy events are part of correctness. Do not reallocate or mutate pinned CPU buffers while GPU reads are pending.
- KV cache zeroing metadata is allocated outside the CuMem KV cache pool so sleep/wake does not discard bookkeeping tensors.
- `gpu_model_runner.py` supports many feature paths; prefer local helpers over adding more top-level coupling.
- V2 runner work should preserve modularity and clean boundaries rather than quickly porting V1 behavior.

## Attention And KV Handoff

- Worker allocates and reshapes KV cache tensors according to attention backend shape/stride contracts.
- `initialize_attn_backend()` and metadata builders must agree with `KVCacheConfig.kv_cache_groups`.
- `CommonAttentionMetadata` is the shared worker-to-attention contract; ubatching and spec decode must preserve aligned CPU/GPU views.
- Block-table padding and hybrid `block_size` vs `kernel_block_size` translation are fragile; update attention tests with worker changes.

## Anti-Patterns

- Do not assume V2 runner supports every V1 path; `gpu_worker.py` still selects V1 by default for unsupported cases.
- Do not insert unconditional device synchronizations in hot paths; async scheduling depends on CPU/GPU overlap.
- Do not change profiling, warmup, or capture order without checking KV cache sizing and CUDA graph memory accounting.
- Do not change request compaction without checking block tables, sampling metadata, and cached request state together.
- Do not touch `worker/gpu/` without treating it as experimental active-development code.

## Tests

```bash
.venv/bin/python -m pytest tests/v1/worker/test_gpu_model_runner.py tests/v1/worker/test_gpu_input_batch.py tests/v1/worker/test_utils.py -v
.venv/bin/python -m pytest tests/v1/streaming_input/test_gpu_model_runner_streaming.py tests/v1/streaming_input/test_gpu_model_runner_v2_streaming.py -v
```
