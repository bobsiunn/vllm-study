# V1 KV Cache와 LMCache Connector 학습 가이드

!!! note
    이 문서는 구현 설계서가 아니라 학습 로드맵입니다. NPU 전용 LMCache
    작업을 설계하기 전에 현재 V1 KV cache, worker block table, LMCache
    connector 경계를 이해하는 데 사용하세요.

## 목표

이 가이드는 다음 세 가지 연결된 흐름에 대한 코드 레벨 mental model을
만드는 데 도움을 줍니다.

1. V1이 KV block을 할당, 캐시, 해제, invalidation하는 방식.
2. scheduler가 소유한 block ID가 worker block table과 slot mapping으로
   변환되는 방식.
3. LMCache가 V1 KV connector API를 통해 scheduler와 worker에 연결되는
   방식.

이 가이드를 마치면 prefix-cache lookup부터 block allocation, worker
execution, external KV load/save, scheduler-side completion handling까지 한
요청을 따라갈 수 있어야 합니다.

## 선수 지식

다음 개념이 익숙하지 않다면 먼저 읽어보세요.

- [PagedAttention](../design/paged_attention.md): OS paging analogy와 vLLM이
  KV memory를 fixed-size block으로 나누는 이유.
- [Automatic Prefix Caching](../design/prefix_caching.md): block hash,
  full-block caching, prefix reuse.
- [Disaggregated Prefilling](../features/disagg_prefill.md): connector,
  lookup buffer, pipe, scheduler connector, worker connector 용어.

## Source Map

Core KV cache 파일:

- `vllm/v1/core/kv_cache_manager.py`: scheduler-facing facade. `KVCacheBlocks`,
  `get_computed_blocks()`, `allocate_slots()`를 포함합니다.
- `vllm/v1/core/kv_cache_coordinator.py`: KV cache group 전체로 fan-out합니다.
- `vllm/v1/core/single_type_kv_cache_manager.py`: full, sliding-window,
  chunked-local, Mamba, cross-attention별 block 계산 규칙.
- `vllm/v1/core/block_pool.py`: physical block ownership, prefix-cache hash
  table, free queue, touch, free, evict, KV event.
- `vllm/v1/core/kv_cache_utils.py`: `KVCacheBlock`, block hash,
  free-block queue.

Scheduler와 worker handoff 파일:

- `vllm/v1/core/sched/scheduler.py`: admission, connector lookup, allocation,
  metadata construction, invalid-block handling, request completion.
- `vllm/v1/core/sched/output.py`: worker로 전달되는 `NewRequestData`,
  `CachedRequestData`, `SchedulerOutput` field.
- `vllm/v1/worker/gpu_input_batch.py`: worker request state와 block-table row
  ownership.
- `vllm/v1/worker/block_table.py`: block ID row와 slot mapping.
- `vllm/v1/worker/gpu_model_runner.py`: block-table update, slot mapping,
  attention metadata, block zeroing, connector output wrapping.

Connector와 LMCache 파일:

- `vllm/distributed/kv_transfer/kv_connector/v1/base.py`: scheduler-side와
  worker-side implementation을 위한 V1 connector API.
- `vllm/v1/worker/kv_connector_model_runner_mixin.py`: forward execution을
  감싸는 model-runner connector lifecycle.
- `vllm/v1/outputs.py`: worker가 반환하는 `KVConnectorOutput`.
- `vllm/distributed/kv_transfer/kv_connector/v1/lmcache_connector.py`:
  `LMCacheConnectorV1` wrapper.
- `vllm/distributed/kv_transfer/kv_connector/v1/lmcache_integration/`: native
  vLLM adapter, multi-process adapter, LMCache utility bridge.

## Core Mental Model

먼저 ownership boundary부터 잡으세요.

- `KVCacheManager`는 scheduler-facing facade입니다. scheduler는 computed
  block 조회, allocation, zeroing할 block ID, prefix-cache reset, free를 이
  객체에 요청해야 합니다.
- `KVCacheCoordinator`는 모델이 하나의 KV cache group을 갖는지, 서로 다른
  block rule을 가진 여러 group을 갖는지를 숨깁니다.
- `SingleTypeKVCacheManager`는 per-group request-to-block list를 소유하고,
  하나의 attention type에 대해 요청이 몇 개의 block을 필요로 하는지
  답합니다.
