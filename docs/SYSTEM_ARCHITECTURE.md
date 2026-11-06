# SYSTEM_ARCHITECTURE — 시스템 아키텍처

버전 0.1 · 관련: `QUANT_MODEL_SPEC.md`, `BACKTEST_SPEC.md §14`, `DATA_SOURCE_SPEC.md`

---

## 1. 설계 원칙

1. **배치 중심 모놀리스**: 신호는 하루 1회(시장별) 산출되므로 실시간 스트리밍·마이크로서비스가 불필요. MVP는 *하나의 Python 퀀트 패키지 + 하나의 API 서비스 + 하나의 웹 앱*.
2. **연구 코드 = 운영 코드**: Feature·엔진·결정 엔진은 단일 Python 패키지(`buyorsell_quant`)로 구현하고 백테스트와 운영 배치가 같은 함수를 호출.
3. **불변 데이터 + 스냅샷**: 원천 데이터·Feature·신호는 덮어쓰지 않고 버전으로 누적.
4. **퀀트 계산은 서버 배치에서만**: 프론트엔드와 API는 저장된 결과를 읽기만 한다. API는 계산하지 않는다.
5. **재현성**: (data snapshot, model version, code commit) → 결정론적 출력.

---

## 2. 기술 스택 (추천)

| 영역 | 선택 | 이유 | 대안 |
|---|---|---|---|
| 퀀트·데이터 언어 | **Python 3.12** | 통계·ML 생태계(statsmodels, arch, scikit-learn, LightGBM, hmmlearn) | — |
| 데이터프레임 | **Polars** (주), NumPy, pandas(라이브러리 호환용) | 패널 연산 성능, 지연 평가 | pandas 단독 |
| 분석 엔진 | **DuckDB** | Parquet 직접 질의, 연구·백테스트용 로컬 OLAP | — |
| 통계·ML | statsmodels, arch (GARCH), scikit-learn, LightGBM, hmmlearn, SciPy | 검증된 구현 | — |
| 가속 | Numba (결정 엔진 상태기계 루프) | 종목×일 순차 상태 처리 | Cython |
| 패키지/환경 | **uv** + `uv.lock`, Docker 이미지 | 재현 가능한 의존성 고정 | Poetry |
| 오케스트레이션 | **Prefect 3** (또는 MVP는 단순 CLI + 스케줄러) | 재시도·의존성·관측 | Dagster, Airflow(과함) |
| 서빙 DB | **PostgreSQL 16** (관리형) | 신호·원장·메타데이터, 제약조건·트리거로 불변성 보장 | — |
| 데이터 레이크 | **S3 호환 객체 저장소** (예: Cloudflare R2, AWS S3) + Parquet | 원천·정규화·Feature 스냅샷, 저비용 | — |
| API | **FastAPI** (Python) | 퀀트 패키지의 Pydantic 스키마 재사용, OpenAPI 자동 생성 | — |
| 프론트엔드 | **Next.js (App Router) + TypeScript + Tailwind CSS** | SSR/ISR로 종목 페이지 SEO·캐싱, 반응형 | SvelteKit |
| 차트 | **TradingView Lightweight Charts** (가격·신호), **Recharts/visx** (확률·보정 곡선) | 금융 차트 경량 라이브러리 (Apache-2.0, 표기 의무 [VERIFY]) | ECharts |
| 캐시 | CDN (HTTP 캐시 + ETag) — MVP는 Redis 없음 | 신호는 하루 1회 바뀌는 불변 데이터 | Redis (Phase 2) |
| 모니터링 | Sentry(에러), Prefect UI(파이프라인), 자체 모델 모니터링 대시보드(Postgres + 내부 페이지), Uptime 체크 | | Grafana |
| 배포 | 웹: Vercel / API: 컨테이너 플랫폼(Cloud Run, Fly.io 등) / 배치: 컨테이너 잡 + 스케줄러 | 관리 부담 최소 | 단일 VM + docker compose |
| CI | GitHub Actions: lint, 타입 검사, 단위 테스트, look-ahead 절단 테스트, 재현성 해시 테스트 | | |

---

## 3. 저장소 구조 (Monorepo)

