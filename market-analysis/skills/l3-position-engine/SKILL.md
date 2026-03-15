---
name: l3-position-engine
description: |
  Medallion Fund 스타일의 퀀트 트레이딩 분석 프레임워크 스킬.
  통계적 이상(statistical anomaly) 탐지, regime 판정, mean reversion 후보 스크리닝,
  Kelly Criterion 포지션 사이징, 리스크 관리를 체계적으로 수행한다.
  사용자가 "퀀트 분석해줘", "stat arb 기회 있어?", "mean reversion 후보 찾아줘",
  "시장 regime 뭐야?", "포지션 크기 계산해줘", "Kelly 적용해줘",
  "페어 트레이딩 분석", "z-score 체크", "변동성 regime", "통계적 차익거래",
  "백테스트 검증해줘", "오버피팅 체크", "리스크 관리 점검",
  "레버리지 어떻게 잡아?", "상관관계 분석", "포트폴리오 최적화",
  "퀀트 전략 설계", "시스템 트레이딩", "알고리즘 트레이딩 분석",
  등 퀀트/통계 기반 트레이딩 분석을 요청할 때 반드시 사용.
  단순 뉴스 요약, 재무제표 분석, 펀더멘털 밸류에이션, 자금 흐름(flow) 분석에는 트리거하지 않는다.
  자금 흐름 분석은 L2-capital-flow 스킬의 영역이다.
  이 스킬은 L2-capital-flow와 보완 관계로, L2가 "돈이 어디로 가는가"를 알려주면
  이 스킬이 "그 안에서 어떤 통계적 기회가 있고 얼마나 베팅할 것인가"를 결정한다.
---

# 퀀트 전략 프레임워크 (Quant Strategy Framework)

## 핵심 철학

> **"We're right 50.75% of the time, but we're 100% right 50.75% of the time."**
> — Robert Mercer

이 스킬은 세 가지 원칙을 따른다:

1. **SSoT = 통계적 비효율성(Statistical Anomaly)**
   - 가격 자체가 노이즈가 아니라, 가격 속에 숨겨진 통계적 패턴이 신호다
   - 뉴스, 전문가 의견, 감정이 아닌 데이터의 통계적 성질만 판단 근거로 삼는다

2. **모델은 현실의 근사치일 뿐이다**
   - "우리 모델이 현실을 반영한다고 믿지 않았다 — 현실의 일부 측면만 반영한다" (Nick Patterson)
   - 모든 판정에 불확실성 수준을 명시한다. 확신적 예측을 하지 않는다

3. **단일 포지션이 아닌 시스템이 수익을 만든다**
   - 개별 거래의 승률은 50.75%면 충분하다. 핵심은 반복과 분산이다
   - 하나의 "대박 아이디어"에 올인하는 접근을 경계한다

---

## Claude의 능력 경계

이 스킬을 사용할 때 Claude가 할 수 있는 것과 없는 것을 명확히 구분한다.

**Claude가 할 수 있는 것 — 적극적으로 수행:**
- 웹 검색으로 변동성, 스프레드, 상관관계 데이터를 수집하여 regime 판정
- Mean reversion 후보 쌍의 논리적 스크리닝과 이격도(Z-score) 개념 적용
- 사용자가 제공한 edge/variance로 Kelly 포지션 크기 계산
- Backtesting 체크리스트로 사용자의 전략 아이디어를 검증
- 리스크 관리 원칙 적용하여 포지션 한도와 경고 제시
- 퀀트 개념 설명 및 전략 설계 가이드

**Claude가 할 수 없는 것 — 솔직히 안내:**
- 실시간 틱/5분봉 데이터로 HMM 파라미터 추정 (Baum-Welch)
- 수천 개 포지션의 공분산 행렬 실시간 계산
- 자동화된 주문 실행 또는 실시간 거래
- Kernel regression 등 비선형 모델의 실시간 피팅

할 수 없는 영역이 요청되면: "이 부분은 Python(statsmodels, hmmlearn) / QuantLib 등으로 직접 구현이 필요합니다. 구현 가이드를 제공해 드릴까요?"로 안내한다.

---

## L1 → L2 → L3 파이프라인 연계 프로토콜

L3는 파이프라인의 최종 단계다. L1이 "계절"을, L2가 "바람 방향"을 제공하면, L3는 그 안에서 "정확히 어디에 얼마를 베팅할지"를 계산한다.

