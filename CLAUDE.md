# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is vLLM?

vLLM is a high-throughput, memory-efficient inference and serving engine for large language models (LLMs). It supports 200+ model architectures, multiple GPU/CPU backends, and provides an OpenAI-compatible API server.

## Development Commands

**All Python commands must use `uv` and `.venv/bin/python` — never system python or bare pip.**

### Environment Setup
```bash
uv venv --python 3.12
source .venv/bin/activate
uv pip install -r requirements/lint.txt
pre-commit install
```

### Install (Python-only changes)
```bash
VLLM_USE_PRECOMPILED=1 uv pip install -e . --torch-backend=auto
```

### Install (with C/C++ changes)
```bash
uv pip install -e . --torch-backend=auto
```

### Test Dependencies
```bash
uv pip install -r requirements/test/cuda.in    # cross-platform
uv pip install -r requirements/test/cuda.txt   # x86_64 only
```

### Run Tests
```bash
.venv/bin/python -m pytest tests/path/to/test_file.py -v
.venv/bin/python -m pytest tests/path/to/test_file.py::test_name -v
```

### Linting
```bash
pre-commit run --all-files                          # all hooks
pre-commit run ruff-check --all-files               # ruff only
pre-commit run mypy-3.10 --all-files --hook-stage manual  # mypy (CI mode)
```

Pre-commit hooks: ruff (lint + format), typos, clang-format (C++/CUDA), markdownlint, actionlint, pip-compile.

## Architecture Overview

### V1 Engine (primary, under `vllm/v1/`)

The V1 engine is the current active architecture. The request lifecycle flows:

1. **Entrypoints** (`vllm/entrypoints/`) — API servers (OpenAI-compatible, Anthropic Messages, gRPC) and the Python `LLM` class. `api_server.py` is the main HTTP server; `llm.py` provides offline batch inference.

2. **AsyncLLM** (`vllm/v1/engine/async_llm.py`) — The top-level async engine. Processes inputs, communicates with EngineCore via ZMQ-based `EngineCoreClient`.

3. **EngineCore** (`vllm/v1/engine/core.py`) — Runs in a separate process. Owns the scheduler and executor. Orchestrates the generate loop: schedule → execute → process outputs.

4. **Scheduler** (`vllm/v1/core/sched/scheduler.py`) — Decides which requests to run each step. Manages the KV cache via `KVCacheManager` (`vllm/v1/core/kv_cache_manager.py`) and block pool (`vllm/v1/core/block_pool.py`).

5. **Executor → Worker → ModelRunner** (`vllm/v1/executor/`, `vllm/v1/worker/`) — Executor manages workers across devices. `gpu_model_runner.py` handles the actual model forward pass, input preparation, and sampling.

6. **Output Processing** (`vllm/v1/engine/output_processor.py`, `detokenizer.py`) — Converts model outputs back to tokens/text and streams results.

### Legacy Engine (`vllm/engine/`)

`LLMEngine` and `AsyncLLMEngine` in `vllm/engine/` are the older engine. New development targets V1.

### Model Layer (`vllm/model_executor/`)

- **`models/`** — 270+ model implementations (one file per architecture). Each model registers in `models/registry.py`.
- **`layers/`** — Reusable building blocks: attention, linear layers, activations, rotary embeddings, quantization, MoE (`fused_moe/`), normalization. These abstract over different backends/quantization schemes.
- **`model_loader/`** — Weight loading logic for various formats (HF, safetensors, GGUF, etc.).

### Attention System

- **V1 attention** (`vllm/v1/attention/`) — Backend selector + implementations (FlashAttention, FlashInfer, TRTLLM-GEN, FlashMLA, Triton).
- **Layer-level** (`vllm/model_executor/layers/attention/`) — Attention layer base classes and legacy backends.

### Key Subsystems

- **Config** (`vllm/config/`) — All configuration dataclasses (`VllmConfig` is the root). Model, cache, parallel, scheduler, speculative configs, etc.
- **Distributed** (`vllm/distributed/`) — Tensor/pipeline/expert parallelism, communication ops, KV transfer for disaggregated serving.
- **Multimodal** (`vllm/multimodal/`) — Processing for images, audio, video inputs.
- **Kernels** (`vllm/kernels/`, `csrc/`) — Python-side kernel dispatch and C++/CUDA kernel implementations (attention, quantization, MoE, sampling, cache management).
- **LoRA** (`vllm/lora/`) — Multi-LoRA adapter serving.
- **Speculative Decoding** (`vllm/v1/spec_decode/`) — EAGLE, n-gram, draft model speculation.
- **Structured Output** (`vllm/v1/structured_output/`) — Constrained generation via xgrammar/guidance.
- **Platforms** (`vllm/platforms/`) — Hardware abstraction (CUDA, ROCm, TPU, CPU, XPU, etc.).

### C++/CUDA (`csrc/`)

Custom CUDA kernels for performance-critical ops: attention (`csrc/attention/`), quantization (`csrc/quantization/`), MoE routing (`csrc/moe/`), cache operations, fused kernels. Built via `setup.py` using PyTorch's cpp_extension.

## Contribution Policy

See `AGENTS.md` for mandatory duplicate-work checks, PR requirements, and the ban on pure code-agent PRs. Before any PR:

```bash
gh issue view <issue_number> --repo vllm-project/vllm --comments
gh pr list --repo vllm-project/vllm --state open --search "<issue_number> in:body"
```
