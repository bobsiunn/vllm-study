# 1.5 Page Frame Allocator: 커널은 물리 프레임을 어떻게 할당하는가

---

## 1. 문제 정의

커널이 새로운 물리 프레임을 할당할 때 고려해야 하는 것들:

```mermaid
flowchart TD
    Need["물리 프레임 필요\n(프로세스 메모리, 파일 캐시, 커널 구조체 등)"]

    subgraph Challenges["할당 과제"]
        C1["단편화 방지\n4KB 단위 요청 + 연속 페이지 요청 혼재"]
        C2["속도\n할당/해제가 빠르게 이뤄져야"]
        C3["Zone 구분\nDMA용 / 일반 / 고메모리 영역 분리"]
        C4["병합 가능성\n작은 조각들을 큰 연속 블록으로 합칠 수 있어야"]
    end

    Need --> Challenges
```

---

## 2. Zone 구조

Linux는 물리 메모리를 **Zone**으로 나눈다:

```mermaid
flowchart TD
    subgraph Physical["물리 메모리 (예: 16 GB)"]
        subgraph DMA["ZONE_DMA\n0 ~ 16 MB"]
            D1["DMA 장치 전용\n(ISA, 오래된 하드웨어)"]
        end
        subgraph DMA32["ZONE_DMA32\n16 MB ~ 4 GB"]
            D2["32-bit 주소만 쓰는\nDMA 장치용"]
        end
        subgraph Normal["ZONE_NORMAL\n4 GB ~ (나머지)"]
            D3["일반 용도\n커널 + 유저 프로세스"]
        end
    end

    Alloc["할당 요청\n(GFP 플래그)"] -->|"GFP_DMA"| DMA
    Alloc -->|"GFP_DMA32"| DMA32
    Alloc -->|"GFP_KERNEL (기본)"| Normal
```

- `GFP_KERNEL`: 일반 커널 할당 (슬립 허용)
- `GFP_ATOMIC`: 인터럽트 핸들러 (슬립 불가)
- `GFP_USER`: 유저 공간 메모리

---

## 3. Buddy Allocator

Linux의 핵심 물리 메모리 할당 알고리즘:

### 원리: 이진 분할 (Binary Split)

```mermaid
flowchart TD
    subgraph FreeArea["free_area[] 배열 (Order 0~10)"]
        FA0["order 0: 4KB 블록 리스트"]
        FA1["order 1: 8KB 블록 리스트"]
        FA2["order 2: 16KB 블록 리스트"]
        FA3["order 3: 32KB 블록 리스트"]
        FA10["order 10: 4MB 블록 리스트\n(2^10 × 4KB)"]
    end
```

### 4MB 블록에서 16KB 할당 요청 시 분할 과정

```mermaid
flowchart LR
    O10["Order-10 블록\n4 MB\n(free_area[10]에서 꺼냄)"]

    subgraph Split1["Order-9 분할"]
        S9A["Order-9 블록\n2 MB (buddy A, 남김)"]
        S9B["Order-9 블록\n2 MB (사용)"]
    end

    subgraph Split2["Order-8 분할"]
        S8A["Order-8 블록\n1 MB (buddy B, 남김)"]
        S8B["Order-8 블록\n1 MB (사용)"]
    end

    subgraph Split3["Order-2 분할"]
        S2A["Order-2 블록\n16 KB ✓ (반환)"]
        S2B["Order-2 블록\n16 KB (남김)"]
    end

    O10 -->|"반으로 쪼갬"| Split1
    S9B -->|"반으로 쪼갬"| Split2
    S8B -->|"계속 분할..."| Split3
    S9A -->|"free_area[9]에 추가"| FA9["free_area[9]"]
    S8A -->|"free_area[8]에 추가"| FA8["free_area[8]"]
    S2B -->|"free_area[2]에 추가"| FA2["free_area[2]"]
```

### 해제 시 병합 (Merge): Buddy 찾기

```mermaid
flowchart TD
    Free["Order-2 블록 해제\n(PFN: 100)"]

    Calc["Buddy PFN 계산\nbuddy = PFN XOR (1 << order)\n= 100 XOR 4 = 104"]

    Check{"PFN 104도\nfree_area[2]에 있음?"}

    Merge["두 블록 병합\n→ Order-3 블록 (PFN: 100)"]
    NoMerge["free_area[2]에 추가\n(병합 불가)"]

    CheckUp["PFN 100의 Order-3 buddy\n= 100 XOR 8 = 108\n→ 또 있으면 병합 반복"]

    Free --> Calc --> Check
    Check -->|"Yes"| Merge --> CheckUp
    Check -->|"No"| NoMerge
```

