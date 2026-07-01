---
slug: kvconnector-disaggregation-analysis-doc
status: plan-written
intent: clear
pending-action: write .omo/plans/kvconnector-disaggregation-analysis-doc.md
approach: Create a new Korean study/analysis document under docs/study that
  explains KVConnector and P/D disaggregation with bullet-first prose, Mermaid
  diagrams, compact code call-site tables, and explicit V1-only terminology.
---

# Draft: kvconnector-disaggregation-analysis-doc

## Components (topology ledger)
| id | outcome | status | evidence path |
| --- | --- | --- | --- |
| C1 | KVConnector concept, local/external token accounting, and control/data plane distinction | active | docs/study/v1_scheduler_schedule_flow_ko.md; docs/study/v1_kv_cache_lmcache_study_guide_ko.md |
| C2 | Role axes and KVConnectorBase_V1 scheduler/worker API map | active | docs/study/v1_pd_connector_communication_ko.md; vllm/distributed/kv_transfer/kv_connector/v1/base.py |
| C3 | ExampleConnector as minimal implementation, with LMCache/NIXL as contrast only | active | vllm/distributed/kv_transfer/kv_connector/v1/example_connector.py; vllm/distributed/kv_transfer/kv_connector/v1/lmcache_connector.py |
| C4 | Intra-instance scheduler-worker communication with diagrams and code call sites | active | docs/study/v1_pd_connector_communication_ko.md sections 3-5 |
| C5 | Inter-instance P/D disaggregation, single-turn and multi-turn bidirectional flows | active | docs/study/v1_pd_connector_communication_ko.md sections 6-7; examples/disaggregated/disaggregated_serving/disagg_proxy_multiturn.py |
| C6 | Failure and cleanup paths: delayed free, invalid blocks, reject cleanup, lease/TTL | active | docs/study/v1_pd_connector_communication_ko.md sections 5 and 8; docs/design/nixl_kv_cache_lease.md |

## Open assumptions (announced defaults)
| assumption | adopted default | rationale | reversible? |
| --- | --- | --- | --- |
| Target file | docs/study/v1_kvconnector_disaggregation_analysis_ko.md | Keeps Korean study docs together and names both KVConnector and disaggregation. | yes |
| Style | Bullet-first, diagram-first, short prose | User explicitly requested bullets and aggressive visualization. | yes |
| Mermaid style | White background with colored nodes, subgraphs, and legends | User explicitly requested readable white-background colored Mermaid. | yes |
| Scope depth | Code-reading analysis, not implementation proposal | Existing docs/study are learning guides; this doc synthesizes them. | yes |
| Failure section | Include as major final section | Connector semantics are incomplete without delayed free, invalid block, reject cleanup, lease/TTL. | yes |

## Findings (cited - path:lines)
- docs/study/v1_pd_connector_communication_ko.md explains the two role axes:
  instance `kv_role` and process `KVConnectorRole`.
- docs/study/v1_scheduler_schedule_flow_ko.md explains that scheduler accounting
  merges local prefix hit and external KV hit only for computed-token decisions.
- docs/study/v1_worker_execute_model_flow_ko.md explains worker-side connector
  execution as a forward wrapper plus per-layer attention hooks.
- docs/study/v1_pd_connector_communication_ko.md sections 3-5 cover
  intra-instance metadata/output transport through SchedulerOutput and
  ModelRunnerOutput.
- docs/study/v1_pd_connector_communication_ko.md sections 6-7 cover
  inter-instance proxy-mediated `kv_transfer_params`, single-turn P->D, and
  multi-turn bidirectional flows.
- docs/features/disagg_prefill.md Development describes the older
  Connector->LookupBuffer->Pipe model and must be treated as historical/V0
  contrast, not the V1 source of truth.

## Decisions (with rationale)
- Write one new document rather than editing existing guides, because the new
  document is a synthesis/analysis outline that spans all five study files.
- Use `KVConnectorBase_V1` as the source of truth for API naming.
- Put `num_computed_tokens = local + external` accounting in section 1 before
  API details, because it explains why connector has scheduler-side hooks.
- Treat ExampleConnector as the concrete minimal implementation, then use LMCache
  and NIXL only to show production concerns.
- Require every Mermaid diagram to include `%%{init: {"theme": "base",
  "themeVariables": {"background": "#ffffff"}} }%%` or equivalent white
  background styling plus colored classes.

## Scope IN
- Create docs/study/v1_kvconnector_disaggregation_analysis_ko.md.
- Cover KVConnector concept, token accounting, role axes, API surface,
  ExampleConnector, intra-instance communication, inter-instance P/D
  disaggregation, and failure/cleanup.
- Use Mermaid diagrams, tables, bullets, and short code snippets.
- Validate Markdown and internal links/paths.

## Scope OUT (Must NOT have)
- Do not modify production code or tests.
- Do not rewrite existing docs/study files.
- Do not present docs/features/disagg_prefill.md Development as current V1
  architecture.
- Do not make a long prose-only essay.
- Do not add unverified claims about current runtime behavior without a source
  path.

## Open questions
None. User approved creating the formal work plan and specified the Mermaid
style requirement.

## Approval gate
status: approved-for-plan
User approved writing the formal work plan. Execution has not been approved.
