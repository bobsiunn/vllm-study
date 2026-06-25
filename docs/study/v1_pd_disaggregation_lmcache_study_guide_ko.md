# V1 P/D Disaggregation Serving과 Connector 학습 가이드

!!! note
    이 문서는 구현 설계서가 아니라 학습 로드맵입니다. V1 KV cache core를
    읽은 뒤, 노드 간 prefill/decode disaggregation serving에서 요청, KV
    metadata, connector, LMCache 경계가 어떻게 이어지는지 이해하는 데
    사용하세요. scheduler-side/worker-side connector의 실제 통신 경로를 그림으로
    보려면 동반 문서 [V1 P/D Connector 통신 구조](v1_pd_connector_communication_ko.md)를
    참고하세요.

!!! warning "Baseline"
    이 가이드는 upstream **`v0.23.0`** 트리에 대해 모든 source-map 경로와 핵심
    심볼(`KVConnectorRole`, `KVConnectorBase_V1.{get_num_new_matched_tokens,
    update_state_after_alloc, build_connector_meta, wait_for_layer_load,
    save_kv_layer, request_finished}`, attention hook `maybe_transfer_kv_layer`)이
    검증되었습니다. KV transfer connector 추상화는 V1 (`KVConnectorBase_V1`)
    기준입니다. 구 `docs/features/disagg_prefill.md`의 "Development"
    섹션(Connector → LookupBuffer → Pipe 3계층)은 **V0 모델**이며 현재 코드에
    존재하지 않으므로 이 가이드에서는 참조하지 않습니다.

## 목표

이 가이드는 다음 네 가지 연결된 흐름에 대한 코드 레벨 mental model을
만드는 데 도움을 줍니다.

1. client 또는 proxy가 하나의 요청을 prefill instance와 decode instance로
   나누어 전달하는 방식.
2. `kv_transfer_params`가 request, engine, scheduler, output 경계를 지나며
   remote KV ownership을 표현하는 방식.
3. V1 KV connector API가 scheduler connector와 worker connector를 나누고,
   attention layer 전후의 KV load/save를 조율하는 방식.
4. LMCache connector가 vLLM connector API 위에서 외부 KV 저장소 또는 전송
   backend와 연결되는 방식.

이 가이드를 마치면 proxy가 prefill 결과를 decode 요청에 붙이는 흐름부터,
decode worker가 remote KV를 attention 계산 전에 load하고 요청 종료 시
connector state를 정리하는 흐름까지 한 요청을 따라갈 수 있어야 합니다.

## 선수 지식

다음 개념이 익숙하지 않다면 먼저 읽어보세요.

- [V1 KV Cache와 LMCache Connector 학습 가이드](v1_kv_cache_lmcache_study_guide_ko.md):
  local KV block allocation, worker block table, LMCache connector entry point.
- 용어의 V1 기준 출처(두 파일):
    - `vllm/config/kv_transfer.py`의 `kv_role`(`kv_producer` / `kv_consumer` /
      `kv_both`)과 `is_kv_producer()` / `is_kv_consumer()` 헬퍼 → producer 측(=
      prefill instance)과 consumer 측(= decode instance)을 정의. 코드에
      "prefill/decode instance"라는 심볼이 따로 있는 게 아니라, instance의
      `kv_role`로 그 역할이 표현됩니다.
    - `vllm/distributed/kv_transfer/kv_connector/v1/base.py`의
      `KVConnectorRole`(SCHEDULER / WORKER) → scheduler connector와 worker
      connector를 정의 (같은 connector를 두 role로 띄운 것).
  (구 `disagg_prefill.md`는 Connector→LookupBuffer→Pipe라는 V0 모델을 설명하므로
  용어 출처로 쓰지 마세요.)
- [NixlConnector Usage Guide](features/nixl_connector_usage.md): 노드 간 KV
  transfer, side channel, producer/consumer role, `kv_transfer_params` 예시.
- [Automatic Prefix Caching](design/prefix_caching.md): local prefix cache hit과
  external KV hit을 구분하기 위한 block hash 개념.

## Source Map

Serving과 request metadata 파일:

- `vllm/entrypoints/openai/engine/serving.py`: OpenAI serving path에서
  `kv_transfer_params`, `do_remote_prefill`, rejection cleanup을 처리합니다.
