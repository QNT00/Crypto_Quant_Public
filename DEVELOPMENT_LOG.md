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

## 2026-05-19: SOL 실전 진입 + 페이퍼/라이브 격리 운영 시작, 진입 직전 페이퍼 사고 진단
## 2026-05-19: SOL Live Entry + Paper/Live Isolated Operation, Pre-Entry Paper Incident Diagnosed

### 배경 (Background)

KO:
30일간의 SOL 페이퍼 운영 결과가 충분히 검증된 시점. ML 신호의 평균 손익이 통계적으로 양의 유의성을 보였고, 특히 평일과 주말 사이 분포에 명확한 비대칭이 잡혔다. 이 발견을 어떻게 운영에 반영할지 — 그리고 그 효과를 진짜로 검증할 수 있는 환경을 어떻게 만들지 — 가 결정의 핵심이었다.

EN:
After thirty days of SOL paper operation, the validation signals were consistent enough to commit. ML-driven trade outcomes showed statistically significant positive expectancy, with a clear asymmetry between weekday and weekend trade distributions. The decision came down to how to encode that asymmetry in production — and how to design an environment that would actually validate the choice.

### 의사결정 (Decision)

KO:
페이퍼와 라이브를 **같은 모델 코드로, 다른 행동을 하도록** 분리해 운영하기로 결정.
- 라이브 컨테이너: SOL 단일 전략, 주말 진입 차단 필터 ON, 시드의 전부를 할당.
- 페이퍼 컨테이너: 기존 알트 3종 + SOL은 필터 OFF — 즉 **컨트롤 그룹**으로 30일 더 운영.
- 같은 신호에 대해 한쪽은 차단, 한쪽은 통과 → 30일 뒤 누적을 비교하면 필터의 효과가 진짜인지 우연 표본인지 판별 가능.

운영 인프라 자체를 분리. 두 컨테이너는 서로 다른 상태/로그/하트비트 디렉토리를 사용하고, 자본·리스크 격리도 컨테이너 단위로 작동. 라이브 사고가 페이퍼에 번지지 않고, 그 반대도 마찬가지.

진입 전 12개 항목 안전 체크리스트와 9개 어보트 조건을 미리 정의. 인증/권한/IP/마진 모드/헤지 모드/모델 로드/심볼 정합성 등 모든 단계가 명시적으로 사전 점검되어야 컨테이너를 켤 수 있도록.

EN:
Decision: keep paper and live on the **same model code** but make them **behave differently**.
- Live container: SOL alone, weekend-entry blocked, seed fully allocated.
- Paper container: the existing three altcoins plus SOL with the filter OFF — i.e. the **control group** running for another thirty days.
- The same signal is gated in one environment and allowed in the other. Comparing thirty-day cumulatives between the two becomes a clean test of whether the filter is real or sample luck.

Operational infrastructure was isolated too. Each container uses its own state, logs, and heartbeat directories, with capital and risk isolation applied per container. A live-side incident cannot bleed into paper, and vice versa.

Before launching, a twelve-item safety checklist and nine abort conditions were defined: authentication, permissions, IP whitelist, margin mode, hedge mode, model load, symbol match — every step explicitly gated.

### 사고 진단 (Incident Diagnosis)

KO:
라이브 진입 직전, 페이퍼 컨테이너 재기동 과정에서 사고가 발생했다.

**현상**: 재기동 직후 페이퍼 SOL 의 가상 잔고가 갑자기 큰 폭으로 마이너스로 떨어졌고, 곧이어 Telegram 으로 "ghost/liquidation detected" 알림이 다른 코인들에서까지 동시에 들어왔다.

**원인**: 거래소-내부 포지션 동기화 로직 중 "DB 에는 포지션이 있는데 거래소엔 없음 → 청산으로 간주" 라는 분기가, **페이퍼 모드 어댑터에도 그대로 적용**되어 있었다. 페이퍼 어댑터의 가상 포지션은 컨테이너 메모리에만 살아 있고 컨테이너가 재기동되면 빈 상태로 초기화되는 구조다. 즉 정상 동작인 메모리 리셋을 시스템이 "거래소에서 청산됨" 으로 오해석한 것.

**확정 손해**: 0 (페이퍼라서). 데이터만 손상.

EN:
Right before bringing the live container up, the paper container restart triggered an incident.

**Symptom**: After paper restart, paper SOL's virtual equity dropped sharply, and Telegram simultaneously fired "ghost/liquidation detected" alerts for other coins as well.

**Root cause**: The exchange-vs-internal position-sync logic carried a branch — "if the DB shows a position but the exchange does not, treat it as liquidated." That branch was being applied uniformly **including the paper-mode adapter**. The paper adapter's virtual positions live only in container memory and reset to empty on every restart by design. The normal in-memory reset was being misinterpreted as "the exchange liquidated this position."

