# 1.6 Page Replacement: 메모리 부족 시 어떤 페이지를 내쫓는가

---

## 1. 문제 정의

물리 메모리가 부족할 때, 어떤 페이지를 **Swap out** (디스크로 내보내기) 할 것인가?

```mermaid
flowchart TD
    Need["새 페이지 필요\n(page fault 또는 할당 요청)"]

    Check{"물리 메모리\n여유 있음?"}

    Alloc["즉시 할당"]
    Select["교체 대상 선택\n(Page Replacement Policy)"]
    SwapOut["선택된 페이지 Swap out\n(디스크에 기록 → 프레임 해제)"]
    SwapIn["새 페이지 Swap in\n(프레임에 로드)"]

    Need --> Check
    Check -->|"Yes"| Alloc
    Check -->|"No"| Select --> SwapOut --> SwapIn
```

---

## 2. LRU와 Linux의 Active/Inactive 리스트

### 이상적인 OPT (Optimal) 알고리즘 (구현 불가)
- 미래에 가장 늦게 사용될 페이지를 교체
- 이론적 최적이지만 미래를 알 수 없음

### Linux의 실용적 해답: 2-리스트 LRU

```mermaid
stateDiagram-v2
    [*] --> Inactive: 페이지 최초 로드\n(inactive list tail에 삽입)

    Inactive --> Active: PG_referenced 2번 set\n(두 번 이상 참조됨)
    Active --> Inactive: 오랫동안 참조 없음\n(active list head → inactive tail)

    Inactive --> Evicted: 메모리 압박\ninactive list tail에서 제거\n(swap out 또는 해제)

    note right of Inactive
        PTE의 Accessed bit로 참조 추적
        (MMU가 접근 시 자동 set)
    end note

    note right of Active
        "hot" 페이지들
        자주 사용되므로 보호
    end note
```

### 리스트 구조

```mermaid
flowchart TD
    subgraph ActiveList["Active List (자주 사용됨)"]
        direction LR
        AH["Head\n(최근 사용)"]
        A2["Page B"]
        A3["Page F"]
        AT["Tail\n(오래됨) →\ninactive로 강등"]
    end

    subgraph InactiveList["Inactive List (교체 후보)"]
        direction LR
        IH["Head\n(새로 들어옴)"]
        I2["Page C"]
        I3["Page A"]
        IT["Tail\n(교체 후보 1순위)"]
    end

    AT -->|"강등"| IH
    IT -->|"Swap out"| Disk["디스크 (Swap Area)"]
    Disk -->|"Swap in (page fault)"| IH
```

---

## 3. PTE Accessed Bit 활용

```mermaid
sequenceDiagram
    participant CPU as CPU
    participant MMU as MMU (하드웨어)
    participant PTE as PTE
    participant Kernel as 커널 (kswapd)

    CPU->>MMU: VA 접근 (read/write)
    MMU->>PTE: Accessed bit = 1 자동 set
    
    Note over Kernel: 주기적으로 (kswapd 실행)
    Kernel->>PTE: Accessed bit 읽기
    alt Accessed bit = 1
        Kernel->>PTE: Accessed bit = 0 으로 클리어
        Kernel->>PTE: PG_referenced 카운터 증가
    else Accessed bit = 0
        Note over Kernel: 최근 접근 없음 → 교체 후보
    end
```

- MMU가 하드웨어적으로 자동 set → 소프트웨어 오버헤드 없음
- 커널이 주기적으로 스캔하며 클리어 → "최근 2번 접근" 여부 추적

---

## 4. Swap 동작 상세

```mermaid
sequenceDiagram
    participant PF as Page Fault Handler
    participant PT as Page Table
    participant Alloc as Page Allocator
    participant Swap as Swap I/O

    Note over PF: 접근한 VA의 PTE Present=0

    PF->>PT: PTE 확인
    alt Swap Entry (Present=0, swap code 있음)
        PF->>Alloc: 새 프레임 할당 (필요 시 교체)
        PF->>Swap: 디스크에서 페이지 읽기 (I/O)
        Swap-->>PF: 데이터 로드 완료
        PF->>PT: PTE 업데이트 (Present=1, PFN 설정)
        PF-->>CPU: 재실행 → 정상 접근
    else 아직 할당 안 된 (demand paging)
        PF->>Alloc: 새 프레임 할당
        PF->>PT: PTE 설정
        PF-->>CPU: 재실행
    end
```

### Swap 장치

```
물리 메모리 (RAM):    빠름, 용량 작음
Swap 파티션 (SSD):   중간, RAM의 10~100배
Swap 파티션 (HDD):   느림, 거의 사용 안 됨
```

- Swap out: 페이지 내용 → 디스크, PTE Present=0 (swap entry 저장)
- Swap in: 디스크 → 새 프레임, PTE Present=1 복원

---

## 5. 교체 정책 비교

```mermaid
flowchart LR
    subgraph Policies["교체 정책"]
        FIFO["FIFO\n가장 오래된 것 제거\n구현 단순, 성능 나쁨\n(Belady's anomaly)"]
        LRU["LRU\n가장 오래 미사용 제거\n이론적으로 좋음\n정확한 구현 비용 높음"]
        Clock["Clock (Pseudo-LRU)\nLRU 근사, 효율적\n(UNIX 전통적 방법)"]
        Linux2Q["Linux 2-List\nActive + Inactive\n현실적 최적 균형"]
    end
```

### Linux 2-리스트가 Clock보다 나은 점

```mermaid
flowchart TD
    subgraph Workingset["Working Set 보호"]
        WS["자주 쓰이는 페이지 = Active List\n→ 교체에서 보호됨\n→ 캐시 효율 높음"]
    end

    subgraph Scan["Scan-resistant"]
        SR["한 번만 읽는 대용량 파일\n→ Inactive List에만 머뭄\n→ Active List 오염 없음"]
    end

    Linux2Q --> Workingset & Scan
```

---

## 6. Chapter 2 복선: vLLM 블록 교체 전략

```mermaid
flowchart LR
    subgraph OS_Replacement["Linux Page Replacement"]
        L1["Inactive list tail → Swap out"]
        L2["PTE Accessed bit 추적"]
        L3["kswapd 비동기 회수"]
    end

    subgraph vLLM_Replacement["vLLM Block Eviction"]
        V1["ref_cnt=0 블록 → 교체 후보\n(실제로는 즉시 해제 or prefix cache 유지)"]
        V2["prefix caching: 재사용 가능 블록은 보존"]
        V3["preemption: 요청 전체를 중단 (swap/recompute)"]
    end

    L1 -.->|"대응"| V1
    L3 -.->|"대응"| V3
```

| OS 개념 | vLLM 개념 | 차이점 |
|---------|-----------|--------|
| Page swap out | 블록 evict / 요청 preemption | vLLM은 블록 단위 아닌 요청 단위 중단 가능 |
| Inactive list | ref_cnt=0 블록들 | 단순화됨 |
| kswapd | Scheduler preemption 로직 | 백그라운드 아닌 스케줄링 시점 |
| Swap in (page fault) | Recompute or swap-in | GPU recompute가 디스크 I/O보다 빠를 수 있음 |
