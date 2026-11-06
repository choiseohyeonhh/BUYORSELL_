# DATA_SOURCE_SPEC — 데이터 확보 전략

버전 0.1 · 조사 기준일: 2026-11-06

> **검증 상태 고지**: 가격·호출 한도·약관은 수시로 변한다. 이 문서의 공급자 정보 중 이번 조사에서 웹으로 확인한 항목은 출처를 달았고, 그 외는 [VERIFY]로 표시했다. 확인된 항목도 상당수가 공식 페이지가 아닌 2차 자료로 확인된 것이므로 **계약 전 공식 약관·견적으로 재확인**해야 한다. 존재하지 않는 API 기능이나 가격을 추정해 기재하지 않았다.

---

## 1. 데이터 요구사항 (모델 명세에서 역산)

| 우선순위 | 데이터 | 필요 필드 | 사용처 | 필수 |
|---|---|---|---|---|
| 1 | 일별 OHLCV | O, H, L, C, V | 모든 엔진 | ✔ |
| 2 | 조정 정보 | 분할·병합·무상증자 계수, 현금배당(배당락일·금액) | 총수익 조정가 P | ✔ |
| 3 | 거래대금 | DV (KR은 거래소 제공값, US는 V × 가격 근사 가능) | Amihud, 유동성 필터 | ✔ |
| 4 | 시장 벤치마크 | SPY, MDY, KODEX 200, KODEX 코스닥150 일별 가격·배당 | 라벨, 국면 | ✔ |
| 5 | 종목 마스터 (상장폐지 포함) | 영구 ID, 티커/종목코드 이력, 상장·폐지일, 폐지 사유, 증권 유형, 시장 구분 | 생존편향 방지 | ✔ |
| 6 | 상장주식수/시가총액 | 일별 또는 공시 시점 | 유니버스(§2.2), TURN, 팩터 | ✔ (US 우선) |
| 7 | 지수 구성 이력 (PIT) | S&P 500/400, KOSPI 200/KOSDAQ 150 편입·편출일 | 유니버스 | 선택 (대체 규칙 있음) |
| 8 | 산업 분류 | GICS 섹터 또는 대체, KRX 업종 | RREV | ✔ (현재값 허용, PIT 선호) |
| 9 | 팩터 수익률 | US: Mkt-RF, SMB, HML 일별 | RMOM, IVOL | ✔ (US) |
| 10 | 무위험 수익률 | US 3개월 T-bill, KR CD 91일 | Sharpe, 현금 | ✔ |
| 11 | 시장 상태 | VIX (US 국면 보조) | 국면 (선택) | 선택 |
| 12 | 거래 상태 | KR 관리종목·투자경고·거래정지·정리매매 | 유니버스, 체결 | ✔ (KR) |
| 13 | 미시구조 (호가·틱) | — | MVP 제외 (`QUANT_RESEARCH.md` E8) | ✗ |
| 14 | 재무제표 | — | MVP 제외, 장기 모델 연구 | ✗ |

**핵심 제약**: 백테스트용 데이터는 *상장폐지 종목 포함 + 조정 정보 포함 + 장기 이력*이어야 하고, 서비스 표시용 데이터는 *상업적 재배포 라이선스*가 있어야 한다. 두 요구는 서로 다른 공급자로 충족될 수 있다.

---

## 2. 미국 시장 공급자 비교

