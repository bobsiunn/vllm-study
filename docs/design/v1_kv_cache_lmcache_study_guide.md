# V1 KV Cache and LMCache Connector Study Guide

!!! note
    This is a study roadmap, not an implementation design. Use it to learn the
    current V1 KV cache, worker block table, and LMCache connector boundaries
    before designing NPU-specific LMCache work.

## Goal

This guide helps you build a code-level mental model for three connected paths:

1. How V1 allocates, caches, frees, and invalidates KV blocks.
2. How scheduler-owned block IDs become worker block tables and slot mappings.
3. How LMCache connects to the scheduler and worker through the V1 KV connector
   APIs.

When you finish this guide, you should be able to trace one request from
prefix-cache lookup through block allocation, worker execution, external KV
load or save, and scheduler-side completion handling.

## Prerequisites

Read these first if the ideas are unfamiliar:

- [PagedAttention](paged_attention.md) for the OS paging analogy and why vLLM
  splits KV memory into fixed-size blocks.
- [Automatic Prefix Caching](prefix_caching.md) for block hashes, full-block
  caching, and prefix reuse.
- [Disaggregated Prefilling](../features/disagg_prefill.md) for connector,
  lookup buffer, pipe, scheduler connector, and worker connector terminology.

## Source Map

Core KV cache files:

- `vllm/v1/core/kv_cache_manager.py`: scheduler-facing facade, including
  `KVCacheBlocks`, `get_computed_blocks()`, and `allocate_slots()`.
- `vllm/v1/core/kv_cache_coordinator.py`: fan-out across KV cache groups.
- `vllm/v1/core/single_type_kv_cache_manager.py`: per-attention-type block
  math for full, sliding-window, chunked-local, Mamba, and cross-attention.
- `vllm/v1/core/block_pool.py`: physical block ownership, prefix-cache hash
  table, free queue, touch, free, evict, and KV events.
- `vllm/v1/core/kv_cache_utils.py`: `KVCacheBlock`, block hashes, and the
  free-block queue.

Scheduler and worker handoff files:

- `vllm/v1/core/sched/scheduler.py`: admission, connector lookup, allocation,
  metadata construction, invalid-block handling, and request completion.
- `vllm/v1/core/sched/output.py`: `NewRequestData`, `CachedRequestData`, and
  `SchedulerOutput` fields sent to workers.
- `vllm/v1/worker/gpu_input_batch.py`: worker request state and block-table row
  ownership.
- `vllm/v1/worker/block_table.py`: block ID rows and slot mapping.
- `vllm/v1/worker/gpu_model_runner.py`: block-table updates, slot mapping,
  attention metadata, block zeroing, and connector output wrapping.

Connector and LMCache files:

- `vllm/distributed/kv_transfer/kv_connector/v1/base.py`: V1 connector API for
  scheduler-side and worker-side implementations.
- `vllm/v1/worker/kv_connector_model_runner_mixin.py`: model-runner connector
  lifecycle around forward execution.
- `vllm/v1/outputs.py`: `KVConnectorOutput` returned from workers.
- `vllm/distributed/kv_transfer/kv_connector/v1/lmcache_connector.py`:
  `LMCacheConnectorV1` wrapper.
- `vllm/distributed/kv_transfer/kv_connector/v1/lmcache_integration/`: native
  vLLM adapter, multi-process adapter, and LMCache utility bridge.

## Core Mental Model

Start with ownership boundaries:

- `KVCacheManager` is the scheduler-facing facade. The scheduler should ask it
  for computed blocks, allocation, block IDs to zero, prefix-cache reset, and
  freeing.
- `KVCacheCoordinator` hides whether the model has one KV cache group or
  multiple groups with different block rules.
- `SingleTypeKVCacheManager` owns per-group request-to-block lists and answers
  how many blocks the request needs for one attention type.
- `BlockPool` owns every physical `KVCacheBlock`, including the `null_block`
  with block ID `0`.

Keep the container shape clear. `KVCacheBlocks.blocks` is grouped by KV cache
group, so `blocks[i][j]` means the `j`-th block in the `i`-th KV cache group.
Do not read it as token-major.

Checkpoint questions:

- Which object owns physical block lifetime?
- Which object decides how many blocks a request needs?
- Why does the scheduler receive `tuple[list[int], ...]` instead of a flat list?

## Allocation Flow

Read `KVCacheManager.allocate_slots()` slowly. Its docstring is the best local
map for the scheduling-time layout:

```text
<computed> <new local computed> <external computed> <new tokens> <lookahead>
```

The main stages are:

