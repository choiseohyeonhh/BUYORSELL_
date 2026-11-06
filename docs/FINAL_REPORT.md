# FINAL_REPORT — BUY OR SELL 기획·설계 최종 보고

작성일 2026-11-06 · 상태: 설계 완료, **구현·실증 검증 전**

> 이 보고서의 모든 모델 성능은 아직 측정되지 않았다. 아래 내용은 연구 근거에 기반한 설계와, 그 설계를 검증하기 위한 절차다. 성과 수치를 예측해서 기재하지 않았다.

---

## 1. 서비스의 최종 정체성

BUY OR SELL은 **"이 종목을 지금 사거나 팔 통계적 근거가 있는가, 그리고 그 근거를 얼마나 믿을 수 있는가"** 를 재현 가능한 수학 모델로 계산하는 Quantitative Investment Intelligence Platform이다.

- 판단 대상: 개별 종목의 **벤치마크 대비** 향후 약 2주(10거래일)·3개월(63거래일) 초과수익.
- 판단 주체: 버전 관리된 결정론적 통계 모델. LLM은 판단에 관여하지 않는다.
- 차별점: 보정된 확률, 확률과 분리된 신뢰도, 사유가 있는 HOLD, 해시 체인 원장 기반 실시간 성과 공개, 근거가 없으면 출시하지 않는 Gate 체계.

## 2. 조사한 주요 퀀트 연구

`QUANT_RESEARCH.md`에 고유 DOI 기준 100여 편을 12개 분야로 정리했다. 핵심:

| 분야 | 대표 연구 |
|---|---|
| 모멘텀 | Jegadeesh-Titman (1993, 2001), Asness-Moskowitz-Pedersen (2013), Moskowitz-Ooi-Pedersen (2012), Daniel-Moskowitz (2016), Barroso-Santa-Clara (2015), George-Hwang (2004), Blitz-Huij-Martens (2011), Chui-Titman-Wei (2010), Cooper-Gutierrez-Hameed (2004) |
| 단기 반전 | Jegadeesh (1990), Lehmann (1990), Da-Liu-Schaumburg (2014), Nagel (2012), Avramov-Chordia-Goyal (2006) |
| 추세 | Hurst-Ooi-Pedersen (2017), Levine-Pedersen (2016), Han-Yang-Zhou (2013), Sullivan-Timmermann-White (1999), Bajgrowicz-Scaillet (2012) |
| 변동성 | Bollerslev (1986), Glosten-Jagannathan-Runkle (1993), Hansen-Lunde (2005), Corsi (2009), Yang-Zhang (2000), Ang-Hodrick-Xing-Zhang (2006), Moreira-Muir (2017) vs Cederburg et al. (2020) |
| 거래량·유동성 | Amihud (2002), Lee-Swaminathan (2000), Gervais-Kaniel-Mingelgrin (2001), Campbell-Grossman-Wang (1993), Abdi-Ranaldo (2017) |
| 팩터 | Fama-French (1993, 2015), Carhart (1997), Frazzini-Pedersen (2014) |
| 머신러닝 | Gu-Kelly-Xiu (2020), Krauss-Do-Huck (2017) |
| 국면 | Hamilton (1989), Ang-Bekaert (2002), Guidolin-Timmermann (2007) |
| 방법론 | Bailey-López de Prado (DSR 2014, PBO 2017), White (2000), Hansen (SPA 2005; MCS 2011), McLean-Pontiff (2016), Harvey-Liu-Zhu (2016), Hou-Xue-Zhang (2020), Jensen-Kelly-Pedersen (2023), Claeskens et al. (2016) |

정직성 고지: 서지 정보는 표준 서지에 근거하며, 이번 세션에서 PDF 원문 전체를 재대조하지는 않았다. Phase 1(R1)에서 원문 대조를 완료 조건으로 두었다. 데이터 공급자 정보 일부는 이번 세션에 웹으로 확인했고(출처 표기), 나머지는 [VERIFY]로 표시했다.

## 3. 가장 유효하다고 판단한 전략