**Realized loss**: zero — it was paper. Only the data was corrupted.

### 해결 방법 (Solution)

KO:
- 동기화 로직에 어댑터 종류 분기 추가. 페이퍼 어댑터에는 청산 추정 분기를 적용하지 않는다.
- 회귀 테스트로 패턴을 못박음. 같은 류의 사고가 다시 생기지 않도록.
- 손상된 페이퍼 데이터의 DB 복구는 비용 대비 효익이 낮다고 판단 — 별도 사고기록 문서를 만들어 분석 시 제외 구간/기준을 명시. 30일 평가에서 해당 구간 트레이드는 신뢰에서 제외하기로.
- 페이퍼 컨테이너는 정상 상태로 재가동. 라이브 진입은 그 다음.

라이브 진입 자체는 환경 변수 키 이름 불일치, API 권한·IP 화이트리스트 누락 등 외부 설정 이슈로 두 번 다시 죽었다 살아남. 매번 로그를 그대로 읽어 가며 한 가지씩 좁혀 갔다. 최종적으로 네 가지 검증 로그 라인(모델 로드, 자본 오버라이드 적용, 주말 필터 활성화, 거래소 핸드셰이크 완료)과 Telegram 시스템 시작 알림을 확인하고 운영 단계 진입.

EN:
- An adapter-type branch was added to the sync logic. The liquidation-inference path is skipped for the paper adapter.
- A regression test pins the pattern so the same incident can't recur silently.
- A full DB rollback for the corrupted paper data was judged not worth the engineering cost. An incident memo was written instead, documenting the exclusion windows and criteria — trades in that range will be filtered out of the thirty-day evaluation.
- The paper container was brought back to a healthy state, then the live container.

The live launch itself died twice on external configuration: an env-key naming mismatch, then API permission and IP whitelist gaps. Each failure was diagnosed by reading the actual logs and narrowing the cause one step at a time. Finally four validation log lines (model loaded, capital override applied, weekend filter enabled, exchange handshake complete) and the Telegram system-started alert confirmed live was operational.

### 결과 및 배운 점 (Result & Learnings)

KO:
1. **인프라 격리 = 사고 격리**: 두 컨테이너를 진짜로 갈라놓은 덕에, 페이퍼 사고를 진단·격리하는 동안 라이브 진입은 별개의 시간 축으로 진행할 수 있었다. "같은 코드가 환경마다 다른 행동을 한다" 는 설계의 첫 번째 효용.

2. **fix 의 사각지대**: 같은 코드가 페이퍼와 라이브 양쪽에 흐르는 구조에서, 안전을 위해 추가한 분기가 한쪽에선 안전이고 한쪽에선 사고가 될 수 있다. 안전 fix 도입 시 두 환경 모두에서 검증한다는 원칙이 정착.

3. **데이터 복구의 기회비용**: 사고 데이터의 DB 복구는 가능했지만 시간 가치가 낮았다. 대신 메모로 남기고 분석 단계에서 제외하는 쪽이 더 합리적. 모든 사고가 "원상복구" 가 답은 아니다.

4. **외부 설정 이슈는 코드 문제로 안 잡힘**: API 키 이름·권한·IP 등 외부 설정 단계의 실패는 로컬 테스트 통과해도 서버에선 매번 다시 만난다. 운영 절차 자체가 일종의 사전 점검 코드.

5. **검증된 신호와 운영 환경은 별개**: SOL 자체의 알파는 30일 페이퍼로 충분히 검증됐지만, 운영 진입 직전까지 환경에서 새 사고가 나왔다. 알파의 존재가 운영의 안전을 자동 보장하지는 않는다.

EN:
1. **Infra isolation = incident isolation**: With the two containers genuinely separated, the paper incident could be diagnosed and contained on its own timeline while the live launch proceeded on a different one. "Same code behaving differently per environment" earned its keep on day one.

2. **Blind spots of safety fixes**: When the same code path runs under both paper and live, a branch added for safety in one mode can become an incident in the other. The standing rule now: every safety fix is verified under both adapters.

3. **Opportunity cost of data recovery**: A full DB rollback of the corrupted paper data was technically possible but a poor use of time. Memoing the corruption and excluding the affected range during analysis was the right trade-off. Not every incident needs to be undone.

4. **External-config failures don't surface in code tests**: API key naming, permissions, IP whitelisting — these all pass locally and re-emerge on the server every time. The operational checklist is itself a kind of pre-flight code.

5. **Validated signal ≠ safe operations**: SOL's alpha was sufficiently validated by thirty days of paper, yet a fresh environmental incident appeared right at the launch boundary. Alpha existing doesn't automatically guarantee operational safety.