---

## 4. Slab Allocator

Buddy는 4KB 단위가 최소 — 더 작은 커널 구조체는?

```mermaid
flowchart TD
    Buddy["Buddy Allocator\n4KB (order-0) 단위"]

    subgraph Slab["Slab Allocator"]
        SC1["kmem_cache_create('task_struct', sizeof(task_struct))"]
        SC2["4KB 슬라브 하나 = task_struct 여러 개"]
        SC3["할당: O(1) (단순 포인터 반환)"]
        SC4["해제: 슬랩에 반환, Buddy로 돌아가지 않음"]
    end

    Buddy -->|"4KB 페이지 제공"| Slab

    subgraph Caches["주요 Slab 캐시"]
        C1["task_struct (~7 KB)"]
        C2["inode (~600 bytes)"]
        C3["dentry (~200 bytes)"]
        C4["struct page (~64 bytes)"]
    end

    Slab --> Caches
```

- `kmalloc()`: 범용 slab 캐시 (2^n 크기별)
- `kfree()`: slab으로 반환, Buddy로는 페이지가 비워질 때만 반환
- 내부 단편화 최소화 + 캐시 재사용 효과

---

## 5. `alloc_pages()` 흐름

커널이 물리 프레임을 요청하는 핵심 함수:

```mermaid
sequenceDiagram
    participant Caller as 커널 코드
    participant AP as alloc_pages(gfp, order)
    participant Zone as Zone (ZONE_NORMAL)
    participant FA as free_area[order]
    participant WM as Watermark 체크
    participant Reclaim as kswapd (페이지 회수)

    Caller->>AP: alloc_pages(GFP_KERNEL, 2) -- 16KB 요청
    AP->>Zone: Zone 선택
    Zone->>WM: free pages > low watermark?
    alt 여유 충분
        WM-->>Zone: OK
        Zone->>FA: free_area[2] 확인
        alt 해당 order 블록 있음
            FA-->>AP: 블록 반환
        else 없음 → 상위 order 분할
            FA-->>AP: order-3 이상 분할 후 반환
        end
        AP-->>Caller: struct page* 반환
    else 메모리 부족
        WM-->>Zone: Fail
        AP->>Reclaim: kswapd 깨움 (비동기)
        Note over AP,Reclaim: 직접 회수 또는 대기
        AP-->>Caller: NULL (실패) 또는 대기 후 재시도
    end
```

---

## 6. Watermark 시스템

```mermaid
flowchart TD
    subgraph Watermarks["메모리 여유 수준"]
        High["high watermark\n(여유 충분)\n정상 동작"]
        Low["low watermark\n(kswapd 깨움)\n백그라운드 회수 시작"]
        Min["min watermark\n(긴급)\n직접 회수, 새 할당 제한"]
        OOM["OOM\n(완전 고갈)\nOOM killer 작동"]
    end

    High -->|"메모리 소비"| Low
    Low -->|"더 소비"| Min
    Min -->|"더 소비"| OOM

    Low -->|"kswapd 활성화"| Reclaim["페이지 회수\n(비동기)"]
    Min -->|"direct reclaim"| DirectR["직접 회수\n(할당 스레드가 직접)"]
```

---

## 7. Chapter 2 복선: `BlockPool` = Buddy Allocator

```mermaid
flowchart LR
    subgraph Buddy_OS["Linux Buddy Allocator"]
        BA1["free_area[order]\n크기별 free list"]
        BA2["alloc_pages(order)\n할당 요청"]
        BA3["__free_pages()\n반환 + 병합"]
    end

    subgraph BlockPool["vLLM BlockPool"]
        BP1["free_block_queue\n(FreeKVCacheBlockQueue)\nfree list"]
        BP2["allocate(num_blocks)\n블록 할당"]
        BP3["free(block)\n반환 (ref_cnt=0)"]
    end

    BA1 -.->|"1:1 대응"| BP1
    BA2 -.->|"1:1 대응"| BP2
    BA3 -.->|"1:1 대응"| BP3
```

- Buddy: 2^n 크기의 연속 프레임을 free list로 관리
- BlockPool: 고정 크기 블록을 free list로 관리 (더 단순 — buddy 병합 없음)
- 핵심 공통점: **free list에서 꺼내고, 해제 시 리스트로 반환**
