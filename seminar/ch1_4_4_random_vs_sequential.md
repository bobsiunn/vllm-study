# 1.4.4 Random vs Sequential 접근: Row Buffer 동작 비교

---

## 1. 핵심 차이

같은 양의 데이터를 읽더라도 **접근 패턴**이 성능을 크게 좌우한다.

```mermaid
flowchart LR
    subgraph Sequential["순차 접근 (Sequential)"]
        S1["Row 0, Col 0"]
        S2["Row 0, Col 1"]
        S3["Row 0, Col 2"]
        S4["Row 0, Col 3"]
        S1 --> S2 --> S3 --> S4
    end

    subgraph Random["랜덤 접근 (Random)"]
        R1["Row 42, Col 7"]
        R2["Row 1023, Col 3"]
        R3["Row 5, Col 511"]
        R4["Row 777, Col 99"]
        R1 --> R2 --> R3 --> R4
    end

    Sequential -->|"Row Buffer Hit 연속"| Fast["빠름\n~15 ns/access"]
    Random -->|"Row Buffer Conflict 연속"| Slow["느림\n~45 ns/access"]
```

---

## 2. 순차 접근: Row Buffer 최대 활용

```mermaid
sequenceDiagram
    participant CPU as CPU
    participant RB as Row Buffer
    participant Array as DRAM Array

    CPU->>Array: ACT Row 100
    Array-->>RB: Row 100 전체 로드 (8 KB)

    CPU->>RB: READ Col 0 → 데이터 반환 (Hit!)
    CPU->>RB: READ Col 1 → 데이터 반환 (Hit!)
    CPU->>RB: READ Col 2 → 데이터 반환 (Hit!)
    CPU->>RB: READ Col 3 → 데이터 반환 (Hit!)

    Note over CPU,Array: Row 100의 8KB 전체를 ACT 1회로 처리
    Note over CPU,Array: 효율적인 Streaming 읽기
```

- ACT 1회로 Row 전체(8 KB) 접근 가능
- 64 byte 캐시 라인 128개를 히트로 처리
- **유효 레이턴시** = (tRCD + tCL) / 128 ≈ 0.23 ns/line

---

## 3. 랜덤 접근: Row Buffer Conflict 반복

```mermaid
sequenceDiagram
    participant CPU as CPU
    participant RB as Row Buffer
    participant Array as DRAM Array

    CPU->>Array: ACT Row 42
    Array-->>RB: Row 42 로드
    CPU->>RB: READ Col 7 → 반환
    CPU->>RB: PRE (다음 접근이 다른 Row)
    RB-->>Array: Row 42 기록

    CPU->>Array: ACT Row 1023
    Array-->>RB: Row 1023 로드
    CPU->>RB: READ Col 3 → 반환
    CPU->>RB: PRE
    RB-->>Array: Row 1023 기록

    CPU->>Array: ACT Row 5
    Array-->>RB: Row 5 로드
    CPU->>RB: READ Col 511 → 반환

    Note over CPU,Array: 매번 ACT + PRE 반복
    Note over CPU,Array: 유효 레이턴시 = tRP + tRCD + tCL = 45 ns/line
```

- 매 접근마다 ACT → PRE 필요
- **유효 레이턴시** = tRP + tRCD + tCL ≈ 45 ns/line
- 순차 대비 **약 3~10배 느림**

---

## 4. 메모리 접근 패턴 성능 비교

```mermaid
block-beta
  columns 3
  A["순차 읽기\n(Row Buffer Hit)\n대역폭: ~50 GB/s\n레이턴시: ~15 ns"]:1
  B["Stride 접근\n(부분 Hit)\n대역폭: ~20 GB/s\n레이턴시: ~30 ns"]:1
  C["완전 랜덤\n(Row Buffer Conflict)\n대역폭: ~5 GB/s\n레이턴시: ~80 ns"]:1

  style A fill:#d4edda,stroke:#28a745
  style B fill:#fff9c4,stroke:#ffc107
  style C fill:#ffcccc,stroke:#dc3545
```

### Stride 접근 패턴

```
Stride 64B  → 캐시 라인 1개씩 건너뜀 → Row Buffer Hit 가능
Stride 4KB  → 다른 페이지 경계 → Row Buffer Hit 가능성 낮음
Stride 8KB  → Row 크기 = 매번 다른 Row → Row Buffer Conflict
```

---

## 5. 실제 예: 행렬 순회 (Row-major vs Column-major)

### C 언어 배열 (Row-major 저장):

```c
double A[1024][1024];  // 8MB, Row 0 = A[0][0..1023] 연속 저장

// 순차 접근 (빠름 ✓)
for (int i = 0; i < 1024; i++)
    for (int j = 0; j < 1024; j++)
        sum += A[i][j];  // A[0][0], A[0][1], ... (Row-major)

// 랜덤 접근 (느림 ✗)
for (int j = 0; j < 1024; j++)
    for (int i = 0; i < 1024; i++)
        sum += A[i][j];  // A[0][0], A[1][0], ... (Column-major)
```

```mermaid
flowchart LR
    subgraph RowMajor["Row-major 순회 (권장)"]
        RM["A[0][0] → A[0][1] → A[0][2] → ...\n같은 Row 연속 접근\nRow Buffer Hit"]
    end

    subgraph ColMajor["Column-major 순회 (비권장)"]
        CM["A[0][0] → A[1][0] → A[2][0] → ...\n다른 Row 교차 접근\nRow Buffer Conflict"]
    end

    RowMajor -->|"5~10배 빠름"| Result["성능 차이"]
    ColMajor --> Result
```

---

## 6. NUMA (Non-Uniform Memory Access)

멀티 소켓 서버에서 메모리 위치도 중요:

```mermaid
flowchart LR
    subgraph Node0["NUMA Node 0"]
        CPU0["CPU 0"]
        MEM0["Local DRAM\n~80 ns"]
    end

    subgraph Node1["NUMA Node 1"]
        CPU1["CPU 1"]
        MEM1["Local DRAM\n~80 ns"]
    end

    CPU0 -->|"Local access\n~80 ns"| MEM0
    CPU0 -->|"Remote access\n~160 ns (2x!)"| MEM1
    CPU1 -->|"Local access\n~80 ns"| MEM1
```

- Remote NUMA 접근: **레이턴시 2배, 대역폭 절반**
- `numactl --localalloc`: 항상 로컬 메모리 사용 강제

---

## 7. Chapter 2 복선: KV Cache 블록 배치 전략

```mermaid
flowchart TD
    subgraph Problem["PagedAttention 접근 패턴"]
        P1["Prefill: 입력 토큰 전체 처리\n→ 순차 블록 할당\n→ 비교적 순차적 접근"]
        P2["Decode: 토큰 1개씩 생성\n→ 모든 이전 블록 접근\n→ 산발적 랜덤 접근"]
    end

    subgraph Optimization["최적화 방향"]
        O1["물리적으로 연속된 블록 우선 할당\n→ HBM Row Buffer Hit 증가"]
        O2["Prefix Caching\n→ 공유 KV 블록은 건드리지 않음\n→ 접근 집중도 감소"]
    end

    Problem --> Optimization
```

- PagedAttention은 메모리 효율 vs 접근 패턴 효율 간 트레이드오프
- vLLM은 가능한 한 연속 블록 할당 시도 (물리 연속성 최대화)
- HBM의 높은 대역폭이 이 트레이드오프를 감수할 수 있게 해줌
