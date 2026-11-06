# QUANT_MODEL_SPEC — 퀀트 모델 명세

버전 0.1 (설계) · 상태: **검증 전 설계 명세**. 모든 [PARAM]은 사전 등록 초기값이며 `BACKTEST_SPEC.md` 절차로 확정한다.
근거 문헌 ID(A1, B3 …)는 `QUANT_RESEARCH.md` 참조.

---

## 0. 설계 요약

```
 Raw OHLCV ─▶ [L0 Price/Volume Transform Layer]  (구 Price Action Engine)
                 │  수익률, 갭, 범위, 거래대금, 잔차 수익률
                 ▼
 ┌──────────────┬──────────────┬──────────────┬──────────────┬───────────────┐
 │ Momentum     │ Mean Rev.    │ Trend        │ Volatility   │ Volume &      │  L1: 엔진별 Feature
 │ (횡단면)     │ (단기 반전)  │ (시계열)     │ (위험·저변동)│ Liquidity     │  + 엔진 점수 S_e
 └──────┬───────┴──────┬───────┴──────┬───────┴──────┬───────┴──────┬────────┘
        │              │              │              │              │
        │          [Market Regime Engine] ── 국면 확률 π_t (필터링) ─┤
        ▼              ▼              ▼              ▼              ▼
 [L2 Signal Ensemble Engine]  정규화 순서형 로지스틱 (Champion)
                              + LightGBM (Challenger, 조건부 채택)
        ▼
 [Calibration]  Temperature scaling → p_up, p_flat, p_down
        ▼
 [L3 Final Decision Engine]   기대 초과수익·비용·위험 게이트·신뢰도·히스테리시스
        ▼
   BUY / HOLD(사유) / SELL  + 확률 + 신뢰도 등급 + 기여도 + 위험 지표
```

| 요청된 모듈 | 본 명세에서의 위치 | 비고 |
|---|---|---|
| 1. Price Action Engine | §3 L0 변환 계층 | 독립 신호 엔진이 아님 (차트 패턴 근거 약함, C10) |
| 2. Momentum Engine | §5 | 중기 핵심 |
| 3. Mean Reversion Engine | §6 | 단기 핵심 |
| 4. Trend Following Engine | §7 | 중기·국면 |
| 5. Volatility Engine | §4 (위험 예측) + §8 (방향 Feature) | 모든 정규화의 기반이므로 먼저 계산 |
| 6. Volume & Liquidity Engine | §9 | 조건 변수·필터·비용 |
| 7. Market Regime Engine | §10 | 필터링 확률만 사용 |
| 8. Statistical/ML Engine | §12 | Challenger |
| 9. Signal Ensemble Engine | §11 | 순서형 로지스틱 스태킹 |
| 10. Final Decision Engine | §13–14 | BUY/HOLD/SELL 상태기계 |

---

## 1. 표기와 공통 정의

| 기호 | 정의 |
|---|---|
| m ∈ {US, KR} | 시장. **모든 모델은 시장별로 완전히 독립 추정**한다. |
| i | 종목 (내부 영구 ID `security_id`, 티커 변경에 불변) |
| t | 거래일 인덱스 (시장별 거래 캘린더). 신호는 t일 **정규장 마감 후** 산출 |
| h ∈ {10, 63} | 예측 기간(거래일). Short = 10 (약 2주), Mid = 63 (약 3개월) [PARAM] |
| O,H,L,C,V,DV | 시가·고가·저가·종가·거래량·거래대금(현지통화). OHLC는 **분할·병합·무상증자 조정**(price adjustment factor) 적용 |
| P_t | 총수익 조정 종가 (분할 + 배당 재투자 조정) |
| r_t | 일 로그수익률 ln(P_t / P_{t-1}) |
| r^on_t | 오버나이트 ln(O_t / C_{t-1}), r^oc_t = ln(C_t / O_t) |
| B(i) | 종목 i의 벤치마크: US — S&P 500 편입 종목은 SPY, MidCap 400 편입 종목은 MDY의 총수익 조정가 / KR — KOSPI 종목은 KODEX 200(069500), KOSDAQ 종목은 KODEX 코스닥150(229200) [PARAM] |
| x^e_t | 벤치마크 대비 초과 로그수익률 r_{i,t} − r_{B(i),t} |
| σ̂_{i,t} | 일간 변동성 예측치 (§4.3) |
| σ̂^e_{i,t} | 일간 초과수익률 변동성 예측치 (§4.4) |
| N_t | t일의 유니버스 종목 수 |

### 1.1 횡단면 정규화 연산자 𝒵

모든 Feature f는 시장 m, 날짜 t의 유니버스 내에서 순위 기반 정규 점수로 변환한다(이상치·분포 왜곡에 강건, 시점 간 비교 가능).

```
rank_i = f_i의 오름차순 순위 (동점은 평균 순위), 결측 제외
u_i    = (rank_i − 0.5) / N_valid
𝒵(f)_i = clip( Φ⁻¹(u_i), −3, 3 )
```

결측 Feature는 𝒵 = 0 (중립)으로 대체하고 `missing_mask`에 기록한다. 결측 비율이 해당 엔진 Feature의 50% 이상이면 엔진 점수 = 0 및 데이터 품질 플래그 설정.

### 1.2 시계열 정규화 (종목 자기 이력 기준)

`pct_own(f, n)` = f_{i,t}의 자기 과거 n일 분포 내 백분위 (t 포함, 최소 n/2 관측).

---

## 2. 유니버스와 적격성

### 2.1 날짜별 Point-in-Time 유니버스 U_{m,t}

종목 i가 t일 유니버스에 포함되려면 **t일 기준으로 알 수 있었던 정보만으로** 다음을 모두 만족:

