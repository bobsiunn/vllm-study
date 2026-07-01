# vLLM Test Guide

This guide applies under `tests`. The root guide still controls environment
setup, Python command usage, and contribution policy.

## Overview

The test suite mirrors major runtime areas but is hardware-aware: many tests
depend on CUDA/ROCm/XPU/CPU markers, shared fixtures, subprocess cleanup, and
platform-specific skip logic.

## Structure

```text
tests/
|-- conftest.py          # global fixtures, assets, cleanup, multimodal helpers
|-- utils.py             # server helpers, GPU marks, platform utilities
|-- v1/                  # V1 runtime tests mirroring vllm/v1
|-- kernels/             # kernel-level attention/MoE/quantization/core tests
|-- models/              # model registry, language, multimodal, pooling tests
|-- entrypoints/         # CLI/API server behavior
|-- distributed/         # distributed and multi-GPU behavior
|-- quantization/        # Python-facing quantization coverage
|-- lora/                # LoRA-specific fixtures and tests
`-- prompts/             # shared prompt fixtures
```

## Where To Look

| Task | Location | Notes |
| --- | --- | --- |
| Shared fixtures | `conftest.py` | Autouse cleanup and common assets live here |
| GPU/platform marks | `utils.py` | `large_gpu_mark`, `multi_gpu_test`, tier helpers |
| V1 scheduler/KV tests | `v1/core/` | Mirrors `vllm/v1/core` guide |
| V1 worker tests | `v1/worker/`, `v1/streaming_input/` | Runner, input batch, streaming paths |
| Attention kernels | `v1/attention/`, `kernels/attention/` | Backend selection plus kernel behavior |
| Model registry examples | `models/registry.py` | Update when adding architectures |
| OpenAI entrypoint tests | `entrypoints/openai/` | Server, chat, responses, tool calling |

## Conventions

- Declared pytest markers are in `pyproject.toml`: `slow_test`,
  `skip_global_cleanup`, `core_model`, `hybrid_model`, `cpu_model`,
  `cpu_test`, `split`, `distributed`, and `optional`.
- Global cleanup is expensive and automatic; use `skip_global_cleanup` only when
  the test is safe without full distributed cleanup.
- Hardware requirements usually use helpers from `tests/utils.py` or direct
  `current_platform` skip checks.
- CPU-only coverage commonly uses `pytestmark = pytest.mark.cpu_test`.
- ROCm tests may need `ROCM_EXTRA_ARGS` or `ROCM_ENGINE_KWARGS` from
  `tests/utils.py` to reduce flakiness.
- Optional tests are skipped unless explicitly enabled with `--optional`.

## Anti-Patterns

- Do not add unconditional GPU tests; gate by platform, memory, or device count.
- Do not bypass shared server and process helpers for entrypoint tests unless the
  test needs a distinct lifecycle.
- Do not remove distributed cleanup to speed up tests; use the marker only when
  the test's state cannot leak.
- Do not add model registry entries without test examples in `tests/models`.

## Commands

```bash
# Single file or node.
.venv/bin/python -m pytest tests/path/to/test_file.py -v
.venv/bin/python -m pytest tests/path/to/test_file.py::test_name -v

# V1 core and worker bundles.
.venv/bin/python -m pytest tests/v1/core/test_scheduler.py tests/v1/core/test_prefix_caching.py -v
.venv/bin/python -m pytest tests/v1/worker/test_gpu_model_runner.py tests/v1/worker/test_gpu_input_batch.py -v

# Kernel-focused suites.
.venv/bin/python -m pytest tests/kernels/attention tests/kernels/quantization -v
```
