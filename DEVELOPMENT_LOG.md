# Development Log — Crypto Quant Project (1인 퀀트 펌)
# 개발 로그 — 멀티전략 암호화폐 자동매매 프레임워크

> **목적 (Purpose)**
> KO: 시스템 개발 과정에서 발생한 문제, 의사결정, 해결 방법을 기록.
>
> EN: Documents problems, decisions, and solutions encountered during system development.

> 운영 정책: 본 로그는 변경 의사결정과 학습 내용을 보존합니다. 구체적 모델 파라미터, 임계값, 시드값 등 전략 노출에 직결되는 수치는 의도적으로 제외하고 코드/설정 파일을 단일 출처로 둡니다.
>
> Operational policy: This log preserves change decisions and learnings. Concrete model parameters, thresholds, and seed values that constitute strategic exposure are intentionally omitted — code/config files remain the single source of truth.

---

## 2026-04-25 ~ 2026-04-26: BTC ML 재시도 — 방법론 파이프라인 연결 완료
## 2026-04-25 ~ 2026-04-26: BTC ML Retry — Methodology Pipeline Wiring Complete

### 배경 (Background)

KO:
4월 23~24일 설계·구현을 완료하고 동결해 두었던 BTC 재시도 파이프라인의 연결 작업을 재개. 면밀한 재점검을 거쳐 추가 결함 두 가지를 발견·수정하고, 전체 파이프라인의 end-to-end 통합 검증까지 완료. 이로써 실험 착수의 모든 전제 조건이 충족됨.

EN:
Resumed the wiring work on the BTC retry pipeline that had been frozen after the April 23–24 design and implementation pass. A careful re-inspection surfaced two additional defects; both were resolved, and end-to-end integration verification of the full pipeline was completed. All preconditions for starting the experiment are now met.

---

### 문제 1: 신규 모듈과 실험 실행 경로의 미연결 (Problem 1: New Modules Disconnected from the Experiment Execution Path)

KO:
모든 신규 통계 모듈이 독립 검증을 통과했으나, 실제 실험이 실행될 때 이 모듈들이 호출되지 않는 상태였음. 실험 진입점에 명시적 라우팅 없이 암묵적 기본값에 의존하는 구조가 원인. 이 상태로 실험을 진행하면 새로운 방법론이 아닌 기존 파이프라인이 실행되면서 그 결과를 v3 결과로 오인하게 됨.

EN:
All new statistical modules had passed individual verification, yet when the experiment would run, none of them were actually invoked. The cause was a structure that relied on implicit defaults at the entry point rather than explicit routing. Running the experiment in that state would execute the old pipeline and produce results that could be mistaken for v3 results.

---

### 문제 2: 라벨링과 실행의 변동성 척도 불일치 (Problem 2: Labeling and Execution Using Different Volatility Measures)

KO:
Triple-Barrier 라벨링과 실거래 손절 로직이 각각 서로 다른 변동성 척도를 사용하고 있었음. 라벨링은 수익률의 롤링 표준편차 기반, 실행은 ATR(Average True Range) 기반. v3 의 핵심 원칙 — "라벨링 손절과 실행 손절의 정의를 일치시킨다" — 이 이 불일치로 인해 지켜지지 않고 있었음.

EN:
The Triple-Barrier labeling and the live-execution stop-loss were each using a different volatility measure — labeling used rolling standard deviation of returns while execution used the Average True Range (ATR). The v3 core principle — "align the labeling stop-loss definition with the execution stop-loss definition" — was violated by this mismatch.

---

### 해결 방법 (Solution)

KO:
**1. 실험 진입점에 명시적 라우팅 적용**:
- 설정에서 라벨링 방식을 단일 진실 원천(single source of truth) 으로 두고, 실험 시작 시 해당 값을 읽어 어느 파이프라인으로 진입할지를 명시적으로 결정. 예상 외의 값이 들어오면 즉시 오류로 중단. 암묵적 폴백 없음.
- 신규 파이프라인과 레거시 파이프라인은 코드 공유 없이 완전 분리하여, 한쪽 수정이 다른 쪽에 영향을 주지 않도록 함.

**2. 라벨링 변동성 척도를 실행 방식에 맞춤**:
- Triple-Barrier 라벨링의 장벽 계산에 사용하는 변동성 척도를 ATR 기반으로 전환. 실행 손절 계산과 완전히 동일한 공식을 사용하도록 정렬. 학습 단계와 실행 단계가 이제 같은 정의 위에서 "같은 게임" 을 평가함.

**3. end-to-end 통합 검증 추가**:
- 전체 파이프라인이 합성 데이터를 통과할 때 오류 없이 완료되고, 결과물이 정해진 스키마를 준수하는지 검증하는 통합 검증 항목 추가. 기존 레거시 경로도 동일 조건에서 회귀 검증.

**4. 결과물 불변성 보장**:
- 실험 결과를 파일 시스템 write-once 모드로 기록. 같은 실험을 재실행해도 기존 결과를 덮어쓸 수 없도록 하여 사후 조작 유혹을 구조적으로 차단.

EN:
**1. Explicit routing at the experiment entry point**:
- The labeling method in configuration serves as the single source of truth. At experiment start, its value is read and the pipeline is routed explicitly; an unexpected value causes an immediate abort. No implicit fallbacks.
- The new pipeline and the legacy pipeline share no code, ensuring that a change in one cannot silently affect the other.

**2. Aligned labeling volatility measure with execution**:
- Converted the barrier calculation in Triple-Barrier labeling to use ATR as the volatility measure — identical to the formula used in the live-execution stop-loss. Training and execution now evaluate the same game under the same definition.

**3. End-to-end integration verification**:
- Added integration verification that confirms the full pipeline completes without error on synthetic data and that results conform to the expected schema. Legacy path regression is verified under the same conditions.

**4. Guaranteed result immutability**:
- Experiment results are written in write-once mode. Re-running the same experiment cannot overwrite existing results, structurally closing the temptation of post-hoc modification.

---

### 결과 및 배운 점 (Result & Learnings)

KO:
1. **단일 진실 원천의 중요성**: 파이프라인 분기를 결정하는 기준이 여러 곳에 흩어져 있으면 어느 한 곳을 빠뜨리는 실수가 반드시 발생한다. 하나의 진입점에서 하나의 값으로 결정되는 구조가 인간 오류에 가장 강하다.

2. **방법론 일치는 선언으로 완성되지 않는다**: "라벨링과 실행을 맞춘다" 는 원칙을 문서에 적어 두었더라도, 실제 코드가 다른 척도를 사용하고 있으면 원칙은 공허하다. 설계 의도와 구현 결과를 별도로 검증해야 하는 이유.

3. **통합 검증은 모듈 검증과 별개의 카테고리**: 모든 모듈이 독립적으로 옳더라도, 그 모듈들이 실제로 연결되어 올바른 순서로 실행되는지는 다른 질문이다. 두 종류의 검증을 명시적으로 분리해 두지 않으면 통합 검증은 누락되기 쉬움.

4. **결과 불변성은 과적합 방지 인프라의 일부**: 실험 결과를 덮어쓸 수 있는 구조는 "결과를 보고 파라미터를 조정한 뒤 다시 실행하는" 행동을 기술적으로 허용한다. write-once 는 단순한 파일 관리 관행이 아니라 과적합 방지 프로토콜의 일환.

EN:
1. **Single source of truth matters**: When the criterion for a pipeline branch is scattered across multiple locations, omitting one is inevitable. A structure where a single entry point reads a single value is the most robust against human error.

2. **Methodology alignment is not complete at the declaration stage**: Writing "align labeling with execution" in a plan means nothing if the implementation uses different measures. The design intent and the implementation result require separate verification.

3. **Integration verification is a different category from module verification**: Even if every module is individually correct, whether those modules are actually connected and executed in the right order is a different question. Unless the two verification types are explicitly separated, integration verification will routinely be skipped.

4. **Result immutability is part of anti-overfit infrastructure**: A structure that allows experiment results to be overwritten technically permits "view results, adjust parameters, re-run" behavior. Write-once is not just file hygiene — it is part of the anti-overfit protocol.

---

## 2026-04-25: 페이퍼 트레이더 관찰성 재설계
## 2026-04-25: Paper Trader Observability Redesign

### 배경 (Background)

KO:
4월 24일 진단을 통해 데이터 파이프라인 사일런트 실패가 근본 원인임을 확인. 단순 재시작이 아닌 관찰성(observability) 과 라이브니스(liveness) 전반을 재설계하기로 결정. "같은 방식으로 망가지지 않게" 가 아니라, "어떤 방식으로 망가지든 빠르게 감지하고 자동으로 복구하게" 가 목표.

EN:
April 24 diagnosis confirmed that a silent data pipeline failure was the root cause. Decided to redesign observability and liveness from scratch rather than just restarting. The goal was not "prevent it from breaking the same way" but "detect any failure mode quickly and recover automatically".

---

### 문제 1: 데이터 신선도에 대한 감시 부재 (Problem 1: No Monitoring of Data Freshness)

KO:
데이터 조회가 예외를 던지지 않아도 새 데이터가 들어오지 않는 상황에 대한 감시가 코드 어디에도 없었음. 재연결 로직이 "연속 예외 발생" 에만 반응하도록 설계되어 있어, 예외 없이 망가지는 사일런트 실패는 영원히 보이지 않는 사각지대였음.

EN:
There was no monitoring anywhere for the case where data-fetch calls return normally yet no new bars arrive. The reconnect logic was designed to fire only on "consecutive exceptions" — silent failures that return without raising were permanently in the blind spot.

---

### 문제 2: 라이브니스 신호가 메인 루프와 무관 (Problem 2: Liveness Signal Unrelated to the Main Loop)

KO:
컨테이너의 자동 재시작을 트리거하는 건강 검사(healthcheck) 가, 운영 특성상 수일씩 갱신되지 않는 것이 정상인 객체를 감시 대상으로 삼고 있었음. "메인 루프가 한 바퀴 돌았는가" 가 아니라 "최근에 거래·체크포인트가 있었는가" 를 보고 있던 셈. 거래가 없는 정상적인 침묵 기간에도 건강 검사가 실패 판정을 내리지 않으므로, 실제 루프 정지 시에도 자동 재시작이 작동하지 않음.

EN:
The container healthcheck that should trigger automatic restart was watching an artifact whose mtime was, by normal operation, expected to be stale for days at a time. It was monitoring "did a trade or checkpoint happen recently" rather than "did the main loop complete a cycle". During any normal quiet period the healthcheck reported healthy, so even a genuine loop stall did not trigger the restart.

---

### 해결 방법 (Solution)

KO:
**1. 데이터 신선도 감시 추가**:
- 데이터 타임스탬프의 진전 여부를 단조 시계로 추적. 조회 성공 여부가 아니라 "마지막으로 새 데이터가 들어온 시점" 을 신선도의 정의로 삼음.
- 신선도가 임계를 초과하면 stale 이벤트 발행 및 데이터 소스 강제 재연결 시도. 복구 시 reconnected 이벤트 발행. 기존 예외 기반 트리거는 유지하되 신선도 트리거를 병렬로 추가.

**2. 메인 루프 전용 라이브니스 산출물로 건강 검사 교체**:
- 메인 루프의 매 사이클마다 전용 라이브니스 산출물을 atomic 하게 갱신. 이 산출물은 오직 "루프가 한 바퀴 돌았다" 만을 보장.
- 건강 검사를 이 산출물의 갱신 주기 검사로 전환. 루프가 실제로 멈추면 건강 검사 실패 → 자동 재시작 작동.

**3. 운영 알림 채널 도입**:
- 알림 추상 계층을 설계하여 구체 구현과 NullNotifier 폴백을 분리. 시스템 시작/종료, 데이터 이상, 포지션 진입·청산, 전략 오류 등 운영 이벤트를 실시간으로 송출.
- 알림 전송 실패가 거래 흐름을 차단하지 않도록 외부 I/O 의 모든 예외를 내부 흡수. 페이퍼와 실거래 모드 양쪽에서 코드 변경 없이 동작.

**4. 운영 복구 절차 명문화**:
- 인시던트 발생 시 실행할 단계별 체크리스트 작성. 각 단계에 "기대 결과" 를 명시하여 사람이 패스/페일을 즉시 판정할 수 있도록 함.

EN:
**1. Added data freshness monitoring**:
- A monotonic clock tracks "wall time of the most recent new bar" — freshness is defined by timestamp advancement, not by call success.
- When freshness exceeds the threshold, a stale event is published and the data source is force-reconnected; recovery emits a reconnected event. The existing exception-based trigger is preserved; the freshness trigger runs in parallel.

**2. Replaced healthcheck with a main-loop-dedicated liveness artifact**:
- Every main loop iteration atomically updates a dedicated liveness artifact. This artifact guarantees exactly one thing: "the loop completed a cycle".
- Healthcheck now watches the freshness of that artifact. When the loop genuinely stalls, healthcheck fails and automatic restart fires.

**3. Operational notification channel**:
- Designed a Notifier abstraction layer with a concrete implementation and a NullNotifier fallback. Operational events — system start/stop, data anomalies, position open/close, strategy errors — are published in real time.
- All notification I/O exceptions are swallowed internally so that no notification failure can block trading. Works identically under paper and live modes.

**4. Documented the recovery procedure**:
- Step-by-step incident checklist with expected output at each stage so that a human can immediately decide pass or fail.

---

### 결과 및 배운 점 (Result & Learnings)

KO:
1. **"살아있다" 는 다층적 명제**: 프로세스 생존, 메인 루프 동작, 데이터 흐름은 모두 다른 사건이다. 라이브니스 감시는 가장 외곽층(데이터 흐름)을 봐야 비로소 의미가 있음. 안쪽 층만 감시하면 "살아있는데 아무것도 안 하는" 상태를 영원히 놓친다.

2. **예외 기반 관측의 한계**: 운영 시스템의 많은 실패는 예외를 던지지 않는다. 신선도·진전 같은 양의 지표를 예외와 독립된 별도 채널로 두어야 사각지대를 없앨 수 있음.

