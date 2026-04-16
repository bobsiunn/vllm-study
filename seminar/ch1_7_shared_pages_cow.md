# 1.7 Shared Pages / Copy-on-Write: 메모리를 어떻게 공유하는가

---

## 1. 왜 공유가 필요한가

```mermaid
flowchart TD
    subgraph Without["공유 없는 경우"]
        P1["Process A\n/bin/bash 코드 (1MB)"]
        P2["Process B\n/bin/bash 코드 (1MB)"]
        P3["Process C\n/bin/bash 코드 (1MB)"]
        RAM1["물리 메모리 3MB 사용"]
        P1 & P2 & P3 --> RAM1
    end

    subgraph With["공유 있는 경우"]
        PA["Process A\n/bin/bash 코드"]
        PB["Process B\n/bin/bash 코드"]
        PC["Process C\n/bin/bash 코드"]
        Shared["물리 메모리 1MB\n(공유 프레임)"]
        PA & PB & PC -->|"같은 PFN 참조"| Shared
    end
```

- 100개의 bash 인스턴스: 공유 없이 100MB, 공유 시 ~1MB
- **_mapcount**: 이 프레임을 가리키는 PTE 수 (공유 시 > 1)

---

## 2. Read-Only Shared Pages (코드, 라이브러리)

```mermaid
flowchart LR
    subgraph VA_A["Process A 가상 공간"]
        VA_Code_A["0x400000 (Code)\nPTE: PFN=7, R/O"]
        VA_Lib_A["0x7f... (libc.so)\nPTE: PFN=50, R/O"]
    end

    subgraph VA_B["Process B 가상 공간"]
        VA_Code_B["0x400000 (Code)\nPTE: PFN=7, R/O"]
        VA_Lib_B["0x7f... (libc.so)\nPTE: PFN=50, R/O"]
    end

    subgraph Phys["물리 메모리"]
        F7["PFN 7\n_mapcount=2\n(프로세스 A, B 공유)"]
        F50["PFN 50\n_mapcount=2\nlibc.so 코드"]
    end

    VA_Code_A --> F7
    VA_Code_B --> F7
    VA_Lib_A --> F50
    VA_Lib_B --> F50

    style F7 fill:#ffe08a
    style F50 fill:#ffe08a
```

- 코드 세그먼트 (text): 모든 인스턴스가 동일 PFN 공유
- Shared Library (.so): 한 번만 물리 메모리에 로드
- PTE에 `W=0` (Writable 비트 off) → 쓰기 시도 시 Page Fault

---

## 3. Copy-on-Write (COW)

### fork() 후 COW 설정

```mermaid
sequenceDiagram
    participant Parent as 부모 프로세스
    participant Kernel as 커널
    participant Child as 자식 프로세스

    Parent->>Kernel: fork() 시스템콜
    
    Note over Kernel: 물리 프레임 복사 없이
    Note over Kernel: 페이지 테이블만 복사

    Kernel->>Parent: 부모 PTE: W=0 (Writable → R/O)
    Kernel->>Child: 자식 PTE: 같은 PFN, W=0
    Kernel->>Child: 자식 반환 (pid=0)
    Kernel->>Parent: 부모 반환 (pid=child_pid)

    Note over Parent,Child: 두 프로세스가 같은 프레임 공유 (R/O)

    Parent->>Kernel: 공유 페이지에 쓰기 시도
    Kernel->>Kernel: Page Fault (W=0 위반)
    Kernel->>Kernel: 새 프레임 할당\n내용 복사 (실제 Copy)
    Kernel->>Parent: 부모 PTE: 새 PFN, W=1
    Kernel->>Child: 자식 PTE: 원래 PFN, W=1 복원
    Parent->>Kernel: 쓰기 완료
```

### COW 상태 전이

```mermaid
stateDiagram-v2
    [*] --> Shared_RO: fork() 직후\n부모/자식 같은 PFN\nW=0

    Shared_RO --> Copied: 어느 한쪽이 쓰기 시도\nPage Fault → 새 프레임 복사
    Shared_RO --> Shared_RO: Read 접근\n(복사 없음)

    Copied --> Independent: 복사 완료\n각자 독립 프레임\nW=1 복원
```

---

## 4. Page Cache: 파일 I/O 공유

```mermaid
flowchart TD
    subgraph PageCache["Page Cache (커널 전역)"]
        PC["파일 데이터 캐시\nstruct address_space 기반\n(inode당 하나)"]
    end

    subgraph Processes["여러 프로세스"]
        PA["Process A\nmmap('/data.csv')"]
        PB["Process B\nmmap('/data.csv')"]
        PC2["Process C\nread('/data.csv')"]
    end

    subgraph Disk["디스크"]
        File["data.csv"]
    end

    PA -->|"VMA → Page Cache"| PC
    PB -->|"VMA → Page Cache"| PC
    PC2 -->|"copy_to_user()"| PC
    PC <-->|"필요 시 I/O"| Disk
```

- 같은 파일을 여러 프로세스가 접근해도 물리 프레임은 하나
- `mmap()`: Page Cache 프레임을 직접 VA에 매핑 (복사 없음)
- `read()`: Page Cache → 유저 버퍼 복사 (한 번의 복사)

---

## 5. `struct page`의 공유 추적

```c
struct page {
    atomic_t _mapcount;  // 이 프레임을 가리키는 PTE 수
    atomic_t _refcount;  // 전체 참조 카운트 (PTE + 커널 내부 사용)
    // ...
};
```

```mermaid
flowchart LR
    subgraph Sharing["PFN 7 공유 상황"]
        PTE_A["Process A PTE → PFN 7"]
        PTE_B["Process B PTE → PFN 7"]
        PTE_C["Process C PTE → PFN 7"]
        Kernel_ref["커널 내부 참조 (kmap 등)"]

        Page7["struct page[7]\n_mapcount = 3\n_refcount = 4"]
    end

    PTE_A & PTE_B & PTE_C -->|"_mapcount"| Page7
    PTE_A & PTE_B & PTE_C & Kernel_ref -->|"_refcount"| Page7
```

- `_mapcount = -1`: 공유 없음 (단독 소유)
- `_mapcount >= 0`: 해당 값 + 1개의 PTE가 참조 중
- 교체 불가 조건: `_refcount > 0` (누군가 참조 중)

---

## 6. Chapter 2 복선: Prefix Caching = Shared Pages

```mermaid
flowchart LR
    subgraph OS_Sharing["Linux Shared Pages"]
        OS1["같은 .so 파일\n→ 여러 프로세스가 같은 프레임 공유\n_mapcount 증가"]
        OS2["COW\n쓰기 시에만 복사"]
    end

    subgraph vLLM_Sharing["vLLM Prefix Caching"]
        vLLM1["같은 시스템 프롬프트\n→ 여러 요청이 같은 KV 블록 공유\nref_cnt 증가"]
        vLLM2["공유 블록은 수정 불가\n(KV Cache는 compute 시 생성, 이후 read-only)"]
    end

    OS1 -.->|"1:1 대응"| vLLM1
    OS2 -.->|"개념 대응"| vLLM2
```

| OS 개념 | vLLM 개념 |
|---------|-----------|
| Shared library pages | 공유 Prefix KV 블록 |
| `_mapcount` | `KVCacheBlock.ref_cnt` |
| COW on write | (해당 없음 — KV는 생성 후 R/O) |
| Page Cache | Prefix hash → block 매핑 테이블 |
| `munmap()` → `_mapcount--` | `free()` → `ref_cnt--` → 0이면 해제 가능 |
