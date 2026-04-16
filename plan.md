# vLLM PagedAttention 코드 분석 스터디 플랜

## Context

Linux OS 메모리 시스템 전문가가 vLLM의 PagedAttention을 코드 레벨에서 분석하여, OS 가상 메모리와 비교하는 세미나를 준비하기 위한 가이드. vLLM의 V1 엔진 기준으로 분석하며, 각 단계에서 OS 대응 개념을 매핑한다.

---

## 핵심 개념 매핑 (OS ↔ vLLM)

| OS 가상 메모리 | vLLM PagedAttention | 대응 코드 위치 |
|---|---|---|
| Page (4KB 고정 크기) | KVCacheBlock (16 tokens 고정) | `v1/core/kv_cache_utils.py:110` |
| Page Frame (물리 메모리) | GPU 메모리의 블록 슬롯 | `v1/worker/block_table.py` |
| Page Table | BlockTable (req → block_id 매핑) | `v1/worker/block_table.py:18` |
| Page Table Entry | block_id (물리 블록 번호) | `v1/core/kv_cache_utils.py:113` |
| Free Page List | FreeKVCacheBlockQueue (이중 연결 리스트) | `v1/core/kv_cache_utils.py:158` |
| Page Frame Allocator | BlockPool | `v1/core/block_pool.py:130` |
| Virtual Address → Physical Address | slot_mapping (논리 위치 → GPU 메모리 슬롯) | `v1/worker/block_table.py` Triton 커널 |
| Page Replacement (LRU) | 블록 eviction (LRU) | `v1/core/block_pool.py` `evict_blocks()` |
| Shared Pages / COW | Prefix Caching (해시 기반 블록 공유) | `v1/core/block_pool.py:34` `BlockHashToBlockMap` |
| Reference Count | `ref_cnt` in KVCacheBlock | `v1/core/kv_cache_utils.py:116` |
| Page Fault | Cache miss → recompute | scheduler 내 prefix miss 처리 |

---

## 전체 코드 구조도

```
요청 수신부터 attention 연산까지의 블록 관리 흐름:

┌─────────────────────────────────────────────────────────────┐
│  API Server (vllm/entrypoints/api_server.py)                │
│  └→ AsyncLLM (vllm/v1/engine/async_llm.py)                 │
│     └→ EngineCoreClient ──[ZMQ]──→ EngineCore              │
└─────────────────────────────────────────────────────────────┘
                                          │
                    ┌─────────────────────┴──────────────────────┐
                    │  EngineCore (vllm/v1/engine/core.py)       │
                    │  매 step마다: schedule() → execute()       │
                    └─────────────┬──────────────────────────────┘
                                  │
           ┌──────────────────────┴──────────────────────┐
           ▼                                             ▼
┌─────────────────────┐                    ┌──────────────────────┐
│  Scheduler          │                    │  Executor → Worker   │
│  (v1/core/sched/    │                    │  → ModelRunner       │
│   scheduler.py)     │                    │  (v1/worker/         │
│                     │                    │   gpu_model_runner.py)│
│  ┌───────────────┐  │                    │                      │
│  │KVCacheManager │  │   SchedulerOutput  │  ┌────────────────┐  │
│  │ ┌───────────┐ │  │ ──(block_ids)───→  │  │ BlockTable     │  │
│  │ │ BlockPool │ │  │                    │  │ (worker/       │  │
│  │ │  ┌──────┐ │ │  │                    │  │  block_table.py)│ │
│  │ │  │Free  │ │ │  │                    │  │  slot_mapping  │  │
│  │ │  │Queue │ │ │  │                    │  │  계산 (Triton) │  │
│  │ │  └──────┘ │ │  │                    │  └───────┬────────┘  │
│  │ │  ┌──────┐ │ │  │                    │          │           │
│  │ │  │Hash  │ │ │  │                    │          ▼           │
│  │ │  │Map   │ │ │  │                    │  ┌────────────────┐  │
│  │ │  └──────┘ │ │  │                    │  │ Attention      │  │
│  │ └───────────┘ │  │                    │  │ Backend        │  │
│  └───────────────┘  │                    │  │ (FlashAttn 등) │  │
└─────────────────────┘                    │  └───────┬────────┘  │
                                           │          │           │
  CPU 측 (메타데이터 관리)                  │          ▼           │
  ─────────────────────                    │  ┌────────────────┐  │
  GPU 측 (실제 연산)                       │  │ CUDA Kernel    │  │
                                           │  │ (csrc/attention/│  │
                                           │  │  attention_    │  │
                                           │  │  kernels.cuh)  │  │
                                           │  │  block_table   │  │
                                           │  │  로 KV 접근    │  │
                                           │  └────────────────┘  │
                                           └──────────────────────┘
```

