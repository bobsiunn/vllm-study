# V1 P/D Disaggregation — Connector 통신 구조

!!! note
    이 문서는 [V1 P/D Disaggregation 학습 가이드](v1_pd_disaggregation_lmcache_study_guide_ko.md)의
    동반 문서로, "scheduler-side connector와 worker-side connector가 실제로 어떻게
    통신하는가"를 코드 경로와 함께 그림으로 정리합니다.

!!! warning "Baseline"
    upstream **`v0.23.0`** 기준. 모든 `file:line`은 이 버전의 작업 트리에서 확인됨.
    Connector 추상화는 V1 (`KVConnectorBase_V1`) 기준이며, 구
    `docs/features/disagg_prefill.md`의 Connector→LookupBuffer→Pipe(V0 모델)는
    현재 코드에 존재하지 않으므로 참조하지 않습니다.

## 1. 두 개의 역할 축 (먼저 구분할 것)

P/D를 이해할 때 가장 헷갈리는 지점은 "역할(role)"이 **서로 다른 두 축**이라는 점입니다.

```mermaid
flowchart TB
    subgraph axisA["축 A — kv_role (instance 단위, 배포 역할)"]
        direction LR
        prod["kv_producer<br/>(= prefill instance)"]
        cons["kv_consumer<br/>(= decode instance)"]
        both["kv_both"]
    end
    subgraph axisB["축 B — KVConnectorRole (connector 단위, 프로세스 역할)"]
        direction LR
        sch["SCHEDULER<br/>engine-core 프로세스"]
        wrk["WORKER<br/>GPU worker 프로세스"]
    end
    axisA -.->|"한 instance는 두 축을 동시에 가짐 — 예: producer instance의 scheduler-side connector"| axisB
```

| 축 | 정의 위치 | 값 | 의미 |
| --- | --- | --- | --- |
| **A. `kv_role`** | `vllm/config/kv_transfer.py` (`kv_producer`/`kv_consumer`/`kv_both`, `is_kv_producer()`/`is_kv_consumer()`) | instance가 KV를 **만드냐(prefill)** vs **받느냐(decode)** | 배포 역할 |
| **B. `KVConnectorRole`** | `vllm/distributed/kv_transfer/kv_connector/v1/base.py:124` (`SCHEDULER`/`WORKER`) | connector가 **scheduler 프로세스**에 있냐 vs **worker 프로세스**에 있냐 | 프로세스 역할 |

> 코드에 `prefill instance` / `decode instance`라는 심볼은 없습니다. instance의
> `kv_role`이 그 역할을 표현합니다. `scheduler connector`/`worker connector`도
> 별개 클래스가 아니라 **같은 connector를 두 `KVConnectorRole`로 띄운 것**입니다.

## 2. 큰 그림 — 통신은 두 종류

```mermaid
flowchart LR
    subgraph D["Decode instance (kv_consumer)"]
        Dsch["scheduler-side<br/>connector"]
        Dwrk["worker-side<br/>connector"]
        Dsch ---|"축 1: 엔진 배관<br/>SchedulerOutput / ModelRunnerOutput"| Dwrk
    end
    subgraph P["Prefill instance (kv_producer)"]
        Psch["scheduler-side<br/>connector"]
        Pwrk["worker-side<br/>connector"]
        Psch ---|"축 1: 엔진 배관"| Pwrk
    end
    Psch <-.->|"축 2a: kv_transfer_params — proxy 경유 HTTP"| Dsch
    Pwrk <==>|"축 2b: KV 바이트 — NIXL/NCCL/Mooncake/LMCache transport"| Dwrk
```

- **축 1 (instance 내부)**: scheduler-side ↔ worker-side. **별도 채널 없음** — vLLM
  엔진이 원래 쓰는 배관에 메타데이터 객체를 얹어 통신.
