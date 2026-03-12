# AlphaRadar — Go 코드 원칙 / 디버깅 (10~11장)

> 원본: CLAUDE.md 10~11장에서 분리
> 관련 문서: [ARCHITECTURE.md](ARCHITECTURE.md), [OPERATIONS.md](OPERATIONS.md)

---

## 10. 기술 스택 & Go 코드 원칙

### 10-1. 기술 스택 결정표

> 전체 스택을 갈아엎지 않는다. MVP → 운영형 자동매매로 단계적 승격.
> 먼저 "범위"를 확정하고, 범위에 맞는 스택만 선택한다.

---

**[범위 결정 — 먼저 읽어라]**

```
최소 운영형 (현재 목표)
  Go + KIS REST/WS + SQLite + 알림
  → 자동매매 엔진만. 전략 설계 UI 없음. 백테스트는 Go 내장.

확장형 (운영 검증 후)
  + PostgreSQL + Redis + Prometheus/Grafana
  → 복수 프로세스, 감사 로그 대용량, 실시간 모니터링 필요 시점

제품형 (범위 확장 결정 시, 현재 보류)
  + Python 백테스트 계층 + Next.js 전략 빌더 UI + FastAPI
  → KIS 공식 저장소(strategy_builder + backtester) 수준까지 따라갈 때
  ※ 지금 이 수준까지 가면 스택이 Go + Python + JS 3개 언어로 늘어남. 신중히 결정.
```

---

**[필수] 지금 확정. 변경 없음 또는 즉시 적용**

| 항목 | 라이브러리 | 이유 |
|------|-----------|------|
| 언어 | Go 1.22+ | 동시성·성능·단일 바이너리. 최적 |
| WebSocket | gorilla/websocket | KIS 실시간 시세. 검증된 라이브러리 |
| TPS/Rate Limit | **golang.org/x/time/rate** | uber-go/ratelimit 교체. Token Bucket 직접 제어. 주문용(3초)·조회용(0.5초) 인스턴스 분리 |
| 로깅 | **uber-go/zap** | slog 교체. KRX 소명용 초단위 Audit Log. zero-allocation → 매매 지연 없음 |
| 스케줄링 | robfig/cron + 상주 프로세스 | 장중 상주 필수. GitHub Actions 불가 |
| 기술지표 | go-talib | 순수 Go 계산. AI 불필요 |
| Circuit Breaker | **sony/gobreaker** | 7-6-4 CLOSED→OPEN→HALF-OPEN 라이브러리 처리 |
| 동시 갱신 방지 | **golang.org/x/sync/singleflight** | 토큰·WS 접속키 동시 갱신 방지 |
| DB Migration | **pressly/goose** | 스키마 변경 누적 관리. 전환 시 재작성 불필요 |
| DB | SQLite + WAL | MVP. 단일 파일, 설치 불필요 |
| 알림 | Slack / Telegram | 유지 |
| 배포 | Docker 멀티스테이지 | CGO_ENABLED=0 |
| 환경변수 | godotenv | 유지 |
| 숫자 정밀도 | int64 (KRW) + float64 (비율) | 유지. shopspring/decimal은 복리 계산 시 검토 |

---

**[선택] 운영 단계 진입 시 추가. 지금은 과함**

| 항목 | 라이브러리 | 추가 시점 |
|------|-----------|-----------|
| DB 승격 | PostgreSQL | 4~6단계. 초단위 감사 로그 대용량 쓰기, 복수 프로세스 동시 쓰기 시 SQLite 한계 도달 |
| 캐시/상태 공유 | Redis | 복수 프로세스 간 주문 상태·idempotency·rate limit 상태 공유가 필요해질 때 |
| 모니터링 | Prometheus + Grafana | 운영 안정화 후. zap 로그 → Prometheus bridge로 전환 |
| 숫자 정밀도 | shopspring/decimal | 복리 계산·포트폴리오 집계 등 float64 오차가 문제될 때 |

---

**[보류] 범위 확장 결정 전까지 도입 안 함**

| 항목 | 이유 |
|------|------|
| Python 백테스트 계층 (QuantConnect Lean) | Go 내장 백테스터(7-5)로 충분. Lean은 Docker + C# 백엔드 추가됨 |
| Next.js 전략 빌더 UI | 제품형 범위. 현재 목표는 엔진 |
| FastAPI 전략 API 서버 | 위와 동일 |
| NATS / Redis Streams / Kafka | 단일 프로세스 goroutine + channel로 충분 |
| .kis.yaml 전략 포맷 | KIS 공식 포맷. 우리는 config.yaml + DB 구조로 대체 |

---

**[KIS 공식 저장소 활용 방침]**