3. **라이브니스 신호는 전용 산출물이어야 함**: "가끔 일어나는 것이 정상" 인 이벤트(거래, 체크포인트)를 라이브니스 기준으로 삼으면 안 됨. 라이브니스 산출물은 메인 루프 사이클 주기로 갱신되어야만 신뢰할 수 있음.

4. **알림은 거래와 절연되어야 함**: 외부 알림 채널 장애가 거래 로직에 전파되는 순간, 알림이 시스템 취약점이 된다. 모든 외부 I/O 는 거래 흐름과 완전히 분리된 경로로 동작해야 함.

EN:
1. **"Alive" is a layered claim**: Process alive, main loop running, and data flowing are three separate events. Liveness monitoring is only meaningful when it watches the outermost layer — data flow. Watching only the inner layer misses the "alive but doing nothing" state entirely.

2. **Exception-based observation has hard limits**: Many operational failures do not raise. Positive indicators like freshness and progress must travel through a channel independent from error reporting.

3. **Liveness signals must be dedicated artifacts**: Events that are expected to be infrequent (trades, checkpoints) cannot serve as liveness signals. A liveness artifact must update at the cadence of the main loop to be trustworthy.

4. **Notifications must be isolated from trading**: The moment an external notification failure can propagate into the trading logic, the notification path becomes a system vulnerability. All external I/O must operate on a path fully separated from the trading flow.

---

## 2026-04-24: 페이퍼 트레이더 신호 중단 — 원인 진단
## 2026-04-24: Paper Trader Signal Outage — Root Cause Diagnosis

### 배경 (Background)

KO:
페이퍼 포워드 테스트 가동 수일째, 마지막 거래 이후 신호가 전혀 나오지 않는 상태가 지속됨을 인지. 즉각 재시작 대신 원인을 끝까지 추적하기로 결정.

EN:
Several days into the paper forward test, noticed the system had produced no signals since the last trade. Decided to trace the root cause before restarting.

---

### 문제: Hung 판정에서 사일런트 실패로 가설 수정 (Problem: Revised Hypothesis from "Hung" to Silent Failure)

KO:
외부에서만 관찰했을 때 "신호가 없는 상태" 가 프로세스 정지인지, 모델이 임계 미만이라 침묵 중인지 구분이 불가능함. 초기에는 커널 수준 시스템 신호의 정체(stall) 패턴을 근거로 "프로세스 정지" 로 판단했으나, 이후 애플리케이션 로그를 확인하니 주기적 하트비트가 멈추지 않고 계속 출력되고 있었음. 메인 루프는 살아있었고, 문제는 데이터 파이프라인이 예외를 던지지 않은 채 조용히 실패한 것이었음.

EN:
From the outside, "no signals" was indistinguishable from a hung process versus a model below its threshold. The initial read was "process hung", based on a stall pattern in a kernel-level system signal. Once application logs were checked, a periodic heartbeat was still printing steadily — the main loop was alive. The problem was a silent failure in the data pipeline: returning without raising, but delivering no new data.

---

### 해결 방법 (Solution)

KO:
**1. 진단 우선순위 재정립**:
- 애플리케이션 로그가 1순위. 가장 직접적이고 오해의 소지가 가장 적음. 커널 수준 신호는 보조 증거로만 활용.
- "살아있는데 아무것도 하지 않는" 상태와 "실제로 멈춘" 상태를 구분하려면, 메인 루프와 독립된 외부 관측 신호(하트비트)가 필수.

**2. 가설 즉시 수정**:
- "프로세스 정지" 가설을 하트비트 증거 확인 즉시 "데이터 파이프라인 사일런트 실패" 로 갱신. 초기 판단이 틀렸을 때 빠르게 수정하는 것이 진단의 본질.

**3. 동결 후 재시작 원칙**:
- 재시작이 운영 상태를 덮어쓰므로 원인 분석을 먼저 완료. 외부 연결 상태와 데이터 호출 흔적 부재를 종합하여 사고 시간대 특정 후 재시작.

EN:
**1. Re-prioritised the diagnostic procedure**:
- Application logs first — most direct, fewest wrong turns. Kernel-level signals are supporting evidence only.
- Distinguishing "alive but doing nothing" from "actually stopped" requires an external heartbeat observable that runs independently of the main loop.

**2. Immediate hypothesis update**:
- Revised "process hung" to "silent data pipeline failure" the moment heartbeat evidence appeared. Updating a wrong initial read quickly is what diagnosis is actually about.

**3. Freeze before restart**:
- Restarting overwrites the live system state. Root cause analysis completed first; the incident window was pinned using connection-state and log-trace evidence before restarting.

---

### 결과 및 배운 점 (Result & Learnings)

KO:
1. **로그가 1순위**: 낮은 수준의 시스템 신호는 그럴싸해 보이지만 해석 위험이 큼. 애플리케이션 로그가 가장 적은 오해를 부른다.

2. **가설 수정이 진단의 본질**: "정지" 로 단정했다가 하트비트를 보고 즉시 갱신했음. 진단은 가설 검증이지 가설 방어가 아님. 초기 판단이 틀렸을 때 빠르게 수정하는 능력이 진단 품질을 결정.

3. **재시작 = 증거 소실**: 운영 사고에서 즉각 재시작의 유혹은 강하지만, 원인 모른 채 재시작하면 같은 사고가 재발해도 두 번째에도 같은 진단을 반복하게 된다.

EN:
1. **Logs first**: Low-level system signals look authoritative but are easy to misread. Application logs lead to the fewest wrong conclusions.

2. **Hypothesis revision is diagnosis**: Called it "hung" first, then immediately updated to "silent pipeline failure" when heartbeat evidence appeared. Diagnosis is hypothesis-testing, not hypothesis-defending. The speed of correcting a wrong initial read defines diagnostic quality.

3. **Restart = evidence destroyed**: The urge to restart immediately is strong in every operational incident, but restarting without knowing the cause means the next identical incident gets diagnosed the same wrong way again.

---

## 2026-04-23 ~ 2026-04-24: BTC ML 재시도 — 통계 방법론 설계 및 파이프라인 구축
## 2026-04-23 ~ 2026-04-24: BTC ML Retry — Statistical Methodology Design and Pipeline Build

### 배경 (Background)

KO:
4월 19일 확정한 방향에 따라 새로운 통계 인프라를 설계·구현. Triple-Barrier 라벨링, Deflated Sharpe 평가, 레짐 분리, 블록 부트스트랩을 독립 모듈로 작성하고 각각 단위 검증을 완료. 그러나 면밀히 검토한 결과 두 가지 구조적 결함을 발견하여, 실험 착수를 보류하기로 결정함.

EN:
Following the direction confirmed on April 19, new statistical infrastructure was designed and implemented. Triple-Barrier labeling, Deflated Sharpe evaluation, regime split, and block bootstrap were each built as independent modules and individually verified. However, a careful review surfaced two structural defects, leading to the decision to defer the experiment.

---

### 문제 1: 구현된 모듈이 실험 실행 경로에 연결되지 않음 (Problem 1: Implemented Modules Not Wired into the Experiment Pipeline)

KO:
새로 작성한 핵심 모듈들은 독립 검증을 통과한 상태였으나, 실제 실험 실행 경로에서 이 모듈들이 호출되지 않음을 발견. 실험을 이 상태로 실행하면 새로운 방법론이 적용된 결과가 아닌, 기존 파이프라인의 결과를 v3 결과로 오인하게 될 위험이 있음.

EN:
The newly implemented core modules had each passed their individual verifications, yet an inspection revealed that none of them were invoked in the actual experiment execution path. Running the experiment in that state would mean misreading results produced by the old pipeline as v3 results.

---

### 문제 2: 한 크로스에셋 피처의 독립 정보 부재 (Problem 2: A Cross-Asset Feature with No Independent Information)

KO:
크로스에셋 신호 중 하나가 데이터 수집 단계의 수식 전개 결과, 기존에 이미 포함된 BTC 시장 변수의 단순 스케일링으로 환원됨을 발견. 이름과 의도는 독립적인 정보처럼 보였으나, 수학적으로 신규 정보가 0 인 피처였음. 단위 검증으로는 절대 드러나지 않는 종류의 결함이며, 데이터 수집 로직의 수식을 손으로 따라가야 비로소 보임.

EN:
One of the cross-asset signals collapsed, through the algebra of the data collection step, into a simple rescaling of a BTC market variable already present in the feature set. Its name and intent suggested independent information, but mathematically it contributed zero new signal. The kind of defect unit verification would never surface — visible only by hand-tracing the collection logic's math.

---

### 해결 방법 (Solution)

KO:
**1. 실험 동결 원칙 적용**:
- 모든 신규 모듈이 실험 실행 경로에 명시적으로 연결되고, 전체 파이프라인을 end-to-end 로 통과하는 통합 검증이 완료될 때까지 BTC 실험 착수를 동결. "코드가 존재하지만 호출되지 않는" 상태로 결과를 내는 것 자체가 자기-기만의 입구.

**2. 독립 정보 없는 피처 완전 제거**:
- 수집 로직을 수학적으로 추적하여 신규 정보가 0 임을 확인 후 해당 피처를 파이프라인에서 완전 삭제. 정보량이 없는 피처를 남기는 것은 잡음을 추가하는 것과 같음.

**3. 통합 검증 추가**:
- 모듈 격리 검증만으로는 파이프라인의 연결 여부를 확인할 수 없음. 전체 파이프라인이 신규 모듈을 실제로 통과하는지를 확인하는 통합 검증 항목을 별도 카테고리로 추가.

**4. 실험 실행 경로에 명시적 라우팅 적용**:
- 어떤 설정을 사용하더라도 실험이 어느 파이프라인으로 진입하는지 명시적으로 결정되도록 단일 라우팅 지점을 도입. 암묵적 폴백 구조를 제거하여 의도치 않게 옛 파이프라인으로 실행되는 경우를 차단.

EN:
**1. Experiment freeze principle**:
- The BTC experiment is frozen until every new module is explicitly wired into the execution path and an end-to-end integration verification passes. Producing results while modules exist but go uncalled is the entry point of self-deception.

**2. Remove the zero-information feature**:
- Confirmed via mathematical tracing that the feature contributed no new signal, then deleted it from the pipeline entirely. A feature with no information is equivalent to adding noise.

**3. Add integration verification**:
- Module-level verification cannot confirm that a module is actually invoked in the pipeline. A separate category of integration verification — checking that the full pipeline passes through the new modules — was added.

**4. Explicit routing at the experiment entry point**:
- Introduced a single routing point that explicitly determines which pipeline an experiment enters, regardless of config. Removed implicit fallback structures that could silently send a run down the old path.

---

### 결과 및 배운 점 (Result & Learnings)

KO:
1. **검증 통과 ≠ 파이프라인 동작**: 모듈 격리 검증은 "그 모듈이 스스로 옳게 동작한다" 는 것만 증명. "그 모듈이 실제 실험에서 호출된다" 는 별도의 검증이 필요하며, 두 가지를 같은 것으로 취급하면 사각지대가 생긴다.

2. **데이터 수학 추적의 필요성**: 피처를 추가할 때 이름·의도의 타당성만 보면 부족하다. 수집 단계까지의 수식 전개를 직접 따라가야 비로소 "신규 정보 0" 류의 결함이 드러남. 단위 검증이 통과한다고 데이터 정의가 맞다는 보장이 없음.

3. **암묵적 폴백의 위험**: 설정 실수 하나로 옛 파이프라인이 조용히 실행될 수 있는 구조는 v3 결과를 v2 로 오인하게 만든다. 분기 지점을 명시적으로 두고, 예상 외의 입력에는 즉시 오류로 반응하도록 설계해야 함.

EN:
1. **Verification pass ≠ pipeline works**: Module-level verification only proves that a module operates correctly in isolation. Whether that module is actually called during the experiment requires a separate check. Treating the two as equivalent creates a blind spot.

2. **Necessity of tracing the data math**: Checking the name and intent of a feature is not sufficient. Hand-tracing the algebra through to the collection step is what surfaces defects of the "zero new information" variety. Passing unit verification does not guarantee that the data definition is correct.

3. **Danger of implicit fallbacks**: A structure where a configuration mistake silently routes the experiment down the old pipeline can cause v3 results to be misread as v2. Routing should be explicit, and unexpected inputs should trigger an immediate error rather than a silent default.

---

## 2026-04-19: BTC ML 재시도 방향 전환 및 페이퍼 포워드 테스트 시작
## 2026-04-19: BTC ML Retry Direction Change and Paper Forward Test Launch

### 배경 (Background)

KO:
이전 ML 리서치에서 단일 코인 전략의 안정성 게이트 통과 여부를 시드 분포 기준으로 검증한 결과, SOL만 통과하고 BTC는 탈락. SOL 전략은 클라우드 환경에서 페이퍼 포워드 테스트를 개시하여 미래 실시간 데이터 기반의 검증 단계로 진입. 동시에 BTC 재시도 방향을 확정 — 파라미터 조정이 아닌 방법론 전면 재설계로 접근. 핵심 명제: "과적합은 과거 성과를 낙관적으로 신뢰하는 것이며, 미래 성과 증거가 필수다."

EN:
In the prior ML research phase, only the SOL strategy cleared the stability gate defined by seed-level performance distribution; BTC did not pass. The SOL strategy was brought up as a paper forward test in a cloud environment, entering the future-data validation phase. The BTC retry direction was simultaneously confirmed — not a parameter adjustment but a full methodology redesign. Core proposition: "Overfitting is optimistic trust in past performance; future-performance evidence is non-negotiable."

---

### 문제 1: 과거 데이터 Lockbox 의 구조적 소진 (Problem 1: Past-Data Lockbox Structurally Exhausted)

KO:
이미 보유한 과거 데이터의 최근 구간을 검증에 소진한 상태. 같은 구간을 재사용하면 데이터 오염이 발생하고, 새로운 구간을 수집하려면 수개월의 대기가 불가피. 결국 어떤 경로를 택해도 "과거를 낙관적으로 본" 결정에 의존하게 되는 구조적 함정이 존재함.