### 상위 스킬 입력 수용

**L1 INVESTCON → L3 Regime 보정:**

| L1 INVESTCON | L3 Regime 보정 | 효과 |
|---------------|----------------|------|
| INVESTCON 2 (강한 방어) | Regime을 최소 🟡 이상으로 강제 상향 | Kelly fraction 자동 축소, 신규 진입 제한 |
| INVESTCON 3 (방어적) | Regime이 🟢이어도 Kelly를 75%로 제한 | 기술적으로 Normal이어도 구조적 경계 반영 |
| INVESTCON 4 (균형) | 보정 없음 | L3 독립 판정 그대로 |
| INVESTCON 5-6 (공격) | Regime이 🟡이어도 Kelly를 50%까지 허용 | 구조적 기회 구간에서 과도한 보수성 방지 |

**L2 Flow 방향 → L3 후보 스크리닝 집중:**

L2가 자금 유입 섹터/자산을 식별했으면, L3 STEP 3에서 해당 영역을 우선 스크리닝한다.

| L2 신호 | L3 적용 |
|---------|---------|
| "자금이 에너지 섹터로 유입" | STEP 3에서 에너지 쌍(XLE/XOP, USO/BNO) 우선 분석 |
| "EM에서 DM으로 자금 이동" | STEP 3에서 국가 쌍(EEM/EFA, EWJ/EWY) 이격 스크리닝 |
| "채권에서 주식으로 로테이션" | STEP 3에서 주식-채권 쌍(SPY/TLT) 모멘텀 분석 |
| "특정 섹터 급격한 유출" | 해당 섹터에서 mean reversion 후보 탐색 (과매도 반등) |

**불일치 처리:**
L1 INVESTCON과 L3 자체 Regime이 불일치하면(예: L1은 INVESTCON 3인데 L3는 🟢 Normal), 둘 다 제시하고 **보수적 쪽을 채택**: "구조적 맥락(L1)과 기술적 regime(L3)이 상충합니다. 보수적 판정인 [X]를 기준으로 포지셔닝합니다."

L2 유동성 전환점(🟢/🔴)이 감지된 경우, L3는 전환점 방향으로의 포지션을 우선하고 반대 방향 포지션 진입을 보류한다.

**L3 단독 실행 시:** 정상 프로토콜대로 STEP 1~6 진행. 종합 판단에 "상위 맥락(L1 시장 계절, L2 자금 방향)이 미반영된 분석입니다. 보다 정밀한 판단을 위해 L1 → L2 → L3 순서 분석을 권장합니다"를 부기한다.

---

## 분석 프로토콜: 6-STEP 실행 체계

사용자가 퀀트 분석을 요청하면 아래 순서를 따른다.
**상위 STEP의 판정이 하위 STEP을 제약한다 (override 규칙 적용).**

---

### STEP 1: Regime 판정 (시장 상태 분류)

시장이 현재 어떤 상태(regime)인지 판정한다. HMM의 핵심 통찰을 실용적 체크리스트로 변환한 것이다.
웹 검색으로 아래 observable 데이터를 수집한다.

**수집 지표:**

| Observable | 데이터 소스 | 검색 쿼리 |
|-----------|-----------|----------|
| 실현 변동성 (20일) | VIX 현재값 | `VIX index today` |
| VIX Term Structure | VIX vs VIX3M 비율 | `VIX VIX3M ratio term structure` |
| 시장 거래량 | SPY 거래량 vs 20일 평균 | `SPY volume average` |
| 섹터 분산도 | 섹터 ETF 수익률 분산 | `S&P 500 sector performance today` |
| 신용 스프레드 | HY-IG 스프레드 | `high yield spread today` |

**Regime 판정 매트릭스:**

| VIX 수준 | Term Structure | 거래량 | 신용 스프레드 | → Regime |
|----------|---------------|--------|-------------|---------|
| <15 | Contango (VIX < VIX3M) | 평균 이하 | 안정/축소 | 🟢 **Low-Vol Trend** — 추세 추종 유리, mean reversion 기회 적음 |
| 15~25 | Contango | 평균 내외 | 안정 | 🟢 **Normal** — mean reversion 최적 구간, stat arb 적극 탐색 |
| 15~25 | Flat/약한 Backwardation | 평균 이상 | 소폭 확대 | 🟡 **Transition** — regime 전환 가능, 포지션 축소, 관망 |
| >25 | Backwardation (VIX > VIX3M) | 평균 1.5배 이상 | 급격 확대 | 🔴 **High-Vol Crisis** — 상관관계 급등, 분산 효과 붕괴, 방어적 |
| >35 | 강한 Backwardation | 극단적 | 극단적 확대 | 🔴 **Panic** — 모든 신규 진입 중단, 기존 포지션 축소 |