```
/
├─ docs/                         설계 문서
├─ quant/                        Python 패키지 buyorsell_quant
│  ├─ data/        ingest/, normalize/, calendar.py, universe.py, adjust.py, quality.py
│  ├─ features/    transforms.py (L0), momentum.py, reversal.py, trend.py, volatility.py, liquidity.py, regime.py
│  ├─ models/      ensemble_ordinal.py, challenger_gbdt.py, calibration.py, confidence.py
│  ├─ decision/    engine.py (상태기계), costs.py, reasons.py
│  ├─ backtest/    (BACKTEST_SPEC §14)
│  ├─ validation/  metrics.py, stats_tests.py, drift.py, reports/
│  ├─ registry/    experiments.py, model_registry.py
│  └─ io/          lake.py (Parquet 스냅샷), db.py (Postgres), hashing.py
├─ pipelines/                    Prefect flows: daily_us.py, daily_kr.py, outcomes.py, retrain.py, monitor.py
├─ services/api/                 FastAPI 앱 (읽기 전용)
├─ apps/web/                     Next.js 앱
├─ infra/                        Dockerfile, IaC, DB 마이그레이션 (SQL)
├─ reports/                      replication/, validation/ (커밋되는 검증 결과)
└─ tests/                        unit/, leakage/, reproducibility/, e2e/
```

---

## 4. 데이터 파이프라인

```
[벤더 API] ─▶ ① Ingest ─▶ raw/ (객체 저장소, 원본 그대로, 불변)
                         │
                         ▼
               ② Normalize & Validate ─▶ normalized/{snapshot_id}/ (Parquet: prices, actions, master, calendar)
                         │                 + data_quality_events (Postgres)
                         ▼
               ③ Adjust & Universe ─▶ adjusted prices, universe_membership
                         │
                         ▼
               ④ Feature Compute (L0, 엔진 Feature, 국면) ─▶ features/{snapshot_id}/{date}.parquet
                         │
                         ▼
               ⑤ Model Inference (Ensemble → Calibration → Confidence) 
                         │
                         ▼
               ⑥ Decision Engine (상태기계, 이전 상태 로드) 
                         │
                         ▼
               ⑦ Publish: signals_current 갱신 + signal_ledger append (해시 체인) + CDN 무효화
                         │
                         ▼
               ⑧ Outcomes (만기 도래 신호 결과 계산) & ⑨ Monitoring (드리프트·성능·데이터)
```

| 단계 | 입력 | 출력 | 실패 시 |
|---|---|---|---|
| ① | 벤더 | raw JSON/CSV + 메타(요청 시각, 응답 해시) | 재시도 → 마감 시 FAIL |
| ② | raw | 표준 스키마, 품질 검사 (`DATA_SOURCE_SPEC.md §7`) | 종목 단위 플래그 / 전체 FAIL |
| ③ | normalized | 조정가, PIT 유니버스 | FAIL |
| ④ | ③ | Feature 행렬 | 종목 단위 결측 처리 |
| ⑤ | ④ + 활성 모델 아티팩트 | 확률, 신뢰도 | 모델 로드 실패 → 전체 중단 |
| ⑥ | ⑤ + 직전 상태 | 결정 + 사유 | — |
| ⑦ | ⑥ | DB 쓰기 (단일 트랜잭션) | 롤백, 이전 신호 유지 |
| ⑧ | 원장 + 최신 가격 | signal_outcomes | 다음 날 재시도 |
| ⑨ | 전 단계 | 지표, 경보 | 경보 발송 |

- 각 Flow run은 `run_id`, `snapshot_id`, `model_versions`, `code_commit`을 `pipeline_runs`에 기록.
- 멱등성: 같은 (market, date, snapshot_id)로 재실행하면 동일 결과, 원장에는 중복 기록하지 않음(유니크 제약). 다른 snapshot_id로 재실행(데이터 정정)되면 원장에 **새 레코드(정정 표시)**를 추가하고 기존 레코드는 유지.

### 4.1 스케줄

