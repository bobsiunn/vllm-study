# 세미나 구성안: OS 메모리 시스템과 vLLM PagedAttention

---

## Chapter 1: OS 메모리 시스템

> 목적: Chapter 2에서 vLLM과 비교할 OS 개념들을 **필요한 만큼만** 복습한다.
> 청중이 OS 전공자라면 리마인드 수준, 비전공자라면 기초 확립 목적.

### 1.1 왜 가상 메모리인가 — 문제 정의

- 물리 메모리의 한계: 프로세스마다 연속 메모리를 요구하면 단편화 발생
- 핵심 아이디어: **고정 크기 단위(page)로 쪼개서 비연속 할당** → 연속인 것처럼 보이게
- 이 아이디어가 왜 중요한지: vLLM이 정확히 같은 문제를 LLM KV cache에서 풀었음 (Chapter 2 복선)

### 1.2 Page와 Page Frame

- **Page**: 가상 주소 공간의 고정 크기 단위 (보통 4KB)
- **Page Frame**: 물리 메모리의 고정 크기 슬롯 (page와 동일 크기)
- 핵심 분리: 논리적 단위(page) vs 물리적 단위(frame) — 이 분리가 유연한 메모리 관리의 기반
- `struct page`: Linux 커널이 각 물리 프레임을 추적하는 메타데이터 구조체
  - `_refcount`: 참조 카운트
  - `flags`: 상태 플래그 (dirty, locked, active 등)
  - `lru`: LRU 리스트 연결 포인터

### 1.3 VA → PA: Page Table과 주소 변환

- **Page Table**: 가상 페이지 번호(VPN) → 물리 프레임 번호(PFN) 매핑
- 주소 변환 공식:
  ```
  VPN    = VA / page_size
  offset = VA % page_size
  PFN    = page_table[VPN]
  PA     = PFN * page_size + offset
  ```
- **MMU**: 하드웨어가 이 변환을 수행 (소프트웨어 개입 없이)
- **TLB**: 자주 쓰는 변환 결과를 캐시하여 page table 접근 비용 절감
- Multi-level page table: 희소한 주소 공간을 효율적으로 표현 (2-level, 4-level)
- **TLB miss 시 page table walk 비용**: 4-level PT에서 최악 4번 메모리 접근 (각각 ~100 cycle)

### 1.4 PA → Data: 물리 메모리의 실제 접근 경로

> PA가 나왔다고 데이터가 바로 오는 것이 아니다.
> PA에서 실제 데이터 비트가 CPU 레지스터에 도달하기까지의 물리적 경로를 추적한다.

#### 1.4.1 CPU Cache 계층 (PA → Cache Line)

```
CPU가 PA를 발행
  │
  ▼
L1 Cache (1~4 cycle, 32-64KB)
  │ miss
  ▼
L2 Cache (4~12 cycle, 256KB-1MB)
  │ miss
  ▼
L3 / LLC (20~40 cycle, 수 MB~수십 MB, 코어 간 공유)
  │ miss
  ▼
Memory Controller → DRAM 접근 시작
```

- **Cache line**: 캐시의 전송 단위 (보통 64 bytes)
  - PA의 하위 비트로 cache line 내 offset 결정
  - PA의 중간 비트로 set index, 상위 비트로 tag 비교
- **Spatial locality**: 연속 접근 시 같은 cache line에서 히트 → 매우 빠름
- **Temporal locality**: 최근 접근한 데이터 재접근 시 캐시 히트
- cache miss 시 비로소 DRAM에 실제 접근 → 아래 경로로 내려감

#### 1.4.2 DRAM 내부 구조 (Memory Controller → Bit)

```
PA (Physical Address) 분해:
┌──────────┬───────────┬──────┬────────┬──────────────┬─────────────┐
│ Channel  │   Rank    │ Bank │  Row   │   Column     │ Byte offset │
│  (1-2b)  │  (1-2b)   │(3-4b)│(14-17b)│  (7-10b)     │   (3-6b)    │
└──────────┴───────────┴──────┴────────┴──────────────┴─────────────┘
```