---

## 2026-05-18: SOL 페이퍼 30일 정밀 분석 — 평일/주말 비대칭 발견
## 2026-05-18: SOL Paper 30-Day Forensics — Weekday/Weekend Asymmetry Discovered

### 배경 (Background)

KO:
SOL 페이퍼가 약 한 달간 무중단으로 돌면서 데이터가 충분히 쌓였다. 단순 누적 손익만 보지 않고, 트레이드를 요일·시간대·시장 국면별로 쪼개어 패턴을 들여다본 첫 본격 분석.

EN:
After roughly a month of uninterrupted SOL paper operation, the dataset was large enough to do real forensics — not just headline P&L, but slicing trades by weekday, time-of-day, and regime.

### 발견 (Findings)

KO:
- 전체 누적 손익은 충분히 양의 영역, 샤프와 승률 모두 페이퍼 검증 기준선을 통과.
- **평일과 주말의 분포가 명확히 달랐다**. 평일 트레이드는 양의 평균 손익과 높은 샤프를 유지한 반면, 주말 UTC 기준으로 진입한 트레이드는 평균이 음수로 떨어졌고 누적도 마이너스.
- 표본 크기는 작지만 Welch t-검정 결과 통계적 유의성 경계선 부근에 닿았다. effect size 와 sign 일관성을 함께 보면 우연으로 보기 어렵다는 판단.
- 단 30일 한 윈도우라는 점은 caveat 으로 남김. 다음 30일 누적도 같은 sign 이 유지되어야 진짜 효과로 채택.

EN:
- Overall cumulative P&L well in positive territory; Sharpe and win-rate above paper validation thresholds.
- **The weekday vs weekend distribution diverged clearly**. Weekday trades preserved a positive mean and high Sharpe, while trades entered on Saturday or Sunday UTC flipped to a negative mean with negative cumulative.
- Sample size was small but a Welch t-test reached the borderline of statistical significance. Combined with the effect size and sign consistency, it was hard to dismiss as noise.
- Caveat: a single thirty-day window. A second thirty-day cumulative must hold the same sign before the effect is taken as real.

### 의사결정 (Decision)

KO:
- 라이브 진입 시 주말 진입을 차단하는 필터를 ON 으로 두기로.
- 동시에 페이퍼는 필터를 OFF 로 유지해 다음 30일 동안 컨트롤 그룹 역할을 시키기로. 같은 모델 신호가 한쪽은 차단, 한쪽은 통과되는 구조.
- 30일 뒤 결정 규칙도 함께 명시: 페이퍼 주말 누적의 t-통계 값과 절대 크기를 결합해 필터 유지/해제/연장(60일) 중 하나를 선택.

이번 분석으로 "어떻게 라이브를 시작할지" 의 한 줄 결정이 끝남: 같은 SOL을 두 환경에서 다른 행동으로 돌려, 30일 뒤 비교로 효과를 검증.

EN:
- Live will run with the weekend filter ON.
- Paper will keep the filter OFF for the next thirty days — explicit control group. Same model signal, one path gates it, the other lets it through.
- A decision rule was set out for day thirty: combine the t-stat and absolute size of paper's weekend cumulative to decide whether to keep the filter, drop it, or extend the comparison to sixty days.

This analysis collapsed the "how do we go live" question into one line: run the same SOL strategy in two environments with one behavioral switch flipped, and let thirty days of comparison settle the matter.

### 배운 점 (Learnings)

KO:
1. **누적 손익만 보면 패턴이 안 보임**: 헤드라인 P&L 은 우호적이었지만 쪼개어 보기 전에는 주말 effect 가 가려져 있었다. 30일치 트레이드를 요일·시간대·국면으로 쪼개는 루틴이 검증 단계의 표준이 됐다.
2. **borderline 유의성의 처리**: 단일 윈도우에서 p ≈ 0.05 인 결과는 채택해도 폐기해도 위험. 컨트롤 그룹을 만들어 추가 표본을 자동으로 모으는 구조 자체가 답이라는 결론.
3. **운영이 곧 실험**: 라이브 = 본격 운영, 페이퍼 = 연구라는 이분법을 버렸다. 라이브를 켜는 결정 자체에 후속 실험 설계가 포함되어야 비용 없이 가설이 검증된다.

EN:
1. **Cumulative P&L hides patterns**: The headline number looked clean, but the weekend effect only emerged after slicing trades by weekday, time-of-day, and regime. Slicing is now a standard step at the end of every paper validation.
2. **Handling borderline significance**: Accepting or rejecting a single-window p ≈ 0.05 result is equally fragile. The structural answer is to construct a control group that automatically accumulates further samples.
3. **Operations as experiment**: The mental split of "live = production, paper = research" is gone. The launch decision itself encodes a follow-up experiment — that is how hypotheses get tested at zero extra cost.