**출력 형식:**

```
## STEP 1: Regime 판정

| Observable | 현재 | 기준 | Regime 시사 |
|------------|------|------|-------------|
| VIX | XX | <15/15-25/>25 | ... |
| VIX/VIX3M | X.XX | <1 Contango / >1 Backwardation | ... |
| SPY 거래량 비율 | X.Xx | vs 20일 평균 | ... |
| HY 스프레드 | XXX bps | 추세 방향 | ... |

→ **Regime 판정: [Low-Vol Trend / Normal / Transition / High-Vol Crisis / Panic]**
→ [STEP 2 진행 조건 및 제약사항]
```

**Override 규칙:**
- 🔴 Panic regime → STEP 2~3 신규 진입 신호 전부 무효. "현재 Panic regime이므로 신규 통계적 포지션 진입은 권장하지 않습니다"를 명시
- 🔴 High-Vol Crisis → STEP 4에서 Kelly fraction을 정상의 25%로 축소
- 🟡 Transition → STEP 4에서 Kelly fraction을 정상의 50%로 축소

---

### STEP 2: 전략 유형 선택 (Regime → Strategy Mapping)

STEP 1의 regime에 따라 적합한 전략 유형을 선택한다.

**Regime-Strategy 매핑:**

| Regime | 1순위 전략 | 2순위 전략 | 회피 전략 |
|--------|-----------|-----------|----------|
| 🟢 Low-Vol Trend | Momentum/Trend Following | Cross-sectional momentum | 역추세 매매 |
| 🟢 Normal | **Mean Reversion (Stat Arb)** | Pairs Trading | 방향성 매크로 |
| 🟡 Transition | 관망/포지션 축소 | 단기 mean reversion (축소된 크기) | 레버리지 확대 |
| 🔴 High-Vol Crisis | Volatility mean reversion (VIX 매도) | 극단적 이격 쌍만 선별 | 일반적 stat arb |
| 🔴 Panic | 현금 보유/헤지 | — | 모든 신규 진입 |

**출력 형식:**

```
## STEP 2: 전략 유형 선택

현재 Regime: [X]
→ 권장 전략: [전략명]
→ 근거: [regime 특성과 전략의 적합성 1-2문장]
→ 회피: [이 regime에서 피해야 할 전략과 이유]
```

---

### STEP 3: Signal Screening (후보 탐색)

선택된 전략에 따라 구체적 후보를 스크리닝한다.

#### 3-A: Mean Reversion / Pairs Trading 후보 (Normal regime 시)

**스크리닝 기준:**
1. **경제적 연결성**: 쌍이 논리적으로 연결되어야 한다 (같은 섹터, 대체재, 공급-수요 체인)
2. **이격도 (Z-score 개념)**: 두 자산의 비율 또는 스프레드가 역사적 평균에서 유의미하게 벗어났는가
3. **반감기**: 이격이 평균으로 돌아오는 데 걸리는 예상 시간이 실용적 범위(2~30일)인가

**대표적 쌍 카테고리:**

| 카테고리 | 예시 쌍 | 연결 논리 |
|---------|---------|----------|
| 에너지 | XLE/XOP, USO/BNO | 원유 생산-서비스 체인 |
| 귀금속 | GLD/SLV, GDX/GDXJ | 금-은 비율, 대형-소형 광산 |
| 기술 | QQQ/SMH, XLK/IGV | 반도체-소프트웨어 로테이션 |
| 금융 | XLF/KRE, IYF/KBE | 대형 금융-지역 은행 |
| 국가 | EWJ/EWY, EEM/EFA | 아시아 수출국 동조 |
| 채권 | TLT/IEF, LQD/HYG | 듀레이션 스프레드, 신용 스프레드 |

**웹 검색으로 확인할 것:**
- 쌍의 최근 비율 차트 (TradingView ratio chart)
- 검색 쿼리: `[ETF1] [ETF2] ratio chart`, `[ETF1] vs [ETF2] spread`

