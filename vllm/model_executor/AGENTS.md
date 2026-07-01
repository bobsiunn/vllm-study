# vLLM Model Executor Guide

This guide applies under `vllm/model_executor`. The root guide still controls
contribution policy, Python environment, lint, and PR rules.

## Overview

`model_executor` owns model implementations, reusable layers, model loading,
quantization hooks, kernels exposed to Python, and execution-time model helpers.

## Structure

```text
vllm/model_executor/
|-- models/        # architecture classes, registry, interfaces
|-- layers/        # reusable neural-network layers and quantization plumbing
|-- model_loader/  # weight loading, formats, sharding, safetensors paths
|-- kernels/       # Python-facing kernel wrappers
|-- offloader/     # CPU/UVA offload and prefetch helpers
`-- warmup/        # warmup utilities
```

## Where To Look

| Task | Location | Notes |
| --- | --- | --- |
| Register a model architecture | `models/registry.py` | Also update `tests/models/registry.py` examples |
| Check model capability flags | `models/interfaces.py`, `models/interfaces_base.py` | Multimodal, pooling, PP, Mamba, transcription |
| Add or change a model class | `models/<architecture>.py` | Match existing architecture family patterns |
| Change multimodal model behavior | `models/interfaces.py`, `vllm/multimodal/` | Processor registry and input contracts interact |
| Change quantized layers | `layers/quantization/`, `tests/quantization/`, `tests/kernels/quantization/` | Python and kernel tests both matter |
| Change loading behavior | `model_loader/`, `tests/model_executor/model_loader/` | Weight formats and lazy imports are fragile |
| Change offload behavior | `offloader/` | Coordinate with scheduler/worker offload paths |

## Code Map

| Symbol | Type | Location | Role |
| --- | --- | --- | --- |
| `_TEXT_GENERATION_MODELS` | registry | `models/registry.py` | HF architecture name to vLLM implementation |
| `ModelRegistry` | class | `models/registry.py` | Resolves supported model classes |
| `SupportsMultiModal` | protocol | `models/interfaces.py` | Required multimodal model contract |
| `SupportsPP` | protocol | `models/interfaces.py` | Pipeline-parallel model behavior |
| `VllmModel` | protocol | `models/interfaces_base.py` | Base model interface shape |
| `QuantizationConfig` | class | `layers/quantization/base_config.py` | Quantization method contract |

## Conventions

- Model registry changes must keep examples in `tests/models/registry.py` in
  sync; the registry file explicitly calls this out.
- Capability is expressed through interfaces and flags, not ad hoc attribute
  checks in unrelated callers.
- Multimodal models must honor the `embed_input_ids(..., is_multimodal=...)`
  contract; missing `is_multimodal` is treated as an update-required error.
- Prefer reusing existing layer, quantization, and weight-loader utilities before
  adding model-local variants.
- Keep architecture aliases in the registry close to the model family they map
  to; many HuggingFace class names intentionally resolve to shared vLLM classes.

## Anti-Patterns

- Do not add a model file without wiring registry entries, capability interfaces,
  and focused tests for the advertised task type.
- Do not hide model-loader side effects behind eager imports in registry paths;
  lazy import keeps optional dependencies and dynamic modules isolated.
- Do not add quantization behavior only at the Python layer when kernels,
  serialization, or config validation also need changes.
- Do not duplicate multimodal preprocessing logic inside model classes when the
  registry or `vllm/multimodal` owns the contract.

## Tests

```bash
.venv/bin/python -m pytest tests/models/test_registry.py tests/models/registry.py -v
.venv/bin/python -m pytest tests/model_executor/model_loader -v
.venv/bin/python -m pytest tests/quantization tests/kernels/quantization -v
```