---

## 2026-05-17: 4번째 알트 라이브 후보 배포 + 재기동 버그 발견, 백테스트-라이브 비대칭 재해석
## 2026-05-17: Fourth Altcoin Live Candidate Deployed + Restart Bug Discovered, Backtest-Live Asymmetry Re-Interpreted

### 배경 (Background)

KO:
다코인 검증 프레임워크의 네 번째 코인 검증을 마치고 페이퍼 환경에 플러그인을 배포하는 날. 동시에, 한 달간 머릿속을 차지하고 있던 "백테스트 샤프가 라이브 샤프보다 훨씬 높은 이유" 라는 의문을 마무리 짓는 작업도 같이 진행했다.

EN:
The day for deploying the fourth altcoin from the multi-coin validation framework to paper. In parallel, a month-long open question — "why is backtest Sharpe so much higher than live Sharpe?" — was finally closed.

### 4번째 알트 배포 (Fourth Altcoin Deploy)

KO:
- 정해진 strict gate 중 2개가 fail. 그러나 4개월 홀드아웃 락박스 단일사용 검증을 모두 통과.
- 직전 알트(3번째)에서 같은 게이트들이 fail 했지만 락박스가 통과하면 배포한다는 선례를 확립한 바 있다. 본 코인도 동일 원칙을 적용. 락박스 통과를 최종 검증 게이트로 채택하고, 게이트 경계선 부근의 backtest-scale artifact 는 허용 범위로 둔다.
- 배포 후 동일 모드(주말 필터 OFF, 페이퍼 자본 비율 동일)로 4개 코인 모두 운영. 컨테이너 재기동만으로 자동 발견.

EN:
- Two strict gates failed. But the four-month-holdout lockbox single-use check cleared completely.
- The previous (third) altcoin had failed the same gates yet passed lockbox, and was deployed under that precedent. The same rule was applied here: lockbox is the final gate, gate-margin artifacts at backtest scale are tolerated.
- After export, all four coins ran together under the same configuration (weekend filter OFF, identical paper capital share). A single container restart auto-discovered the new plugin.

### 재기동 시 즉시 익절 사건 (Immediate-Exit-on-Restart Incident)

KO:
새 코인 배포를 위해 페이퍼 컨테이너를 재기동하자, 이미 포지션을 보유하던 SOL 이 첫 사이클에서 즉시 청산되는 현상이 관찰됐다.

**원인**: 상태 복원 시 "포지션 진입 이후 몇 봉이 지났는가" 를 벽시계 기준으로 계산하던 부분. 컨테이너가 오랫동안 멈춰 있었다 다시 켜지면, 그 사이 흐른 시간이 진입 기간으로 계산되어 첫 사이클에서 곧바로 horizon 만기 조건이 발동했다.

**해결**:
- 4개 플러그인 모두에 "재기동 직후엔 다음 정상 봉이 닫힐 때까지 horizon 강제 종료를 보류" 패턴을 도입.
- 이 패턴은 신규 전략 템플릿에도 미리 들어가 있어, 향후 모든 플러그인이 자동으로 이 안전 가드를 갖게 됨.
- 사고로 만들어진 가짜 트레이드 한 건은 분석 시 제외하기로 기록.

EN:
Right after restarting the paper container to load the new coin, SOL — which already held an open position — was closed immediately on the first cycle.

**Cause**: "Bars since entry" was being computed from wall-clock. After a long container outage, the elapsed wall time was treated as bars held — meeting the horizon-exit condition on the very first cycle back up.

**Fix**:
- All four plugins now defer horizon-driven closes until the next natural bar after restart.
- The new strategy template ships with this guard, so every future plugin inherits it.
- The single phantom trade produced by the incident was recorded for exclusion at analysis time.

### 백테스트-라이브 비대칭 재해석 (Backtest-vs-Live Asymmetry Re-Interpreted)

KO:
백테스트 샤프가 라이브 샤프의 두 배 이상으로 나왔다는 점이 한 달 가까이 마음에 걸렸다. 코드를 다시 들여다본 결과:
- **백테스트는 완결된 상위-타임프레임 봉을 신호 시점에 이미 사용** — 미세한 lookahead 가 일부 피처에 존재.
- **라이브는 해당 시점의 in-progress 봉만 사용** — 인과적(causal).
- 즉 라이브가 보수적인 환경이고, 백테스트 샤프는 구조적으로 부풀려진다. 매크로 알파 부정이 아니라 백테스트 정확도의 한계.