- **축 2 (instance 사이)**: P ↔ D. 엔진 배관을 **우회**.
    - 2a: scheduler 정보(`kv_transfer_params`)는 **proxy의 HTTP** request/response로.
    - 2b: 실제 KV 바이트는 **connector 자체 transport**로 worker끼리 직접.

## 3. 축 1 — instance 내부 scheduler ↔ worker

두 반쪽은 서로를 직접 호출하지 않고 **직렬화 가능한 메타데이터 객체**로만 대화합니다.

- 내려감(scheduler→worker): `KVConnectorMetadata`
- 올라옴(worker→scheduler): `KVConnectorOutput`

```mermaid
sequenceDiagram
    autonumber
    participant Sc as scheduler-side connector
    participant SO as SchedulerOutput
    participant Pipe as engine ZMQ pipe
    participant Wc as worker-side connector
    participant MO as ModelRunnerOutput

    Note over Sc: 스케줄링 단계 (engine-core 프로세스)
    Sc->>Sc: build_connector_meta() — base.py:510
    Sc->>SO: kv_connector_metadata 부착 — output.py:233
    SO->>Pipe: SchedulerOutput 전송
    Pipe->>Wc: bind_connector_metadata(meta) — base.py:211
    Note over Wc: forward 단계 (GPU worker 프로세스)
    Wc->>Wc: forward hook이 _get_connector_metadata()로 읽음 — base.py:231
    Wc->>Wc: get_finished() 등으로 KVConnectorOutput 생성 — mixin:78
    Wc->>MO: ModelRunnerOutput.kv_connector_output 부착 — mixin:48
    Wc->>Wc: clear_connector_metadata() — base.py:223
    MO->>Pipe: ModelRunnerOutput 반환
    Pipe->>Sc: scheduler가 kv_connector_output 읽음 — scheduler.py:1340
    Sc->>Sc: update_connector_output() — base.py:532
    Sc->>Sc: _update_from_kv_xfer_finished() / invalid_block_ids — scheduler.py:1597,1358
```

> worker가 여러 개(TP/PP)면 각자 `KVConnectorWorkerMetadata`를 내고
> `aggregate()`(base.py:161)로 합쳐 scheduler-side에 전달됩니다.

## 4. Worker-side forward lifecycle (KV load/save)

한 번의 forward를 감싸는 4개 hook. **launch(비동기 시작)** 과 **barrier(완료 대기)** 가 번갈아 나옵니다.

```mermaid
sequenceDiagram
    autonumber
    participant FC as forward context
    participant W as worker-side connector
    participant Attn as attention layers
    participant Buf as paged KV buffer

    FC->>W: start_load_kv() — base.py:292 (launch: remote KV → buffer 비동기 LOAD)
    loop layer마다 (maybe_transfer_kv_layer 데코레이터)
        Attn->>W: wait_for_layer_load(layer) — base.py:310 (barrier)
        W-->>Attn: 이 layer KV 준비 완료
        Attn->>Attn: attention 계산
        Attn->>W: save_kv_layer(layer, kv, meta) — base.py:324 (launch: buffer → 외부 비동기 SAVE)
    end
    FC->>W: wait_for_save() — base.py:346 (barrier: 모든 save 완료까지 block)
    Note over Buf: wait_for_save 이후에야 buffer 재사용 안전<br/>(진행 중 save가 끝나기 전 overwrite 방지)
```

- `start_load_kv` / `save_kv_layer`: 복사를 **시작만** 하고 리턴 → model 실행과 KV 전송 overlap.
- `wait_for_layer_load` / `wait_for_save`: 그 async 작업의 **완료 배리어**.
- `wait_for_save`의 스코프 = **forward 1회 내부**. 같은 forward 종료 후 다음 step이
  paged buffer를 덮어쓰기 전에 진행 중인 save를 마치게 함.

## 5. 요청 종료 시 — 비동기 block free 핸드셰이크

`wait_for_save`가 "forward 1회"라면, 이건 **요청 1건 전체**에 걸친 block 소유권 이전입니다.