- `BlockPool`은 block ID `0`인 `null_block`을 포함해 모든 physical
  `KVCacheBlock`을 소유합니다.

Container shape를 명확히 기억하세요. `KVCacheBlocks.blocks`는 KV cache
group 기준으로 묶입니다. 따라서 `blocks[i][j]`는 `i`번째 KV cache group의
`j`번째 block을 의미합니다. token-major 구조로 읽으면 안 됩니다.

Checkpoint 질문:

- 어떤 객체가 physical block lifetime을 소유하나요?
- 어떤 객체가 요청에 필요한 block 개수를 결정하나요?
- scheduler가 flat list 대신 `tuple[list[int], ...]`를 받는 이유는
  무엇인가요?

## Allocation Flow

`KVCacheManager.allocate_slots()`를 천천히 읽으세요. 이 함수의 docstring은
scheduling 시점의 layout을 이해하는 가장 좋은 local map입니다.

```text
<computed> <new local computed> <external computed> <new tokens> <lookahead>
```

주요 단계는 다음과 같습니다.

1. sliding-window attention처럼 early token이 더 이상 필요 없는 attention
   type에 대해 skipped block을 해제합니다.
2. local prefix hit, external connector hit, new token, speculative lookahead
   token에 필요한 block 수를 계산합니다.
3. external computed token과 아직 model computation이 필요한 token을 위한
   block을 할당합니다.
4. caching이 꺼져 있거나 async KV-transfer path 때문에 `delay_cache_blocks`가
   설정된 경우가 아니라면 full block을 즉시 cache합니다.

특히 다음 인자를 집중해서 보세요.

- `num_new_computed_tokens`: vLLM prefix caching으로 이미 cache된 token.
- `num_external_computed_tokens`: LMCache처럼 vLLM 밖에 KV가 존재하는 token.
- `delay_cache_blocks`: future transfer가 완료되기 전에 vLLM이 block을
  cached로 표시하지 못하게 합니다.
- `num_lookahead_tokens`: speculative decoding을 위한 extra slot.

Checkpoint 질문:

- external KV를 load해야 하는 경우 `num_new_tokens == 0`이어도 allocation이
  가능한 이유는 무엇인가요?
- attention window 안에 있는 connector hit에는 어떤 block이 할당되나요?
- 새로 할당된 full-attention block ID는 언제 worker zeroing 대상으로
  기록되나요?

## BlockPool과 Prefix Cache

`BlockPool`은 physical block owner입니다. 다음 operation을 순서대로
공부하세요.

1. `get_new_blocks()`: free queue에서 block을 가져오며, 필요하면 오래된
   cached hash를 evict합니다.
2. `touch()`: 다른 request가 hit한 cached block의 reference를 증가시킵니다.
3. `free_blocks()`: reference를 감소시키고 reference가 0인 block을 free queue
   뒤에 붙입니다.
4. `cache_full_blocks()`: block hash를 할당하고 full block을 prefix-cache
   hash table에 삽입합니다.
5. `evict_blocks()`: connector가 invalid라고 보고한 block의 cached hash를
   제거합니다. 이때 in-use block을 반드시 free하는 것은 아닙니다.

`null_block`은 특별합니다. skipped slot이나 out-of-window slot을 표현하며,
일반 free-list block처럼 취급하면 안 됩니다.

Checkpoint 질문:

- cached block이 cache table에는 남아 있으면서 동시에 evictable이 되는
  경우는 언제인가요?
- request block을 reverse order로 free하는 이유는 무엇인가요?
- worker가 scheduler가 할당하지 않은 block ID를 connector를 통해 보고하지
  못하도록 보호하는 invariant는 무엇인가요?

## Scheduler Connector Path

scheduler는 worker가 무엇을 실행할지 결정하기 전에 local prefix caching과
external KV-cache lookup을 함께 고려합니다.

`vllm/v1/core/sched/scheduler.py`에서 다음 순서를 따라가세요.

1. scheduler는 `KVCacheManager`를 만들고, `kv_transfer_config`가 설정된 경우
   scheduler-side connector도 만듭니다.
2. waiting request에 대해 local prefix-cache lookup 이후
   `get_num_new_matched_tokens()`로 connector의 external matched token 수를
   묻습니다.
3. `num_external_computed_tokens`와 필요 시 `delay_cache_blocks`를 넣어
   `KVCacheManager.allocate_slots()`를 호출합니다.