| 조건 | US [PARAM] | KR [PARAM] |
|---|---|---|
| 지수 편입 | t일 기준 S&P 500 또는 S&P MidCap 400 구성종목 (PIT 구성 데이터 미확보 시 §2.2 대체 규칙) | t일 기준 KOSPI 200 또는 KOSDAQ 150 구성종목 (동일) |
| 상장 기간 | ≥ 252 거래일 (Mid), ≥ 126 거래일 (Short) | 동일 |
| 가격 | C_t ≥ $5 | C_t ≥ ₩1,000 |
| 유동성 | median(DV, 63일) ≥ $5M | median(DV, 63일) ≥ ₩3,000,000,000 |
| 거래 상태 | 최근 5거래일 내 거래정지 없음 | 거래정지·관리종목·투자경고/위험·정리매매 아님 |
| 증권 유형 | 보통주 (ADR 포함 여부 [PARAM] 기본 제외) | 보통주 (우선주, 스팩, 리츠, ETF/ETN 제외) |

### 2.2 대체 유니버스 규칙 (PIT 지수 구성 데이터가 없을 때)

t일 기준 시장의 전체 보통주 중 시가총액 상위 N_m 종목 (US: 900, KR: 350)을 t일 시가총액으로 선정 [PARAM]. 시가총액 = C_t × 상장주식수(t일 기준 공시값). 이 규칙은 상장폐지 종목을 포함한 전체 종목 데이터가 있을 때만 생존편향이 없다 (`DATA_SOURCE_SPEC.md §2`).

### 2.3 제외 사유 코드
`EX_LISTING_AGE`, `EX_PRICE`, `EX_LIQUIDITY`, `EX_HALT`, `EX_ADMIN`(관리·경고), `EX_TYPE`, `EX_DATA`(데이터 품질). 유니버스 밖 종목에 대해서는 신호를 산출하지 않으며 UI는 "분석 대상 아님 (사유)"를 표시한다.

---

## 3. L0 — 가격·거래량 변환 계층 (Price Action Layer)

모든 엔진이 공유하는 결정론적 변환. 독립 신호를 내지 않는다.

| 출력 | 정의 |
|---|---|
| `ret_d` | r_t |
| `ret_on`, `ret_oc` | r^on_t, r^oc_t |
| `ret_k` | ln(P_t / P_{t−k}), k ∈ {5, 21, 63, 126, 252} |
| `xret_d` | x^e_t |
| `range_gk` | Garman-Klass 일 분산: 0.5·ln(H/L)² − (2ln2 − 1)·ln(C/O)² |
| `dv` | DV_t (원천 거래대금; 미제공 시 V_t × (H+L+C)/3 근사 + 플래그) |
| `resid_mkt_ind` | 시장·산업 잔차 수익률 (§6.2) |
| `resid_ff` | 팩터 잔차 수익률 (§5.3) |
| `gap_flag` | \|r^on_t\| > 6·σ̂_{i,t−1} (이벤트성 갭; 데이터 오류 검사와 해석에 사용) |

---

## 4. Volatility Engine — 위험 예측 (선행 계산)

근거: D1, D2, D4, D5, D6, D8.

### 4.1 Yang-Zhang 실현변동성 (창 n)

```
σ²_on  = 1/(n−1) Σ (r^on_j − mean(r^on))²
σ²_oc  = 1/(n−1) Σ (r^oc_j − mean(r^oc))²
σ²_RS  = 1/n Σ [ ln(H_j/C_j)·ln(H_j/O_j) + ln(L_j/C_j)·ln(L_j/O_j) ]
k      = 0.34 / (1.34 + (n+1)/(n−1))
σ²_YZ(n) = σ²_on + k·σ²_oc + (1−k)·σ²_RS          (j = t−n+1 … t)
```
산출: `VOL_YZ_21`, `VOL_YZ_63`, `VOL_YZ_252` (일 단위 표준편차).

### 4.2 GJR-GARCH(1,1)-t

```
r_t = μ + ε_t,  ε_t = σ_t z_t,  z_t ~ standardized Student-t(ν)
σ²_t = ω + (α + γ·1[ε_{t−1}<0])·ε²_{t−1} + β·σ²_{t−1}
제약: ω>0, α≥0, γ≥0, β≥0, α + γ/2 + β < 0.999
```
- 추정: 종목별 최근 1,000거래일(최소 500) 일 수익률(%) 최대우도. 주 1회 재추정, 일별로는 파라미터 고정 상태에서 σ²_t 갱신.
- h일 누적 분산 예측: σ̄² = ω/(1−φ), φ = α+γ/2+β 일 때 Σ_{j=1}^{h} [σ̄² + φ^{j−1}(σ²_{t+1} − σ̄²)].
- 수렴 실패·제약 위반 시 EWMA(λ=0.94) 대체 + 플래그.

### 4.3 HAR 예측 (범위 기반 RV 대리변수)

일 분산 대리변수 RV_t = range_gk_t (+ r^on_t² 오버나이트 성분).
```
RV^{(w)}_t = mean(RV_{t−4..t}),  RV^{(m)}_t = mean(RV_{t−21..t})
ln RV^{(h)}_{t+1:t+h} = b0 + b_d ln RV_t + b_w ln RV^{(w)}_t + b_m ln RV^{(m)}_t + e
```
- 시장별 **패널 풀링** 추정(종목별 평균 제거), 확장창, 분기 재추정. 로그 변환 편의 보정: exp(·)·exp(s²/2), s² = 잔차분산.

### 4.4 최종 변동성 예측

```
σ̂²_{i,t}(h) = ½·GARCH_h + ½·HAR_h       (동일가중 결합, K2)
σ̂_{i,t}      = sqrt(σ̂²_{i,t}(h) / h)     (일 단위 환산)
```
- 단, 검증 구간에서 Model Confidence Set(J3, QLIKE 손실)이 한 모형만 남기면 해당 모형 단독 사용 [검증 후 확정].
- 초과수익 변동성 σ̂^e: 같은 절차를 x^e_t에 적용하되, MVP는 EWMA(λ=0.97, 최근 252일) 단순 추정 [PARAM].

