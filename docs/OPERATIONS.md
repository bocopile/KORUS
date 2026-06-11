# AlphaRadar — 운영 수칙 / 참고 링크 (12~13장)

> 원본: CLAUDE.md 12~13장에서 분리
> 관련 문서: [STRATEGY.md](STRATEGY.md), [DEVELOPMENT.md](DEVELOPMENT.md)
> 우선순위: 이 문서의 운영/안전 규칙이 모든 다른 규칙보다 우선합니다.

---

## 12. 운영 필수 수칙

> **투자 경고**: 이 시스템은 보조 도구입니다. 모든 투자 책임은 본인에게 있습니다.

**필수 운영 순서:**
1. `IS_PAPER_TRADING=true` 모의투자 **최소 3개월** 검증
2. 소액 실전 (총 자산 **10% 이내**) 테스트
3. 결과 검토 후 점진적 확대 → Tier3는 마지막 활성화

**보안:**
- `.env` Git 커밋 절대 금지
- GitHub Secrets으로 API 키 관리
- 서버 SSH 키 인증만 허용

**Graceful Degradation 원칙:**
부분 실패는 전체 중단이 아닌 `null/stale/skip/alert`로 격리한다. **신규 매수는 fail-closed(장애 시 중단), 매도/손절/정산은 fail-open(장애 시에도 지속).**

**장애 대응:** (Claude 단일 벤더 SPOF 상세 절차는 **12-1**)
- KIS API 장애 → Fail-Closed (신규 매매 자동 중단). 체결 모니터링은 재연결 시도, 손절/정산은 계속
- 관점 1개 장애(서브에이전트 호출 실패) → 나머지 2개로 합의 (2/2 일치 시 SOFT_WARN·0.70 실행) — DEGRADED
- 관점 2개 장애 → confidence_score = 0.55 → skip + 긴급 알림 (보유 청산 시그널만 예외)
- **관점 3개 동시 장애 = Anthropic 전면 장애(SPOF)** → **AI_HALT** 발동 (12-1). 신규 진입 차단, 결정론 보호 유지
- 네트워크 장애 → WebSocket 자동 재연결 + 실패 시 알림
- 데이터 불일치 → 매매 중단 + 수동 확인

---

## 12-1. Claude 단일 벤더(SPOF) 장애 대응 절차

> v1.5에서 멀티벤더(Gemini/ChatGPT)를 폐기하면서 **Anthropic API가 단일 장애점(SPOF)**이 되었다.
> 우회할 다른 벤더가 없으므로, 대응 전략은 "다른 벤더로 페일오버"가 아니라
> **"AI 없이도 도는 결정론 보호 모드로 안전하게 강등(degrade)"**이다.
>
> **핵심 원칙**: AI는 *신규 진입의 판단*에만 필요하다. 손절·청산·Guard·정산은 전부 AI 없는 Go 코드 →
> Anthropic이 죽어도 **지키는 로직은 계속 돈다**. 한 줄 요약: *"버는 행동은 멈추고, 지키는 행동은 계속."*

### 12-1-1. 장애 분류

| 등급 | 정의 | 판정 기준 | 상태 전환 |
|------|------|-----------|-----------|
| **부분 (DEGRADED)** | 3관점 중 1~2개 호출 실패 | 개별 서브에이전트 timeout(30s)/파싱실패/일시 429 | 합의 축소(3-4 재계산). **HALT 아님** |
| **전면 (AI_HALT=SPOF)** | 3관점 동시 실패 = Anthropic 전면 불가 | 연속 3회 5xx/타임아웃, 또는 서킷 OPEN | NORMAL → AI_HALT |
| **인증·계정** | 키 만료/권한박탈/결제실패/정책차단 | 401 / 403 / 402 | 즉시 AI_HALT + **사람 호출(자동복구 불가)** |
| **레이트리밋** | 일시적 한도 초과 | 429 (+Retry-After) | 백오프 재시도. 지속 시 DEGRADED→AI_HALT |

> transient(5xx/429)는 재시도로 흡수, **hard(401/403/402)는 자동복구 불가** → 즉시 사람 개입.

### 12-1-2. 감지 (Detection)