```
블록 할당/해제 흐름:

  새 Request 도착
       │
       ▼
  Scheduler.schedule()
       │
       ├─ KVCacheManager.get_num_blocks_to_allocate(request)
       │   └─ SingleTypeKVCacheManager.get_num_blocks_to_allocate()
       │       └─ 필요 블록 수 = ceil(num_tokens / block_size) - 이미할당된 블록
       │
       ├─ 블록 충분한가? ──No──→ preempt (우선순위 낮은 request 해제)
       │                         └─ KVCacheManager.free(victim_req)
       │                             └─ BlockPool.free_blocks()
       │                                 └─ ref_cnt--, 0이면 FreeQueue.append()
       │
       └─ Yes → KVCacheManager.allocate(request)
                 ├─ prefix caching 활성화?
                 │   └─ Yes → BlockPool.get_cached_block(hash)
                 │             ├─ 히트 → ref_cnt++, 블록 재사용
                 │             └─ 미스 → 새 블록 할당
                 └─ BlockPool.get_new_blocks(n)
                     └─ FreeQueue.popleft() × n
                         ├─ cached block이면 evict (hash map에서 제거)
                         └─ ref_cnt = 1로 설정
```

```
주소 변환 흐름 (OS page table walk에 대응):

  논리적 토큰 위치 (request_idx, token_position)
       │
       ▼
  ┌──────────────────────────────┐
  │ block_idx = pos / block_size │   ← VPN = VA / page_size
  │ offset    = pos % block_size │   ← offset = VA % page_size
  └──────────────┬───────────────┘
                 │
                 ▼
  ┌──────────────────────────────────────────┐
  │ block_id = block_table[req_idx][block_idx]│  ← PFN = page_table[VPN]
  └──────────────┬───────────────────────────┘
                 │
                 ▼
  ┌──────────────────────────────────────────┐
  │ slot = block_id * block_size + offset    │  ← PA = PFN * page_size + offset
  └──────────────┬───────────────────────────┘
                 │
                 ▼
  GPU KV cache 텐서에서 해당 slot의 K, V 읽기/쓰기
  k_cache[block_id, head_idx, :, offset, :]
  v_cache[block_id, head_idx, :, offset]
```

---

## 주요 구조체 레퍼런스

### 1. KVCacheBlock (`vllm/v1/core/kv_cache_utils.py:110`)

OS 대응: `struct page` (Linux 커널의 물리 페이지 디스크립터)

```python
@dataclass(slots=True)
class KVCacheBlock:
    block_id: int                              # page frame number
    ref_cnt: int = 0                           # _refcount in struct page
    _block_hash: BlockHashWithGroupId | None    # 콘텐츠 해시 (prefix caching용)
    prev_free_block: KVCacheBlock | None        # free list 이전 노드
    next_free_block: KVCacheBlock | None        # free list 다음 노드
    is_null: bool = False                       # zero page 역할
```

핵심: 실제 KV 데이터는 GPU 텐서(`k_cache`, `v_cache`)에 있고, 이 구조체는 **CPU 측 메타데이터만** 관리. OS에서 `struct page`가 실제 메모리 데이터가 아닌 메타데이터인 것과 동일.

### 2. FreeKVCacheBlockQueue (`vllm/v1/core/kv_cache_utils.py:158`)

OS 대응: 커널의 free page list (per-zone free_area)

```python
class FreeKVCacheBlockQueue:
    num_free_blocks: int
    fake_free_list_head: KVCacheBlock    # sentinel (경계 조건 제거)
    fake_free_list_tail: KVCacheBlock    # sentinel

    # 주요 연산 (모두 O(1)):
    popleft() → KVCacheBlock             # 할당: LRU 순서로 꺼냄
    append(block)                        # 해제: 큐 뒤에 추가 (MRU)
    appendleft(block)                    # 해제: 큐 앞에 추가 (LRU, 우선 evict 대상)
    remove(block)                        # 중간 삭제: ref_cnt 변경 시
```