### 4.5 위험 출력 (사용자 표시용)

| 출력 | 정의 |
|---|---|
| `exp_vol_h` | σ̂_{i,t}·√h (기간 수익률 1σ) |
| `cvar95_h` | 기간 절대수익률의 95% CVaR. 표준화 잔차 z의 경험분포(최근 1,000일)에서 ES_{5%}(z)를 구해 −(μ̂_h − σ̂√h·ES) [근사, 정규 가정 금지] |
| `vol_pct` | pct_own(σ̂, 756) |
| `vol_regime_i` | vol_pct ≥ 0.8 → HIGH, ≤ 0.2 → LOW, else MID |

---

## 5. Momentum Engine (횡단면)

근거: A1–A3, A5–A9, A11, A12. 역할: **종목 간 상대 강도**.

### 5.1 MOM_12_1
```
MOM_12_1 = ln(P_{t−21} / P_{t−252})
```

### 5.2 HIGH52
```
HIGH52 = P_t / max(P_{t−251..t})        (총수익 조정 종가 기준)
```

### 5.3 RMOM (잔차 모멘텀, Blitz et al. 2011의 일별 변형)
- 팩터: US — Kenneth French 일별 Mkt−RF, SMB, HML / KR — 자체 구축 시장(유니버스 가치가중) 초과수익, SMB(시총 하위30% − 상위30%, 월 리밸런싱). HML은 재무 데이터 도입 전까지 KR에서 제외.
- 추정창 W = [t−755, t−22] (최소 504일), 일 초과수익 회귀:
```
r_{i,d} − rf_d = a + b'F_d + ε_{i,d}
RMOM = Σ_{d=t−252}^{t−22} ε̂_{i,d} / ( sd(ε̂_W) · sqrt(231) )
```

### 5.4 엔진 점수
```
S_mom = mean( 𝒵(MOM_12_1), 𝒵(RMOM), 𝒵(HIGH52) )     [사전 등록 동일가중, 부호 +]
```

### 5.5 진단 Feature (모델 입력 아님, 연구·모니터링용)
`MOM_12_7 = ln(P_{t−147}/P_{t−252})`, `MOM_6_2 = ln(P_{t−21}/P_{t−126})` (A9), 모멘텀 팩터 자체의 실현변동성 `VOL_UMD_126` (A6).

### 5.6 크래시 조건 입력 (국면 상호작용)
- `I_crash = π^{HV}_t · 1[MKT_RET_24M < 0]` — 고변동성 확률 × 직전 24개월 시장 수익 음(−) (A5).
- `I_down36 = 1[MKT_RET_36M < 0]` (A12).
- 이 값들은 §11의 상호작용 항 S_mom × I 로만 사용 (계수는 추정, 부호 제약 ≤ 0).

---

## 6. Mean Reversion Engine (단기 반전)

근거: B1–B5, E4, E5. 역할: **유동성 기반 과도 움직임의 반전**.

### 6.1 원 반전
```
REV_5  = −ln(P_t / P_{t−5})
REV_21 = −ln(P_t / P_{t−21})
```

### 6.2 잔차 반전 RREV_21 (B3)
- 산업 수익 R^{ind}_{d}: 동일 산업(US: GICS Sector 11개 또는 SIC 기반 대체 [VERIFY 데이터], KR: KRX 업종 분류) 유니버스 종목의 동일가중 일수익 (자기 자신 제외).
- 추정창 [t−272, t−21] (최소 200일):
```
r_{i,d} = a + b_m r_{B(i),d} + b_ind R^{ind}_{d} + ε_{i,d}
RREV_21 = −Σ_{d=t−20}^{t} (r_{i,d} − â − b̂_m r_{B,d} − b̂_ind R^{ind}_d) / ( sd(ε̂) · sqrt(21) )
```

### 6.3 평균회귀 성향 (조건 변수, Experimental)
- `VR5 = Var(5일 수익) / (5·Var(1일 수익))` 최근 252일 (B7). VR5 < 1 은 반전 성향.
- `OU_HL`: 63일 MA 대비 로그 괴리 y의 AR(1) y_d = c + φ y_{d−1} + e 에서 반감기 −ln2/ln φ (0<φ<1일 때) (B9).
- 이 둘은 Champion 모델에 넣지 않고 Challenger 및 진단에서 증분 정보를 검정.

### 6.4 실무 지표 (Experimental — Champion 제외)
`RSI_14` (Wilder 평활), `PCTB_20` = (C − (MA20 − 2·SD20)) / (4·SD20). 연속값으로만 쓰며 임계값 규칙을 쓰지 않는다. 채택 조건: §15 증분 검정 통과.

### 6.5 엔진 점수
```
S_rev = mean( 𝒵(REV_5), 𝒵(RREV_21) )       [부호 +: 하락한 종목일수록 반등 기대]
```

---

## 7. Trend Following Engine (시계열)

근거: A4, C3, C5, C6, C9. 역할: **종목 자신의 절대 추세와 추세 품질** (횡단면 모멘텀과 관점 분리).

### 7.1 다기간 변동성 정규화 추세
```
TS_k   = ln(P_t / P_{t−k}) / ( σ̂_{i,t} · sqrt(k) ),   k ∈ {21, 63, 252}
TREND  = mean( clip(TS_21, −2, 2), clip(TS_63, −2, 2), clip(TS_252, −2, 2) )
```

### 7.2 장기 이동평균 위치·기울기
```
MA200_POS = (ln P_t − ln MA200_t) / (σ̂_{i,t} · sqrt(200))
MA50_SLOPE = (ln MA50_t − ln MA50_{t−10}) / (σ̂_{i,t} · sqrt(10))
```
MACD, 이중 MA 교차, Donchian 돌파는 TS_k와 선형 중복(C5)이므로 추가하지 않는다. (Challenger 증분 검정 대상에도 넣지 않음 — 다중검정 비용 절감.)