- **Circuit Breaker** (sony/gobreaker, 기존 의존성): 연속 실패 3회 또는 (10건 중 60% 실패) → **OPEN**(호출 즉시 차단). OpenTimeout 60s 후 half-open에서 1건 프로브.
- **재시도**: 5xx/타임아웃 → exponential backoff + jitter (1s→2s→4s, 최대 3회). 429 → `Retry-After` 준수.
- **헬스 프로브**: 장중 1분 주기 경량 ping 호출로 상태 선반영.
- **타임아웃**: 서브에이전트 호출 `ai.timeout`(30s) 초과 = 실패 카운트 1.

### 12-1-3. 강등 상태와 허용 동작 (Degradation Matrix)

| 동작 | NORMAL | DEGRADED (관점 1~2개 장애) | AI_HALT (SPOF) | 근거 |
|------|:---:|:---:|:---:|------|
| 신규 매수/진입 (AI 시그널 필요) | ✅ | △ 2/2 합의 시만(0.70, 50%축소). 1/2·risk veto면 skip | ⛔ 중단 (fail-closed) | 진입 판단은 AI 필수 |
| 손절(-7%/-3%) 자동 매도 | ✅ | ✅ 계속 | ✅ 계속 (fail-open) | 결정론 규칙, AI 불필요 |
| 추적 손절 / 익절 | ✅ | ✅ 계속 | ✅ 계속 | 결정론 |
| Guard 계층 (HARD_BLOCK 등) | ✅ | ✅ 계속 | ✅ 계속 | 결정론 |
| 체결 모니터링 (WebSocket) | ✅ | ✅ 계속 | ✅ 계속 | KIS, AI 무관 |
| 미체결 정리(14:50)·정산/스냅샷(15:30) | ✅ | ✅ 계속 | ✅ 계속 | 결정론 |
| 리밸런싱 — 신규 시작 | ✅ | △ 후보 합의 가능 시만 | ⛔ 보류 | 목표비중 승인에 AI 관여 |
| 리밸런싱 — 진행분 완료 | ✅ | ✅ 계속 | ✅ 마지막 승인 스냅샷으로 완료 | 계획 이미 결정론 확정 |
| 모닝 브리핑 | ✅ | ✅ (단일 Claude) | ⚠️ 데이터-only fallback | AI 요약 생략, raw 지표·뉴스만 |
| 뉴스 감성 분석 | ✅ | △ 뉴스 관점 생존 시만 | ⛔ skip | 뉴스 관점 서브에이전트 의존 |

### 12-1-4. 대응 런북 (Runbook)

```
[T+0] 감지: 서킷 OPEN 또는 연속 3회 실패
  → 상태 = AI_HALT, 긴급 알림(Slack+Telegram): "AI_HALT: Anthropic 전면 장애 의심"
[T+0] 즉시 조치 (자동):
  - 신규 진입 시그널 생성 전면 차단 (fail-closed)
  - 결정론 보호(손절/추적/Guard/모니터링/정산) 정상 유지 — 무간섭
  - 진행 중 리밸런싱은 마지막 승인 스냅샷으로 완료, 신규 시작 보류
  - 미체결 '신규 매수' 지정가 잔량 취소 검토(체결 위험 격리). '매도' 주문은 유지
[T+1m~] 자동 복구 프로브: half-open 1건 성공 → RECOVERING / 실패 → AI_HALT 유지
[사람 개입 트리거]:
  - 401/403/402 (인증·계정·결제) → 자동복구 불가. 운영자가 키/결제/한도 점검
  - AI_HALT 15분 이상 지속 → 운영자 호출, 보유 포지션 수동 청산/홀드 판단
[복구 검증] RECOVERING:
  - 헬스 프로브 연속 3회 성공 + 3관점 정상 응답 1회 → NORMAL 복귀
  - 복귀 즉시 신규 진입 재개 금지 → 다음 정규 스케줄(7-3)부터 재평가
[복구 후 backlog 정책]:
  - 장애 중 누락 시그널 재생성/소급 실행 금지 (stale). 현재 데이터로 새로 평가
  - 장애 구간 상태 전환 이력·실패 원본 DB 저장 → `system_events` 테이블 (ARCHITECTURE 5-3)
```

### 12-1-4-1. 프로세스 재시작 시 Reconciliation (crash 복구)

> 코어 프로세스 비정상 종료/재시작 시, 메모리 상태를 잃으므로 기동 직후 KIS·DB와 정합을 맞춘다.

