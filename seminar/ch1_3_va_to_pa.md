# 1.3 VA → PA 변환: TLB와 Page Table Walk

---

## 1. 문제: 모든 메모리 접근마다 변환이 필요하다

프로세스가 발행하는 모든 주소는 가상 주소(VA)다.  
CPU가 실제로 데이터를 읽으려면 반드시 물리 주소(PA)로 변환해야 한다.

```mermaid
flowchart LR
    Code["프로세스\narr[i]"] -->|"가상 주소 VA"| MMU["MMU\n(하드웨어)"]
    MMU -->|"물리 주소 PA"| DRAM["DRAM\n(실제 데이터)"]
    Kernel["커널\n(소프트웨어)"] -->|"page table 설정"| MMU
```

변환 비용이 높으면 **모든 메모리 접근이 느려진다** — 이것이 TLB의 존재 이유다.

---

## 2. VA 비트 구조 (x86-64, 4KB page)

x86-64에서 실제로 사용되는 VA는 48비트다 (나머지는 sign extension).

```
48-bit Virtual Address (사용 중인 부분):
┌────────┬────────┬────────┬────────┬──────────────┐
│  PGD   │  PUD   │  PMD   │  PTE   │    Offset    │
│ [47:39]│ [38:30]│ [29:21]│ [20:12]│   [11:0]     │
│  9 bits│  9 bits│  9 bits│  9 bits│   12 bits    │
│ (512개)│ (512개)│ (512개)│ (512개)│  (4096 bytes)│
└────────┴────────┴────────┴────────┴──────────────┘
```

```mermaid
block-beta
  columns 5
  PGD["PGD 인덱스\nbits 47~39\n9 bits = 512 entries"]:1
  PUD["PUD 인덱스\nbits 38~30\n9 bits = 512 entries"]:1
  PMD["PMD 인덱스\nbits 29~21\n9 bits = 512 entries"]:1
  PTE["PTE 인덱스\nbits 20~12\n9 bits = 512 entries"]:1
  OFF["Page Offset\nbits 11~0\n12 bits = 4096 bytes"]:1

  style OFF fill:#ffe08a,stroke:#f0a500
```

- 각 level은 **512개 entry**의 테이블을 인덱싱 (9 bits = 2^9)
- 총 VA 공간: 2^48 = 256 TB
- Offset 12 bits = 4096 bytes = 4 KB page 내 위치

---

## 3. 4-Level Page Table Walk

### 구조 개관

```mermaid
flowchart TD
    CR3["CR3 레지스터\n(현재 프로세스의 PGD 물리 주소)"]

    subgraph Level1["Level 1: PGD (Page Global Directory)"]
        PGD["PGD 테이블\n512 entries × 8 bytes = 4KB\nVA[47:39] 로 인덱싱"]
    end

    subgraph Level2["Level 2: PUD (Page Upper Directory)"]
        PUD["PUD 테이블\n512 entries × 8 bytes = 4KB\nVA[38:30] 로 인덱싱"]
    end

    subgraph Level3["Level 3: PMD (Page Middle Directory)"]
        PMD["PMD 테이블\n512 entries × 8 bytes = 4KB\nVA[29:21] 로 인덱싱"]
    end

    subgraph Level4["Level 4: PTE (Page Table Entry)"]
        PTE["PTE 테이블\n512 entries × 8 bytes = 4KB\nVA[20:12] 로 인덱싱"]
    end

    subgraph Result["결과"]
        PFN["PFN (물리 프레임 번호)\n+ VA[11:0] Offset\n= 물리 주소 PA"]
    end

    CR3 -->|"물리 주소"| Level1
    PGD -->|"다음 테이블 물리 주소"| Level2
    PUD -->|"다음 테이블 물리 주소"| Level3
    PMD -->|"다음 테이블 물리 주소"| Level4
    PTE -->|"PFN 추출"| Result
```

### 단계별 동작

```mermaid
sequenceDiagram
    participant CPU as CPU
    participant CR3 as CR3 레지스터
    participant PGD as PGD 테이블 (물리 메모리)
    participant PUD as PUD 테이블 (물리 메모리)
    participant PMD as PMD 테이블 (물리 메모리)
    participant PTE as PTE 테이블 (물리 메모리)

    CPU->>CR3: 현재 프로세스 PGD 주소 읽기
    CR3-->>CPU: PGD_base (물리 주소)

    CPU->>PGD: PGD_base + VA[47:39] × 8
    PGD-->>CPU: PUD_base (PGD entry에서 추출)

    CPU->>PUD: PUD_base + VA[38:30] × 8
    PUD-->>CPU: PMD_base

    CPU->>PMD: PMD_base + VA[29:21] × 8
    PMD-->>CPU: PTE_base

    CPU->>PTE: PTE_base + VA[20:12] × 8
    PTE-->>CPU: PFN (물리 프레임 번호)

    Note over CPU: PA = PFN × 4096 + VA[11:0]
```

---

## 4. TLB: Translation Lookaside Buffer

4-level walk는 **메모리 접근 4번**을 의미한다. 모든 접근마다 이를 수행하면 성능이 4배 저하된다.

**해결책**: 변환 결과를 하드웨어 캐시에 저장 → **TLB**