```mermaid
sequenceDiagram
    autonumber
    participant Sc as scheduler (_free_request)
    participant Cn as scheduler-side connector
    participant BP as BlockPool / KVCacheManager
    participant Wc as worker-side connector

    Sc->>Cn: request_finished(request, block_ids) — base.py:542
    alt 비동기 전송 중 (True 반환)
        Cn-->>Sc: (True, kv_transfer_params)
        Note over Sc,BP: _free_blocks() 호출 안 함 → block은 pin 상태 유지 — scheduler.py:1902
        Wc->>Wc: 비동기 send/save 진행 (여러 step)
        Wc->>Sc: get_finished()가 해당 req_id 반환 — base.py:357
        Sc->>BP: 이제 _free_blocks() — scheduler.py:1907
    else 동기/connector 없음 (False 반환)
        Cn-->>Sc: (False, None)
        Sc->>BP: 즉시 _free_blocks()
    end
    Sc-->>Sc: kv_transfer_params를 요청 output으로 반환 (→ outputs.py)
```

- `request_finished`가 `True` → "전송 끝날 때까지 block 잡아둬". `get_finished`가 그
  req_id를 돌려줄 때까지 free 지연.
- 왜? prefill 측에서 finish 즉시 block을 free하면, BlockPool이 그 물리 페이지를
  재할당 → 다음 forward가 덮어씀 → 진행 중 전송이 **망가진 KV**를 보냄.
- 실패 경로: 일부 block load 실패는 `get_block_ids_with_load_errors()`(base.py:375)로
  보고 → scheduler가 `invalid_block_ids`로 처리(scheduler.py:1358).

## 6. 축 2 — instance 사이 P/D 흐름 (실제 proxy 기준)

아래는 `examples/disaggregated/disaggregated_serving/disagg_proxy_multiturn.py`의
single-turn 경로입니다. proxy가 실제로 어떻게 orchestration하는지 코드 그대로
반영합니다.

```mermaid
sequenceDiagram
    autonumber
    participant Cl as client
    participant Px as proxy
    participant P as Prefill instance (do_remote_decode=True)
    participant D as Decode instance (do_remote_prefill=True)

    Cl->>Px: 원 요청 (conversation_id 포함)
    Note over Px: kv_transfer_params 주입<br/>do_remote_decode=True, do_remote_prefill=False
    Px->>P: prefill 요청 — stream=False, max_tokens=1 (_send_to_prefill)
    Note over P: prompt prefill 1토큰만 수행, KV 생성<br/>request_finished → block을 lease TTL 동안 pin
    P-->>Px: 응답 + kv_transfer_params (P의 remote_block_ids/engine_id/request_id)
    Note over Px: P params에 remote_host 주입 후 D 요청에 부착
    Px->>D: decode 요청 — stream=True (_stream_from_decode)
    Note over D: get_num_new_matched_tokens → external KV hit → block 할당
    D->>P: worker가 connector transport로 P의 KV PULL (start_load_kv)
    Note over D: wait_for_layer_load 후 generation
    D-->>Px: SSE 스트림 (마지막 chunk에 D의 kv_transfer_params)
    Note over Px: 최종 chunk의 kv_transfer_params를<br/>conversation_id로 캐시 (다음 turn용)
    Px-->>Cl: 최종 응답
```

핵심:

- **proxy는 KV tensor를 직접 다루지 않습니다.** request/response body 안의
  `kv_transfer_params`(metadata)만 운반하고, 실제 KV 바이트는 D worker가 P worker
  로부터 connector transport로 직접 가져옵니다.
- **P 요청은 `stream=False, max_tokens=1`** — prefill만 강제하고 decode는 안 함
  (`_send_to_prefill`, py:184). **D 요청은 `stream=True`** 이고 proxy는
  **마지막 chunk**에서 `kv_transfer_params`를 뽑아 캐시합니다(`_stream_from_decode`,
  py:255~260; SSE 경로는 `_stream_from_decode_sse`, py:298~301).
