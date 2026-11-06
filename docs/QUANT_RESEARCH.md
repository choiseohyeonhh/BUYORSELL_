# QUANT_RESEARCH — 학술 연구 조사 및 전략 평가

버전 0.1 · 관련 문서: `QUANT_MODEL_SPEC.md`(채택 결과의 수학적 구현), `BACKTEST_SPEC.md`(재현 실험)

---

## 0. 조사 방법과 정직성 고지

### 0.1 방법

1. 9개 연구 분야(A–I) + 방법론(J: 백테스트 통계, K: 예측 결합·확률 보정, L: 이상현상 재현성)로 구분.
2. 각 분야에서 (a) 효과를 처음 문서화한 원전, (b) 국제·장기 표본 재검증, (c) 실패·한계를 보고한 반론 연구를 함께 수집. "효과가 있다"는 연구만 모으는 확증편향을 피하기 위해 (c)를 필수로 포함.
3. 출처는 Journal of Finance(JF), Journal of Financial Economics(JFE), Review of Financial Studies(RFS), Econometrica, Journal of Portfolio Management(JPM), Quantitative Finance, Journal of Financial Markets, Management Science, NBER/SSRN/arXiv 등 동료심사 저널 및 공인 워킹페이퍼로 한정. 블로그·유튜브·벤더 마케팅 자료는 근거로 쓰지 않음.

### 0.2 검증 상태 (중요)

- 본 문서의 서지 정보(저자, 연도, 저널, 권호, DOI)는 표준 서지에 근거해 기재했다. **이번 설계 작업 세션에서 각 논문의 PDF 원문을 전부 다시 열람·대조하지는 않았다.** 따라서 모든 항목에 `원문 대조` 상태를 두고, 개발 착수 전 Phase 1(`DEVELOPMENT_ROADMAP.md`)에서 리서치 담당자가 원문을 대조해 `✔`로 갱신한다.
- 수치 결과는 해당 논문의 **핵심 결론으로 널리 인용되는 정성적·정량적 결과만** 기재했고, 불확실한 수치는 쓰지 않았다. 표의 "실증 성과" 열은 *논문이 보고한 결과*이지 BUY OR SELL의 기대 성과가 아니다.
- 가장 중요한 원칙: **논문의 결과는 우리 모델이 작동한다는 증거가 아니다.** 논문은 가설의 사전 확률(prior)을 높여 줄 뿐이며, 채택 여부의 최종 판단은 자사 데이터·자사 체결 가정 하의 재현 실험(`BACKTEST_SPEC.md §9 Replication Gate`)으로 결정한다.

### 0.3 평가 기준과 분류

각 연구를 10개 기준(학술 신뢰성, 데이터 규모·품질, 기간 적절성, 실증 수준, 거래비용 반영, 시장 적용성, 재현 가능성, 최근 유효성, 구현 난이도, 기여도)으로 1–5점 평가하고 다음으로 분류한다.

| 분류 | 정의 |
|---|---|
| **Core** | 다수 시장·장기 표본에서 재현되었고, 일별 OHLCV로 구현 가능하며, 비용 반영 후에도 경제적 의미가 남을 가능성이 높음 → 핵심 엔진의 Feature 후보 |
| **Supporting** | 효과 자체보다 *조건부 정보*(위험 조정, 국면 조건, 신호 정규화)로 가치가 큼 → 보조 입력·위험 게이트 |
| **Experimental** | 근거가 혼재하거나, 실무 지표지만 학술 근거가 약하거나, 데이터 제약이 큼 → Phase 2 이후 증분 검증 후에만 도입 |
| **Rejected** | 다중검정·비용 반영 후 근거 소멸, 또는 데이터 확보 불가 → MVP 제외 |

### 0.4 요약 판정표

| 분야 | 판정 | 핵심 근거 | 서비스 반영 |
|---|---|---|---|
| A. 횡단면 모멘텀 (12-1) | **Core** (중기) | Jegadeesh-Titman 1993/2001, Asness-Moskowitz-Pedersen 2013 | Momentum Engine |
| A. 잔차 모멘텀 | **Core 후보** | Blitz-Huij-Martens 2011 | Momentum Engine (크래시 완화) |
| A. 52주 고가 근접도 | **Core 후보** | George-Hwang 2004 | Momentum Engine |
| A. 모멘텀 크래시·변동성 스케일링 | **Supporting** | Daniel-Moskowitz 2016, Barroso-Santa-Clara 2015 | 국면 게이트, 위험 조정 |
| A. 한국 시장 모멘텀 | **불확실 (시장별 재추정 필수)** | Chui-Titman-Wei 2010 (집단주의 문화권 약함), Griffin-Ji-Martin 2003 | 시장별 독립 가중치 |
| B. 단기 반전 (1주~1개월) | **Core** (단기) | Jegadeesh 1990, Lehmann 1990, Da-Liu-Schaumburg 2014 | Mean Reversion Engine |
| B. 장기 반전 (3–5년) | **Rejected** (MVP) | De Bondt-Thaler 1985 — 투자기간 불일치 | 장기 모델 연구 |
| B. 페어 트레이딩·공적분 | **Experimental** | Gatev-Goetzmann-Rouwenhorst 2006 (최근 수익 감소 보고) | 개별 종목 신호와 구조 불일치 |
| B. RSI·볼린저 고정 임계값 | **Experimental** | 학술 근거 약함; 일반 기술적 분석의 데이터 스누핑 문제(Sullivan-Timmermann-White 1999) | 연속 Feature로만 검증 |
| C. 시계열 추세 (TSMOM, 이동평균) | **Core** (중기) / **Supporting** (단기 국면) | Moskowitz-Ooi-Pedersen 2012, Hurst-Ooi-Pedersen 2017, Han-Yang-Zhou 2013 | Trend Engine |
| C. MACD·Donchian 별도 지표 | **Experimental (중복)** | 선형 필터로 MA·TSMOM과 정보가 중복(Levine-Pedersen 2016) | 추가하지 않음 |
| D. 변동성 군집·예측 (GARCH, HAR) | **Core** (위험 측정) | Engle 1982, Bollerslev 1986, Corsi 2009, Hansen-Lunde 2005 | Volatility Engine |
| D. 고유변동성 퍼즐 / 저변동성 | **Supporting** | Ang-Hodrick-Xing-Zhang 2006, Frazzini-Pedersen 2014 | Feature (음의 기대수익) |
| D. 변동성 관리 포트폴리오 | **Supporting (논쟁 중)** | Moreira-Muir 2017 vs Cederburg et al. 2020 | 신호 정규화에만 사용 |
| E. Amihud 비유동성 | **Supporting** | Amihud 2002 | 유동성 필터·비용 모델·Feature |
| E. 거래량-모멘텀 상호작용 | **Supporting** | Lee-Swaminathan 2000, Gervais-Kaniel-Mingelgrin 2001 | 조건부 Feature |
| E. 거래량-수익 자기상관 | **Supporting** | Campbell-Grossman-Wang 1993, Llorente et al. 2002 | 단기 반전의 조건 변수 |
| E. OBV·A/D·MFI | **Experimental** | 원전이 실무서, 동료심사 근거 희박 | 증분 검증 후 |
| E. 주문흐름 불균형·호가 스프레드 | **Rejected** (MVP) | Cont-Kukanov-Stoikov 2014 — 틱 데이터 필요, 초단기 | 데이터 확보 불가 |
| F. 팩터 모델 (FF3/4/5) | **Supporting** (위험 분해·잔차화) | Fama-French 1993/2015, Carhart 1997 | 잔차 모멘텀, 성과 귀속 |
| F. BAB·저베타 | **Supporting** | Frazzini-Pedersen 2014 | Feature 후보 |
| F. QMJ·재무 팩터 | **Experimental** | Asness-Frazzini-Pedersen 2019 | 재무 데이터 도입 후 |
| G. 머신러닝 (GBDT 등) | **Experimental → 조건부 Core** | Gu-Kelly-Xiu 2020, Krauss-Do-Huck 2017 | Challenger 모델; 선형 기준 대비 유의 개선 시에만 |
| H. 국면 (Markov switching, HMM) | **Supporting** | Hamilton 1989, Ang-Bekaert 2002 | Market Regime Engine (필터링 확률만) |
| H. 시장 상태와 모멘텀 | **Supporting** | Cooper-Gutierrez-Hameed 2004 | 모멘텀 가중 조건 |
| I. 변동성 타깃팅 | **Supporting** | Harvey et al. 2018 | 위험 표시·정규화 |
| I. Kelly | **Rejected** (사용자 노출) | Kelly 1956 — 추정오차에 극도로 민감 | 내부 분석에만 |
| I. 손절매 연구 | **Supporting** | Kaminski-Lo 2014 | SELL 위험 게이트 설계 근거 |
| I. CVaR | **Supporting** | Artzner et al. 1999, Rockafellar-Uryasev 2000 | 하방 위험 표시 |
| J. 백테스트 과최적화 | **Core (방법론)** | Bailey-López de Prado 2014, Bailey et al. 2017, White 2000, Hansen 2005 | 검증 체계 전반 |
| K. 예측 결합·확률 보정 | **Core (방법론)** | Timmermann 2006, Claeskens et al. 2016, Niculescu-Mizil-Caruana 2005 | 앙상블·보정 |
| L. 이상현상 재현성 | **Core (방법론)** | McLean-Pontiff 2016, Harvey-Liu-Zhu 2016, Hou-Xue-Zhang 2020, Jensen-Kelly-Pedersen 2023 | 기대 성과 할인, t>3 기준 |

