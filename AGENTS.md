# vLLM Agent Guide

**Generated:** 2026-07-01
**Commit:** b66f7094d
**Branch:** vllm-study

These instructions apply to all AI-assisted contributions to `vllm-project/vllm`.
Breaching contribution policy can result in automatic banning.

## Contribution Policy

Before proposing a PR, run duplicate-work checks:

```bash
gh issue view <issue_number> --repo vllm-project/vllm --comments
gh pr list --repo vllm-project/vllm --state open --search "<issue_number> in:body"
gh pr list --repo vllm-project/vllm --state open --search "<short area keywords>"
```

- If an open PR already addresses the same fix, do not open another.
- If your approach is materially different from an existing PR, explain the
  difference in the issue.
- Do not open one-off PRs for tiny edits such as a single typo, isolated style
  change, or one mutable default. Mechanical cleanup is acceptable only when
  bundled with substantive work.
- Pure code-agent PRs are not allowed. A human submitter must understand and
  defend every changed line, review every change, and run relevant tests.
- AI-assisted PR descriptions must include why the work is not duplicating an
  existing PR, test commands and results, and a clear AI-assistance statement.
- If work is duplicate or trivial busywork, fail closed and do not proceed.

## Overview

This is a mixed Python, C++/CUDA, and Rust inference-engine repository. The main
Python package is `vllm`; native kernels live under `csrc`; an early Rust
frontend lives under `rust`.

## Structure

```text
vllm-study/
|-- vllm/                 # Python package, entrypoints, runtime, model code
|   |-- v1/               # V1 runtime; read nested guides before editing
|   `-- model_executor/   # model classes, layers, loaders, quantization
|-- csrc/                 # C++/CUDA/ROCm/CPU native kernels and torch bindings
|-- tests/                # pytest suite with hardware/platform markers
|-- rust/                 # Rust frontend workspace
|-- benchmarks/           # standalone benchmark runners
|-- examples/             # runnable examples and API usage
|-- docs/                 # user, design, and contributor docs
|-- docker/               # multi-stage CUDA/CPU/ROCm/XPU/TPU images
`-- .buildkite/           # release, wheel, image, and hardware CI matrix
```

## Scoped Guides

Read the nearest guide before editing in that subtree:

- `vllm/v1/AGENTS.md` for V1 engine, scheduler, worker, attention, metrics.
- `vllm/v1/core/AGENTS.md` for scheduler and KV cache manager internals.
- `vllm/v1/worker/AGENTS.md` for GPU workers and model runners.
- `vllm/v1/attention/AGENTS.md` for attention backend selection and layouts.
- `vllm/model_executor/AGENTS.md` for model/layer/loader changes.
- `csrc/AGENTS.md` for native kernels and bindings.
- `tests/AGENTS.md` for test fixtures, markers, and hardware gating.
- `rust/AGENTS.md` for Rust frontend work.

## Where To Look

| Task | Location | Notes |
| --- | --- | --- |
| CLI entry | `vllm/entrypoints/cli/main.py` | `pyproject.toml` maps `vllm` here |
| Serve command | `vllm/entrypoints/cli/serve.py` | Launches OpenAI-compatible server |
| API server | `vllm/entrypoints/openai/api_server.py` | App setup, engine client, workers |
| V1 request flow | `vllm/v1/engine/`, `vllm/v1/core/`, `vllm/v1/worker/` | Runtime responsibility split |
| Legacy engine | `vllm/engine/` | V0 path; do not port behavior into V1 blindly |
| Models and layers | `vllm/model_executor/` | Registry, model classes, layers, loaders |
| Distributed KV transfer | `vllm/distributed/kv_transfer/`, `vllm/v1/core/`, `vllm/v1/worker/` | Crosses V1 boundary |
| Build logic | `setup.py`, `CMakeLists.txt`, `pyproject.toml` | Native extension and package assembly |
| CI matrix | `.github/workflows/`, `.buildkite/`, `docker/` | No top-level `Makefile` workflow |

## Code Map

LSP/codegraph tools were unavailable when this map was generated; reference
centrality is unmeasured.

