# 1.8 OOM: 메모리가 완전히 부족하면 무슨 일이 일어나는가

---

## 1. OOM 발생 조건

```mermaid
flowchart TD
    Alloc["새 페이지 할당 요청"]

    subgraph Defenses["방어선"]
        W1["high watermark\n(여유 충분) → 즉시 할당"]
        W2["low watermark\n→ kswapd 깨워 비동기 회수"]
        W3["min watermark\n→ direct reclaim (할당 스레드가 직접 회수)"]
        W4["모든 회수 실패\n(swap도 꽉 참, 회수 가능 페이지 없음)"]
    end

    OOM["OOM Killer 호출\nout_of_memory()"]

    Alloc --> W1
    W1 -->|"실패"| W2
    W2 -->|"실패"| W3
    W3 -->|"실패"| W4
    W4 --> OOM

    style OOM fill:#ffcccc,stroke:#dc3545
```

---

## 2. OOM Score 계산

커널은 각 프로세스에 `oom_score`를 계산해 누구를 죽일지 결정:

```mermaid
flowchart TD
    subgraph OOM_Score["oom_score 계산 (/proc/PID/oom_score)"]
        Base["기본 점수\n= 프로세스 RSS (Resident Set Size)\n  / 전체 물리 메모리 × 1000"]
        Adj["oom_score_adj 보정\n(-1000 ~ +1000)\n관리자/프로세스가 설정 가능"]
        Final["최종 oom_score\n= Base + oom_score_adj\n높을수록 먼저 죽음"]
        Base --> Adj --> Final
    end

    subgraph Protected["보호 대상"]
        P1["oom_score_adj = -1000\n→ OOM에서 완전히 보호\n(init, sshd, systemd 등)"]
    end
```

### 점수에 영향을 주는 요소

| 요소 | 영향 |
|------|------|
| 메모리 사용량 (RSS) | 많이 쓸수록 점수 높음 (먼저 죽음) |
| `oom_score_adj` | -1000 (보호) ~ +1000 (우선 타겟) |
| 루트 프로세스 | 소폭 감점 (보호) |
| 자식 프로세스 메모리 | 부모 점수에 포함 가능 |

---

## 3. OOM Killer 동작 흐름

```mermaid
sequenceDiagram
    participant Kernel as 커널 (out_of_memory)
    participant Scanner as 프로세스 스캐너
    participant Victim as 선택된 프로세스
    participant Signal as 시그널

    Kernel->>Scanner: 모든 프로세스 oom_score 계산
    Scanner-->>Kernel: 최고 점수 프로세스 반환

    Kernel->>Signal: SIGKILL 전송 (Victim에게)
    Signal->>Victim: 즉시 종료

    Note over Victim: 프로세스 종료 → 물리 메모리 해제
    Note over Kernel: 메모리 확보 → 원래 할당 재시도

    alt 재시도 성공
        Kernel-->>Kernel: 정상 복귀
    else 여전히 부족
        Kernel->>Scanner: 다시 OOM Killer 실행
    end
```

---

## 4. OOM 판정 흐름 (커널 내부)

```mermaid
flowchart TD
    OOM_Entry["out_of_memory() 호출"]

    Check1{"oom_killer_disabled?\n(전역 플래그)"}
    Panic["커널 패닉\n(서버 재부팅)"]

    Check2{"회수 가능한\n페이지 있음?"}
    Reclaim["한 번 더 회수 시도"]

    Check3{"Notifier\n(외부 핸들러) 처리?"}

    Select["oom_select_bad_process()\n최고 oom_score 프로세스 선택"]

    Kill["oom_kill_process()\nSIGKILL 전송\n+ 커널 로그 출력"]

    OOM_Entry --> Check1
    Check1 -->|"Yes (임베디드 등)"| Panic
    Check1 -->|"No"| Check2
    Check2 -->|"Yes"| Reclaim --> OOM_Entry
    Check2 -->|"No"| Check3
    Check3 -->|"처리됨"| Return["복귀"]
    Check3 -->|"없음"| Select --> Kill

    style Kill fill:#ffcccc
    style Panic fill:#ff9999
```

---

## 5. OOM 로그 해석

실제 OOM killer 발생 시 커널 로그:

```
Out of memory: Kill process 1234 (python3) score 892 or sacrifice child
Killed process 1234 (python3) total-vm:8GB, anon-rss:7.8GB, file-rss:200MB
```

| 필드 | 의미 |
|------|------|
| `score 892` | OOM score (높을수록 먼저 죽음) |
| `total-vm` | 가상 메모리 크기 |
| `anon-rss` | 익명 페이지 (heap, stack) RSS |
| `file-rss` | 파일 매핑 RSS (코드, 라이브러리) |

---

## 6. OOM 방어 전략

```mermaid
flowchart LR
    subgraph Strategies["OOM 방지 전략"]
        S1["oom_score_adj 설정\n중요 프로세스: -500 이하\n$ echo -500 > /proc/PID/oom_score_adj"]
        S2["cgroup memory limit\n컨테이너/서비스별 메모리 상한\n초과 시 해당 cgroup만 OOM"]
        S3["overcommit 설정\nvm.overcommit_memory=2\n→ 물리 메모리 + swap의 N%만 할당 허용"]
        S4["swap 공간 확보\nSSD swap으로 OOM 발생 완충"]
    end
```

---

## 7. Chapter 2 복선: vLLM의 Request Preemption

```mermaid
flowchart LR
    subgraph OS_OOM["Linux OOM Killer"]
        L1["메모리 고갈 시\noom_score 기반 프로세스 선택\nSIGKILL → 메모리 회수"]
        L2["단위: 프로세스 (조악함)"]
        L3["복구: 불가 (죽은 프로세스)"]
    end

    subgraph vLLM_Preempt["vLLM Request Preemption"]
        V1["GPU 메모리 부족 시\nwaiting queue 기반 요청 선택\n블록 회수 후 재큐"]
        V2["단위: 요청 (세밀함)"]
        V3["복구 가능\n(recompute 또는 CPU swap 후 재시작)"]
    end

    L1 -.->|"대응"| V1
    L2 -.->|"개선"| V2
    L3 -.->|"개선"| V3
```

| OS OOM Killer | vLLM Preemption | 차이 |
|---------------|-----------------|------|
| oom_score (메모리 사용량) | 우선순위 (priority) 또는 도착 순서 | vLLM이 더 정밀한 정책 가능 |
| SIGKILL (복구 불가) | preempt + requeue (복구 가능) | vLLM이 안전함 |
| 전체 프로세스 메모리 반환 | 해당 요청의 KV 블록만 반환 | vLLM이 세밀함 |
| 커널 자동 | Scheduler 명시적 제어 | vLLM이 예측 가능 |