```mermaid
flowchart TD
    CPU["CPU: VA 발행"]
    TLB{"TLB 조회\nVA → PA 캐시"}
    Hit["TLB Hit\n~1 cycle\nPA 즉시 반환"]
    Miss["TLB Miss\n~수십 cycle\nPage Table Walk"]
    Walk["4-Level Walk\n메모리 접근 4회"]
    Update["TLB 업데이트\nVPN → PFN 캐싱"]
    PA["PA 확보\n데이터 접근"]

    CPU --> TLB
    TLB -->|"캐시 있음"| Hit
    TLB -->|"캐시 없음"| Miss
    Miss --> Walk
    Walk --> Update
    Update --> PA
    Hit --> PA
```

### TLB 스펙 (일반적인 x86-64 CPU)

| 항목 | L1 ITLB | L1 DTLB | L2 STLB |
|------|---------|---------|---------|
| 용량 | ~128 entries | ~64 entries | ~1536 entries |
| 레이턴시 | 1 cycle | 1 cycle | ~7 cycles |
| 커버리지 (4KB) | 512 KB | 256 KB | 6 MB |
| 커버리지 (2MB) | 256 MB | 128 MB | 3 GB |

### Context Switch 시 TLB

```mermaid
flowchart LR
    subgraph Before["프로세스 A 실행 중"]
        TLBA["TLB\nA의 VPN→PFN 매핑들"]
    end

    subgraph Switch["Context Switch"]
        CR3W["CR3에 B의 PGD 주소 기록"]
        Flush["TLB Flush\n(또는 ASID로 구분)"]
    end

    subgraph After["프로세스 B 실행 중"]
        TLBB["TLB\nB의 VPN→PFN 매핑들\n(처음엔 비어 있음 → miss 급증)"]
    end

    Before --> Switch --> After
```

- Context switch 후 TLB miss가 급증 → **TLB warmup cost**
- ASID (Address Space ID): 프로세스 ID를 TLB entry에 태깅해 flush 없이 공존 가능

---

## 5. Page Table Walk 비용 분석

```mermaid
block-beta
  columns 4
  T1["1회\nPGD 접근\n~100 ns"]:1
  T2["2회\nPUD 접근\n~100 ns"]:1
  T3["3회\nPMD 접근\n~100 ns"]:1
  T4["4회\nPTE 접근\n~100 ns"]:1

  style T1 fill:#ffcccc
  style T2 fill:#ffcccc
  style T3 fill:#ffcccc
  style T4 fill:#ffcccc
```

- TLB miss 1회 = DRAM 접근 4회 = **~400 ns 추가 지연**
- TLB hit = **~1 cycle ≈ 0.3 ns** (캐시 히트 수준)
- hit rate 99% 유지가 성능에 결정적

### Page Table 메모리 오버헤드

```
최악의 경우: 48-bit VA 공간 전체를 flat page table로 만들면?
→ 2^36 entries × 8 bytes = 512 GB (불가능)

4-level 계층적 table: 사용하는 범위만 테이블 생성
→ 일반 프로세스: 수 KB ~ 수 MB 수준
```

---

## 6. Huge Page와 TLB 효율

4KB page의 한계: TLB 64 entries × 4KB = 고작 256KB 커버

```mermaid
quadrantChart
    title Page 크기별 TLB 커버리지 vs 내부 단편화
    x-axis "내부 단편화 작음" --> "내부 단편화 큼"
    y-axis "TLB 커버리지 작음" --> "TLB 커버리지 큼"
    quadrant-1 이상적 (단편화도 크고 커버리지도 큼)
    quadrant-2 최적 영역
    quadrant-3 최악 (둘 다 나쁨)
    quadrant-4 TLB miss 많음

    "4KB (기본)": [0.1, 0.2]
    "2MB (Huge)": [0.5, 0.75]
    "1GB (Huge)": [0.9, 0.95]
```

- **2MB Huge Page**: TLB 64 entries × 2MB = 128MB 커버 → TLB miss 대폭 감소
- 대규모 메모리 접근 워크로드 (DB, HPC, vLLM 추론)에서 효과적

---

## 7. Chapter 2 복선: Block Table = Page Table

vLLM의 `BlockTable`은 Page Table과 동일한 역할을 한다:

```mermaid
flowchart LR
    subgraph OS["Linux 커널"]
        VA2["가상 주소 (VA)\nVPN + Offset"]
        PT2["Page Table\nVPN → PFN"]
        PA2["물리 주소 (PA)\nPFN × 4096 + Offset"]
        VA2 -->|"4-level walk"| PT2 -->|"PFN 반환"| PA2
    end

    subgraph vLLM["vLLM PagedAttention"]
        TI["토큰 인덱스\nlogical_block_num + block_offset"]
        BT["BlockTable\nlogical_block_num → physical_block_num"]
        BA["물리 블록 주소\nphysical_block_num × block_size + offset"]
        TI -->|"1-level lookup"| BT -->|"block_num 반환"| BA
    end

    PT2 -.->|"1:1 대응"| BT
    VA2 -.->|"1:1 대응"| TI
    PA2 -.->|"1:1 대응"| BA
```

- OS: 4-level walk (하드웨어 지원, 복잡)
- vLLM: 1-level lookup (소프트웨어, 단순) — GPU는 가상 메모리 없음
- 핵심 아이디어는 동일: **논리 → 물리 매핑 테이블**