EN:
The most recent slice of historical data had already been consumed as a validation holdout. Reusing it would contaminate the result; collecting a fresh slice requires waiting months. Either path leads back to the same structural trap — a decision that relies on optimism about the past.

---

### 문제 2: 학습 라벨과 실거래 청산의 정의 불일치 (Problem 2: Mismatch Between Training Label and Live Exit Definition)

KO:
학습 단계는 고정된 시간 후의 수익 부호를 예측하도록 라벨을 정의. 그러나 실거래에서는 손절 또는 목표가 도달, 시간 초과 중 먼저 일어나는 사건으로 포지션이 마감. 두 정의가 서로 다른 사건을 모델링하고 있어, 학습된 함수와 실제 거래 결과 사이에 구조적 괴리가 존재함.

EN:
Training defined labels by the sign of return after a fixed horizon. In live trading, a position closes on whichever comes first — a stop-loss, a target, or a timeout. The two definitions model different events, creating a structural gap between the learned function and the actual trade outcome.

---

### 해결 방법 (Solution)

KO:
**1. Lockbox 정의 전환**:
- 과거 데이터 lockbox 를 폐기하고, 페이퍼 포워드 테스트(미래 실시간 데이터 흐름) 자체를 최종 OOS 검증으로 재정의. 미래 데이터는 기다림을 강제함으로써 자기-기만 가능성을 구조적으로 줄임.
- 게이트 기준, 판정 기간, 실패 시 fallback 을 실험 착수 전에 명문화하여 사후 조정 유혹을 차단.

**2. Triple-Barrier 라벨링 채택 (Lopez de Prado)**:
- 상단 목표가(TP), 하단 손절가(SL), 수직 시간 초과(timeout) 세 장벽 중 먼저 닿는 사건을 라벨로 정의.
- 실거래 손절 로직과 라벨링 손절 정의가 동일한 변동성 척도를 기준으로 일치하도록 설계. 학습된 함수와 실행 결과가 같은 게임 위에 놓임.

**3. 통계 엄밀성 강화**:
- Deflated Sharpe Ratio (DSR, Bailey & Lopez de Prado 2014) 채택. 다중 탐색 시도 수와 수익률 분포의 비정규성을 함께 보정. FDR 기반 접근 대비 금융 데이터에 더 직접적으로 대응.
- 시드 수를 대폭 늘려 안정성 신뢰구간의 통계적 신뢰도를 확보.
- Walk-forward 방식을 rolling 으로 전환. 폴드별 OOS 길이를 동일하게 맞춰 폴드 간 통계 비교 왜곡을 제거.
- 자기상관을 보존하는 블록 부트스트랩으로 시장 레짐별 신뢰구간 산출.

EN:
**1. Redefining the lockbox**:
- Abandoned the past-data lockbox. The paper forward test — real future data — becomes the final OOS holdout. Future data enforces waiting, structurally removing the room for self-deception.
- Gates, judgment periods, and fallbacks are committed to writing before the experiment begins, blocking post-hoc adjustment.

**2. Triple-Barrier labeling (Lopez de Prado)**:
- Label is set by whichever arrives first among upper target (TP), lower stop-loss (SL), and vertical timeout.
- The labeling SL and the live-execution SL share the same volatility measure, ensuring the learned function is evaluated under the same regime it executes in.

**3. Statistical rigor**:
- Adopted the Deflated Sharpe Ratio (DSR, Bailey & Lopez de Prado 2014), which adjusts jointly for the number of trials tried and for non-normality of returns — a more direct treatment than FDR for financial data.
- Increased the seed count substantially to establish a statistically credible stability confidence interval.
- Switched walk-forward to rolling mode, aligning OOS length across all folds to remove cross-fold comparison distortion.
- Applied circular block bootstrap (Politis & Romano 1992) for per-regime confidence intervals, preserving autocorrelation.

---

### 결과 및 배운 점 (Result & Learnings)

KO:
1. **Lockbox 는 미래에 있다**: 과거 데이터로 OOS 를 재구성하려는 모든 시도는 "과거를 낙관적으로 본" 결정을 어딘가에 포함한다. 페이퍼 포워드 테스트는 기다림 자체를 검증 프로세스의 일부로 만들어 자기-기만의 구조적 입구를 막음.

2. **라벨링은 손익 정의의 일부**: 라벨링 함수와 실행 청산 조건이 서로 다른 사건을 정의하면, 모델은 사실상 "다른 게임" 을 학습한 것이다. 두 정의의 일치는 알파의 정의를 일관되게 유지하는 토대.

3. **사전 커밋의 가치**: 게이트와 fallback 을 결과를 보기 전에 명문화하면, 결과가 경계선에 걸렸을 때 "이번만 예외" 의 유혹이 구조적으로 차단된다. 이 사전 커밋이 없으면 판단자가 흔들리는 순간이 반드시 온다.

EN:
1. **The lockbox lives in the future**: Any attempt to manufacture OOS from past data contains an optimistic-about-the-past decision somewhere. A paper forward test makes waiting itself part of the validation process, closing the structural entry point for self-deception.

2. **Labeling is part of the P&L definition**: If the labeling function and the live exit condition define different events, the model has been trained on a different game. Aligning the two is the foundation for a consistent alpha definition.

3. **Value of precommit**: Writing down gates and fallbacks before results are seen structurally blocks the "just this once" temptation when results land near the boundary. Without the precommit, the decision-maker will waver at exactly that moment.

---

## 2026-03-06 ~ 2026-03-07: AdaptiveTradingSystem 코인 구성 재설계 및 ML/DL 파이프라인 전면 재학습, 대시보드 양 시스템 연결
## 2026-03-06 ~ 2026-03-07: AdaptiveTradingSystem Coin Universe Redesign & Full ML/DL Pipeline Retraining, Dashboard Connected to Both Systems

### 배경 (Background)

KO:
이전 학습에서 이종(異種) 코인을 하나의 모델로 학습시키는 구조적 문제를 인식.
밈코인과 중량급 알트코인을 단일 HMM + TCN + PPO 파이프라인으로 학습하면 서로 다른 시장 구조가 충돌하여
어느 그룹에도 최적화되지 못하는 결과가 반복됨.

AggressiveTradingSystem이 이미 밈코인·경량 알트코인 군(群)을 담당하고 있음을 재확인.
→ 역할 분리 설계로 전환: Adaptive는 **ETH/SOL/ADA/XRP 중량급 알트코인 전용**으로 재정의.

EN:
Identified a structural problem from previous training: mixing heterogeneous coin profiles in a single model.
Training meme coins and heavyweight altcoins together under a single HMM + TCN + PPO pipeline causes conflicting market structures, consistently producing suboptimal results for both groups.

Confirmed that AggressiveTradingSystem already covers the meme coin / light altcoin universe.
→ Transitioned to role separation: Adaptive redefined as **ETH/SOL/ADA/XRP heavyweight altcoins only**.

---

### 문제 1: MAJOR_COINS 배제 방식의 불명확성 (Problem 1: Ambiguity of MAJOR_COINS Exclusion Approach)

KO:
기존 학습 코드는 전체 코인 목록에서 주요 코인을 **배제(exclude)** 하는 방식으로 학습 대상을 간접 지정했음.
- 어떤 코인이 학습되는지 코드만 보고 즉시 파악 불가 → 유지보수성 저하
- data/ 디렉토리에 새 코인이 추가될 때 의도치 않게 학습 대상에 편입될 위험
- 밈코인과 중량급 알트가 혼재하게 된 근본 원인

EN:
The existing training code indirectly specified the coin universe by **excluding** major coins from a full list.
- Impossible to immediately tell from code which coins were being trained — hurts maintainability
- Risk of unintended coin inclusion whenever a new coin is added to the data/ directory
- Root cause of the meme/heavyweight mixing problem

---

### 문제 2: 절대 수치 기반 성과 기준의 과적합 유발 (Problem 2: Absolute Performance Criteria Inducing Metric Overfitting)

KO:
기존 검증 기준(Sharpe > 1.0, WR > 50% 등)은 절대 수치 기반이었음.
특정 시장 국면(강세장/하락장)에 따라 동일 전략도 기준 충족 여부가 달라지고,
기준을 맞추기 위해 파라미터를 조정하면 기준 자체에 과적합(Overfitting to the metric)이 발생함.

EN:
Previous validation criteria (Sharpe > 1.0, WR > 50%, etc.) were absolute-threshold based.
The same strategy can pass or fail purely based on market period (bull/bear), and tuning parameters to meet the numbers produces overfitting to the metric itself.

---

### 해결 방법 (Solution)

KO:
**1. TARGET_COINS 명시적 포함 방식으로 전환** (`train_rl_v2.py`, `train_regime_v2.py`, `train_signal_v2.py`):
- 배제 방식 폐기 → 학습 대상 코인을 코드 상단에 `TARGET_COINS` 집합으로 명시
- 대상: ETH, SOL, ADA, XRP. data/ 내 다른 코인은 자동으로 무시됨

**2. B&H × 평균 레버리지 상대 벤치마크 도입** (`train_rl_v2.py`):
- 기준선 = 동기간 Buy & Hold 수익률 × 평균 레버리지 추정값
- "시장이 나빠서 손실"과 "전략이 나빠서 손실"을 분리하여 평가
- MDD 기준도 B&H MDD 대비 상대값으로 전환

**3. config.py, risk_manager.py 수정**:
- symbols: 7종목 → ETH/SOL/ADA/XRP 4종목
- risk 파라미터: 중량급 알트 특성에 맞게 조정
- RL 모델 로딩 우선순위: adaptive 전용 → altcoin → 기본 순으로 폴백 체인 추가

**4. 대시보드 양 시스템 연결**:
- 기존에 구현해 둔 대시보드 프론트엔드/백엔드에 Aggressive(Bybit), Adaptive(Binance) 두 시스템을 동시에 연결
- 각 시스템 포지션·누적 손익·레짐 분류 결과를 독립 패널로 확인 가능

EN:
**1. Switched to explicit TARGET_COINS inclusion** (`train_rl_v2.py`, `train_regime_v2.py`, `train_signal_v2.py`):
- Dropped exclusion approach → explicitly declared target coins as a `TARGET_COINS` set at each script's top
- Targets: ETH, SOL, ADA, XRP. All other coins in data/ are automatically ignored

**2. Introduced B&H × average leverage relative benchmark** (`train_rl_v2.py`):
- Baseline = same-period Buy & Hold return × average leverage estimate
- Separates "market was bad" from "strategy was bad"
- MDD benchmark similarly converted to relative vs. B&H MDD

**3. config.py, risk_manager.py updates**:
- symbols: 7 coins → ETH/SOL/ADA/XRP (4 coins)
- risk parameters: adjusted for heavyweight altcoin characteristics
- RL model loading: fallback chain — adaptive-specific → altcoin → default

**4. Dashboard connected to both systems**:
- Connected Aggressive (Bybit) and Adaptive (Binance) to the existing dashboard frontend/backend
- Independent panels for positions, cumulative P&L, and regime classification per system

---

### 재학습 결과 (Retraining Results)

KO:
3단계 파이프라인 순차 재학습 완료 (regime → signal → RL).
TCN 검증 정확도 평균 41.7% (기준선 33.3%, 합격 기준 38% 초과 ✅).
RL 검증: ADA·ETH·XRP Sharpe 양수·PF 1.0 초과, SOL Sharpe 음수·PF 1.0 미만 (모니터링 필요).
전체 평균 Sharpe 양수, B&H×레버리지 기준선 대비 전략 누적 수익 초과 ✅.

EN:
3-stage pipeline retrained sequentially (regime → signal → RL).
TCN validation accuracy avg 41.7% (baseline 33.3%, pass threshold >38% ✅).
RL validation: ADA/ETH/XRP positive Sharpe, PF > 1.0. SOL negative Sharpe, PF < 1.0 (requires monitoring).
Overall average Sharpe positive; strategy cumulative return exceeded B&H × leverage baseline ✅.

---

### 결과 및 배운 점 (Result & Learnings)

KO:
1. **코인 동질성(Homogeneity)이 ML 성능의 핵심 변수**: 코인 선정 기준 변경만으로 RL 에이전트 평균 Sharpe가 음수에서 양수로 전환됨. 모델 구조·하이퍼파라미터보다 학습 데이터 동질성이 더 강한 영향을 미칠 수 있음.

2. **배제 방식 vs 포함 방식**: 포함 대상을 명시하는 것이 배제 대상을 명시하는 것보다 유지보수성과 의도 명확성 모두에서 우월함. 데이터가 동적으로 추가되는 환경에서 배제 방식은 의도치 않은 학습 대상 포함 위험을 항상 내포함.

3. **메트릭 과적합(Metric Overfitting)**: 절대 기준을 목표로 파라미터를 조정하면 기준은 충족하지만 실전 성과와 무관해질 수 있음. 상대 기준(B&H × 레버리지)은 시장 국면에 무관하게 공정한 비교를 제공함.

EN:
1. **Coin homogeneity is a primary ML performance variable**: Changing coin selection alone shifted average RL Sharpe from negative to positive. Data homogeneity can outweigh architecture and hyperparameter choices in impact.

2. **Exclusion vs. inclusion approach**: Explicitly declaring what to include is superior in both maintainability and clarity. In environments where data grows dynamically, exclusion-based logic always carries unintended inclusion risk.

3. **Metric overfitting**: Tuning parameters to meet absolute criteria can satisfy the metric while becoming disconnected from live performance. Relative benchmarks (B&H × leverage) provide market-phase-agnostic comparisons that are harder to game.

---

## 2026-03-04: AggressiveTradingSystem 손실 원인 분석 및 Contrarian(역방향) 모드 전환
## 2026-03-04: AggressiveTradingSystem Loss Root Cause Analysis & Contrarian Mode Pivot

### 배경 (Background)