| 공급자 | 이력·상장폐지 | 조정 정보 | 호출 제한 / 가격 | 상업적 이용·재배포 | 평가 |
|---|---|---|---|---|---|
| **Nasdaq Data Link — Sharadar (SEP 가격, TICKERS 마스터, ACTIONS)** | 재판매사 문서 기준 1998년~현재, 상장폐지 종목 포함 ([QuantRocket](https://www.quantrocket.com/pricing/data/sharadar)) [VERIFY 공식 페이지] | 제공 [VERIFY 상세 필드] | 구독제, 가격 미확인 [VERIFY] | 전문가 사용자는 Nasdaq Data Link에서 직접 구매 필요(재판매사 안내). 재배포 조건 [VERIFY] | **연구·백테스트 1순위 후보** |
| **Massive (구 Polygon.io)** | 상장폐지 포함 여부 [VERIFY] | 분할·배당 엔드포인트 [VERIFY] | 2차 자료 기준 무료 Basic(분당 5회, EOD, 2년 이력), 유료 개인 플랜 존재; 사업용·재배포는 별도 가격 ([apicostcalc](https://apicostcalc.com/blog/polygon-massive-rebrand-api-pricing.html)) | 개인 플랜은 상업 이용 불가 가능성, Business 플랜 필요 [VERIFY] | **운영 EOD 후보** |
| **Tiingo** | 장기 EOD [VERIFY 시작 연도·폐지 종목] | 조정가 제공 | 공식 문서: Basic/Power는 개인·내부 용도, Commercial은 내부 상업 용도, **어떤 형태의 재배포도 불가**, 재배포는 별도 라이선스(정액제) ([Tiingo docs](https://www.tiingo.com/documentation/general)) | 재배포 라이선스 별도 계약 필요 | **운영 EOD 후보** (재배포 계약 시) |
| **Alpha Vantage** | 상장폐지 커버리지 불명확 [VERIFY] | 조정가 제공 | 공식 premium 페이지 기준 무료 하루 25회, 유료 플랜은 분당 한도제 ([alphavantage.co/premium](https://www.alphavantage.co/premium)) | [VERIFY] | 프로토타입용. 주 공급자로 부적합 (호출 한도, 생존편향) |
| **Yahoo Finance** | 상장폐지 종목 누락 (생존편향) | 조정가 | 공식 공개 API 없음 (비공식 스크래핑) | 이용약관상 상업 이용 부적합 [VERIFY] | **제외** (탐색적 분석 외 사용 금지) |
| **EODHD** | 상장폐지 종목 데이터 별도 제공 안내 ([EODHD](https://eodhd.com/financial-apis/landing-delisted/)) | 제공 | [VERIFY] | [VERIFY] | 대안 후보 |
| **CRSP (WRDS)** | 최고 품질, 1925년~, 상장폐지 수익 포함 | 제공 | 기관 구독 | 학술 라이선스 — 상업 서비스 사용 불가 [VERIFY] | 학술 협업 시 검증 기준 데이터로만 |
| **Kenneth French Data Library** | 팩터 일별·월별 | — | 무료 | 이용 조건 [VERIFY] | **US 팩터 1순위** |
| **FRED (St. Louis Fed)** | DTB3(3개월 T-bill), VIXCLS 등 | — | 무료 API 키 | 시리즈별 저작권 상이 (VIX는 Cboe 소유) [VERIFY] | **무위험수익률 1순위** |
| **SEC EDGAR** | 공시, 상장주식수(XBRL dei) | — | 무료, 공정 사용 정책(요청 속도 제한·User-Agent 요구) | 공공 데이터 | 상장주식수 보조, 장기 모델 재무 데이터 |

### 미국 추천 조합
- **연구·백테스트**: Sharadar (가격 + 마스터 + 기업행위 + 일별 시총) + Kenneth French 팩터 + FRED.
- **운영 (매일 EOD)**: 연구와 같은 공급자의 일별 갱신을 쓰는 것이 이상적(정의 일치). 단 웹 표시(가격 차트) 재배포 라이선스가 없으면 Tiingo 또는 Massive의 재배포 가능 플랜을 별도 계약.
- **교차 검증**: 두 번째 공급자(무료 등급)로 매일 무작위 50종목 종가 대조 (`§7`).

---

## 3. 한국 시장 공급자 비교

| 공급자 | 범위 | 조정 정보 | 제한 | 상업적 이용 | 평가 |
|---|---|---|---|---|---|
| **KRX Open API** (openapi.krx.co.kr) | 일별매매정보(주식·지수 등). 인증키 발급 후 **API별 이용 신청·승인** 필요 | 2차 자료 기준 **원주가만 제공, 수정주가 미제공** | 2차 자료 기준 인증키당 일 10,000건 | 2차 자료 기준 **비상업적 목적만 허용, 원천 데이터 제3자 재배포 금지, 출처 표기** ([krx-openapi PyPI](https://pypi.org/project/krx-openapi/)) [VERIFY 공식 약관] | 연구용 원천 데이터 후보. **운영 사용 불가 가능성 높음** |
| **KRX 정보데이터시스템 (data.krx.co.kr)** | 통계·상장폐지 현황 등 | 일부 | 웹 조회 | 데이터 상품 구매·이용계약 [VERIFY] | 상장폐지 종목 목록 확보 |
| **코스콤(KOSCOM) 시세 정보** | 거래소 공식 시세 정보 분배 | 제공 [VERIFY] | 계약 | **상업적 시세 이용·재배포의 공식 경로** [VERIFY 상품·비용] | **운영 1순위 후보** (계약 필요) |
| **공공데이터포털 — 금융위원회_주식시세정보** | 일별 시세. 금융위 공지 기준 실시간 아님, 익일 13시 이후 개방 ([금융위원회](https://fsc.go.kr/in060501)) | [VERIFY] | 개발/운영 계정별 일일 호출 한도 상이 | 데이터별 이용허락범위 유형 확인 필요 [VERIFY] | 공식·무료 보조 소스 후보. 1일 지연으로 **장 마감 후 당일 신호에는 부적합**, 교차 검증·백필용 |
| **FnGuide (DataGuide) 등 상용 DB** | 수정주가·상장폐지 종목·재무 장기 이력 | 제공 | 연간 구독 | 계약 조건 [VERIFY] | **KR 백테스트 1순위 후보** (상장폐지 포함 수정주가) |
| **증권사 Open API (예: 한국투자증권 KIS Developers)** | 계좌 기반 시세 조회 | 일부 | 계좌 필요 | 개인 매매용 약관, 재배포 불가 가능성 높음 [VERIFY] | 운영 부적합 |
| **pykrx / FinanceDataReader (오픈소스)** | KRX·포털 웹 스크래핑 래퍼 | 일부 | 비공식 | 원천 사이트 약관 종속 | **프로토타입 전용**, 운영·공개 서비스 사용 금지 |
| **DART Open API** (opendart.fss.or.kr) | 공시, 주요사항보고서(유무상증자·분할·감자), 재무 | 기업행위 원천 | 호출 한도 [VERIFY — 2차 자료 간 수치 불일치] | [VERIFY] | 기업행위 교차 검증, 장기 모델 재무 |
| **한국은행 ECOS API** | CD 91일, 국고채 금리 | — | 무료 키 | [VERIFY] | **KR 무위험수익률 1순위** |

### 한국 추천 조합
- **연구·백테스트**: 상용 DB(FnGuide 등)에서 상장폐지 종목 포함 수정주가·거래대금·시가총액·관리종목 이력 확보. 확보 전 프로토타입은 KRX Open API(원주가) + DART 기업행위로 **자체 수정계수 계산** 경로를 병행 개발 (데이터 공급자 종속 탈피, 교차 검증).
- **운영**: 코스콤 등 공식 시세 이용계약. 계약 전까지 KR 서비스는 비공개 그림자 운영만.
- **무위험**: ECOS.

---

## 4. 시장별 필수 데이터 매핑

| 데이터 | US 소스 | KR 소스 |
|---|---|---|
| OHLCV | Sharadar SEP (연구) / 운영 벤더 | 상용 DB (연구) / 코스콤 (운영) / KRX Open API (프로토타입) |
| 조정계수 | Sharadar ACTIONS + 자체 재계산 | 상용 DB + DART 주요사항보고서 기반 자체 재계산 |
| 상장폐지 | Sharadar TICKERS | KRX 정보데이터시스템 상장폐지 현황 + 상용 DB |
| 벤치마크 | SPY, MDY (같은 벤더) | KODEX 200(069500), KODEX 코스닥150(229200) — ETF 상장 이전 기간은 지수 사용 [VERIFY] |
| 시가총액 | Sharadar DAILY [VERIFY] | KRX 일별 시가총액 |
| 산업 분류 | 벤더 섹터 필드 [VERIFY GICS 사용권] | KRX 업종 |
| 팩터 | Kenneth French | 자체 구축 |
| 무위험 | FRED DTB3 | ECOS CD91 |
| 거래 상태 | 벤더 | KRX (관리·경고·정지) |
| 거래 캘린더 | NYSE 휴장일 (오픈소스 `exchange_calendars` + 수동 검증) | KRX 휴장일 (동일) |

---

## 5. 데이터 수집 일정

| 작업 | US | KR |
|---|---|---|
| EOD 수집 시작 | 18:30 ET [VERIFY 벤더 확정 시각] | 18:00 KST |
| 재시도 | 15분 간격 최대 8회 | 동일 |
| 수집 마감 (이후 미수신 시 H_DATA) | 22:00 ET | 22:00 KST |
| 정정 데이터 재수집 | 익일 수집 시 직전 5거래일 재요청, 변경분 버전 기록 | 동일 |

---

## 6. 라이선스 전략

| 단계 | 사용 범위 | 필요한 권리 |
|---|---|---|
| 연구 (비공개) | 내부 분석 | 공급자 내부 이용 라이선스 |
| 그림자 운영 (비공개) | 내부 신호 산출 | 내부 상업 이용 라이선스 |
| 공개 MVP | 신호(파생 데이터) + 가격 차트 표시 | (a) 파생 데이터 공개 허용 여부 확인 (b) 가격 표시 시 재배포 라이선스 |

- 원시 가격을 대량 노출하지 않는 UI 설계(차트는 종목 상세에서만, 다운로드 기능 없음)로 재배포 범위를 최소화한다.
- 파생 신호·확률이 "공급자 데이터의 재배포"로 간주되는지는 계약별로 다르므로 계약서에 **파생 데이터 공개 조항**을 명시 요청.
- 거래소 데이터 수수료: EOD(지연) 데이터는 실시간 대비 거래소 표시 수수료 부담이 낮거나 없을 수 있으나 공급자 계약에 따름 [VERIFY].

---

## 7. 데이터 품질 관리

### 7.1 수집 시 검사 (일별, 자동)

| 검사 | 규칙 | 실패 시 |
|---|---|---|
| 완전성 | 유니버스 종목 중 당일 데이터 수신 비율 ≥ 98% | WARN, < 95% FAIL |
| OHLC 일관성 | L ≤ min(O,C) ≤ max(O,C) ≤ H, 가격 > 0, V ≥ 0 | 해당 종목 FAIL |
| 이상 수익 | \|r\| > 40% (KR: 가격제한폭 ±30% 초과 시 즉시) 이고 기업행위 이벤트 없음 | 검토 큐, 해당 종목 WARN |
| 0 거래량 연속 | 5일 이상 | 거래정지 여부 확인 |
| 교차 대조 | 보조 소스와 종가 차이 > 0.5% (무작위 50종목) | WARN, 비율 > 5% FAIL |
| 조정 대조 | 자체 재계산 조정계수 vs 벤더 조정가 차이 > 1bp | 검토 큐 |
| 캘린더 | 휴장일 데이터 존재, 거래일 누락 | FAIL |
| 과거 값 변경 | 이전 스냅샷 대비 과거 가격 변경 | 버전 기록, 변경 리포트 |

### 7.2 스냅샷과 버전
- 원천 응답(raw)은 변경 없이 객체 저장소에 날짜·소스별로 보존 (`SYSTEM_ARCHITECTURE.md §4`).
- 정규화 테이블은 `snapshot_id`(UTC 타임스탬프 + 내용 해시)로 버전 관리. 모든 신호는 snapshot_id를 기록.

---

## 8. MVP 비용 최소화 경로

| 단계 | 데이터 | 비용 수준 |
|---|---|---|
| Phase 0–1 (연구) | US: Sharadar 개인/연구 구독 [VERIFY 가격]. KR: KRX Open API(무료) + DART + ECOS로 프로토타입, 상장폐지 포함 상용 DB 1년 구독 협상 | 저 |
| Phase 2 (그림자 운영) | 동일 + 운영 벤더 내부 상업 플랜 | 중 |
| Phase 3 (공개) | US 재배포 플랜(Tiingo/Massive 등 견적 비교), KR 코스콤 등 시세 이용계약 | 중~고 — **출시 비용의 최대 항목**, 견적 확보가 Gate 조건 |

무료 소스(Yahoo, 비공식 스크래핑)로 공개 서비스를 운영하는 경로는 라이선스·생존편향 문제로 채택하지 않는다.

---

## 9. 미해결 확인 사항 (Phase 0 체크리스트)

1. Sharadar SEP/TICKERS/ACTIONS/DAILY 공식 필드 정의, 상장폐지 커버리지, 상업 이용·파생 데이터 공개 조건, 가격.
2. Tiingo·Massive 재배포 플랜 견적과 파생 데이터 조항.
3. KRX Open API 공식 이용약관(상업 이용 금지 범위), 호출 한도.
4. 코스콤 시세 이용계약 상품·비용, 파생 데이터 공개 가능 여부.
5. 상용 KR DB의 상장폐지 종목 수정주가 제공 시작 연도.
6. 공공데이터포털 주식시세정보 이용허락 유형과 제공 시작일.
7. DART Open API 호출 한도 및 이용 조건 (2차 자료 간 불일치).
8. GICS 분류 사용권 (S&P/MSCI 상표·라이선스) — 미확보 시 자체 산업 분류 또는 SIC 사용.
9. Kenneth French 데이터 상업 서비스 내부 사용 허용 여부.

### 조사 출처 (이번 세션 확인)
- [Alpha Vantage Premium](https://www.alphavantage.co/premium), [Macroption: Alpha Vantage limits](https://www.macroption.com/alpha-vantage-api-limits/)
- [apicostcalc: Polygon → Massive rebrand pricing](https://apicostcalc.com/blog/polygon-massive-rebrand-api-pricing.html), [Massive vs Tiingo](https://massive.com/compare/massive-vs-tiingo)
- [Tiingo API documentation — General](https://www.tiingo.com/documentation/general), [Tiingo IEX API product page](https://api.tiingo.com/products/iex-api)
- [krx-openapi (PyPI)](https://pypi.org/project/krx-openapi/), [pykrx-openapi (PyPI)](https://pypi.org/project/pykrx-openapi/)
- [QuantRocket: Sharadar pricing/data](https://www.quantrocket.com/pricing/data/sharadar), [QuantRocket Sharadar docs](https://www.quantrocket.com/docs/data/fundamental/sharadar/)
- [EODHD: Delisted companies data](https://eodhd.com/financial-apis/landing-delisted/)
- [금융위원회 금융공공데이터 안내](https://fsc.go.kr/in060501)