---

## 1. 연구 카드 형식

각 연구는 다음 형식으로 기록한다. 공간 절약을 위해 일부 필드는 묶어서 기재.

```
[ID] 논문명
- 저자 / 연도 / 저널(권(호):쪽) / DOI·URL
- 목적 | 데이터·기간 | 방법
- 핵심 발견 | 실증 성과(논문 보고)
- 한계 | 재현 가능성 | 서비스 적용 | 분류 및 이유 | 원문 대조: ☐
```

DOI는 `https://doi.org/<DOI>` 로 접근 가능하다.

---

## A. 모멘텀 (Momentum)

### A1. Returns to Buying Winners and Selling Losers: Implications for Stock Market Efficiency
- Jegadeesh, N., Titman, S. / 1993 / JF 48(1):65–91 / 10.1111/j.1540-6261.1993.tb04702.x
- 목적: 과거 3–12개월 승자 매수·패자 매도 전략의 수익성 검증 | 데이터: NYSE/AMEX 1965–1989 | 방법: J개월 형성·K개월 보유 포트폴리오, 중첩 포트폴리오, 형성 후 1주 건너뛰기 변형
- 핵심 발견: 3–12개월 형성·보유 전략이 유의한 양(+)의 수익. 6/6 전략 월 약 1% 수준. 보유 12개월 이후 일부 반전.
- 실증 성과: 체계적 위험이나 선행-후행 반응으로 설명되지 않는 초과수익 보고.
- 한계: 거래비용 미반영(이후 연구에서 소형주·저유동성 집중 시 비용 문제 제기), 미국 단일 시장.
- 재현: 높음 (일별/월별 가격만 필요) | 적용: 중기 모멘텀 Feature `MOM_12_1`의 근거 | **Core** — 이후 국제·장기 표본에서 반복 재현(A2, A3, A10) | 원문 대조: ☐

### A2. Profitability of Momentum Strategies: An Evaluation of Alternative Explanations
- Jegadeesh, N., Titman, S. / 2001 / JF 56(2):699–720 / 10.1111/0022-1082.00342
- 목적: 1993 논문 이후(1990년대) 표본 외 검증과 행동재무 설명 평가 | 데이터: 1965–1998 | 방법: 원 논문 이후 기간 OOS 검증, 장기 반전 분석
- 핵심 발견: 원 논문 발표 이후에도 모멘텀 수익 지속(표본 외 확인). 보유 후 13–60개월에 수익 반전 → 지연 과잉반응(행동) 설명과 정합.
- 한계: 미국 한정 | 재현: 높음 | 적용: 중기 보유기간 상한(≤12개월) 설계 근거 | **Core** | 원문 대조: ☐

### A3. Value and Momentum Everywhere
- Asness, C., Moskowitz, T., Pedersen, L. / 2013 / JF 68(3):929–985 / 10.1111/jofi.12021
- 목적: 8개 시장·자산군에서 가치와 모멘텀의 보편성 검증 | 데이터: 미국·영국·유럽·일본 주식, 국가 지수, 채권, 통화, 원자재 (1972–2011 전후) | 방법: 순위 가중 포트폴리오, 공통 요인 분석
- 핵심 발견: 모멘텀 프리미엄은 대부분 시장·자산군에서 양(+). 가치와 모멘텀은 음의 상관 → 결합 시 위험조정 성과 개선. **일본 주식에서 모멘텀은 약함**.
- 한계: 대형·유동 종목 중심 | 재현: 높음 (AQR 데이터 라이브러리 공개) | 적용: 모멘텀의 범용성 prior, 아시아 시장 약세 경고 | **Core** | 원문 대조: ☐

### A4. Time Series Momentum
- Moskowitz, T., Ooi, Y.H., Pedersen, L. / 2012 / JFE 104(2):228–250 / 10.1016/j.jfineco.2011.11.003
- 목적: 자산 자신의 과거 수익률 부호가 미래 수익을 예측하는가(시계열 모멘텀) | 데이터: 58개 유동 선물·선도 (주가지수, 통화, 원자재, 채권), 1965–2009 | 방법: 변동성 정규화 수익률의 시계열 회귀, 12개월 lookback·1개월 보유 전략
- 핵심 발견: 1–12개월 수익률이 이후 수익을 양(+)으로 예측, 더 긴 기간에선 부분 반전. 극단적 시장에서 성과 양호.
- 한계: 개별 주식 아닌 선물; 개별 주식에선 횡단면 모멘텀과 중첩 | 재현: 높음 | 적용: Trend Engine의 변동성 정규화 TSMOM 정의, 시장 지수 국면 신호 | **Core** (개별 주식 적용은 HYP로 재검증) | 원문 대조: ☐

### A5. Momentum Crashes
- Daniel, K., Moskowitz, T. / 2016 / JFE 122(2):221–247 / 10.1016/j.jfineco.2015.12.002
- 목적: 모멘텀 전략의 극단 손실(크래시) 조건 규명 | 데이터: 미국 1927–2013 및 국제·타 자산 | 방법: 국면 조건부 수익·변동성 회귀, 동적 가중 전략
- 핵심 발견: 크래시는 **약세장 이후 + 높은 시장 변동성 + 시장 반등** 국면에 집중(1932, 2009). 패자 포트폴리오가 높은 옵션성 베타를 가짐. 예측된 평균·분산 기반 동적 가중이 Sharpe 대폭 개선.
- 한계: 동적 가중 파라미터의 추정 위험 | 재현: 높음 | 적용: 모멘텀 엔진의 **국면 조건부 가중 축소** 규칙(`QUANT_MODEL_SPEC §5.6`) | **Supporting** | 원문 대조: ☐

### A6. Momentum Has Its Moments
- Barroso, P., Santa-Clara, P. / 2015 / JFE 116(1):111–120 / 10.1016/j.jfineco.2014.11.010
- 목적: 모멘텀의 위험(변동성) 관리 효과 | 데이터: 미국 1927–2011 | 방법: 직전 6개월 일별 실현분산으로 모멘텀 노출 스케일링
- 핵심 발견: 모멘텀 전략 자체의 변동성은 예측 가능하고, 이를 스케일링하면 크래시 대부분 제거, Sharpe 약 0.53 → 0.97로 개선(논문 보고).
- 한계: 포트폴리오 수준 결과 — 개별 종목 신호로 직접 이전 불가 | 재현: 높음 | 적용: 모멘텀 팩터 변동성이 높을 때 Momentum Engine 가중 하향(국면 입력) | **Supporting** | 원문 대조: ☐

### A7. The 52-Week High and Momentum Investing
- George, T., Hwang, C.Y. / 2004 / JF 59(5):2145–2176 / 10.1111/j.1540-6261.2004.00695.x
- 목적: 52주 고가 대비 현재가 비율의 예측력 | 데이터: 미국 1963–2001 | 방법: 비율 기준 포트폴리오, JT 모멘텀과 경합 회귀
- 핵심 발견: 52주 고가 근접도가 과거 수익률 모멘텀보다 미래 수익을 더 잘 설명하며, 장기 반전이 적음 → 기준점(anchoring) 편향 설명.
- 한계: 이후 국제 연구에서 결과 혼재 | 재현: 높음 | 적용: `HIGH52` Feature | **Core 후보** (자사 재현 실험 필수) | 원문 대조: ☐

### A8. Residual Momentum
- Blitz, D., Huij, J., Martens, M. / 2011 / Journal of Empirical Finance 18(3):506–521 / 10.1016/j.jempfin.2011.01.003
- 목적: 팩터 노출을 제거한 잔차 수익률 기반 모멘텀 | 데이터: 미국 1930–2009 | 방법: FF3 롤링 회귀 잔차의 12-1 누적, 잔차 변동성으로 표준화
- 핵심 발견: 잔차 모멘텀은 전통 모멘텀과 유사한 수익에 변동성은 약 절반, 크래시 노출 감소.
- 한계: 회귀 추정창·팩터 선택에 민감 | 재현: 중간(팩터 수익률 필요 — 미국은 Kenneth French 라이브러리, 한국은 자체 구축) | 적용: `RMOM` Feature, 크래시 완화 | **Core 후보** | 원문 대조: ☐