```
[코어 기동 시 순서]
1. KIS 잔고/미체결 조회 ↔ 내부 orders/positions 대조 (Reconciliation).
2. 상태 불일치 → 매매 중단(HALT) + 알림. 수동 확인 전 신규 진입 금지.
3. 미완료 rebalance_runs(status=EXECUTING) 탐지 → 멱등키(rebalance_ledger)로 기체결분 제외,
   잔여만 재개 또는 ABORTED 처리(승인 스냅샷 기준).
4. 부분체결 주문: filled_qty 기준 잔량 재계산 후 처리.
5. data_freshness 검사 → stale 소스 기반 신규 진입 보류.
6. 정합 완료 후에만 NORMAL 진입.
※ 모든 주문은 idempotency_key로 보호되므로 재시작 중복 주문은 발생하지 않아야 한다.
```

### 12-1-5. 구현 체크리스트

- [ ] gobreaker 서킷 + backoff/jitter 재시도 (`ai/client.go`)
- [ ] 기동 시 Reconciliation(KIS 잔고/미체결 ↔ DB) + 미완료 rebalance_runs 재개/중단 처리
- [ ] 상태 머신 `NORMAL/DEGRADED/AI_HALT/RECOVERING` 전역 관리 + 전환 이력 `system_events` 기록
- [ ] AI_HALT 시 **신규 진입 경로만 차단**, 결정론 보호 경로(`portfolio`·`guard`·`trading`)는 독립적으로 무간섭 동작
- [ ] 인증 실패(401/403/402)와 일시 장애(5xx/429) 분기 처리
- [ ] API 키 만료 D-7 사전 알림 + rate-limit 사용량 모니터링
- [ ] **월 1회 SPOF 드릴**: 키를 일시 무효화해 AI_HALT 전환 → 결정론 보호 지속 → 복구를 모의 검증

---

## 12-2. KRX 이상거래 감시 대응 지침

> KRX 시장감시위원회는 자동매매 포함 모든 주문 패턴을 24시간 모니터링합니다.
> 아래 규칙은 Guard 계층(7-6-5)과 연동하여 코드로 강제됩니다.

**금지 패턴 (Guard 강제 적용):**

| 패턴 | Guard 처리 | 근거 |
|------|-----------|------|
| 주문 후 즉시 취소 반복 (허수호가/Spoofing) | HARD_BLOCK (3회 연속 미체결 취소 시 당일 해당 종목 신규 주문 차단) | 시세조종 오인 방지 |
| 15:20~15:30 동시호가 시간대 신규 매수 | HARD_BLOCK | 종가 형성 관여 방지 |
| 초단위 고빈도 주문 | HARD_BLOCK (최소 주문 간격 3초 미준수 시 차단) | HFT 의심 패턴 방지 |
| 상/하한가 ±3% 이내 반복 주문 | SOFT_WARN | 상/하한가 관여 의심 방지 |
| 소형주 집중 매매 | HARD_BLOCK (MIN_TRADE_VALUE 필터) | 가격 영향력 차단 |

**소명 대응 원칙:**
- 거래소 소명 요청 시 `guard_logs` + `orders` + `ai_consensus` 데이터를 근거로 제출
- 모든 주문에 전략 판단 근거(`signal_id → ai_consensus.id`) 연결 필수
- 정정/취소 주문 시 원주문 ID(`parent_order_id`) 반드시 연결 저장

---

## 12-3. 소명용 Audit Log 규격

> 이상거래 소명 요청 시 제출 가능한 형태로 로그를 보존합니다.

| 항목 | 저장 위치 | 보존 기간 |
|------|-----------|-----------|
| 전략 판단 근거 | `ai_responses.reason` (consensus_id 조인) + `ai_consensus.dissent_reason` + `guard_logs` | 영구 |
| 주문 시점 시세 스냅샷 | `orders.context_json` | 영구 |
| 정정/취소 사유 | `orders.cancel_reason` | 영구 |
| 원주문-정정/취소 연결 | `orders.parent_order_id` | 영구 |