- **방향 플래그**(NIXL 기준, `nixl/scheduler.py:602`):
  `do_remote_decode=True` ⇒ 그 노드는 **Prefill 노드**(`is_p_node`),
  `do_remote_prefill=True` ⇒ 그 노드는 **Decode 노드**(`is_d_node`, remote KV fetch).

### 제어 평면 vs 데이터 평면 — serving은 metadata만 운반

serving layer가 나르는 건 **오직 `kv_transfer_params: dict[str, Any]` 하나**입니다.
remote KV의 위치/식별 정보(block ids, engine id, host/port, request id, do_remote_*
플래그)를 담은 **불투명 JSON**이고, serving 단은 그 내용을 해석하지 않습니다. KV
tensor는 이 경로를 절대 통과하지 않습니다.

`kv_transfer_params`가 schema를 가로지르며 보존되는 흐름:

| 단계 | 위치 | 형태 |
| --- | --- | --- |
| Request schema (들어올 때) | `serve/disagg/protocol.py:112` `GenerateRequest.kv_transfer_params` | `dict[str, Any] \| None` |
| Engine output carrier | `vllm/outputs.py:123,143` `RequestOutput.kv_transfer_params` | 같은 dict (출처 = `request_finished` 반환값) |
| 스트리밍 병합 | `vllm/outputs.py:149` `RequestOutput.add()` | 항상 **최신 chunk 값으로 덮음** (proxy의 "마지막 chunk 캡처"와 대응) |
| Response schema (나갈 때) | `serve/disagg/protocol.py:208` + `serve/disagg/serving.py:325` | `final_res.kv_transfer_params`를 그대로 복사 |

```mermaid
flowchart LR
    req["client/proxy JSON<br/>kv_transfer_params (dict)"] --> eng["engine"]
    eng --> ro["RequestOutput<br/>.kv_transfer_params"]
    ro --> resp["response JSON<br/>kv_transfer_params (dict)"]
    eng -.->|"KV 바이트는 이 평면을 안 탐"| note["worker↔worker<br/>connector transport (축 2b)"]
```

> 핵심: **제어 평면(metadata)은 HTTP/serving**을 타고, **데이터 평면(KV 바이트)은
> connector transport**를 탑니다. serving은 데이터 평면을 전혀 모른 채 metadata
> 봉투만 pass-through합니다 — 이 분리가 P/D 아키텍처의 근간입니다.

## 7. Multi-turn bidirectional KV transfer

multiturn proxy의 진짜 포인트는 **이전 turn의 KV가 D에 남아 있다는 것**입니다.
다음 turn의 prompt = (이전 대화 + 새 입력)인데, 공유 prefix의 KV는 이미 **D**의
블록에 있습니다. 그래서 KV가 **양방향**으로 흐릅니다: 이전 turn 재사용은 D→P,
현재 prefill은 P→D.

### Proxy의 conversation cache

- `ConversationKVCache` (py:83): `conversation_id` 키로 D의 `kv_transfer_params`를
  저장. **단일 사용**(`get`은 `pop`, py:102) + **TTL 450s**.
- TTL은 의도적으로 **NIXL abort timeout(480s)보다 작게** 잡습니다(py:153~155):
  producer가 abort/lease 만료로 블록을 회수하기 전에 cache 항목이 먼저 죽도록 →
  죽은 remote block을 참조하는 일을 방지. (이게 PD 가이드 Failure 체크리스트의
  "lease/TTL 만료" 항목의 실제 구현입니다.)
- `conversation_id`가 없으면 cross-turn 재사용 비활성화 → 매 turn 재계산(py:372).

### 전체 흐름 (Step 0 ~ 끝, `_handle_request` 기준)

HIT(turn N+1) / MISS(첫 turn)와 SSE 스트리밍까지 한 장에 담았습니다.