4. connector가 request와 allocated block state를 기억할 수 있도록
   `connector.update_state_after_alloc()`를 호출합니다.
5. `connector.build_connector_meta()`로 `SchedulerOutput.kv_connector_metadata`를
   만듭니다.

Checkpoint 질문:

- external cache hit는 어디에서 model computation을 줄이나요?
- connector state는 왜 allocation 전이 아니라 allocation 후에 update되어야
  하나요?
- 어떤 scheduler output field가 connector metadata를 모든 worker에
  전달하나요?

## SchedulerOutput Handoff

`SchedulerOutput`은 scheduler에서 worker로 전달되는 핵심 handoff
object입니다. KV 학습에서는 다음 field에 집중하세요.

- `scheduled_new_reqs`: `NewRequestData` object list. 각 new request는
  `block_ids`와 `num_computed_tokens`를 가집니다.
- `scheduled_cached_reqs`: worker-side에서 이미 알고 있는 request를 위한
  `CachedRequestData`. `new_block_ids`, `num_computed_tokens`, token diff를
  전달합니다.
- `kv_connector_metadata`: worker connector를 위한 scheduler-side connector
  instruction.
- `new_block_ids_to_zero`: 사용 전에 GPU memory를 zeroing해야 하는 block ID.

Checkpoint 질문:

- 어떤 field가 stable request state이고 어떤 field가 per-step diff인가요?
- resumed request는 block ID를 단순 append하는 request와 어떻게 다른가요?
- zeroing 정보가 `BlockPool` 내부가 아니라 scheduler output에 있는 이유는
  무엇인가요?

## Worker BlockTable Path

worker는 scheduler block ID를 attention backend가 소비하는 tensor로
변환합니다.

다음 순서로 읽으세요.

1. `gpu_input_batch.py`가 `MultiGroupBlockTable`을 만들고 request row를
   소유합니다.
2. `BlockTable.add_row()`가 new request의 row를 초기화합니다.
3. `BlockTable.append_row()`가 existing request에 new block ID를 append합니다.
4. `BlockTable.commit_block_table()`이 CPU block-table row를 GPU로 복사합니다.
5. `BlockTable.compute_slot_mapping()`이 token position을 KV cache slot으로
   mapping합니다.

Hybrid case에서는 allocation block size가 kernel block size보다 클 때
`BlockTable.map_to_kernel_blocks()`가 하나의 KV-manager block ID를 여러
attention-kernel block ID로 쪼갤 수 있습니다.

Checkpoint 질문:

- `InputBatch`의 request에 대응하는 `BlockTable` row는 무엇인가요?
- block ID는 언제까지 CPU-side bookkeeping이고, 언제 GPU-visible이 되나요?
- `slot_mapping`은 token position과 physical KV slot을 어떻게 연결하나요?

## GPUModelRunner Commit Path

`GPUModelRunner`에서 scheduler output이 execution metadata로 바뀌는 흐름을
따라가세요.

- New/cached request state가 `InputBatch`에 적용됩니다.
- request가 resumed되었는지에 따라 new block ID가 append되거나 replace됩니다.
- `new_block_ids_to_zero`는 해당 block이 사용되기 전에 zeroing됩니다.
- attention metadata가 만들어지기 전에 block table이 commit되고 slot mapping이
  계산됩니다.

Connector-aware execution을 볼 때는 block table과 connector lifecycle을 같은
mental trace 안에 두세요. worker는 scheduler가 이미 할당한 block 안으로
external KV를 load하고 있을 수 있습니다.

Checkpoint 질문:

- new block ID는 worker state에 처음 어디에서 들어오나요?
- 어떤 operation이 block-table row를 GPU attention kernel에 보이게 하나요?
- block zeroing은 왜 단순 cleanup이 아니라 correctness의 일부인가요?

## KV Connector V1 Boundary

`KVConnectorBase_V1`은 split API를 정의합니다.

Scheduler-side method는 다음 질문에 답합니다.

- 이 request와 match되는 external token은 몇 개인가요?
- 이번 step에 worker로 어떤 connector metadata를 보내야 하나요?
- worker-side connector가 completion, failure, event를 보고했나요?
- request block을 지금 free해야 하나요, 아니면 async connector가 아직
  사용 중인가요?

Worker-side method는 실제 KV movement를 수행합니다.