**Channel → Rank → Bank → Row → Column 계층:**

- **Channel**: 독립적인 메모리 버스 (dual-channel이면 2개)
  - 채널 간 접근은 완전 병렬 → 대역폭 2배
- **Rank**: 같은 채널의 DIMM 면 (앞면/뒷면 또는 DIMM 간)
  - 같은 채널의 rank는 버스를 공유 → 번갈아 접근
- **Bank** (핵심): 같은 rank 내의 독립 메모리 유닛 (DDR4: 16개, DDR5: 32개)
  - **Bank 간 접근은 병렬 가능** → bank-level parallelism
  - 같은 bank의 다른 row 접근 시 **row conflict** → 비용 큼
- **Row** (= DRAM Page): bank 내의 가로줄 (보통 8KB)
  - **Row Buffer**: 현재 열려 있는 row의 사본 (SRAM, 고속)
  - Row hit: 같은 row의 다른 column 접근 → ~13ns (CAS latency만)
  - Row miss (= Row conflict): 다른 row 접근 → ~28-36ns (precharge + activate + CAS)
- **Column**: row 내의 특정 위치
  - column 접근 시 **burst length** (8 또는 16)만큼 연속 데이터 전송

#### 1.4.3 DRAM 접근 시나리오별 레이턴시

```
시나리오                          레이턴시     원인
─────────────────────────────────────────────────────────
L1 cache hit                      ~1ns       SRAM, 파이프라인 내
L2 cache hit                      ~4ns       SRAM
L3 cache hit                      ~12ns      SRAM, 코어 간 공유
DRAM row buffer hit (같은 row)     ~13ns      CAS만 필요
DRAM row buffer miss (빈 bank)     ~22ns      RAS + CAS
DRAM row conflict (다른 row)       ~35ns      PRE + RAS + CAS
                                              (row 닫고 → 새 row 열고 → 읽기)
```

#### 1.4.4 Random Access vs Sequential Access — 왜 차이가 나는가

**Sequential (연속 접근)**:
- 같은 row buffer에서 연속 column을 읽음 → row hit 연속 → ~13ns/접근
- Prefetch가 동작: CPU가 다음 cache line을 미리 요청
- DRAM burst mode: 한 번의 RAS로 여러 column을 burst 전송
- 결과: **높은 bandwidth 활용** (DDR5-5600 기준 이론 ~44.8 GB/s/channel)

**Random (무작위 접근)**:
- 매 접근마다 다른 row, 다른 bank → row miss/conflict 빈발
- Prefetch 무효: 다음 주소를 예측할 수 없어 prefetch 불가
- Row buffer를 계속 닫았다 열었다 → ~35ns/접근
- 결과: **latency 지배적**, 실효 bandwidth 이론치의 20-30%

**OS에서의 영향**:
- Page 크기 4KB = 약 64개 cache line → page 내부는 sequential에 가까움
- Page walk 자체는 random (multi-level PT의 각 level이 다른 위치)
- TLB는 이 random walk 비용을 숨기는 것
- **Huge page (2MB, 1GB)**: TLB miss 빈도 감소 + 더 넓은 범위에서 spatial locality 확보

#### 1.4.5 Prefetch 메커니즘

- **Hardware prefetcher**: CPU가 접근 패턴 탐지 → 다음 cache line 미리 fetch
  - stride prefetcher: 일정 간격 접근 패턴 감지 (배열 순회 등)
  - next-line prefetcher: 인접 cache line 미리 로드
  - 효과: sequential 접근 시 DRAM latency를 거의 숨길 수 있음
- **Software prefetch**: `__builtin_prefetch()` 등으로 명시적 힌트
  - OS 커널의 `prefetchw` 활용 (page table walk 등)