```mermaid
sequenceDiagram
    autonumber
    participant Cl as client
    participant Px as proxy (_handle_request)
    participant Ca as ConversationKVCache
    participant P as Prefill instance
    participant D as Decode instance

    Note over Px: Step 0 — 준비
    Cl->>Px: POST /v1/chat/completions (conversation_id, stream?)
    Px->>Px: conversation_id = body.pop(), client_wants_stream 판정

    Note over Px: Step 1 — 직전 turn의 D 블록 조회
    Px->>Ca: get(conversation_id) — pop, 단일사용
    alt 캐시 HIT (turn N+1)
        Ca-->>Px: D의 kv_transfer_params
        Px->>Px: do_remote_decode=True, do_remote_prefill=False (D blocks 유지)
    else 캐시 MISS (첫 turn) 또는 conversation_id 없음
        Ca-->>Px: None
        Px->>Px: do_remote_decode=True, remote_block_ids=None
    end

    Note over Px: Step 2 — Prefill 노드 (stream=False, max_tokens=1)
    Px->>P: POST /v1/.. + kv_transfer_params
    opt HIT일 때만 (bidirectional)
        P->>D: ① P worker가 D의 이전-turn KV PULL (역방향)
    end
    Note over P: 새 토큰만 prefill, request_finished → 블록 lease pin
    P-->>Px: 200 JSON + kv_transfer_params (do_remote_prefill=True, P blocks)
    Px->>Px: p_kv_params["remote_host"] = P.host

    Note over Px: Step 3 — Decode 노드 (stream=True, SSE)
    Px->>D: POST /v1/.. (stream=True) + P blocks
    D->>P: ② D worker가 P의 KV PULL (정방향)
    loop SSE 청크 (aiter_lines)
        D-->>Px: data: {delta/text} (토큰)
        alt 스트리밍 클라이언트
            Px-->>Cl: 그 줄 그대로 relay (_stream_from_decode_sse)
        else 비스트리밍 클라이언트
            Px->>Px: collected_text 누적 (_stream_from_decode)
        end
    end
    D-->>Px: data: {finish_reason, kv_transfer_params} (마지막 청크)
    Px->>Ca: put(conversation_id, D params + remote_host=D.host)
    D-->>Px: data: [DONE]

    Note over Px: 응답 마무리
    alt 스트리밍 클라이언트
        Px-->>Cl: [DONE]까지 relay 종료
    else 비스트리밍 클라이언트
        Px-->>Cl: JSONResponse (collected_text 조립)
    end
```

①(D→P)과 ②(P→D) 두 전송이 모두 일어나는 게 **bidirectional**입니다(①은 HIT일
때만). P 노드가 prefill 노드이면서 동시에 remote 블록을 읽을 수 있는 건 connector의
`is_bidirectional_kv_xfer_enabled` 경로(`nixl/scheduler.py:449`) 덕분입니다.
proxy↔D는 평범한 OpenAI **SSE 스트리밍**이고, proxy는 항상 D에 `stream=True`로
요청해 토큰은 클라이언트로(relay 또는 누적), `kv_transfer_params`는 캐시로
라우팅합니다.

### `kv_transfer_params` 필드 인벤토리

proxy ↔ P/D 사이를 오가는 메타데이터 필드 (proxy + `nixl/scheduler.py:664~669` 기준):

| 필드 | 누가 만드나 | 누가 읽나 | 의미 |
| --- | --- | --- | --- |
| `do_remote_decode` | proxy(P 요청), connector output | 받는 instance | True면 그 노드 = Prefill(`is_p_node`) |
| `do_remote_prefill` | proxy(D 요청), connector output | 받는 instance | True면 그 노드 = Decode(remote KV fetch) |
| `remote_block_ids` | producer `request_finished` | 읽는 worker | 읽어갈 remote 블록 ID (그룹별 list) |
| `remote_engine_id` | producer `request_finished` | 읽는 worker | 어느 엔진의 블록인지 |
| `remote_request_id` | producer `request_finished` | proxy/읽는 worker | 원격 요청 식별자 |
| `remote_host` | **proxy가 주입** (py:257,300,424) | 읽는 worker | 응답한 instance의 host |
| `remote_port` | producer/config | 읽는 worker | side channel 포트 |

