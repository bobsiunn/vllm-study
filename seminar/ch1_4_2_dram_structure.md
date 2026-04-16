# 1.4.2 DRAM 물리 구조: Channel / Rank / Bank / Row / Column

---

## 1. 전체 계층 구조

CPU에서 DRAM까지는 여러 계층을 통해 연결된다.

```mermaid
flowchart TD
    CPU["CPU (Memory Controller 내장)"]

    subgraph CH0["Channel 0 (독립 버스)"]
        subgraph DIMM0["DIMM 0 (메모리 모듈)"]
            subgraph R0["Rank 0 (앞면 chip들)"]
                B0["Bank 0"]
                B1["Bank 1"]
                B2["Bank 2"]
                B3["Bank 3"]
                Bdot["... (Bank 7까지)"]
            end
            subgraph R1["Rank 1 (뒷면 chip들)"]
                B4["Bank 0"]
                B5["Bank 1"]
            end
        end
        subgraph DIMM1["DIMM 1"]
            R2["Rank 0, 1"]
        end
    end

    subgraph CH1["Channel 1 (독립 버스)"]
        DIMM2["DIMM 2, 3"]
    end

    CPU -->|"64-bit 버스"| CH0
    CPU -->|"64-bit 버스"| CH1
```

| 계층 | 설명 | 병렬성 |
|------|------|--------|
| **Channel** | 독립적인 메모리 버스 (보통 2~4개) | 완전 병렬 |
| **Rank** | DIMM 위 칩들의 그룹 (앞/뒤) | 번갈아 접근 |
| **Bank** | 칩 내부 독립 배열 (보통 8~16개) | 동시 활성화 가능 |
| **Row** | Bank 내 행 (보통 65536개) | 한 번에 1개 열림 |
| **Column** | Row 내 단위 데이터 | 순서대로 읽기 |

---

## 2. Bank 내부 구조: Row와 Row Buffer

```mermaid
flowchart TD
    subgraph Bank["Bank (하나의 독립 메모리 배열)"]
        direction TB
        subgraph Array["Memory Array (DRAM 셀)"]
            Row0["Row 0: [Col0][Col1][Col2]...[Col1023]  (8KB)"]
            Row1["Row 1: [Col0][Col1][Col2]...[Col1023]"]
            RowN["Row N: ...  (총 65536 rows)"]
        end

        RB["Row Buffer (Sense Amplifier)\n현재 열려있는 Row의 복사본\n크기: 1 Row = 8KB"]
    end

    IO["I/O 버스 → CPU"]

    Array -->|"RAS: Row 전체를 Row Buffer로 복사"| RB
    RB -->|"CAS: 원하는 Column 데이터 읽기"| IO
```

- **Row Buffer**: Bank당 하나. 현재 활성화된 Row의 내용을 담는 고속 버퍼.
- 모든 DRAM 접근은 먼저 Row를 Buffer로 올린 후 Column을 읽는다.

---

## 3. DRAM 명령 시퀀스

```mermaid
sequenceDiagram
    participant MC as Memory Controller
    participant RB as Row Buffer
    participant Array as DRAM Array

    Note over MC,Array: 상황 1: Row Buffer Empty (최초 접근)
    MC->>Array: ACT (Activate) Row N
    Array-->>RB: Row N 전체 복사 (RAS → tRAS 대기)
    MC->>RB: READ Column C
    RB-->>MC: 데이터 반환 (CAS → tCAS 대기)

    Note over MC,Array: 상황 2: Row Buffer Hit (같은 Row 연속 접근)
    MC->>RB: READ Column C+1
    RB-->>MC: 즉시 반환 (ACT 불필요!)

    Note over MC,Array: 상황 3: Row Buffer Conflict (다른 Row 접근)
    MC->>RB: PRE (Precharge) — Row Buffer 비우기
    RB-->>Array: 데이터 기록 (tRP 대기)
    MC->>Array: ACT (Activate) Row M
    Array-->>RB: Row M 복사
    MC->>RB: READ Column
    RB-->>MC: 데이터 반환
```

---

## 4. Row Buffer 상태 전이

```mermaid
stateDiagram-v2
    [*] --> Empty: 초기 상태

    Empty --> Active: ACT (Activate)\ntRAS 대기 (~45ns)
    Active --> Active: READ/WRITE\n(같은 Row — Row Buffer Hit)\n빠름 (~15ns)
    Active --> Empty: PRE (Precharge)\ntRP 대기 (~15ns)
    Empty --> Active: ACT (새 Row)\ntRAS 대기

    note right of Active
        Row Buffer Hit:
        같은 Row에 연속 접근 시
        ACT/PRE 없이 즉시 읽기
    end note
```

---

## 5. 주소 인터리빙 (Address Interleaving)

메모리 컨트롤러는 연속 주소를 어떻게 Bank에 매핑하는지:

```mermaid
flowchart LR
    subgraph PA["물리 주소 비트"]
        COL["Column 비트\n[5:0]"]
        BK["Bank 비트\n[8:6]"]
        ROW["Row 비트\n[33:9]"]
        CH["Channel 비트\n[35:34]"]
    end

    subgraph Layout["연속 주소의 Bank 분산"]
        A0["0x0000 → Bank 0"]
        A1["0x0040 → Bank 1"]
        A2["0x0080 → Bank 2"]
        A3["0x00C0 → Bank 3"]
        A4["0x0100 → Bank 0 (다른 Row)"]
    end

    BK --> Layout
```

- 연속 주소를 다른 Bank로 분산 → **Bank 병렬 접근** 가능
- 64-byte 캐시 라인 경계마다 다른 Bank → 캐시 라인 읽기 중 다음 준비 가능

---

## 6. Chapter 2 복선: HBM의 뱅크 병렬성

```mermaid
flowchart LR
    subgraph DRAM["CPU DRAM (DDR4)"]
        D1["8 Banks per rank\n채널당 대역폭 ~25 GB/s\nRow Buffer: 8 KB"]
    end

    subgraph HBM["GPU HBM3"]
        H1["32 Banks per channel\n채널 수: 16~32개\n총 대역폭: ~3 TB/s\nRow Buffer: 1 KB (더 세밀)"]
    end

    DRAM -->|"60배 대역폭 향상"| HBM
```

- HBM: bank 수가 훨씬 많고, 각 bank의 Row Buffer가 작아 **충돌 확률 감소**
- KV Cache의 비순차 블록 접근도 HBM에서는 상대적으로 덜 불리함
- 그러나 접근 패턴 최적화 (코어레스드 접근)는 여전히 중요
