# kvconnector-disaggregation-analysis-doc - Work Plan

## TL;DR (For humans)
<!-- Fill this LAST, after the detailed plan below is written, so it summarizes the REAL plan. -->
<!-- Plain English for a non-engineer: NO file paths, NO todo numbers, NO wave/agent/tool names. -->

**What you'll get:** A new Korean study document that explains KVConnector and
P/D disaggregation from the scheduler, worker, proxy, and cleanup perspectives.
It will be visual-first: bullet sections, colored Mermaid diagrams, tables, and
compact code call-site notes rather than long prose.

**Why this approach:** The plan starts with token accounting and role axes so
the later scheduler-worker and P/D flows are not ambiguous. It treats V1
`KVConnectorBase_V1` as the source of truth and uses older disaggregated-prefill
docs only as historical contrast.

**What it will NOT do:** It will not change code or tests, rewrite existing
study docs, or present the old Connector->LookupBuffer->Pipe model as current
V1 architecture.

**Effort:** Medium
**Risk:** Low - documentation-only, but terminology precision matters.
**Decisions to sanity-check:** Target path
`docs/study/v1_kvconnector_disaggregation_analysis_ko.md`; include
failure/cleanup as a full section; require white-background colored Mermaid.

Your next move: start work from this plan, or ask for a review/change to the
plan first. Full execution detail follows below.

---

> TL;DR (machine): Medium doc-only plan; create one Korean KVConnector/P-D disaggregation analysis doc with colored white-background Mermaid, code call-site tables, and markdown QA.

## Scope
### Must have
- Create `docs/study/v1_kvconnector_disaggregation_analysis_ko.md`.
- Explain KVConnector with a scheduler-first mental model:
  - why it exists: external KV hit, remote KV load/save, offload, disaggregation
  - how it differs from local prefix cache
  - how new-request computed-token accounting combines local and external hits
  - where RUNNING progress, spec tokens, placeholders, and lookahead fit as
    separate scheduler adjustments
- Explain both role axes:
  - `kv_role`: `kv_producer`, `kv_consumer`, `kv_both`
  - `KVConnectorRole`: `SCHEDULER`, `WORKER`
- Map `KVConnectorBase_V1` APIs by scheduler-side and worker-side responsibility.
- Analyze `ExampleConnector` as the minimum concrete connector implementation.
- Explain KVConnector communication in two categories:
  - intra instance: scheduler-side connector <-> worker-side connector
  - inter instance: prefill instance <-> decode instance
- Intra-instance section must cover the substance of
  `docs/study/v1_pd_connector_communication_ko.md` sections 3-5:
  - `KVConnectorMetadata` downlink through `SchedulerOutput`
  - worker forward wrapper and per-layer attention hooks
  - `KVConnectorOutput` uplink through `ModelRunnerOutput`
  - delayed block free through `request_finished` / `finished_sending`
- Inter-instance section must cover the substance of
  `docs/study/v1_pd_connector_communication_ko.md` sections 6-7:
  - proxy-driven prefill/decode request split
  - `kv_transfer_params` as HTTP/control-plane metadata
  - actual KV bytes moving through connector transport
  - single-turn P->D and multi-turn bidirectional D->P->D flows
- Include a failure/cleanup section covering:
  - decode admission reject cleanup
  - invalid block reporting and recompute/fail behavior
  - async load failure
  - request abort/finish
  - NIXL lease and decoder KV TTL as reference concepts
- Use bullet-heavy writing, tables, compact code snippets, and Mermaid diagrams.
- Mermaid diagrams must be readable on a white background and use color
  deliberately:
  - include a white-background Mermaid init block or equivalent
  - use colored `classDef` styles for scheduler, worker, proxy, metadata/control,
    KV/data, failure/cleanup, and storage/transport roles
  - add legends or labels where colors carry meaning
- Link back to the source study docs and code paths.

### Must NOT have (guardrails, anti-slop, scope boundaries)
- Do not modify production code, tests, or existing study documents.
- Do not treat `docs/features/disagg_prefill.md` Development as the current V1
  architecture. It may only appear as a historical/V0 contrast.
- Do not write long prose-only sections. Break explanations into bullets, tables,
  diagrams, and short paragraphs.
- Do not add unsupported claims about current runtime behavior without a source
  path or existing study-doc reference.
- Do not over-focus on NIXL transport internals before explaining the generic
  V1 connector boundary.
- Do not make the document an implementation proposal for a new connector.

