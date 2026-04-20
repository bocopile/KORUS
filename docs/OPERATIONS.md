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

**장애 대응:**
- KIS API 장애 → Fail-Closed (신규 매매 자동 중단)
- AI 1개 장애 → 나머지 2개로 합의 진행 (2/2 일치 시 실행)
- AI 2개 장애 → confidence_score = 0.55 → skip (자동 실행 안 함) + 긴급 알림 (청산 시그널 예외 허용)
- AI 3개 장애 → 시그널 생성 skip + 브리핑 skip + 긴급 알림
- 네트워크 장애 → WebSocket 자동 재연결 + 실패 시 알림
- 데이터 불일치 → 매매 중단 + 수동 확인

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
| OpenAI API (ChatGPT) | https://platform.openai.com/docs |
| Google AI (Gemini) | https://ai.google.dev/docs |

---

## 변경 이력

| 버전 | 날짜 | 주요 변경 |
|------|------|-----------|
| v1.3 | 2026-03-12 | KRX 이상거래 감시 대응(12-2), 소명 Audit Log 규격(12-3), 법적 주의사항(12-4), 신규 라이브러리 참고 링크 추가 |
| v1.0 | 2026-03-09 | 초기 설계 완료. 전체 아키텍처, 3-AI 합의, 4-Layer Guard, 3-Tier 전략, Tier별 분석 관점(장기=구조/Moat, 중기=촉매/수급), 데이터 체이닝, DB 스키마(KRW int64), VerseKit 리포트(일간/월간/연간), 관측성, 운영 수칙 |

---

*Last Updated: 2026-03-09*
*Project: AlphaRadar (KORUS)*
