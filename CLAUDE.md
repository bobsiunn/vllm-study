# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

## What this repository is

This is a working copy of `vllm-project/vllm` (a fast LLM inference and serving
engine) being used as a **study repository**, with focus on the V1 runtime —
especially the KV cache manager and scheduler under `vllm/v1/core`. Korean study
guides authored in this fork live as top-level `*_study_guide_ko.md` files and in
recent commits; they are notes, not part of the engine.

The root `AGENTS.md` (imported above) is authoritative for contribution policy and
environment/test setup. Do not open low-value or duplicate PRs against upstream;
see its Contribution Policy section.

## Layered AGENTS.md knowledge bases (read these first)

The most valuable architecture docs are nested, area-specific `AGENTS.md` files.
Read the relevant one **before** editing code in that subtree — they encode
invariants that are easy to break and hard to rediscover:

- `vllm/v1/AGENTS.md` — V1 runtime overview, "where to look" table, process model.
- `vllm/v1/core/AGENTS.md` — scheduler + KV cache stack (highest-value area here).
- `vllm/v1/worker/AGENTS.md` — worker lifecycle, GPU model runner, KV/attention handoff.
- `vllm/v1/attention/AGENTS.md` — backend contract, selection, KV layout side effects.

## Commands

All Python must go through `uv` and `.venv/bin/python` — never system `python3`
or bare `pip` (enforced by root `AGENTS.md`).

```bash
# Environment
uv venv --python 3.12
source .venv/bin/activate
uv pip install -r requirements/lint.txt && pre-commit install

# Install (Python-only changes use the precompiled kernels)
VLLM_USE_PRECOMPILED=1 uv pip install -e . --torch-backend=auto
# Install (C/C++/CUDA changes recompile)
uv pip install -e . --torch-backend=auto

# Lint (matches CI)
pre-commit run --all-files
pre-commit run ruff-check --all-files
pre-commit run mypy-3.10 --all-files --hook-stage manual

# Tests — run a single file or test
.venv/bin/python -m pytest tests/v1/core/test_scheduler.py -v
.venv/bin/python -m pytest tests/v1/core/test_scheduler.py::test_name -v
```

Optional tests (marked `optional`) are skipped unless you pass `--optional`.
The nested `AGENTS.md` files list curated test bundles for each area (KV
cache/scheduler, worker/attention, engine).

## V1 architecture (the big picture)

V1 is the re-architected runtime, organized **by runtime responsibility**, not as
a flat API. It is multi-process by default: API server processes talk to engine
core processes over ZMQ; each engine core owns a scheduler + KV cache and
dispatches to GPU workers. With data parallelism there is one engine core per DP
rank plus a coordinator process when `data_parallel_size > 1`.

Request lifecycle, end to end:

1. **Front end** — `vllm/v1/engine/llm_engine.py` (sync) / `async_llm.py` (async)
   accept requests; `entrypoints/openai/` and `entrypoints/anthropic/` expose API
   servers. `engine/__init__.py` is a wire/data-model module, not just glue.
2. **Engine core loop** — `engine/core.py` (`EngineCore` / `EngineCoreProc`) runs
   the inner schedule → execute → update loop inside its own process.
3. **Scheduler** — `core/sched/scheduler.py` does admission, token budgeting,
   batching, and preemption, producing a `SchedulerOutput`. V1 budgets prompt and
   output tokens **uniformly** — prefill and decode are not separate phases.
4. **KV cache stack** — `KVCacheManager` (scheduler-facing facade) →
   `KVCacheCoordinator` (fans work across cache groups) →
   `SingleTypeKVCacheManager` (per-type block math: full / sliding / chunked-local
   / Mamba / cross-attn) → `BlockPool` (owns physical `KVCacheBlock`s, the
   prefix-cache hash table, free-list order, and KV events). Block `0` is the
   reserved `null_block`; never free or allocate it normally.
5. **Worker execution** — `worker/gpu_worker.py` (lifecycle, memory profiling, KV
   allocation) drives `worker/gpu_model_runner.py` (main GPU path: input prep,
   execution, sampling, spec decode, KV tensors). `worker/gpu/` is **experimental
   Model Runner V2** — not a stable replacement.
6. **Attention** — `attention/selector.py` picks a backend (a process-global
   choice that can call `set_kv_cache_layout()`); `attention/backend.py` is the
   contract; `attention/backends/` holds hardware/kernel-specific implementations
   with **non-uniform** shape/stride/support assumptions.
7. **Output** — `Scheduler.update_from_output()` consumes worker output, frees or
   updates request/cache state, and emits `EngineCoreOutputs` back to the front end.

Adjacent subsystems: `sample/` (sampler, rejection sampler, logits processors),
`spec_decode/` (EAGLE/ngram/medusa — couples runner buffers, attention metadata,
and scheduler slots), `kv_offload/` + `simple_kv_offload/` (CPU offload),
`structured_output/` (one grammar backend initialized and reused), and
`executor/` (uniproc / multiproc / Ray boundary). Connector-backed KV transfer
(disaggregated prefill, external tokens) spans `vllm/v1` **and**
`vllm/distributed/kv_transfer/` — analyze both sides together.

## Repo-wide layout

- `vllm/` — the engine. `config/` holds per-subsystem config dataclasses;
  `model_executor/` holds model implementations; `engine/` (top-level) is the
  legacy V0 path (deprecated — do not port V0-only behavior into V1).
- `csrc/` — C++/CUDA kernels; `cmake/` + `CMakeLists.txt` + `setup.py` build them.
- `tests/` — mirrors source, mostly `tests/v1/<area>/`.
- `benchmarks/`, `examples/`, `docs/` (`docs/usage/v1_guide.md`, `docs/design/`).

## Naming conventions

Data carriers use `*Metadata`, `*Stats`, `*Spec`, `*Config`, `*Output` suffixes —
reuse these rather than inventing parallel names. `KVCacheBlocks.blocks` is grouped
by KV cache group (not token-major). Prefix-cache hits are block-aligned; a full
prompt hit intentionally recomputes the last token so logits can be produced.