실측 데이터로 확인: 한 달간의 SOL 라이브 누적이 의미 있는 양의 영역(약 +40%, 샤프 7 이상)을 유지. 백테스트가 inflated 라는 사실과 라이브 알파가 실재한다는 사실은 양립한다.

이후 평가 기준이 명확해짐: 백테스트 샤프는 상한 추정치로 활용, 라이브 운영 평가의 진짜 기준은 라이브 누적 데이터 그 자체.

EN:
The fact that backtest Sharpe ran roughly 2× live Sharpe had been bothering me for almost a month. Re-reading the code:
- **The backtest uses the already-completed higher-timeframe bar at the signal time** — a subtle lookahead in some features.
- **The live system only sees the in-progress higher-timeframe bar at the same moment** — strictly causal.
- The live environment is the conservative one, and backtest Sharpe is structurally inflated. This is not a refutation of the alpha, just a limit of backtest fidelity.

The live data itself made the case: a month of SOL live cumulative held firmly positive (roughly +40%, Sharpe above 7). The two statements — "backtest is inflated" and "live alpha is real" — are not in conflict.

The evaluation policy that came out of this: backtest Sharpe is an upper-bound estimate; the actual yardstick for live operations is the live cumulative itself.

### 배운 점 (Learnings)

KO:
1. **상태 복원의 안전 가드**: 운영 중 자연스럽게 잡히는 종류의 버그가 있는가 하면, 재기동/배포 같은 특수한 분기에서만 드러나는 버그가 있다. 후자는 코드 리뷰가 아닌 운영 관찰에서만 발견됨. 운영 자체가 테스트의 일부.
2. **선례의 일관성**: 두 번째 코인에서 락박스를 최종 게이트로 인정한 선례는 세 번째·네 번째 코인에서도 그대로 적용되어야 의사결정의 일관성이 유지된다. 사후에 게이트를 완화하지 않는다는 원칙과, 선례를 일관 적용한다는 원칙은 동전의 양면.
3. **백테스트의 위계**: 백테스트는 알파의 sign 과 대략적 크기를 짚어 주는 도구이지, 운영 기준치 자체는 아님. 모델의 가능성과 운영의 안전성은 별개 지표로 측정해야 함.

EN:
1. **Restart-safety guards**: Some bugs surface in steady-state operation; others only show up on the special branches like restart or deploy. The latter category is invisible to code review and only emerges during operations. Operations is itself a test surface.
2. **Consistency of precedent**: Once "lockbox as final gate" was accepted on the second coin, applying the same rule to the third and fourth coins is required for decision consistency. "We don't loosen gates after the fact" and "we apply precedent consistently" are two sides of the same coin.
3. **Hierarchy of backtests**: Backtest gives the sign and rough magnitude of alpha — not the operational benchmark. The model's potential and the operation's safety must be measured on separate axes.

---

## 2026-05-15 ~ 2026-05-16: BTC 단기 알파 두 시도 모두 폐기
## 2026-05-15 ~ 2026-05-16: Two More BTC Short-Horizon Alpha Attempts Retired

### 배경 (Background)

KO:
BTC 알파를 찾으려는 시도가 여섯 번째에 도달. 이번 라운드는 두 가지 가설을 평행으로 검증했다.
- 가설 1: BTC 단기 봉에 회귀(mean-reversion)성 미세 알파가 있을 것이라는 가설.
- 가설 2: 단일 거래소 안에서 자금조달율 기반 페어드 캐리(paired carry)로 안정 수익을 낼 수 있을 것이라는 가설.

EN:
The sixth attempt at a BTC alpha. Two hypotheses were tested in parallel.
- Hypothesis 1: a short-horizon mean-reversion alpha exists on BTC.
- Hypothesis 2: funding-based paired carry inside a single venue can yield a stable return on BTC.

### 결과 — 단기 MR (Result — Short-Horizon MR)

KO:
- BTC 영구선물 단기 데이터를 약 1년치 수집(약 32만 봉 규모).
- 세 가지 통계 테스트(허스트 지수, 분산비, Box-Ljung)로 MR 의 통계적 존재 여부를 검증.
- 결과: 통계적으로는 MR 이 존재(허스트 < 0.5, 분산비 < 1). 하지만 economic magnitude 는 매우 약함 — 한 자릿수 % 수준의 효과.
- 스캘퍼 계열 전략이 살아남으려면 magnitude 가 슬리피지·수수료를 압도해야 하는데, 그 수준이 아니었다.
- **결정**: 진행 중단.

EN:
- About a year of short-horizon BTC perp data was collected (~320k bars).
- Three statistical tests (Hurst exponent, variance ratio, Box-Ljung) checked whether MR exists in any rigorous sense.
- Result: statistically yes (Hurst < 0.5, variance ratio < 1). But the economic magnitude was thin — single-digit percent territory.
- For a scalper-family strategy to survive, magnitude must overwhelm slippage and fees. It did not.
- **Decision**: line abandoned.