```
공식 저장소(koreainvestment/open-trading-api)는 Go 코드 없음 — Python 전용.
활용 방법:
  1. examples_llm/ → API 파라미터·헤더·응답 구조 확인 (Go struct 정의 레퍼런스)
  2. postman/ → 요청/응답 JSON 샘플 (Go 파싱 테스트용)
  3. kis_auth.py → OAuth2 + WS 접속키 인증 흐름 확인
  4. strategy_builder/, backtester/ → 참고만. 직접 포팅 안 함.

커뮤니티 Go 클라이언트 (구현 참고용):
  - github.com/gobenpark/kinvest-go  ← 완성도 가장 높음
  - github.com/suapapa/go_kinvest    ← 국내주식 주문 경량 구현
  ※ 직접 사용보다는 구현 패턴 참고용. AlphaRadar는 자체 클라이언트 직접 구현.
```

---

### 10-2. Goroutine 레이어 아키텍처

> 각 레이어를 독립 Goroutine으로 분리, Channel로 연결. 한 곳 병목이 전체 매매 타이밍에 영향 없음.

```
[Provider Layer] — 수집 Goroutine들
  G1: KIS WebSocket → 실시간 체결가/호가 수신 → pricesCh
  G2: 뉴스 수집기   → 뉴스 이벤트 감지       → newsCh
  G3: KRX 수집기    → 장전 수급/특이 지표     → krxCh

[Strategy Layer] — 판단 Goroutine
  G4: pricesCh + newsCh + krxCh 수신
      → 기술분석 + AI 합의 + 이벤트 매매 판단
      → TechnicalSignal / EventSignal 생성 → signalCh

[Execution Layer] — 주문 Goroutine
  G5: signalCh 수신
      → Guard 4-Layer 검증
      → golang.org/x/time/rate (TPS 제어)
      → KIS 주문 API 호출
      → orderResultCh

[Audit Layer] — 로깅 Goroutine (매매 로직과 완전 분리)
  G6: 모든 레이어 이벤트 → zap JSON 로그 비동기 기록
      → SQLite/PostgreSQL audit 테이블 저장
      → Slack/Telegram 알림 발송

채널 설계:
  pricesCh   chan PriceEvent    // buffered, 크기 1000
  newsCh     chan NewsEvent     // buffered, 크기 100
  krxCh      chan KRXSignal     // buffered, 크기 200
  signalCh   chan Signal        // buffered, 크기 50
  orderResultCh chan OrderResult // buffered, 크기 100
```

---

### 10-3. 공통 Go 코드 원칙

```
코드 품질:
- exported 함수/타입 godoc 주석
- fmt.Errorf("context: %w", err) 에러 래핑
- defer로 리소스 정리
- 첫 인자: ctx context.Context

동시성:
- 레이어 간 통신: channel (위 10-2 참조)
- 레이어 내부 공유 상태: sync.RWMutex (읽기 다수/쓰기 소수)
- 토큰·WS접속키 캐시: sync.RWMutex + singleflight
- go vet, go test -race 필수

보안:
- .env → godotenv, 민감 정보 로그 금지, .gitignore에 .env
- zap 커스텀 핸들러로 known 패턴(KIS_APP_KEY, ACCOUNT_NO, token) 자동 마스킹

숫자 정밀도:
- KRW 금액: int64 (원 단위 정수). DB price/amount 컬럼도 INTEGER.
- 수익률/비율: float64. 표시 시 소수점 2자리 반올림.
- 필요 시 shopspring/decimal 도입 검토 (복리 계산 등).

로깅: uber-go/zap (JSON, DEBUG/INFO/WARN/ERROR)
- 로그 레벨: 주문 체결=INFO, Guard 차단=WARN, 시스템 장애=ERROR
- KRX 소명용 Audit Log: 주문 시점 시세 스냅샷 + 전략 판단 근거 + 취소 사유 포함
- 비동기 기록 (zap의 WriteSyncer를 별도 Goroutine으로 분리) → 매매 지연 없음

테스트: *_test.go, mock, go test -race
배포: Docker 멀티스테이지, CGO_ENABLED=0
Graceful Shutdown: SIGINT/SIGTERM → 신규 주문 중단 → 미체결 취소 → 채널 drain → DB flush → 종료

관측성:
- MVP: zap 구조화 로그 + Slack 알림
- 운영: Prometheus exporter + Grafana 대시보드 전환 (zap → Prometheus bridge)
- 핵심 메트릭: API 응답시간, Guard 차단율, 시그널 적중률, 체결 지연시간, 일일 거래 수
```

---

## 11. 디버깅 / 코드리뷰 프롬프트

```
[디버깅]
역할: Go 디버깅 전문가
입력: 코드, 에러, 스택트레이스
출력: 원인 설명 + 수정 코드 + 재발 방지

[코드리뷰]
역할: 시니어 Go 개발자 + 퀀트 시스템 전문가
입력: 코드
리뷰: race condition, 성능, 보안, Go 관용어, 트레이딩 엣지케이스
출력: 심각도(Critical/High/Medium/Low) + 개선 코드
```

---