- **Prefetch 실패 시나리오**: random 접근, 불규칙 stride → prefetch가 오히려 대역폭 낭비
- **DRAM 측 open page policy**: row buffer를 열어두고 다음 접근 기대
  - hit rate 높으면 유리, 낮으면 close page policy가 나음

#### 1.4.6 정리 — VA부터 Data까지 전체 경로

```
CPU: load [VA]
  │
  ├─ (1) TLB lookup ─────── hit → PA 즉시 확보
  │                          miss → page table walk (4번 메모리 접근, 각각 아래 경로 반복)
  │
  ├─ (2) PA로 cache 조회 ── L1 hit → 1ns, 끝
  │                          L1 miss → L2 → L3 → miss면 (3)으로
  │
  ├─ (3) Memory Controller
  │       PA를 channel/rank/bank/row/column으로 디코딩
  │
  ├─ (4) Bank에 명령 발행
  │       ├─ Row buffer hit → CAS만 → ~13ns
  │       ├─ Row buffer empty → RAS + CAS → ~22ns
  │       └─ Row conflict → PRE + RAS + CAS → ~35ns
  │
  ├─ (5) Burst 전송: 64 bytes (cache line) 단위로 bus를 통해 전달
  │
  └─ (6) Cache에 채우고 CPU 레지스터에 전달 → load 완료
```

**이 경로가 중요한 이유 (Chapter 2 복선)**:
- GPU의 HBM은 이 구조가 **근본적으로 다름** (bank 수, 대역폭, latency 특성)
- vLLM의 PagedAttention이 block 단위로 접근하는 것은 이 물리 계층을 고려한 설계
- paged attention 커널의 warp 간 블록 분배는 bank-level parallelism과 유사한 동기

### 1.5 Page Frame Allocator

- **Free list**: 사용 가능한 프레임들의 연결 리스트
  - Linux의 `free_area` per-zone 관리
  - 할당: 리스트에서 꺼냄, 해제: 리스트에 반환
- **Buddy allocator**: 연속 프레임 할당을 위한 이진 분할 알고리즘
  - 2^n 단위로 분할/병합 → 외부 단편화 완화
  - vLLM과의 핵심 차이점 복선: vLLM은 연속 할당이 **불필요**하므로 buddy가 필요 없음
- `mem_map[]`: 전체 물리 프레임의 `struct page` 배열 — 인덱스가 곧 PFN

### 1.6 Page Replacement (교체 정책)

- 물리 메모리 부족 시: 어떤 프레임을 회수할 것인가?
- **LRU 기반 교체**: Least Recently Used 프레임을 우선 교체
  - Linux의 active/inactive 리스트 (근사 LRU)
  - 이중 연결 리스트 + `list_del()`로 O(1) 제거
- **Reference count**: `_refcount`가 0이 되어야 프레임 회수 가능
- **Swap**: 디스크로 내보내고 나중에 다시 로드 (page fault로 트리거)

### 1.7 Shared Pages와 Copy-on-Write (COW)

- **Shared mapping**: 여러 프로세스가 동일한 물리 프레임을 공유
  - 대표 사례: shared library (libc.so 등)
  - 같은 PFN을 여러 page table이 참조 → `_refcount` 증가
- **Copy-on-Write**: fork() 시 부모/자식이 같은 프레임을 공유하다가, 쓰기 발생 시 복사
  - 읽기 전용인 동안은 공유 → 메모리 절약
- **Page Cache**: 파일 내용을 메모리에 캐시
  - `find_get_page(mapping, index)`로 조회
  - 같은 파일을 여러 프로세스가 읽으면 동일 프레임 공유

### 1.8 OOM과 프로세스 관리

- 물리 메모리가 완전히 부족할 때: **OOM Killer**가 프로세스를 종료
- Swap이 있으면 디스크로 밀어내지만, 성능 저하 심각
- 스케줄러의 메모리 인식: 프로세스 실행 시 충분한 메모리가 있는지 확인

### 1.9 Chapter 1 정리 — Chapter 2로의 브릿지