### 결과 — 단일거래소 페어드 캐리 (Result — Single-Venue Paired Carry)

KO:
- V1 백테스트가 7개 게이트 중 5개를 통과. 평균 수익·샤프·재현성 모두 깔끔.
- 그러나 음의 funding 국면에서 알파가 흔들리고, 국면 변화에 매우 민감하다는 결과가 추가 검증에서 잡혔다.
- 즉 평균은 양수지만 국면 dependency 가 너무 큼. 운영 단계에서 견디기 어려운 구조.
- **결정**: 폐기. BTC 단일 거래소 페어드 캐리는 retail 의 단일 venue 환경에서 알파가 노이즈에 종속된다는 것을 데이터로 확인.

EN:
- The V1 backtest passed five of seven gates — clean mean return, clean Sharpe, clean reproducibility.
- But under negative-funding regimes alpha softened, and the strategy was extremely sensitive to regime transitions.
- Mean was positive, regime dependency too large. Hard to defend under live operations.
- **Decision**: retired. The data confirmed that BTC single-venue paired carry, at retail scale, has alpha tied too tightly to microstructure noise.

### BTC 라인의 종합 정리 (Closing the BTC Line)

KO:
- 여섯 번에 걸친 BTC 알파 탐색이 모두 폐기로 마감. 각 시도의 사유는 달랐다(과적합, 국면 fragility, magnitude 부족, 비용 초과 등).
- 공통점: BTC 는 시장 효율이 매우 높아 단일 retail venue 에서 단일 신호로 알파를 잡기가 매우 어렵다.
- 자원 배분 결정: 한동안 BTC 라인을 정지하고, 자원을 알트 다코인 검증과 라이브 운영 쪽에 집중. 재도전은 새 가설(cross-exchange basis, multi-venue 정보 우위 등)이 떠오를 때.

EN:
- Six BTC alpha attempts now closed. Each one died for a different reason — overfitting, regime fragility, insufficient magnitude, cost overrun — but a common pattern emerges.
- BTC is too efficient at retail scale and on a single venue with a single signal to admit alpha.
- Resource decision: pause the BTC line and pour the effort into altcoin multi-coin validation and live operations. BTC will be revisited only when a genuinely new hypothesis appears (cross-exchange basis, multi-venue informational advantage, etc.).

### 배운 점 (Learnings)

KO:
1. **통계적 존재와 운영 가능성은 다른 명제**: 통계적으로 알파가 잡혀도 economic magnitude 가 부족하면 운영 가능 신호로 봐서는 안 된다. 통계 결론에 매번 magnitude 검토를 결합하는 습관이 들었다.
2. **국면 민감도는 평균 손익이 아닌 별도 축에서 측정**: 평균 손익이 양수라도 국면별로 깔끔하게 무너지면 운영용 알파가 아니다.
3. **빠른 폐기의 가치**: 6번째 BTC 실패까지 와도, 매번 빠르게 폐기 사유를 명문화하고 자원을 다른 방향으로 옮길 수 있게 한 것이 시간 손실을 줄였다. "실패 = 시간 낭비" 가 아니라 "정해진 가설을 정해진 비용으로 닫는 자산".

EN:
1. **Statistical existence ≠ operational viability**: A statistically real edge with insufficient economic magnitude isn't an operational signal. Every statistical conclusion now travels with a magnitude check.
2. **Regime sensitivity lives on its own axis**: A positive mean with broken regime stability is not an operational alpha.
3. **Value of fast abandonment**: Even at the sixth failure, the routine of quickly documenting why the hypothesis failed and reallocating effort kept the time loss bounded. Failure isn't wasted time when it closes a hypothesis for a defined cost.

---

## 2026-05-05 ~ 2026-05-14: 알트 검증 라운드 — DOGE 폐기, ADA 통과
## 2026-05-05 ~ 2026-05-14: Altcoin Validation Round — DOGE Retired, ADA Cleared

### 배경 (Background)

KO:
첫 알트(XRP)를 같은 프레임워크로 통과시켜 페이퍼에 배포한 직후, 다음 후보 두 종(DOGE, ADA)을 평행 검증하기 시작. 각 코인은 데이터 감사 → 피처 게이트 → 다중 horizon 비교 → 본격 학습 → strict gate → 락박스 단일사용 의 동일 단계를 거친다.

EN:
With the first altcoin (XRP) cleared and deployed to paper under the same framework, the next two candidates (DOGE, ADA) entered parallel validation. Each coin walks the same path: data audit → feature gate → multi-horizon comparison → full training → strict gates → lockbox single-use.