### 7.3 엔진 점수
```
S_trend = mean( 𝒵(TREND), 𝒵(MA200_POS), 𝒵(MA50_SLOPE) )
```
또한 **비정규화 원값** `TREND`(절대값 의미 유지)를 결정 엔진의 위험 게이트(§13.4)에 사용.

### 7.4 모멘텀과의 중복 관리
S_mom과 S_trend의 횡단면 상관을 매일 기록한다. 학습창 평균 상관이 0.8을 넘으면 앙상블에서 두 점수를 평균한 단일 점수로 대체하는 사전 등록 규칙을 적용한다 [PARAM] (공선성으로 인한 계수 불안정 방지).

---

## 8. Volatility Engine — 방향 Feature (저위험 이상현상)

근거: D9, F5.

```
IVOL = sd(ε̂_{i,d}),  d ∈ [t−62, t],  ε̂: §5.3 팩터 회귀 잔차 (계수는 §5.3 창 추정치 사용)
BETA = Dimson 베타: r_{i,d} = a + b0 r_{M,d} + b1 r_{M,d−1} + e, BETA = b0 + b1, 창 252일
       (KR 비동시거래 보정; US도 동일 정의로 통일)
S_lowrisk = mean( −𝒵(IVOL), −𝒵(BETA) )
```

---

## 9. Volume & Liquidity Engine

근거: E1–E5, E7.

| Feature | 정의 | 역할 |
|---|---|---|
| `ILLIQ` | mean_{63일}( \|r_d\| / DV_d ) | 필터·비용·조건 변수 |
| `TURN` | mean_{63일}( DV_d / MktCap_d ) (시총 미확보 시 결측) | 모멘텀 조건 변수 (E2) |
| `ABVOL` | ln(mean DV_{t−4..t}) − ln(mean DV_{t−252..t−22}) | 단기 방향 Feature (E3, +) 및 반전 조건 (E4) |
| `DILLIQ` | ln ILLIQ_21 − ln ILLIQ_252 | 유동성 악화 감지 (위험 표시) |
| `SPREAD_AR` | Abdi-Ranaldo 추정 스프레드의 63일 중앙값 (E7) | 거래비용 |

엔진 점수:
```
S_vol = 𝒵(ABVOL)                           [단기 모델 전용, 부호 +]
```
상호작용(§11): `S_rev × 𝒵(ABVOL)` (E4: 거래량 동반 움직임의 반전), `S_mom × (−𝒵(TURN))` (E2).

---

## 10. Market Regime Engine

근거: H1–H3, A5, A12, B4, C9. 시장 m 단위로 산출, 모든 종목이 공유.

### 10.1 입력
- 시장 벤치마크(US: SPY, KR: KODEX 200) 일 로그수익 r_M, YZ 실현변동성.
- 시장 폭(Breadth): `BREADTH_200 = 유니버스 중 P_t > MA200_t 비율`.

### 10.2 R1: 2-상태 Gaussian HMM
- 관측: 겹치지 않는 5거래일 블록의 y_w = (주간 로그수익, ln σ_YZ,5일). 
- 상태 2개, 상태별 다변량 정규. EM(Baum-Welch) 확장창 추정, 분기별 재추정, 초기값 20회 무작위 → 최대우도 선택(고정 seed).
- 레이블 정렬: 변동성 평균이 큰 상태를 HV로 정의.
- 출력: **필터링 확률** π^{HV}_t = P(S_t = HV | y_{1..t}). 블록 중간 날짜는 마지막 완료 블록 값 + 일간 갱신은 forward 재귀로 부분 블록 반영하지 않음(단순성) [PARAM].
- 스무딩 확률(Viterbi, forward-backward)은 학습·평가 어디에도 입력으로 쓰지 않는다.

### 10.3 R2: 규칙 기반 상태
```
TREND_M   = 1[P_M,t > MA200_M,t]
MKT_RET_24M, MKT_RET_36M = ln(P_M,t / P_M,t−504), ln(P_M,t / P_M,t−756)
VOL_PCT_M = pct_own(σ_YZ,21 of market, 2520)     (10년)
```

### 10.4 국면 출력
| 국면 코드 | 조건 | UI 표시 |
|---|---|---|
| `BULL_CALM` | TREND_M=1, π^{HV} < 0.5 | 상승 추세 · 안정 |
| `BULL_VOLATILE` | TREND_M=1, π^{HV} ≥ 0.5 | 상승 추세 · 변동성 확대 |
| `BEAR_CALM` | TREND_M=0, π^{HV} < 0.5 | 하락/횡보 · 안정 |
| `BEAR_VOLATILE` | TREND_M=0, π^{HV} ≥ 0.5 | 하락 · 고변동성 |
| `CRISIS` | TREND_M=0, π^{HV} ≥ 0.7 [PARAM], VOL_PCT_M ≥ 0.9 | 위기 국면 |

"횡보(Sideways)"는 별도 국면이 아니라 사후 평가 분류로 사용 (`BACKTEST_SPEC.md §10`)한다 — 실시간 판별 근거가 약하기 때문.

### 10.5 모델 입력으로 쓰이는 국면 변수
`π^{HV}_t`, `TREND_M`, `I_crash`, `I_down36`, `VOL_PCT_M`, `BREADTH_200`.

---

## 11. Signal Ensemble Engine (L2) — Champion

### 11.1 예측 대상 (라벨)