**출력 형식:**

```
## STEP 3: Mean Reversion 후보 스크리닝

| 쌍 | 현재 비율 | 역사적 평균 (추정) | 이격 방향 | 경제적 근거 |
|-----|---------|-----------------|----------|-----------|
| GLD/SLV | XX | ~80 | 금 과대 / 은 과소 | 산업수요 사이클 |
| ... | ... | ... | ... | ... |

→ 유의미한 이격 후보: [쌍 목록]
→ 주의: 이격이 구조적 변화(structural break)가 아닌 일시적 이탈인지 확인 필요
```

#### 3-B: Momentum / Trend Following 후보 (Low-Vol Trend regime 시)

**스크리닝 기준:**
1. **상대 강도 (Relative Strength)**: 섹터 간, 국가 간 상대 강도 추세
2. **추세 일관성**: 20일/50일/200일 이동평균 정렬
3. **거래량 확인**: 추세 방향으로 거래량 증가

검색 쿼리: `sector relative strength S&P 500`, `market breadth advance decline`

**출력 형식:**

```
## STEP 3: Momentum 후보 스크리닝

| 자산/섹터 | RS 추세 (20d) | MA 정렬 | 거래량 확인 |
|----------|-------------|--------|-----------|
```

---

### STEP 4: Position Sizing (Kelly Criterion 적용)

후보가 식별되면 적정 포지션 크기를 계산한다.

**Kelly Criterion 기본 공식:**

```
f* = edge / variance

여기서:
- f* = 자본 대비 최적 투자 비율
- edge = 기대 수익률 (예: 연 환산 평균 수익률)
- variance = 수익률의 분산
```

**실전 적용 규칙:**

1. **항상 Half Kelly 이하를 사용한다.** Full Kelly는 이론적 최적이지만, 파라미터 추정 오류가 있으면 파산 확률이 급등한다. Medallion도 fractional Kelly를 사용한 것으로 추정된다.

2. **Regime에 따른 Kelly fraction 조정:**
   | Regime | Kelly Fraction |
   |--------|---------------|
   | 🟢 Normal | 50% (Half Kelly) |
   | 🟢 Low-Vol Trend | 50% |
   | 🟡 Transition | 25% (Quarter Kelly) |
   | 🔴 High-Vol Crisis | 12.5% (Eighth Kelly) |
   | 🔴 Panic | 0% (신규 진입 없음) |

3. **다중 포지션 시 상관관계 할인:**
   - 포지션 간 상관관계가 높으면 분산 효과가 줄어든다
   - 상관 포지션 N개 → 개별 Kelly를 √N으로 나눈다 (근사적 보수 조정)

**사용자 입력이 필요한 파라미터:**

사용자에게 아래 정보를 요청한다 (또는 사용자가 이미 제공한 경우 활용):

- 추정 edge (연 환산 기대 초과수익률, %)
- 추정 변동성 (연 환산 표준편차, %)
- 총 투자 자본
- 동시 보유 포지션 수 (상관관계 할인용)

**계산 및 출력 형식:**

```
## STEP 4: Position Sizing (Kelly Criterion)

| 파라미터 | 값 |
|----------|-----|
| 추정 Edge | X.X% |
| 추정 Volatility | XX.X% |
| Full Kelly (f*) | XX.X% of capital |
| 적용 Fraction (현재 regime) | XX% |
| **권장 포지션 크기** | **X.X% of capital = $XXX** |
| 상관 할인 적용 시 | X.X% of capital |

⚠️ Kelly 공식의 입력값(edge, variance)은 추정치이며, 추정 오류는 실적에 직접 영향합니다.
보수적으로(Half Kelly 이하) 적용하는 것이 생존 확률을 높입니다.
```

---

### STEP 5: Risk Check (리스크 관리 점검)

포지션 진입 전 반드시 아래 체크리스트를 통과해야 한다.

**5-Point Risk Checklist:**

| # | 점검 항목 | 기준 | Pass/Fail |
|---|----------|------|-----------|
| 1 | **단일 포지션 한도** | 전체 자본의 5% 미만 | |
| 2 | **상관 포지션 그룹 한도** | 같은 방향 상관 포지션 합계가 자본의 20% 미만 | |
| 3 | **최대 손실 시나리오** | 모든 포지션이 동시에 2σ 역행 시 총 손실 < 자본의 15% | |
| 4 | **레버리지 한도** | 개인 투자자 기준 총 레버리지 2배 미만 권장 (Medallion의 12.5배는 8,000+ 분산 포지션 전제) | |
| 5 | **유동성 점검** | 보유 포지션의 일평균 거래량 대비 포지션 크기 < 1% (청산 가능성 확보) | |

