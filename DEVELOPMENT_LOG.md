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

## 2026-04-25: 페이퍼 트레이더 사일런트 행 사고 후속 조치 — 데이터 신선도 가드, 하트비트 파일, Telegram 알림, Healthcheck 교정
## 2026-04-25: Paper Trader Silent Hang Postmortem Actions — Data Freshness Guard, Heartbeat File, Telegram Notifier, Healthcheck Correction

### 배경 (Background)

KO:
4월 19일부터 가동 중이던 SOL 페이퍼 포워드 테스트가 4월 21일 마지막 트레이드 이후 약 3일간 신호 0건 상태로 정지. 24일 점검 시 컨테이너는 살아있고 5분 주기 하트비트 로그도 정상 출력되었으나, 거래소로의 TCP 연결이 0개, 시그널·트레이드 파일은 갱신되지 않은 상태. Healthcheck가 "unhealthy" 였음에도 자동 재시작이 트리거되지 않아 사고가 장기화됨.

EN:
The SOL paper forward test running since April 19 stopped producing signals after the final trade on April 21 — three days of total silence. On April 24 inspection, the container was alive and the 5-minute heartbeat log was firing normally, yet there were zero outbound TCP connections to the exchange and the signal/trade files had not advanced. Despite the healthcheck reading "unhealthy" the entire time, the restart policy never triggered and the outage stretched on.

---

### 문제 1: 데이터 파이프라인의 사일런트 실패 (Problem 1: Silent Data Pipeline Failure)

KO:
**현상**: 데이터 fetch 호출은 예외를 던지지 않고 정상적으로 반환되었으나, 새로운 캔들이 들어오지 않음. 기존 코드의 재연결 로직은 "연속 N회 예외" 발생 시에만 트리거되도록 설계되어 있어, 캐시된 응답이나 빈 응답을 받는 사일런트 실패에는 반응하지 못함.

**구조적 원인**:
- 데이터 신선도(freshness)에 대한 관찰이 코드 어디에도 존재하지 않음.
- 하트비트 로그는 데이터 흐름과 독립적인 타이머에서 찍혀, 데이터가 끊겨도 "살아 있는 것처럼" 보였음.
- 관측 가능한 신호가 "예외" 단 하나에 의존 → 예외 없이 망가지는 모든 실패가 보이지 않음.

EN:
**Symptom**: The data fetch call kept returning successfully without raising, yet new candles never arrived. The existing reconnect logic was gated on "N consecutive exceptions" — it had no path for the silent failure mode where the call succeeds but the data is empty or cached.

**Structural cause**:
- No observation of data freshness existed anywhere in the loop.
- Heartbeat logs ran on a timer independent of data flow, so "alive process" and "live data" were conflated.
- Observability rested on a single signal — exceptions. Any failure that did not raise was invisible.

---

### 문제 2: Healthcheck가 잘못된 파일을 감시 (Problem 2: Healthcheck Watching the Wrong File)

KO:
**현상**: docker-compose healthcheck가 `state/portfolio.db` 의 mtime 을 600초 기준으로 검사. SQLite WAL 모드에서 본체 파일은 체크포인트 시에만 갱신되므로 며칠 동안 mtime 이 멈추는 것이 정상. 결과적으로 healthcheck 는 거래 발생/체크포인트 여부를 보고 있었을 뿐, 메인 루프 가동 여부와 무관했음.

EN:
**Symptom**: The docker-compose healthcheck inspected `state/portfolio.db` mtime against a 600s threshold. Under SQLite WAL mode the main DB file only updates on checkpoints, so a multi-day stale mtime is normal. The healthcheck was effectively probing "did a trade or checkpoint happen recently" — entirely unrelated to whether the event loop was running.

---

### 해결 방법 (Solution)