| 순위 | 전략 | 기간 | 근거 강도 | 조건 |
|---|---|---|---|---|
| 1 | 횡단면 모멘텀 (12-1, 잔차 모멘텀, 52주 고가) | 중기 | 다수 시장·장기 재현 | 크래시 국면 축소, **한국은 근거 약함 → 독립 추정** |
| 2 | 단기 반전 (특히 산업·시장 조정 잔차 반전) | 단기 | 장기 재현, 유동성 공급 보상 | 비용·유동성 필터, 고변동성 시 강화 |
| 3 | 시계열 추세 (다기간 변동성 정규화) | 중기·국면 | 장기 다자산 증거 | 개별 주식에선 모멘텀과 중복 관리 |
| 4 | 변동성 예측 (GJR-GARCH + HAR) | 전 기간 | 가장 예측 가능한 양 | 방향이 아닌 위험·정규화·임계값 용도 |
| 5 | 저위험 (IVOL, 베타) | 중기 | 지지적 | 보조 Feature |
| 6 | 거래량·유동성 (비정상 거래량, Amihud, 회전율) | 단기·조건 | 지지적 | 조건 변수·비용 |

제외: RSI/볼린저 고정 임계값 규칙, MACD·Donchian 별도 신호(정보 중복), 페어 트레이딩(서비스 구조 불일치), 주문흐름 불균형(틱 데이터 필요), 딥러닝(MVP), Kelly 포지션 지시.

## 4. 최종 추천 모델 아키텍처

```
L0 가격·거래량 변환 → L1 엔진 5개(모멘텀·반전·추세·저위험·거래량) + 변동성 위험 예측 + 국면 엔진(필터링 확률)
→ L2 정규화 순서형 로지스틱 (동일가중 방향으로 수축, 부호 제약, 이론 기반 상호작용 소수)
→ Temperature 보정 → 3분류 확률 (상승/횡보/하락)
→ L3 결정 엔진 (기대 초과수익·비용·신뢰도·위험 게이트·히스테리시스) → BUY/HOLD(사유)/SELL
Challenger: LightGBM (단조 제약) — Champion 대비 유의한 OOS 개선 시에만 확률 평균으로 합류
```
상세: `QUANT_MODEL_SPEC.md`. 시장(US/KR)×기간(Short/Mid) 4개 모델을 완전히 독립 추정.

## 5. BUY / HOLD / SELL 결정 방식

1. 라벨: t+1 시가 기준 h일 벤치마크 초과 로그수익 X를 변동성·비용 기반 임계값 τ = max(왕복비용, 0.5·σ̂√h)로 UP/FLAT/DOWN 분류.
2. 보정 확률에서 방향 우위 E = p_up − p_down, 기대 초과수익 μ̂ = σ̂√h·Σ p_k·m̄_k, 순기대 NER = μ̂ − 왕복비용.
3. **BUY**: E ≥ δ_B, NER > 0, 신뢰도 A/B, 위기 국면 아님(또는 더 강한 우위), 데이터 정상, 지속성 확인.
4. **SELL**: E ≤ −δ_S, μ̂ < 0, 신뢰도 A/B, 지속성 — 또는 **위험 SELL**(극단 변동성 + 강한 하락 추세 + 시장 고변동/유동성 악화, Kaminski-Lo 근거).
5. **HOLD**는 중간값이 아니라 사유가 있는 상태: 우위 없음 / 비용 미달 / 불확실(OOD·엔진 불일치·모델 점검) / 위기 국면 / 확인 중 / 데이터 부족.
6. 히스테리시스(이탈 임계값 = 진입의 절반)와 최소 유지기간(5일/21일)으로 신호 깜빡임 방지.
7. δ는 임의 지정이 아니라 검증 폴드에서 비용 차감 Sharpe 기준, 평탄 영역 선호 규칙으로 선택. 신뢰도 등급은 메타 라벨 모형(판단이 비용을 넘어 맞을 확률)의 분위로 정의.

## 6. 단기·중기 모델 설계