### A9. Is Momentum Really Momentum?
- Novy-Marx, R. / 2012 / JFE 103(3):429–453 / 10.1016/j.jfineco.2011.05.003
- 핵심 발견: 모멘텀 수익은 주로 12–7개월 전 수익("중간 기간 과거 성과")에서 발생, 최근 6–2개월의 기여는 작음.
- 이후 반론 연구가 존재(국제 표본에서 약화) | 적용: 12-1을 (12-7)과 (6-2)로 분해해 **둘 다 Feature 후보로 두고 자사 데이터로 결정** | **Supporting** | 원문 대조: ☐

### A10. Momentum Investing and Business Cycle Risk: Evidence from Pole to Pole
- Griffin, J., Ji, X., Martin, J.S. / 2003 / JF 58(6):2515–2547 / 10.1046/j.1540-6261.2003.00614.x
- 핵심 발견: 40개국에서 모멘텀 이익이 보편적이나 **아시아 시장에서 약함**; 거시 위험으로 설명되지 않음.
- 적용: 한국 시장 모멘텀 효과에 대한 낮은 prior | **Supporting** | 원문 대조: ☐

### A11. Individualism and Momentum around the World
- Chui, A., Titman, S., Wei, K.C.J. / 2010 / JF 65(1):361–392 / 10.1111/j.1540-6261.2009.01532.x
- 핵심 발견: 국가 개인주의 지수와 모멘텀 이익이 양의 관계. 개인주의 지수가 낮은 **일본·한국 등 동아시아에서 모멘텀이 약하거나 유의하지 않음**.
- 적용: **한국 모델은 모멘텀 가중을 미국에서 가져오지 않고 독립 추정**; 한국에서 모멘텀이 유의하지 않으면 중기 한국 모델은 추세·저변동성·유동성 중심으로 구성 | **Supporting (시장 구분 근거)** | 원문 대조: ☐
- 추가 조사 과제 [VERIFY]: 한국 시장 모멘텀·반전에 관한 국내 학술지(재무연구, 증권학회지, 한국증권학회지 등) 실증 연구를 Phase 1에서 원문 기반으로 별도 수집. 본 문서에는 원문 미확인 국내 연구를 기재하지 않았다.

### A12. Market States and Momentum
- Cooper, M., Gutierrez, R., Hameed, A. / 2004 / JF 59(3):1345–1365 / 10.1111/j.1540-6261.2004.00665.x
- 핵심 발견: 직전 36개월 시장 수익이 양(UP)일 때만 모멘텀이 유의한 양의 수익, DOWN 이후에는 음(-)에 가까움.
- 적용: Market Regime Engine의 장기 시장 상태 변수(`MKT_STATE_36M`)와 모멘텀 상호작용 | **Supporting** | 원문 대조: ☐

### A13. 모멘텀의 원인에 관한 이론 (설명용)
- Barberis, Shleifer, Vishny (1998) "A Model of Investor Sentiment", JFE 49(3):307–343, 10.1016/S0304-405X(98)00027-0 — 보수주의·대표성 편향
- Daniel, Hirshleifer, Subrahmanyam (1998) "Investor Psychology and Security Market Under- and Overreactions", JF 53(6):1839–1885, 10.1111/0022-1082.00077 — 과신·자기귀인
- Hong, Stein (1999) "A Unified Theory of Underreaction, Momentum Trading, and Overreaction", JF 54(6):2143–2184, 10.1111/0022-1082.00184 — 점진적 정보 확산
- Moskowitz, Grinblatt (1999) "Do Industries Explain Momentum?", JF 54(4):1249–1290, 10.1111/0022-1082.00146 — 산업 모멘텀
- Grundy, Martin (2001) "Understanding the Nature of the Risks and the Source of the Rewards to Momentum Investing", RFS 14(1):29–78, 10.1093/rfs/14.1.29 — 동적 팩터 노출
- Rouwenhorst (1998) "International Momentum Strategies", JF 53(1):267–284, 10.1111/0022-1082.95722 — 유럽 12개국 재현
- 용도: 사용자 설명("왜 모멘텀이 작동하는가") 및 장기 유효성 prior. 직접 Feature 아님. 원문 대조: ☐

### A 분야 결론

| 질문 | 결론 |
|---|---|
| 원인 | 정보의 점진적 확산과 투자자 과소반응(행동), 일부 위험 보상. 단일 합의 없음. |
| 관측 기간 | 형성 3–12개월, 최근 1개월 제외(단기 반전과 분리). 12-1이 표준. |
| 리밸런싱 | 월 1회가 표준(중기 모델). 주간 갱신은 회전율만 높임 → 중기 모델은 월간 결정 + 주간 모니터링. |
| 시장별 차이 | 미국·유럽 강함, 일본·한국 약함 → **시장별 독립 추정 필수** |
| 크래시 | 약세장 후 고변동성 반등기 → 국면 조건 축소 |
| 한계 | 비용, 공매도 제약(우리는 롱 판단만 제공하므로 패자측 수익 일부는 서비스에서 실현 불가), 발표 후 프리미엄 감소(L 참조) |

---

## B. 평균 회귀 (Mean Reversion)

### B1. Evidence of Predictable Behavior of Security Returns
- Jegadeesh, N. / 1990 / JF 45(3):881–898 / 10.1111/j.1540-6261.1990.tb05110.x
- 핵심 발견: 개별 주식의 **1개월 수익률에 유의한 음의 1차 자기상관**(단기 반전), 12개월 수준에선 양의 자기상관.
- 적용: 단기 모델의 `REV_21` 근거 | **Core** (단기) | 원문 대조: ☐

### B2. Fads, Martingales, and Market Efficiency
- Lehmann, B. / 1990 / QJE 105(1):1–28 / 10.2307/2937816
- 핵심 발견: 주간 수익률 반전 전략이 유의한 수익. 다만 회전율이 매우 높아 거래비용에 민감.
- 적용: `REV_5` 근거, **비용 반영 필수** | **Core 후보** (단기, 비용 후 재검증) | 원문 대조: ☐

### B3. A Closer Look at the Short-Term Return Reversal
- Da, Z., Liu, Q., Schaumburg, E. / 2014 / Management Science 60(3):658–674 / 10.1287/mnsc.2013.1766
- 핵심 발견: 단기 반전에서 **산업 수익·뉴스(현금흐름 정보) 성분을 제거한 잔차 반전**이 훨씬 강하고 지속적. 원 반전 신호는 정보 기반 움직임(반전하지 않음)과 유동성 기반 움직임(반전)이 섞여 있음.
- 적용: `RREV_21`(산업·시장 조정 잔차 반전)을 원 반전보다 우선 후보로 | **Core 후보** | 원문 대조: ☐

### B4. Evaporating Liquidity
- Nagel, S. / 2012 / RFS 25(7):2005–2039 / 10.1093/rfs/hhs066
- 핵심 발견: 단기 반전 수익은 유동성 공급의 대가이며, **시장 변동성(VIX)이 높을 때 크게 증가**.
- 적용: 단기 반전 신호 강도와 변동성 국면의 상호작용 Feature | **Supporting** | 원문 대조: ☐

### B5. Liquidity and Autocorrelations in Individual Stock Returns
- Avramov, D., Chordia, T., Goyal, A. / 2006 / JF 61(5):2365–2394 / 10.1111/j.1540-6261.2006.01060.x
- 핵심 발견: 단기 반전 이익은 **고회전·비유동 종목**에 집중되며 거래비용 반영 시 상당 부분 소멸.
- 적용: 반전 신호는 유동성 필터 통과 종목에만, 비용 차감 기대수익으로 판단 | **Supporting (경고)** | 원문 대조: ☐

### B6. Does the Stock Market Overreact?
- De Bondt, W., Thaler, R. / 1985 / JF 40(3):793–805 / 10.1111/j.1540-6261.1985.tb05004.x
- 핵심 발견: 과거 3–5년 패자가 이후 승자를 능가(장기 반전).
- 적용: 투자기간(1–6개월)과 불일치 | **Rejected (MVP)**, 장기 모델 연구 후보 | 원문 대조: ☐

### B7. Mean Reversion in Stock Prices: Evidence and Implications / Stock Market Prices Do Not Follow Random Walks
- Poterba, J., Summers, L. (1988) JFE 22(1):27–59, 10.1016/0304-405X(88)90021-9
- Lo, A., MacKinlay, A.C. (1988) RFS 1(1):41–66, 10.1093/rfs/1.1.41 — 분산비(variance ratio) 검정
- 적용: 종목별 평균회귀 성향 측정 도구(분산비 VR(q))로 **Supporting** — Mean Reversion Engine의 조건 변수 | 원문 대조: ☐

