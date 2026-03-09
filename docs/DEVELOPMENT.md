# AlphaRadar — Go 코드 원칙 / 디버깅 (10~11장)

> 원본: CLAUDE.md 10~11장에서 분리
> 관련 문서: [ARCHITECTURE.md](ARCHITECTURE.md), [OPERATIONS.md](OPERATIONS.md)

---

## 10. 공통 Go 코드 원칙

```
코드 품질:
- exported 함수/타입 godoc 주석
- fmt.Errorf("context: %w", err) 에러 래핑
- defer로 리소스 정리
- 첫 인자: ctx context.Context

동시성:
- WebSocket → channel → 주문 (producer-consumer)
- 공유 상태: sync.Mutex 또는 channel
- go vet, go test -race 필수

보안:
- .env → godotenv, 민감 정보 로그 금지, .gitignore에 .env

숫자 정밀도:
- KRW 금액: int64 (원 단위 정수). DB 스키마 price/amount 컬럼도 INTEGER.
- 수익률/비율: float64. 표시 시 소수점 2자리 반올림.
- 필요 시 shopspring/decimal 도입 검토 (복리 계산 등).

로깅: slog (JSON, DEBUG/INFO/WARN/ERROR)
- 민감 데이터 자동 마스킹: API 키, 계좌번호, 토큰 → "****1234" 형태
- slog 커스텀 핸들러로 known 패턴(KIS_APP_KEY, ACCOUNT_NO 등) 자동 치환
- 로그 레벨: 주문 체결=INFO, Guard 차단=WARN, 시스템 장애=ERROR

테스트: *_test.go, mock, go test -race
배포: Docker 멀티스테이지, CGO_ENABLED=0
Rate Limit: uber-go/ratelimit (KIS TPS 등 config.yaml 참조)
Graceful Shutdown: os.Signal → 미체결 정리 → 상태 저장 → 연결 종료

관측성 (Observability):
- 초기: slog 구조화 로그 + Slack 알림으로 핵심 지표 추적
- 핵심 메트릭: API 응답시간, Guard 차단율, 시그널 적중률, 체결 지연시간, 일일 거래 수
- 이후: Prometheus exporter + Grafana 대시보드 전환 가능 (구조화 로그 기반)
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