- execution 전에 scheduler metadata를 bind합니다.
- forward pass 전에 KV load를 시작합니다.
- attention layer가 data를 필요로 할 때 per-layer load를 기다립니다.
- execution 중 per-layer KV를 save합니다.
- save를 기다린 뒤 finished send, finished receive, stats, event,
  worker metadata, invalid block ID를 보고합니다.

Checkpoint 질문:

- 어떤 connector call이 scheduler-only인가요?
- 어떤 connector call이 model forward 전후에 반드시 일어나야 하나요?
- 한 step이 끝난 뒤 worker에서 scheduler로 어떤 data가 흐르나요?

## Model Runner Connector Lifecycle

`KVConnectorModelRunnerMixin`은 worker-side lifecycle을 감쌉니다.

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

model forward 중에는 attention-layer code가 `wait_for_layer_load()`나
`save_kv_layer()` 같은 connector hook을 호출할 수 있습니다. mixin은 그런
per-layer work 바깥에서 bind, start, finish, output, clear lifecycle을
소유합니다.

결과로 만들어지는 `KVConnectorOutput`은 다음을 포함할 수 있습니다.

- `finished_sending`과 `finished_recving` request ID.
- connector stats, KV cache event, worker metadata.
- external load가 실패한 block을 나타내는 `invalid_block_ids`.
- handshake aggregation이 필요한 connector를 위한 `expected_finished_count`.

Checkpoint 질문:

- 매 worker execution 후 어떤 state를 clear해야 하나요?
- `finished_sending`과 `finished_recving`이 분리된 이유는 무엇인가요?
- invalid block ID는 어떻게 scheduler-side recompute decision으로 이어지나요?

## LMCacheConnectorV1 Path

`LMCacheConnectorV1`은 vLLM connector wrapper입니다. native vLLM adapter 또는
최신 LMCache adapter 중 하나를 선택한 뒤 V1 connector API를 그 adapter로
delegate합니다.

중요한 delegated method는 다음과 같습니다.

- Worker path: `start_load_kv()`, `wait_for_layer_load()`, `save_kv_layer()`,
  `wait_for_save()`, `get_finished()`, load-error reporting.
- Scheduler path: `get_num_new_matched_tokens()`, `update_state_after_alloc()`,
  `build_connector_meta()`, `update_connector_output()`, `request_finished()`,
  `take_events()`.
- Event path: LMCache event는 vLLM `BlockStored` event로 변환되고 worker 전체에
  걸쳐 aggregate됩니다.

Checkpoint 질문:

- vLLM의 책임은 어디에서 끝나고 LMCache-specific request tracking은 어디에서
  시작되나요?
- 어떤 LMCache call이 vLLM이 할당한 block ID를 필요로 하나요?
- adapter의 어떤 부분이 lookup, load, save, completion tracking인가요?

## RequestTracker

`lmcache_integration/vllm_v1_adapter.py`에서 `RequestTracker`는 vLLM request
state와 LMCache request state를 잇는 핵심 bridge입니다. 이 객체는 다음을
추적합니다.

- request ID, prompt length, token ID, allocated block ID, saved-token count.
- disaggregated-prefill metadata, multimodal hash, request config, decode phase,
  `skip_save`.

향후 NPU-side LMCache 작업에서는 hardware movement를 바꾸기 전에 이 항목들을
semantic requirement로 이해해야 합니다. 실제 data mover는 달라질 수 있지만
request identity, token coverage, allocated block ID, save/load state는 vLLM
scheduler decision과 일관성을 유지해야 합니다.

Checkpoint 질문:

- lookup key에는 어떤 tracker field가 필요한가요?
- external KV를 vLLM-allocated block에 mapping하려면 어떤 field가 필요한가요?
- 잘못된 token range를 save하거나 load하지 않으려면 어떤 field가 필요한가요?

## Invalid Block Handling

`KVConnectorOutput.invalid_block_ids`와 이를 처리하는 scheduler method를
공부하세요. 이 흐름에는 policy split이 있습니다.

1. worker connector가 external load에 실패한 block ID를 보고합니다.
2. scheduler는 async waiting request를 scan하지만 cache eviction candidate는
   모으지 않습니다. 해당 block은 아직 cached 상태가 아니기 때문입니다.
3. scheduler는 running sync-load request를 scan하고 invalid block 및 downstream
   block을 prefix-cache eviction 대상으로 모을 수 있습니다.