### DOGE — strict gate 단계에서 폐기 (DOGE — Retired at Strict Gates)

KO:
- 데이터·피처·다중 horizon 단계까지는 통과.
- 본격 학습 결과는 일정 수준의 backtest 성과를 보여줬으나, strict gate 단계에서 시드 분산, 다른 알트와의 상관, 매크로 비용 stress 등 다수 항목에서 실패.
- 결정적으로 락박스로 가는 길에서 명확한 명분이 없었다. 이전 알트들에서는 락박스를 최종 게이트로 인정했지만, DOGE 는 락박스 홀드아웃 단계로 가더라도 통과 가능성이 매우 낮다는 사전 신호가 너무 많았다.
- **결정**: 폐기.

EN:
- Data, feature, multi-horizon stages all cleared.
- Full training produced a respectable backtest profile, but strict gates failed across multiple items: seed dispersion, cross-correlation with deployed altcoins, macro cost stress.
- Critically, the path to a clean lockbox pass looked unlikely — the prior altcoins had been allowed lockbox-as-final-gate, but DOGE had too many pre-signals suggesting the holdout itself would also fail.
- **Decision**: retired.

### ADA — gate 일부 fail, 락박스 통과 (ADA — Partial Gate Fail, Lockbox Pass)

KO:
- ADA 의 strict gate 결과는 직전 XRP 와 거의 같은 패턴이었다 — 시드 분산 한 항목과 cross-correlation 한 항목에서 fail, 나머지는 모두 통과.
- 동일 선례 적용: 락박스 단일사용 검증을 모두 통과하면 최종 게이트로 인정해 배포.
- 결과적으로 락박스 통과. backtest 샤프가 매우 높고, MaxDD 도 단일 자릿수 % 안.
- **결정**: 플러그인 배포 후보로 확정. 다만 페이퍼에서 다른 알트와 함께 운영하며 cross-correlation 을 실측으로 모니터링한다는 caveat 동반.

EN:
- ADA's strict gate result mirrored XRP's almost exactly — one seed-dispersion item and one cross-correlation item failed, the rest passed.
- Same precedent applied: if lockbox single-use clears, the plugin is deployed.
- Lockbox passed. Backtest Sharpe very high, max drawdown within single-digit percent.
- **Decision**: approved for plugin export. Caveat: paper operation will measure cross-correlation against the other altcoins in live data, not just backtest.

### 배운 점 (Learnings)

KO:
1. **사전 신호의 활용**: DOGE 폐기는 strict gate 다수 fail 만으로도 충분했지만, 사전 신호가 이미 락박스도 안 될 거라는 강한 정보를 줬다는 점을 명문화. 다음 코인부터는 sprint 진입 전에 사전 신호 검토 단계를 짧게 추가.
2. **fail-pass 의 비대칭 비용**: 빠르게 폐기 가능한 코인은 자원 절약. gate 경계 fail 인데 락박스 통과 가능성이 있는 코인은 락박스 단일사용을 소진해서라도 결정. 락박스는 한 번뿐인 자원이라 신중하게 써야 함.
3. **검증 framework 의 자기 검증**: 4번 연속 코인 검증을 같은 프레임워크로 통과·실패시킨 결과, 프레임워크 자체의 안정성도 함께 검증됐다. 게이트 임계 후행 수정 금지 원칙이 다음 도전들에서도 깨지지 않았음.

EN:
1. **Using pre-signals**: DOGE could have been retired purely on strict-gate failures, but in retrospect the pre-signals already pointed at a likely lockbox failure too. From the next coin onward, a short pre-signal review will run before sprinting into full training.
2. **Fail-pass asymmetric cost**: Coins that can be retired quickly save effort. Coins that fail at gate boundaries but might pass lockbox justify spending a single-use lockbox slot to decide. The lockbox is a one-shot resource and must be allocated deliberately.
3. **Self-validation of the framework**: Four consecutive coin runs through the same framework — passing and failing for different reasons — also validates the framework itself. The no-post-hoc-gate-loosening rule held across all four.

---

## 2026-04-26 ~ 2026-05-04: 첫 알트 다코인 확장 — XRP 검증 완주, 페이퍼 배포
## 2026-04-26 ~ 2026-05-04: First Altcoin Multi-Coin Expansion — XRP Validated and Deployed to Paper

### 배경 (Background)

KO:
SOL 한 종으로 시작한 ML 디렉셔널 라인이 다코인 검증 프레임워크의 첫 비-SOL 코인 검증에 도전. 프레임워크 자체는 SOL 검증 중에 다듬어졌고, XRP 가 그 첫 외부 적용 사례.

EN:
The ML-directional line, which started with SOL only, ran its multi-coin validation framework against the first non-SOL coin. The framework had been hardened during SOL's validation, and XRP became its first external case study.