KO:
**1. 데이터 신선도 가드 (`main.py`)**:
- `last_new_bar_wall` 단조 시계로 "마지막으로 *새로운* 캔들을 본 시점" 추적. 단순 fetch 성공이 아니라 캔들 타임스탬프 진전 여부를 기준.
- 임계값(`stale_threshold_seconds`) 초과 시 `DATA_STALE` 이벤트 발행 + 데이터 소스 강제 재연결 시도. 복구 성공 시 `DATA_RECONNECTED` 발행.
- 기존 "연속 예외" 트리거는 유지하되, 사일런트 실패까지 커버하는 두 번째 트리거를 병렬 추가.

**2. 하트비트 파일 + Healthcheck 교정**:
- 매 폴 사이클마다 `state/heartbeat.txt` 에 ISO 타임스탬프 + `data_age_sec` 를 atomic-rename 으로 기록.
- docker-compose healthcheck 를 해당 파일의 mtime 검사로 교체 (임계값 = 폴 주기의 3배 + 마진). 메인 루프가 진짜로 멈추면 healthcheck 가 실패 → `restart: always` 가 컨테이너를 재기동.
- 하트비트 로그 자체에도 `last_data_age` 필드 추가 → 사람 눈으로 즉시 데이터 흐름 상태 식별.

**3. Telegram 알림 도입 (`core/notifier.py`)**:
- Notifier 추상 클래스 + TelegramNotifier (rate-limited, retry, swallow-all-errors) + NullNotifier 폴백.
- EventBus 에 새 이벤트(`SYSTEM_STARTED`, `DATA_STALE`, `DATA_RECONNECTED`) 추가. 시작/종료/포지션 진입·청산/데이터 stale·복구/전략 에러를 Telegram 으로 송출.
- 메시지에 `[paper]` / `[live]` 모드 태그 자동 부착. 동일 플러그인이 실거래 모드에서 그대로 동작하므로, 실거래 전환 시 코드 변경 불필요.
- 알림 송출 자체가 거래 흐름을 차단하지 않도록 모든 예외를 내부에서 흡수.

**4. 운영 문서 (`DEPLOY_RECOVERY.md`)**:
- 서버에서 실행할 백업 → pull → 환경변수 → 재시작 → 검증 단계를 체크리스트화.
- 검증 단계마다 "기대 출력"을 명시해 사람이 결과를 판정 가능하도록 함.

EN:
**1. Data freshness guard (`main.py`)**:
- A monotonic `last_new_bar_wall` clock tracks "wall time of the most recent *new* bar" — advancement of bar timestamps, not just successful fetches.
- When the gap exceeds `stale_threshold_seconds`, publish a `DATA_STALE` event and force a data-source reconnect; on success, publish `DATA_RECONNECTED`.
- Original "consecutive-exception" trigger retained; the new freshness trigger runs alongside to also catch silent failures.

**2. Heartbeat file + healthcheck correction**:
- Each poll cycle atomic-rewrites `state/heartbeat.txt` with an ISO timestamp + `data_age_sec`.
- docker-compose healthcheck switched to inspect that file's mtime (threshold = 3× poll interval + margin). When the loop genuinely stalls, healthcheck fails and `restart: always` recycles the container.
- Heartbeat log line itself now carries `last_data_age` — human eyes can spot a stalled data flow at a glance.

**3. Telegram notifier (`core/notifier.py`)**:
- Notifier abstract base + TelegramNotifier (rate-limited, retried, swallows all errors) + NullNotifier fallback.
- EventBus gained `SYSTEM_STARTED`, `DATA_STALE`, `DATA_RECONNECTED`. Startup, shutdown, position open/close, data stale/reconnect, and strategy-error events route to Telegram.
- Messages auto-tag `[paper]` / `[live]` from runtime mode. The same plugin works under live trading without code changes.
- Notification delivery is fenced from the trading loop — every exception is logged and swallowed.

**4. Operations doc (`DEPLOY_RECOVERY.md`)**:
- A server-side checklist: backup → pull → env → restart → verify. Each verify step states the expected output so a human can decide pass/fail.