- `vllm/entrypoints/serve/disagg/protocol.py`: disaggregated serving 전용 request
  and response schema를 정의합니다.
- `vllm/entrypoints/serve/disagg/serving.py`: token-level disaggregated serving
  handler입니다.
- `vllm/outputs.py`: generation output이 `kv_transfer_params`를 담아 proxy 또는
  client로 되돌아가는 경계입니다.

Configuration과 role selection 파일:

- `vllm/config/kv_transfer.py`: `KVTransferConfig`, `kv_connector`, `kv_role`,
  `kv_load_failure_policy`, `kv_connector_extra_config`를 정의합니다.
- `vllm/config/vllm.py`: KV transfer config 후처리, compatibility check, HMA와
  CUDA graph 관련 connector 조건을 확인합니다.
- `vllm/engine/arg_utils.py`: CLI와 engine args에서 `--kv-transfer-config`가
  `VllmConfig`로 들어가는 경로입니다.

Connector 공통 경계 파일:

- `vllm/distributed/kv_transfer/kv_connector/v1/base.py`: V1 scheduler-side와
  worker-side connector API의 중심입니다.
- `vllm/distributed/kv_transfer/kv_connector/factory.py`: connector name을 실제
  connector class로 해석합니다.
- `vllm/distributed/kv_transfer/kv_transfer_state.py`: process-local KV transfer
  group state를 관리합니다.
- `vllm/distributed/kv_transfer/__init__.py`: global KV transfer group accessor를
  제공합니다.

Scheduler와 worker integration 파일:

- `vllm/v1/core/sched/scheduler.py`: connector lookup, allocation, metadata
  construction, invalid block handling, request completion을 연결합니다.
- `vllm/v1/core/sched/output.py`: scheduler가 worker에 넘기는 request data와
  connector metadata field를 확인합니다.
- `vllm/v1/worker/kv_connector_model_runner_mixin.py`: model runner forward
  주변에서 connector lifecycle을 실행합니다.
- `vllm/v1/worker/gpu/kv_connector.py`: GPU worker와 connector 사이의 V1
  integration path입니다.
- `vllm/model_executor/layers/attention/kv_transfer_utils.py`: attention layer
  진입 전 `wait_for_layer_load()`, 종료 후 `save_kv_layer()`를 호출합니다.

LMCache와 비교용 connector 파일:

- `vllm/distributed/kv_transfer/kv_connector/v1/lmcache_connector.py`:
  `LMCacheConnectorV1` wrapper입니다.
- `vllm/distributed/kv_transfer/kv_connector/v1/lmcache_mp_connector.py`:
  standalone `lmcache server`를 사용하는 multi-process connector입니다.
- `vllm/distributed/kv_transfer/kv_connector/v1/lmcache_integration/`: vLLM V1
  adapter, multi-process adapter, utility bridge입니다.
- `vllm/distributed/kv_transfer/kv_connector/v1/example_connector.py`: connector
  API를 이해하기 위한 최소 구현입니다.
- `vllm/distributed/kv_transfer/kv_connector/v1/nixl/`: 노드 간 P/D transfer의
  대표 구현체입니다. LMCache disagg 예제에서도 underlying KV transmission
  비교 대상으로 유용합니다.

예제와 테스트:

- `examples/disaggregated/disaggregated_serving/`: proxy-based P/D serving 예제.
- `examples/disaggregated/lmcache/`: LMCache 기반 disaggregated prefill 예제.
- `tests/v1/kv_connector/unit/`: connector lifecycle, failure recovery,
  bidirectional transfer, LMCache, NIXL 단위 테스트.
- `tests/v1/kv_connector/nixl_integration/`: NIXL 기반 P/D accuracy와 edge case
  integration test.

## Code Reading Path

V1 core KV cache를 이미 읽었다면, 이제는 local block ownership보다
request-level ownership handoff를 먼저 따라가세요.