4. affected request는 longest valid computed prefix로 truncate되고,
   external-computed-token state가 줄어듭니다.
5. recompute policy에서는 async failure가 retry 대상으로 표시되고 sync affected
   request는 recompute를 위해 reschedule됩니다. fail policy에서는 affected
   request가 failed 처리되고 sync cached block이 evict됩니다.

Checkpoint 질문:

- async load failure는 왜 prefix cache에서 evict되지 않나요?
- scheduler는 affected request를 어떻게 결정하나요?
- failure policy는 recompute와 fail behavior를 어떻게 바꾸나요?
- worker가 보고한 block ID에 대해 어떤 조건이 항상 참이어야 하나요?

## End-to-End Exercises

mental model이 완성되었는지 확인하려면 다음 exercise를 해보세요.

### Exercise 1: Local prefix-cache prefill

local prefix-cache hit는 있지만 connector hit는 없는 request를 trace하세요.

1. `get_computed_blocks()`가 full cached block을 찾습니다.
2. `allocate_slots()`가 해당 block을 touch하고 suffix를 위한 slot을
   할당합니다.
3. `SchedulerOutput`이 block ID를 worker로 보냅니다.
4. `BlockTable`이 row를 commit하고 slot mapping을 계산합니다.

### Exercise 2: LMCache external-hit prefill

external LMCache hit가 있는 request를 trace하세요.

1. scheduler가 local computed token과 external matched token을 얻습니다.
2. `allocate_slots()`가 external computed token을 위한 block을 할당합니다.
3. connector metadata가 worker에게 무엇을 load해야 하는지 알려줍니다.
4. worker connector가 attention이 필요로 하기 전에 allocated block으로 KV를
   load합니다.
5. worker가 `KVConnectorOutput`을 반환해 scheduler-side connector state를
   update합니다.

### Exercise 3: Invalid external blocks

external load failure를 trace하세요.

1. worker가 `invalid_block_ids`를 보고합니다.
2. scheduler가 affected block hash를 evict합니다. sync cached block의 경우
   policy에 따라 eviction 대상이 됩니다.
3. affected request는 external-computed-token credit을 잃습니다.
4. policy에 따라 request가 recompute를 위해 reschedule되거나 fail 처리됩니다.

## 읽고 실행할 테스트

system `python3`가 아니라 `.venv/bin/python`을 사용하세요.

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

code를 변경하기 전에 test를 먼저 읽으세요. test는 전체 scheduler와 model
runner file보다 의도된 behavior를 더 compact하게 보여줍니다.

## NPU LMCache 학습 메모

hardware transfer code에서 시작하지 마세요. invariant에서 시작하세요.

- vLLM scheduler는 block allocation decision과 block ID validity를 소유합니다.
- Worker connector는 scheduler state가 이미 이해하고 있는 block 안팎으로 KV를
  이동합니다.
- Connector metadata는 serializable해야 하며, worker가 수행해야 하는 정확한
  per-step work를 설명해야 합니다.
- Slot mapping과 block table은 attention-kernel contract입니다. 어떤 NPU path든
  동등한 token-to-KV-slot semantics를 보존해야 합니다.
- Request tracking은 token coverage, allocated block ID, saved range, load
  range, multimodal identity, completion state를 보존해야 합니다.

transfer granularity, layout conversion, async completion, failure reporting 같은
open NPU question은 별도의 notes file에 기록하세요. V1 invariant가 명확해지기
전까지는 이런 open question을 current-code reading path에 섞지 마세요.

## Quick Self-Check

NPU design으로 넘어가기 전에 code를 보지 않고 다음 질문에 답할 수 있는지
확인하세요.

1. locally computed token, local prefix-cache hit, external connector hit의
   차이는 무엇인가요?
2. `KVCacheBlocks`가 여러 block list를 담을 수 있는 이유는 무엇인가요?
3. `BlockPool`은 cached block을 reuse, touch, evict할 수 있는지 어떻게
   결정하나요?
4. 어떤 scheduler output field가 block ID와 connector work를 worker에게
   알려주나요?
5. `BlockTable`은 block ID와 token position을 slot mapping으로 어떻게
   변환하나요?
6. `KVConnectorModelRunnerMixin`은 model forward 전후에 무엇을 하나요?
7. LMCache는 request에 할당된 block ID를 어떻게 알게 되나요?
8. connector가 block load 실패를 보고하면 무엇이 일어나야 하나요?