---

### 결과 및 배운 점 (Result & Learnings)

KO:
1. **"살아있다"의 정의는 다층적**: 프로세스가 죽지 않은 것, 메인 루프가 도는 것, 데이터가 흐르는 것은 모두 다른 사건. Healthcheck 와 알림은 가장 외곽층(데이터 흐름)을 보아야 의미가 있음. 안쪽 층(프로세스 생존)만 보면 "살아있는데 일을 안 하는" 상태가 안 잡힘.

2. **예외 기반 관측의 한계**: 분산 시스템의 절반 이상의 실패는 예외를 던지지 않는다. 신선도(freshness)나 진전(progress) 같은 양의 지표를 별도 채널로 가져야 함.

3. **WAL 파일 주의**: SQLite WAL 모드에서 본체 DB 파일의 mtime 은 거의 의미 없는 신호. 라이브니스 신호로 채택할 때 매번 점검 필요.

4. **알림은 거래 흐름과 절연(isolated)되어야 함**: Telegram 송출 실패가 트레이딩을 막아서는 안 됨. 모든 외부 I/O 는 swallow-all-errors 로 감쌌고, 거래 결정은 알림 결과를 기다리지 않음.

5. **재발 방지의 한계**: 본 수정으로 동일 실패 모드는 잡히지만, 다른 종류의 장애(API 스키마 변경, 디스크 가득참, EC2 장애)는 별도로 다뤄야 함. "이 문제는 안 일어남" 과 "어떤 문제도 안 일어남" 은 다른 명제.

EN:
1. **"Alive" is layered**: Process not dead, main loop turning, and data flowing are three distinct events. A healthcheck or alert is only meaningful when it inspects the outermost layer (data flow). Watching only the inner layer (process liveness) misses the "alive but doing nothing" state.

2. **Exception-based observation is insufficient**: More than half of distributed-system failures don't raise. Liveness and progress signals must travel through a separate channel from error reporting.

3. **Beware of WAL files**: A SQLite WAL-mode main DB file's mtime is a near-meaningless liveness signal. Any liveness probe touching a DB file must be reviewed for this.

4. **Notifications must be isolated from trading**: A Telegram delivery failure must not block trades. All external I/O paths are wrapped in swallow-all-errors, and trade decisions never wait on notification results.

5. **Limits of "won't happen again"**: This fix prevents the specific failure mode but says nothing about API schema changes, disk-full, EC2 outages, or any other class of failure. "This particular problem won't recur" and "no problem will ever recur" are different claims.

---

## 2026-04-24: 페이퍼 트레이더 정지 사건 진단 — Hung 판정 → 재해석
## 2026-04-24: Paper Trader Stall Diagnosis — Initial Hung Verdict, Re-Interpreted

### 배경 (Background)

KO:
4월 21일 마지막 트레이드 이후 페이퍼 트레이더가 신호를 멈춤. 사용자가 사고를 인지하고 점검 요청.

EN:
After the last trade on April 21 the paper trader stopped producing signals. The user noticed and requested an investigation.

---

### 문제: Hung 여부 판정의 어려움 (Problem: Diagnosing Hung vs. Idle)

KO:
"신호가 안 나는 것"인지 "프로세스가 뻗은 것"인지 구분 불가. 외부 관찰만으로는 이벤트 루프가 도는지, 거래소 연결이 살아있는지, 모델이 단순히 임계 미만이라 침묵 중인지 결정할 수 없음. 1차 진단에서 컨텍스트 스위치 카운트가 20초 동안 변하지 않은 것을 근거로 "hung" 으로 결론지었으나, 이후 docker logs 를 확인하니 5분 주기 하트비트가 계속 찍혀 있음을 발견 — **메인 루프는 살아있고**, ctxt_switches 0 은 단지 epoll 대기의 정상 상태였음.