## Verification strategy
> Zero human intervention - all verification is agent-executed.
- Test decision: none for runtime tests; this is a documentation-only change.
- QA framework:
  - Markdown lint for the created document.
  - Link/path existence checks for referenced local files.
  - Mermaid/style grep checks for white-background init and colored classDefs.
  - Scope grep checks to prevent V0 Connector->LookupBuffer->Pipe from being
    described as current V1 architecture.
- Evidence:
  - `.omo/evidence/task-<N>-kvconnector-disaggregation-analysis-doc.md`
  - `.omo/evidence/final-kvconnector-disaggregation-analysis-doc.md`

## Execution strategy
### Parallel execution waves
> Target 5-8 todos per wave. Fewer than 3 (except the final) means you under-split.
- Wave 1 builds the conceptual and API foundation.
- Wave 2 builds intra-instance and inter-instance communication sections.
- Wave 3 adds failure/cleanup, visual polish, and verification.
- Keep the file self-contained but avoid duplicating whole source documents.

### Dependency matrix
| Todo | Depends on | Blocks | Can parallelize with |
| --- | --- | --- | --- |
| 1 | none | 2, 3, 4, 5, 6, 7 | none |
| 2 | 1 | 3, 4, 5, 6, 7 | none |
| 3 | 1, 2 | 6, 7 | 4, 5 |
| 4 | 1, 2 | 6, 7 | 3, 5 |
| 5 | 1, 2 | 6, 7 | 3, 4 |
| 6 | 3, 4, 5 | 7 | none |
| 7 | 6 | final verification | none |

## Todos
> Implementation + Test = ONE todo. Never separate.
<!-- APPEND TASK BATCHES BELOW THIS LINE WITH edit/apply_patch - never rewrite the headers above. -->
- [x] 1. Create the document scaffold and visual style contract.
  What to do / Must NOT do: Create
  `docs/study/v1_kvconnector_disaggregation_analysis_ko.md` with a Korean title,
  short baseline note, reading prerequisites, a table of contents, and a
  reusable Mermaid style note. Include the requirement that every Mermaid diagram
  uses a white background and colored classes. Do not fill body sections with
  placeholder prose beyond short TODO markers that this same plan removes later.
  Parallelization: Wave 1 | Blocked by: none | Blocks: 2, 3, 4, 5, 6, 7
  References (executor has NO interview context - be exhaustive):
  `docs/study/v1_pd_connector_communication_ko.md`, `docs/study/v1_pd_disaggregation_lmcache_study_guide_ko.md`,
  `docs/study/v1_scheduler_schedule_flow_ko.md`, `docs/study/v1_worker_execute_model_flow_ko.md`,
  `docs/study/v1_kv_cache_lmcache_study_guide_ko.md`.
  Acceptance criteria (agent-executable): `test -f docs/study/v1_kvconnector_disaggregation_analysis_ko.md` and `rg -n "themeVariables|background.*#ffffff|classDef" docs/study/v1_kvconnector_disaggregation_analysis_ko.md`.
  QA scenarios (name the exact tool + invocation): happy: `rg -n "^#|^##|classDef" docs/study/v1_kvconnector_disaggregation_analysis_ko.md` shows the scaffold and style markers. failure: `rg -n "Connector.*LookupBuffer.*Pipe" docs/study/v1_kvconnector_disaggregation_analysis_ko.md` must not present that sequence as current V1 architecture. Evidence `.omo/evidence/task-1-kvconnector-disaggregation-analysis-doc.md`.
  Commit: Y | docs(study): add kvconnector disaggregation analysis guide scaffold

- [x] 2. Write sections 1-3: concept, token accounting, roles, and API map.
  What to do / Must NOT do: Fill sections:
  `1. KVConnector란 무엇인가`, `2. 두 가지 role 축`, and
  `3. KVConnectorBase_V1 API 지도`. Include:
  - KVConnector purpose: external KV hit, remote KV load/save, offload, P/D
    disaggregation.
  - local prefix cache vs external KV hit comparison table.
  - computed-token accounting:
    `num_computed_tokens = num_new_local_computed_tokens + num_external_computed_tokens`
    for a new WAITING request, plus bullets explaining RUNNING progress, spec
    tokens, placeholders, and lookahead as separate scheduler accounting/allocation
    adjustments.
  - control plane vs data plane short definition.
  - `kv_role` vs `KVConnectorRole` 2-axis matrix.
  - scheduler-side and worker-side API tables for `KVConnectorBase_V1`.
  Do not imply external KV hit is already resident in local GPU memory.
  Parallelization: Wave 1 | Blocked by: 1 | Blocks: 3, 4, 5, 6, 7
  References (executor has NO interview context - be exhaustive):
  `docs/study/v1_scheduler_schedule_flow_ko.md:192`,
  `docs/study/v1_scheduler_schedule_flow_ko.md:423`,
  `docs/study/v1_kv_cache_lmcache_study_guide_ko.md:147`,
  `docs/study/v1_pd_connector_communication_ko.md:14`,
  `vllm/config/kv_transfer.py`,
  `vllm/distributed/kv_transfer/kv_connector/v1/base.py`.
  Acceptance criteria (agent-executable): `rg -n "num_new_local_computed_tokens|num_external_computed_tokens|kv_role|KVConnectorRole|get_num_new_matched_tokens|wait_for_layer_load" docs/study/v1_kvconnector_disaggregation_analysis_ko.md`.
  QA scenarios (name the exact tool + invocation): happy: `rg -n "local prefix|external KV|제어 평면|데이터 평면" docs/study/v1_kvconnector_disaggregation_analysis_ko.md` finds all four concepts. failure: `rg -n "external KV.*이미.*GPU|외부 KV.*이미.*GPU" docs/study/v1_kvconnector_disaggregation_analysis_ko.md` should find no misleading phrasing. Evidence `.omo/evidence/task-2-kvconnector-disaggregation-analysis-doc.md`.
  Commit: Y | docs(study): explain kvconnector roles and token accounting