> `remote_host`는 connector가 아니라 **proxy가** 응답 instance 주소로 채워 넣는다는
> 점이 포인트 — connector는 자기 host를 모르고, 노드 간 라우팅은 proxy 책임.

## 8. Admission 전 reject 시 remote-prefill block cleanup

D측 요청이 **엔진에 들어가기 전에 거부**(검증 실패/과부하 등)되면, P 노드는 이미
그 `request_id`용 prefill 블록을 lease로 pin해 둔 상태입니다. 아무도 안 알려주면
lease/TTL 만료까지 **블록이 stranded**됩니다. vLLM은 전용 cleanup API를 만들지
않고, **가짜 pre-aborted 요청을 엔진에 밀어넣어 평소의 `request_finished` 훅을
재사용**해 이 블록을 조기 회수합니다.

```mermaid
sequenceDiagram
    autonumber
    participant Sv as serving (_with_kv_transfer_rejection_cleanup)
    participant AL as AsyncLLM
    participant EC as EngineCore
    participant Sc as Scheduler
    participant Cn as NIXL connector (sched-side)
    participant Wk as worker-side connector
    participant P as P node

    Note over Sv: create_* 코루틴이 ErrorResponse/예외 반환<br/>= admission 전 reject (do_remote_prefill=True일 때만)
    Sv->>AL: notify_kv_transfer_request_rejected(req_id, kv_params) — serving.py:435
    Note over AL: 합성 EngineCoreRequest 생성<br/>abort_immediately=True, kv_params in extra_args — async_llm.py:733
    AL->>EC: add_request_async(request)
    EC->>Sc: add_request(request)
    EC->>Sc: abort_requests([req_id]) → finish_requests(FINISHED_ABORTED) — core.py:373,384
    Sc->>Cn: request_finished(request, block_ids) — base.py:542
    Note over Cn: do_remote_prefill 아직 True<br/>(update_state_after_alloc 미호출) = 스케줄 전 abort
    Cn->>Wk: _reqs_need_recv[req_id] = (request, []) — nixl/scheduler.py:618
    Wk->>P: req_id 블록 풀어라 통지
    Note over P: pin됐던 prefill 블록 회수 (lease 만료 전 조기 free)
```

읽는 순서:

1. **감지** — `entrypoints/openai/engine/serving.py:413` `_with_kv_transfer_rejection_cleanup`.
   `create_*` 코루틴을 감싸고(호출처: `chat_completion/serving.py:231`,
   `completion/serving.py:124`, `responses/serving.py:327`), `has_kv_connector` +
   `kv_transfer_params.do_remote_prefill`일 때만 동작(serving.py:422~423). awaitable이
   `ErrorResponse`/예외면 `finally`에서 notify(435).
2. **브리지** — `v1/engine/async_llm.py:723` `notify_kv_transfer_request_rejected`.
   거부된 req를 **`abort_immediately=True`** 합성 `EngineCoreRequest`로 만들어 제출(746,748).
3. **엔진 코어** — `v1/engine/core.py:373~376`. `add_request` 직후 `abort_immediately`면
   `abort_requests` → `finish_requests(FINISHED_ABORTED)`(384).
4. **scheduler→connector** — `request_finished`(base.py:542) 정상 경로 진입.
5. **cleanup** — `nixl/scheduler.py:609~620`. 스케줄 전 abort라 `do_remote_prefill`이
   아직 True → 빈 block_ids를 `_reqs_need_recv`에 넣어 worker가 P에 free 통지.

> 핵심 설계: **전용 cleanup RPC 없음.** "스케줄 전 abort된 요청"을 `request_finished`
> 훅이 `do_remote_prefill==True`로 식별해 정상 종료 경로에 흡수시킨다. 이게 §5의 free
> 핸드셰이크가 실패 경로에서 재사용되는 모습이고, §7의 lease/TTL(proxy 450s < abort
> 480s)은 이 통지가 유실됐을 때의 안전망이다.

