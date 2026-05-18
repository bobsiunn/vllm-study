# vLLM V1 Attention Guide

Use this before changing `vllm/v1/attention`. Attention code is split between a
shared backend contract, process-global backend selection, metadata builders, and
hardware-specific kernels.

## Overview

Worker code builds common metadata and KV tensors; attention backends define how
that metadata is interpreted and which KV cache layout/stride is legal.

## Where To Look

| Task | Location | Notes |
| --- | --- | --- |
| Backend contract | `backend.py` | `AttentionBackend`, metadata builders, common metadata |
| Backend selection | `selector.py` | Platform selection, cached lookup, KV layout side effect |
| Registry/overrides | `backends/registry.py` | Enum names to class paths, custom backend registration |
| Shared helpers | `backends/utils.py` | KV layout, per-layer params, local-attn splitting |
| Common implementations | `backends/flash_attn.py`, `backends/flashinfer.py`, `backends/triton_attn.py` | Backend-specific shape/stride/support rules |
| MLA implementations | `backends/mla/` | Dense and sparse MLA backend families |
| Triton ops | `ops/` | Kernel wrappers and attention helper kernels |

## Conventions

- Backend enum names are user-visible selectors; keep names stable and update tests when adding values.
- `register_backend()` overrides import paths at runtime. `CUSTOM` must be registered before use.
- `selector._cached_get_attn_backend()` may call `set_kv_cache_layout()`; this mutates process-global layout state.
- Backend `get_required_kv_cache_layout()` must match worker KV tensor allocation and reshape logic.
- Metadata builders convert `CommonAttentionMetadata` into backend-specific metadata; keep slicing/ubatching semantics aligned.
- FlashInfer, FlashAttention, Triton, ROCm, XPU, CPU, Mamba, and MLA backends do not share identical shape/stride/support assumptions.
- Mamba-like backend additions must update `MAMBA_TYPE_TO_BACKEND_MAP` and the mamba enum together.

## Anti-Patterns

- Do not import backend classes eagerly in selection code unless the current path already requires it; lazy import avoids optional backend side effects.
- Do not change global KV cache layout for one backend without checking other selected layers in the same process.
- Do not assume all attention backends support cascade attention, sinks, sliding window, MLA, FP8, or CUDA graph capture.
- Do not add backend-specific fields to `CommonAttentionMetadata`; use backend metadata classes/builders.
- Do not update attention kernels without checking worker KV cache tensor allocation and block-table tests.

## Tests

```bash
.venv/bin/python -m pytest tests/v1/attention/test_attention_backends.py tests/v1/attention/test_attention_backends_selection.py tests/v1/attention/test_batch_reordering.py -v
.venv/bin/python -m pytest tests/v1/attention/test_attention_splitting.py tests/v1/attention/test_chunked_local_attention.py tests/v1/attention/test_mla_backends.py tests/v1/attention/test_sparse_mla_backends.py -v
.venv/bin/python -m pytest tests/kernels/attention/test_attention_selector.py tests/kernels/attention/test_flashinfer.py tests/kernels/attention/test_flashmla.py -v
```