t일 신호의 실행은 t+1 시가. 기간 h 초과 로그수익:
```
X_{i,t}(h) = ln( P^O_{i,t+1+h} / P^O_{i,t+1} ) − ln( P^O_{B,t+1+h} / P^O_{B,t+1} )
   P^O: 총수익 조정 시가 (시가에 동일 조정계수 적용)
τ_{i,t}(h) = max( c^{rt}_{i,t},  κ · σ̂^e_{i,t} · sqrt(h) ),   κ = 0.5 [PARAM]
Y = UP   if X >  τ
    DOWN if X < −τ
    FLAT otherwise
```
- c^{rt}: 왕복 거래비용 추정 (§14.2).
- κ = 0.5 는 정규 근사에서 대략 UP/FLAT/DOWN ≈ 31%/38%/31% 균형을 의도한 값이며, 민감도 분석에서 {0.25, 0.5, 0.75}를 보고한다.
- 상장폐지로 t+1+h 시가가 없으면 상장폐지 수익률 규칙(`BACKTEST_SPEC.md §5.6`) 적용.

### 11.2 입력 벡터

**Short (h=10)**
```
x_S = [ S_rev, S_vol, S_mom, S_trend, S_lowrisk,
        S_rev·VOL_PCT_M,              (B4: 고변동성 시 반전 강화, 부호 ≥0)
        S_rev·𝒵(ABVOL),                (E4, 부호 제약 없음)
        π^HV, TREND_M ]               (국면 주효과: 클래스 기저율 이동)
```
**Mid (h=63)**
```
x_M = [ S_mom, S_trend, S_lowrisk, S_rev,
        S_mom·I_crash,                 (A5, 부호 ≤0)
        S_mom·I_down36,                (A12, 부호 ≤0)
        S_mom·(−𝒵(TURN)),              (E2, 부호 ≥0)
        π^HV, TREND_M, BREADTH_200 ]
```
국면 주효과 변수는 종목 간에 동일하므로 횡단면 순위에 영향을 주지 않고 **시점별 UP/DOWN 기저율**만 이동시킨다.

### 11.3 모형: 정규화 순서형 로지스틱 (Proportional Odds)

클래스 순서 DOWN(0) < FLAT(1) < UP(2).
```
P(Y ≤ k | x) = σ( θ_k − βᵀx ),  k ∈ {0, 1},  θ_0 < θ_1
p_down = σ(θ_0 − βᵀx)
p_flat = σ(θ_1 − βᵀx) − σ(θ_0 − βᵀx)
p_up   = 1 − σ(θ_1 − βᵀx)
```
추정 목적함수:
```
min_{θ,β}  − Σ_t w_t Σ_{i∈U_t} log p_{Y_{i,t}}(x_{i,t})  +  λ ‖ β − β⁰ ‖²
   w_t = 1 / N_t                   (날짜별 동일 총가중 — 종목 수가 많은 시기 과대반영 방지)
   β⁰  = c · s                     (s: 사전 등록 부호 벡터, 주효과 엔진 = +1, 상호작용·국면 = 0)
   c   : 1차원 사전 적합 (x의 엔진 주효과 평균 1개 변수로 순서형 로짓 적합한 계수)
   부호 제약: 사전 등록 부호와 반대인 계수는 0으로 고정 (L-BFGS-B 경계 제약)
```
- **해석**: λ→∞ 이면 엔진 동일가중 모형(K2의 "결합 퍼즐" 대응), λ→0 이면 자유 추정. λ는 학습창 내부 Purged K-Fold(K=5, embargo=h)에서 로그손실 최소화로 {0.01, 0.1, 1, 10, 100} 중 선택.
- 표본 날짜: Short은 5거래일마다, Mid는 21거래일마다 (라벨 중첩 축소). 학습창: 확장창(expanding), 시작일 고정.
- 재추정 주기: 분기 1회. 확장창이므로 계수 변화가 점진적이다. 직전 버전 대비 계수가 2 표준오차(블록 부트스트랩) 이상 변하면 `COEF_SHIFT` 경보 → 사람 검토 후 배포.

### 11.4 비교 대상 결합 방법 (검증에서 반드시 함께 보고)

| 방법 | 정의 | 역할 |
|---|---|---|
| EW | 엔진 주효과 동일가중 합 1개 변수 순서형 로짓 | **기준선 (필수)** — Champion은 이것을 유의하게 이겨야 함 |
| LIN | 비정규화 선형 가중 (λ=0) | 과적합 진단 |
| ORD-RIDGE | §11.3 | **Champion 후보** |
| MNL | 다항 로지스틱 (클래스별 계수) | 순서 가정 검증 |
| BMA | 엔진 부분집합 모형 2^5개의 BIC 근사 사후확률 가중 (K4) | 비교 |
| DMA | 망각계수 α=0.99, λ_f=0.99 동적 가중 (K5) | 비교 — 가중 급변 위험 확인용 |
| REGIME | 국면별 별도 계수 (BULL/BEAR × CALM/VOLATILE) | 비교 — 표본 분할로 분산 증가 예상 |
| GBDT | §12 | Challenger |

**선택 규칙 (사전 등록)**: Walk-forward OOS 로그손실이 가장 낮은 방법이 EW 대비 Diebold-Mariano 검정(HAC, 날짜 클러스터) p < 0.05로 우월하고, Model Confidence Set(90%)에 포함되며, 계수 안정성(분기별 부호 반전 없음)을 만족할 때 채택. 여러 방법이 MCS에 남으면 **가장 단순한 방법**(EW < ORD-RIDGE < MNL < 나머지)을 채택.

### 11.5 확률 보정 (Calibration)

다중클래스 Temperature scaling (K7; 1개 파라미터로 분산이 작음):
```
logits ℓ = (θ_0 − βᵀx, θ_1 − βᵀx) 의 누적 로짓을 T로 나눔:
p̃(·) = 위 식에서 (θ_k − βᵀx) → (θ_k − βᵀx)/T
T* = argmin_T  calibration fold 로그손실
```
- Calibration fold: 학습창 마지막 252거래일(Short) / 504거래일(Mid). 이 기간은 β 추정에서 제외하고 T만 추정(분리).
- 보정 후 검증 기준(ECE, 신뢰도 기울기)은 `MODEL_VALIDATION_SPEC.md §3`.
- 기준 미달 시 확률 **수치 비표시** 정책 (`MODEL_VALIDATION_SPEC.md §3.6`).