| Symbol | Type | Location | Refs | Role |
| --- | --- | --- | --- | --- |
| `main` | function | `vllm/entrypoints/cli/main.py` | n/a | Installed `vllm` command |
| `ServeSubcommand` | class | `vllm/entrypoints/cli/serve.py` | n/a | Server CLI command |
| `build_async_engine_client` | function | `vllm/entrypoints/openai/api_server.py` | n/a | Server-to-engine client setup |
| `LLM` | class | `vllm/entrypoints/llm.py` | n/a | Offline Python API facade |
| `SamplingParams` | class | `vllm/sampling_params.py` | n/a | User sampling contract |
| `LLMEngine` | class | `vllm/v1/engine/llm_engine.py` | n/a | V1 sync engine facade |
| `AsyncLLM` | class | `vllm/v1/engine/async_llm.py` | n/a | V1 async facade |
| `EngineCore` | class | `vllm/v1/engine/core.py` | n/a | Schedule/execute/update loop |
| `Scheduler` | class | `vllm/v1/core/sched/scheduler.py` | n/a | Admission, batching, preemption |
| `KVCacheManager` | class | `vllm/v1/core/kv_cache_manager.py` | n/a | Scheduler-facing KV allocation |
| `GPUModelRunner` | class | `vllm/v1/worker/gpu_model_runner.py` | n/a | Main V1 GPU execution path |
| `AttentionBackend` | class | `vllm/v1/attention/backend.py` | n/a | Attention backend contract |

## Development Workflow

- Never use system `python3` or bare `pip`/`pip install`. Use `uv` and
  `.venv/bin/python`.
- For Python-only changes, prefer precompiled kernels during local install:
  `VLLM_USE_PRECOMPILED=1 uv pip install -e . --torch-backend=auto`.
- For C/C++/CUDA changes, reinstall without `VLLM_USE_PRECOMPILED`.
- Install lint hooks before normal development:

```bash
uv venv --python 3.12
source .venv/bin/activate
uv pip install -r requirements/lint.txt
pre-commit install
```

## Test And Lint Commands

```bash
# Test dependencies. Use cuda.in on non-x86_64 platforms.
uv pip install -r requirements/test/cuda.in
# On x86_64, the pinned file is also valid:
uv pip install -r requirements/test/cuda.txt

# Specific pytest file or test.
.venv/bin/python -m pytest tests/path/to/test_file.py -v

# Lint gates.
pre-commit run
pre-commit run --all-files
pre-commit run ruff-check --all-files
pre-commit run mypy-3.12 --all-files --hook-stage manual
```

## Conventions

- Python style follows Google-style docstrings with `Args:`, `Returns:`, and
  `Raises:`, not Sphinx fields.
- Python line length is 88 characters.
- Ruff, mypy, pytest markers, typos, SPDX, clang-format, generated docs checks,
  and requirements compilation are wired through `pyproject.toml` and
  `.pre-commit-config.yaml`.
- Pytest markers include `slow_test`, `skip_global_cleanup`, `core_model`,
  `hybrid_model`, `cpu_model`, `cpu_test`, `split`, `distributed`, and
  `optional`.
- Build/CI is direct-command driven: `uv`, `pre-commit`, `pytest`, `cargo`,
  `docker`, and Buildkite scripts. Do not assume a `make` entrypoint exists.

## Anti-Patterns

- Do not modify `AGENTS.md` or guides it links to without first reading
  `docs/contributing/editing-agent-instructions.md`.
- Do not reproduce upstream tool docs in agent guides; keep project-specific
  invariants here and area knowledge in scoped guides.
- Do not port V0-only behavior into V1 without reading `docs/usage/v1_guide.md`
  and the relevant `vllm/v1` guide.
- Do not analyze connector-backed KV flows only inside `vllm/v1`; the connector
  interfaces and implementations also live under `vllm/distributed/kv_transfer`.
- Do not use `VLLM_TRACE_FUNCTION=1` except as last-resort debugging; docs warn
  it can slow token generation by over 100x.

## Commit Messages

AI-assisted commits should include appropriate trailers, for example:

```text
Co-authored-by: GitHub Copilot
Co-authored-by: Claude
Co-authored-by: gemini-code-assist
Signed-off-by: Your Name <your.email@example.com>
```