핵심 개념 체크리스트 (이것들이 Chapter 2에서 1:1 매핑됨):

| 개념 | 역할 | Chapter 2 대응 |
|------|------|----------------|
| Page (4KB) | 관리 단위 | KVCacheBlock (16 tokens) |
| Page Frame | 물리 슬롯 | GPU 메모리 블록 |
| `struct page` | 프레임 메타데이터 | `KVCacheBlock` dataclass |
| Page Table | 주소 변환 테이블 | `BlockTable` 텐서 |
| `VPN→PFN→PA` | 주소 변환 공식 | `block_idx→block_id→slot` |
| MMU | 하드웨어 주소 변환 | CUDA 커널 |
| DRAM (ch/rank/bank/row/col) | 물리 데이터 접근 | HBM (stack/bank/row/col) |
| Cache line (64B) + prefetch | 접근 최적화 | Coalesced access + warp scheduling |
| Row buffer hit/miss | 접근 레이턴시 결정 | Bank-level parallelism |
| Free list | 가용 프레임 관리 | `FreeKVCacheBlockQueue` |
| Buddy allocator | 연속 프레임 할당 | 불필요 (비연속이 기본) |
| LRU replacement | 교체 정책 | LRU eviction |
| `_refcount` | 참조 카운트 | `ref_cnt` |
| Shared pages | 프레임 공유 | Prefix Caching |
| Page Cache | 파일 내용 캐시 | `BlockHashToBlockMap` |
| OOM Killer | 메모리 부족 대응 | Request preemption |
| Swap (disk) | 메모리 확장 | Recompute (재계산) |

---

## Chapter 2: vLLM PagedAttention — OS 메모리의 GPU 재해석

> 목적: Chapter 1의 각 OS 개념이 vLLM에서 어떻게 대응되는지 **코드와 함께** 보여준다.
> 매 섹션에서 "OS에서는 X → vLLM에서는 Y" 패턴으로 설명.

### 2.1 문제 정의 — LLM Serving의 메모리 문제

- KV cache란 무엇인가: Transformer의 attention 연산에서 이전 토큰의 Key, Value를 저장
- 기존 방식의 문제:
  - request마다 **최대 시퀀스 길이만큼** 연속 메모리를 미리 할당
  - 실제로는 시퀀스 길이가 천차만별 → 60-80% 메모리 낭비 (내부 단편화)
  - 여러 request 간 메모리 공유 불가 → 외부 단편화
- OS 비유: `malloc(MAX_SEQ_LEN)`을 매번 하는 것과 같음 → 이것이 왜 나쁜지 OS 전문가는 즉시 이해

### 2.2 핵심 아이디어 — "KV Cache에 Paging을 적용하자"

- OS의 해법을 그대로 차용: **고정 크기 블록으로 쪼개서 비연속 할당**
- `page_size = 4KB` → `block_size = 16 tokens`
- 차이점: OS는 연속 가상 주소 공간이 전제 → vLLM은 **처음부터 비연속이 자연스러움**
  - Attention 연산은 KV cache를 순서대로 읽지만, 물리적 연속성은 불필요
  - 이것이 PagedAttention의 핵심 통찰

### 2.3 `struct page` → `KVCacheBlock` (블록 메타데이터)

**OS 복기**: `struct page`는 물리 프레임의 메타데이터. 실제 데이터가 아님.

**vLLM 대응**: `KVCacheBlock` (`vllm/v1/core/kv_cache_utils.py:110`)

| `struct page` 필드 | `KVCacheBlock` 필드 | 역할 |
|---|---|---|
| PFN (배열 인덱스) | `block_id` | 물리 블록 식별자 |
| `_refcount` | `ref_cnt` | 참조 카운트 (공유 시 증가) |
| `flags` | `is_null`, `_block_hash` | 상태 정보 |
| `lru` (리스트 포인터) | `prev_free_block`, `next_free_block` | free list 연결 |