### 11.6 기여도 (설명 가능성)

선형 지표 η = βᵀx 의 항별 분해:
```
contrib_j = β_j · x_j        (주효과와 상호작용을 엔진 단위로 합산)
```
UI에는 엔진 단위 기여(모멘텀, 단기 반전, 추세, 저위험, 거래량, 국면)를 막대로 표시. 상호작용 항은 해당 주효과 엔진에 귀속. 이 분해는 정확(선형 모형)하며 근사가 아니다.

---

## 12. Statistical / ML Prediction Engine — Challenger

근거: G1, G2. 목적: 검증된 Feature 간 **비선형 상호작용**의 증분 가치 측정.

| 항목 | 명세 |
|---|---|
| 알고리즘 | LightGBM multiclass (3클래스) |
| 입력 | L1 원 Feature 전부 (엔진 점수가 아닌 원 Feature 𝒵값: MOM_12_1, RMOM, HIGH52, REV_5, RREV_21, TREND, MA200_POS, MA50_SLOPE, IVOL, BETA, ABVOL, 𝒵(ILLIQ), 𝒵(TURN), vol_pct) + 국면 변수 + Experimental Feature(RSI_14, PCTB_20, VR5, OU_HL) |
| 단조 제약 | 사전 등록 부호가 있는 Feature는 monotone_constraints 적용 (UP 클래스 기준) |
| 하이퍼파라미터 탐색 공간 | num_leaves ∈ {7, 15, 31}, min_data_in_leaf ∈ {500, 2000}, learning_rate = 0.05, feature_fraction = 0.7, bagging_fraction = 0.7, lambda_l2 ∈ {1, 10}, early stopping (purged 내부 검증, 최대 2000 rounds). **총 12개 조합 고정** — 시행 횟수는 DSR 계산에 포함 |
| seed | 고정 (재현성), deterministic=true, num_threads 고정 |
| 보정 | Isotonic(클래스별 one-vs-rest 후 정규화) — calibration fold 표본 ≥ 10,000일 때, 아니면 Temperature |
| 채택 규칙 | (a) Champion 대비 OOS 로그손실 DM 검정 p < 0.05 (b) 비용 차감 BUY 포트폴리오 DSR 개선 (c) PBO < 0.2. 충족 시 **Champion과 확률 동일가중 평균**(대체 아님, K2). |
| 설명 | TreeSHAP 엔진 단위 합산 (근사임을 UI에 명시) |

---

## 13. Final Decision Engine (L3)

### 13.1 입력 요약

| 기호 | 정의 |
|---|---|
| p_up, p_flat, p_down | 보정 확률 |
| E | p_up − p_down (방향 우위, −1~1) |
| m̄_k | 학습창에서 추정한 클래스 k의 조건부 평균 표준화 초과수익 E[X/(σ̂^e√h) \| Y=k] |
| μ̂ | 기대 초과수익 = σ̂^e√h · Σ_k p_k m̄_k |
| c^{rt} | 왕복 비용 (§14.2) |
| NER | μ̂ − c^{rt} (순 기대 초과수익) |
| G | 신뢰도 등급 A/B/C/D (§14.1) |
| Q | 데이터 품질 상태 (OK / WARN / FAIL) |

### 13.2 진입 조건 (HOLD/SELL → BUY)

모두 만족:
```
B1  E   ≥ δ_B                         δ_B: Short 0.15, Mid 0.15 [PARAM, §13.6에서 선택]
B2  NER > 0
B3  G ∈ {A, B}
B4  regime ≠ CRISIS   또는  (regime = CRISIS 이고 E ≥ δ_B + 0.10)     [PARAM, §13.6 채택 검정]
B5  Q = OK, 유니버스 적격
B6  지속성: Short — 2회 연속 평가일(연속 2거래일) 조건 충족 / Mid — 주간 평가일(그 주 마지막 거래일) 1회
```

### 13.3 진입 조건 (HOLD/BUY → SELL)

**(a) 확률 기반 SELL**
```
S1  E   ≤ −δ_S                        δ_S = 0.15 [PARAM]
S2  μ̂  < 0
S3  G ∈ {A, B}
S4  Q = OK
S5  지속성: B6과 동일
```
**(b) 위험 기반 SELL (`SELL_RISK`)** — Kaminski-Lo(I4): 추세·변동성 조건부 손실 제한
```
R1  vol_pct ≥ 0.95                     (종목 자기 이력 대비 극단 변동성)
R2  TREND ≤ −1.0                       (§7.1 원값: 변동성 정규화 다기간 추세가 강한 하락)
R3  π^HV ≥ 0.5  또는  DILLIQ ≥ ln 2    (시장 고변동성 또는 유동성 2배 악화)
→ 지속성 조건 없이 즉시 발동 (최소 유지기간 무시)
```
`SELL_RISK`는 UI에서 "SELL · 위험 신호"로 구분 표시한다. 이 규칙은 기대수익 개선이 아니라 **꼬리위험 축소** 목적이며, 백테스트에서 (i) BUY 포트폴리오 MDD·CVaR 개선, (ii) Sharpe의 유의한 악화 없음(Ledoit-Wolf 검정 p>0.10)일 때만 채택한다.

### 13.4 유지·이탈 조건 (히스테리시스)