핵심: 이중 연결 리스트의 O(1) 중간 삭제가 가능한 이유는, `KVCacheBlock`에 직접 `prev/next` 포인터가 있어서 별도 탐색 없이 unlink 가능. OS의 `list_del()` 매크로와 동일 원리.

### 3. BlockPool (`vllm/v1/core/block_pool.py:130`)

OS 대응: page frame allocator (buddy allocator의 단순화 버전)

```python
class BlockPool:
    num_gpu_blocks: int                           # 총 물리 메모리 (블록 단위)
    blocks: list[KVCacheBlock]                    # 전체 블록 배열 (mem_map[])
    free_block_queue: FreeKVCacheBlockQueue        # free list
    cached_block_hash_to_block: BlockHashToBlockMap # prefix cache (page cache)
    null_block: KVCacheBlock                       # zero page

    # 주요 연산:
    get_cached_block(hash, group_ids) → list[Block] | None  # page cache lookup
    cache_full_blocks(request, blocks, ...)                  # 완성된 블록 캐싱
    get_new_blocks(n) → list[Block]                          # 새 블록 n개 할당
    free_blocks(blocks)                                      # 블록 해제 (ref_cnt--)
    evict_blocks(n) → int                                    # LRU eviction
```

### 4. BlockTable (`vllm/v1/worker/block_table.py:18`)

OS 대응: page table (MMU가 참조하는 주소 변환 테이블)

```python
class BlockTable:
    block_table: CpuGpuBuffer   # shape: (max_num_reqs, max_num_blocks_per_req)
                                 # dtype: int32 — 각 엔트리가 block_id (= PFN)
    slot_mapping: CpuGpuBuffer  # shape: (max_num_batched_tokens,)
                                 # dtype: int64 — 최종 물리 슬롯 주소
    block_size: int              # 블록당 토큰 수 (page size)
    use_hybrid_blocks: bool      # 할당 block_size ≠ 커널 block_size일 때

    # 주요 연산:
    append_row(block_ids, row_idx)              # PTE 추가
    compute_slot_mapping(num_reqs, ...)         # Triton 커널로 주소 변환
    commit_block_table(num_reqs)                # CPU → GPU 전송
```

핵심: `CpuGpuBuffer`는 CPU/GPU 양쪽에 미러링된 버퍼. CPU에서 block table을 수정한 후 `commit_block_table()`로 GPU에 복사. OS에서 page table이 메모리에 있지만 TLB에 캐시되는 것과 유사한 2계층 구조.

### 5. BlockHashToBlockMap (`vllm/v1/core/block_pool.py:34`)

OS 대응: page cache (파일 내용의 메모리 캐시)

```python
class BlockHashToBlockMap:
    _cache: dict[BlockHashWithGroupId, KVCacheBlock | dict[int, KVCacheBlock]]
    #         ↑ hash(토큰 시퀀스)          ↑ 해당 블록 (또는 충돌 시 dict)

    # 주요 연산:
    get_one_block(key) → Block | None    # 캐시 조회
    insert(key, block)                   # 캐시 등록 (블록이 가득 찼을 때)
    pop(key, block_id) → Block | None    # 캐시에서 제거 (eviction 시)
```

핵심: 블록의 **내용**(토큰 시퀀스)을 해시하여 동일한 내용의 블록을 재사용. OS에서 `find_get_page(mapping, index)`로 page cache를 조회하는 것과 대응. 해시 충돌 처리를 위해 단일 블록 또는 dict 형태의 union type 사용 (GC 비용 최적화).

### 6. CacheConfig (`vllm/config/cache.py:41`)

OS 대응: 커널 부팅 파라미터 (vm.swappiness, page size 등)

```python
@config
class CacheConfig:
    block_size: int = 16                      # page size (토큰 단위)
    gpu_memory_utilization: float = 0.9       # 물리 메모리 중 사용 비율
    enable_prefix_caching: bool = True        # page cache 활성화 여부
    prefix_caching_hash_algo: str = "sha256"  # 해시 알고리즘
    num_gpu_blocks_override: int | None       # 물리 블록 수 직접 지정
    sliding_window: int | None                # sliding window (working set 크기)
    cache_dtype: CacheDType = "auto"          # KV 캐시 데이터 타입 (fp16/fp8)
```