강조할 점:
- 실제 KV 데이터는 GPU 텐서(`k_cache[num_blocks, ...]`)에 있음
- `KVCacheBlock`은 CPU 측 메타데이터만 관리
- `struct page`가 실제 메모리 내용을 담지 않는 것과 정확히 동일한 설계

### 2.4 `mem_map[]` + Free list → `BlockPool` + `FreeKVCacheBlockQueue` (할당자)

**OS 복기**: `mem_map[]`은 전체 `struct page` 배열, free list는 가용 프레임 관리.

**vLLM 대응**:
- `BlockPool.blocks[]` = `mem_map[]` (`vllm/v1/core/block_pool.py:130`)
- `FreeKVCacheBlockQueue` = free page list (`vllm/v1/core/kv_cache_utils.py:158`)

비교 포인트:
- **OS**: buddy allocator로 2^n 연속 프레임 할당 가능 → vLLM은 이것이 **불필요**
- **OS**: `list_del()` 매크로로 O(1) 리스트 조작 → vLLM도 동일 (이중 연결 리스트)
- **OS**: sentinel node 패턴 → vLLM의 `fake_free_list_head/tail`과 동일
- **OS**: zero page (read-only 공유) → vLLM의 `null_block` (padding용)

할당/해제 흐름 비교:
```
OS:   alloc_pages() → free_area에서 꺼냄 → refcount=1
vLLM: get_new_blocks() → FreeQueue.popleft() → ref_cnt=1

OS:   free_pages() → refcount-- → 0이면 free_area에 반환
vLLM: free_blocks() → ref_cnt-- → 0이면 FreeQueue.append()
```

### 2.5 Page Table → `BlockTable` (주소 변환)

**OS 복기**: page table은 VPN→PFN 매핑. MMU가 하드웨어로 변환.

**vLLM 대응**: `BlockTable` (`vllm/v1/worker/block_table.py:18`)

주소 변환 공식 비교:
```
OS:   PA  = page_table[VPN] * page_size + offset
       ↕
vLLM: slot = block_table[req][pos // block_size] * block_size + pos % block_size
```

비교 포인트:
- **OS**: per-process page table → vLLM: **per-request** block table (개념 동일)
- **OS**: page table이 메모리에 존재, TLB가 캐시 → vLLM: CPU에서 구성, GPU로 복사 (`commit_block_table()`)
- **OS**: multi-level page table (희소 주소 공간 효율화) → vLLM: **flat 1-level** (시퀀스가 dense하므로)
- **구현 차이**: vLLM은 모든 request의 block table을 하나의 2D 텐서 `(num_reqs, max_blocks_per_req)`에 패킹 → batch 처리에 유리

### 2.6 MMU → CUDA 커널 (주소 변환)

**OS 복기**: MMU가 하드웨어로 주소 변환. 소프트웨어 개입 없이 매 메모리 접근마다.

**vLLM 대응**: CUDA 커널 내부에서 소프트웨어로 변환 (`csrc/attention/attention_kernels.cuh:252`)

```c++
// CUDA 커널 내 주소 변환 (OS의 MMU에 대응):
const int64_t physical_block_number = block_table[block_idx];   // PFN 조회
k_ptr = k_cache + physical_block_number * kv_block_stride + ...; // PA 계산
```

비교 포인트:
- **OS**: 하드웨어(MMU) 변환 → vLLM: 소프트웨어(CUDA 커널) 변환
- **OS**: TLB miss → page table walk (수십 사이클) → vLLM: block_table은 GPU L2 cache에 들어갈 정도로 작음 (수십~수백 KB)
- **Paged Attention V1 vs V2**:
  - V1: 한 워프가 전체 시퀀스의 모든 블록 순회 (짧은 시퀀스에 적합)
  - V2: 파티션별 병렬 처리 후 reduce (긴 시퀀스에 적합)
  - OS 비유: V1 = single-level walk, V2 = 여러 코어가 page walk를 분담하는 것과 유사