EN:
Could not tell from the outside whether "no signals" meant "model is silent below threshold" or "process is wedged". The first diagnosis pinned it as hung based on voluntary_ctxt_switches being constant for 20 seconds; once docker logs were inspected, the 5-minute heartbeat had been printing the entire time — **the loop was alive**, and the constant ctxt_switches reading was simply normal idle epoll behavior.

---

### 해결 방법 (Solution)

KO:
**1. 진단 절차 재정립**:
- 진단 1순위: docker logs (가장 결정적). ctxt_switches, wchan 같은 커널 신호는 보조 증거.
- ep_poll wchan 은 hung 의 근거가 아님 — asyncio 정상 대기 상태.
- 살아있는데 일 안 하는 케이스 vs 정말 죽은 케이스를 구분하려면 외부 관측 신호(하트비트 로그)가 필수.

**2. 결정적 증거 확보**:
- TCP 연결 0개 + 5일치 로그에서 fetch 관련 흔적 0건 → 데이터 파이프라인이 처음부터 관찰 불가능한 상태로 운영되어 왔음을 확인.
- 마지막 시그널 로그 시각, signals.jsonl 파일 mtime, healthcheck 실패 이력으로 사고 시간 윈도우 특정.

**3. 후속 조치 분리**:
- 즉시 재시작 vs 원인 분석 후 재시작 중에서 후자 선택. "재시작하면 증거 사라짐" 원칙.
- 동시에 "복구 전 보완책 설계" 를 진행해 다음 사이클에서 동일 사고 재발 방지.

EN:
**1. Re-prioritised the diagnostic procedure**:
- Top priority: docker logs (most decisive). Kernel signals like ctxt_switches and wchan are supporting evidence at best.
- ep_poll wchan is not evidence of hang — it's the normal idle state for asyncio loops.
- Distinguishing "alive but idle" from "actually dead" requires an external observable (heartbeat log).

**2. Established decisive evidence**:
- Zero TCP connections + zero fetch-related entries across five days of logs → the data pipeline had been operating without observability from day one.
- Last-signal timestamp, signals.jsonl mtime, and healthcheck failure history together pinned the incident window.

**3. Sequencing**:
- Chose "diagnose first, restart later" over "restart now". Restarting destroys evidence.
- In parallel, designed the prevention work so the next cycle would not repeat the same failure (see 2026-04-25 entry).

---

### 결과 및 배운 점 (Result & Learnings)

KO:
1. **로그가 1순위**: 컨텍스트 스위치, wchan, fd 목록 같은 커널 정보는 매력적이지만 해석 위험이 큼. 애플리케이션 로그가 가장 적은 오해를 부른다.

2. **초기 가설을 수정할 용기**: "hung" 으로 단정한 후 하트비트 로그를 보고 "데이터 파이프라인 사일런트 실패"로 가설을 갱신했음. 진단의 핵심은 가설 검증이지 가설 방어가 아님.

3. **재시작 = 증거 소실**: 분산 시스템 사고에서 "일단 재시작" 의 유혹은 강하지만, 같은 사고가 재발하면 두 번째도 같은 진단으로 돌아갈 수밖에 없음. 가능한 한 동결 상태에서 증거를 모은 뒤 재시작.

EN:
1. **Logs first**: Kernel-level data (ctxt switches, wchan, fd listing) is alluring but easy to misinterpret. Application logs lead to the fewest mistakes.

2. **Update the prior**: Wrongly called it "hung" first, then revised to "silent data pipeline failure" once heartbeats were checked. Diagnosis is hypothesis-testing, not hypothesis-defending.

3. **Restart = evidence destroyed**: The temptation to "just restart" is strong in any distributed-system incident, but if the same incident recurs you'll diagnose it the same way the second time. Freeze, gather, then restart.

---

## 2026-04-23 ~ 2026-04-24: BTC ML Retry v3 인프라 구축 + 다중 라운드 감사
## 2026-04-23 ~ 2026-04-24: BTC ML Retry v3 Infrastructure Build + Multi-Round Audit