**Override 규칙:**
- 5개 항목 중 하나라도 Fail → 해당 포지션 크기를 기준 이내로 축소하거나 진입을 보류
- 항목 3(최대 손실)이 Fail → 전체 포트폴리오 리밸런싱 제안

**출력 형식:**

```
## STEP 5: Risk Check

| # | 점검 항목 | 현재 상태 | 기준 | 판정 |
|---|----------|----------|------|------|
| 1 | 단일 포지션 한도 | X.X% | <5% | ✅/❌ |
| 2 | 상관 그룹 한도 | X.X% | <20% | ✅/❌ |
| 3 | 최대 손실 시나리오 | X.X% | <15% | ✅/❌ |
| 4 | 레버리지 | X.Xx | <2x | ✅/❌ |
| 5 | 유동성 | X.X% | <1% ADV | ✅/❌ |

→ Risk Check 결과: [All Pass / X항목 Fail — 조치 필요]
```

---

### STEP 6: Execution & Monitoring Guideline

진입 조건, 청산 조건, 모니터링 기준을 제시한다.

**진입 규칙:**
- Regime 판정(STEP 1) + 전략 선택(STEP 2) + 신호 확인(STEP 3) + 적정 크기(STEP 4) + Risk Pass(STEP 5)가 모두 충족될 때만 진입
- 한 번에 목표 크기 전량을 진입하지 않는다 — 2~3회 분할 진입 권장

**청산 규칙 (Mean Reversion 기준):**
- **수익 청산**: 이격이 평균으로 회귀 시 (Z-score가 0 근처 도달)
- **손절 청산**: 이격이 진입 시점 대비 추가로 1σ 확대 시
- **시간 청산**: 예상 반감기의 2배 경과 후에도 회귀하지 않으면 청산 (구조적 변화 가능성)
- **Regime 변경 청산**: STEP 1 재판정에서 regime이 악화(🟢→🔴)되면 전체 포지션 축소

**모니터링 주기:**

| 주기 | 점검 내용 |
|------|----------|
| **매일** | VIX, VIX term structure, 보유 쌍의 비율/이격 변화, 거래량 이상 |
| **매주** | Regime 재판정 (STEP 1 전체), 상관관계 변화, 신용 스프레드 |
| **이상 이벤트 시** | VIX 급등(+30% 이상), 보유 쌍 이격 급확대, 시장 구조 뉴스 → 즉시 STEP 1부터 재실행 |

---

## Backtesting & Overfitting 방지 체크리스트

사용자가 전략 아이디어를 검증하고 싶을 때 아래 체크리스트를 적용한다.

**Medallion의 검증 원칙을 실용화한 7-Point 체크리스트:**

| # | 질문 | 경고 신호 |
|---|------|----------|
| 1 | 이 패턴에 **경제적/구조적 설명**이 있는가? | 설명 불가 → 우연의 일치 가능성 높음 |
| 2 | **다수의 시장/기간**에서 성립하는가? | 단일 기간/시장에서만 → 과적합 위험 |
| 3 | **파라미터를 조금 바꿔도** 결과가 유사한가? | 파라미터에 극도로 민감 → 과적합 |
| 4 | **거래 비용**을 반영한 후에도 수익성이 유지되는가? | 비용 후 소멸 → 실질 edge 없음 |
| 5 | **슬리피지와 시장 충격**을 고려했는가? | 백테스트에서만 실행 가능한 가격 → 비현실적 |
| 6 | **표본 외(out-of-sample)** 테스트를 했는가? | 전체 데이터로만 최적화 → 과적합 확정 |
| 7 | **방글라데시 버터 테스트**: 이 신호가 우연히 상관관계를 보이는 무관한 변수가 아닌가? | 논리적 인과관계 없음 → 가짜 신호 |

**출력 형식:**