`k_cache` 텐서 레이아웃 설명:
```
k_cache shape: [num_blocks, num_kv_heads, head_size/x, block_size, x]
                ↑ block_id가 이 차원의 인덱스 = PFN이 물리 메모리의 인덱스인 것과 동일
```

### 2.7 DRAM vs HBM — 물리 메모리 접근의 근본적 차이

> Chapter 1.4에서 CPU DRAM의 접근 경로를 봤다면, 여기서는 GPU HBM의 접근 경로를 비교한다.
> PagedAttention의 block 크기와 접근 패턴이 왜 이렇게 설계되었는지 물리 계층에서 이해.

#### 2.7.1 DRAM vs HBM 아키텍처 비교

```
CPU + DDR5 DRAM                         GPU + HBM2e/HBM3
──────────────────                      ──────────────────
채널: 2-4개                              채널 (스택): 6-12개 (HBM3: 12)
버스 폭: 64-bit/channel                 버스 폭: 1024-bit/stack (!)
Bank: 16-32/rank                        Bank: 32/channel (pseudo-channel 포함)
대역폭: ~50-100 GB/s (총합)             대역폭: ~2-5 TB/s (총합)
레이턴시: ~35ns (row miss)              레이턴시: ~100-120ns (row miss)
위치: 별도 DIMM 슬롯                    위치: GPU 다이 위에 3D 적층

핵심 차이: HBM은 latency는 더 높지만, bandwidth가 ~30-50배 높음
→ "latency를 bandwidth로 숨기는" 설계
```

#### 2.7.2 GPU 메모리 계층과 접근 경로

```
GPU thread가 k_cache[block_id, ...] 접근
  │
  ├─ (1) L1 Cache / Shared Memory (per-SM)
  │       ~28 cycle, 48-228 KB
  │       같은 warp 내 thread들이 같은 cache line 접근 시 → coalesced
  │
  ├─ (2) L2 Cache (전체 SM 공유)
  │       ~200 cycle, 40-96 MB (A100: 40MB, H100: 50MB)
  │       block_table 자체는 여기 캐시됨 (수 KB~수백 KB)
  │
  └─ (3) HBM (Global Memory)
         ~400-800 cycle, 40-80 GB
         k_cache, v_cache 텐서의 실제 데이터
```

#### 2.7.3 PagedAttention에서의 접근 패턴 분석

**Block table 접근** (Chapter 1의 page table walk에 대응):
```
block_table 크기: num_reqs × max_blocks_per_req × 4 bytes (int32)
예: 256 reqs × 512 blocks × 4B = 512 KB → L2 cache에 충분히 들어감
→ CPU의 TLB 역할을 GPU L2가 수행 (거의 항상 히트)
```

**KV cache 접근** (Chapter 1의 DRAM 접근에 대응):
```
하나의 block 접근 크기:
  block_size × head_size × sizeof(dtype)
  = 16 tokens × 128 dim × 2 bytes (fp16) = 4 KB per head per K or V

  전체 KV heads 포함: 4KB × num_kv_heads × 2(K+V)
  GQA 8 heads: 4KB × 8 × 2 = 64 KB per block
```

**Sequential vs Random 접근 — DRAM과의 차이**:

| 접근 패턴 | CPU DRAM | GPU HBM |
|-----------|----------|---------|
| **Block 내부** (같은 block의 tokens) | row buffer hit 가능 (~13ns) | **coalesced access** — warp 내 thread들이 연속 주소 접근 시 하나의 트랜잭션으로 병합 |
| **Block 간** (다른 block_id로 점프) | row conflict 가능 (~35ns) | 다른 HBM bank로 분산 가능 — **bank-level parallelism**으로 latency 숨김 |
| **Random 접근 비용** | prefetch 실패, bandwidth 낭비 | 수천 thread가 동시 접근 → **latency hiding** (한 warp이 stall하면 다른 warp 실행) |