**orders 테이블 보완 컬럼 (ARCHITECTURE.md 5-3 연동):**
```sql
-- ※ 아래 컬럼은 ARCHITECTURE.md 5-3 기본 스키마에 이미 포함됨.
-- 하위호환 마이그레이션 참고용으로만 남겨 둠.
ALTER TABLE orders ADD COLUMN parent_order_id INTEGER;
ALTER TABLE orders ADD COLUMN cancel_reason TEXT;
ALTER TABLE orders ADD COLUMN context_json TEXT;
```

---

## 12-4. 법적 주의사항

> ⚠️ 자본시장법 위반 방지 — 운영 전 반드시 확인

- **타인 자금 운용 금지**: 금융투자업 등록 없이 타인의 자금을 AlphaRadar로 운용하는 것은 자본시장법 위반
- **불공정거래 금지**: 시세조종·허수호가·내부자 정보 기반 매매 → 형사처벌 대상 (자본시장법 제176~178조)
- **자동매매 감시 인지**: AlphaRadar는 합법적 알고리즘 매매이나, 소형주 고빈도 주문 또는 주문/취소 반복 시 이상거래 탐지 대상 가능
- **매매 로그 보관 의무**: 세금 신고 5년, 이상거래 소명 목적 → `orders` / `positions` 테이블 영구 보관

---

## 13. 참고 링크

| 항목 | URL |
|------|-----|
| KIS Developers | https://apiportal.koreainvestment.com |
| KIS GitHub 샘플 | https://github.com/koreainvestment/open-trading-api |
| KIS examples_llm (Go 구현 레퍼런스) | https://github.com/koreainvestment/open-trading-api/tree/main/examples_llm |
| OpenDART API | https://opendart.fss.or.kr |
| 한국은행 ECOS | https://ecos.bok.or.kr/api |
| KRX Open API | https://data.krx.co.kr |
| FRED API | https://fred.stlouisfed.org/docs/api |
| SEC EDGAR API | https://efts.sec.gov/LATEST/search-index |
| Claude Cowork 스케줄링 | https://support.claude.com/en/articles/13854387 |
| go-talib | https://github.com/markcheno/go-talib |
| gorilla/websocket | https://github.com/gorilla/websocket |
| golang.org/x/time/rate | https://pkg.go.dev/golang.org/x/time/rate |
| golang.org/x/sync/singleflight | https://pkg.go.dev/golang.org/x/sync/singleflight |
| sony/gobreaker | https://github.com/sony/gobreaker |
| pressly/goose | https://github.com/pressly/goose |
| robfig/cron | https://github.com/robfig/cron |
| Claude API (Anthropic) | https://docs.anthropic.com |
| Claude Agent SDK (서브에이전트) | https://docs.anthropic.com/en/api/agent-sdk |
| ~~OpenAI / Google AI~~ | v1.5에서 멀티벤더 폐기 — Claude 단독 운영 (ARCHITECTURE 3-4) |

---

## 변경 이력

| 버전 | 날짜 | 주요 변경 |
|------|------|-----------|
| v1.5 | 2026-06-10 | 멀티벤더(OpenAI/Google) 의존 제거 → Claude 단독 운영. 단일 벤더 SPOF 대응(장애 시 신규매매 중단·청산 fail-open) 명시. 참고 링크에서 OpenAI/Google 삭제 |
| v1.5.1 | 2026-06-11 | **12-1 Claude 단일 벤더(SPOF) 장애 대응 절차 상세화**: 장애 분류(부분/전면/인증/레이트리밋), 감지(서킷·백오프·헬스프로브), 강등 매트릭스(NORMAL↔AI_HALT), 대응 런북, 구현 체크리스트(월 1회 SPOF 드릴 포함) |
| v1.3 | 2026-03-12 | KRX 이상거래 감시 대응(12-2), 소명 Audit Log 규격(12-3), 법적 주의사항(12-4), 신규 라이브러리 참고 링크 추가 |
| v1.0 | 2026-03-09 | 초기 설계 완료. 전체 아키텍처, 3-AI 합의(v1.5에서 Claude 서브에이전트 합의로 전환), 4-Layer Guard, 3-Tier 전략, Tier별 분석 관점(장기=구조/Moat, 중기=촉매/수급), 데이터 체이닝, DB 스키마(KRW int64), VerseKit 리포트(일간/월간/연간), 관측성, 운영 수칙 |

---

*Last Updated: 2026-06-11*
*Project: AlphaRadar (KORUS)*