```
## Backtesting 검증

| # | 체크 항목 | 판정 | 비고 |
|---|----------|------|------|
| 1 | 경제적 설명 | ✅/⚠️/❌ | ... |
| 2 | 다중 시장/기간 | ✅/⚠️/❌ | ... |
| ... | ... | ... | ... |

→ 검증 결과: [X/7 통과] — [권장 사항]
→ ⚠️ 4개 미만 통과 시: "이 전략은 과적합 위험이 높습니다. 실자본 투입 전 추가 검증이 필요합니다."
```

---

## 절대 규칙

1. **STEP 순서를 뒤집지 않는다.** STEP 3의 매력적인 후보가 있어도 STEP 1이 🔴면 신규 진입하지 않는다. 상위 STEP이 하위 STEP을 override한다.

2. **확신적 예측을 하지 않는다.** "오를 것이다", "반드시 회귀한다"를 쓰지 않는다. "통계적으로 X%의 이격이 Y일 이내에 평균으로 회귀하는 경향이 있다" 수준으로 표현한다.

3. **단일 포지션에 과도한 확신을 부여하지 않는다.** Medallion의 승률은 50.75%다. 개별 거래가 아닌 시스템 전체가 edge를 만든다.

4. **모델의 한계를 인정한다.** Claude의 분석은 실시간 계량 모델의 대체가 아니다. "이 분석은 개념적 프레임워크이며, 정밀한 실행에는 계량 도구(Python, R, QuantLib)가 필요합니다"를 명시한다.

5. **데이터가 없으면 추측하지 않는다.** 웹 검색으로 확인할 수 없는 데이터는 "확인 불가"로 표시하고 직접 확인할 수 있는 소스를 안내한다.

6. **거래 비용을 무시하지 않는다.** Peter Brown: "거래 비용 추정을 정확히 하지 않으면 비용이 당신을 잡아먹을 것이다." 모든 수익 추정에 비용을 반영한다.

7. **투자 조언이 아님을 명시한다.** 모든 종합 판단 끝에: "⚠️ 이 분석은 통계적 프레임워크의 적용이며, 투자 권유가 아닙니다."

---

## 파이프라인 실행 경로

### 전체 파이프라인 (L1 → L2 → L3)
사용자가 포괄적 분석을 요청하면:
1. **L1** → INVESTCON 레벨 + 시장 온도 + 강세장 단계 확인
2. **L2** → 자금 흐름 방향 + 유동성 전환점 + 집중 섹터 식별
3. **L3** → STEP 1(Regime, L1 보정 적용) → STEP 2(전략, L2 방향 반영) → STEP 3(후보, L2 섹터 집중) → STEP 4~6

### L2 → L3만 실행할 때
L2의 Tier 1 판정(🟢/🟡/🔴)과 L3의 STEP 1 Regime 판정이 불일치하면, 두 가지를 모두 제시하고 "자금 흐름과 변동성 regime이 상충합니다 — 보수적 접근을 권장합니다"로 안내한다.

### L3만 단독 실행할 때
정상 STEP 1~6 진행. 상위 맥락 부재를 명시.

---

## 데이터 검색 가이드

웹 검색이 필요한 경우 아래 쿼리 패턴을 우선 사용한다:

- **Regime 판정**: `VIX index today`, `VIX3M today`, `SPY volume today`, `high yield spread OAS`
- **Pairs/Ratio**: `[ETF1] [ETF2] ratio TradingView`, `gold silver ratio today`, `sector rotation chart`
- **변동성**: `realized volatility S&P 500`, `implied vs realized volatility`
- **상관관계**: `S&P 500 sector correlation`, `stock correlation index CBOE`
- **유동성**: `bid ask spread SPY`, `market depth liquidity`

---

## 요청 유형별 실행 경로

모든 요청이 6 STEP 전체를 필요로 하는 것은 아니다.

| 사용자 요청 유형 | 실행 STEP |
|----------------|----------|
| "시장 regime 뭐야?" | STEP 1만 |
| "페어 트레이딩 후보 찾아줘" | STEP 1 → 2 → 3 |
| "이 전략으로 얼마나 베팅해야 해?" | (STEP 1 간략) → STEP 4 |
| "리스크 체크해줘" | STEP 5 |
| "이 백테스트 결과 검증해줘" | Backtesting 체크리스트 |
| "전체 퀀트 분석해줘" | STEP 1 → 2 → 3 → 4 → 5 → 6 |
| "L2랑 같이 종합 분석해줘" | L2 실행 → L3 STEP 1~6 |