KO:
13일간 실거래 운영 후 유의미한 손실 발생. 봇을 정지하고 102건 전체 거래 로그를 구조적으로 분석.
단순 파라미터 조정이 아닌 **전략 구조 자체의 근본 원인** 파악을 목표로 함.

EN:
After 13 days of live operation, the system incurred significant drawdown. Bot was halted and all 102 completed trades were systematically analyzed.
The goal was to identify the **structural root cause** — not just tune parameters.

---

### 문제 1: K-Means 레짐 분류기의 후행성 (Problem 1: K-Means Regime Classifier Lag)

KO:
**현상**: AggressiveTradingSystem은 설계 의도상 **역추세(Contrarian)** 전략이었으나,
실질적으로는 **후행 추세 추종(Lagging Trend-Following)** 전략으로 동작하고 있었음.

**원인 분석**:
- K-Means 레짐 분류기가 사용하는 특성(ADX, EMA 정렬 등)은 본질적으로 **후행 지표(Lagging Indicators)**
- `STRONG_DOWNTREND` 레짐 인식 시점 = 하락이 이미 성숙한 시점 → 반등이 임박한 시점에 SHORT 진입
- `STRONG_UPTREND` 레짐 인식 시점 = 상승이 이미 성숙한 시점 → 조정이 임박한 시점에 LONG 진입
- 결과: **방향이 구조적으로 틀리는 패턴** 확인 (손절 거래의 80% 이상에서 진입 직후 반대 방향으로 가격 이동)

EN:
**Symptom**: AggressiveTradingSystem was designed as a **Contrarian** strategy, but was operating in practice as a **Lagging Trend-Following** strategy.

**Root cause analysis**:
- Features used by K-Means classifier (ADX, EMA alignment, etc.) are inherently **lagging indicators**
- When `STRONG_DOWNTREND` is recognized = the decline is already mature → SHORT entered just as a bounce is imminent
- When `STRONG_UPTREND` is recognized = the rally is already mature → LONG entered just as a pullback is imminent
- Result: **Systematically wrong direction** confirmed (over 80% of stop-loss trades moved against entry immediately)

---

### 문제 2: 고정 익절 미체결 (Problem 2: Fixed TP Never Triggered)

KO:
**현상**: 102건 전체 거래에서 고정 익절(TP) 체결 건수 = 0건.
전체 수익 거래는 100% 트레일링 스탑 이탈로 청산.

**분석**:
- 설정된 R:R 배수에 도달하기 위해 필요한 가격 이동 거리 자체가 과도함
- 방향이 맞는 경우에도 가격이 TP 라인까지 도달하기 전에 역전되는 패턴 반복
- "이름만 수익 거래"(트레일링 스탑 이탈)와 "진짜 수익 거래"(TP 체결)를 같은 기준으로 보면 성과 과대평가

EN:
**Symptom**: Across all 102 completed trades, fixed TP fill count = 0.
Every winning trade was closed via trailing stop escape — none reached the fixed TP target.

**Analysis**:
- The required price move to reach the set R:R target was excessive for the system's average hold duration
- Even when direction was correct, price reversed before reaching the TP line
- Conflating "trailing stop escapes" (name-only wins) with "fixed TP fills" (real wins) overstates performance

---

### 문제 3: 레짐별 손익 편차 및 밴드에이드 함정 (Problem 3: Regime PnL Bias & Band-Aid Trap)

KO:
특정 레짐(예: `STRONG_DOWNTREND`)의 성과가 극히 나쁜 것을 발견.
초기 반응은 "이 레짐만 진입 제한" 이었으나, 이는 **밴드에이드 해법**임을 인식:

- 하락장에서 `STRONG_DOWNTREND` 후행 진입이 문제 → 제한하면 단기 개선
- 상승장이 오면 `STRONG_UPTREND` 후행 진입이 동일한 문제를 일으킴
- 특정 레짐 차단은 근본 원인(후행성)을 해결하지 못하고 시장 국면별로 문제가 반복됨

**결론**: 레짐 필터링이 아닌 **신호 방향 자체의 구조적 수정**이 필요.

EN:
Specific regimes (e.g., `STRONG_DOWNTREND`) showed extremely poor performance.
Initial reaction was "restrict entry in this regime" — but this was identified as a **band-aid fix**:

- In a bear market, lagging `STRONG_DOWNTREND` entries cause losses → blocking it helps short-term
- In a bull market, lagging `STRONG_UPTREND` entries will produce the same problem
- Blocking specific regimes does not solve the root cause (lag); the problem recurs as market conditions change

**Conclusion**: The fix must target the **structural direction bias**, not individual regime filters.

---

### 해결 방법: Contrarian 모드 도입 (Solution: Contrarian Mode Implementation)

KO:
K-Means 분류기의 후행성을 **결함이 아닌 신호로 재해석**:
레짐 분류 시점 = 해당 추세가 성숙한 시점 = 반전 임박 시점
→ 분류기가 생성하는 방향을 반전시키면 역추세 진입이 됨

**구현 방식** (3개 파일 수정):
1. `signal_generator.py`: 최종 신호 출력 시 `LONG↔SHORT` 반전 로직 추가 (`reverse_signals` 플래그)
2. `config.py`: `reverse_signals = True` 설정, 검증 기간 레버리지 최소화
3. `main.py`: 시스템 시작 시 `[CONTRARIAN MODE]` 로그 출력, 각 신호에 `[REVERSED]` 태그 추가

**의도적으로 변경하지 않은 것**:
- SL 거리 (변경 시 효과 측정이 오염됨)
- TP 배수 (독립 변수 유지)
- 진입 임계값 (신호 방향만 검증)

EN:
Reframed the K-Means classifier's lag as **a signal, not a defect**:
Regime classification time = the moment that trend is mature = reversal is imminent
→ Reversing the classifier's output direction produces counter-trend entries

**Implementation** (3 files modified):
1. `signal_generator.py`: Added `LONG↔SHORT` flip logic at final signal output (`reverse_signals` flag)
2. `config.py`: Set `reverse_signals = True`, minimized leverage for verification period
3. `main.py`: Added `[CONTRARIAN MODE]` log at system start, `[REVERSED]` tag on each signal

**Intentionally unchanged**:
- SL distance (changing it would contaminate the effect measurement)
- TP multiplier (kept as independent variable)
- Entry threshold (only entry direction is being validated)

---

### 검증: 6개월 백테스트 (Validation: 6-Month Backtest)

KO:
Binance Futures 5분봉 7종목 6개월치 데이터를 다운로드, 동일 파라미터에서 원래 방향 vs 반전 방향 비교.
서버에서 운영 중인 K-Means 모델을 로컬에 복사하여 동일한 레짐 분류 조건 적용.

**결과**:
- 원래 방향: 7종목 중 다수에서 수익 저조 또는 손실
- 반전 방향: 7종목 중 6종목에서 개선 확인
  - 원래 방향에서 손실을 기록하던 종목들이 반전 후 수익으로 전환
  - 승률 상승 (원래 방향 평균 ~41% → 반전 방향 평균 ~49%)
  - Profit Factor: 원래 방향에서 1.0 미만이던 종목들이 반전 후 전부 1.0 초과
  - MDD: 대부분 종목에서 개선
- 1개 종목(DOGE)에서만 원래 방향이 우세 — 해당 기간 특수성으로 판단

**백테스트 한계**:
- 슬리피지, 체결 지연, 펀딩비 미반영
- 과거 6개월 데이터에 대한 사후 검증 — 미래 성과를 보장하지 않음
- 소액 실전 배포 후 2~4주 추가 검증 필요

EN:
Downloaded 6 months of 5-minute data for all 7 symbols from Binance Futures.
Compared original direction vs. reversed direction under identical parameters.
Used the actual K-Means model running on the server to ensure consistent regime classification.

**Results**:
- Original direction: majority of symbols showed poor or negative performance
- Reversed direction: improvement confirmed in 6 of 7 symbols
  - Symbols that were losing in original direction turned profitable after reversal
  - Win rate improvement (original ~41% avg → reversed ~49% avg)
  - Profit Factor: all symbols previously below 1.0 crossed above 1.0 after reversal
  - MDD: improved in most symbols
- 1 symbol (DOGE) remained better in original direction — attributed to period-specific price behavior

**Backtest limitations**:
- Does not account for slippage, fill delays, or funding fees
- Retrospective validation on past data — does not guarantee future performance
- Additional live verification required over 2–4 weeks post-deployment

---

### 결과 및 배운 점 (Result & Learnings)

KO:
1. **후행 지표는 양날의 검**: 레짐 분류기가 후행이라는 것은 약점이 아닐 수 있음.
   방향을 반전시키면 오히려 반전 시점 포착에 유리한 구조가 됨.
   핵심은 "이 도구가 무엇을 잘 하는가"를 파악하는 것.

2. **밴드에이드 vs 근본 수정 구분**: 특정 레짐 차단은 데이터에 과적합된 해법.
   시장 국면이 바뀌면 다른 레짐에서 동일 문제가 반복됨.
   진짜 수정은 문제의 발생 원리를 바꾸는 것이어야 함.

3. **독립 변수 통제**: 방향만 바꾸고 SL/TP/임계값을 유지함으로써
   "방향 반전의 효과"를 다른 변수와 혼재 없이 측정 가능.
   검증 설계에서 독립 변수 통제는 필수.

4. **Exit 유형 분류의 중요성**: 트레일링 스탑 이탈과 고정 TP 체결을 같은 "수익 거래"로 집계하면
   시스템의 실제 성과가 왜곡됨. 청산 유형별 별도 분석이 진단 정확도를 크게 높임.

EN:
1. **Lagging indicators are a double-edged sword**: A lagging regime classifier is not necessarily a weakness.
   Reversing its direction output turns it into a structure that's well-positioned to capture trend reversals.
   The key insight is understanding *what the tool actually does well*.

2. **Band-aid vs. root fix**: Blocking specific regimes is a data-overfit solution.
   When market conditions change, the same problem re-emerges in a different regime.
   A real fix changes the mechanism that causes the problem.

3. **Isolate the independent variable**: By changing only direction while holding SL/TP/thresholds constant,
   the effect of direction reversal can be measured cleanly without confounding variables.
   Controlling independent variables is essential in any system validation design.

4. **Exit type classification matters**: Grouping trailing stop escapes and fixed TP fills under the same "winning trade" label
   distorts the system's true performance picture. Analyzing by exit type significantly improves diagnostic accuracy.

---

## 2026-02-26: 머신러닝 기반 HFT(고빈도 매매) 추론 엔진 및 C++ 실행단 통합 파이프라인 완성
## 2026-02-26: Machine Learning-Based HFT Inference Engine & C++ Execution Pipeline Integration Complete

### 배경 (Background)

KO:
C++을 활용한 초저지연(Ultra-Low Latency) 주문 집행 엔진 아키텍처를 앞서 구축하였으나, 
이를 구동할 "두뇌" 역할을 하는 머신러닝 추론 파이프라인의 실시간 연동이 필요했다.
단순히 호가창(Orderbook)을 분석하는 것을 넘어, 100ms 단위의 미시적 틱(Tick) 및 체결 데이터를 수집하여 모델을 학습시키고, 
이를 기반으로 0.1초 이내에 시장 상황을 판단해 C++ 엔진에 동적 비중(Dynamic Sizing)과 타점을 전송하는 완전히 자율화된 HFT 사이클 구조를 완성하고자 하였다.

EN:
Having previously built an ultra-low latency execution engine in C++, the system needed its "brain" — a real-time machine learning inference pipeline. 
The overall goal was not just basic orderbook analysis, but to collect micro-tick data at 100ms intervals, train a predictive model, 
and deploy it to evaluate market micro-structure in under 0.1 seconds. 
This inference module would then transmit confidence-adjusted dynamic sizing and entry targets directly to the C++ engine, completing a fully autonomous HFT cycle.

---

### 문제: 라이브 틱 데이터 부재 및 머신러닝 추론 병목 (Problem: Lack of Live Tick Data & ML Inference Bottlenecks)

KO:
1. **고빈도 데이터 수집의 한계**: 거래소의 일반적인 REST API로는 과거의 깊은 호가창(Depth) 데이터를 한 번에 확보할 수 없었다.
2. **복잡한 뉴럴넷 모델의 지연 속도**: 기존에 사용하던 딥러닝(PyTorch 등) 모델은 추론(Inference)에 수십 밀리초 이상이 소요되어 나노초 단위 경쟁이 필수적인 HFT 환경에서는 병목이 발생했다.

EN:
1. **Limits of High-Frequency Data Collection**: Standard REST APIs do not provide historical deep orderbook data (Depth) essential for micro-structure analysis.
2. **Latency of Complex Neural Networks**: The existing deep learning models (PyTorch-based) required tens of milliseconds for inference, creating an unacceptable bottleneck in an HFT environment where nanosecond-level competition is critical.

---

### 해결 방법: HFT 특화 데이터 파이프라인 및 XGBoost 아키텍처 도입 (Solution: HFT-Specific Data Pipeline & XGBoost Architecture)

KO:
1. **웹소켓 로거 및 아카이브 데이터 병합 채택**: 
   - 실시간 체결(`@aggTrade`) 및 5단계 호가(`@depth5`) 데이터를 100ms 주기로 로깅하는 전용 콜렉터 데몬 구축.
   - 단기적인 데이터 부족 문제를 해소하기 위해 과거 바이낸스 일일 틱 아카이브 데이터를 병렬로 비동기 다운로드 및 압축 해제하여 대규모 초기 훈련 데이터셋으로 병합하는 구조 추가 구현.
2. **초고속 결정 트리(XGBoost) 도입 및 과적합 방지**:
   - 무거운 뉴럴넷 대신 밀리초 단위 추론이 가능한 XGBoost 채택.
   - 타겟 레이블(Label)을 단순한 가격 방향이 아닌, 슬리피지와 수수료를 극복 가능한 '틱(Tick) 단위 목표 수익 초과 달성 여부'로 엄격히 재설계.
   - 금융 데이터의 노이즈 처리 및 과적합(Overfitting) 방지를 위해 트리의 깊이(Depth)를 얕게 제한하고 파라미터 튜닝 적용.