GPU가 random 접근을 CPU보다 잘 견디는 이유:
1. **Massive parallelism**: 수천 개의 concurrent thread → memory latency hiding
2. **HBM의 넓은 버스**: 1024-bit로 한 번에 128 bytes 전송
3. **Bank 수**: HBM bank가 매우 많아서 동시 접근 시 bank conflict 확률 낮음
4. **Warp scheduling**: 한 warp이 memory stall이면 즉시 다른 warp으로 전환

#### 2.7.4 vLLM이 이 물리 계층을 활용하는 방식

**Block 크기 선택 (16 tokens)의 물리적 근거**:
- 16 tokens × 128 dim × 2B = 4KB per head → HBM의 burst 전송 단위에 정렬
- 너무 작으면: block table 오버헤드 증가, coalesced access 효율 저하
- 너무 크면: 내부 단편화 증가 (마지막 블록의 낭비)
- 16은 warp size(32)의 절반 → thread-block 매핑에 자연스러움

**Warp-block 매핑** (`csrc/attention/attention_kernels.cuh`):
```c++
// 각 warp이 하나의 block을 담당 → bank-level parallelism 활용
for (int block_idx = start_block_idx + warp_idx; block_idx < end_block_idx;
     block_idx += NUM_WARPS) {
    const int64_t physical_block_number = block_table[block_idx];
    // 각 warp이 서로 다른 physical_block_number를 접근
    // → HBM의 서로 다른 bank에 분산될 확률 높음
}
```

- OS 대응: 다중 코어가 서로 다른 bank를 접근하여 병렬성 확보하는 것과 동일 동기
- 차이점: CPU는 코어 수가 적어서(~수십) bank parallelism 활용이 제한적
  GPU는 warp 수가 많아서(수천) 자연스럽게 bank 간 분산

**Coalesced access 패턴** (Chapter 1의 spatial locality에 대응):
```
Block 내부: thread 0→token 0, thread 1→token 1, ... (연속 메모리)
→ coalesced: 하나의 128B 트랜잭션으로 병합
→ CPU에서 같은 cache line 내 sequential 접근이 빠른 것과 동일 원리
  단, GPU는 32 thread가 동시에 연속 접근 → 더 효율적
```

**Prefetch 비교**:
```
CPU:  hardware prefetcher가 stride 감지 → next cache line 미리 fetch
      random 접근 시 무효화

GPU:  prefetch 대신 "thread-level parallelism"으로 latency hiding
      warp A가 memory wait → warp B 실행 → warp C 실행 → ...
      A의 데이터 도착 시 A 재개
      → prefetch가 아닌 "massive occupancy"로 latency를 숨김
```

#### 2.7.5 정리 — 물리 접근 경로 대비

```
CPU (Chapter 1.4):                    GPU (Chapter 2.7):
VA → TLB → PA                        slot = block_table[idx] * bs + offset
PA → L1 → L2 → L3 → DRAM            addr → L1/SMEM → L2 → HBM
DRAM: ch/rank/bank/row/col            HBM: stack/bank/row/col
row conflict: ~35ns                   bank miss: ~400 cycle (~200ns)
prefetch로 숨김                       thread parallelism으로 숨김
bandwidth: ~50-100 GB/s               bandwidth: ~2-5 TB/s
```

PagedAttention이 작동하는 이유의 물리적 본질:
- **block 단위 접근** → HBM burst와 coalesced access에 정렬
- **block 간 random jump** → GPU의 massive parallelism + 넓은 HBM 버스로 비용 흡수
- CPU에서 같은 설계를 하면 random 접근 비용이 치명적이지만, GPU에서는 자연스러움
- 즉, **PagedAttention은 GPU 메모리 계층의 특성에 최적화된 paging 구현**

### 2.8 LRU Replacement → Block Eviction (교체 정책)

**OS 복기**: 메모리 부족 시 LRU 프레임을 swap out.

**vLLM 대응**: `BlockPool.evict_blocks()` + `FreeKVCacheBlockQueue`의 LRU 순서