- [x] 3. Write section 4: ExampleConnector and concrete implementation reading.
  What to do / Must NOT do: Add a section explaining how to read
  `example_connector.py` as a minimal connector. Map each key API method to what
  the example does, then add a compact contrast table showing what production
  connectors such as LMCache/NIXL add: async completion, transport, request
  tracking, invalid blocks, leases/TTL. Do not deep-dive into NIXL transport
  implementation here; save that for inter-instance/failure sections.
  Parallelization: Wave 2 | Blocked by: 1, 2 | Blocks: 6, 7
  References (executor has NO interview context - be exhaustive):
  `vllm/distributed/kv_transfer/kv_connector/v1/example_connector.py`,
  `vllm/distributed/kv_transfer/kv_connector/v1/lmcache_connector.py`,
  `vllm/distributed/kv_transfer/kv_connector/v1/lmcache_integration/vllm_v1_adapter.py`,
  `docs/study/v1_pd_disaggregation_lmcache_study_guide_ko.md:327`.
  Acceptance criteria (agent-executable): `rg -n "ExampleConnector|LMCacheConnectorV1|NIXL|최소 구현|production" docs/study/v1_kvconnector_disaggregation_analysis_ko.md`.
  QA scenarios (name the exact tool + invocation): happy: `rg -n "get_num_new_matched_tokens|update_state_after_alloc|build_connector_meta|request_finished" docs/study/v1_kvconnector_disaggregation_analysis_ko.md` shows these APIs in the ExampleConnector section or shared API map. failure: `rg -n "NIXL.*세부.*먼저|transport.*먼저" docs/study/v1_kvconnector_disaggregation_analysis_ko.md` must not show the section prioritizing transport internals before the generic connector contract. Evidence `.omo/evidence/task-3-kvconnector-disaggregation-analysis-doc.md`.
  Commit: Y | docs(study): map example connector implementation

- [x] 4. Write section 5-6: communication categories and intra-instance flow.
  What to do / Must NOT do: Add:
  - `KVConnector 통신의 두 종류` overview.
  - `Intra instance 통신` section based on sections 3-5 of
    `v1_pd_connector_communication_ko.md`.
  Include at least two Mermaid diagrams:
  - big picture showing intra vs inter communication.
  - scheduler-worker sequence showing metadata downlink, worker lifecycle,
    output uplink, and scheduler post-processing.
  Include code call-site tables for:
  - `SchedulerOutput.kv_connector_metadata`
  - `bind_connector_metadata`, `start_load_kv`
  - `maybe_transfer_kv_layer`
  - `KVConnectorOutput`
  - `request_finished` / delayed free
  Ensure diagrams use white background and colored classes.
  Parallelization: Wave 2 | Blocked by: 1, 2 | Blocks: 6, 7
  References (executor has NO interview context - be exhaustive):
  `docs/study/v1_pd_connector_communication_ko.md:66`,
  `docs/study/v1_pd_connector_communication_ko.md:90`,
  `docs/study/v1_pd_connector_communication_ko.md:151`,
  `docs/study/v1_pd_connector_communication_ko.md:265`,
  `vllm/v1/core/sched/output.py`,
  `vllm/v1/worker/kv_connector_model_runner_mixin.py`,
  `vllm/model_executor/layers/attention/kv_transfer_utils.py`,
  `vllm/v1/outputs.py`.
  Acceptance criteria (agent-executable): `rg -n "intra|SchedulerOutput|ModelRunnerOutput|KVConnectorMetadata|KVConnectorOutput|request_finished|finished_sending" docs/study/v1_kvconnector_disaggregation_analysis_ko.md`.
  QA scenarios (name the exact tool + invocation): happy: `rg -n "sequenceDiagram|flowchart|classDef.*scheduler|classDef.*worker|classDef.*metadata" docs/study/v1_kvconnector_disaggregation_analysis_ko.md` confirms diagrams and colored roles. failure: `rg -n "직접 호출|direct call" docs/study/v1_kvconnector_disaggregation_analysis_ko.md` must not claim scheduler-side and worker-side connector directly call each other. Evidence `.omo/evidence/task-4-kvconnector-disaggregation-analysis-doc.md`.
  Commit: Y | docs(study): document intra instance connector communication