### B8. Pairs Trading: Performance of a Relative-Value Arbitrage Rule
- Gatev, E., Goetzmann, W., Rouwenhorst, K.G. / 2006 / RFS 19(3):797–827 / 10.1093/rfs/hhj020
- 핵심 발견: 거리 기반 페어 트레이딩이 1962–2002 기간 유의한 초과수익. 저자들이 후기 표본에서 수익 감소를 보고.
- 한계: 신호가 "종목 A의 매수/매도"가 아니라 "A 대비 B의 상대가치" → 단일 종목 BUY/HOLD/SELL 구조와 불일치, 공매도 필요.
- **Experimental** — 개별 종목 신호의 보조 정보("동종 그룹 대비 괴리")로만 향후 검토 | 원문 대조: ☐

### B9. Statistical Arbitrage in the US Equities Market
- Avellaneda, M., Lee, J.-H. / 2010 / Quantitative Finance 10(7):761–782 / 10.1080/14697680903124632
- 방법: PCA/ETF 팩터 잔차를 **Ornstein-Uhlenbeck 과정**으로 모형화, s-score로 진입·청산.
- 핵심 발견: 2000년대 중반 이후 성과 저하 보고, 거래량 정보 추가 시 개선.
- 적용: OU 반감기 추정을 Mean Reversion Engine의 **조건 변수**(회귀 속도)로 사용. 고빈도 롱숏 전략 자체는 범위 밖 | **Experimental** | 원문 대조: ☐

### B10. 공적분·OU 이론
- Engle, R., Granger, C. (1987) "Co-integration and Error Correction", Econometrica 55(2):251–276, 10.2307/1913236
- Uhlenbeck, G., Ornstein, L. (1930) "On the Theory of the Brownian Motion", Physical Review 36(5):823–841, 10.1103/PhysRev.36.823
- 용도: 수학적 정의 근거 | 원문 대조: ☐

### B11. RSI, 볼린저 밴드, Z-Score (실무 지표)
- 원전: Wilder, J.W. (1978) *New Concepts in Technical Trading Systems* (RSI); Bollinger, J. (2001) *Bollinger on Bollinger Bands*. 동료심사 연구 아님.
- 관련 학술 검증: Brock-Lakonishok-LeBaron 1992(C1), Sullivan-Timmermann-White 1999(C2), Park-Irwin 2007(C7) — 고정 규칙형 기술적 분석은 데이터 스누핑 보정·비용 반영 시 근거가 크게 약화.
- 판정: **Experimental.** 70/30 같은 전통 임계값은 쓰지 않는다. RSI(14)·%B는 연속값으로 변환해 단기 반전 Feature와의 **증분 예측력**(정보 중복 제거 후)을 검정하고, 유의하지 않으면 제외한다. 사전 판단: RSI는 단기 반전(REV_5/REV_21)과 상관이 매우 높아 증분 정보가 작을 가능성이 높다 [HYP].

### B 분야 결론
- 1주~1개월 반전은 가장 잘 재현된 단기 효과이나 **유동성 공급 보상** 성격 → 비용·유동성·변동성 국면 조건부로만 신호화.
- 평균회귀가 실패하는 상황: 정보(뉴스·실적)에 의한 가격 변화(B3), 강한 추세장, 유동성 위기 초기. → 산업·시장 조정 잔차 사용, 거래량 급증 조건, 추세 강도 조건으로 완화.
- 페어/공적분: 서비스 구조와 불일치하여 MVP 제외.

---

## C. 추세 추종 (Trend Following)

### C1. Simple Technical Trading Rules and the Stochastic Properties of Stock Returns
- Brock, W., Lakonishok, J., LeBaron, B. / 1992 / JF 47(5):1731–1764 / 10.1111/j.1540-6261.1992.tb04681.x
- 핵심 발견: DJIA 1897–1986에서 이동평균·박스권 돌파 규칙이 매수 신호 후 더 높은 수익, 부트스트랩으로 귀무모형 기각.
- 한계: 거래비용 미반영, 단일 지수 | **Supporting** (C2로 강하게 반박됨) | 원문 대조: ☐

### C2. Data-Snooping, Technical Trading Rule Performance, and the Bootstrap
- Sullivan, R., Timmermann, A., White, H. / 1999 / JF 54(5):1647–1691 / 10.1111/0022-1082.00163
- 핵심 발견: 약 8,000개 규칙 전체를 대상으로 White Reality Check 적용 시 BLL 표본 기간에선 우월 규칙이 유의하나, **이후 표본(1987–1996)에선 유의하지 않음**.
- 적용: 규칙 탐색에 대한 다중검정 보정 필수(J 분야) | **Core (방법론적 경고)** | 원문 대조: ☐

### C3. A Century of Evidence on Trend-Following Investing
- Hurst, B., Ooi, Y.H., Pedersen, L. / 2017 / JPM 44(1):15–29 / 10.3905/jpm.2017.44.1.015
- 핵심 발견: 1880년 이후 67개 시장(주가지수·채권·통화·원자재) 1·3·12개월 TSMOM 결합이 수십 년 단위로 일관된 양의 성과, 위기 국면 방어.
- 한계: 선물 기반, 개별 주식 아님 | 적용: **다기간 결합**(1/3/12개월) 설계의 근거, 시장 지수 추세 국면 | **Core** (지수·국면) | 원문 대조: ☐