3. **분리형 아키텍처(Brain-Brawn Separation) 강화**:
   - Python(두뇌)은 ML 추론과 자금 관리(수량 동적 조절)만 담당하고, 모델이 산출한 예측 확신도(Confidence)에 비례해 동적 수량(Dynamic Quantity)을 명시한 명령을 Shared Memory(IPC)로 C++(심장)에 전송.

EN:
1. **Websocket Logger & Archive Data Merging**:
   - Built a dedicated collector daemon logging real-time aggregated trades (`@aggTrade`) and optimal depth (`@depth5`) at 100ms intervals.
   - To overcome immediate data sparsity, implemented an asynchronous downloader to fetch and merge historical daily tick archives as a bootstrap training set.
2. **High-Speed Decision Trees (XGBoost) & Overfitting Prevention**:
   - Replaced heavy neural nets with XGBoost for sub-millisecond inference execution.
   - Redefined the target label away from simple price direction towards a strict 'Tick-level excess return' threshold to account for slippage and transaction fees.
   - Restrained tree depth and applied strict parameter tuning to combat noise and prevent overfitting in non-stationary financial data.
3. **Brain-Brawn Separation Architecture**:
   - Established an asynchronous hybrid pipeline where Python handles ML inference and dynamic position sizing, securely passing signal confidence and dynamic order quantity via Shared Memory (IPC) to the C++ core for ultra-fast execution.

---

### 결과 및 배운 점 (Result & Learnings)

KO:
1. **완전한 HFT 사이클 가동**: 수집 $\rightarrow$ 학습 $\rightarrow$ 실시간 추론 $\rightarrow$ IPC 통신 $\rightarrow$ C++ 트레일링 실행으로 이어지는 매매 사이클이 성공적으로 통합 가동됨.
2. **시스템 무결성과 폴백(Fallback)**: 학습된 객체 파일(.json)이 없을 시 기존 휴리스틱 룰 베이스로 유연하게(Graceful degradation) 동작하는 폴백 모드를 덧붙여 운영 리스크를 완화함.
3. **학습 데이터 퀄리티의 한계 돌파**: 결국 극초단타 영역에서의 엣지(Edge)는 화려한 전략 알고리즘보다, 얼마나 깊고 넓은 **미시적 거래 데이터(Micro-structure data)를 가공해 낼 수 있는 인프라 엔지니어링** 역량에 압도적으로 의존한다는 결론을 재확인.

EN:
1. **Full HFT Lifecycle Operable**: Successfully united the entire pipeline: Collection $\rightarrow$ Training $\rightarrow$ Real-time Inference $\rightarrow$ IPC $\rightarrow$ C++ Trailing Execution.
2. **Loose Coupling & Fallback Integrity**: Secured operational integrity by engineering a fallback mechanism that automatically reverts to a heuristic rule-base if the ML model file is unavailable.
3. **Supremacy of Data Engineering**: Reaffirmed that within robust HFT architectures, the competitive edge relies almost entirely on the engineering capability to harvest and preprocess massive micro-structure data, validating that infrastructure outranks algorithmic complexity.

---

## 2026-02-23: C++ 고빈도 매매 엔진(HFT Engine) 설계 및 구현
## 2026-02-23: C++ High-Frequency Trading Engine — Design & Implementation

### 배경 (Background)

KO:
파이썬 기반 트레이딩 봇은 신호 생성과 학습에는 적합하지만, 실제 주문 집행 단계에서
GIL(Global Interpreter Lock)과 네트워크 레이턴시가 **수백 밀리초(ms) 단위의 지연**을 만들어
빠른 시장 변동에 대응하지 못하는 구조적 한계가 있음.

특히 **포지션이 잡힌 이후에도** ML/DL 신호를 실시간으로 수신하여
주문을 넣었다 뺐다 하며 트레일링 스탑처럼 **큰 이익을 보게**하는 시스템을 목표로,
**주문 집행 전용 C++20 HFT 엔진**을 밑바닥부터 설계하고 구현.

EN:
Python-based trading bots are suited for signal generation and ML training, but at the order execution layer,
GIL (Global Interpreter Lock) and network overhead introduce **hundreds of milliseconds of latency**,
making them structurally unable to respond to rapid market movements.

The goal was not just fast entry — but also **continuous position optimization after entry**:
receiving ML/DL signals in real-time and adjusting orders at tick-level speed to extract maximum profit,
functioning as an extreme trailing stop driven by machine learning.
To achieve this, a **C++20 HFT execution engine** was designed and implemented from scratch.

---

### 문제: 파이썬 → 거래소 주문 경로의 지연 (Problem: Python → Exchange Order Path Latency)

KO:
**현상**: 파이썬에서 `ccxt`를 통해 거래소에 주문을 넣는 경우:
1. Python → HTTP REST 요청 → 거래소 응답: **200~500ms**
2. 트레일링 스탑 업데이트 주기: **60초 단위 스캔** (그 사이 가격 급변 시 대응 불가)
3. 시장가 주문(Market Order)이 체결되기까지 추가 슬리피지 발생

**결론**: "언제 사야 하는지"를 아무리 정확하게 알아내도, "주문을 넣는 속도"가 느리면 의미가 없음.

EN:
**Symptom**: Python's `ccxt`-based order path:
1. Python → HTTP REST → Exchange Response: **200–500ms**
2. Trailing stop scan interval: **60-second loops** (unable to react to rapid price moves in between)
3. Additional slippage on Market Orders before fill

**Conclusion**: No matter how accurate the signal is, slow order execution makes it meaningless.

---

### 해결 방법: 3-Phase C++ HFT 아키텍처 (Solution: 3-Phase C++ HFT Architecture)

KO:
C++20으로 3단계 구조의 초저지연 엔진 `EngineHFT`를 설계:

**Phase 1 — Python↔C++ 공유 메모리 IPC (`SharedMemReader`)**
- 파이썬의 ML/DL 신호를 **공유 메모리(Shared Memory)**로 C++에 전달.
- Boost.Interprocess 기반, Windows(`mmap`) / Linux(`/dev/shm`) / Mac(POSIX) 크로스플랫폼 지원.
- C++ 측은 **스핀락(Spin-Lock)으로 나노초 단위 폴링**, CPU Core 1에 Thread Affinity 고정.
- `_mm_pause()` 명령으로 CPU 발열 제어하면서도 지연 시간 최소화 (~10ns 수준).

**Phase 2 — 바이낸스 WebSocket 실시간 스트리밍 (`BinanceWebSocket`)**
- Boost.Beast + OpenSSL 기반 **비동기(Async) WebSocket** 연결.
- `btcusdt@trade` 스트림에서 **매 체결 틱(Tick)마다** 가격을 수신.
- JSON 파싱 없이 **C 문자열 직접 검색**(`std::search`)으로 가격 추출 → 파싱 오버헤드 제거.
- `TCP_NODELAY` 적용, Core 2에 Thread Affinity 고정.
- 매 틱마다 `OrderManager::onTick()` 호출 → 트레일링 스탑 나노초 수준 갱신.

**Phase 3 — REST API 즉시 주문 실행 (`BinanceRestClient` + `OrderManager`)**
- HMAC-SHA256 서명 생성 후 바이낸스 선물 API (`/fapi/v1/order`)에 시장가 주문 발사.
- `OrderManager`는 **Lock-Free 원자적 상태 관리** (`std::atomic`): IPC 스레드와 WebSocket 스레드 간 경쟁 조건(Race Condition) 제거.
- 트레일링 스탑 + 고정 익절(TP) + 고정 손절(SL) 로직을 C++ 내부에서 틱 단위로 관리.
- **포지션 보유 중에도** 파이썬에서 ML/DL 신호가 갱신될 때마다 즉시 스탑/익절 라인을 재조정하여 이익 극대화.

**빌드 시스템**: CMake + vcpkg (Boost, OpenSSL, nlohmann_json)

EN:
Designed a 3-phase ultra-low-latency engine `EngineHFT` in C++20:

**Phase 1 — Python↔C++ Shared Memory IPC (`SharedMemReader`)**
- Delivers Python ML/DL signals to C++ via **Shared Memory**.
- Boost.Interprocess-based, cross-platform: Windows (`mmap`), Linux (`/dev/shm`), Mac (POSIX).
- C++ side uses **Spin-Lock polling at nanosecond granularity**, pinned to CPU Core 1 with Thread Affinity.
- `_mm_pause()` instruction controls CPU heat while minimizing latency (~10ns level).

**Phase 2 — Binance WebSocket Real-Time Streaming (`BinanceWebSocket`)**
- Boost.Beast + OpenSSL **async WebSocket** connection.
- Receives price on **every trade tick** from `btcusdt@trade` stream.
- No JSON parsing — **raw C-string search** (`std::search`) extracts price directly → zero parsing overhead.
- `TCP_NODELAY` enabled, pinned to CPU Core 2 with Thread Affinity.
- Calls `OrderManager::onTick()` per tick → nanosecond-level trailing stop updates.

**Phase 3 — REST API Instant Order Execution (`BinanceRestClient` + `OrderManager`)**
- HMAC-SHA256 signature generation → market order fire to Binance Futures API (`/fapi/v1/order`).
- `OrderManager` uses **Lock-Free atomic state management** (`std::atomic`): eliminates race conditions between IPC thread and WebSocket thread.
- Trailing stop + fixed TP + fixed SL logic managed per-tick inside C++.

**Build system**: CMake + vcpkg (Boost, OpenSSL, nlohmann_json)

---

### 결과 (Result)

KO:
- CMake 빌드 성공 (Windows MSVC / C++20)
- `test_shm.py` ↔ `EngineHFT` 공유 메모리 양방향 통신 검증 완료 (쓰기 지연: ~1µs)
- WebSocket 실시간 틱 수신 + `OrderManager` 트레일링 스탑 연동 검증 완료
- REST API 시장가 주문 발사 및 응답 수신 테스트 코드 준비 (테스트넷 키 삽입 시 즉시 실전 가능)

EN:
- CMake build successful (Windows MSVC / C++20)
- `test_shm.py` ↔ `EngineHFT` shared memory bidirectional communication verified (write latency: ~1µs)
- WebSocket real-time tick reception + `OrderManager` trailing stop integration verified
- REST API market order fire and response reception test code ready (live-ready upon testnet key insertion)

---

### 배운 것 (Learnings) / Learnings

KO:
1. **Sleep(0)도 느리다**: HFT에서는 `std::this_thread::sleep_for`조차 수십 µs의 커널 스케줄링 지연을 만듦.
   진정한 초저지연은 `_mm_pause()` + Spin-Lock + Thread Affinity의 조합으로만 달성 가능.

2. **JSON 파싱은 병목이다**: 매 틱마다 `nlohmann::json::parse()`를 호출하면 µs 단위 지연 발생.
   `std::search`로 `"p":"` 패턴을 직접 찾아 가격을 추출하는 것이 WebSocket 처리 속도의 핵심.

3. **Lock-Free 설계의 중요성**: IPC 스레드(Core 1)와 WebSocket 스레드(Core 2)가 같은 포지션 상태를 공유할 때
   `std::mutex`는 µs 단위 블로킹을 만듦. `std::atomic` + `memory_order_acquire/release`로 무경합 상태 전환 구현.

4. **크로스플랫폼 공유 메모리**: Windows의 `Named Memory-Mapped File`과 Linux/Mac의 POSIX `shm_open`은
   Boost.Interprocess로 통합할 수 있지만, 파이썬 측 코드도 OS별 분기가 필요함 (`mmap` vs `multiprocessing.shared_memory`).

EN:
1. **Even Sleep(0) is too slow**: In HFT, `std::this_thread::sleep_for` introduces tens of µs of kernel scheduling delay.
   True ultra-low latency requires `_mm_pause()` + Spin-Lock + Thread Affinity combined.

2. **JSON parsing is a bottleneck**: Calling `nlohmann::json::parse()` per tick adds µs-level delay.
   Using `std::search` to find `"p":"` pattern and extracting price directly is the key to WebSocket processing speed.

3. **Lock-Free design matters**: When IPC thread (Core 1) and WebSocket thread (Core 2) share position state,
   `std::mutex` creates µs-level blocking. `std::atomic` + `memory_order_acquire/release` enables contention-free state transitions.

4. **Cross-platform shared memory**: Windows `Named Memory-Mapped File` and Linux/Mac POSIX `shm_open`
   can be unified via Boost.Interprocess, but Python-side code also requires OS-specific branching (`mmap` vs `multiprocessing.shared_memory`).

---

## 2026-02-23: V2 학습 파이프라인 실전 가동 및 독립형 리팩토링
## 2026-02-23: V2 Training Pipeline Production Run & Standalone Refactoring

### 배경 (Background)

KO:
2/22에 설계한 V2 아키텍처(HMM + TCN + PPO)를 **실제 15개 코인의 18개월치 데이터**로
학습시키는 과정.

EN:
The V2 architecture (HMM + TCN + PPO) designed on 2/22 was trained against **18 months of real data
across 15 coins**.


### 문제 1: 로컬 사양 제한으로 RL 학습 중단 (Problem 1: RL Training Freeze Due to Local Hardware Limits)

KO:
**현상**: `train_rl_v2.py` 실행 시 RL 에이전트(PPO)가 학습 도중 컴퓨터가 멈추거나 극도로 느려짐.

**원인 분석**:
- `_get_observation()` 함수가 **매 스텝(10만 번)마다** TCN 딥러닝 추론 + MarketAnalyzer 전체 분석을 호출
- `n_steps=2048`, `batch_size=512`로 롤아웃 버퍼가 RAM을 과도하게 점유
- `max_steps=2000`으로 에피소드가 길어 한 에피소드 완료까지 수분 소요