| Job | US | KR |
|---|---|---|
| daily pipeline ①–⑦ | 평일 18:30 ET 시작, 20:00 ET 공개 목표 | 평일 18:00 KST 시작, 19:30 KST 공개 목표 |
| outcomes ⑧ | 매일 ① 이후 | 동일 |
| monitor ⑨ | 매일 ⑦ 이후, 주간 리포트 월요일 | 동일 |
| retrain | 분기 (§MODEL_VALIDATION_SPEC 6.2) | 동일 |
| 백업 | Postgres PITR + 일일 논리 백업, 객체 저장소 버전 관리 | |

휴장일은 거래 캘린더로 건너뛴다.

---

## 5. 퀀트 엔진 실행 구조

- 계산 단위: (market, date). US ~900 종목 × Feature ~20개는 단일 컨테이너(4 vCPU, 8–16GB)에서 수 분 내 처리 가능한 규모로 추정 [HYP — 프로파일링으로 확인].
- GARCH 종목별 재추정(주 1회)은 병렬화 (프로세스 풀).
- 모델 아티팩트: `models/{model_version}/` — `coefficients.json`, `calibration.json`, `feature_spec.json`(정의 해시), `train_meta.json`, `sha256sums`.
- 추론 시 `feature_spec.json` 해시와 현재 Feature 코드의 정의 해시가 다르면 실행 중단(명세 불일치 방지).

---

## 6. 데이터베이스 (PostgreSQL)

### 6.1 주요 테이블

```sql
-- 참조 데이터
securities(security_id PK, market, name_ko, name_en, security_type, listing_date, delisting_date, delisting_reason)
security_identifiers(security_id FK, id_type /*ticker|krx_code|figi*/, value, valid_from, valid_to)
trading_calendar(market, date, is_trading_day, session_close_local)
tax_schedule(market, segment, valid_from, valid_to, sell_tax_rate)

-- 시계열 (서빙·UI용 최근 데이터; 전체 이력은 Parquet)
prices_daily(security_id, date, open, high, low, close, volume, traded_value, adj_factor, snapshot_id, PRIMARY KEY(security_id, date, snapshot_id))
universe_membership(market, date, security_id, eligible, exclusion_code)
regime_daily(market, date, regime_code, p_high_vol, trend_m, vol_pct_m, breadth_200, model_version)

-- 신호
signals_current(market, security_id, horizon, as_of_date, decision, reason, state_since, payload JSONB, ledger_seq FK)
signal_ledger(seq BIGSERIAL PK, market, as_of_date, security_id, horizon, payload JSONB, payload_hash BYTEA,
              prev_chain_hash BYTEA, chain_hash BYTEA, created_at TIMESTAMPTZ, supersedes_seq BIGINT NULL,
              UNIQUE(market, as_of_date, security_id, horizon, payload_hash))
signal_outcomes(ledger_seq FK, version, matured_date, excess_log_return, label, net_return, data_snapshot_id, computed_at)
performance_aggregates(market, horizon, window, as_of_date, metrics JSONB, ci JSONB, n_signals, n_dates)

-- 모델·실험·운영
model_registry(...)          -- MODEL_VALIDATION_SPEC §6.6
experiments(...)             -- BACKTEST_SPEC §13
pipeline_runs(run_id, market, date, snapshot_id, code_commit, model_versions JSONB, status, started_at, finished_at)
data_quality_events(id, market, date, security_id, check, severity, detail JSONB)
monitoring_metrics(market, horizon, date, metric, value)
```

### 6.2 불변성 보장
- `signal_ledger`, `signal_outcomes`, `model_registry`, `experiments`에 대해 애플리케이션 역할에서 `UPDATE`, `DELETE` 권한 회수 + `BEFORE UPDATE OR DELETE` 트리거로 예외 발생.
- 정정은 `supersedes_seq`를 가진 새 레코드로만.

### 6.3 해시 체인 원장
```
payload_canonical = JSON (키 정렬, 공백 없음, 숫자 소수 10자리 고정)
payload_hash      = SHA-256(payload_canonical)
chain_hash_n      = SHA-256(chain_hash_{n−1} || payload_hash_n)
```
- 매일 시장별 마지막 chain_hash를 **일일 앵커**로 공개 (웹사이트 원장 페이지 + 공개 Git 저장소 커밋 [PARAM]). 외부에 공개된 앵커가 있으면 과거 레코드 변조가 탐지 가능.
- 원장 페이지에서 사용자가 특정 날짜의 레코드를 내려받아 해시를 재계산할 수 있도록 검증 스크립트 공개.