### 배경 (Background)

KO:
Phase 1 검증에서 SOL 만 안정성 게이트를 통과하고 BTC/ETH 는 시드 표준편차 게이트에서 탈락. BTC 직접 ML 재시도를 결정하되, 단순한 파라미터 조정이 아닌 **재현성·통계 엄밀성·관찰성** 전반을 재설계. 사용자 정의: "과적합 = 과거 성과를 낙관적으로 신뢰. 미래 성과 증거 필수."

EN:
In Phase 1 only SOL passed the seed-stability gate; BTC and ETH failed on seed-stddev. Decided on a BTC retry that is not a parameter tweak but a redesign of **reproducibility, statistical rigor, and observability**. User definition: "Overfitting = optimistic trust in past performance. Future-performance evidence required."

---

### 문제 1: 단일 라운드 감사로는 결함이 잔존 (Problem 1: Defects Survive a Single Audit Pass)

KO:
초기 v3 설계 후 1차 감사에서 다수 결함을 잡았으나, 실제 코드 작성·테스트 후 2차 감사에서 추가 결함 발견. 감사를 단발성 이벤트가 아닌 라운드 기반 과정으로 운영해야 함이 명확해짐.

EN:
After the initial v3 design, the first audit found numerous defects, but a second audit conducted post-implementation surfaced additional defects. It became clear that audits must be run as multi-round cycles, not as a one-shot event.

---

### 문제 2: 모듈은 작성되었으나 파이프라인에 연결되지 않음 (Problem 2: Modules Implemented But Not Wired)

KO:
3차 감사 (르네상스/Citadel/HRT/Jump 의사결정자 관점) 결과 — triple-barrier 라벨링, DSR 평가, 시드 생성기, 정권 분리, 블록 부트스트랩 모듈은 모두 작성·단위테스트 통과 상태였으나 `run_experiment.py` 와 `trainer.py` 에서 호출되지 않음을 확인. 즉 "코드는 있지만 실험은 옛날 버전을 쓰는" 고립 상태.

EN:
The third audit (from the perspective of top quant-firm decision-makers) revealed that triple-barrier labeling, DSR evaluation, seed generator, regime split, and block bootstrap modules were all implemented and unit-tested, but neither `run_experiment.py` nor `trainer.py` invoked them. The code existed, but experiments were still running on the old machinery — orphan modules.

---

### 문제 3: 도메넌스 피처의 수학적 결함 (Problem 3: Mathematical Flaw in the Dominance Feature)

KO:
크로스에셋 피처 중 BTC 도메넌스 변화율을 사용 중이었으나, 데이터 수집기 구현이 `total_market_cap = btc_cap / current_dominance` 로 역산한 정적 상수에 의존. 결과적으로 `dominance` 피처가 `btc_cap` 의 단순 스케일링으로 환원되어 기존 BTC 가격 수익률과 수학적으로 동등한 신호가 됨. 신규 정보 0.

EN:
A BTC dominance feature was being added to the cross-asset set, but the data collector implementation derived `total_market_cap = btc_cap / current_dominance` (a static constant). This collapses the `dominance` feature into a scaled `btc_cap`, mathematically equivalent to existing BTC return signals. Zero new information.

---

### 해결 방법 (Solution)

KO:
**1. 라운드 기반 감사 프로토콜**:
- 1차: 설계 문서 단계에서 결함 탐색.
- 2차: 코드 작성 후 단위 테스트 결과 + 데이터 흐름 검토.
- 3차: 외부 시각(최고 퀀트펌 의사결정자 관점) 으로 사일런트 결함 탐색.
- 각 라운드 결과를 `research/experiments/btc_retry_log.md` 에 append-only 로 기록.

