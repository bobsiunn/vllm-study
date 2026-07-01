# vLLM Native Code Guide

This guide applies under `csrc`. The root guide still controls contribution
policy, environment setup, and AI-assisted PR requirements.

## Overview

`csrc` contains C++/CUDA/ROCm/CPU kernels, torch extension bindings, stable ABI
variants, and low-level collective or quantization implementations used from the
Python package.

## Structure

```text
csrc/
|-- attention/          # attention kernels and helpers
|-- core/               # shared scalar/type utilities
|-- cpu/                # CPU kernels and micro-GEMM paths
|-- libtorch_stable/    # stable ABI CUDA-facing declarations/ops
|-- moe/                # fused MoE kernels and routing helpers
|-- quantization/       # machete, marlin, w8a8 kernels
|-- quickreduce/        # ROCm quick all-reduce path
`-- rocm/               # ROCm-specific support
```

## Where To Look

| Task | Location | Notes |
| --- | --- | --- |
| Python-visible ops | `ops.h`, `torch_bindings.cpp` | Keep signatures aligned with callers |
| Build wiring | `CMakeLists.txt`, `setup.py`, `cmake/` | Reinstall without precompiled kernels |
| Stable ABI declarations | `libtorch_stable/` | Some declarations intentionally duplicate `ops.h` |
| Attention kernels | `attention/`, `libtorch_stable/attention/` | Coordinate with V1 attention metadata/layout |
| MoE kernels | `moe/`, `tests/kernels/moe/` | Expert routing and layout assumptions are tested |
| Quantization kernels | `quantization/`, `tests/kernels/quantization/` | Python quantization configs also need coverage |
| CPU kernels | `cpu/`, `tests/kernels/core/` | CPU wheel path differs from CUDA/ROCm |

## Conventions

- `ops.h` is still used by CPU builds even when corresponding CUDA declarations
  also exist under `csrc/libtorch_stable`.
- Platform-specific paths are selected through build configuration and torch
  platform detection in `setup.py`; do not assume CUDA is always the target.
- Native changes require editable install without `VLLM_USE_PRECOMPILED` so local
  kernels are rebuilt.
- Kernel changes usually need both Python-facing tests and kernel-specific tests;
  the relevant suites live under `tests/kernels/`.

## Anti-Patterns

- Do not change tensor shape, stride, or dtype assumptions in native kernels
  without checking Python wrappers, V1 worker allocation, and attention metadata.
- Do not update torch binding signatures without updating all Python call sites
  and tests that exercise the op.
- Do not treat ROCm, XPU, CPU, and CUDA behavior as interchangeable; platform
  gates and fallback paths are explicit in this repo.
- Do not rely on precompiled kernels to validate native edits.

## Commands

```bash
# Rebuild local native extensions after C/C++/CUDA edits.
uv pip install -e . --torch-backend=auto

# Focused kernel suites.
.venv/bin/python -m pytest tests/kernels/attention tests/kernels/moe -v
.venv/bin/python -m pytest tests/kernels/quantization tests/kernels/core -v

# Native formatting/lint hooks.
pre-commit run clang-format --all-files
pre-commit run copyright-header --all-files
```