| 현재 상태 | 이탈 조건 | 다음 상태 |
|---|---|---|
| BUY | SELL 조건 충족 | SELL |
| BUY | E < δ_B^exit (= δ_B / 2) 또는 NER ≤ −c^{rt}/2 또는 G = D 또는 Q = FAIL, **그리고** 최소 유지기간 경과 | HOLD |
| SELL | BUY 조건 충족 | BUY |
| SELL | E > −δ_S^exit (= −δ_S / 2) 또는 G = D 또는 Q = FAIL, 그리고 최소 유지기간 경과 | HOLD |
| SELL_RISK | R1–R3 중 2개 이상 해소 & 최소 5거래일 경과 | HOLD (재평가) |

최소 유지기간 [PARAM]: Short 5거래일, Mid 21거래일. `SELL_RISK`와 데이터 FAIL은 유지기간을 무시한다.

### 13.5 HOLD 사유 코드 (HOLD는 중간값이 아니라 "거래하지 않는 것이 합리적인 상태"의 분류)

| 코드 | 조건 | 사용자 문구 |
|---|---|---|
| `H_NO_EDGE` | \|E\| < min(δ_B, δ_S) | 매수·매도 어느 쪽도 통계적 우위가 없습니다 |
| `H_COST` | E ≥ δ_B 이나 NER ≤ 0 | 상승 우위가 있으나 거래비용을 넘을 만큼 크지 않습니다 |
| `H_UNCERTAIN` | 우위 있으나 G ∈ {C, D} | 신호는 있으나 신뢰도가 낮습니다 (사유 하위코드: OOD / 엔진 불일치 / 모델 점검 중) |
| `H_REGIME` | B4 위반 | 시장 위기 국면으로 신규 매수 판단을 보류합니다 |
| `H_PERSIST` | B6/S5 미충족 (신호 발생 1일차) | 신호 확인 중입니다 (다음 평가일 확정) |
| `H_DATA` | Q ≠ OK 또는 이력 부족 | 데이터가 충분하지 않습니다 |
| `H_HOLDING` | 이전 BUY/SELL에서 이탈 조건 미충족이나 진입 조건도 아님 → 실제로는 이전 상태 유지 | (표시 상태는 BUY/SELL 유지 — "유지 중" 배지) |

### 13.6 임계값 선택 절차 (임의 지정 금지)

- 후보 격자: δ_B, δ_S ∈ {0.05, 0.10, 0.15, 0.20, 0.25, 0.30}, 대칭 제약 δ_B = δ_S 를 기본 (자유도 축소).
- 목적함수: 검증 폴드(walk-forward 내부)에서 **BUY 상태 동일가중 포트폴리오의 비용 차감 초과수익 Sharpe**, 제약: 연평균 회전율 ≤ Short 2,500% / Mid 600% [PARAM], 연간 BUY 신호 수 ≥ 유니버스의 5%.
- 선택된 δ의 인접값(±0.05) 성과가 최고값 대비 70% 미만이면 "날카로운 최적점(과적합 의심)"으로 보고하고 인접 3개 평균 성과가 가장 높은 δ를 채택(평탄 영역 선호).
- 격자 크기(6)는 DSR의 시행 수에 포함.

### 13.7 평가 시점과 갱신 주기

| | Short | Mid |
|---|---|---|
| 계산 | 매 거래일 마감 후 | 매 거래일 마감 후 (확률·위험 표시 갱신) |
| 상태 전이 평가 | 매 거래일 | 주 1회 (그 주 마지막 거래일), `SELL_RISK`·FAIL은 매일 |
| 라벨 기간 | t+1 시가 ~ t+11 시가 | t+1 시가 ~ t+64 시가 |
| 표시 문구 | "향후 약 2주(10거래일)" | "향후 약 3개월(63거래일)" |

---

## 14. 신뢰도와 비용

### 14.1 신뢰도 등급 G (확률과 별개 개념)

**확률**은 "결과가 UP/FLAT/DOWN일 가능성"이고, **신뢰도**는 "그 확률 추정과 판단을 얼마나 믿을 수 있는가"이다.

1) 신뢰도 구성 요소 (모두 t일에 관측 가능)

| 요소 | 정의 |
|---|---|
| `c_edge` | \|E\| |
| `c_agree` | 판단 방향 sgn(E)과 같은 부호인 엔진 기여의 비중: Σ_j max(0, sgn(E)·contrib_j) / Σ_j \|contrib_j\| |
| `c_ood` | 학습 Feature 분포 대비 Mahalanobis 거리 D² 의 학습 분포 내 백분위 (공분산: Ledoit-Wolf 수축 추정) |
| `c_regime` | max(π^HV, 1 − π^HV) (국면 확신도), 그리고 최근 10일 π^HV 변동폭 |
| `c_volpct` | vol_pct |
| `c_model` | 해당 시장·기간 모델의 최근 126일 롤링 Brier Skill Score 상태 (`MODEL_VALIDATION_SPEC.md §5`) |

2) 메타 라벨 모형 (J6의 meta-labeling)
```
정답 이벤트 Z = 1[ sgn(E)·X > c^{rt} ]   (판단 방향이 비용을 넘어 맞았는가; \|E\| ≥ 0.05 인 표본만)
q = P(Z = 1 | c_edge, c_agree, c_ood, c_regime, c_volpct)   — L2 로지스틱, 특성 5개, walk-forward OOS 예측값으로만 학습
```
3) 등급 산출
```
q의 검증 구간 분위수로 경계 설정 [PARAM]: A = 상위 20%, B = 다음 30%, C = 다음 30%, D = 하위 20%
하드 캡: c_ood ≥ 0.99 → 최대 C / Q = WARN → 최대 C / Q = FAIL → D / c_model = ALERT → 최대 C
```
4) 등급별 사후 적중률(OOS)과 그 95% 신뢰구간을 모델 성과 페이지에 공개한다. A 등급의 적중률이 B 이하로 역전되면 등급 체계 재검토 (`MODEL_VALIDATION_SPEC.md §5.4`).

### 14.2 거래비용 모델