**2. 결함 우선순위 분류**:
- C(Critical, 실험 중단): 도메넌스 피처 결함, 모듈 미연결.
- M(Major, 별도 작업으로 추적): n_trials 처리, 시드 카운트 하드코딩, 라벨링 timeout 부호.
- m(Minor, 문서화): VIF 표준화, 캘린더 월 폴드 길이 분산.

**3. 신규 모듈 + 테스트 추가** (수치는 코드/설정 참조):
- triple-barrier 라벨링, DSR + 블록 부트스트랩, 정권 분리, 시드 생성기, 다중공선성 체크(상관·VIF).
- 단위 테스트 78건 통과. 사일런트 회귀 방지를 위한 회귀 테스트 추가.

**4. 실험 실행은 모듈 연결 후로 보류**:
- 모듈만 작성된 상태로 실험 실행 시 v2 결과로 오인할 위험. 연결 작업이 완료될 때까지 실험 동결.

EN:
**1. Round-based audit protocol**:
- Round 1: defect-hunt at design-document stage.
- Round 2: post-implementation review with unit-test results + data flow inspection.
- Round 3: external lens (top quant-firm decision-maker perspective) to surface silent defects.
- Append-only record of every round in `research/experiments/btc_retry_log.md`.

**2. Defect priority taxonomy**:
- C (Critical, halts the experiment): dominance feature flaw, modules not wired.
- M (Major, tracked as separate work items): n_trials handling, hard-coded seed count, labeling timeout sign.
- m (Minor, documented): VIF standardization, calendar-month fold length variance.

**3. New modules + tests** (numeric values live in code/config only):
- triple-barrier labeling, DSR + block bootstrap, regime split, seed generator, multicollinearity check (corr + VIF).
- 78 unit tests passing. Regression coverage added to prevent silent regressions.

**4. Experiments deferred until modules are wired**:
- Running experiments with the modules existing-but-disconnected risks misreading old-pipeline results as v3 results. Experiments are frozen until wiring is complete.

---

### 결과 및 배운 점 (Result & Learnings)

KO:
1. **다중 라운드 감사가 단일 라운드보다 압도적으로 우월**: 1차 → 2차 → 3차 사이에 잡힌 결함의 종류가 매번 달랐음. 작성자와 검토자의 시야각이 다르므로, 같은 사람이 시간차를 두거나 관점을 바꿔도 새 결함이 나옴.

2. **"테스트가 통과했다 ≠ 파이프라인이 동작한다"**: 단위 테스트는 모듈 격리 검증일 뿐. 통합 테스트가 없으면 모듈이 호출되지 않는 사일런트 디스커넥트도 충분히 발생.

3. **수학적 정합성 검증의 위치**: 피처 추가 시 데이터 수집기 구현까지 들여다봐야 함. 이름·의도가 옳다고 구현이 옳다는 보장이 없음. 도메넌스 결함은 단위 테스트로는 영원히 잡히지 않았을 것 — 데이터 수학적 정의를 따라가야 보임.

4. **외부 시각의 가치**: 자기 코드를 자기 시각으로만 검토하면 같은 사각지대가 유지됨. "다른 사람" 의 입장에서 다시 봐야 새 결함이 나타남.

EN:
1. **Multi-round audits crush single-round audits**: The classes of defects caught across rounds 1, 2, and 3 were different each time. Writer and reviewer have different fields of view; even the same person catches new things across temporal or perspective shifts.

2. **"Tests pass ≠ pipeline works"**: Unit tests verify modules in isolation. Without an integration test, an orphan module never invoked from the main pipeline can pass every unit test indefinitely.

3. **Where to check mathematical soundness**: When adding a feature, follow it down to the data collector. Correct intent doesn't guarantee correct implementation. The dominance defect would never have been caught by unit testing — only by tracing the math of the data definition.

4. **Value of an outside lens**: Reviewing your own code from your own perspective preserves the same blind spots. Forcing a "different person's" perspective surfaces fresh defects.

---