1. V1 용어 정의 (두 파일에서 따로 읽습니다)
   - `vllm/config/kv_transfer.py`: `kv_role`(`kv_producer` / `kv_consumer` /
     `kv_both`)과 `is_kv_producer()` / `is_kv_consumer()`가 producer 측(=
     `prefill instance`)과 consumer 측(= `decode instance`)을 정의합니다.
     코드에 "prefill/decode instance" 심볼은 없고, instance의 `kv_role`이 그
     역할을 표현합니다.
   - `vllm/distributed/kv_transfer/kv_connector/v1/base.py`의
     `KVConnectorRole`(SCHEDULER / WORKER): `scheduler connector`와
     `worker connector`는 별개 클래스가 아니라 같은 connector를 두 role로 띄운
     것입니다.
   - 확인할 것: 한 vLLM instance의 `kv_role`(producer/consumer)과 그 안의
     connector `KVConnectorRole`(scheduler/worker)은 서로 다른 축의 역할입니다.
     P와 D가 별도 instance일 때 connector가 왜 scheduler-side와 worker-side로
     나뉘나요?
2. `examples/disaggregated/disaggregated_serving/`와
   `examples/disaggregated/lmcache/`
   - proxy가 prefill server와 decode server에 어떤 순서로 request를 보내는지
     확인합니다.
   - 확인할 것: prefill response에서 나온 `kv_transfer_params`는 decode request의
     어느 field로 들어가나요?
3. `vllm/config/kv_transfer.py` (정의·검증만 — 동작 제한 코드는 소비처에 있음)
   - `KVTransferConfig`, `is_kv_transfer_instance` / `is_kv_producer` /
     `is_kv_consumer` property(109~119), `__post_init__`의 검증(kv_connector를
     켜면 kv_role 필수, 103~107)을 읽습니다.
   - 확인할 것: `kv_role`이 실제 동작을 제한하는 곳은 helper **소비처**입니다.
     ① `is_kv_transfer_instance` → KV transfer on/off 자체
     (`v1/attention/selector.py:87`, `kv_transfer_state.py:78`, `config/vllm.py`),
     ② `is_kv_producer`/`is_kv_consumer` → connector 내부 send/recv 분기
     (`mooncake`/`p2p`/`moriio` connector). 단 **NIXL은 kv_role을 쓰지 않고**
     per-request `do_remote_decode`/`do_remote_prefill`로 방향을 정합니다
     (`nixl/scheduler.py:602`) — role 게이팅 방식이 connector마다 다릅니다.