**해결** (환경 파일 수정 없이 `train_rl_v2.py`만 변경):
- `n_steps`: 2048 → **512** (RAM 1/4 절감)
- `batch_size`: 512 → **128**
- `n_epochs`: 10 → **5** (CPU 연산 절반)
- `max_steps`: 2000 → **1000** (에피소드 길이 단축)
- `LightweightObsWrapper` 래퍼 추가: 관측값을 `float32`로 강제 변환하여 메모리 누수 방지
- `gc.collect()` 학습 후 호출: 즉시 메모리 해제

EN:
**Symptom**: During `train_rl_v2.py` execution, the RL agent (PPO) caused the computer to freeze or become extremely slow.

**Root cause analysis**:
- `_get_observation()` was calling TCN deep learning inference + full MarketAnalyzer analysis **every step (100K times)**
- `n_steps=2048`, `batch_size=512` caused rollout buffer to over-consume RAM
- `max_steps=2000` made each episode take several minutes to complete

**Fix** (modified `train_rl_v2.py` only — no environment file changes):
- `n_steps`: 2048 → **512** (RAM usage reduced to 1/4)
- `batch_size`: 512 → **128**
- `n_epochs`: 10 → **5** (CPU computation halved)
- `max_steps`: 2000 → **1000** (episode length shortened)
- Added `LightweightObsWrapper`: forces observations to `float32` to prevent memory leaks
- `gc.collect()` called post-training for immediate memory release

---

### 배운 것 (Learnings) / Learnings

KO:
1. **코드 이식성(Portability) 설계**: 상속(`from indicator_cache import IndicatorCache`)은 편리하지만,
   폴더를 옮기거나 V1 파일 구조가 다른 환경에서는 즉시 깨짐.
   프로덕션 배포를 고려한다면 **독립형 클래스**가 훨씬 안전.

2. **RL 메모리 버짓**: 강화학습에서 `n_steps × obs_dim × num_envs × float32`가 롤아웃 버퍼 크기를 결정.
   40차원 관측 + 2048 스텝만으로도 수백 MB를 차지할 수 있음.
   로컬 사양이 제한적이면 **먼저 n_steps를 줄이는 것**이 가장 효과적인 메모리 절감 수단.

EN:
1. **Portability by design**: Inheritance (`from indicator_cache import IndicatorCache`) is convenient but
   breaks immediately when folders move or V1 file structures differ.
   For production deployment, **standalone classes** are far safer.

2. **RL memory budget**: In reinforcement learning, `n_steps × obs_dim × num_envs × float32` determines rollout buffer size.
   40-dim observations + 2048 steps can consume hundreds of MBs.
   For limited local hardware, **reducing n_steps first** is the most effective memory reduction lever.

---

## 2026-02-22: AdaptiveTradingSystem V2 통합 — 하이브리드 인공지능 트레이딩 봇 구축
## 2026-02-22: AdaptiveTradingSystem V2 Integration — Hybrid AI Trading Bot Architecture

### 배경 (Background)

KO:
V1(Rule-based 시스템)의 잦은 휩쏘(Whipsaw)로 인한 승률 하락과 레짐 판단의 모호성을 해결하기 위해,
비정상적인 금융 시계열 데이터에 강건한 **강화학습 및 딥러닝 기반 V2 업그레이드**를 단행.

EN:
To resolve the degrading win rate caused by frequent whipsaws and ambiguous regime detection in V1 (Rule-based system),
a major **Deep Learning & Reinforcement Learning (V2) upgrade** was executed, proving robust against non-stationary financial time series.

---

### 문제 1: 정적 임계값과 비정상성 편향 (Problem 1: Static Thresholds & Non-stationarity Bias)

KO:
**현상**: V1 모델에서 보조지표(RSI, MACD)의 고정된 임계값(예: RSI > 60)은 시장의 변동성이 바뀔 때마다 오작동을 일으킴.
과거에 성공했던 임계값이 미래에도 통한다는 보장이 없는 전형적인 '비정상성(Non-stationarity)' 한계 노출.

EN:
**Symptom**: In the V1 model, static thresholds for technical indicators (e.g., RSI > 60) failed whenever market volatility regimes shifted.
This exposed the classic 'Non-stationarity' limitation—past successful thresholds do not guarantee future performance.

---

### 문제 2: 다차원 리스크 관리 부재 (Problem 2: Lack of Multi-dimensional Risk Management)

KO:
**현상**: 승률뿐만 아니라 진입 금액(Position Size), 레버리지(Leverage), 조기 청산 인내도(Exit Patience) 등을
시장 상황에 따라 유기적으로 조절해야 함에도 불구하고, V1에서는 하드코딩된 비율로 단방향 관리만 됨.

EN:
**Symptom**: Market conditions require organic adjustment not only of entry direction but also Position Sizing, Leverage, and Exit Patience.
V1 rigidly applied hardcoded ratios, creating a one-dimensional risk management bottleneck.

---

### 해결 방법: 하이브리드 V2 아키텍처 도입 (Solution: Hybrid V2 Architecture)

KO:
결정 단위를 3단계로 쪼개고 각각 최적의 AI 방법론을 적용한 통합 파이프라인(`train_all_v2.py`) 설계:

1. **환경 분석**: 비지도 학습(Unsupervised)과 지도 학습(Supervised)을 결합한 앙상블 모델 도입.
   단순 가격 지표를 다수의 파생 변수(시장 미시구조 통계 포함)로 확장하고, 현재를 넘어 **미래의 상태 전환 확률(Regime Transition Probability)**을 예측.
2. **시그널 생성**: CNN 계열의 시계열 특화 딥러닝 모델 도입.
   과적합 방지를 위해 인과적 패딩(Causal Padding)을 적용하여 철저히 미래 데이터를 차단하고 확률 분포 기반 신뢰도(Confidence) 반환.
3. **자금 관리**: 심층 강화학습(Deep RL) 에이전트 도입.
   다차원 시장 환경을 다차원 행동 공간(매매 방향, 리스크 파라미터 등)으로 맵핑하여 리스크 조정 수익률 기반 보상 함수를 극대화하도록 자가 학습.
4. **하방 호환성 (Fallback)**:
   V2 딥러닝 컴포넌트 오류 시, 프로그램이 크래시(Crash)되지 않고 기존 안전한 V1(Rule-based) 엔진으로 우회하도록 마이크로서비스 관점의 구조적 안정성을 채택.

EN:
Divided the decision unit into 3 layers, applying optimal AI methodologies per layer, integrated via `train_all_v2.py`:

1. **Environment Analysis**: Combined Unsupervised and Supervised ensemble.
   Expanded features to a large set of structural variables (including market microstructure statistics) to predict the **Regime Transition Probability** rather than just identifying the current state.
2. **Signal Generation**: Introduced a CNN-family deep learning model specialized for time series.
   Applied Causal Padding strictly to prevent look-ahead bias, outputting probabilistic Confidence.
3. **Capital Management**: Introduced a Deep Reinforcement Learning Agent.
   Mapped a multi-dimensional observation space to a multi-dimensional continuous action space (Direction, Risk Parameters, etc.), self-optimizing a risk-adjusted reward function.
4. **Backward Compatibility (Fallback)**:
   Adopted a robust microservice-like architecture: if V2 deep learning components fail to load or error out, the system gracefully falls back to the reliable V1 (Rule-based) engine instead of crashing.

---

### 결과 및 배운 점 (Result & Learnings)

KO:
1. **과적합 통제의 중요성**: '수익이 나는 척'하는 코드를 짜는 것은 쉽지만, `rolling(raw=True)` 적용이나 교차피처 미래 참조를 원천 차단하는 구조를 세우는 과정이 V2 성패의 가장 큰 핵심이었음.
2. **소프트웨어 공학적 설계 가치**: 단일 봇을 수학적으로 고도화하는 것을 넘어, 객체지향의 상속(Inheritance)과 폴백(Fallback) 패턴을 활용함으로써 시스템 운영 중에 AI 모델을 바꿔치기(Hot-swap)할 수 있는 무중단 시스템 베이스를 확보함.

EN:
1. **Controlling Overfitting is Everything**: Writing code that "looks profitable" is easy. Enforcing `rolling(raw=True)` and architecturally blocking look-ahead bias across cross-features was the true differentiator for V2's integrity.
2. **Engineering Resiliency**: Beyond mathematical upgrades, utilizing Inheritance and Fallback patterns secured a zero-downtime foundation, natively allowing hot-swapped AI model weights during live operations.

---

## 2026-02-21: AdaptiveTradingSystem 전략 재설계 — MomentumTrend v1 → ABT v2
## 2026-02-21: AdaptiveTradingSystem Strategy Overhaul — MomentumTrend v1 → ABT v2

### 배경 (Background)

KO:
이전까지 Adaptive는 Trend+Pullback 전략(RSI>60=Long 류)을 사용했으나 평균 승률 24%로 실패 판정.
원인 분석 후 **전략 로직 자체를 교체**하는 방향으로 진행.

EN:
The Adaptive system had been running a Trend+Pullback strategy (RSI-based momentum entry)
but achieved only ~24% win rate. Root cause analysis concluded a fundamental logic flaw,
leading to a complete strategy replacement.

---

### 문제 1: MomentumTrend v1 시도 및 실패 (Problem 1: MomentumTrend v1 Attempt)

KO:
**시도한 것**: Trend+Pullback 실패 후, 순수 추세 추종(MomentumTrend)으로 전환.
- RSI/Stoch/BB 완전 제거 — 추세와 역방향 신호를 생성하던 원인
- ADX Hard Gate 추가 — ADX 기준값 미달 시 즉시 NEUTRAL (횡보 자동 차단)
- EMA 정렬 6봉(30분) 지속성 확인 — 즉각적 EMA 플립 방지

**결과** (3구간 백테스트):

| 기간 | 승률 | 손익비 | 수익률 |
|------|-----|------|------|
| 상승장 (6개월) | ~33% | ~1.6:1 | 대폭 손실 |
| 하락장 (6개월) | ~30% | ~1.7:1 | 대폭 손실 |
| 횡보 (2개월) | ~31% | ~1.6:1 | 손실 |

**진전된 부분**: 손익비 > 1 (평균 수익 > 평균 손실) — 트레일링 스탑 버그 수정 효과
**실패 원인**: 5분봉 EMA 기반 추세 추종 승률은 구조적으로 ~33% 상한 존재
→ 필요 손익분기 승률(~39%)에 미달 → 파라미터 조정으로 해결 불가

EN:
**What was tried**: After Trend+Pullback failure, pivoted to pure momentum trend-following.
- Removed RSI/Stoch/BB — these were generating signals counter to trend direction
- Added ADX Hard Gate — below threshold, return NEUTRAL immediately (ranging market filter)
- Required EMA alignment persistence across multiple bars — prevents entries on transient EMA flips

**Results** (3-period backtest):

| Period | WR | P/F | Return |
|--------|-----|-----|--------|
| Bull (6 months) | ~33% | ~1.6:1 | Large loss |
| Bear (6 months) | ~30% | ~1.7:1 | Large loss |
| Ranging (2 months) | ~31% | ~1.6:1 | Loss |

**What improved**: Profit factor > 1 (avg win > avg loss) — trailing stop bug fix took effect
**Failure reason**: 5-minute EMA-based trend following has a structural ~33% WR ceiling.
Break-even WR needed (~39%) cannot be reached; no parameter adjustment can close this gap.

---

### 문제 2: 트레일링 스탑 Dead Parameter 버그 (Problem 2: Trailing Stop Dead Parameter)

KO:
`risk_manager.py`의 `update_trailing_stop()`에서 config로 주입된 `trailing_stop_distance`가
실제 로직에 반영되지 않고 ATR 하드코딩 값만 사용되던 버그 발견.
- 결과: 0~2% 수익 구간에서 스탑이 바짝 붙어 TP 전에 청산 → R:R 역전 현상
- 수정: config 파라미터를 실제 로직에 반영, 초기 수익 구간에서 트레일링 없음으로 변경

EN:
Discovered a dead-parameter bug in `risk_manager.py`'s `update_trailing_stop()`:
the config-injected `trailing_stop_distance` was loaded but never applied.
Hardcoded ATR multipliers were used instead.
- Effect: stop loss hugged the position at 0–2% profit, triggering before TP → R:R inversion
- Fix: wired config parameter into actual logic; removed trailing in early profit phase

---

### 해결 방법: ABT v2 전략 채택 (Solution: ABT v2 Strategy)

KO:
**핵심 결정**: 5분봉 추세 추종의 구조적 한계를 인정하고 전략 자체를 교체.

**새 전략**: ABT (Adaptive Breakout-Trend)
- 타임프레임을 상향 조정하여 노이즈 대 신호 비율 개선
- 진입 구조: **다단계 필터** (환경 필터 → 방향 판단 → 타이밍 트리거 순차 검증)
- R:R 구조를 하향 조정하여 손익분기 승률 요구치를 낮춤
- 다단계 트레일링 스탑: 수익 구간별 차등 적용 (초기 수익 보호 → 후기 수익 극대화)

**참고 근거**: 이전 프로젝트에서 유사 전략의 소규모 실거래 성공 경험을 참고하되,
설계는 완전히 새로 수행.

EN:
**Core decision**: Acknowledged the structural WR ceiling of short-timeframe trend following.
Replacing the strategy rather than tuning parameters.

**New strategy**: ABT (Adaptive Breakout-Trend)
- Upgraded to a higher timeframe for improved signal-to-noise ratio
- Entry structure: **Multi-layer filter** (Environment → Direction → Timing sequential validation)
- Adjusted R:R target downward to reduce break-even win rate requirement
- Multi-phase trailing stop: differentiated by profit stage (early profit protection → late profit maximization)

**Rationale**: Referenced partial success from a similar strategy in a prior project's small-scale live trading.
Design was built entirely from scratch.

---

### 배운 것 (Learnings) / Learnings

KO:
1. **전략 교체 기준**: 파라미터 조정으로 해결할 수 없는 구조적 한계가 확인되면 전략 자체를 교체해야 함.
   "어떤 숫자를 바꾸면 나아지지 않을까"라는 생각이 드는 순간이 교체 타이밍의 신호.