## 2026-04-19: Phase 2 Paper Forward Test 가동 + BTC Retry v3 Plan 확정
## 2026-04-19: Phase 2 Paper Forward Test Launch + BTC Retry v3 Plan Finalized

### 배경 (Background)

KO:
Phase 1 ML 리서치 완료 후 SOL 모델만 안정성 게이트 통과. AWS EC2 + Docker 로 SOL 페이퍼 포워드 테스트를 가동하여 미래 데이터 기반 검증 단계로 진입. 동시에 BTC 직접 ML 재시도를 위한 v3 plan 확정 — 핵심 패러다임은 "과거 lockbox 폐기, 페이퍼 포워드 테스트가 최종 lockbox" + "라벨링과 실행의 구조적 일치".

EN:
Phase 1 ML research complete; SOL is the only model that passed the stability gate. The SOL paper forward test went live on AWS EC2 + Docker, entering the future-data validation phase. In parallel, the BTC retry v3 plan was finalized — core paradigm shift: "abandon the past-data lockbox; the paper forward test itself is the final lockbox" + "labeling and execution must be structurally aligned".

---

### 문제 1: 과거 데이터 Lockbox 의 구조적 한계 (Problem 1: Structural Limits of Past-Data Lockbox)

KO:
Phase 1 가 이미 사용 가능한 과거 데이터의 마지막 구간을 lockbox 로 소진. 같은 구간을 BTC 재시도에 재사용하면 데이터 오염, 새로 수집하려면 6~12개월 대기. 결국 어떤 형태든 "과거를 낙관적으로 본" 결정에 의존하게 됨.

EN:
Phase 1 had already consumed the latest available block of historical data as its lockbox. Reusing it for the BTC retry would contaminate the data; collecting fresh data means 6–12 months of waiting. Either path forces an "optimistic-about-the-past" decision somewhere in the chain.

---

### 문제 2: ATR 기반 Fixed-Horizon 라벨링과 SL 실행의 부정합 (Problem 2: Mismatch Between ATR Fixed-Horizon Labeling and SL Execution)

KO:
실행 단계의 손절 로직과 학습 단계의 라벨링 정의가 서로 다른 사건을 모델링. 학습은 "고정 시간 후 수익 부호" 를 예측하지만 실거래는 "SL 터치 또는 timeout 까지의 결과" 로 마감 — 학습된 함수와 실행 결과 사이에 구조적 괴리 존재.

EN:
The execution-side stop-loss and the training-side labeling were modeling different events. Training predicted "return sign after a fixed horizon" but live trading closed on "SL touch or timeout" — a structural gap between the learned function and the executed outcome.

---

### 해결 방법 (Solution)

KO:
**1. Lockbox 정의 재구성**:
- 과거 데이터 lockbox 폐기. 페이퍼 포워드 테스트(미래 실데이터 흐름) 자체를 최종 lockbox 로 재정의.
- 사전 커밋: 게이트·기간·실패 시 fallback 을 plan 단계에서 명문화하여 사후 조정 유혹 차단.

**2. 라벨링과 실행 일치 (Triple-Barrier)**:
- Lopez de Prado 의 triple-barrier 라벨링 채택: TP / SL / vertical(timeout) 중 먼저 닿는 사건이 라벨.
- 실행 SL 과 라벨링 SL 의 정의가 일치 → 학습된 함수가 실거래에서 그대로 평가됨.

**3. 통계 엄밀성**:
- 시드 다수화로 안정성 신뢰구간 확보.
- FDR → DSR 로 전환 (Bailey & Lopez de Prado 2014). 다중 시도 + 비정규성 보정.
- 자기상관 보존을 위한 블록 부트스트랩으로 정권별 신뢰구간 산출.

**4. Walk-forward Rolling**:
- Expanding → Rolling 으로 전환. 폴드별 OOS 길이 동일하게 정렬해 통계 비교 왜곡 제거.
- Purge / embargo 가 horizon 과 lookback 에 결합되도록 동적 산출.