- [x] 5. Write section 7: inter-instance P/D disaggregation.
  What to do / Must NOT do: Add an inter-instance section based on sections 6-7
  of `v1_pd_connector_communication_ko.md`. Include:
  - single-turn P->D sequence.
  - multi-turn bidirectional D->P->D sequence.
  - `kv_transfer_params` field inventory table.
  - explicit control-plane vs data-plane mapping.
  - proxy responsibility: copying/injecting metadata such as `remote_host`, not
    moving tensors.
  Diagrams must use white background and colors for proxy, P, D, control-plane
  metadata, and KV data path.
  Parallelization: Wave 2 | Blocked by: 1, 2 | Blocks: 6, 7
  References (executor has NO interview context - be exhaustive):
  `docs/study/v1_pd_connector_communication_ko.md:331`,
  `docs/study/v1_pd_connector_communication_ko.md:400`,
  `docs/study/v1_pd_connector_communication_ko.md:448`,
  `docs/study/v1_pd_connector_communication_ko.md:559`,
  `examples/disaggregated/disaggregated_serving/disagg_proxy_multiturn.py`,
  `vllm/entrypoints/serve/disagg/protocol.py`,
  `vllm/entrypoints/serve/disagg/serving.py`,
  `vllm/outputs.py`.
  Acceptance criteria (agent-executable): `rg -n "inter instance|kv_transfer_params|do_remote_decode|do_remote_prefill|remote_block_ids|remote_host|bidirectional|ConversationKVCache" docs/study/v1_kvconnector_disaggregation_analysis_ko.md`.
  QA scenarios (name the exact tool + invocation): happy: `rg -n "P->D|D->P|제어 평면|데이터 평면|proxy|remote_host" docs/study/v1_kvconnector_disaggregation_analysis_ko.md` confirms the intended flow. failure: `rg -n "proxy.*tensor|proxy.*KV tensor|프록시.*텐서" docs/study/v1_kvconnector_disaggregation_analysis_ko.md` must not claim proxy moves KV tensors. Evidence `.omo/evidence/task-5-kvconnector-disaggregation-analysis-doc.md`.
  Commit: Y | docs(study): document inter instance disaggregation flow

- [x] 6. Write section 8-9: failure/cleanup and final mental model.
  What to do / Must NOT do: Add failure/cleanup as a major section, then finish
  with a one-page mental model summary. Cover:
  - delayed free: `request_finished` pins blocks until `finished_sending`.
  - `invalid_block_ids` and recompute/fail behavior at a high level.
  - admission-before-decode reject cleanup through synthetic `abort_immediately`.
  - lease/heartbeat/TTL as NIXL reference concepts.
  - final summary: token accounting decides "how much"; connector lifecycle
    decides "how KV is obtained"; proxy/control-plane metadata decides "which
    remote KV to use".
  Use at least one Mermaid failure/cleanup sequence and one compact checklist.
  Parallelization: Wave 3 | Blocked by: 3, 4, 5 | Blocks: 7
  References (executor has NO interview context - be exhaustive):
  `docs/study/v1_pd_connector_communication_ko.md:265`,
  `docs/study/v1_pd_connector_communication_ko.md:576`,
  `docs/study/v1_kv_cache_lmcache_study_guide_ko.md:479`,
  `docs/design/nixl_kv_cache_lease.md`,
  `vllm/entrypoints/openai/engine/serving.py`,
  `vllm/v1/engine/async_llm.py`,
  `vllm/v1/engine/core.py`,
  `vllm/v1/core/sched/scheduler.py`.
  Acceptance criteria (agent-executable): `rg -n "request_finished|finished_sending|invalid_block_ids|abort_immediately|lease|TTL|heartbeat|한 줄 요약|mental model" docs/study/v1_kvconnector_disaggregation_analysis_ko.md`.
  QA scenarios (name the exact tool + invocation): happy: `rg -n "failure|cleanup|recompute|delayed free|lease|TTL|abort" docs/study/v1_kvconnector_disaggregation_analysis_ko.md` confirms cleanup coverage. failure: `rg -n "실패.*무시|cleanup.*불필요|free.*즉시.*항상" docs/study/v1_kvconnector_disaggregation_analysis_ko.md` must not contain misleading cleanup simplifications. Evidence `.omo/evidence/task-6-kvconnector-disaggregation-analysis-doc.md`.
  Commit: Y | docs(study): add connector failure cleanup analysis