2. **승률 vs 손익비의 균형**: 손익비 > 1 달성 자체는 의미 있지만 충분하지 않음.
   손익분기 승률(= 1 / (1 + 손익비)) 이상의 승률이 반드시 필요.
   예: 손익비 1.6:1 → 손익분기 승률 38.5% → 실제 승률 33%이면 손실.

3. **타임프레임 선택의 중요성**: 동일한 전략 로직이라도 5분봉과 15분봉에서 신뢰도가 다름.
   짧은 타임프레임일수록 노이즈가 많아 진입 신호가 빠르게 무효화될 수 있음.

4. **3-Layer 구조의 가치**: 진입 조건을 "환경 필터 → 방향 판단 → 타이밍"으로 분리하면
   각 계층이 독립적으로 책임을 지므로 어느 계층에서 문제가 발생하는지 분석이 쉬워짐.
   단일 점수 합산 구조에서는 불가능한 투명성.

EN:
1. **When to replace vs. tune**: When a structural ceiling is confirmed and no parameter change
   can overcome it, the strategy logic itself must be replaced. Asking "what number should I change?"
   is the signal that replacement — not tuning — is needed.

2. **Win rate vs. profit factor balance**: Achieving PF > 1 is meaningful but insufficient.
   Win rate must exceed the break-even WR (= 1 / (1 + PF)).
   Example: PF 1.6:1 → break-even WR 38.5% → actual WR 33% → still a loss.

3. **Timeframe selection matters**: The same strategy logic produces very different reliability
   on 5-minute vs. 15-minute candles. Shorter timeframes have more noise,
   causing entry signals to become invalidated rapidly.

4. **Value of layered entry structure**: Separating entry conditions into Environment → Direction → Timing
   makes each layer independently accountable, making it easy to diagnose which layer failed.
   This transparency is impossible with a single aggregated score approach.

---

## 2026-02-19: Fixed Take-Profit 도입 및 핵심 버그 수정
## 2026-02-19: Introducing Fixed Take-Profit & Critical Bug Fix

### 문제 (Problem) / Problem

KO:
1. 트레일링 스탑 업데이트 시 고정 익절(TP) 주문이 함께 취소됨
   - `update_trailing_stop()`에서 `cancel_all_orders()`를 호출하는 구조 때문
   - TP 활성화(`use_take_profit: True`)를 해도 실제로는 매번 취소되어 무효화됨
   - 결과적으로 수동으로 TP 가격을 기억해서 직접 청산하는 임시방편으로 운영 중

2. 수익 중인 포지션이 역전되어 손실로 마무리되는 패턴 반복
   - 트레일링 스탑만으로는 빠른 역전 움직임에 대응 불가
   - 특히 수익이 쌓이던 포지션이 순식간에 원래 손절선까지 내려오는 경우 발생

EN:
1. Fixed take-profit (TP) orders were being cancelled every time the trailing stop updated.
   - Root cause: `update_trailing_stop()` called `cancel_all_orders()` internally.
   - Even with `use_take_profit: True` in config, TP orders were silently wiped on every trailing stop adjustment.
   - Workaround in place: manually remembering TP prices and closing positions by hand.

2. Profitable positions repeatedly reversed into losses.
   - Trailing stop alone was too slow to react to sharp reversals.
   - Positions that had built up unrealized profit would collapse back to the original stop-loss level.

---

### 가설 (Hypothesis) / Hypothesis

KO:
- 공격적 단타 시스템(5분봉, 7종목)에서는 큰 추세 수익보다
  일관된 R:R로 승률을 확보하는 것이 기대수익(EV) 측면에서 유리
- 트레일링 스탑은 손실 제한 역할에 집중, TP는 익절 확정 역할로 분리

EN:
- For an aggressive scalping system (5-min candles, 7 symbols), securing consistent R:R wins
  produces better expected value (EV) than chasing large trend moves.
- Design philosophy: trailing stop handles downside protection; fixed TP handles profit capture. Separate concerns.

---

### 해결 방법 (Solution) / Solution

KO:

**버그 수정** (`execution_engine.py`):
```python
# 변경 전: 모든 주문 취소 (TP까지 사라짐)
self._retry(self.exchange.cancel_all_orders, lsym)

# 변경 후: 스탑로스 주문 ID만 취소 (TP 보존)
stop_order_id = position.get('stop_order_id')
if stop_order_id:
    self._retry(self.exchange.cancel_order, stop_order_id, lsym)
```

**포지션 청산 시나리오별 동작 정리**:
| 상황 | SL 주문 | TP 주문 |
|---|---|---|
| 트레일링 스탑 업데이트 | 취소 후 재배치 | 유지 |
| SL 발동 (손절) | 체결 | 거래소가 자동 취소 |
| TP 발동 (익절) | 거래소가 자동 취소 | 체결 |
| 수동 종료 | `cancel_all_orders`로 취소 | `cancel_all_orders`로 취소 |

EN:

**Bug fix** (`execution_engine.py`):
```python
# Before: cancel ALL orders (TP disappears as side effect)
self._retry(self.exchange.cancel_all_orders, lsym)

# After: cancel only the stop-loss order by ID (TP is preserved)
stop_order_id = position.get('stop_order_id')
if stop_order_id:
    self._retry(self.exchange.cancel_order, stop_order_id, lsym)
```

**Exit scenario matrix**:
| Scenario | SL Order | TP Order |
|---|---|---|
| Trailing stop update | Cancelled & replaced | Preserved |
| SL triggered (stop-out) | Filled | Auto-cancelled by exchange |
| TP triggered (take profit) | Auto-cancelled by exchange | Filled |
| Manual close | Cancelled via `cancel_all_orders` | Cancelled via `cancel_all_orders` |

---

### 추가 변경사항 / Additional Changes

**SIDEWAYS 레짐 진입 임계값 강화** (`signal_generator.py`):

KO:
- 문제: SIDEWAYS 레짐에서 손실 비중이 높음 (2/17~18 로그 분석)
- 해결: 완전 차단이 아닌 임계값 상향 (고품질 신호만 통과, 강한 신호는 여전히 진입 가능)
- SIDEWAYS_QUIET, SIDEWAYS_CHOPPY 각각 임계값 상향 조정

EN:
- Problem: SIDEWAYS regime showed a disproportionate share of losses (log analysis 2/17–18).
- Solution: Raised entry threshold per regime — not a full block, but only high-confidence signals pass.
- SIDEWAYS_QUIET and SIDEWAYS_CHOPPY thresholds raised independently.

**최대 동시 포지션 축소 / Max concurrent positions reduced** (`config.py`):

KO: 문제: 소규모 계좌에서 다수 포지션 × 리스크 비율 = 과도한 동시 노출 → "잔고 부족" 에러 반복 발생
    해결: 최대 동시 포지션 수 축소 (계좌 규모 증가 시 확대 예정)

EN: Problem: On a small account, multiple simultaneous positions × risk per trade = excessive exposure → repeated "insufficient balance" errors.
    Solution: Reduced max concurrent positions. Will scale up as account grows.

---

### 결과 (Result) / Result

KO:
- 2026-02-19 이후부터 클린 데이터 수집 시작 (버그 수정 + 전체 설정 반영)
- 이전 로그(2/17~18) 기준 승률: 51~62% 범위 (설정 전환 시점 혼재)
- 2주 후(3월 초) 레짐별 승률, TP 체결률 분석 예정

EN:
- Clean data collection begins from 2026-02-19 (all bug fixes + config changes applied)
- Prior log win rates (2/17–18): 51–62% range (mixed — config transitions overlap)
- Planned analysis in ~2 weeks (early March): win rate by regime, TP fill rate

---

### 배운 것 (Learnings) / Learnings

KO:
1. **거래소 API 특성**: Bybit는 주문 수정(modify) API 미지원 → cancel + 재주문 방식 강제.
   이 구조에서 `cancel_all_orders`는 의도치 않은 부수효과를 만들 수 있음.
   → 항상 주문 ID 기반 개별 취소를 기본으로 사용해야 함.

2. **Dead Parameter 발견**: `config.py`의 `trailing_stop_distance` 값은 로드 후 미사용.
   실제 트레일링 스탑은 `risk_manager.py`에 하드코딩된 단계별 로직으로 동작.
   → config 값을 바꿔도 동작에 영향 없음. 추후 코드 정리 필요.

3. **데이터 기반 의사결정의 한계**: 3일 로그에서 SIDEWAYS 레짐 성과가 날마다 달랐음
   (2/17: SIDEWAYS 손실 우세 vs 2/18 오전: SIDEWAYS 수익 우세).
   하드코딩 임계값은 단기 데이터 과적합 위험. 2~3주 후 재검토 필요.

EN:
1. **Exchange API constraint**: Bybit does not support order modify — forced into cancel + re-submit pattern.
   In this architecture, `cancel_all_orders` creates unintended side effects.
   → Default to individual order cancellation by ID; use `cancel_all_orders` only for full position closes.

2. **Dead parameter discovered**: `trailing_stop_distance` in `config.py` is loaded but never read.
   Actual trailing stop logic is hardcoded in `risk_manager.py` as a staged tightening process.
   → Changing the config value has zero effect. Cleanup needed.

3. **Limits of small-sample decisions**: SIDEWAYS regime performance varied day by day across only 3 days of logs.
   Hardcoded thresholds risk overfitting to short-term data. Revisit in 2–3 weeks.

---

## 2026-02-18: 시스템 파라미터 조정
## 2026-02-18: System Parameter Tuning

### 변경 내용 / Changes

KO:
- 확신도 임계값 상향 (저품질 신호 필터링 강화)
- 트레일링 스탑 거리 조정 (포지션 여유 확보 — 실제 dead parameter였음, 추후 정리 필요)
- 스캔 주기 연장: 30초 → 60초 (중복 진입 시도 감소)

EN:
- Confidence threshold raised (stronger filtering of low-quality signals)
- Trailing stop distance adjusted (give positions more room — later confirmed dead parameter, cleanup needed)
- Scan interval extended: 30s → 60s (reduce duplicate entry attempts during open positions)

### 이유 / Rationale

KO:
- 기존 임계값에서 기준치를 겨우 넘긴 신호들이 손실 기여 비중 높음
- 30초 스캔은 포지션 보유 중 중복 시도로 "잔고 부족" 에러 빈발

EN:
- Signals just above the previous threshold were disproportionately contributing to losses.
- 30-second scan interval caused duplicate entry attempts while positions were open, generating frequent "insufficient balance" errors.

---

## 2026-02-05 ~ 2026-02-09: AdaptiveTradingSystem 개발 완료 및 멀티봇 충돌 해결
## 2026-02-05 ~ 2026-02-09: AdaptiveTradingSystem Completion & Multi-Bot Conflict Resolution

### 배경 / Background

KO:
AggressiveTradingSystem을 개발하는 동시에, **AdaptiveTradingSystem(단타)**과
**TrendFollowingSystem(중장기)**을 별도로 개발하는 멀티봇 아키텍처를 구성한 기간.

EN:
This period covered building a multi-bot architecture in parallel:
**AdaptiveTradingSystem** (short-term scalping) and **TrendFollowingSystem** (swing/trend).

---

### 문제 1: 백테스터 버그 (Problem 1: Backtester Bugs)

KO:
AdaptiveTradingSystem의 백테스터(`backtester.py`)에서 두 가지 핵심 버그 발견:

1. **레버리지 1x 고정 버그**: `risk_manager.py`에서 ATR 기반 변동성 계산이 잘못 처리되어
   레버리지가 항상 1배로 고정됨. 포지션 사이징 전체가 의도와 달리 작동하고 있었음.

2. **P&L 이중 적용 버그**: equity curve 계산에서 레버리지가 두 번 적용되던 문제.
   실제 수익률이 과대 계상되어 백테스트 결과가 왜곡됨.
   Dead code 제거 후 수정.

EN:
Two critical bugs found in `backtester.py`:

1. **Leverage stuck at 1x**: ATR-based volatility in `risk_manager.py` was not being processed correctly,
   causing leverage to default to 1x regardless of market conditions.
   The entire position sizing logic was silently broken.

2. **P&L double-application**: The equity curve calculation was applying leverage twice,
   inflating simulated returns. Removed dead code and corrected.

---

### 문제 2: 고정 임계값의 한계 (Problem 2: Fixed Entry Threshold Limitations)

KO:
초기 설계는 롱/숏 진입 확신도를 고정 임계값으로 적용했음.
그러나 실전에서는 시장 레짐(추세/횡보/고변동성)에 따라 신호의 신뢰도가 달라지는 문제.
- 추세장에서의 신호와 횡보장에서의 같은 수치 신호는 신뢰도가 다름
- 하나의 임계값으로 모든 시장 상황을 처리하는 것은 근본적 한계

**해결**: 레짐별 동적 임계값 구현
- `STRONG_UPTREND`: 롱 진입에 유리하게 조정
- `STRONG_DOWNTREND`: 숏 진입에 유리하게 조정
- `HIGH_VOLATILITY`: 진입 조건 강화 (모든 방향)
- 기본 임계값 위에 additive adjustment를 적용하는 구조

EN:
Initial design used fixed confidence thresholds for long/short entry.
In live trading, the reliability of signals varies significantly by market regime.
- The same numerical confidence score means different things in a trending vs. ranging market
- A single threshold cannot meaningfully filter across all conditions

**Solution**: Regime-conditional dynamic thresholds
- `STRONG_UPTREND`: Long-entry threshold adjusted favorably
- `STRONG_DOWNTREND`: Short-entry threshold adjusted favorably
- `HIGH_VOLATILITY`: Entry conditions tightened for all directions
- Structure: additive adjustments on top of a base threshold (not absolute overrides)

---

### 문제 3: 두 봇 포지션 충돌 (Problem 3: Multi-Bot Position Conflicts)

KO:
AdaptiveTradingSystem(단타)과 TrendFollowingSystem(중장기)을 동시에 운영할 때
같은 거래소 계정에서 충돌 발생:

1. **추매 문제**: 동일 코인에서 두 봇이 같은 방향 진입 → 의도치 않은 포지션 확대
2. **스탑 간섭**: 단타봇의 좁은 스탑로스가 중장기 포지션까지 청산
3. **반대 진입**: 두 봇이 같은 코인에서 반대 방향 진입 → 헤지 발생 (양쪽 수수료 낭비)

**해결**:
1. 두 시스템의 포지션 추적 로직 완전 분리 (shared state 제거)
2. 각 봇이 독립적으로 API에 연결, nohup으로 별도 프로세스 운영
3. 계좌 규모가 커지면 거래소 서브계정으로 완전 분리 예정
4. 소수점 처리 버그 추가 수정 (저가 코인 주문 수량 정밀도)

EN:
Running AdaptiveTradingSystem (short-term) and TrendFollowingSystem (swing) simultaneously
on the same exchange account caused three classes of conflict:

1. **Unintended pyramiding**: Both bots entering the same direction on the same coin → position size grows beyond intent
2. **Stop interference**: Short-term bot's tight stop-loss inadvertently closing long-term positions
3. **Hedging**: Bots entering opposite directions on the same coin → fee-burning hedge with no intent

**Resolution**:
1. Fully decoupled position tracking logic between the two systems (removed shared state)
2. Each bot runs as an independent process with its own API connection via nohup
3. Sub-account separation planned when capital grows sufficiently
4. Fixed decimal precision bug for low-price coins (order quantity precision)

---

### TrendFollowingSystem 설계 결정 / TrendFollowingSystem Design Decisions

KO:
단타 시스템과 상관관계를 낮추기 위한 설계 선택:
- **타임프레임**: 4시간봉 + 일봉 (멀티 타임프레임, 단타와 완전히 다름)
- **진입 로직**: 일봉 추세 확인 → 4시간봉에서 타이밍 포착
- **보유 기간**: 수일 ~ 수주
- **청산**: 일봉 ATR 기반 트레일링 (단타보다 훨씬 넓은 스탑)
- **레버리지**: 낮게 설정 (장기 보유 중 변동성 흡수 필요)

백테스트 결과 (6개 코인, 12개월 하락장):
- Buy & Hold 대비 유의미한 Alpha 확인 → 절대 수익보다 리스크 대비 수익이 목표
- MDD: 개별 코인 대비 낮게 유지 (방어적 구조 확인)
- 거래 빈도: 월 1~2회 수준 (단타와 상관관계 낮음)

EN:
Design choices to minimize correlation with the scalping system:
- **Timeframe**: 4-hour + Daily (multi-timeframe — completely separate from 5-min scalping)
- **Entry logic**: Confirm trend on daily chart → time entry on 4H chart
- **Hold duration**: Days to weeks
- **Exit**: Daily ATR-based trailing stop (much wider than scalping system)
- **Leverage**: Conservative (must absorb volatility during extended holds)

Backtest results (6 coins, 12-month bear market):
- Meaningful alpha confirmed vs. Buy & Hold → goal is risk-adjusted return, not absolute return
- MDD: substantially lower than individual coins (defensive structure validated)
- Trade frequency: ~1–2 per month (low correlation with scalping system)

---

### 배운 것 (Learnings) / Learnings

KO:
1. **백테스터는 별도로 검증해야 함**: 전략 코드보다 백테스터 자체의 버그(레버리지 1x 고정, P&L 이중 적용)가
   결과를 더 크게 왜곡할 수 있음. "백테스트가 안 좋다 = 전략이 나쁘다"가 아닐 수 있음.

2. **단일 임계값 vs 레짐별 임계값**: 시장은 상태가 바뀐다. 추세장 신호와 횡보장 신호는
   같은 수치여도 다른 의미를 가짐. 임계값은 레짐에 따라 조정되어야 함.

3. **멀티봇 간섭은 설계 단계에서 고려해야 함**: 단일 봇 테스트에서 잘 작동하던 것이
   두 봇을 동시에 운영하면 예상치 못한 상호작용을 만들 수 있음. 포지션 상태 관리를
   처음부터 독립적으로 설계하는 것이 중요.

4. **실거래 = 가장 정직한 모의거래**: 소액 실거래를 페이퍼 트레이딩 대신 선택.
   슬리피지, 체결 지연, 심리적 압박 등 시뮬레이션에서 잡히지 않는 요소를 직접 경험.

EN:
1. **Validate the backtester, not just the strategy**: Bugs in the backtester itself (leverage stuck at 1x,
   P&L double-counted) distorted results more than strategy flaws would. "Poor backtest" ≠ "poor strategy"
   without first verifying the testing infrastructure.

2. **Single threshold vs. regime-conditional thresholds**: Markets shift between states.
   The same confidence score means different things in trending vs. ranging conditions.
   Thresholds must adapt to regime, not just to a fixed signal strength.

3. **Multi-bot interference must be designed for upfront**: What works cleanly in single-bot testing
   can produce unexpected interactions in multi-bot operation (shared positions, stop interference, hedging).
   Position state management must be fully isolated from the start.

4. **Live trading is the most honest paper trading**: Chose small-capital live trading over simulation.
   Slippage, fill delays, and psychological pressure are not captured in backtests — they must be experienced directly.

---

## 2025: ML 기반 가격 예측 시스템 구축 → 한계 발견 → 레짐 기반 시스템으로 전환
## 2025: Building an ML Price Prediction System → Discovering Its Limits → Pivoting to Regime-Based Design

### 구축한 것 (What Was Built)

KO:
단순한 모델 실험이 아닌, **프로덕션 수준의 ML 트레이딩 파이프라인 전체**를 직접 설계하고 구현.

**1. 데이터 파이프라인**
- `Master_Data_Pump.py`: 6개 코인 × 7개 타임프레임 × 1.5년치 OHLCV 수집 (총 ~300MB Parquet)
- `Live_Data_Pump.py`: 60초 주기 증분 동기화 (마지막 타임스탬프 기준 delta fetch)
- ThreadPoolExecutor 병렬 다운로드, API 속도 제한 자동 처리

**2. 피처 엔진 (72개 지표)**
- `indicators.py`: 추세(MA/EMA 7종), 모멘텀(RSI/MACD/Stochastic), 변동성(ATR/BB), 거래량(OBV/VWAP)
- **독자 설계 피처**: 가격 속도(Velocity), 가속도(Acceleration), 드래그(Drag), 엔트로피(Entropy)
- 시장 미시구조: 호가창 불균형(Order Book Imbalance), 펀딩비, 미결제약정
- 확률 피처: Monte Carlo 1000회 시뮬레이션 기반 `Prob_Up_1h`

**3. ML 학습 파이프라인**
- `Feature_Factory.py`: Triple Barrier 라벨링 적용
  - Profit Target: 진입 + ATR × 배수
  - Stop Loss: 진입 − ATR × 배수
  - Time Limit: N봉 타임아웃
  - 라벨: 0(손절), 1(타임아웃), 2(익절)
- `Model_Training.py`: PyTorch LSTM + Attention 아키텍처
  - LSTM 2레이어, Attention 가중합, FC 레이어
  - **메모리맵(Memmap) 학습**: 7GB+ 데이터를 RAM 없이 학습하는 엔지니어링
- `AutoML_Master.py`: 코인별 XGBoost 앙상블 (6개 모델 독립 학습)
- `Live_Inference_Bridge.py`: 실시간 추론 (최근 N봉 → z-score 정규화 → LSTM → 확률 출력)

**4. Streamlit 대시보드**
- 봇 실행/정지, 누적 P&L 곡선, 실시간 로그
- 멀티타임프레임 차트 (Plotly), 펀딩비/공포탐욕지수, 호가창 시각화
- 전략 시뮬레이션 샌드박스, Kelly Criterion 계산기

EN:
This was not a simple model experiment — a **full production-grade ML trading pipeline** was designed and implemented end-to-end.

**1. Data Pipeline**
- `Master_Data_Pump.py`: 1.5 years of OHLCV for 6 coins × 7 timeframes (~300MB Parquet)
- `Live_Data_Pump.py`: 60-second incremental sync (delta fetch from last timestamp)
- Parallel downloads via ThreadPoolExecutor with automatic API rate-limit handling

**2. Feature Engine (72 indicators)**
- `indicators.py`: Trend (7 MA/EMA variants), Momentum (RSI/MACD/Stochastic), Volatility (ATR/BB), Volume (OBV/VWAP)
- **Custom-designed features**: Price Velocity, Acceleration, Drag, Entropy — physics-inspired metrics
- Market microstructure: Order Book Imbalance, Funding Rate, Open Interest
- Probability feature: `Prob_Up_1h` via Monte Carlo simulation (1,000 paths)

**3. ML Training Pipeline**
- `Feature_Factory.py`: Triple Barrier Labeling (methodology from Lopez de Prado's *Advances in Financial Machine Learning*)
  - Profit Target / Stop Loss: ATR-based multipliers
  - Time Limit: N-bar timeout
  - Labels: 0 (stop-out), 1 (timeout), 2 (take-profit)
- `Model_Training.py`: PyTorch LSTM + Attention
  - Multi-layer LSTM, attention-weighted pooling, FC layers
  - **Memory-mapped training**: trained on 7GB+ datasets without loading into RAM
- `AutoML_Master.py`: Coin-specific XGBoost ensemble (6 independent models)
- `Live_Inference_Bridge.py`: Real-time inference — last N bars → z-score normalization → LSTM → softmax probabilities

**4. Streamlit Dashboard**
- Bot start/stop, cumulative P&L curve, real-time log tail
- Multi-timeframe interactive charts (Plotly), funding rates, Fear & Greed Index, order book visualization
- Strategy simulation sandbox, Kelly Criterion calculator

---

### 한계 발견 (Problem Discovered)

KO:
기술 스택은 완성됐지만 **실전 수익성 검증에서 한계**에 도달.
- 학습 데이터(in-sample)에서는 모델이 수렴하고 정확도도 나옴
- 실전 추론(out-of-sample)에서 예측 정확도가 무작위 수준으로 하락
- 원인 분석:
  1. **비정상성(Non-stationarity)**: 금융 시계열의 통계적 특성이 시간에 따라 변함. 과거 패턴이 미래에 반복된다는 보장 없음.
  2. **과적합(Overfitting)**: 72개 피처, 7GB 데이터에도 불구하고 모델이 과거 노이즈를 패턴으로 학습. 학습 데이터에서만 작동.
  3. **시장 효율성**: 단순히 반복 가능한 가격 패턴은 이미 다른 참여자들이 차익거래로 소멸시킴.

EN:
The technical stack was complete, but **real-world profitability validation revealed fundamental limits**.
- In-sample: model converged, accuracy metrics looked promising
- Out-of-sample: prediction accuracy dropped to near-random levels
- Root cause analysis:
  1. **Non-stationarity**: Financial time series change statistical properties over time; past patterns don't repeat reliably.
  2. **Overfitting**: Despite 72 features and 7GB of data, the model memorized historical noise as signal.
  3. **Market efficiency**: Simple exploitable price patterns are already arbitraged away by other participants.

---

### 전환 (Pivot)

KO:
"미래 가격의 방향을 예측"하는 것 자체가 근본적으로 불안정한 목표라는 결론.
방향을 바꿔 **"현재 시장이 어떤 상태(regime)인가"를 분류**하는 접근으로 전환:

| 이전 접근 | 현재 접근 |
|---|---|
| 미래 가격 예측 (Price Prediction) | 현재 레짐 분류 (Regime Classification) |
| 절대적 방향 예측 | 확률적 상태 식별 |
| LSTM 시퀀스 예측 | KMeans 클러스터링 + 규칙 기반 앙상블 |
| 단일 모델 (BTC 중심) | 레짐별 차별화된 신호 + 리스크 파라미터 |

이 전환이 현재 세 개 시스템(Aggressive/Adaptive/TrendFollowing)의 설계 철학이 됨.

EN:
Concluded that predicting future price direction is a fundamentally unstable objective.
Pivoted to **classifying the current market regime** instead:

| Previous Approach | Current Approach |
|---|---|
| Price Prediction | Regime Classification |
| Absolute direction forecast | Probabilistic state identification |
| LSTM sequence prediction | KMeans clustering + rule-based ensemble |
| Single model (BTC-focused) | Regime-specific signals + risk parameters |

This pivot became the core design philosophy of the current three-system architecture
(Aggressive / Adaptive / TrendFollowing).

---

### 배운 것 (Learnings) / Learnings

KO:
1. **파이프라인 완성 ≠ 전략 타당성**: 데이터 수집, 피처 엔지니어링, 모델 학습, 실시간 추론까지 모두 구현했지만, 기술적 완성도와 전략적 수익성은 독립적인 문제.
2. **Triple Barrier 라벨링의 한계**: Lopez de Prado 방법론을 적용했음에도 out-of-sample 성능이 in-sample을 따라가지 못함. 라벨링 방식이 정교해도 기저 신호(underlying signal)가 약하면 무의미.
3. **금융 ML의 핵심 질문**: "무엇을 예측할 것인가"가 "어떻게 예측할 것인가"보다 중요. 가격 방향 대신 레짐을 예측 대상으로 바꾼 것이 아키텍처 전체를 바꿈.
4. **엔지니어링 역량의 축적**: 데이터 파이프라인, 피처 엔진, 메모리맵 학습, 실시간 추론 브릿지 등 현재 시스템의 인프라 설계에 직접 활용됨.

EN:
1. **Pipeline completeness ≠ strategy validity**: Built the full stack (data → features → training → live inference) but found that engineering quality and strategic profitability are independent problems.
2. **Limits of Triple Barrier labeling**: Even applying Lopez de Prado's methodology, out-of-sample performance failed to match in-sample. Sophisticated labeling doesn't compensate for a weak underlying signal.
3. **The core question in financial ML**: *What* to predict matters more than *how* to predict. Changing the prediction target from price direction to market regime restructured the entire architecture.
4. **Engineering foundation carried forward**: The data pipeline, feature engine, memory-mapped training, and live inference bridge designs directly informed the infrastructure of the current system.

