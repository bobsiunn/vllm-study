# 1.4.5 Prefetch: 데이터를 미리 가져오기

---

## 1. 핵심 아이디어

Prefetch = **CPU가 데이터를 요청하기 전에 미리 캐시에 로드해 두기**

```mermaid
flowchart LR
    subgraph NoPrefetch["Prefetch 없음"]
        NP1["CPU: 데이터 요청"]
        NP2["DRAM 접근 (80 ns 대기)"]
        NP3["CPU: 데이터 수신 후 연산"]
        NP1 --> NP2 --> NP3
    end

    subgraph WithPrefetch["Prefetch 있음"]
        WP1["CPU: 곧 필요할 데이터 예측"]
        WP2["Prefetch 요청 (비동기)"]
        WP3["CPU: 현재 데이터 연산 중..."]
        WP4["DRAM → 캐시 로드 완료 (병렬!)"]
        WP5["CPU: 다음 데이터 요청 → 캐시 Hit!"]
        WP1 --> WP2
        WP2 --> WP4
        WP3 --> WP5
        WP4 --> WP5
    end
```

- 연산과 메모리 접근을 **겹침 (overlap)** → 유효 레이턴시 숨김
- 전제 조건: **접근 패턴을 예측할 수 있어야 함**

---

## 2. Hardware Prefetcher

CPU에 내장된 회로가 접근 패턴을 자동 감지:

```mermaid
flowchart TD
    subgraph HW_Prefetch["Hardware Prefetcher (자동)"]
        SP["Stream Prefetcher\n연속 접근 감지\n예: 0x1000, 0x1040, 0x1080...\n→ 0x10C0 자동 prefetch"]
        STP["Stride Prefetcher\n일정 간격 감지\n예: 0, 128, 256, 384...\n→ 512 자동 prefetch"]
        IP["IP-based Prefetcher\n인스트럭션 주소별 패턴 추적\n동일 load 명령의 이전 접근 이력"]
    end

    Detect["패턴 감지"] --> SP & STP & IP
    SP & STP & IP --> Action["L1/L2 Prefetch 요청 발행"]
    Action --> Cache["캐시에 미리 로드"]
```

### Hardware Prefetcher가 잘 동작하는 패턴

| 패턴 | 예시 | 결과 |
|------|------|------|
| 순차 | `arr[0], arr[1], arr[2], ...` | 매우 효과적 |
| 고정 Stride | `arr[0], arr[8], arr[16], ...` | 효과적 |
| 역방향 | `arr[N], arr[N-1], ...` | 효과적 |
| 랜덤 | `arr[hash(i)]` | 무효 (예측 불가) |
| Pointer chasing | `p = p->next` (linked list) | 무효 |

---

## 3. Software Prefetch

프로그래머 또는 컴파일러가 명시적으로 prefetch 명령 삽입:

```c
// x86 intrinsic
#include <immintrin.h>

void process_array(double* arr, int n) {
    for (int i = 0; i < n; i++) {
        // 64개 원소 앞을 미리 prefetch
        _mm_prefetch((char*)&arr[i + 64], _MM_HINT_T0);
        
        // 현재 원소 처리
        result += arr[i] * 2.0;
    }
}
```

```mermaid
sequenceDiagram
    participant CPU as CPU (연산)
    participant PF as Prefetch 엔진
    participant Cache as L1 Cache
    participant DRAM as DRAM

    CPU->>PF: prefetch arr[64] (비동기)
    CPU->>Cache: arr[0] 요청 (Hit!)
    PF->>DRAM: arr[64] 미리 로드 (병렬)
    CPU->>Cache: arr[1] 처리...
    DRAM-->>Cache: arr[64] 캐시에 도착
    CPU->>Cache: arr[64] 요청 (Hit!)
```

### Prefetch 힌트 종류

| 힌트 | 의미 |
|------|------|
| `_MM_HINT_T0` | L1 캐시로 가져옴 (곧 사용) |
| `_MM_HINT_T1` | L2 캐시로 가져옴 |
| `_MM_HINT_T2` | L3 캐시로 가져옴 |
| `_MM_HINT_NTA` | Non-Temporal: 캐시 오염 최소화 (한 번만 사용) |

---

## 4. Non-Temporal Store (Streaming Write)

대용량 데이터를 **캐시를 거치지 않고** 직접 DRAM에 쓰기:

```mermaid
flowchart LR
    subgraph Normal["일반 쓰기 (느림)"]
        N1["Write: 캐시 라인 읽어옴\n(Read-for-Ownership)"]
        N2["캐시에서 수정"]
        N3["나중에 Write-back"]
        N1 --> N2 --> N3
    end

    subgraph NT["Non-Temporal 쓰기 (빠름)"]
        T1["movnt 명령"]
        T2["Write Combining Buffer"]
        T3["직접 DRAM 기록 (캐시 skip)"]
        T1 --> T2 --> T3
    end
```

- 대규모 memcpy, 비디오 프레임 처리, 파일 스트리밍에 유용
- 캐시 오염 없이 DRAM 대역폭 최대화

---

## 5. Prefetch 효과 측정

```mermaid
block-beta
  columns 2
  A["순차 배열 접근\n(HW Prefetch 효과적)\n유효 대역폭: ~45 GB/s\n캐시 미스 거의 없음"]:1
  B["Pointer Chasing\n(HW Prefetch 무효)\n유효 대역폭: ~1 GB/s\n매 접근마다 캐시 미스"]:1

  style A fill:#d4edda
  style B fill:#ffcccc
```

- Linked list 순회: pointer chasing → prefetch 불가 → DRAM 레이턴시 직격
- 이것이 **배열 > 연결리스트** 성능 차이의 근본 원인

---

## 6. Chapter 2 복선: GPU Prefetch와 KV Cache

```mermaid
flowchart LR
    subgraph GPU_Memory["GPU 메모리 접근 패턴"]
        WG["Warp (32 threads)\n동시에 메모리 접근"]
        Coal["Coalesced Access\n32 threads가 연속 주소 접근\n→ 1 transaction (128 bytes)\n→ HBM 효율 최대화"]
        Uncoal["Uncoalesced Access\n32 threads가 산발적 주소 접근\n→ 32 transactions\n→ HBM 대역폭 낭비"]
        WG --> Coal & Uncoal
    end

    subgraph PagedAttn["PagedAttention 최적화"]
        PA1["블록 내 토큰 연속 저장\n→ Warp가 연속 주소 접근\n→ Coalesced access 달성"]
    end

    Coal --> PagedAttn
```

- GPU의 "Coalesced memory access" = CPU의 "sequential prefetch"와 동일한 원리
- vLLM의 KV 블록 크기 (16 tokens)는 Coalesced access를 위해 튜닝된 값
- 블록 내부는 연속 저장 → Warp 단위 접근 시 효율적
