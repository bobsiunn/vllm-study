# 1.2 Page와 Page Frame

---

## 1. 핵심 구분: 논리 vs 물리

페이징의 핵심은 **두 개의 다른 세계**를 분리하는 것이다.

```mermaid
flowchart LR
    subgraph Logical["논리 세계 (소프트웨어 시점)"]
        Page["Page\n가상 주소 공간의 고정 크기 단위\n프로세스가 보는 메모리 조각\n크기: 4KB (보통)"]
    end

    subgraph Physical["물리 세계 (하드웨어 시점)"]
        Frame["Page Frame\n물리 RAM의 고정 크기 슬롯\nDRAM 칩 위의 실제 공간\n크기: Page와 동일 (4KB)"]
    end

    subgraph Mapping["매핑 (커널 + MMU)"]
        PT["Page Table\nVPN → PFN"]
    end

    Page -->|"page table을 통해"| PT
    PT -->|"물리 위치 알려줌"| Frame
```

| | Page (논리) | Page Frame (물리) |
|---|---|---|
| **위치** | 가상 주소 공간 | 물리 RAM |
| **번호** | VPN (Virtual Page Number) | PFN (Page Frame Number) |
| **개수** | 가상 공간 크기에 따라 (x86-64: 2^52개) | RAM 크기에 따라 (16GB RAM / 4KB = 4M개) |
| **메타데이터** | PTE (Page Table Entry) | `struct page` |

---

## 2. 주소 비트 분해

4KB page 기준으로 64-bit VA는 다음과 같이 분해된다:

```
64-bit Virtual Address:
┌─────────────────────────────────────────┬────────────┐
│          VPN (Virtual Page Number)      │   Offset   │
│  상위 52 bits (page table 인덱스로 사용) │  하위 12 bits │
│                                         │ (page 내 위치) │
└─────────────────────────────────────────┴────────────┘
                                           ↑
                                   2^12 = 4096 = 4KB
```

```mermaid
block-beta
  columns 12
  B51["bit 51~48\nPGD\n인덱스"]:3
  B47["bit 47~39\nPUD\n인덱스"]:3
  B38["bit 38~30\nPMD\n인덱스"]:3
  B29["bit 29~21\nPTE\n인덱스"]:2
  B20["bit 20~12\nPTI"]:1
  B11["bit 11~0\nOffset\n(4KB 내 위치)"]:1

  style B11 fill:#ffe08a,stroke:#f0a500
```

- **상위 bits (VPN)**: page table 탐색에 사용 (4-level: PGD/PUD/PMD/PTE 인덱스)
- **하위 12 bits (offset)**: page 내의 byte 위치 (0~4095)

---

## 3. `struct page` — 물리 프레임의 메타데이터

Linux 커널은 모든 물리 프레임에 대해 `struct page` 하나를 유지한다.  
(`mem_map[]` 배열에 저장 — 인덱스 = PFN)

```c
// include/linux/mm_types.h (단순화)
struct page {
    unsigned long flags;       // PG_locked, PG_dirty, PG_active, PG_uptodate ...
    atomic_t _refcount;        // 참조 카운트 (0이면 해제 가능)
    atomic_t _mapcount;        // page table entry 수 (몇 개 프로세스가 참조)
    struct list_head lru;      // LRU 리스트 연결 (active/inactive list)
    struct address_space *mapping; // page cache: 어느 파일의 몇 번째 page?
    pgoff_t index;             // 파일 내 offset
    void *virtual;             // 가상 주소 (커널 주소 공간에서)
    /* ... 기타 필드 생략 ... */
};
```

```mermaid
classDiagram
    class struct_page {
        +unsigned long flags
        +atomic_t _refcount
        +atomic_t _mapcount
        +list_head lru
        +address_space* mapping
        +pgoff_t index
        +void* virtual
    }

    note for struct_page "PFN이 인덱스인 mem_map[] 배열의 원소\n실제 데이터를 담지 않음 — 메타데이터만"
```

### 주요 flags 비트

| Flag | 의미 |
|------|------|
| `PG_locked` | 현재 I/O 중 — 다른 접근 차단 |
| `PG_dirty` | 내용이 수정됨 — 디스크에 써야 함 |
| `PG_active` | active LRU 리스트에 있음 |
| `PG_unevictable` | 절대 evict 불가 (mlock 등) |
| `PG_uptodate` | 디스크와 동기화됨 |
| `PG_referenced` | 최근 접근됨 (LRU 결정에 사용) |

---

## 4. `mem_map[]` — 전체 프레임 배열