### 7. KVCacheSpec / FullAttentionSpec (`vllm/v1/kv_cache_interface.py:69, 113`)

OS 대응: page descriptor의 크기/타입 정보

```python
@dataclass(frozen=True)
class KVCacheSpec:
    block_size: int           # 블록당 토큰 수
    page_size_bytes → int     # 블록 하나의 바이트 크기 (property)

@dataclass(frozen=True)
class FullAttentionSpec(AttentionSpec):
    # AttentionSpec으로부터 상속:
    num_kv_heads: int         # KV head 수
    head_size: int            # head 차원
    dtype: torch.dtype        # 데이터 타입
    kv_quant_mode: KVQuantMode  # 양자화 모드

    # page_size_bytes = block_size × num_kv_heads × head_size × 2(K+V) × sizeof(dtype)
```

### 8. CUDA 커널 주요 파라미터 (`csrc/attention/attention_kernels.cuh:85`)

```c++
// paged_attention_kernel 시그니처 (주요 파라미터만):
__device__ void paged_attention_kernel(
    scalar_t* out,                    // 출력 텐서
    const scalar_t* q,                // Query [num_seqs, num_heads, head_size]
    const cache_t* k_cache,           // KV cache [num_blocks, num_kv_heads, head_size/x, block_size, x]
    const cache_t* v_cache,           // KV cache [num_blocks, num_kv_heads, head_size, block_size]
    const int* block_tables,          // 블록 테이블 [num_seqs, max_num_blocks_per_seq]
    const int* seq_lens,              // 시퀀스 길이 [num_seqs]
    ...
)
// Grid: (num_heads, num_seqs, max_num_partitions)

// 핵심 주소 변환 (line 252-253):
const int64_t physical_block_number = static_cast<int64_t>(block_table[block_idx]);
// k_cache 접근: k_cache + physical_block_number * kv_block_stride + ...
```

`k_cache`의 레이아웃이 `[num_blocks, ...]`인 점에 주목 — 전체 GPU KV cache가 하나의 연속 텐서이고, `block_id`가 이 텐서의 첫 번째 차원 인덱스. OS에서 물리 메모리가 하나의 연속 주소 공간이고 PFN이 그 인덱스인 것과 정확히 대응.

---

## 스터디 순서 (6단계)

### Phase 1: 블록 = 페이지 (자료구조 이해)

**목표**: PagedAttention의 가장 기본 단위인 "블록"이 무엇인지, OS의 "페이지"와 어떻게 대응되는지 이해

**읽을 파일**:

1. **`vllm/config/cache.py:40-90`** — `CacheConfig` 클래스
   - `block_size = 16` (OS의 page size = 4KB에 대응)
   - `gpu_memory_utilization`, `enable_prefix_caching` 등 캐시 정책 설정
   - `num_gpu_blocks_override` (OS의 물리 메모리 크기 제한에 대응)

2. **`vllm/v1/kv_cache_interface.py`** — `KVCacheSpec`, `FullAttentionSpec`
   - `page_size_bytes` property: 블록 하나의 실제 바이트 크기 계산
   - 블록 = `block_size × num_kv_heads × head_size × 2(K+V) × dtype_size`

3. **`vllm/v1/core/kv_cache_utils.py:110-155`** — `KVCacheBlock` dataclass
   - `block_id`: page frame number에 대응
   - `ref_cnt`: OS page frame의 reference count
   - `_block_hash`: prefix caching용 (shared page 식별)
   - `prev_free_block`, `next_free_block`: free list 연결 포인터

**포인트**: OS에서 `struct page`가 물리 페이지의 메타데이터이듯, `KVCacheBlock`은 GPU 메모리 블록의 메타데이터. 실제 KV 데이터는 GPU 텐서에 있고, `KVCacheBlock`은 CPU 측 관리 구조체.

---

### Phase 2: Free List = Page Frame Allocator

**목표**: 블록 할당/해제 메커니즘이 OS의 buddy system이나 free list와 어떻게 비교되는지 이해

**읽을 파일**:

1. **`vllm/v1/core/kv_cache_utils.py:158-210`** — `FreeKVCacheBlockQueue`
   - 이중 연결 리스트 기반 free block 큐
   - `popleft()`: 할당 (LRU 순서, OS의 free frame 할당에 대응)
   - `append()` / `appendleft()`: 해제 후 큐에 반환
   - `remove()`: O(1) 중간 삭제 (OS free list는 보통 O(1) 불가)
   - fake head/tail 패턴으로 경계 조건 제거

2. **`vllm/v1/core/block_pool.py:130-250`** — `BlockPool`
   - `__init__`: 모든 블록을 미리 할당 (`blocks = [KVCacheBlock(idx) for idx in range(num_gpu_blocks)]`)
   - `free_block_queue`: FreeKVCacheBlockQueue 인스턴스
   - `null_block`: block_id=0인 특수 블록 (OS의 zero page에 대응)
   - `allocate()` → `free_block_queue.popleft()`
   - `free_blocks()` → ref_cnt 감소, 0이면 큐에 반환
   - LRU eviction order: least recently used → front of queue

**포인트**: OS는 buddy allocator로 연속 프레임 할당이 중요하지만, vLLM은 비연속 할당이 핵심 혁신. 각 request가 비연속적인 블록들을 사용할 수 있어서 메모리 단편화 문제를 해결.

---

### Phase 3: Page Table = Block Table (주소 변환)

**목표**: 논리적 토큰 위치 → 물리적 GPU 메모리 슬롯 변환 과정 이해

**읽을 파일**:

1. **`vllm/v1/worker/block_table.py:18-100`** — `BlockTable` 클래스
   - `block_table: torch.Tensor` shape `(max_num_reqs, max_num_blocks_per_req)` — 2차원 page table
   - `slot_mapping: torch.Tensor` — 최종 물리 주소 (token position → GPU slot)
   - `append_row()`: request에 새 블록 추가 (page table entry 추가)
   - `_compute_slot_mapping_kernel()`: **Triton 커널**로 slot_mapping 계산
     - `slot = block_table[req][pos // block_size] * block_size + pos % block_size`
     - 이것은 OS의 `physical_addr = page_table[VPN] * page_size + offset`과 정확히 동일

2. **`vllm/v1/attention/backend.py`** — `CommonAttentionMetadata`
   - `block_table_tensor`: 모든 request의 block table을 하나의 텐서로 묶음
   - `slot_mapping`: attention 커널에 전달되는 최종 매핑

**포인트**: OS에서 MMU가 하드웨어로 주소 변환하듯, vLLM은 Triton/CUDA 커널이 block table을 참조하여 주소 변환. 차이점은 OS는 per-process page table이지만, vLLM은 batch 내 모든 request의 page table을 하나의 텐서에 패킹.

---

### Phase 4: CUDA 커널 = MMU (하드웨어 수준 주소 변환)

**목표**: 실제 attention 연산에서 block table을 어떻게 사용하는지 커널 코드 이해

**읽을 파일**:

1. **`csrc/attention/attention_kernels.cuh:80+`** — 핵심 커널 템플릿
   - Grid 구조: `(num_heads, num_seqs, max_num_partitions)` — 시퀀스별, 헤드별 병렬 처리
   - `block_table` 배열에서 `physical_block_number` 조회
   - `physical_block_number * block_size`로 KV cache 텐서 내 오프셋 계산
   - 블록 단위로 K, V를 로드하여 attention score 계산

2. **`csrc/attention/paged_attention_v1.cu`** — V1 커널
   - 단일 패스: 모든 블록 순회하며 attention 계산
   - 짧은 시퀀스에 적합

3. **`csrc/attention/paged_attention_v2.cu`** — V2 커널 (partitioned)
   - 2패스: (1) 파티션별 partial attention (2) reduce
   - 긴 시퀀스에 적합 (병렬성 향상)
   - `exp_sums`, `max_logits`, `tmp_out` 중간 버퍼 사용

**포인트**: V1 vs V2는 OS에서 single-level vs multi-level page table walk에 비유 가능. V2의 파티셔닝은 긴 시퀀스를 여러 GPU 워프가 나눠 처리하는 것으로, TLB miss를 줄이기 위해 page walk를 병렬화하는 것과 유사한 동기.

---

### Phase 5: Scheduler = OS Memory Manager (정책 계층)

**목표**: 블록 할당/해제 결정이 스케줄링과 어떻게 연동되는지 이해