| | Short | Mid |
|---|---|---|
| 기간 | 10거래일 | 63거래일 |
| 주 근거 | 단기(잔차) 반전, 비정상 거래량 | 모멘텀(12-1·잔차·52주), 추세, 저위험 |
| 조건 변수 | 시장 변동성(반전 강화), 거래량(반전 조건) | 크래시 국면·36개월 시장 하락(모멘텀 축소), 회전율 |
| 학습 표본 | 5거래일마다 | 21거래일마다 |
| 상태 평가 | 매일 (2일 연속 확인) | 주 1회 (위험 SELL은 매일) |
| 최소 유지 | 5거래일 | 21거래일 |

## 7. 백테스트 및 검증 방법

- PIT 유니버스(상장폐지 포함), t+1 시가 체결(+스프레드·충격), 시점별 한국 거래세 표, 상·하한가 체결 불가 처리, 상장폐지 수익 규칙.
- Walk-forward 확장창 + 내부 Purged K-Fold/Embargo, 잠금 홀드아웃(최근 24개월, 1회), 63거래일 그림자 운영.
- 기준선(기저율·국면·단일요인·동일가중) 대비 DM 검정, SPA, MCS, DSR, PBO(CSCV), 블록 부트스트랩 CI, 국면별·스트레스 분석, 실험 레지스트리로 시행 수 집계.
- 출시 기준 C1–C10 사전 등록 (`BACKTEST_SPEC.md §12`), 운영 중 드리프트·보정·성능 감시와 확률 표시 정책 자동 전환 (`MODEL_VALIDATION_SPEC.md`).

## 8. 한국·미국 데이터 확보 방안

- **US**: 연구 — Sharadar(Nasdaq Data Link; 상장폐지 포함, 1998~, 공식 확인 필요) + Kenneth French 팩터 + FRED. 운영 표시 — Tiingo(재배포는 별도 라이선스 필요, 공식 문서 확인) 또는 Massive(구 Polygon) 사업용 플랜 견적 비교.
- **KR**: 연구 — 상장폐지 포함 수정주가 상용 DB 확보가 핵심 과제, 프로토타입은 KRX Open API(비상업·원주가만·일 1만 건, 2차 자료) + DART 기업행위로 자체 수정계수. 운영 — 코스콤 등 공식 시세 이용계약 전까지 KR은 비공개 그림자 운영. 무위험 — ECOS.
- 무료 비공식 소스(Yahoo, 스크래핑)는 공개 서비스에 사용하지 않음.

## 9. 추천 기술 스택

Python 3.12(Polars, DuckDB, statsmodels, arch, scikit-learn, LightGBM, hmmlearn, Numba, uv) · Prefect 3 · PostgreSQL 16(불변 원장, 해시 체인) · S3 호환 객체 저장소 + Parquet 스냅샷 · FastAPI(읽기 전용) · Next.js + TypeScript + Tailwind · Lightweight Charts · CDN 캐시 · Sentry · GitHub Actions(누출·재현성 테스트 포함). 배치 중심 모놀리스, 마이크로서비스 없음.

## 10. MVP 기능

종목 검색(초성 포함) · 단기/중기 판단 카드 · 보정 확률(표시 정책 연동) · 신뢰도 등급 · 엔진별 판단 근거 · 과거 유사 신호 성과 · 위험 요인(예상 변동폭, CVaR, 유동성) · 가격/거래량 차트와 신호 마커 · 시장 국면 · 모델 성과(실시간 원장 vs 가상 성과 분리) · 방법론/모델 카드 · 신호 원장 조회 · 반응형(모바일·데스크톱). 계정·알림·스크리너·LLM 설명은 후순위.

## 11. 주요 리스크와 미해결 과제