비교 포인트:
- **OS**: active/inactive 리스트로 근사 LRU → vLLM: 정확한 LRU (이중 연결 리스트 순서)
- **OS**: eviction → swap to disk → page fault 시 swap in → vLLM: eviction → **recompute** (재계산이 기본)
  - GPU↔CPU swap도 옵션이지만 PCIe 대역폭 병목으로 recompute가 보통 더 빠름
  - 이것은 OS 관점에서 흥미로운 차이: "디스크보다 재계산이 빠른" 환경
- **OS**: OOM Killer가 프로세스 자체를 종료 → vLLM: Scheduler가 request를 **preempt** (나중에 재개 가능)

### 2.9 Shared Pages / Page Cache → Prefix Caching (블록 공유)

**OS 복기**: 같은 shared library를 여러 프로세스가 공유. page cache로 파일 내용 캐시.

**vLLM 대응**: `BlockHashToBlockMap` (`vllm/v1/core/block_pool.py:34`)

동작 비교:
```
OS:   find_get_page(mapping, index)  → 파일의 해당 페이지가 캐시에 있는가?
vLLM: get_cached_block(hash, ids)    → 이 토큰 시퀀스의 블록이 캐시에 있는가?

OS:   캐시 히트 → refcount++ → 물리 프레임 재사용
vLLM: 캐시 히트 → ref_cnt++ → GPU 블록 재사용 (KV 재계산 불필요)

OS:   캐시 미스 → 디스크에서 읽어서 새 프레임에 로드
vLLM: 캐시 미스 → 새 블록 할당 후 모델 forward로 KV 계산
```

실용 시나리오:
- 같은 system prompt ("You are a helpful assistant...")를 가진 수백 개 요청
- 첫 번째 요청: 블록 채우고 해시 등록
- 이후 요청: 해시 조회로 즉시 재사용 → prefill 계산 스킵 → **대규모 throughput 향상**

OS 대비 vLLM의 차별점:
- OS page cache: **파일 오프셋** 기반 식별 (같은 파일의 같은 위치)
- vLLM prefix cache: **콘텐츠 해시** 기반 식별 (내용이 같으면 공유)
  - content-addressable storage에 더 가까움
  - 해시 알고리즘 선택 가능: sha256 (안전), xxhash (빠름)

### 2.10 종합 비교 — 설계 철학의 차이

| 관점 | OS 가상 메모리 | vLLM PagedAttention |
|------|---------------|---------------------|
| **관리 대상** | 범용 데이터 (프로세스 메모리) | KV cache (attention 연산 전용) |
| **연속성 요구** | 가상 주소는 연속, 물리는 비연속 | 처음부터 비연속이 자연스러움 |
| **주소 변환** | 하드웨어 (MMU) | 소프트웨어 (CUDA 커널) |
| **할당 단위** | 다양한 크기 필요 (buddy) | 고정 크기만 (단순 free list) |
| **교체 시 비용** | Disk I/O (ms 단위) | Recompute (us~ms 단위) |
| **공유 식별** | 파일+오프셋 (inode 기반) | 콘텐츠 해시 (content-addressable) |
| **메모리 부족 대응** | OOM Kill (프로세스 종료) | Preempt (요청 일시 중단, 재개 가능) |
| **메타데이터 위치** | 같은 메모리 (CPU RAM) | 분리 (메타데이터=CPU, 데이터=GPU) |

핵심 메시지:
- vLLM은 OS 가상 메모리의 핵심 아이디어를 **LLM serving 도메인에 맞게 단순화**한 것
- OS는 범용성을 위해 복잡한 구조(multi-level PT, buddy, swap) 필요
- vLLM은 도메인 특성(고정 크기, 비연속 OK, recompute 가능)을 활용해 더 단순하면서도 효과적인 설계 도출
- "좋은 시스템 설계는 도메인의 제약을 정확히 이해하는 것에서 시작된다"
