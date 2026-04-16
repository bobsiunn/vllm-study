# 1.4.6 VA → 실제 데이터: End-to-End 전체 경로 정리

---

## 1. 전체 경로 한눈에

`x = arr[i]` 한 줄이 실행될 때 일어나는 모든 일:

```mermaid
flowchart TD
    Code["프로세스 코드\nx = arr[i]"]

    subgraph Step1["① VA 발행"]
        VA["가상 주소 VA 계산\narr 기준주소 + i × sizeof(double)"]
    end

    subgraph Step2["② TLB 조회"]
        TLB_Q{"TLB에 VA → PA\n매핑 있음?"}
        TLB_H["TLB Hit\n~1 cycle, ~0.3 ns\nPA 즉시 반환"]
        TLB_M["TLB Miss\n4-Level Page Table Walk\n~수십 cycle, ~10~20 ns"]
    end

    subgraph Step3["③ 캐시 조회"]
        Cache_Q{"PA가 L1/L2/L3\n캐시에 있음?"}
        L1H["L1 Hit: ~1 ns"]
        L2H["L2 Hit: ~5 ns"]
        L3H["L3 Hit: ~20 ns"]
        CacheMiss["Cache Miss\nDRAM 접근 필요"]
    end

    subgraph Step4["④ DRAM 접근"]
        RB_Q{"Row Buffer\n상태?"}
        RBH["Row Buffer Hit\n~15 ns"]
        RBM["Row Buffer Miss\n~45 ns"]
        RBC["Row Buffer Conflict\n~60 ns"]
        BurstRead["64 bytes burst 전송\n캐시 라인 채움"]
    end

    subgraph Step5["⑤ 데이터 반환"]
        Return["CPU 레지스터에 로드\n(64 bytes 중 필요한 8 bytes)"]
    end

    Code --> Step1 --> Step2
    TLB_Q -->|"Hit"| TLB_H --> Step3
    TLB_Q -->|"Miss"| TLB_M --> Step3
    Cache_Q -->|"L1 Hit"| L1H --> Step5
    Cache_Q -->|"L2 Hit"| L2H --> Step5
    Cache_Q -->|"L3 Hit"| L3H --> Step5
    Cache_Q -->|"Miss"| CacheMiss --> Step4
    RB_Q -->|"Hit"| RBH --> BurstRead
    RB_Q -->|"Miss"| RBM --> BurstRead
    RB_Q -->|"Conflict"| RBC --> BurstRead
    BurstRead --> Step5
```

---

## 2. 레이턴시 누적 테이블

| 경우 | TLB | Cache | DRAM | 합계 |
|------|-----|-------|------|------|
| **최선** | Hit (0.3 ns) | L1 Hit (1 ns) | — | **~1 ns** |
| **일반** | Hit (0.3 ns) | L3 Hit (20 ns) | — | **~20 ns** |
| **보통** | Miss (15 ns) | L3 Hit (20 ns) | — | **~35 ns** |
| **최악** | Miss (15 ns) | Miss | RB Conflict (60 ns) | **~90 ns** |

---

## 3. 병목 구간별 최적화 전략

```mermaid
flowchart LR
    subgraph TLB_OPT["TLB Miss 최소화"]
        T1["Huge Page (2MB, 1GB)\nTLB 커버리지 512배 증가"]
        T2["접근 지역성 유지\n같은 page 반복 접근"]
    end

    subgraph Cache_OPT["Cache Miss 최소화"]
        C1["데이터 구조 정렬\nArray of Structs → Struct of Arrays"]
        C2["접근 순서 최적화\nRow-major 순회"]
        C3["캐시 라인 정렬\nalignof(64)"]
    end

    subgraph DRAM_OPT["DRAM 레이턴시 숨기기"]
        D1["Prefetch\nHW 자동 or SW 명시적"]
        D2["Sequential 접근 패턴 유지\nRow Buffer Hit 극대화"]
        D3["Bank 병렬성 활용\n인터리브 주소 설계"]
    end
```

---

## 4. 실제 성능 측정 예 (간단한 benchmark)

### Sequential vs Random 차이 요약

```
arr[0], arr[1], arr[2], ... (Sequential, 16 GB array):
  → TLB Hit Rate: ~99.9%
  → L1/L2 Hit Rate: ~95% (HW Prefetch 동작)
  → 유효 대역폭: ~40 GB/s
  → 시간: ~0.4 ns/element

arr[random()], arr[random()], ... (Random, 16 GB array):
  → TLB Hit Rate: ~60% (huge page 없이)
  → L1/L2/L3 Hit Rate: ~10% (cache working set 초과)
  → 유효 대역폭: ~2 GB/s
  → 시간: ~40 ns/element (100x 느림!)
```

```mermaid
block-metrics
```

```mermaid
block-beta
  columns 2
  A["Sequential\n40 GB/s\n0.4 ns/element\n✓ TLB Hit\n✓ L2/L3 Hit\n✓ Row Buffer Hit"]:1
  B["Random\n2 GB/s\n40 ns/element\n✗ TLB Miss 빈번\n✗ Cache Miss\n✗ Row Buffer Conflict"]:1

  style A fill:#d4edda,stroke:#28a745
  style B fill:#ffcccc,stroke:#dc3545
```

---

## 5. 전체 하드웨어 협력 구조

```mermaid
flowchart TD
    subgraph SW["소프트웨어 (커널 + 프로세스)"]
        Proc["프로세스 (VA 사용)"]
        KernelPT["커널: Page Table 관리"]
    end

    subgraph HW["하드웨어 (자동 동작)"]
        MMU_HW["MMU: VA→PA 변환"]
        TLB_HW["TLB: 변환 캐시"]
        L1_HW["L1 Cache"]
        L2_HW["L2 Cache"]
        L3_HW["L3 Cache"]
        PF_HW["Hardware Prefetcher"]
        MC_HW["Memory Controller"]
        DRAM_HW["DRAM (Channel/Rank/Bank)"]
    end

    Proc -->|"VA 발행"| MMU_HW
    MMU_HW <-->|"캐시"| TLB_HW
    MMU_HW -->|"PA"| L1_HW
    L1_HW <--> L2_HW <--> L3_HW
    L3_HW <-->|"캐시 미스"| MC_HW
    MC_HW <--> DRAM_HW
    PF_HW -->|"선제 로드"| L2_HW
    KernelPT -->|"TLB 설정"| TLB_HW
```

---

## 6. Chapter 2 복선: GPU에서의 동일 경로

| CPU 경로 | GPU 경로 | 비고 |
|----------|----------|------|
| VA → TLB → PA | VA → GPU MMU → PA | GPU도 가상 주소 사용 (CUDA unified memory) |
| PA → L1/L2/L3 | PA → Shared Mem / L1 / L2 | GPU L1: 192KB/SM, L2: 20~80MB |
| PA → DRAM | PA → HBM | 대역폭 50x (3 TB/s vs 50 GB/s) |
| HW Prefetcher | Warp Coalescing | 같은 원리: 연속 접근 = 고효율 |
| Page Table Walk | Block Table Lookup | 1-level (단순) vs 4-level |

- GPU에서 KV Cache 접근은 이 전체 경로의 **GPU 버전**
- 핵심 병목: HBM 대역폭 (충분히 큼) + 비코어레스드 접근 (피해야 함)