| 리스크/과제 | 상태 |
|---|---|
| 모델이 OOS에서 기준선을 이기지 못할 가능성 (특히 비용 차감 후, 발표 후 감쇠) | Gate로 출시 차단 — 이 결과도 수용 |
| 한국 시장 모멘텀 약세 (국제 비교 연구) | 독립 추정, Mid-KR 출시 불확실 |
| 한국 상장폐지 포함 데이터와 상업 라이선스 | 미확보 — 비용·일정 최대 불확실성 |
| 데이터 공급자 약관·가격 | [VERIFY] 9개 항목 체크리스트 |
| 규제(유사투자자문업 신고 필요성, 미국 publisher's exclusion) | 법률 의견서 필요 |
| 학술 문헌 원문 대조, 국내 연구 수집 | Phase 1 R1–R3 |
| 산업 분류·지수 구성 PIT 이력 | 대체 규칙 있음, 편향 명시 |
| 실적 발표 등 이벤트 위험 | MVP는 미반영(한계 공시) |
| 확률 보정이 표본 부족으로 불안정할 가능성(특히 Mid) | 확률 표시 정책으로 수치 숨김 |

## 12. 생성한 문서

| 경로 | 내용 |
|---|---|
| `README.md` | 개요, 문서 목록, 표기 규약 |
| `docs/PRODUCT_PLAN.md` | 서비스 철학, 사용자, 시나리오, 기능, MVP, 차별점 |
| `docs/QUANT_RESEARCH.md` | 연구 카드, 분야별 결론, 판정표 |
| `docs/QUANT_MODEL_SPEC.md` | 엔진 수식, 앙상블, 보정, 결정 엔진, 출력 명세 |
| `docs/BACKTEST_SPEC.md` | 데이터 분리, 체결·비용, 편향 방지, 지표, 출시 기준 |
| `docs/MODEL_VALIDATION_SPEC.md` | 보정, 유의성, 드리프트, 재학습, 버전 관리 |
| `docs/DATA_SOURCE_SPEC.md` | 공급자 비교, 라이선스, 품질 관리 |
| `docs/SYSTEM_ARCHITECTURE.md` | 스택, 파이프라인, DB, API, 배포 |
| `docs/UI_UX_SPEC.md` | 화면, 흐름, 디자인 시스템 |
| `docs/DEVELOPMENT_ROADMAP.md` | Phase·Gate·의존성 |
| `docs/LEGAL_REGULATORY_REVIEW.md` | 한·미 규제 쟁점, 검토 체크리스트 |
| `docs/FINAL_REPORT.md` | 본 문서 |

## 13. 개발 착수를 위한 다음 단계

1. **데이터 확인·계약 (Phase 0.1)**: Sharadar 공식 사양·가격, 한국 상장폐지 포함 DB 견적, KRX Open API 약관 확인.
2. **법률 상담 예약**: 유사투자자문업 해당 여부(무료·비개인화 MVP 기준).
3. **저장소 골격**: monorepo, uv, CI(누출·재현성 테스트 틀 먼저).
4. **US 데이터 파이프라인 + PIT 유니버스** → G0.
5. **Replication Gate RG1–RG8** → G1. 여기서 모멘텀이 재현되지 않으면 모델 개발이 아닌 데이터 감사부터.
6. 병행: 디자인 시스템과 목업 기반 UI 프로토타입, 페르소나 사용성 테스트 준비.

---

## 핵심 질문에 대한 답

> **"BUY OR SELL의 핵심 퀀트 모델은 어떤 연구와 통계적 근거를 기반으로 어떤 방식으로 투자 판단을 내리는가?"**

**무엇을 예측하는가.** 종목 i의 t+1 시가부터 h일(10 또는 63거래일) 동안의 벤치마크 대비 초과 로그수익 X를, 그 종목의 예측 변동성과 거래비용으로 정한 임계값 τ = max(c^{rt}, 0.5·σ̂^e√h)에 따라 상승·횡보·하락의 3개 순서형 클래스로 예측한다. 절대 수익률이나 목표가가 아니라, 비용을 넘는 의미 있는 상대 움직임의 확률을 예측한다.

**어떤 근거를 쓰는가.** 입력은 수십 년, 여러 시장에서 반복 확인된 가격·거래량 기반 효과만 쓴다.
- 중기: 횡단면 모멘텀(Jegadeesh-Titman 1993; Asness-Moskowitz-Pedersen 2013), 잔차 모멘텀(Blitz-Huij-Martens 2011), 52주 고가(George-Hwang 2004), 다기간 시계열 추세(Moskowitz-Ooi-Pedersen 2012; Hurst-Ooi-Pedersen 2017), 저위험 이상현상(Ang et al. 2006; Frazzini-Pedersen 2014).
- 단기: 단기 반전(Jegadeesh 1990; Lehmann 1990)과 그 잔차 버전(Da-Liu-Schaumburg 2014), 비정상 거래량(Gervais et al. 2001). 반전은 유동성 공급의 대가이므로 시장 변동성(Nagel 2012), 거래량(Campbell-Grossman-Wang 1993)과의 조건으로 강도를 조절한다.
- 위험 쪽에서는 변동성 군집(Engle 1982; Bollerslev 1986)을 이용해 GJR-GARCH(Glosten et al. 1993)와 HAR(Corsi 2009)을 결합해 변동성을 예측한다. 이 예측은 모든 신호의 정규화, 라벨 임계값, 위험 표시에 쓴다.
- 모멘텀이 크게 무너지는 국면, 즉 약세장 뒤 고변동성 반등기(Daniel-Moskowitz 2016)와 36개월 시장 하락 이후(Cooper et al. 2004)는 HMM 국면 엔진의 **필터링** 확률(Hamilton 1989)로 실시간 추정하고, 모멘텀 효과를 줄이는 상호작용 항으로 반영한다.

**어떻게 결합하는가.** 각 엔진은 문헌이 정한 부호를 가진 Feature의 횡단면 순위 정규 점수를 동일가중 평균한 엔진 점수 S_e를 낸다. 엔진 점수는 순서형 로지스틱 회귀 P(Y ≤ k) = σ(θ_k − βᵀx)로 결합한다. 이때 β는 동일가중 사전값 쪽으로 수축된 릿지 벌점으로 추정한다(추정 가중이 동일가중을 OOS에서 이기기 어렵다는 예측 결합 퍼즐, Claeskens et al. 2016; DeMiguel et al. 2009). 문헌과 반대 부호는 0으로 제약한다. 결과 확률은 분리된 보정 구간에서 Temperature scaling으로 보정한다. 비선형 모형(LightGBM)은 OOS 개선이 통계적으로 유의할 때만 합류한다(Gu-Kelly-Xiu 2020이 비선형성의 잠재 가치를 보였지만 근거로 충분하지는 않다).

**어떻게 판단하는가.** 보정 확률에서 방향 우위 E = p_up − p_down과 비용 차감 기대 초과수익을 계산한다. 우위가 사전에 검증된 임계값 δ를 넘고, 기대수익이 비용을 넘고, 메타 라벨 모형 기준 신뢰도가 상위 등급이며, 위기 국면이 아니고, 신호가 지속될 때만 BUY(대칭 조건이면 SELL)를 낸다. 극단적 변동성과 강한 하락 추세가 겹치면 위험 기반 SELL을 낸다(Kaminski-Lo 2014). 그 외에는 사유가 있는 HOLD를 낸다. 히스테리시스와 최소 유지기간으로 신호가 자주 바뀌는 것을 막는다.

**왜 믿을 수 있는지(또는 없는지) 어떻게 아는가.** 문헌상 효과는 발표 후 줄어들고(McLean-Pontiff 2016), 많은 이상현상이 재현에 실패한다(Hou-Xue-Zhang 2020). 따라서 논문은 사전 근거일 뿐 증거가 아니다. 증거는 다음 절차로만 얻는다: 자사 데이터에서 효과를 먼저 재현(Replication Gate) → 생존편향과 look-ahead를 차단한 walk-forward 검증 → 기준선 대비 Diebold-Mariano·SPA·MCS 검정 → 시행 횟수를 반영한 Deflated Sharpe와 PBO → 잠금 홀드아웃 1회 → 63거래일 비공개 그림자 운영. 출시 후에는 해시 체인 원장에 기록한 실제 신호의 사후 결과로 보정과 성과를 공개한다. 이 기준을 통과하지 못한 시장·기간 조합은 출시하지 않는다.