### C4. Two Centuries of Trend Following
- Lempérière, Y., Deremble, C., Seager, P., Potters, M., Bouchaud, J.-P. / 2014 / Journal of Investment Strategies 3(3):41–61 / arXiv:1404.3274 (https://arxiv.org/abs/1404.3274)
- 핵심 발견: 지수가중 이동평균 기반 추세 신호의 장기 유의성 (t-stat이 매우 높음 보고).
- **Supporting** | 원문 대조: ☐

### C5. Which Trend Is Your Friend?
- Levine, A., Pedersen, L. / 2016 / Financial Analysts Journal 72(3):51–66 / 10.2469/faj.v72.n3.3
- 핵심 발견: 이동평균 교차, TSMOM, MACD 등 대부분의 추세 지표는 **과거 수익률의 선형 가중합(필터)**으로 표현되며 서로 정보가 크게 중복.
- 적용: **MACD·이중 MA 교차·Donchian을 별도 엔진 신호로 추가하지 않는 근거.** Trend Engine은 다기간 변동성 정규화 수익률 결합 1개 계열 + MA 기울기로 충분 [HYP 검증] | **Core (중복 제거 근거)** | 원문 대조: ☐

### C6. A New Anomaly: The Cross-Sectional Profitability of Technical Analysis
- Han, Y., Yang, K., Zhou, G. / 2013 / JFQA 48(5):1433–1461 / 10.1017/S0022109013000586
- 핵심 발견: 변동성 상위 10분위 포트폴리오에서 이동평균 타이밍이 매수보유 대비 유의한 초과수익; 고변동성 종목일수록 효과 큼.
- 적용: 추세 신호와 변동성의 상호작용 Feature | **Supporting** | 원문 대조: ☐

### C7. What Do We Know About the Profitability of Technical Analysis?
- Park, C.-H., Irwin, S. / 2007 / Journal of Economic Surveys 21(4):786–826 / 10.1111/j.1467-6419.2007.00519.x
- 핵심 발견(서베이): 다수 연구가 수익성 보고하나 데이터 스누핑, 사후 규칙 선택, 위험·비용 추정 문제가 광범위. 1990년대 이후 주식시장에서 효과 약화 경향.
- **Supporting (메타 근거)** | 원문 대조: ☐

### C8. Technical Trading Revisited: False Discoveries, Persistence Tests, and Transaction Costs
- Bajgrowicz, P., Scaillet, O. / 2012 / JFE 106(3):473–491 / 10.1016/j.jfineco.2012.06.001
- 핵심 발견: FDR 통제 하에서 DJIA 기술적 규칙의 OOS 수익성은 **거래비용 반영 시 소멸**, 좋은 규칙의 지속성도 낮음.
- 적용: 고정 규칙 탐색 금지, FDR 절차 채택 | **Core (방법론)** | 원문 대조: ☐

### C9. Forecasting the Equity Risk Premium: The Role of Technical Indicators
- Neely, C., Rapach, D., Tu, J., Zhou, G. / 2014 / Management Science 60(7):1772–1791 / 10.1287/mnsc.2013.1838
- 핵심 발견: 이동평균·모멘텀·거래량 기반 기술 지표가 미국 **시장 전체** 주식 프리미엄을 거시 변수와 비슷한 수준으로 OOS 예측, 경기 국면별로 상호보완.
- 적용: Market Regime Engine에 지수 수준 추세 지표 포함 근거 | **Supporting** | 원문 대조: ☐

### C10. Foundations of Technical Analysis
- Lo, A., Mamaysky, H., Wang, J. / 2000 / JF 55(4):1705–1765 / 10.1111/0022-1082.00265
- 핵심 발견: 커널 회귀로 차트 패턴 자동 인식, 일부 패턴이 조건부 수익 분포에 정보 제공(통계적 차이), 경제적 수익성은 별도.
- 적용: Price Action(패턴) 엔진을 독립 엔진으로 두지 않는 판단의 참고 — 정보는 있으나 효과 크기·비용 대비 근거 부족 | **Experimental** | 원문 대조: ☐

### C 분야 결론
- 개별 주식에서 추세의 독립 정보는 횡단면 모멘텀과 상당 부분 중첩 → Trend Engine은 **시계열 관점(절대 추세·추세 품질)** 에 집중하고, Momentum Engine은 **횡단면 관점(상대 강도)** 에 집중하여 역할을 분리.
- Whipsaw 대응: 다기간 결합(신호 평균화), 변동성 정규화, 결정 단계의 히스테리시스.
- 추세 지표 다수 추가는 정보 중복 → 다중검정 비용만 증가.

---

## D. 변동성 (Volatility)

### D1. Autoregressive Conditional Heteroscedasticity…
- Engle, R. / 1982 / Econometrica 50(4):987–1007 / 10.2307/1912773 — ARCH, 변동성 군집의 계량 모형화. **Core(이론)**

### D2. Generalized Autoregressive Conditional Heteroskedasticity
- Bollerslev, T. / 1986 / Journal of Econometrics 31(3):307–327 / 10.1016/0304-4076(86)90063-1 — GARCH(1,1). **Core**

### D3. Conditional Heteroskedasticity in Asset Returns: A New Approach
- Nelson, D. / 1991 / Econometrica 59(2):347–370 / 10.2307/2938260 — EGARCH, 비대칭(레버리지) 효과. **Supporting**

### D4. On the Relation between the Expected Value and the Volatility of the Nominal Excess Return on Stocks
- Glosten, L., Jagannathan, R., Runkle, D. / 1993 / JF 48(5):1779–1801 / 10.1111/j.1540-6261.1993.tb05128.x — GJR-GARCH. **Core 후보**(주식의 음의 수익-변동성 비대칭)

### D5. A Forecast Comparison of Volatility Models: Does Anything Beat a GARCH(1,1)?
- Hansen, P.R., Lunde, A. / 2005 / Journal of Applied Econometrics 20(7):873–889 / 10.1002/jae.800
- 핵심 발견: 330개 ARCH계 모형 비교. 환율에선 GARCH(1,1)을 이기는 모형이 없었고, **주식(IBM)에선 레버리지 효과를 포함한 모형이 우월**.
- 적용: 기본 GJR-GARCH(1,1), 기준선 GARCH(1,1) — 과도한 모형 탐색 금지 근거 | **Core** | 원문 대조: ☐

### D6. A Simple Approximate Long-Memory Model of Realized Volatility (HAR-RV)
- Corsi, F. / 2009 / Journal of Financial Econometrics 7(2):174–196 / 10.1093/jjfinec/nbp001
- 핵심 발견: 일·주·월 실현변동성의 선형 결합으로 장기 기억 근사, 단순하면서 예측력 우수.
- 적용: 일중 데이터가 없으므로 **범위 기반(Yang-Zhang) 일별 분산을 RV 대리변수**로 HAR 적용 [HYP] | **Core** | 원문 대조: ☐

### D7. Modeling and Forecasting Realized Volatility
- Andersen, T., Bollerslev, T., Diebold, F., Labys, P. / 2003 / Econometrica 71(2):579–625 / 10.1111/1468-0262.00418 — 실현변동성 이론. **Supporting**

### D8. 범위 기반 변동성 추정량
- Parkinson, M. (1980) J. Business 53(1):61–65, 10.1086/296071
- Garman, M., Klass, M. (1980) J. Business 53(1):67–78, 10.1086/296072
- Yang, D., Zhang, Q. (2000) "Drift-Independent Volatility Estimation Based on High, Low, Open, and Close Prices", J. Business 73(3):477–492, 10.1086/209650 — 시가 갭과 드리프트에 강건
- 적용: OHLC만으로 효율적 변동성 추정 → **Yang-Zhang을 기본** | **Core** | 원문 대조: ☐

### D9. The Cross-Section of Volatility and Expected Returns
- Ang, A., Hodrick, R., Xing, Y., Zhang, X. / 2006 / JF 61(1):259–299 / 10.1111/j.1540-6261.2006.00836.x
- 핵심 발견: **고유변동성(FF3 잔차 변동성)이 높은 종목의 이후 수익이 매우 낮음**("IVOL 퍼즐"); 시장 변동성 변화에 대한 민감도도 음의 가격.
- 한계: 소형·저유동성, 단기 반전과의 상호작용으로 일부 설명된다는 후속 연구 존재 | 적용: `IVOL` Feature (음의 방향 가설) | **Supporting** | 원문 대조: ☐

### D10. Volatility-Managed Portfolios
- Moreira, A., Muir, T. / 2017 / JF 72(4):1611–1644 / 10.1111/jofi.12513
- 핵심 발견: 직전 실현분산의 역수로 팩터 노출을 조정하면 다수 팩터에서 알파 발생.
- **Supporting** | 원문 대조: ☐

### D11. On the Performance of Volatility-Managed Portfolios
- Cederburg, S., O'Doherty, M., Wang, F., Yan, X. / 2020 / JFE 138(1):95–117 / 10.1016/j.jfineco.2020.04.015
- 핵심 발견: 103개 전략에 대해 **실시간 구현 가능한 OOS 기준으로는 변동성 관리 전략이 원 전략을 체계적으로 능가하지 못함**.
- 적용: 변동성 스케일링을 "수익 향상 장치"로 쓰지 않고 **신호 정규화·위험 표시** 용도로 한정 | **Supporting (반론)** | 원문 대조: ☐

### D 분야 결론
- 변동성은 수익률보다 **훨씬 예측 가능**(군집) → 서비스 내 가장 신뢰도 높은 정량 정보. 위험 표시, 신호 정규화, 라벨 임계값, 국면 판단에 사용.
- 방향 예측 변수로서 변동성은 IVOL/저변동성(음의 기대수익) 한정으로 Supporting.
- 내재변동성(옵션)은 미국 대형주만 확보 가능·고비용 → MVP 제외, 시장 수준 VIX(지수)만 국면 입력으로 검토.

---

## E. 거래량 및 시장 미시구조

### E1. Illiquidity and Stock Returns: Cross-Section and Time-Series Effects
- Amihud, Y. / 2002 / Journal of Financial Markets 5(1):31–56 / 10.1016/S1386-4181(01)00024-6
- 정의: ILLIQ = 평균(|r_d| / 거래대금_d). 핵심 발견: 비유동성 프리미엄(횡단면 양의 관계), 시장 비유동성 상승 시 동시점 수익 하락.
- 적용: **유동성 필터, 거래비용(시장충격) 모델, Feature** | **Supporting (Core급 실용성)** | 원문 대조: ☐

### E2. Price Momentum and Trading Volume
- Lee, C., Swaminathan, B. / 2000 / JF 55(5):2017–2069 / 10.1111/0022-1082.00280
- 핵심 발견: 과거 회전율이 모멘텀의 크기와 지속성에 정보 제공. 저회전 승자·고회전 패자 조합에서 모멘텀 강함; 고회전 종목은 이후 수익 낮음.
- 적용: `TURN` Feature와 모멘텀×회전율 상호작용 | **Supporting** | 원문 대조: ☐

### E3. The High-Volume Return Premium
- Gervais, S., Kaniel, R., Mingelgrin, D. / 2001 / JF 56(3):877–919 / 10.1111/0022-1082.00349
- 핵심 발견: 비정상적으로 높은 거래량을 보인 종목은 이후 1개월간 수익이 높음(가시성 가설).
- 적용: `ABVOL` Feature (단기) | **Supporting** | 원문 대조: ☐

### E4. Trading Volume and Serial Correlation in Stock Returns
- Campbell, J., Grossman, S., Wang, J. / 1993 / QJE 108(4):905–939 / 10.2307/2118454
- 핵심 발견: 거래량이 많은 날의 가격 변화는 이후 더 강하게 반전(비정보적 유동성 수요).
- **Supporting** | 원문 대조: ☐

### E5. Dynamic Volume-Return Relation of Individual Stocks
- Llorente, G., Michaely, R., Saar, G., Wang, J. / 2002 / RFS 15(4):1005–1047 / 10.1093/rfs/15.4.1005
- 핵심 발견: 거래량 동반 수익의 지속/반전은 **정보 비대칭 정도**에 따라 달라짐 — 정보거래 비중 높은 종목(소형, 고스프레드)은 지속, 헤지거래 비중 높은 종목은 반전.
- 적용: 반전 Feature와 거래량·규모의 상호작용 근거 (E4와 함께) | **Supporting** | 원문 대조: ☐

### E6. Liquidity Risk and Expected Stock Returns
- Pástor, L., Stambaugh, R. / 2003 / JPE 111(3):642–685 / 10.1086/374184 — 시장 유동성 위험 요인. 적용: 시장 유동성 국면 변수 후보. **Supporting** | 원문 대조: ☐

### E7. 스프레드 추정 (일별 데이터)
- Corwin, S., Schultz, P. (2012) "A Simple Way to Estimate Bid-Ask Spreads from Daily High and Low Prices", JF 67(2):719–760, 10.1111/j.1540-6261.2012.01729.x
- Abdi, F., Ranaldo, A. (2017) "A Simple Estimation of Bid-Ask Spreads from Daily Close, High, and Low Prices", RFS 30(12):4437–4480, 10.1093/rfs/hhx084
- 적용: 호가 데이터 없이 **거래비용 모델의 스프레드 추정** | **Core (비용 모델)** | 원문 대조: ☐

### E8. The Price Impact of Order Book Events
- Cont, R., Kukanov, A., Stoikov, S. / 2014 / Journal of Financial Econometrics 12(1):47–88 / 10.1093/jjfinec/nbt003
- 핵심 발견: 단기 가격 변화는 주문흐름 불균형(OFI)에 선형적으로 설명됨(초~분 단위).
- 한계: 레벨1 호가 틱 데이터 필요, 효과 지속시간이 매우 짧음 → 1–4주 예측과 무관 | **Rejected (MVP)** | 원문 대조: ☐

### E9. OBV, VPT, A/D, MFI, VWAP
- 원전: Granville (1963) 등 실무서. 동료심사 실증 근거 희박.
- **Experimental** — E2/E3 기반 거래량 Feature 대비 증분 정보 검정 후에만 도입. VWAP는 일중 데이터가 필요하므로 일별 대용치((H+L+C)/3)를 쓰는 경우 의미가 다름을 명시.

### E 분야 결론
- 일별 OHLCV + 거래대금으로 구현 가능한 것: Amihud, 회전율, 비정상 거래량, 범위 기반 스프레드 추정 → 채택 후보.
- 호가·체결 틱 데이터 기반(OFI, 실제 스프레드, 깊이): 비용·라이선스 대비 1–4주 예측 기여 불명 → MVP 제외.

---

## F. 팩터 모델

### F1. Common Risk Factors in the Returns on Stocks and Bonds
- Fama, E., French, K. / 1993 / JFE 33(1):3–56 / 10.1016/0304-405X(93)90023-5 — 시장·SMB·HML. **Supporting**(위험 분해)

### F2. On Persistence in Mutual Fund Performance
- Carhart, M. / 1997 / JF 52(1):57–82 / 10.1111/j.1540-6261.1997.tb03808.x — 모멘텀(UMD) 추가 4요인. **Supporting**

### F3. A Five-Factor Asset Pricing Model
- Fama, E., French, K. / 2015 / JFE 116(1):1–22 / 10.1016/j.jfineco.2014.10.010 — 수익성(RMW)·투자(CMA) 추가. **Experimental**(재무 데이터 필요)

### F4. Size, Value, and Momentum in International Stock Returns
- Fama, E., French, K. / 2012 / JFE 105(3):457–472 / 10.1016/j.jfineco.2012.05.011
- 핵심 발견: 북미·유럽·아시아태평양에서 가치·모멘텀 프리미엄, **일본에서는 모멘텀 프리미엄이 거의 없음**. 지역별 모형이 글로벌 모형보다 설명력 높음.
- 적용: 시장별 독립 모형 근거 | **Supporting** | 원문 대조: ☐

### F5. Betting Against Beta
- Frazzini, A., Pedersen, L. / 2014 / JFE 111(1):1–25 / 10.1016/j.jfineco.2013.10.005
- 핵심 발견: 레버리지 제약으로 고베타 자산이 과대평가 → 저베타 롱·고베타 숏 BAB 팩터가 미국·국제 주식 및 타 자산군에서 양의 알파.
- 적용: `BETA` Feature (음의 방향 가설) | **Supporting** | 원문 대조: ☐

### F6. Quality Minus Junk
- Asness, C., Frazzini, A., Pedersen, L. / 2019 / Review of Accounting Studies 24(1):34–112 / 10.1007/s11142-018-9470-2 — 수익성·성장·안전성·배당 기반 품질 팩터. **Experimental**(재무 데이터)

### F 분야 결론
- 가격·거래량 기반 팩터(모멘텀, 저베타, 저IVOL, 비유동성)만 MVP Feature 후보.
- FF 팩터는 **잔차화(잔차 모멘텀·잔차 반전·IVOL)와 성과 귀속**에 사용. 미국은 Kenneth French Data Library의 일별 팩터 사용 가능 [VERIFY 이용조건], 한국은 자체 구축(시가총액·장부가 필요 → 한국 HML은 재무 데이터 도입 전까지 시장·SMB·산업 요인만 사용).

---

## G. 머신러닝 기반 퀀트 모델

### G1. Empirical Asset Pricing via Machine Learning
- Gu, S., Kelly, B., Xiu, D. / 2020 / RFS 33(5):2223–2273 / 10.1093/rfs/hhaa009
- 데이터: 미국 약 3만 개 종목, 1957–2016, 94개 종목 특성 + 거시 변수 | 방법: 선형·Elastic Net·PCR/PLS·GLM·RF·GBRT·신경망 OOS 비교
- 핵심 발견: 트리·신경망이 선형 대비 OOS 예측(R²)과 포트폴리오 성과 개선. 월간 OOS R²는 1% 미만의 작은 값. **가장 중요한 예측 변수는 모멘텀·단기 반전·유동성·변동성 등 가격 기반 신호**. 비선형 상호작용이 이득의 원천.
- 한계: 대규모 횡단면·장기 데이터 전제, 소형주 비중, 거래비용 미반영 성과
- 적용: (1) 가격·거래량 기반 Feature 중심 설계의 강한 근거 (2) 비선형 모형의 잠재적 가치 → Challenger 모델 | **Core 근거 / ML은 Experimental** | 원문 대조: ☐

### G2. Deep Neural Networks, Gradient-Boosted Trees, Random Forests: Statistical Arbitrage on the S&P 500
- Krauss, C., Do, X.A., Huck, N. / 2017 / European Journal of Operational Research 259(2):689–702 / 10.1016/j.ejor.2016.10.031
- 핵심 발견: 과거 수익률 Feature만으로 일간 S&P 500 종목 예측, 앙상블이 우수. **2010년 이후 수익성 급감** 보고(비용 반영 시 더욱).
- 적용: 단기 ML 성과의 시간 감쇠 경고 | **Experimental** | 원문 대조: ☐

### G3. Deep Learning with Long Short-Term Memory Networks for Financial Market Predictions
- Fischer, T., Krauss, C. / 2018 / EJOR 270(2):654–669 / 10.1016/j.ejor.2017.11.054 — 유사한 감쇠 패턴. **Rejected (MVP)**: 해석성·재현성·데이터 대비 과적합 위험

### G4. 알고리즘 원전
- Chen, T., Guestrin, C. (2016) "XGBoost: A Scalable Tree Boosting System", KDD, 10.1145/2939672.2939785
- Ke, G. et al. (2017) "LightGBM: A Highly Efficient Gradient Boosting Decision Tree", NeurIPS 30 (https://papers.nips.cc/paper/6907)
- 용도: 구현 참조

### G5. The Virtue of Complexity in Return Prediction
- Kelly, B., Malamud, S., Zhou, K. / 2024 / JF 79(1):459–503 / 10.1111/jofi.13298
- 핵심 주장: 적절한 정규화 하에서 과모수화 모형이 시장 수익 예측에 이점. 학계 논쟁 중.
- **Experimental (모니터링)** | 원문 대조: ☐

### G 분야 결론
- ML 도입 원칙: **정규화 로지스틱(선형) 기준 모델이 Champion, LightGBM이 Challenger.** Challenger는 동일 Feature·동일 walk-forward에서 Brier/Log-loss가 Diebold-Mariano 검정으로 유의하게 낮고, 비용 차감 전략 성과(DSR)도 개선될 때만 채택.
- SVM(확률 출력 부재·스케일 민감), 딥러닝(데이터 대비 과적합·재현성) → MVP 제외.
- HMM은 예측기보다 국면 추정기로 사용(H 분야).

---

## H. 시장 국면 (Regime)

### H1. A New Approach to the Economic Analysis of Nonstationary Time Series and the Business Cycle
- Hamilton, J. / 1989 / Econometrica 57(2):357–384 / 10.2307/1912559 — Markov switching 모형. **Supporting(이론)**

### H2. International Asset Allocation with Regime Shifts
- Ang, A., Bekaert, G. / 2002 / RFS 15(4):1137–1187 / 10.1093/rfs/15.4.1137
- 핵심 발견: 주식 수익률에 고변동성·고상관 국면(약세)과 저변동성 국면 존재. 국면 인식 자산배분이 가치 있음.
- **Supporting** | 원문 대조: ☐

### H3. Asset Allocation under Multivariate Regime Switching
- Guidolin, M., Timmermann, A. / 2007 / Journal of Economic Dynamics and Control 31(11):3503–3544 / 10.1016/j.jedc.2006.12.004
- 핵심 발견: 미국 주식·채권 수익에 4개 국면(crash, slow growth, bull, recovery) 식별.
- 적용: 국면 수 선택은 BIC + 해석 가능성, MVP는 2–3 국면 | **Supporting** | 원문 대조: ☐

### H4. A Simple Framework for Analysing Bull and Bear Markets
- Pagan, A., Sossounov, K. / 2003 / Journal of Applied Econometrics 18(1):23–46 / 10.1002/jae.664 — 규칙 기반 강세·약세 사후 판별(사후 라벨링 용). **Supporting(평가용)**

### H5. Duration Dependence in Stock Prices
- Lunde, A., Timmermann, A. / 2004 / Journal of Business & Economic Statistics 22(3):253–273 / 10.1198/073500104000000136 — 강세·약세 정의(누적 ±20% 등)와 지속기간 의존성. **Supporting(평가용)**

### H6. Bayesian Online Changepoint Detection
- Adams, R., MacKay, D. / 2007 / arXiv:0710.3742 (https://arxiv.org/abs/0710.3742)
- 적용: 급격한 국면 전환 감지 보조 | **Experimental** | 원문 대조: ☐

### H 분야 결론
- 국면 엔진은 **필터링 확률 P(S_t | 데이터 ≤ t)만** 사용. 스무딩 확률(미래 데이터 사용)은 look-ahead → 금지.
- HMM 단독은 국면 전환 감지 지연과 파라미터 불안정성이 있음 → (1) 2-상태 Gaussian HMM(시장 수익·변동성) + (2) 규칙 기반 상태(200일 MA, 36개월 수익, 변동성 백분위)를 병렬 산출하고, 둘 다 Feature로 제공한 뒤 앙상블이 사용 여부를 학습.
- 사후 국면 라벨(H4, H5)은 **평가용(국면별 성과 분해)** 으로만 사용, 모델 입력 금지.

---

## I. 포트폴리오·위험 관리

| ID | 연구 | 서지 | 핵심 | 판정·적용 |
|---|---|---|---|---|
| I1 | A New Interpretation of Information Rate | Kelly, J.L. (1956) Bell System Technical Journal 35(4):917–926, 10.1002/j.1538-7305.1956.tb03809.x | 로그 성장 최대화 베팅 비율 | **Rejected(사용자 노출)**: 기대수익 추정오차에 극도로 민감. 내부 분석에서 분수 Kelly 참고만 |
| I2 | Coherent Measures of Risk | Artzner, Delbaen, Eber, Heath (1999) Mathematical Finance 9(3):203–228, 10.1111/1467-9965.00068 | VaR의 비가법성 문제, 일관 위험측도 | **Supporting**: CVaR 채택 근거 |
| I3 | Optimization of Conditional Value-at-Risk | Rockafellar, Uryasev (2000) Journal of Risk 2(3):21–41, 10.21314/JOR.2000.038 | CVaR 계산·최적화 | **Supporting**: 하방 위험 지표 |
| I4 | When Do Stop-Loss Rules Stop Losses? | Kaminski, Lo (2014) Journal of Financial Markets 18:234–254, 10.1016/j.finmar.2013.07.001 | 랜덤워크에선 손절이 기대수익 감소, **모멘텀·국면 전환이 있는 수익과정에선 가치를 더할 수 있음** | **Supporting**: SELL 위험 게이트를 고정 % 손절이 아닌 추세+변동성 조건으로 설계 |
| I5 | The Properties of Equally Weighted Risk Contribution Portfolios | Maillard, Roncalli, Teïletche (2010) JPM 36(4):60–70, 10.3905/jpm.2010.36.4.060 | 리스크 패리티 | **Supporting(Phase 3 포트폴리오 기능)** |
| I6 | Optimal Versus Naive Diversification | DeMiguel, Garlappi, Uppal (2009) RFS 22(5):1915–1953, 10.1093/rfs/hhm075 | 추정오차로 인해 1/N이 평균-분산 최적화를 OOS에서 이기기 어려움 | **Core(방법론)**: 앙상블 가중 추정 시 동일가중 기준선 필수 |
| I7 | The Impact of Volatility Targeting | Harvey, Hoyle, Korgaonkar, Rattray, Sargaison, Van Hemert (2018) JPM 45(1):14–33, 10.3905/jpm.2018.45.1.014 | 변동성 타깃팅은 위험자산에서 꼬리위험 감소, Sharpe 개선은 자산별 상이 | **Supporting**: 위험 표시 |

### I 분야 결론 — 종목 신호와 포트폴리오 위험의 연결
- BUY OR SELL MVP는 **종목 단위 판단**을 제공하지만, 백테스트는 "신호를 따르는 표준 포트폴리오"(동일가중 / 변동성 역가중)로 평가한다 (`BACKTEST_SPEC.md §6`).
- 사용자에게 포지션 크기를 지시하지 않는다(자문 규제 리스크 + Kelly 추정오차). 대신 예상 변동성·CVaR·유동성을 정보로 제공.

---

## J. 백테스트 통계 및 과최적화 방지 (방법론 Core)

| ID | 연구 | 서지 | 채택 내용 |
|---|---|---|---|
| J1 | A Reality Check for Data Snooping | White, H. (2000) Econometrica 68(5):1097–1126, 10.1111/1468-0262.00152 | 다중 전략 중 최고 성과의 유의성 검정 |
| J2 | A Test for Superior Predictive Ability | Hansen, P.R. (2005) JBES 23(4):365–380, 10.1198/073500105000000063 | SPA 검정 (RC 개선판) — **채택** |
| J3 | The Model Confidence Set | Hansen, Lunde, Nason (2011) Econometrica 79(2):453–497, 10.3982/ECTA5771 | 모델 후보 집합 축소 — **채택** |
| J4 | The Deflated Sharpe Ratio | Bailey, López de Prado (2014) JPM 40(5):94–107, 10.3905/jpm.2014.40.5.094 | 시행 횟수·비정규성 보정 Sharpe — **채택** |
| J5 | The Probability of Backtest Overfitting | Bailey, Borwein, López de Prado, Zhu (2017) Journal of Computational Finance 20(4):39–69, 10.21314/JCF.2016.322 | CSCV로 PBO 추정 — **채택** |
| J6 | Advances in Financial Machine Learning | López de Prado, M. (2018) Wiley, ISBN 978-1-119-48208-6 | Purged K-Fold, Embargo, 메타 라벨링, Triple-barrier — **채택(일부)** |
| J7 | The Stationary Bootstrap | Politis, Romano (1994) JASA 89(428):1303–1313, 10.1080/01621459.1994.10476870 | 시계열 신뢰구간 — **채택** |
| J8 | Robust Performance Hypothesis Testing with the Sharpe Ratio | Ledoit, Wolf (2008) Journal of Empirical Finance 15(5):850–859, 10.1016/j.jempfin.2008.03.002 | 두 전략 Sharpe 차이 검정 — **채택** |
| J9 | The Statistics of Sharpe Ratios | Lo, A. (2002) Financial Analysts Journal 58(4):36–52, 10.2469/faj.v58.n4.2453 | 자기상관 보정 Sharpe 표준오차 — **채택** |
| J10 | Comparing Predictive Accuracy | Diebold, Mariano (1995) JBES 13(3):253–263, 10.1080/07350015.1995.10524599 | 예측 손실 차이 검정 — **채택** |
| J11 | Controlling the False Discovery Rate | Benjamini, Hochberg (1995) JRSS-B 57(1):289–300, 10.1111/j.2517-6161.1995.tb02031.x | Feature 선별 다중검정 — **채택** |

## K. 예측 결합·확률 보정 (방법론 Core)

| ID | 연구 | 서지 | 채택 내용 |
|---|---|---|---|
| K1 | Forecast Combinations | Timmermann, A. (2006) Handbook of Economic Forecasting Vol.1 Ch.4:135–196, 10.1016/S1574-0706(05)01004-9 | 결합의 이론·실증 정리 |
| K2 | The Forecast Combination Puzzle: A Simple Theoretical Explanation | Claeskens, Magnus, Vasnev, Wang (2016) IJF 32(3):754–762, 10.1016/j.ijforecast.2015.12.005 | 추정 가중이 동일가중보다 OOS에서 못한 이유(추정 분산) → **동일가중 방향 수축** |
| K3 | A Simple Explanation of the Forecast Combination Puzzle | Smith, Wallis (2009) Oxford Bulletin of Economics and Statistics 71(3):331–355, 10.1111/j.1468-0084.2008.00541.x | 동일 |
| K4 | Bayesian Model Averaging: A Tutorial | Hoeting, Madigan, Raftery, Volinsky (1999) Statistical Science 14(4):382–417, 10.1214/ss/1009212519 | BMA 비교 후보 |
| K5 | Online Prediction Under Model Uncertainty via Dynamic Model Averaging | Raftery, Kárný, Ettler (2010) Technometrics 52(1):52–66, 10.1198/TECH.2009.08104 | DMA 비교 후보 (망각계수) |
| K6 | Stacked Generalization | Wolpert, D. (1992) Neural Networks 5(2):241–259, 10.1016/S0893-6080(05)80023-1 | 스태킹 |
| K7 | Predicting Good Probabilities with Supervised Learning | Niculescu-Mizil, Caruana (2005) ICML, 10.1145/1102351.1102430 | Platt vs Isotonic 비교 — 소표본엔 Platt, 대표본엔 Isotonic |
| K8 | Transforming Classifier Scores into Accurate Multiclass Probability Estimates | Zadrozny, Elkan (2002) KDD, 10.1145/775047.775151 | 다중클래스 보정 |
| K9 | Verification of Forecasts Expressed in Terms of Probability | Brier, G. (1950) Monthly Weather Review 78(1):1–3, 10.1175/1520-0493(1950)078<0001:VOFEBP>2.0.CO;2 | Brier score |
| K10 | Strictly Proper Scoring Rules, Prediction, and Estimation | Gneiting, Raftery (2007) JASA 102(477):359–378, 10.1198/016214506000001437 | 적정 점수 규칙(로그손실, Brier, RPS) |
| K11 | A Comprehensive Look at the Empirical Performance of Equity Premium Prediction | Welch, Goyal (2008) RFS 21(4):1455–1508, 10.1093/rfs/hhm014 | 시장 수익 예측 변수 대부분 OOS 실패 → **역사적 평균(기저율) 기준선** 필수 |
| K12 | Predicting Excess Stock Returns Out of Sample | Campbell, Thompson (2008) RFS 21(4):1509–1531, 10.1093/rfs/hhm055 | 작은 OOS R²도 경제적 가치 가능; 부호·크기 제약이 OOS 개선 |

## L. 이상현상의 재현성과 감쇠 (방법론 Core)

| ID | 연구 | 서지 | 핵심 | 설계 반영 |
|---|---|---|---|---|
| L1 | Does Academic Research Destroy Stock Return Predictability? | McLean, Pontiff (2016) JF 71(1):5–32, 10.1111/jofi.12365 | 97개 예측 변수의 수익이 표본 외 약 26%, **발표 후 약 58% 감소** | 논문 보고 효과 크기를 기대치로 쓰지 않음; 최근 기간 가중 평가 |
| L2 | …and the Cross-Section of Expected Returns | Harvey, Liu, Zhu (2016) RFS 29(1):5–68, 10.1093/rfs/hhv059 | 수백 개 팩터 탐색 → 새 팩터는 **t > 3.0** 기준 권고 | 신규 Feature 채택 기준 t>3 (HAC) |
| L3 | Replicating Anomalies | Hou, Xue, Zhang (2020) RFS 33(5):2019–2133, 10.1093/rfs/hhy131 | 452개 이상현상 중 다수가 소형주 영향 통제(NYSE 기준점·가치가중) 시 유의성 상실; **모멘텀·거래 마찰 범주에서 실패 비율 높음** | 소형·비유동 종목 제외 유니버스, 가치가중 결과 병행 보고 |
| L4 | Is There a Replication Crisis in Finance? | Jensen, Kelly, Pedersen (2023) JF 78(5):2465–2518, 10.1111/jofi.13249 | 베이지안 관점에서 대다수 팩터가 재현되며 국제적으로도 유사 | L3과 균형: 정의를 투명하게 하면 재현 가능 → **정의 고정·사전 등록** |

> L3과 L4의 상반된 결론은 "효과의 존재"보다 "정의·유니버스·가중 방식에 대한 민감도"가 핵심임을 보여준다. 따라서 BUY OR SELL은 Feature 정의를 사전 등록하고, 동일가중/가치가중·대형주/중형주 하위 표본에서 모두 부호가 일관될 것을 채택 조건으로 둔다 (`BACKTEST_SPEC.md §9`).

---

## 2. 전략별 실증 근거 비교 (BUY OR SELL 관점)

점수: 1(낮음)–5(높음). "재현"=일별 OHLCV로 재현 가능성, "비용"=비용 반영 후 생존 가능성, "최근"=2010년 이후 유효 가능성(문헌 기반 추정), "기여"=서비스 기여도. **이 점수는 문헌 기반 사전 평가이며 자사 실험 결과가 아니다.**

| 전략 | 학술 신뢰 | 표본 | 재현 | 비용 | 최근 | 한국 적용 | 난이도(5=쉬움) | 기여 | 분류 |
|---|---|---|---|---|---|---|---|---|---|
| 12-1 횡단면 모멘텀 | 5 | 5 | 5 | 4 | 3 | 2 | 5 | 5 | Core(중기) |
| 잔차 모멘텀 | 4 | 4 | 4 | 4 | 3 | 3 | 3 | 4 | Core 후보 |
| 52주 고가 | 4 | 4 | 5 | 4 | 3 | 3 | 5 | 3 | Core 후보 |
| 단기 반전(1M) | 5 | 5 | 5 | 2 | 3 | 3 | 5 | 4 | Core(단기) |
| 잔차 단기 반전 | 4 | 4 | 4 | 3 | 3 | 3 | 3 | 4 | Core 후보 |
| TSMOM/MA 추세 | 5 | 5 | 5 | 4 | 3 | 3 | 5 | 4 | Core(중기)/국면 |
| GJR-GARCH/HAR 변동성 | 5 | 5 | 5 | n/a | 5 | 5 | 3 | 5 | Core(위험) |
| IVOL/저베타 | 4 | 4 | 4 | 3 | 3 | 3 | 3 | 3 | Supporting |
| Amihud/회전율 | 4 | 4 | 5 | n/a | 4 | 4 | 5 | 4 | Supporting |
| 비정상 거래량 | 3 | 3 | 5 | 3 | 2 | 3 | 5 | 2 | Supporting |
| HMM 국면 | 4 | 3 | 4 | n/a | 3 | 3 | 3 | 4 | Supporting |
| RSI/볼린저 고정 규칙 | 1 | 2 | 5 | 2 | 1 | 2 | 5 | 1 | Experimental |
| 페어/공적분 | 4 | 4 | 3 | 2 | 2 | 2 | 2 | 1 | Experimental |
| OFI/호가 | 4 | 3 | 1 | 1 | 4 | 1 | 1 | 1 | Rejected |
| GBDT(LightGBM) | 4 | 4 | 4 | ? | 3 | ? | 3 | 3 | Experimental(Challenger) |
| LSTM/딥러닝 | 3 | 3 | 2 | 1 | 2 | ? | 1 | 1 | Rejected(MVP) |

---

## 3. 조사 결론: 모델 설계로의 연결

1. **단기(1–4주)** 의 가장 강한 근거는 **단기 반전**(유동성 공급 보상) — 조건: 유동성·비용·변동성 국면·뉴스성 움직임 제거(잔차화). 단기 모멘텀은 근거 약함.
2. **중기(1–6개월)** 의 가장 강한 근거는 **횡단면 모멘텀(12-1, 잔차, 52주 고가)과 시계열 추세** — 조건: 크래시 국면 축소, 한국 시장 독립 검증.
3. **변동성**은 방향보다 *크기*가 예측 가능 → 위험 표시·정규화·라벨 임계값·국면 판단에 핵심. 방향 Feature로는 IVOL/저베타만.
4. **거래량**은 독립 신호보다 **조건 변수**(모멘텀·반전 강도 조절)와 비용 추정에 가치.
5. **ML**은 Feature를 대체하지 않고, 검증된 Feature 간 비선형 상호작용을 포착하는 Challenger로만.
6. **방법론(J, K, L)** 이 모델 성공 여부를 좌우: 다중검정 보정, 동일가중 기준선, 확률 보정, 발표 후 감쇠 감안.
7. "Price Action Engine"은 독립 신호 엔진으로 두지 않는다: 차트 패턴 근거(C10)가 약하고, 가격 정보는 이미 모멘텀·반전·추세·변동성 엔진이 다른 시간척도로 사용 → **공통 Feature 계층(가격 변환)** 으로 재정의 (`QUANT_MODEL_SPEC.md §3`).

## 4. Phase 1 연구 과제 (착수 전 미완료 항목)

| ID | 과제 | 완료 조건 |
|---|---|---|
| R1 | 본 문서 전 항목 원문 대조(서지·핵심 수치) | 모든 `원문 대조: ☐` → `✔` 또는 수정 |
| R2 | 한국 시장 모멘텀·반전·저변동성 국내외 실증 연구 원문 수집 (2010년 이후 우선) | 최소 8편 카드화 |
| R3 | 한국 시장 가격제한폭(±30%, 2015.6 확대)·공매도 금지 기간(2020.3–2021.5 전면금지, 2023.11–2025.3 전면금지 [VERIFY])이 반전·모멘텀에 미친 영향 연구 | 백테스트 구조 변화 더미 설계 반영 |
| R4 | 각 Core Feature 자사 데이터 재현 (`BACKTEST_SPEC.md §9`) | Replication Gate 보고서 |
