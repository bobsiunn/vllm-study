# Code Review: v1_kvconnector_disaggregation_analysis_ko.md

Target: `docs/study/v1_kvconnector_disaggregation_analysis_ko.md`

Review scope: correctness against the local checkout at `b66f7094d`, focused on
KVConnector V1 APIs, scheduler/worker flows, P/D disaggregation metadata, and
source citations.

Skill perspective check:

- `omo:remove-ai-slops` consulted. No deletion-only tests, tautological tests, or
  requested-removal tests are present because this is a documentation-only
  target. Perspective violation: none in tests. Documentation slop concern:
  incomplete API maps can create false confidence, covered below.
- `omo:programming` consulted. No code was changed. Perspective applied to
  Python API claims: strict signatures, boundary ownership, and avoiding
  implementation-mirroring or stale API descriptions. Violation: the document
  presents an outdated `request_finished` API shape for current Python code.

## CRITICAL

None.

## HIGH

1. `docs/study/v1_kvconnector_disaggregation_analysis_ko.md:205` omits the
   required `block_ids` argument from the current V1 scheduler-side
   `request_finished` API. The local base signature is
   `request_finished(self, request, block_ids)` in
   `vllm/distributed/kv_transfer/kv_connector/v1/base.py:542`, and the scheduler
   dispatches `self.connector.request_finished(request, block_ids[0])` for
   non-HMA connectors or `request_finished_all_groups(request, block_ids)` for
   HMA connectors in `vllm/v1/core/sched/scheduler.py:2118` and
   `vllm/v1/core/sched/scheduler.py:2126`. A reader implementing a connector
   from the table would write the wrong override signature and fail at runtime
   when the scheduler finishes a request.

## MEDIUM

1. `docs/study/v1_kvconnector_disaggregation_analysis_ko.md:207` introduces the
   "worker-side API" map but leaves out current worker-to-scheduler metadata
   APIs that are active in the local flow. `KVConnectorBase_V1` exposes
   `build_connector_worker_meta()` at
   `vllm/distributed/kv_transfer/kv_connector/v1/base.py:429`, the model runner
   stores it in `output.kv_connector_worker_meta` at
   `vllm/v1/worker/kv_connector_model_runner_mixin.py:109`, and
   `KVOutputAggregator` aggregates it at
   `vllm/distributed/kv_transfer/kv_connector/utils.py:130`. The omission makes
   the lifecycle diagram and API table look like `finished_sending`,
   `finished_recving`, and `invalid_block_ids` are the only uplink mechanisms,
   which is not true for offloading/simple CPU offload and multi-connector
   implementations.

2. `docs/study/v1_kvconnector_disaggregation_analysis_ko.md:407` summarizes the
   worker context exit path without the current stats/events/worker-meta fields
   in `KVConnectorOutput`. Local code also fills `kv_connector_stats`,
   `kv_cache_events`, and `kv_connector_worker_meta` at
   `vllm/v1/worker/kv_connector_model_runner_mixin.py:107` through
   `vllm/v1/worker/kv_connector_model_runner_mixin.py:109`; the output dataclass
   includes these fields plus `expected_finished_count` in
   `vllm/v1/outputs.py:196`. The prose later says worker metadata can be
   aggregated, but the concrete lifecycle excerpt does not show the mechanism,
   so production connector behavior is under-described.

## LOW

1. `docs/study/v1_kvconnector_disaggregation_analysis_ko.md:431` shows the
   attention wrapper no-op guard as only "no KV group or not V1". Current local
   code also skips transfer when `attn_metadata is None` or
   `not connector.has_connector_metadata()` in
   `vllm/model_executor/layers/attention/kv_transfer_utils.py:47`. The main
   layer-wise load/save explanation is correct, but the excerpt hides two real
   guards that matter when reasoning about why a connector hook did not run.

2. `docs/study/v1_kvconnector_disaggregation_analysis_ko.md:631` lists common
   `kv_transfer_params` keys but omits current NIXL fields that are required or
   semantically important in the local code, including `tp_size` for heartbeat
   tracking and `remote_num_tokens` for bidirectional D-to-P external-token
   accounting. See `vllm/distributed/kv_transfer/kv_connector/v1/nixl/scheduler.py:187`,
   `vllm/distributed/kv_transfer/kv_connector/v1/nixl/scheduler.py:411`, and
   `vllm/distributed/kv_transfer/kv_connector/v1/nixl/scheduler.py:672`.

3. `docs/study/v1_kvconnector_disaggregation_analysis_ko.md:199` uses shortened
   citations such as `base.py:454` and `scheduler.py:618` in the API tables.
   Later source lists use full paths and the files do exist, but the table
   anchors are ambiguous in this repository where there are many `base.py` and
   `scheduler.py` files.

## Verification Notes

- Loaded and applied the required `remove-ai-slops` and `programming` skill
  perspectives before judging relevance and maintainability.
- Checked the target document line-by-line with `nl`.
- Verified the primary cited local code paths with codegraph/source reads:
  `KVConnectorBase_V1`, scheduler `schedule()` and cleanup/update paths, worker
  KV connector mixin, attention KV transfer wrapper, disagg protocol/output
  schemas, NIXL scheduler, and the multiturn proxy.
- Re-ran a repository-relative citation existence/range check with shell/perl
  and absolute `/usr/bin/wc`; no full-path citation was missing or out of range.

## Status

codeQualityStatus: BLOCK

recommendation: REQUEST_CHANGES

blockers:

- Fix the stale `request_finished` API description so it includes `block_ids`
  and documents the HMA `request_finished_all_groups` dispatch path.