편도 비용(비율):
```
c^{ow}_{i,t} = fee_m + tax_m(side) + ½·SPREAD_AR_{i,t} + k_imp · σ̂_{i,t} · sqrt( Q_ref / ADV_{i,t} )
c^{rt}       = c^{ow}(buy) + c^{ow}(sell)
```
| 항목 | US [PARAM/VERIFY] | KR [PARAM/VERIFY] |
|---|---|---|
| fee_m | 0 (무수수료 브로커 가정) + 규제 수수료(SEC Section 31, 매도) [VERIFY 현행 요율] | 위탁수수료 0.015% (양방향) [PARAM] |
| tax_m(sell) | 0 | 증권거래세+농특세: 시점별 표 적용 (`BACKTEST_SPEC.md §5.3`) |
| ½ spread | Abdi-Ranaldo 63일 중앙값의 절반, 하한 = ½ tick / P | 동일 (호가단위 표 적용) |
| 시장충격 | k_imp = 1.0, Q_ref = $10,000 (개인 기준) | Q_ref = ₩10,000,000 |

시장충격 형태는 제곱근 법칙(Tóth et al. 2011, Physical Review X 1:021006, 10.1103/PhysRevX.1.021006)을 사용. 개인 주문 규모에선 무시할 수준이지만 저유동 종목의 경고용으로 유지.

---

## 15. Feature 추가·제거 절차 (모델 복잡도 통제)

새 Feature f를 Champion에 추가하려면 모두 만족:
1. `QUANT_RESEARCH.md`에 근거 연구 카드 존재 (Core/Supporting) 또는 Experimental에서 승격 사유 문서화.
2. 정의·부호·창 길이를 **실험 전에** 실험 레지스트리에 등록.
3. 기존 Feature에 대한 잔차화 후(Fama-MacBeth 회귀에서 기존 엔진 점수 통제) 평균 계수의 Newey-West t > 3.0 (L2).
4. 모델 추가 시 OOS 로그손실 DM 검정 p < 0.05, MCS 포함.
5. US·KR 각각 독립 판단 (한 시장 통과가 다른 시장 채택 근거 아님).
6. Benjamini-Hochberg FDR 10%로 해당 분기 검정된 모든 후보를 함께 보정.

Feature 제거: 최근 3년 OOS에서 잔차 계수 t < 1.0이 2분기 연속이면 제거 후보 → 제거 시 OOS 성과가 유의하게 악화되지 않으면 제거.

---

## 16. 출력 명세

### 16.1 종목 신호 레코드 (JSON, 원장에 정규화 직렬화 후 해시)

```json
{
  "schema_version": "1.0",
  "signal_id": "US-10-2026-11-06-SEC000123",
  "market": "US",
  "security_id": "SEC000123",
  "symbol": "AAPL",
  "as_of_date": "2026-11-06",
  "data_cutoff_utc": "2026-11-06T22:30:00Z",
  "horizon_days": 10,
  "decision": "HOLD",
  "decision_reason": "H_NO_EDGE",
  "state_since": "2026-10-28",
  "probabilities": {"up": 0.34, "flat": 0.37, "down": 0.29, "display_allowed": true},
  "edge": 0.05,
  "expected_excess_return": 0.0021,
  "round_trip_cost": 0.0009,
  "confidence": {"grade": "B", "q": 0.56, "caps": []},
  "contributions": {"momentum": 0.08, "reversal": -0.11, "trend": 0.05, "low_risk": 0.02, "volume": 0.00, "regime": -0.01},
  "risk": {"exp_vol_h": 0.052, "cvar95_h": -0.094, "vol_pct": 0.41, "liquidity_flag": false},
  "regime": {"code": "BULL_CALM", "p_high_vol": 0.18},
  "data_quality": "OK",
  "model_version": "US-S-1.0.0",
  "feature_snapshot_hash": "sha256:…",
  "code_commit": "…"
}
```
(수치는 형식 예시이며 실제 결과가 아니다.)

### 16.2 결정론 요구사항
- 동일 (data snapshot hash, model_version, code_commit) → 동일 출력. 부동소수점 결과는 소수 10자리 반올림 후 직렬화.
- 모든 난수는 `seed = hash(model_version, market, horizon)`에서 파생.

---

## 17. 장기(Long-Term) 모델 확장 검토 (Phase 3 연구)

- 기간 h = 252, 월간 리밸런싱. 후보: 장기 반전(B6), 가치·수익성·품질(F3, F6 — 재무 데이터 필요), 저베타(F5).
- 전제: DART/SEC EDGAR 기반 **Point-in-Time 재무 데이터**(공시일 기준) 구축. 재무 데이터 도입은 Mid 모델 대비 증분 검정 통과 시에만.
- 표본 수 문제: 연간 독립 관측이 적어 확률 보정 신뢰구간이 넓음 → 확률 비표시·등급만 표시 가능성.

---

## 18. 미해결 설계 이슈

| ID | 이슈 | 해결 방법 |
|---|---|---|
| Q1 | KR에서 모멘텀이 유의하지 않을 가능성(A11) | Mid-KR은 S_mom 계수가 부호 제약으로 0이 될 수 있음 — 그 자체로 허용. Mid-KR이 EW 기준 대비 유의성 미달 시 출시 보류 |
| Q2 | 공매도 금지 기간 등 구조 변화 (KR) | 구조변화 더미를 평가 분해에만 사용, 학습 입력 아님 |
| Q3 | 벤치마크 ETF 이전 기간(ETF 상장 전) | 지수 총수익 지수 또는 유니버스 가치가중 수익으로 대체 [VERIFY 데이터] |
| Q4 | 산업 분류 PIT 이력 | 미확보 시 현재 분류 사용 + 편향 가능성 명시 |
| Q5 | 이벤트(실적 발표일) 정보 없음 | 갭 플래그로 사후 진단만; 실적일 데이터 확보 시 Phase 2 검토 |