4. `vllm/config/vllm.py`와 `vllm/engine/arg_utils.py`
   - `arg_utils.py`는 `--kv-transfer-config` JSON을 `KVTransferConfig`로
     파싱·전달만 합니다(`arg_utils.py:1474, 2245`). 실제 게이팅은 전부
     `config/vllm.py`에 있습니다: `_post_init_kv_transfer_config`(757),
     `_verify_kv_transfer_compat`(804), HMA 자동 비활성화(1479~1500), CUDA graph
     PIECEWISE 강등(1244~1269).
   - 확인할 것: connector가 켜질 때 어떤 기능이 제한되거나 자동 조정되나요?
     → 정리된 표는 동반 문서
     [Connector 통신 구조 §8](v1_pd_connector_communication_ko.md#8-connector를-켜면-바뀌는-것-제한자동조정)
     참고. 요약: offload→connector 자동주입, CUDA graph→PIECEWISE, HMA 미지원 시
     hybrid KV cache off (제한) / `expandable_segments`·routed-experts·일부
     chunked-prefill 조합은 거부 (에러).
5. `vllm/entrypoints/openai/engine/serving.py` → `async_llm.py` → `core.py` → connector
   - 감지는 `serving.py:413` `_with_kv_transfer_rejection_cleanup`(ErrorResponse/예외
     시 `notify_kv_transfer_request_rejected` 호출, 435)에서 시작하지만, 실제 block
     free는 `async_llm.py:723`이 **`abort_immediately=True` 합성 요청**을 엔진에 넣어
     (`core.py:373`) 평소의 `request_finished`(base.py:542) 훅을 재사용합니다.
     P 블록은 `nixl/scheduler.py:609~620`에서 `do_remote_prefill`이 아직 True인 걸
     단서로 풀립니다.
   - 확인할 것: engine admission 전에 request가 reject되면 remote-prefill block
     cleanup은 어디서 시작되나요? → 전체 흐름(sequence)은 동반 문서
     [Connector 통신 구조 §8](v1_pd_connector_communication_ko.md#8-admission-전-reject-시-remote-prefill-block-cleanup)
     참고.
6. `vllm/entrypoints/serve/disagg/protocol.py`,
   `vllm/entrypoints/serve/disagg/serving.py`, `vllm/outputs.py`
   - request schema와 output schema에서 `kv_transfer_params`가 어떻게 보존되는지
     읽습니다.
   - 확인할 것: serving layer는 KV tensor를 알지 못하면서 어떤 metadata만
     운반하나요?
7. `vllm/distributed/kv_transfer/kv_connector/v1/base.py`
   - scheduler-side method와 worker-side method를 나누어 읽습니다.
   - 확인할 것: allocation 전 lookup, allocation 후 metadata build, forward 전
     load, forward 후 save, request finish cleanup이 각각 어떤 API에 대응하나요?
8. `vllm/v1/core/sched/scheduler.py`
   - `get_num_new_matched_tokens`, `update_state_after_alloc`,
     `build_connector_meta`, `request_finished`, `invalid_block_ids`를 검색해서
     읽습니다.
   - 확인할 것: local prefix hit과 external KV hit은 scheduling decision에서
     어디서 합쳐지고 어디서 분리되나요?
9. `vllm/v1/worker/kv_connector_model_runner_mixin.py`와
   `vllm/v1/worker/gpu/kv_connector.py`
   - worker가 connector metadata를 받아 forward 주변에서 어떤 lifecycle을
     실행하는지 읽습니다.
   - 확인할 것: connector가 no-forward path를 만들 수 있는 조건은 무엇인가요?
10. `vllm/model_executor/layers/attention/kv_transfer_utils.py`
    - attention layer forward를 감싸는 `maybe_transfer_kv_layer()` 데코레이터를
      읽습니다. 이 wrapper는 진입 시 `connector.wait_for_layer_load(layer_name)`,
      종료 시 `connector.save_kv_layer(layer_name, kv_cache, attn_metadata)`를
      호출하며, V1 KV transfer group이 없으면 no-op이 됩니다.
    - 확인할 것: remote KV load가 attention computation보다 반드시 먼저
      완료되어야 하는 layer-level boundary는 어디인가요? (`wait_for_layer_load`가
      `func()` 실행 앞에 있는 이유)
11. `vllm/distributed/kv_transfer/kv_connector/v1/example_connector.py`
    - 최소 구현으로 V1 connector API의 control flow를 확인합니다.
    - 확인할 것: example은 어떤 부분을 단순화하고, production connector에서는
      어떤 부분이 transport/failure handling으로 커지나요?
12. `vllm/distributed/kv_transfer/kv_connector/v1/lmcache_connector.py`와
    `vllm/distributed/kv_transfer/kv_connector/v1/lmcache_integration/`
    - LMCache가 vLLM request, block id, token range, layer-wise KV를 어떤 adapter
      형태로 변환하는지 읽습니다.
    - 확인할 것: LMCache hit은 V1 scheduler에서 `num_external_computed_tokens`와
      어떤 관계를 갖나요?
13. `vllm/distributed/kv_transfer/kv_connector/v1/nixl/`
    - NIXL은 먼저 깊게 구현할 대상이 아니라 비교 대상으로 읽습니다.
    - 확인할 것: LMCache 중심 분석에서 NIXL은 remote transfer, side channel,
      lease, heartbeat, producer/consumer role을 이해하기 위한 reference
      implementation으로 사용하세요.

## Core Mental Model

P/D disaggregation의 첫 번째 boundary는 KV block이 아니라 request입니다.

- Prefill instance는 prompt prefill을 수행하고, decode instance가 읽을 수 있는
  remote KV 위치와 block metadata를 `kv_transfer_params`로 돌려줍니다.
- Proxy 또는 client는 이 metadata를 decode request에 붙입니다.
- Decode instance는 local prefix cache hit처럼 보이는 것이 아니라 external KV
  hit으로 scheduler에 반영하고, 필요한 local block을 할당한 뒤 worker에게
  connector metadata를 전달합니다.
- Worker connector는 attention layer가 KV를 읽기 전에 remote KV를 local KV
  cache tensor로 load하고, 필요하면 forward 이후 새 KV를 외부에 save합니다.

즉, local V1 KV cache에서 `block_id`가 physical page ownership을 뜻했다면,
P/D disaggregation에서 `kv_transfer_params`는 remote producer, remote block,
connection, lease, request lifecycle을 연결하는 request-level contract입니다.

Checkpoint 질문:

- `kv_transfer_params`는 tensor data인가요, tensor data를 찾기 위한 metadata인가요?
- prefill response와 decode request 사이에서 이 metadata를 보관하는 주체는
  누구인가요?
- scheduler connector와 worker connector가 같은 process에 없을 수도 있다는
  사실이 API 모양에 어떤 영향을 주나요?

## Serving and Proxy Flow

노드 간 P/D serving은 보통 세 단계로 이해하면 됩니다.

1. Client가 proxy로 원 요청을 보냅니다.
2. Proxy가 prefill instance에 요청을 보내고, prefill instance는 prompt KV를
   만들고 `kv_transfer_params`를 반환합니다.
3. Proxy가 decode instance에 원 요청과 prefill `kv_transfer_params`를 함께
   보내고, decode instance는 remote KV를 load한 뒤 generation을 수행합니다.

여기서 중요한 점은 proxy가 KV tensor를 직접 다루지 않는다는 것입니다. Proxy는
request body와 response body 안의 metadata를 전달하고, 실제 KV transfer는
decode worker와 connector가 수행합니다.

Checkpoint 질문:

- prefill instance가 반환하는 `kv_transfer_params`에는 decode가 remote KV를
  찾기 위해 어떤 identity가 필요하나요?
- request가 decode engine admission 전에 reject되면, prefill side에 pin된 KV는
  어떤 cleanup path로 해제되나요?
- streaming response에서는 final chunk와 intermediate chunk 중 어디에 connector
  metadata가 실릴 수 있나요?

## Connector Lifecycle

V1 connector는 scheduler-side decision과 worker-side execution을 분리합니다.

Scheduler-side에서 확인할 lifecycle:

1. request의 `kv_transfer_params`를 보고 external KV hit 가능성을 계산합니다.
2. allocation 이후 local block ID와 remote KV metadata를 연결합니다.
3. worker가 이해할 connector metadata를 `SchedulerOutput`에 넣습니다.
4. request finished, aborted, rejected, invalid block event에 따라 connector state를
   정리합니다.

Worker-side에서 확인할 lifecycle:

1. scheduler가 만든 connector metadata를 batch 또는 model runner state에
   반영합니다.
2. attention layer 진입 전에 해당 layer의 KV load가 완료될 때까지 기다립니다.
3. attention layer 종료 후 새 KV layer를 connector에 save합니다.
4. forward 없이 connector operation만 수행하는 path가 가능한지 확인합니다.

Checkpoint 질문:

- allocation 전 connector lookup과 allocation 후 connector metadata build를
  분리하는 이유는 무엇인가요?
- connector가 full request 단위가 아니라 layer 단위 load/save hook을 갖는 이유는
  무엇인가요?
- failure policy가 `fail`인지 recovery인지에 따라 scheduler는 어떤 block을
  invalid로 보나요?

## LMCache Focus

LMCache를 중심으로 볼 때는 NIXL 자체보다 adapter boundary가 먼저입니다.

- `LMCacheConnectorV1`는 vLLM V1 connector API를 구현하는 wrapper입니다.
- `lmcache_integration/vllm_v1_adapter.py`는 vLLM의 request/block/layer 개념을
  LMCache가 이해하는 lookup, load, store operation으로 변환합니다.
- `LMCacheMPConnector`는 standalone `lmcache server`를 통해 여러 vLLM instance가
  KV를 공유하는 mode를 보여줍니다.

LMCache 분석의 핵심 질문은 다음입니다.

- LMCache hit은 scheduler에서 몇 token의 external computed token으로
  표현되나요?
- hit token range와 vLLM block allocation range는 항상 같은 block boundary를
  공유하나요?
- LMCache save는 request completion 시점에 끝나나요, attention layer별로
  점진적으로 발생하나요?
- request abort 또는 rejection이 발생하면 LMCache 쪽 pending load/save state는
  어디서 정리되나요?

## NIXL을 비교 대상으로 읽는 법

NIXL은 노드 간 P/D transfer의 transport-heavy reference implementation으로
사용하세요. LMCache study의 첫 번째 대상은 아니지만, 다음 개념을 확인하는 데
좋습니다.

- `kv_producer`, `kv_consumer`, `kv_both` role이 실제 connector에서 어떻게
  분기되는지.
- side channel host/port가 remote peer handshake에 어떻게 사용되는지.
- lease, heartbeat, TTL이 remote KV lifetime을 어떻게 보호하는지.
- bidirectional transfer에서 D가 이전 turn의 KV를 보관하고, 다음 turn의 P가
  다시 그것을 pull하는 구조가 어떻게 표현되는지.

NIXL을 읽을 때는 `connector.py`에서 전체 wiring을 보고, `scheduler.py`,
`worker.py`, `metadata.py`, `base_worker.py`, `base_scheduler.py` 순서로
확장하세요. 단, LMCache adapter를 읽기 전에는 transport 세부 구현에 너무 오래
머물지 않는 것이 좋습니다.

## Failure and Cleanup Checklist

P/D disaggregation에서 정상 흐름만 보면 핵심을 놓치기 쉽습니다. 다음 실패
경로를 반드시 같이 표시하세요.

- Prefill은 성공했지만 decode request가 engine admission 전에 reject되는 경우.
- Decode worker가 remote KV load에 실패하는 경우.
- 일부 block만 invalid로 판정되어 local prefix cache 또는 connector cache를
  오염시킬 수 있는 경우.
- Request가 abort되어 scheduler-side state와 worker-side pending operation이
  서로 다른 시점에 정리되는 경우.
- Lease 또는 TTL이 만료되어 decode가 remote KV를 읽기 전에 producer-side block이
  사라지는 경우.

Checkpoint 질문:

- 실패가 local recompute로 복구되나요, request error로 끝나나요?
- 실패한 remote KV block이 local prefix cache에 cached로 남을 수 있나요?
- cleanup notification이 best-effort라면, connector는 어떤 timeout 또는 lease로
  orphaned state를 회수하나요?

## 권장 학습 순서

1. 기존 V1 KV cache guide의 `Allocation Flow`와 `Connector` 부분을 다시
   훑어 local block ownership을 복습합니다.
2. `examples/disaggregated/`의 proxy 예제를 읽어 request-level P/D 흐름을
   그립니다. (P/D 용어는 `disagg_prefill.md` 대신 `KVConnectorBase_V1` /
   `KVConnectorRole` 기준으로 잡습니다.)
3. `KVTransferConfig`와 serving layer에서 `kv_transfer_params`의 입출력 경계를
   확인합니다.
4. `KVConnectorBase_V1`와 scheduler integration을 읽어 external KV hit이
   allocation에 반영되는 지점을 찾습니다.
5. worker connector와 attention hook을 읽어 실제 layer-wise load/save 지점을
   찾습니다.
6. `ExampleConnector`로 최소 connector implementation을 확인합니다.
7. `LMCacheConnectorV1`와 `lmcache_integration/`을 중심으로 LMCache adapter
   boundary를 분석합니다.
8. NIXL은 노드 간 transfer, lease, heartbeat, bidirectional flow를 비교하기
   위한 reference로 읽습니다.
9. 마지막으로 unit/integration tests에서 정상, 실패, cleanup path가 어떤
   behavior로 고정되어 있는지 확인합니다.

## 최종 정리 산출물

공부가 끝나면 다음 네 가지를 직접 작성해보세요.

- P/D request sequence diagram: client, proxy, prefill instance, decode instance,
  connector, worker를 포함합니다.
- `kv_transfer_params` field inventory: 누가 만들고, 누가 읽고, 언제 폐기하는지
  표로 정리합니다.
- Scheduler-to-worker connector lifecycle table: lookup, allocation,
  metadata build, layer load, layer save, request finish를 한 줄씩 정리합니다.
- LMCache vs NIXL comparison note: LMCache adapter boundary와 NIXL transport
  boundary가 어디서 다른지 정리합니다.

이 네 가지가 완성되면, Core KV cache에서 시작한 local block mental model을
노드 간 P/D disaggregation serving의 request and connector mental model로
확장한 상태가 됩니다.