## 9. Connector를 켜면 바뀌는 것 (제한·자동조정)

`--kv-transfer-config`로 connector가 켜지면 `VllmConfig` post-init에서 다른 기능이
**에러로 막히거나(restriction)** **조용히 조정(auto-adjust)** 됩니다. `arg_utils.py`는
`--kv-transfer-config` JSON을 `KVTransferConfig`로 파싱해 전달만 하고(`arg_utils.py:
1474, 2245`), 실제 게이팅은 전부 `config/vllm.py`에 있습니다.

### 자동조정 (조용히 바뀜)

| 무엇 | 트리거 | 동작 | 위치 |
| --- | --- | --- | --- |
| **connector 자동 선택** | `cache_config.kv_offloading_size` 설정 | kv_transfer_config 없으면 생성 후 backend별로 `OffloadingConnector`/`SimpleCPUOffloadConnector`/`LMCacheMPConnector` 지정 + `kv_role="kv_both"` | `vllm.py:774~802` (`_post_init_kv_transfer_config`) |
| **CUDA graph 강등** | connector가 `requires_piecewise_for_cudagraph` + full graph 모드 | `cudagraph_mode` → **PIECEWISE** (layerwise async op은 graph capture 불가), `warning_once` | `vllm.py:1244~1269` |
| **Hybrid KV cache(HMA) 비활성화** | connector가 `SupportsHMA` 미구현 | `disable_hybrid_kv_cache_manager=True`. 영향: hybrid SSM(Jamba/Bamba)은 HMA 필수라 **startup 실패**, sliding-window attention은 **성능 저하** | `vllm.py:1479~1500` |

### 제한 (에러로 막음)

| 무엇 | 조건 | 이유 | 위치 |
| --- | --- | --- | --- |
| `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` | KV connector + cumem allocator 꺼짐 | CUDA VMM이 KV 가상주소를 다른 물리 페이지로 remap → pinned/registered 메모리(NIXL/Mooncake의 IB region) 무효화 → 첫 노드 간 전송에서 RDMA 실패 | `vllm.py:804~845` (`_verify_kv_transfer_compat`) |
| `--enable-return-routed-experts` | `is_kv_transfer_instance` | PD: P에 캡처된 routing이 D에 도달 못함 / offload: KV가 GPU 밖이면 slot_mapping 의미가 바뀌어 slot-indexed buffer 깨짐 | `vllm.py:874~886` |
| chunked prefill | `ExampleHiddenStatesConnector` + chunked prefill | 이 connector가 chunked prefill 미지원 | `vllm.py:763~772` |

> 정리: connector를 켜면 (1) offload 경로는 connector를 **자동 주입**하고, (2) CUDA
> graph는 **PIECEWISE로 내려가며**, (3) connector가 HMA 미지원이면 hybrid KV cache가
> **꺼지고**, (4) `expandable_segments`·routed-experts·일부 chunked-prefill 조합은
> **아예 거부**됩니다. 핵심 동인은 connector가 KV 메모리를 pin/register하고
> layer 단위 async 전송을 한다는 점입니다.

## 10. Code Map (v0.23.0)