```
물리 메모리 전체:
┌──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│Frame0│Frame1│Frame2│Frame3│Frame4│Frame5│Frame6│  ...  (물리 DRAM)
└──────┴──────┴──────┴──────┴──────┴──────┴──────┘
   ↑       ↑       ↑
   ↓       ↓       ↓
┌──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│pg[0] │pg[1] │pg[2] │pg[3] │pg[4] │pg[5] │pg[6] │  ...  (mem_map[])
└──────┴──────┴──────┴──────┴──────┴──────┴──────┘
   PFN=0  PFN=1  PFN=2  ...

mem_map[PFN] = 해당 물리 프레임의 struct page
```

**메모리 오버헤드**: `struct page` 하나 ≈ 64 bytes  
→ 16GB RAM = 4M frames → 4M × 64B = **256MB** (약 1.5% 오버헤드)

---

## 5. Page Table Entry (PTE)

각 VPN에 대한 PTE 구조 (x86-64):

```
63        52 51           12 11  9  8  7  6  5  4  3  2  1  0
┌──────────┬───────────────┬────────────────────────────────────┐
│ Reserved │  PFN (40 bits)│ Ignored │G │PS │D │A │PCD│PWT│U │W │P │
└──────────┴───────────────┴────────────────────────────────────┘
```

| 비트 | 이름 | 의미 |
|------|------|------|
| 0 (P) | Present | 1이면 물리 메모리에 있음. 0이면 page fault |
| 1 (W) | Writable | 1이면 쓰기 가능 |
| 2 (U) | User | 1이면 유저 공간 접근 가능 |
| 5 (A) | Accessed | MMU가 접근 시 자동 set (LRU에 활용) |
| 6 (D) | Dirty | MMU가 쓰기 시 자동 set |
| 12~51 | PFN | 물리 프레임 번호 (40 bits → 최대 4PB 물리 메모리) |

---

## 6. Page와 Page Frame의 관계 정리

```mermaid
flowchart TD
    subgraph ProcessA["Process A"]
        VA_A["VA: 0x1000 (Page 1)"]
        VA_A2["VA: 0x5000 (Page 5)"]
    end

    subgraph ProcessB["Process B"]
        VA_B["VA: 0x1000 (Page 1)"]
        VA_B2["VA: 0x3000 (Page 3)"]
    end

    subgraph PageTables["Page Tables (커널 관리)"]
        PTA["PT_A:\nVPN1 → PFN 7\nVPN5 → PFN 3"]
        PTB["PT_B:\nVPN1 → PFN 2\nVPN3 → PFN 7"]
    end

    subgraph PhysMem["물리 메모리 (mem_map[])"]
        F2["PFN 2\nstruct page[2]"]
        F3["PFN 3\nstruct page[3]"]
        F7["PFN 7\nstruct page[7]\n_mapcount=2 ← 두 프로세스 공유"]
    end

    VA_A -->|"MMU 변환"| PTA
    VA_A2 -->|"MMU 변환"| PTA
    VA_B -->|"MMU 변환"| PTB
    VA_B2 -->|"MMU 변환"| PTB

    PTA -->|"VPN1→PFN7"| F7
    PTA -->|"VPN5→PFN3"| F3
    PTB -->|"VPN1→PFN2"| F2
    PTB -->|"VPN3→PFN7"| F7

    style F7 fill:#ffe08a,stroke:#f0a500
```

- **같은 VA라도 프로세스마다 다른 PA** (격리 보장)
- **하나의 PA를 여러 프로세스가 공유 가능** (shared pages, 1.7절)
- `struct page[7]._mapcount = 2` → 두 프로세스가 PFN 7을 참조 중

---

## 7. Chapter 2 복선: `struct page` → `KVCacheBlock`

vLLM은 `struct page`와 정확히 같은 역할의 구조체를 GPU 블록 메타데이터로 사용한다:

```mermaid
flowchart LR
    subgraph OS["Linux 커널"]
        SP["struct page\n- PFN (배열 인덱스)\n- _refcount\n- flags\n- lru (연결 포인터)"]
        MM["mem_map[]\n(전체 프레임 배열)"]
        SP --> MM
    end

    subgraph vLLM["vLLM"]
        KB["KVCacheBlock\n- block_id\n- ref_cnt\n- _block_hash\n- prev/next_free_block"]
        BL["blocks[]\n(BlockPool 내 전체 블록 배열)"]
        KB --> BL
    end

    SP -.->|"1:1 대응"| KB
    MM -.->|"1:1 대응"| BL
```
