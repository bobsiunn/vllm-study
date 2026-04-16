# 1.4.1 CPU 캐시 계층: L1 / L2 / L3 / DRAM

---

## 1. 왜 캐시가 필요한가

CPU 연산 속도와 DRAM 속도 간의 격차 (Memory Wall):

```mermaid
block-beta
  columns 1
  CPU["CPU 연산\n~0.3 ns (3 GHz, 1 cycle)"]
  L1["L1 Cache (32~64 KB)\n~1 ns / 4 cycles\nSRAM, on-die"]
  L2["L2 Cache (256 KB ~ 1 MB)\n~3~5 ns / 12 cycles\nSRAM, on-die"]
  L3["L3 Cache (8~64 MB)\n~10~20 ns / 40 cycles\nSRAM, on-die (shared)"]
  DRAM["DRAM (8~64 GB)\n~60~100 ns / 200+ cycles\n가장 느림, off-chip"]

  style CPU fill:#c8e6c9
  style L1 fill:#dcedc8
  style L2 fill:#fff9c4
  style L3 fill:#ffe0b2
  style DRAM fill:#ffccbc
```

| 계층 | 용량 | 레이턴시 | 대역폭 | 위치 |
|------|------|----------|--------|------|
| L1 | 32~64 KB | ~1 ns | ~1 TB/s | 코어당 |
| L2 | 256 KB~1 MB | ~5 ns | ~400 GB/s | 코어당 |
| L3 | 8~64 MB | ~20 ns | ~200 GB/s | 소켓 공유 |
| DRAM | 8~256 GB | ~80 ns | ~50 GB/s | 보드 |

---

## 2. 캐시 라인 (Cache Line)

캐시는 **바이트 단위가 아니라 캐시 라인 단위**로 동작한다.

```mermaid
flowchart LR
    subgraph DRAM_Block["DRAM"]
        DB["...  [byte 0~63]  [byte 64~127]  [byte 128~191]  ..."]
    end

    subgraph CacheLine["캐시 라인 (64 bytes)"]
        CL["byte 0 | byte 1 | ... | byte 63"]
    end

    subgraph CPU["CPU"]
        REG["레지스터: 8 bytes"]
    end

    DRAM_Block -->|"64 bytes 통째로 로드"| CacheLine
    CacheLine -->|"필요한 8 bytes만"| CPU
```

- x86-64에서 캐시 라인 크기 = **64 bytes** (고정)
- 1 byte만 필요해도 64 bytes 전체를 DRAM에서 가져옴
- **Spatial locality** 활용: 인접 데이터를 미리 캐싱

---

## 3. 캐시 구조: Set-Associative

캐시는 `Set × Way` 구조로 구성된다.

```mermaid
flowchart TD
    subgraph PA_Breakdown["물리 주소(PA) 분해"]
        direction LR
        TAG["Tag 비트\n[PA 상위]"]
        SET["Set Index 비트\n[PA 중간]"]
        OFF2["Offset 비트\n[PA 하위 6 bits = 64 bytes]"]
    end

    subgraph Cache["8-way Set-Associative Cache"]
        direction TB
        S0["Set 0\nWay0 | Way1 | Way2 | Way3 | Way4 | Way5 | Way6 | Way7"]
        S1["Set 1\nWay0 | Way1 | Way2 | ... | Way7"]
        SN["Set N\n..."]
    end

    SET -->|"어느 set?"| Cache
    TAG -->|"어느 way? (tag 비교)"| Cache
```

### 예: 32KB L1 캐시, 8-way, 64B 캐시 라인

```
총 라인 수 = 32KB / 64B = 512 lines
Set 수 = 512 / 8 = 64 sets
Set Index 비트 수 = log2(64) = 6 bits
Offset 비트 수 = log2(64) = 6 bits
Tag 비트 수 = 48 - 6 - 6 = 36 bits
```

---

## 4. 캐시 히트/미스 흐름

```mermaid
sequenceDiagram
    participant CPU as CPU
    participant L1 as L1 Cache
    participant L2 as L2 Cache
    participant L3 as L3 Cache
    participant DRAM as DRAM

    CPU->>L1: PA로 데이터 요청
    alt L1 Hit (~1 ns)
        L1-->>CPU: 데이터 반환 ✓
    else L1 Miss
        L1->>L2: 요청 전달
        alt L2 Hit (~5 ns)
            L2-->>L1: 캐시 라인 전달
            L1-->>CPU: 데이터 반환 ✓
        else L2 Miss
            L2->>L3: 요청 전달
            alt L3 Hit (~20 ns)
                L3-->>L2: 캐시 라인 전달
                L2-->>L1: 캐시 라인 전달
                L1-->>CPU: 데이터 반환 ✓
            else L3 Miss (~80 ns)
                L3->>DRAM: 캐시 라인(64B) 요청
                DRAM-->>L3: 데이터 반환
                L3-->>L2: 전달
                L2-->>L1: 전달
                L1-->>CPU: 데이터 반환
            end
        end
    end
```

---

## 5. 캐시 교체 정책 (Eviction Policy)

캐시가 가득 찼을 때 어떤 라인을 내보낼지:

```mermaid
flowchart LR
    subgraph Policies["교체 정책"]
        LRU["LRU\n(Least Recently Used)\n가장 오래된 것 제거\n구현 비용 높음"]
        PLRU["Pseudo-LRU\n실제 CPU에서 사용\nLRU 근사, 비용 절감"]
        Random["Random\n무작위 제거\n구현 단순"]
    end
```

- 실제 CPU L1/L2: **Pseudo-LRU** 또는 **Tree-LRU** 사용
- L3: 제조사마다 다름 (Intel: 독자 알고리즘)

---

## 6. False Sharing (캐시 오염)

```mermaid
flowchart TD
    subgraph CL64["캐시 라인 64 bytes"]
        CA["Counter A\n(Core 0 접근)"]
        CB["Counter B\n(Core 1 접근)"]
        note["← 같은 캐시 라인에 위치!"]
    end

    Core0["Core 0\nCounter A 수정"] -->|"캐시 라인 무효화"| CL64
    Core1["Core 1\nCounter B 수정"] -->|"캐시 라인 무효화"| CL64
    CL64 -->|"지속적 무효화 ping-pong"| Problem["성능 저하\n(메모리 접근 수준으로 느려짐)"]
```

- 서로 다른 데이터지만 **같은 캐시 라인**에 있으면 서로 간섭
- 해결: 패딩으로 다른 캐시 라인에 배치 (`alignas(64)`)

---

## 7. Chapter 2 복선: GPU HBM의 캐시 구조

vLLM이 동작하는 GPU의 메모리 계층:

```mermaid
block-beta
  columns 1
  SM["SM (Streaming Multiprocessor) 레지스터\n수 KB, ~1 cycle"]
  Shared["Shared Memory / L1 Cache\n~192 KB per SM, ~20 cycles"]
  L2G["L2 Cache (GPU)\n~20~80 MB, ~200 cycles"]
  HBM["HBM (High Bandwidth Memory)\n24~80 GB, ~400 cycles\n대역폭: ~2~3 TB/s"]

  style SM fill:#c8e6c9
  style Shared fill:#dcedc8
  style L2G fill:#fff9c4
  style HBM fill:#ffccbc
```

- GPU HBM은 CPU DRAM보다 **대역폭이 50배** 높음 (~3 TB/s vs ~50 GB/s)
- KV Cache는 HBM에 저장 → 대역폭이 성능 병목
- KV Cache 접근 패턴 (순차 vs 랜덤)이 처리량에 직접 영향