---

## 7. API (읽기 전용, `/v1`)

| Method | Path | 설명 | 캐시 |
|---|---|---|---|
| GET | `/v1/search?q=&market=` | 종목 검색 (이름·티커·코드·한글 초성) | 1일 |
| GET | `/v1/stocks/{market}/{symbol}` | 종목 기본 정보, 유니버스 적격성 | 1일 |
| GET | `/v1/stocks/{market}/{symbol}/signals` | 단기·중기 현재 신호 (§16.1 레코드 + 표시 정책 적용) | 다음 공개 시각까지 |
| GET | `/v1/stocks/{market}/{symbol}/signals/history?horizon=&from=&to=` | 신호 이력 + 결과 | 다음 공개 시각까지 |
| GET | `/v1/stocks/{market}/{symbol}/chart?range=` | 가격·거래량·이동평균·신호 마커 (라이선스 범위 내) | 다음 공개 시각까지 |
| GET | `/v1/stocks/{market}/{symbol}/similar-signals?horizon=` | 같은 신호·등급의 과거 결과 분포 (서비스 전체 기준) | 1일 |
| GET | `/v1/markets/{market}/regime` | 국면 현재값·최근 이력 | 다음 공개 시각까지 |
| GET | `/v1/performance?market=&horizon=&window=` | 공개 성과 (`MODEL_VALIDATION_SPEC.md §5.4`) | 1일 |
| GET | `/v1/ledger/{market}/{date}` | 해당일 원장 레코드 + 앵커 해시 | 영구(불변) |
| GET | `/v1/models` · `/v1/models/{version}` | 모델 카드 | 배포 시 무효화 |
| GET | `/v1/status` | 데이터·파이프라인 상태 (지연 여부) | 1분 |

- 응답 공통 필드: `as_of_date`, `data_cutoff_utc`, `model_version`, `disclaimer_version`.
- 레이트 리밋: IP당 분당 60 요청 [PARAM] (데이터 대량 수집·재배포 방지).
- 원시 가격의 대량 다운로드 API는 제공하지 않는다 (라이선스).

---

## 8. 프론트엔드 아키텍처

- Next.js App Router, 종목 페이지는 **ISR**(공개 시각 후 재검증) — 대부분의 요청이 CDN 캐시에서 처리.
- 서버 컴포넌트에서 API 호출, 클라이언트 컴포넌트는 차트·인터랙션만.
- i18n: ko(기본), en. 숫자·날짜·통화 로캘 포맷.
- 퀀트 로직 없음: 표시 정책(확률 표시 여부 등)도 API 응답의 플래그를 따른다.

---

## 9. 보안·운영

| 영역 | 조치 |
|---|---|
| 비밀 관리 | 벤더 API 키는 플랫폼 시크릿 매니저, 저장소 커밋 금지 |
| DB 권한 | 역할 분리: `pipeline_writer`(INSERT), `api_reader`(SELECT), `admin` |
| 개인정보 | MVP는 계정 없음 → 수집 최소화 (접속 로그만, IP 익명화 보존기간 [PARAM]) |
| 가용성 | API·웹은 무상태, CDN 캐시로 DB 장애 시에도 최근 신호 열람 가능 |
| 감사 | 파이프라인 실행·모델 배포·롤백 이벤트 로그 |
| 관측 | 파이프라인 SLA(공개 지연) 경보, 에러 추적, 모델 모니터링 대시보드 |

---

## 10. 확장 경로 (MVP 이후)

| 필요 | 확장 |
|---|---|
| 계정·관심종목·알림 (Phase 2) | 인증(관리형 Auth), `users`, `watchlists`, 알림 큐 (신호 변경 이벤트 → 이메일/푸시) |
| 트래픽 증가 | Redis 캐시, 읽기 복제본 |
| 장기 모델·재무 데이터 | PIT 재무 테이블, 별도 Feature 모듈 |
| 연구 규모 증가 | 백테스트를 분산 실행(Ray 등) — 필요 시점에만 |