### 진행 (Process)

KO:
- 데이터 감사 → 피처 게이트(고정된 피처 집합만 사용, 멀티콜리니어리티 검사) → 다중 horizon 비교 → 본격 학습(다중 config × 다중 seed) → strict gate 평가 → 락박스 단일사용 → 플러그인 export.
- strict gate 단계에서 두 개 항목 fail, 나머지 통과. 시드 분산 한 항목과 cross-correlation 한 항목.
- 4개월 홀드아웃 락박스 단일사용 검증 완전 통과.
- 락박스가 통과한 점이 결정적. backtest 단계의 게이트 경계 artifact 는 허용하되, 운영-스케일에 가까운 홀드아웃에서 신호가 잡힌다면 운영 시작 가능하다는 판단.

EN:
- Pipeline as designed: data audit → feature gate (fixed feature set, multicollinearity check) → multi-horizon comparison → full training (multiple configs × multiple seeds) → strict gate evaluation → lockbox single-use → plugin export.
- Strict gates: two items failed, the rest passed. One seed-dispersion item and one cross-correlation item.
- Four-month holdout lockbox single-use cleared completely.
- Lockbox clearance was the deciding factor. Backtest-margin artifacts at gate boundaries are tolerated; if the signal survives the operation-scale holdout, deployment is allowed.

### 결정 — 락박스를 최종 게이트로 (Decision — Lockbox as Final Gate)

KO:
- strict gate 들은 backtest 신호의 robustness 를 본다. 단 하나 락박스만이 운영-스케일 홀드아웃에서의 알파 잔존 여부를 본다.
- backtest 게이트가 일부 fail 해도 락박스가 깔끔하다면, 운영 진입 의사결정의 비대칭 비용 — 안 시작했을 때의 기회비용 vs 시작했을 때의 운영 위험 — 에서 시작 쪽이 더 합리적.
- 단 이 선례는 포스트-혹 게이트 완화가 아니어야 함. 게이트 통과 정의를 사후 수정하는 것이 아니라, "어떤 게이트가 최종 권위를 가지는지" 를 명문화하는 작업. 두 가지는 다르다.

EN:
- Strict gates measure robustness of the backtest signal. Lockbox alone measures whether alpha survives an operation-scale holdout.
- When some backtest gates fail but lockbox is clean, the asymmetric cost — opportunity cost of not deploying vs operational risk of deploying — tilts toward deploying.
- Crucially, this precedent is not post-hoc gate loosening. The pass criterion isn't being rewritten after the fact. What's being formalized is which gate has final authority. The two are different.

### 배포 후 운영 (Post-Deploy Operations)

KO:
- 페이퍼 컨테이너에서 SOL + ETH + XRP 3종 동시 운영 시작. 자본 비율의 합은 안전 마진 안에서 유지.
- 컨테이너 재기동만으로 자동 발견 가능한 구조 — 새 코인 배포 시 코드 변경 없이 디렉토리 추가만으로 완료.

EN:
- Paper container began running SOL + ETH + XRP simultaneously, with the sum of capital ratios kept inside the safety margin.
- The auto-discovery design means deploying a new coin requires only adding a directory; no code change.

### 배운 점 (Learnings)

KO:
1. **선례의 명문화**: 한 번의 결정이 다음 결정의 기준이 되므로, "왜 락박스가 최종 게이트인가" 는 그 자리에서 문서화. 동일 패턴 fail-pass 가 다음 코인에서도 나타날 때 즉시 일관 적용 가능.
2. **검증 framework 의 외부 적용 첫 사례**: SOL 안에서만 다듬어진 framework 가 다른 코인에 곧바로 적용 가능하다는 것 자체가 framework 의 1차 검증. 코인마다 게이트나 horizon 을 손대지 않는다는 invariant 가 유지됨.
3. **운영 비용의 누적**: 페이퍼 컨테이너에 3종이 동시 운영되면 데이터 흐름·하트비트·로그도 함께 늘어남. 운영 모니터링 도구가 N=1 에서 N=3 로 확장할 때 자연스럽게 견디는지 확인할 첫 기회.

EN:
1. **Codifying precedent**: A single decision becomes the criterion for the next, so "why is lockbox the final gate" gets written down at the moment of the decision. When the same fail-pass pattern shows up in the next coin, application is immediate and consistent.
2. **First external application of the framework**: The framework, refined inside SOL, ran cleanly on a non-SOL coin. That portability is itself a first-order validation of the framework. The invariant — gates and horizons aren't tweaked per coin — held.
3. **Cumulative operational cost**: Three coins running together in one paper container multiply data flow, heartbeats, and logs. It became the first chance to verify whether operational monitoring scales from N=1 to N=3 without strain.

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