**읽을 파일**:

1. **`vllm/v1/core/kv_cache_manager.py`** — `KVCacheManager`
   - `allocate()`: request에 블록 할당
   - `free()`: request 완료 시 블록 회수
   - `get_num_blocks_to_allocate()`: 필요한 블록 수 계산
   - 여러 `SingleTypeKVCacheManager`를 조율 (full attention, sliding window 등)

2. **`vllm/v1/core/single_type_kv_cache_manager.py`** — 타입별 매니저
   - `FullAttentionManager` (~line 420): 표준 attention용
   - `SlidingWindowManager` (~line 481): sliding window용 (OS의 working set과 유사)
   - `req_to_blocks` dict: request별 할당된 블록 추적 (OS의 per-process VMA)

3. **`vllm/v1/core/sched/scheduler.py:67+`** — `Scheduler`
   - `kv_cache_manager.allocate()` 호출하여 새 request에 블록 할당
   - 메모리 부족 시 preemption (OS의 swapping에 대응)
   - `kv_cache_manager.free()` 호출하여 완료된 request의 블록 회수

4. **`vllm/v1/core/kv_cache_coordinator.py`** — `KVCacheCoordinator`
   - `BlockPool`을 공유하며 여러 attention 타입의 캐시 매니저를 조율
   - OS의 zone allocator (DMA zone, Normal zone 등)와 유사한 역할

**포인트**: OS에서 OOM killer가 프로세스를 죽이듯, vLLM scheduler는 메모리 부족 시 request를 preempt. 핵심 차이는 OS는 디스크로 swap하지만, vLLM은 기본적으로 KV cache를 재계산(recompute) — GPU↔CPU swap도 옵션으로 존재.

---

### Phase 6: Prefix Caching = Shared Pages / COW

**목표**: 동일한 프롬프트 접두사를 공유하는 메커니즘 이해 (OS의 shared library 매핑, COW와 비교)

**읽을 파일**:

1. **`vllm/v1/core/block_pool.py:34-128`** — `BlockHashToBlockMap`
   - 블록 해시 → 블록 매핑 (OS의 page cache에 대응)
   - `get_one_block()`: 캐시 히트 시 기존 블록 반환 (ref_cnt 증가)
   - `insert()`: 새 블록을 캐시에 등록
   - `pop()`: eviction 시 캐시에서 제거

2. **`vllm/v1/core/kv_cache_utils.py:36-107`** — 해시 유틸리티
   - `BlockHash`: 블록 내용의 해시 (토큰 시퀀스 기반)
   - `BlockHashWithGroupId`: 해시 + KV cache 그룹 ID
   - `NONE_HASH`: 초기 시드 (체인 해싱의 시작점)

3. **`vllm/v1/core/kv_cache_metrics.py`** — 캐시 성능 측정
   - block residency time, reuse gap 추적
   - hit/miss count (OS의 page cache hit ratio와 동일 개념)

**포인트**: OS에서 같은 shared library를 여러 프로세스가 공유하듯, vLLM에서 같은 system prompt를 가진 여러 request가 동일한 KV cache 블록을 공유. `ref_cnt`가 0이 되어야 블록 회수 가능 — OS의 page reference counting과 동일. 해시 기반 식별은 OS의 content-addressable storage (dedup)와 유사.

---

## 세미나 구성 제안

1. **도입**: OS 가상 메모리 복습 (page, page table, frame allocator, TLB)
2. **문제 제기**: LLM serving에서 KV cache 메모리 관리 문제 (단편화, 낭비)
3. **Phase 1-2**: 블록과 할당자 (OS page/frame 대응)
4. **Phase 3-4**: 주소 변환 (page table → block table → CUDA 커널)
5. **Phase 5**: 스케줄러의 메모리 관리 정책 (OS memory manager 대응)
6. **Phase 6**: Prefix caching (shared pages, COW 대응)
7. **정리**: OS vs vLLM 핵심 차이점
   - OS는 연속 가상 주소 공간 → 비연속 물리 매핑
   - vLLM은 처음부터 비연속 할당이 자연스러움 (이것이 PagedAttention의 핵심 기여)
   - OS는 disk swap, vLLM은 recompute가 기본
   - OS는 하드웨어(MMU) 주소 변환, vLLM은 소프트웨어(CUDA 커널) 주소 변환