**5. Phase 2 (SOL Paper) 가동**:
- 6개월 + 6개월 2단계 페이퍼 검증으로 통계 유의성 점검. 단계별 sharpe / drawdown / 트레이드 수 사전 커밋.
- 통과 시 점진적 실거래(1% → 10% → 100%) 단계 명시.

EN:
**1. Redefining the lockbox**:
- The past-data lockbox is abandoned. The paper forward test (real future data flow) becomes the final lockbox.
- Precommit: gates, durations, and fallbacks are written into the plan to remove post-hoc adjustment temptation.

**2. Aligning labeling with execution (triple-barrier)**:
- Adopted Lopez de Prado's triple-barrier labeling: the first of TP / SL / vertical(timeout) sets the label.
- Execution SL definition and labeling SL definition match — the learned function is evaluated under the same regime it executes in.

**3. Statistical rigor**:
- Many-seed runs to establish a credible stability CI.
- FDR → DSR (Bailey & Lopez de Prado 2014). Multiple-trials + non-normality adjustment.
- Block bootstrap (autocorrelation-preserving) for per-regime confidence intervals.

**4. Walk-forward rolling**:
- Expanding → rolling. Equal OOS length per fold removes statistical-comparison distortion.
- Purge / embargo derived dynamically from horizon and lookback.

**5. Phase 2 (SOL paper) launch**:
- Two-stage 6-month + 6-month paper validation for statistical significance. Stage-wise Sharpe / drawdown / trade count precommitted.
- On pass, incremental live ramp (1% → 10% → 100%) explicitly defined.

---

### 결과 및 배운 점 (Result & Learnings)

KO:
1. **"Lockbox 는 미래에 있다"**: 과거 데이터로 OOS 를 재구성하려는 모든 시도는 결국 "과거를 낙관적으로 본" 결정에 닿음. 미래 데이터(페이퍼 포워드)는 자료가 충분히 쌓일 때까지 기다리기를 강제하므로 자기-기만 가능성이 본질적으로 작음.

2. **라벨링은 손익 정의의 일부**: 라벨링 함수와 실행 청산 조건이 다르면, 모델은 사실상 "다른 게임" 을 학습한 것. 두 정의의 일치는 알파의 정의를 일관되게 만드는 토대.

3. **사전 커밋의 가치**: 게이트와 fallback 을 결과를 보기 전에 명문화하면, 사후에 "이번만 예외" 유혹이 구조적으로 차단됨. 이 사전 커밋이 없으면 paper 결과가 borderline 일 때 결정자가 흔들림.

EN:
1. **"The lockbox lives in the future"**: Any attempt to manufacture OOS from past data eventually leans on an optimistic-about-the-past decision. Future data (paper forward) structurally enforces waiting until enough data exists, leaving little room for self-deception.

2. **Labeling is part of the P&L definition**: If the labeling function and the execution close condition disagree, the model has effectively been trained on a *different* game. Aligning the two is the foundation for a consistent alpha definition.

3. **Value of precommit**: Precommitting gates and fallbacks before results are seen structurally blocks "just this once" exceptions later. Without the precommit, a borderline paper result will shake the decision-maker every time.

---

## 사용된 참고 문서 (Reference Documents)

KO:
- Lopez de Prado, *Advances in Financial Machine Learning* (2018) — triple-barrier 라벨링.
- Bailey, D. H. & Lopez de Prado, M. (2014) *The Deflated Sharpe Ratio*.
- Politis, D. N. & Romano, J. P. (1992) *A Circular Block-Resampling Procedure*.

EN:
- Lopez de Prado, *Advances in Financial Machine Learning* (2018) — triple-barrier labeling.
- Bailey, D. H. & Lopez de Prado, M. (2014) *The Deflated Sharpe Ratio*.
- Politis, D. N. & Romano, J. P. (1992) *A Circular Block-Resampling Procedure*.