- [x] 7. Polish, link-check, Mermaid-style check, and Markdown QA.
  What to do / Must NOT do: Remove TODO placeholders, reduce duplicated prose,
  verify local file references exist, ensure every Mermaid diagram has white
  background/color styling, and run Markdown lint. Keep the final doc readable
  as a study document, not a raw source-map dump.
  Parallelization: Wave 3 | Blocked by: 6 | Blocks: final verification
  References (executor has NO interview context - be exhaustive):
  `docs/study/v1_kvconnector_disaggregation_analysis_ko.md`,
  `.markdownlint.yaml`,
  `.pre-commit-config.yaml`.
  Acceptance criteria (agent-executable):
  - `uvx pre-commit run markdownlint-cli2 --files docs/study/v1_kvconnector_disaggregation_analysis_ko.md` exits 0.
  - `git diff --check -- docs/study/v1_kvconnector_disaggregation_analysis_ko.md` exits 0.
  - `rg -n "TODO|TBD|<fill|Connector.*LookupBuffer.*Pipe.*현재|proxy.*KV tensor" docs/study/v1_kvconnector_disaggregation_analysis_ko.md` returns no problematic matches.
  - `rg -n "themeVariables|#ffffff|classDef" docs/study/v1_kvconnector_disaggregation_analysis_ko.md` confirms Mermaid styling.
  QA scenarios (name the exact tool + invocation): happy: run the four acceptance commands and save output. failure: intentionally inspect any `rg` hits from the negative check and patch them before completion. Evidence `.omo/evidence/task-7-kvconnector-disaggregation-analysis-doc.md`.
  Commit: Y | docs(study): polish kvconnector disaggregation study guide

## Final verification wave
> Runs in parallel after ALL todos. ALL must APPROVE. Surface results and wait for the user's explicit okay before declaring complete.
- [x] F1. Plan compliance audit
  Verify the created document covers every Must Have, includes the explicit
  Must NOT guardrails, and follows the approved outline. Evidence:
  `.omo/evidence/final-kvconnector-disaggregation-analysis-doc.md`.
- [x] F2. Code quality review
  Documentation quality review: check terminology consistency, V1 vs V0
  separation, local path references, and whether code snippets match cited
  call sites. Evidence:
  `.omo/evidence/final-kvconnector-disaggregation-analysis-doc.md`.
- [x] F3. Real manual QA
  Open/render the Markdown preview if available, or use a Mermaid-capable
  markdown preview/checker if present. At minimum, inspect Mermaid source blocks
  for init/style/class coverage and run Markdown lint. Evidence:
  `.omo/evidence/final-kvconnector-disaggregation-analysis-doc.md`.
- [x] F4. Scope fidelity
  Confirm only the new study document and `.omo/evidence` artifacts were changed
  by execution, unless the user separately approves more. Evidence:
  `.omo/evidence/final-kvconnector-disaggregation-analysis-doc.md`.

## Commit strategy
- One documentation commit is acceptable after all todos pass:
  `docs(study): add kvconnector disaggregation analysis guide`.
- Do not commit `.omo/evidence` unless the repository convention for this work
  explicitly wants planning/evidence artifacts committed.
- Include AI-assistance trailers if the user requests a commit, consistent with
  root `AGENTS.md`.

## Success criteria
- `docs/study/v1_kvconnector_disaggregation_analysis_ko.md` exists.
- The document is in Korean and uses bullet-heavy sections, tables, Mermaid
  diagrams, and compact code call-site snippets.
- Section coverage includes:
  - KVConnector concept and token accounting
  - two role axes
  - `KVConnectorBase_V1` scheduler/worker API map
  - ExampleConnector implementation analysis
  - intra-instance communication
  - inter-instance P/D disaggregation
  - failure/cleanup
  - final mental model summary
- Mermaid diagrams are readable on a white background and use colored classes or
  equivalent styling.
- Markdown lint and diff checks pass.
- The document does not present the V0 Connector->LookupBuffer->Pipe model as
  current V1 architecture.