1. Free skipped blocks for attention types that no longer need early tokens,
   such as sliding-window attention.
2. Count required blocks for local prefix hits, external connector hits, new
   tokens, and speculative lookahead tokens.
3. Allocate blocks for external computed tokens and for tokens that still need
   model computation.
4. Cache full blocks immediately unless caching is disabled or
   `delay_cache_blocks` is set for an async KV-transfer path.

Focus especially on these arguments:

- `num_new_computed_tokens`: tokens already cached by vLLM prefix caching.
- `num_external_computed_tokens`: tokens whose KV exists outside vLLM, such as
  in LMCache.
- `delay_cache_blocks`: prevents vLLM from marking blocks cached before a
  future transfer completes.
- `num_lookahead_tokens`: extra slots for speculative decoding.

Checkpoint questions:

- Why can allocation happen when `num_new_tokens == 0` if external KV must be
  loaded?
- What blocks are allocated for connector hits that are inside the attention
  window?
- When are newly allocated full-attention block IDs recorded for worker zeroing?

## BlockPool and Prefix Cache

`BlockPool` is the physical block owner. Study these operations in order:

1. `get_new_blocks()`: takes blocks from the free queue, evicting old cached
   hashes if needed.
2. `touch()`: increments references for cached blocks hit by another request.
3. `free_blocks()`: decrements references and appends zero-reference blocks
   back to the free queue.
4. `cache_full_blocks()`: assigns block hashes and inserts full blocks into the
   prefix-cache hash table.
5. `evict_blocks()`: removes cached hashes for connector-reported invalid
   blocks without necessarily freeing in-use blocks.

The `null_block` is special. It represents skipped or out-of-window slots and
must not be treated like a normal free-list block.

Checkpoint questions:

- When does a cached block remain in the cache table but also become evictable?
- Why are request blocks freed in reverse order?
- What invariant protects against a connector reporting a block ID that the
  scheduler never allocated?

## Scheduler Connector Path

The scheduler combines local prefix caching and external KV-cache lookup before
it decides what the worker should run.

Study this sequence in `vllm/v1/core/sched/scheduler.py`:

1. The scheduler creates `KVCacheManager` and, when `kv_transfer_config` is set,
   creates a scheduler-side connector.
2. For waiting requests, it asks the connector for external matched tokens with
   `get_num_new_matched_tokens()` after local prefix-cache lookup.
3. It calls `KVCacheManager.allocate_slots()` with
   `num_external_computed_tokens` and possibly `delay_cache_blocks`.
4. It calls `connector.update_state_after_alloc()` so the connector can remember
   the request and allocated block state.
5. It builds `SchedulerOutput.kv_connector_metadata` with
   `connector.build_connector_meta()`.

Checkpoint questions:

- Where do external cache hits reduce model computation?
- Why must connector state be updated after allocation, not before allocation?
- Which scheduler output field carries connector metadata to every worker?

## SchedulerOutput Handoff

`SchedulerOutput` is the main handoff object from scheduler to worker. For KV
study, focus on these fields:

- `scheduled_new_reqs`: list of `NewRequestData` objects. Each new request has
  `block_ids` and `num_computed_tokens`.
- `scheduled_cached_reqs`: `CachedRequestData` for already-known worker-side
  requests. It carries `new_block_ids`, `num_computed_tokens`, and token diffs.
- `kv_connector_metadata`: scheduler-side connector instructions for worker
  connectors.
- `new_block_ids_to_zero`: block IDs whose GPU memory must be zeroed before
  use.

Checkpoint questions:

- Which fields are stable request state and which fields are per-step diffs?
- How does a resumed request differ from a request that merely appends block IDs?
- Why does zeroing live in scheduler output instead of inside `BlockPool`?

## Worker BlockTable Path

The worker converts scheduler block IDs into tensors consumed by attention
backends.

Read the path in this order:

1. `gpu_input_batch.py` creates a `MultiGroupBlockTable` and owns request rows.
2. `BlockTable.add_row()` initializes a row for a new request.
3. `BlockTable.append_row()` appends new block IDs for an existing request.
4. `BlockTable.commit_block_table()` copies CPU block-table rows to the GPU.
5. `BlockTable.compute_slot_mapping()` maps token positions to KV cache slots.

For hybrid cases, `BlockTable.map_to_kernel_blocks()` can split a KV-manager
block ID into multiple attention-kernel block IDs when the allocation block
size is larger than the kernel block size.

Checkpoint questions:

- What row in `BlockTable` corresponds to a request in `InputBatch`?
- When are block IDs still CPU-side bookkeeping and when are they GPU-visible?
- How does `slot_mapping` connect token positions to physical KV slots?

## GPUModelRunner Commit Path

In `GPUModelRunner`, follow how scheduler output becomes execution metadata:

- New and cached request state is applied to `InputBatch`.
- New block IDs are appended or replaced depending on whether the request was
  resumed.
- `new_block_ids_to_zero` is zeroed before those blocks are used.
- The block table is committed and slot mapping is computed before attention
  metadata is built.

For connector-aware execution, keep the block table and connector lifecycle in
the same mental trace. The worker may be loading external KV into blocks that
the scheduler already allocated.

Checkpoint questions:

- Where does a new block ID first enter worker state?
- Which operation makes block-table rows visible to GPU attention kernels?
- Why is block zeroing part of correctness rather than just cleanup?

## KV Connector V1 Boundary

`KVConnectorBase_V1` defines a split API.

Scheduler-side methods answer questions such as:

- How many external tokens match this request?
- What connector metadata should be sent to the workers this step?
- Has a worker-side connector reported completion, failure, or events?
- Should request blocks be freed now, or is an async connector still using them?

Worker-side methods do the actual KV movement:

- Bind scheduler metadata before execution.
- Start loading KV before the forward pass.
- Wait for per-layer loads when the attention layer needs the data.
- Save per-layer KV during execution.
- Wait for saves and report finished sends, finished receives, stats, events,
  worker metadata, and invalid block IDs.

Checkpoint questions:

- Which connector calls are scheduler-only?
- Which connector calls must happen around model forward?
- What data flows worker-to-scheduler after a step finishes?

## Model Runner Connector Lifecycle

`KVConnectorModelRunnerMixin` wraps the worker-side lifecycle:

```text
bind_connector_metadata
start_load_kv
model forward
wait_for_save
get_finished
get_block_ids_with_load_errors
get_kv_connector_stats
get_kv_connector_kv_cache_events
build_connector_worker_meta
clear_connector_metadata
```

During model forward, attention-layer code can call connector hooks such as
`wait_for_layer_load()` and `save_kv_layer()`. The mixin owns the outer
bind, start, finish, output, and clear lifecycle around that per-layer work.

The resulting `KVConnectorOutput` can contain:

- `finished_sending` and `finished_recving` request IDs.
- connector stats, KV cache events, and worker metadata.
- `invalid_block_ids` for blocks whose external load failed.
- `expected_finished_count` for connectors that need handshake aggregation.

Checkpoint questions:

- What state must be cleared after every worker execution?
- Why are `finished_sending` and `finished_recving` separate?
- How does an invalid block ID become a scheduler-side recompute decision?

## LMCacheConnectorV1 Path

`LMCacheConnectorV1` is a vLLM connector wrapper. It chooses either the native
vLLM adapter or the latest LMCache adapter, then delegates the V1 connector API
to that adapter.

Important delegated methods include:

- Worker path: `start_load_kv()`, `wait_for_layer_load()`, `save_kv_layer()`,
  `wait_for_save()`, `get_finished()`, and load-error reporting.
- Scheduler path: `get_num_new_matched_tokens()`, `update_state_after_alloc()`,
  `build_connector_meta()`, `update_connector_output()`, `request_finished()`,
  and `take_events()`.
- Event path: LMCache events are converted into vLLM `BlockStored` events and
  aggregated across workers.

Checkpoint questions:

- Where does vLLM stop and LMCache-specific request tracking begin?
- Which LMCache calls need allocated block IDs from vLLM?
- Which parts of the adapter are lookup, load, save, and completion tracking?

## RequestTracker

In `lmcache_integration/vllm_v1_adapter.py`, `RequestTracker` is the key bridge
between vLLM request state and LMCache request state. It tracks:

- request ID, prompt length, token IDs, allocated block IDs, and saved-token
  count.
- disaggregated-prefill metadata, multimodal hashes, request configs, decode
  phase, and `skip_save`.

For future NPU-side LMCache work, treat these as semantic requirements to
understand before changing hardware movement. The exact data mover may change,
but request identity, token coverage, allocated block IDs, and save/load state
must remain coherent with vLLM scheduler decisions.

Checkpoint questions:

- Which tracker fields are needed for lookup keys?
- Which fields are needed to map external KV into vLLM-allocated blocks?
- Which fields are needed to avoid saving or loading the wrong token range?

## Invalid Block Handling

Study `KVConnectorOutput.invalid_block_ids` and the scheduler methods that
process it. The flow has a policy split:

1. A worker connector reports block IDs whose external load failed.
2. The scheduler scans async waiting requests without collecting cache eviction
   candidates, because those blocks are not cached yet.
3. The scheduler scans running sync-load requests and can collect invalid and
   downstream blocks for prefix-cache eviction.
4. Affected requests are truncated to their longest valid computed prefix, and
   their external-computed-token state is reduced.
5. With recompute policy, async failures are marked for retry and sync affected
   requests are rescheduled to recompute. With fail policy, affected requests
   are failed, and sync cached blocks are evicted.

Checkpoint questions:

- Why are async load failures not evicted from prefix cache?
- How does the scheduler decide which requests are affected?
- How does failure policy change recompute versus fail behavior?
- What must remain true about block IDs reported by workers?

## End-to-End Exercises

Use these exercises to check whether the mental model is complete.

### Exercise 1: Local prefix-cache prefill

Trace a request that has local prefix-cache hits but no connector hits:

1. `get_computed_blocks()` finds full cached blocks.
2. `allocate_slots()` touches those blocks and allocates slots for the suffix.
3. `SchedulerOutput` sends block IDs to the worker.
4. `BlockTable` commits the rows and computes slot mapping.

### Exercise 2: LMCache external-hit prefill

Trace a request that has external LMCache hits:

1. Scheduler gets local computed tokens and external matched tokens.
2. `allocate_slots()` allocates blocks for external computed tokens.
3. Connector metadata tells workers what to load.
4. Worker connector loads KV into the allocated blocks before attention needs it.
5. Worker returns `KVConnectorOutput` to update scheduler-side connector state.

### Exercise 3: Invalid external blocks

Trace a failed external load:

1. Worker reports `invalid_block_ids`.
2. Scheduler evicts affected block hashes.
3. Affected requests lose external-computed-token credit.
4. The request is rescheduled to recompute missing KV.

## Tests to Read and Run

Use `.venv/bin/python`, not system `python3`.

Core KV cache tests:

```bash
.venv/bin/python -m pytest tests/v1/core/test_kv_cache_utils.py tests/v1/core/test_single_type_kv_cache_manager.py tests/v1/core/test_prefix_caching.py tests/v1/core/test_scheduler.py -v
```

Worker block-table tests:

```bash
.venv/bin/python -m pytest tests/v1/worker/test_gpu_input_batch.py tests/v1/worker/test_gpu_model_runner.py -v
```

Connector and LMCache tests:

```bash
.venv/bin/python -m pytest tests/v1/kv_connector/unit/test_lmcache_connector.py tests/v1/kv_connector/unit/test_lmcache_integration.py tests/v1/kv_connector/unit/test_kv_connector_lifecycle.py tests/v1/kv_connector/unit/test_remote_prefill_lifecycle.py tests/v1/kv_connector/unit/test_remote_decode_lifecycle.py tests/v1/kv_connector/unit/test_output_aggregator.py tests/v1/kv_connector/unit/test_invalid_blocks_correctness.py tests/v1/kv_connector/unit/test_error_propagation.py -v
```

Read the tests before changing code. They show the intended behavior more
compactly than the full scheduler and model runner files.

## NPU LMCache Study Notes

Do not start from hardware transfer code. Start from invariants:

- vLLM scheduler owns block allocation decisions and block ID validity.
- Worker connectors move KV into or out of blocks that scheduler state already
  understands.
- Connector metadata must be serializable and must describe the exact per-step
  work workers need to perform.
- Slot mapping and block tables are attention-kernel contracts; any NPU path
  must preserve equivalent token-to-KV-slot semantics.
- Request tracking must preserve token coverage, allocated block IDs, saved
  ranges, load ranges, multimodal identity, and completion state.

Keep a separate notes file for open NPU questions, such as transfer granularity,
layout conversion, async completion, and failure reporting. Do not mix those
open questions into the current-code reading path until the V1 invariants are
clear.

## Quick Self-Check

Before moving on to NPU design, make sure you can answer these without reading
the code:

1. What is the difference between a locally computed token, a local prefix-cache
   hit, and an external connector hit?
2. Why can `KVCacheBlocks` contain multiple block lists?
3. How does `BlockPool` decide whether a cached block can be reused, touched, or
   evicted?
4. Which scheduler output fields tell the worker about block IDs and connector
   work?
5. How does `BlockTable` turn block IDs and token positions into slot mappings?
6. What does `KVConnectorModelRunnerMixin` do before and after model forward?
7. How does LMCache learn the allocated block IDs for a request?
8. What should happen if a connector reports that a block failed to load?