| 심볼 / 필드 | 위치 | 역할 |
| --- | --- | --- |
| `KVConnectorRole` (SCHEDULER/WORKER) | `kv_connector/v1/base.py:124` | connector 프로세스 역할 |
| `kv_role`, `is_kv_producer/consumer` | `config/kv_transfer.py` | instance 배포 역할 (producer/consumer) |
| `build_connector_meta` | `base.py:510` | (sched) worker로 보낼 `KVConnectorMetadata` 생성 |
| `kv_connector_metadata` | `v1/core/sched/output.py:233` | SchedulerOutput에 실리는 다운링크 필드 |
| `bind/clear/_get_connector_metadata` | `base.py:211/223/231` | (worker) 메타데이터 설치·소비·정리 |
| `start_load_kv` | `base.py:292` | (worker) remote KV 비동기 load 시작 |
| `wait_for_layer_load` | `base.py:310` | (worker) layer KV load 완료 배리어 |
| `save_kv_layer` | `base.py:324` | (worker) layer KV 비동기 save 시작 |
| `wait_for_save` | `base.py:346` | (worker) forward 종료 시 save 완료 배리어 |
| `get_finished` | `base.py:357` | (worker→sched) 전송 완료 req_id 통지 |
| `KVConnectorOutput` | `v1/worker/kv_connector_model_runner_mixin.py:78` | 업링크 객체 (finished/invalid/stats) |
| `update_connector_output` | `base.py:532` | (sched) worker 결과 반영 |
| `get_num_new_matched_tokens` | `base.py:454` | (sched) external KV hit 토큰 수 계산 |
| `update_state_after_alloc` | `base.py:489` | (sched) 할당된 block ↔ remote KV 연결 |
| `request_finished` | `base.py:542` | (sched) 종료 시 block free 지연 + `kv_transfer_params` |
| `get_block_ids_with_load_errors` | `base.py:375` | (worker) load 실패 block 보고 |
| `maybe_transfer_kv_layer` (데코레이터) | `model_executor/layers/attention/kv_transfer_utils.py:15` | attention forward를 감싸 layer hook 호출 |
| `_post_init_kv_transfer_config` | `config/vllm.py:757` | offload→connector 자동 주입, kv_role=kv_both |
| `_verify_kv_transfer_compat` | `config/vllm.py:804` | expandable_segments 비호환 거부 |
| HMA 자동 비활성화 | `config/vllm.py:1479~1500` | connector가 `SupportsHMA` 미구현 시 |
| CUDA graph → PIECEWISE | `config/vllm.py:1244~1269` | `requires_piecewise_for_cudagraph` 시 |
| `_with_kv_transfer_rejection_cleanup` | `openai/engine/serving.py:413` | admission-전 reject 감지→notify |
| `notify_kv_transfer_request_rejected` | `v1/engine/async_llm.py:723` | `abort_immediately` 합성 요청 제출 |
| `abort_immediately` 처리 | `v1/engine/core.py:373` | add 직후 abort→`request_finished` 유도 |
| reject cleanup 분기 | `nixl/scheduler.py:609~620` | `do_remote_prefill` still-True → P 블록 free |
| `do_remote_decode` / `do_remote_prefill` 해석 | `nixl/scheduler.py:602` | `is_p_node`/`is_d_node` 방향 결정 |
| `request_finished` 반환 params | `nixl/scheduler.py:664` | `remote_block_ids/engine_id/request_id` 생성 |
| block lease/TTL pin | `nixl/scheduler.py:642~655` | `_reqs_need_send` + lease로 블록 회수 지연 |
| `ConversationKVCache` | `examples/.../disagg_proxy_multiturn.py:83` | proxy의 conv별 D-params 캐시 (단일사용+TTL) |
| `_send_to_prefill` / `_stream_from_decode` | `disagg_proxy_multiturn.py:178/204` | P=non-stream max_tokens=1, D=stream+params 캡처 |

## 11. 한 줄 요약

> **instance 내부** scheduler↔worker = 엔진의 `SchedulerOutput`/`ModelRunnerOutput`
> 배관에 `KVConnectorMetadata`(↓)·`KVConnectorOutput`(↑)을 실어 통신.
> **instance 사이**는 별개 — scheduler끼리는 proxy의 `kv_transfer_params`(HTTP),
> worker끼리는 connector transport로 KV 바이트를 직접 옮긴다.
> **multi-turn**에서는 이전 turn의 D 블록을 다음 turn의 P가 다시 읽어(D→P) KV가
> **양방향**으로 흐르고, proxy의 `ConversationKVCache`(단일사용+TTL<abort)가 그
> 연결고리다.
