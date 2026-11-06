# BUY OR SELL

**Quantitative Investment Intelligence Platform** — 실증적으로 검증된 퀀트 전략과 통계 모델로 개별 종목의 BUY / HOLD / SELL 판단과 그 신뢰도를 제공하는 개인 투자자용 의사결정 지원 서비스.

> 현재 단계: **설계(Design) 단계**. 서비스 코드는 아직 없으며, 모든 모델은 *검증 전 가설*이다.
> 문서에 등장하는 임계값·가중치·기간은 별도 표기가 없는 한 "사전 등록된 초기값(pre-registered default)"이며, 실제 값은 `BACKTEST_SPEC.md`의 절차로 추정·검증된 뒤 확정된다.

## 설계 문서

| # | 문서 | 내용 |
|---|------|------|
| 1 | [docs/PRODUCT_PLAN.md](docs/PRODUCT_PLAN.md) | 서비스 철학, 타깃, 시나리오, 기능, MVP, 차별점, 법적 검토 요약 |
| 2 | [docs/QUANT_RESEARCH.md](docs/QUANT_RESEARCH.md) | 학술 연구 조사, 전략별 근거, Core/Supporting/Experimental/Rejected 분류 |
| 3 | [docs/QUANT_MODEL_SPEC.md](docs/QUANT_MODEL_SPEC.md) | 엔진별 수학적 정의, Feature, 앙상블, BUY/HOLD/SELL 알고리즘 |
| 4 | [docs/BACKTEST_SPEC.md](docs/BACKTEST_SPEC.md) | 백테스트 프레임워크, 데이터 분리, 체결·비용, 편향 방지 |
| 5 | [docs/MODEL_VALIDATION_SPEC.md](docs/MODEL_VALIDATION_SPEC.md) | 확률 보정, 통계적 유의성, 드리프트, 재학습, 변경 이력 |
| 6 | [docs/DATA_SOURCE_SPEC.md](docs/DATA_SOURCE_SPEC.md) | 데이터 공급자 비교, 라이선스, 한국/미국 확보 전략 |
| 7 | [docs/SYSTEM_ARCHITECTURE.md](docs/SYSTEM_ARCHITECTURE.md) | 기술 스택, 파이프라인, DB 스키마, API, 배포 |
| 8 | [docs/UI_UX_SPEC.md](docs/UI_UX_SPEC.md) | 모바일/데스크톱 화면, 사용자 흐름, 디자인 시스템 |
| 9 | [docs/DEVELOPMENT_ROADMAP.md](docs/DEVELOPMENT_ROADMAP.md) | 단계, 의존성, 난이도, 완료 조건(Gate) |
| 10 | [docs/LEGAL_REGULATORY_REVIEW.md](docs/LEGAL_REGULATORY_REVIEW.md) | 한·미 규제, 데이터 라이선스, 성과 표시, 전문가 검토 항목 |
| 11 | [docs/FINAL_REPORT.md](docs/FINAL_REPORT.md) | 최종 보고 및 핵심 질문에 대한 답 |

## 문서 읽는 순서

`FINAL_REPORT` → `PRODUCT_PLAN` → `QUANT_RESEARCH` → `QUANT_MODEL_SPEC` → `BACKTEST_SPEC` → `MODEL_VALIDATION_SPEC` → 나머지.

## 표기 규약

- **[FACT]** 원문 또는 공식 자료로 확인된 사실 (출처 명기)
- **[HYP]** 설계 가설 — 자사 데이터로 검증 전
- **[PARAM]** 사전 등록된 초기 파라미터 — 검증 절차로 확정 예정
- **[VERIFY]** 착수 전 공식 자료로 재확인이 필요한 정보 (가격, 약관, 세율 등 변동 가능 정보)
