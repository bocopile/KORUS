# AlphaRadar — 아키텍처 / 스케줄링 / 데이터 (3~5장)

> 원본: CLAUDE.md 3~5장에서 분리
> 관련 문서: [STRATEGY.md](STRATEGY.md), [AI_PROMPTS.md](AI_PROMPTS.md), [OPERATIONS.md](OPERATIONS.md)

---

## 3. 아키텍처

### 3-1. 디렉토리 구조

```
alpharadar/
├── cmd/main.go                # 엔트리포인트
├── internal/
│   ├── briefing/              # 0단계: 모닝 브리핑 (econ 학습카드 포함)
│   ├── econ/                  # 0단계 보강: 경제 기초공부/용어 학습카드 (독립 모듈, 브리핑에 주입)
│   ├── krx/                   # 1단계: KRX 수집
│   ├── usmarket/              # 2단계: 미장 수집
│   ├── ai/                    # Claude 엔진 + 서브에이전트 합의 계층
│   │   ├── client.go          # Claude (Anthropic) 클라이언트 (단일 벤더)
│   │   ├── agent.go           # SubAgent 인터페이스 (관점별 Claude 호출)
│   │   ├── fundamental.go     # 펀더멘털/공시/실적 관점 서브에이전트
│   │   ├── news.go            # 뉴스/매크로/센티먼트 관점 서브에이전트
│   │   ├── risk.go            # 리스크/손절 관점 서브에이전트 (거부권)
│   │   └── consensus.go       # 서브에이전트 합의 엔진 (2/3 다수결)
│   ├── analysis/              # 3단계: 통합 분석 (AI 합의 결과 소비)
│   ├── technical/             # 4단계: 기술적 분석 (Go 직접 계산, AI 불필요)
│   ├── backtest/              # 5단계: 백테스팅
│   ├── portfolio/             # 포트폴리오 구성 + 리밸런싱 엔진 (결정론, 3-8)
│   │   ├── target.go          # 목표비중 정책 (TargetPolicy, Σw=1 검증)
│   │   ├── snapshot.go        # 보유/잔고 스냅샷 (KIS 조회)
│   │   ├── valuation.go       # 평가금액 → 현재비중
│   │   ├── drift.go           # 드리프트 = 현재−목표, ±밴드 이탈 추출
│   │   ├── planner.go         # 리밸런싱 주문계획 (매도우선·현금/호가 제약)
│   │   ├── sizer.go           # 주문수량 산출 (단주·최소주문금액)
│   │   └── ledger.go          # 리밸런싱 원장 (멱등키)
│   ├── guard/                 # Guard 계층: 주문 전 안전장치
│   │   ├── engine.go          # GuardEngine (규칙 순차 평가)
│   │   ├── market.go          # 시장 환경 Guard
│   │   ├── account.go         # 계좌/리스크 Guard
│   │   ├── stock.go           # 종목 Guard
│   │   └── order.go           # 주문 직전 Guard
│   ├── trading/               # 6단계: KIS 자동매매
│   │   ├── auth.go            # 인증
│   │   ├── order.go           # 주문 실행
│   │   ├── monitor.go         # WebSocket 모니터링
│   │   └── safety.go          # 런타임 안전장치 (kill switch 등)
│   ├── report/                # 리포트 생성 (일간/월간/연간)
│   ├── notify/                # 알림 발송
│   ├── storage/               # DB 접근 계층
│   └── config/                # 설정 관리
├── templates/
│   └── reports/               # HTML 리포트 템플릿 (VerseKit)
│       ├── daily.html         # 일일 리포트
│       ├── monthly.html       # 월간 리포트
│       └── annual.html        # 연간 리포트
├── data/
│   ├── alpharadar.db          # SQLite 메인 DB
│   ├── raw/                   # 원천 수집 데이터 (날짜별 JSON)
│   ├── reports/               # 생성된 리포트 파일 (HTML/CSV/PNG)
│   └── briefings/             # 브리핑 원문
├── config.yaml                # 런타임 설정 (Guard 임계값, AI 모델, Tier 파라미터)
├── .env.example               # 시크릿 템플릿 (API 키)
├── .github/workflows/         # CI/테스트용
├── Dockerfile
├── go.mod
└── go.sum
```

### 3-2. 데이터 흐름

```
[수집]              [AI 분석]              [Guard]          [실행]         [보고]
뉴스 수집 ──────┐
거시지표 수집 ──┼→ Claude 서브에이전트 합의 → 시그널 → Guard 계층 → 주문 실행 → 알림
KRX 수집 ──────┤   ├─ 펀더멘털 관점   (go-talib)  ↓           ↓         ↓
US 수집 ───────┘   ├─ 뉴스 관점            PASS/BLOCK   체결 모니터링  리포트
                   └─ 리스크 관점(거부권)     ↓
                      ↓                차단 시 로그 + 알림
                   투표/합의                ↓
                   (2/3 다수결)        런타임 안전장치
                                    (kill switch, reconciliation)

Guard 계층 상세:
  시그널 → [시장환경 Guard] → [계좌/리스크 Guard] → [종목 Guard] → [주문직전 Guard] → 주문 실행
           하나라도 HARD BLOCK → 주문 차단 + 사유 로그
           SOFT WARN → 경고 알림 (주문은 진행)
```

### 3-3. 인프라 단계별 전환

```
[개발/0단계]  개인 Mac + Claude Cowork
[1~3단계]     개인 Mac + Go cron (아침에 PC 켜져 있으면 됨)
[4~6단계]     Linux 서버(VPS 또는 미니PC) + systemd (장중 상주 필수)
```

### 3-4. Claude 서브에이전트 합의 아키텍처

> **v1.5 변경**: 멀티벤더(Claude+Gemini+ChatGPT) 합의를 폐기하고 **Claude 단독 + 관점 렌즈형 서브에이전트 합의**로 전환.
> 벤더 다양성 대신 **관점 다양성**으로 단일 모델의 환각·편향을 억제한다. (Claude Code Agent/서브에이전트 메커니즘 활용)

```
[AI 엔진]
┌─────────────────────────────────────────────────────────┐
│              SubAgent 인터페이스 (동일 Claude 모델)        │
│  Analyze(ctx, lens, prompt, data) → AnalysisResult       │
├──────────────┬──────────────┬───────────────────────────┤
│ 펀더멘털 관점 │   뉴스 관점   │  리스크 관점               │
│ 공시/실적/밸류│ 매크로/센티  │  손절/변동성/집중도(거부권)│
└──────────────┴──────────────┴───────────────────────────┘
         ↓             ↓             ↓
    ┌─────────────────────────────────────┐
    │       ConsensusEngine                │
    │  3개 관점 응답 비교 → 투표 → 최종 판정│
    └─────────────────────────────────────┘

[관점 렌즈 (고정 역할 — 각 관점 = 독립 컨텍스트 Claude 서브에이전트)]
- fundamental: 펀더멘털/공시/실적/밸류에이션 관점 → BUY|SELL|HOLD + reason
- news:        뉴스/매크로/수급/센티먼트 관점
- risk:        리스크/손절/변동성/집중도 관점 (보수 편향, 거부권 성격)
→ 세 관점 방향이 모두 일치할 때 confidence_score 최고점.
→ 서브에이전트는 서로의 응답을 참조하지 않고 병렬 실행(관점 오염 방지).
   (기존 'Swarm 관점 필드' 설계를 단일 스키마 다관점 → 관점별 서브에이전트로 승격)

[합의 모드]
- majority (기본):    2/3 다수결. 시그널 방향(BUY/SELL/HOLD) 기준.
- unanimous:          3/3 만장일치만 실행. 가장 보수적.
- primary_with_check: 펀더멘털 관점 메인 + 나머지 1개 이상 동의 시 실행.

[태스크별 활용]
태스크                     방식                                근거
─────────────────────────────────────────────────────────────
시그널 생성 (매수/매도)    3관점 서브에이전트 합의 (majority)   안전 최우선. 단일 관점 편향 방지.
모닝 브리핑               단일 Claude (관점 분리 불필요)        비용 절감. 브리핑은 참고용.
뉴스 감성 분석            뉴스 관점 서브에이전트 단독           감성은 뉴스 렌즈가 전담.
기술적 분석               Go 코드 직접 계산                    AI 불필요. 수학적 지표.
섹터/종목 추천            3관점 서브에이전트 합의 (majority)    종목 선정 편향 위험 큼.

[합의 프로세스]
1. 동일 프롬프트 + 동일 데이터를 3개 관점 서브에이전트에 병렬 전송 (goroutine, 각 독립 컨텍스트)
2. 각 응답을 동일 JSON 스키마로 정규화 (SubAgent.Analyze)
3. ConsensusEngine이 응답 비교:
   - signal 방향: BUY/SELL/HOLD 투표 (2/3 이상 → 채택)
   - strength: 3개 평균값 (소수점 반올림)
   - confidence_score: float64 0.0~1.0 로 표준화 (high/medium/low 레이블 사용 금지)
     - 3/3 일치 → 1.0
     - 2/3 일치 → 0.65 (의도적 설계: 다수결은 항상 SOFT_WARN → 포지션 50% 축소)
       ※ 다수결 합의를 만장일치보다 보수적으로 집행하는 것이 전략 의도입니다.
     - 1/3 또는 0/3 → 0.0, HOLD 강제 (판단 보류)
     실행 기준: ≥0.75 실행 가능 / 0.60~0.74 SOFT_WARN / <0.60 skip
   - **리스크 HARD veto**: risk 관점이 SELL/HOLD면 다수가 BUY여도 **신규 BUY는 HARD_BLOCK(실행 금지)**. (단일모델 상관오류 방어 — 청산/손절/추적은 영향 없음)
   - reason: 합의된 측 관점들의 reason 병합
4. 합의 결과 + 개별 관점 응답 모두 DB 저장 (관점별 정확도 분석)

[서브에이전트 장애 시 confidence_score 재계산 규칙]
정상(3관점): 위 표준 점수 적용.
1관점 장애(타임아웃/파싱실패) → 2관점 합의:
  2/2 일치  → confidence_score = 0.70 (SOFT_WARN, 포지션 50% 축소)
  1/2 불일치 → confidence_score = 0.0 → skip
  ※ **risk 관점이 장애인 경우**: HARD veto 검증 불가 → **신규 BUY는 skip(보수)**. 청산/손절은 진행(fail-open).
     (veto의 목적인 상관오류 방어가 정작 risk 다운 시 무력화되는 것을 차단)
2관점 장애 → 1관점만:
  confidence_score = 0.55 → 항상 skip (자동 실행 안 함)
  단, 보유 포지션 청산 시그널은 예외 허용 (fail-open 원칙)

[단일 벤더 리스크 — 멀티벤더 폐기의 트레이드오프]
- 벤더 다양성 상실 → Anthropic API 전면 장애 = 전체 시그널 중단(SPOF).
  대응: 장애 시 신규 자동매매 중단 + 보유 포지션 청산만 fail-open 허용 (12장 장애 대응).
- 동일 모델의 상관된 오류 가능 → 관점 프롬프트를 강하게 분리하고 risk 관점에 **HARD veto**(신규 BUY 차단)를 부여해 상쇄.
- 비용: 시그널 1건 = 3 서브에이전트 호출. 모의투자 기간은 primary_with_check 권장.

[비용 최적화]
- 시그널 생성: 하루 10~20회 × 3관점 ≈ 30~60 호출/일 (동일 Claude 모델)
- 브리핑: 1회/일 단일 호출
- 예상 일일 비용: 모델 가격에 따라 변동. 모의투자 기간은 primary_with_check 모드 권장.
```

### 3-5. 환경변수 (.env.example)

```env
# KIS API
KIS_APP_KEY=
KIS_APP_SECRET=
KIS_ACCOUNT_NO=
IS_PAPER_TRADING=true

# AI (Claude 단독 + 서브에이전트 합의)
ANTHROPIC_API_KEY=
AI_CONSENSUS_MODE=majority          # majority | unanimous | primary_with_check

# 알림
SLACK_WEBHOOK_URL=
TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=

# 데이터
FRED_API_KEY=
DB_PATH=./data/alpharadar.db
```

### 3-6. 런타임 설정 (config.yaml)

```yaml
# .env: 시크릿(API 키) → 환경변수
# config.yaml: 튜닝 가능한 파라미터 → 코드 수정 없이 조정
# ※ 값의 정의 원본은 6-2(리스크 상수) + 6-3(Tier별 상세). 여기는 런타임 로드용.

trading:
  paper_mode: true

# 아래 값들은 6-2 리스크 상수에서 정의. 코드는 이 파일에서 로드.
risk:       # → 6-2 참조
tiers:      # → 6-2, 6-3 참조
guard:      # → 6-2, 7-6-5 참조
  stale_threshold: "30m"
  max_daily_entries_per_tier: 2
  half_days: ["2026-01-26", "2026-09-15"]  # 단축 매매일 (설/추석 전일 등): 12:20 마감은 프로젝트 내부 정책 (KRX 공식 반장 제도 없음)

kis:
  tps: 2                            # KIS REST API 초당 호출 한도

ai:
  consensus_mode: "majority"        # majority | unanimous | primary_with_check
  timeout: "30s"
  model: "claude-sonnet-4-6"        # 단일 벤더. 배포 시 최신 안정 버전으로 갱신.
  subagents:                        # 관점 렌즈형 서브에이전트 (동일 모델, 역할만 분리)
    - fundamental                   # 펀더멘털/공시/실적
    - news                          # 뉴스/매크로/센티먼트
    - risk                          # 리스크/손절 (거부권)

# ※ 이 파일은 Git 추적 대상 (시크릿 없음). .env는 .gitignore.
```

> **config.yaml 실제 구현 시**: 6-2 리스크 상수와 6-3 Tier 파라미터의 모든 값을 YAML 키로 풀어서 작성.
> 문서에서는 중복 방지를 위해 6-2/6-3을 원본으로 하고 여기서는 구조만 표시.

### 3-7. 모듈 분리 설계 (모듈 카탈로그)

> 결정(2026-06): KORUS는 모놀리식이 아니라 **책임별 독립 모듈**로 구성한다.
> 코어 도메인은 단일 Go 바이너리 내부 패키지, 주변부(수집·브리핑)는 cron 잡으로 분리하는 **혼합형**.

| 모듈 | 책임 / 입·출력 | 실행주기 | 의존 | 로드맵 | 배치 |
|------|----------------|----------|------|--------|------|
| **US 수집기** | 미장 시세·지표·뉴스 → 정규화 store | 장마감 후 일배치 | 독립 | 2 | cron 잡 |
| **KR 수집기** | KRX 시세·공시·환율 → store | 장중+마감 | 독립 | 1 | cron 잡 |
| **경제학습/브리핑** | 개념DB+수집데이터 → 모닝브리핑·용어카드 | 매일 아침 | 수집기 | 0 | cron 잡 |
| **포트폴리오 구성기** | 유니버스 → 목표비중 후보 | 주간·이벤트 | 수집기+Claude | 3 | 코어 패키지 |
| **리밸런싱 엔진** | 목표 vs 보유 차분 → 주문계획 | 분기·밴드이탈 | 구성기+스냅샷 | 5 | 코어 패키지 |
| **주문/안전 게이트** | 계획 검증 → KIS 실행 → 원장 | 트리거 시 | 엔진+KIS | 6 | 코어 패키지 |
| **Claude 서브에이전트 오케스트레이터** | 3관점 합의·승인·감사로그 | 게이트 전단계 | 전 모듈 횡단 | 0·3·6 | 코어 패키지 |

→ 기술분석(4단계)은 별도 모듈이 아니라 구성기/엔진의 신호 입력으로 흡수.

```
[분리 방식 — 혼합형]
- 코어(구성기·리밸런싱·게이트·오케스트레이터) = 단일 Go 바이너리 내부 패키지
  → 트랜잭션·결정론·원자성 보장.
- 수집기(KR/US)·경제학습/브리핑 = cron 별도 잡 프로세스
  → 장애 격리 + 실행주기 독립.
- 근거: 현 단계에서 메시지큐/MSA는 과설계. 핵심 격리만 확보, 운영부담 최소화.
  확장형 단계(Redis/큐 도입)에서 재평가. → CLAUDE.md '시스템 범위 결정' 참조.

[프로세스 간 핸드오프 계약 — 혼합형 필수]
- 핸드오프 매체: 오직 DB(store) 경유. 프로세스 경계를 넘는 in-memory 채널 금지.
  · 실시간 경로(KIS WebSocket 체결/호가)만 코어 프로세스 내부 goroutine+채널(DEV 10-2). 경계 안 넘음.
  · 배치 수집(macro/news/krx/us/econ/briefing)은 별도 cron 잡이 DB write → 코어가 DB read.
- 단일 writer 규칙: 각 store 테이블은 지정된 1개 잡만 write(소유권). 코어는 read-only.
- 중복 실행 방지: 각 cron 잡은 job_runs 락 레코드(또는 파일락)로 동시 2회 실행 차단.
- 신선도 계약: 코어는 소비 전 data_freshness로 stale 검사(기대시각 +30분 초과 → stale).
  stale 기반 신규 진입 금지(fail-closed), 청산/손절은 진행(fail-open).
- 재시작 안전: 잡·코어 각자 idempotent. 코어 기동 시 미완료 주문은 reconciliation(OPERATIONS 12-1)으로 정합.
```

### 3-8. 포트폴리오·리밸런싱 엔진 (최우선 기능)

> 사용자 지정 최우선 기능. **AI 비결정성을 주문 경로에서 차단**하는 것이 핵심 원칙.
> 목표비중 산정·승인은 STRATEGY **6-4**(ApprovedTargetSnapshot), 밴드·트리거 상수는 6-2/6-3 참조 (Tier1 ±5%p, 분기). 리밸런싱은 Tier1 코어에만 적용(Tier2/3은 시그널).

```
[파이프라인 — internal/portfolio]
TargetPolicy → Snapshot → Valuation → Drift(±밴드)
  → Planner(매도우선·현금/호가 제약) → Sizer(단주·최소금액)
  → [Guard 게이트] → [Trading 실행] → Ledger(멱등키)

안전장치는 guard/ + trading/safety.go 재사용:
  가격이탈 중단 · 멱등키 · 킬스위치 · 부분체결 재시도 · DRY-RUN.
[Trading 실행]은 시그널 경로와 공유하는 단일 Execution goroutine(G5, DEV 10-2)으로 직렬화
  → 리밸런싱 매수/매도와 시그널 주문이 같은 rate limiter·계좌 락을 공유(이중 경로 충돌 방지).
```

[Claude 서브에이전트 역할 경계 — 합의 결과]

| 단계 | Claude 관여 | 결정 주체 |
|------|------------|-----------|
| 목표비중 후보 제안 | ✅ 3관점 제안·근거 | 정책/사람 승인 |
| 드리프트·수량 계산 | ❌ 차단 | 결정론 Go 코드 |
| 리밸런싱 실행 여부 | ✅ 합의 승인·감사로그 | Guard + 사람 |
| 주문 수량/호가/체결 | ❌ 차단 | 결정론 Go 코드 |

→ LLM은 "무엇을/왜"(목표·승인)까지, "얼마나/어떻게"(수량·주문)는 결정론 코드. 재현성·소명가능성 확보.

[참고 오픈소스 (2026-06 검증)]

| 저장소 | 활용 | 주의 |
|--------|------|------|
| zodia8393/etf-rebalancing-bot | ±밴드·멱등·가격가드·국내+해외 KIS 주문 패턴 1차 레퍼런스 | 코드 차용 전 라이선스 확인 |
| hyunyulhenry/quant_py | 목표−보유 차분 매매 + TWAP 분할 알고리즘 | 교재 코드, 운영 검증 필요 |
| koreainvestment/open-trading-api · kis-ai-extensions | KIS 인증·주문·잔고 인터페이스(공식) | 리밸런싱 스킬 미포함 → 직접 구현 |
| anthropics/financial-services | 리밸런싱/절세 프롬프트·체크리스트(Apache-2.0) | 실행엔진 아님, 프롬프트만 |
| tradermonty/claude-trading-skills | portfolio-manager 분석 스킬 구조(MIT) | Alpaca 기준, KIS 아님 |

미채택(참고 후보): Kabu Agent(Gemini·제안형), Stockelper(OpenAI·UI/로드맵), vibe-investing(KIS 미연동).

---

## 4. 스케줄링

### 4-1. 방식

```
[0단계]    Claude Cowork — 모닝 브리핑만. 코드 불필요.
[1단계~]   Go 내장 cron (robfig/cron) + systemd

Go 내장 cron 선택 근거:
- GitHub Actions: 최대 실행 6시간(무료), cron 최대 15분 지연 → 장중 상주 불가
- Go cron: 제약 없음, WebSocket 6시간 상주 가능, 단일 바이너리 배포
- GitHub Actions는 CI/테스트/배포 용도로만 사용
```

### 4-2. 일일 스케줄 (KST)

```
시간         작업                              비고
──────────────────────────────────────────────────────
06:00       미장 마감 데이터 수집
06:30       모닝 뉴스 수집 + AI 요약
07:00       모닝 브리핑 발송
08:00       일일 리포트 발송 (전일 결과)        → docs/REPORTS.md 참조
08:30       KRX 장전 특이 지표 수집
08:50       오늘의 매매 후보 종목 발송
09:05       장 시작 → WebSocket 모니터링         상주 프로세스

뉴스 수집 주기 (시간대별 차등):
  장전  06:00~08:50  10분 간격   (브리핑 준비, 가장 중요)
  장중  09:00~14:50  20분 간격   (장중 속보 대응)
  장후  15:00~20:00  60분 간격   (익일 참고)
  긴급  공시/속보     즉시        (KIS 실시간 이벤트 트리거)
14:50       장 마감 전 미체결 주문 정리
15:30       장 마감 데이터 저장 + 정산

매월 1일 08:00   월간 리포트 (전월)              일일 리포트 대체
매년 1/2 08:00   연간 리포트 (전년)              월간 리포트와 함께

※ 휴장일: KRX 휴장 캘린더 API 확인, 휴장이면 매매 관련 skip.
※ 시간대: KST (UTC+9, DST 없음).
```

### 4-3. Go cron 구현 예시

```go
loc, _ := time.LoadLocation("Asia/Seoul")
c := cron.New(cron.WithLocation(loc))

c.AddFunc("0 6 * * 1-5",   collectUSMarket)
c.AddFunc("30 6 * * 1-5",  collectNews)
c.AddFunc("0 7 * * 1-5",   sendBriefing)
c.AddFunc("0 8 * * 1-5",   sendDailyReport)   // 전일 결과
c.AddFunc("30 8 * * 1-5",  collectKRX)
c.AddFunc("50 8 * * 1-5",  sendCandidates)
c.AddFunc("5 9 * * 1-5",   startMonitoring)
c.AddFunc("50 14 * * 1-5", cleanupOrders)
c.AddFunc("30 15 * * 1-5", saveEndOfDay)
c.Start()

// ※ 태스크 충돌 방지: 각 태스크에 sync.Mutex 적용.
//    이전 태스크 미완료 시 다음 실행 skip + 경고 로그.
//    예: 06:30 뉴스 수집이 07:00까지 지연되면 브리핑은 수집 완료 후 실행.
```

### 4-4. systemd 서비스

```ini
# /etc/systemd/system/alpharadar.service
[Unit]
Description=AlphaRadar Trading System
After=network.target

[Service]
Type=simple
User=alpharadar
WorkingDirectory=/opt/alpharadar
ExecStart=/opt/alpharadar/alpharadar
Restart=always
RestartSec=10
EnvironmentFile=/opt/alpharadar/.env

[Install]
WantedBy=multi-user.target
```

---

## 5. 데이터

### 5-1. 수집처

**국내 시장 (KRX)**

| 용도 | 소스 | 비고 |
|------|------|------|
| 실시간 시세/주문 | KIS Open API | REST + WebSocket |
| 거래소 통계 | KRX Open API | 거래량, 수급, 시총 |
| 공시/재무 | OpenDART API | 전자공시, 재무제표 |
| 거시지표 | 한국은행 ECOS API | 금리, 환율, 경제지표 |
| 뉴스 | 네이버 금융, 연합인포맥스 | 크롤링 or RSS |

**미국 시장 (US)**

| 용도 | 소스 | 비고 |
|------|------|------|
| 지수/ETF/종목 시세 | Yahoo Finance / Alpha Vantage | 무료 tier |
| 거시지표 | FRED API | 무료 |
| 고용/물가 | BLS API | CPI, 실업률 |
| 기업 공시 | SEC EDGAR API | 10-K, 10-Q |
| Fear & Greed | CNN / alternative.me API | |

**수집 원칙:**
1. 원천(raw) 데이터 보존 — 가공 전 원본을 날짜별 JSON으로 저장
2. collected_at과 published_at 분리 관리
3. 거시지표 수정 발표 대비 revision_flag 포함
4. API별 Rate Limit 준수 (golang.org/x/time/rate — 주문용/조회용 인스턴스 분리)
5. 기대 시각 대비 30분 초과 시 stale 플래그

### 5-2. 저장 형태

```
SQLite (구조화 데이터)          JSON 파일 (원천 데이터)
├── orders                     data/raw/
├── positions                    ├── 2026-03-09/
├── signals                      │   ├── macro.json
├── daily_snapshots              │   ├── krx.json
├── macro_daily                  │   ├── us.json
├── briefings                    │   └── news.json
├── ai_responses                 └── ...
├── ai_consensus
└── guard_logs
                               data/briefings/
                                 ├── 2026-03-09.md
                                 └── ...
                               data/reports/
                                 ├── 2026-03-09-daily.html
                                 ├── 2026-03-monthly.html
                                 ├── 2025-annual.html
                                 └── ...
```

**SQLite 설정:**
```go
db.Exec("PRAGMA journal_mode=WAL")    // 읽기/쓰기 동시성 개선
db.Exec("PRAGMA busy_timeout=5000")
db.Exec("PRAGMA foreign_keys=ON")     // FK 무결성 강제 (감사 추적용)
```
> **FK 표기 규약**: 아래 스키마의 `-- FK lineage` 주석은 논리적 참조 관계다. 감사 추적(12-3)·재시작
> 복구가 걸린 테이블(orders, signals, rebalance_*, approved_target_snapshots)은 구현 시 실제
> `REFERENCES`(+인덱스)로 강제하고 `PRAGMA foreign_keys=ON`을 켠다. 수집/로그성 테이블은 논리 참조로 둔다.

### 5-3. DB 스키마

```sql
-- ※ KRW 금액 컬럼: INTEGER (원 단위). 수익률/비율만 REAL. (10장 숫자 정밀도 참조)
CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    idempotency_key TEXT UNIQUE NOT NULL,
    ticker TEXT NOT NULL,
    tier INTEGER NOT NULL,
    side TEXT NOT NULL,              -- BUY, SELL
    order_type TEXT NOT NULL,        -- MARKET, LIMIT
    quantity INTEGER NOT NULL,
    price INTEGER,                   -- 원 단위
    status TEXT NOT NULL,            -- PENDING, FILLED, PARTIAL, CANCELLED, FAILED
    filled_qty INTEGER DEFAULT 0,
    filled_price INTEGER,            -- 원 단위
    signal_id INTEGER,
    parent_order_id INTEGER,          -- 정정/취소 시 원주문 참조 (소명 Audit Log)
    cancel_reason TEXT,               -- 'STOP_LOSS' | 'TAKE_PROFIT' | 'GUARD_BLOCK' | 'MANUAL' | 'EXPIRY' | 'RECONCILE' | 'AMEND'
    context_json TEXT,                -- 주문 시점 스냅샷: { price, ask1, bid1, vix, spread_pct }
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);
-- FK lineage: orders.signal_id → signals.id → signals.consensus_id → ai_consensus.id

CREATE TABLE positions (
    id INTEGER PRIMARY KEY,
    ticker TEXT UNIQUE NOT NULL,
    name TEXT NOT NULL,
    tier INTEGER NOT NULL,
    quantity INTEGER NOT NULL,
    avg_price INTEGER NOT NULL,      -- 원 단위
    stop_loss INTEGER NOT NULL,      -- 원 단위
    target_price INTEGER NOT NULL,   -- 원 단위
    entry_date TEXT NOT NULL,
    updated_at TEXT NOT NULL
);

-- 포트폴리오 구성/리밸런싱 영속화 (STRATEGY 6-4, ARCHITECTURE 3-8). 소명 추적(12-3) 연동.
CREATE TABLE approved_target_snapshots (
    id INTEGER PRIMARY KEY,
    date TEXT NOT NULL,
    tier INTEGER NOT NULL,           -- 현재는 1(코어)만 리밸런싱 대상
    status TEXT NOT NULL,            -- PROPOSED, APPROVED, REJECTED, SUPERSEDED, EXPIRED (72h 미승인, STRATEGY 6-4)
    consensus_id INTEGER,            -- ai_consensus.id (목표비중 후보 합의 근거)
    approved_by TEXT,                -- 승인자 (사람/정책)
    approved_at TEXT,
    note TEXT,
    created_at TEXT NOT NULL
);

CREATE TABLE target_items (
    id INTEGER PRIMARY KEY,
    snapshot_id INTEGER NOT NULL,    -- approved_target_snapshots.id
    ticker TEXT NOT NULL,
    target_weight REAL NOT NULL,     -- 0.0~1.0 (Tier 배분 내 비중)
    UNIQUE(snapshot_id, ticker)
);

CREATE TABLE rebalance_runs (
    id INTEGER PRIMARY KEY,
    snapshot_id INTEGER NOT NULL,    -- 사용한 승인 스냅샷 (approved_target_snapshots.id)
    trigger TEXT NOT NULL,           -- QUARTERLY, BAND_BREACH, MANUAL
    status TEXT NOT NULL,            -- PLANNED, EXECUTING, DONE, ABORTED
    dry_run INTEGER DEFAULT 0,
    started_at TEXT NOT NULL,
    finished_at TEXT
);

CREATE TABLE rebalance_ledger (
    id INTEGER PRIMARY KEY,
    run_id INTEGER NOT NULL,         -- rebalance_runs.id
    idempotency_key TEXT UNIQUE NOT NULL,  -- date+ticker+side+round 해시 (DEV 10-4)
    ticker TEXT NOT NULL,
    side TEXT NOT NULL,              -- BUY, SELL
    planned_qty INTEGER NOT NULL,
    drift_before REAL,               -- 현재−목표 (실행 전)
    order_id INTEGER,                -- orders.id (실행 시 연결)
    created_at TEXT NOT NULL
);
-- FK lineage(리밸런싱): rebalance_ledger.run_id → rebalance_runs.id → approved_target_snapshots.id → ai_consensus.id
--                      rebalance_ledger.order_id → orders.id

CREATE TABLE portfolio_snapshots (
    date TEXT NOT NULL,
    ticker TEXT NOT NULL,
    eval_amount INTEGER NOT NULL,    -- 평가금액 (원)
    weight REAL NOT NULL,            -- 현재비중 (평가금액 기준)
    PRIMARY KEY(date, ticker)
);

CREATE TABLE daily_snapshots (
    date TEXT PRIMARY KEY,
    total_asset INTEGER NOT NULL,     -- 원 단위
    daily_pnl INTEGER NOT NULL,      -- 원 단위
    daily_pct REAL NOT NULL,         -- 비율 (%)
    cumulative_pnl INTEGER NOT NULL, -- 원 단위
    tier1_value INTEGER,             -- 원 단위
    tier2_value INTEGER,             -- 원 단위
    tier3_value INTEGER,             -- 원 단위
    cash INTEGER NOT NULL,           -- 원 단위
    positions_json TEXT
);

CREATE TABLE signals (
    id INTEGER PRIMARY KEY,
    ticker TEXT NOT NULL,
    tier INTEGER NOT NULL,
    signal TEXT NOT NULL,
    strength INTEGER NOT NULL,
    reason TEXT,
    entry_price INTEGER,             -- 원 단위
    stop_loss INTEGER,               -- 원 단위
    target_price INTEGER,            -- 원 단위
    consensus_id INTEGER,            -- ai_consensus.id (서브에이전트 합의 결과 참조). 기술분석(7-4) 단독 시그널은 NULL 허용
    news_ids TEXT,                   -- JSON 배열: 근거 뉴스 ID 목록 (소명 Audit Log 연동)
    created_at TEXT NOT NULL
);

CREATE TABLE macro_daily (
    date TEXT PRIMARY KEY,
    data_json TEXT NOT NULL,
    collected_at TEXT NOT NULL
);

CREATE TABLE briefings (
    date TEXT PRIMARY KEY,
    content TEXT NOT NULL,
    sent_at TEXT,
    status TEXT NOT NULL             -- DRAFT, SENT, FAILED
);

-- 경제 학습카드 (econ 모듈, AI_PROMPTS 7-0-6). 진도/복습 큐 관리.
CREATE TABLE econ_cards (
    card_id TEXT PRIMARY KEY,        -- econ-0042
    term TEXT NOT NULL,
    level TEXT NOT NULL,             -- L1, L2, L3
    status TEXT NOT NULL,            -- NEW, TAUGHT, UNDERSTOOD
    taught_date TEXT,                -- 최초 학습일
    next_review TEXT,                -- 다음 복습 예정일 (1·3·7·30일 간격)
    review_count INTEGER DEFAULT 0,
    card_json TEXT,                  -- 마지막 생성 카드 본문(JSON)
    updated_at TEXT NOT NULL
);

CREATE TABLE ai_responses (
    id INTEGER PRIMARY KEY,
    task_type TEXT NOT NULL,         -- SIGNAL, BRIEFING, NEWS_SENTIMENT, SECTOR
    agent_lens TEXT NOT NULL,        -- 관점: FUNDAMENTAL, NEWS, RISK (모두 동일 Claude 모델)
    prompt_hash TEXT NOT NULL,       -- 동일 프롬프트 그룹핑용
    response_json TEXT NOT NULL,     -- AI 원본 응답
    reason TEXT,                     -- 판단 근거 요약 (소명 Audit Log 12-3에서 조인 참조)
    signal_direction TEXT,           -- BUY, SELL, HOLD (시그널 태스크 시)
    strength INTEGER,
    confidence_score REAL,           -- 0.0~1.0 float
    latency_ms INTEGER,             -- 응답 시간
    cost_usd REAL,                  -- 호출 비용
    created_at TEXT NOT NULL
);

CREATE TABLE ai_consensus (
    id INTEGER PRIMARY KEY,
    task_type TEXT NOT NULL,
    prompt_hash TEXT NOT NULL,
    consensus_mode TEXT NOT NULL,    -- MAJORITY, UNANIMOUS, PRIMARY_WITH_CHECK
    vote_result TEXT NOT NULL,       -- "3/3 BUY", "2/3 SELL", "0/3 HOLD"
    final_signal TEXT NOT NULL,      -- BUY, SELL, HOLD
    final_strength INTEGER,
    final_confidence_score REAL,     -- 0.0~1.0 float
    dissent_reason TEXT,             -- 소수 의견 AI의 반대 근거
    response_ids TEXT NOT NULL,      -- ai_responses.id 목록 (JSON 배열)
    created_at TEXT NOT NULL
);

CREATE TABLE guard_logs (
    id INTEGER PRIMARY KEY,
    signal_id INTEGER,
    ticker TEXT NOT NULL,
    guard_layer TEXT NOT NULL,       -- MARKET, ACCOUNT, STOCK, ORDER
    rule_name TEXT NOT NULL,         -- 예: "vix_block", "position_limit"
    result TEXT NOT NULL,            -- PASS, HARD_BLOCK, SOFT_WARN
    reason TEXT,
    context_json TEXT,               -- 판단 시점 상태값 (VIX, 잔고, 비중 등)
    created_at TEXT NOT NULL
);

-- 시스템/AI 상태 전환 이력 (SPOF 런북 12-1 연동: AI_HALT/복구/장애 감사)
CREATE TABLE system_events (
    id INTEGER PRIMARY KEY,
    event_type TEXT NOT NULL,        -- STATE_CHANGE, AI_HALT, AI_RECOVER, DEGRADED, KIS_HALT, RECONCILE_FAIL, SPOF_DRILL
    component TEXT NOT NULL,          -- AI, KIS, WEBSOCKET, REBALANCE, GUARD
    from_state TEXT,                 -- NORMAL, DEGRADED, AI_HALT, RECOVERING
    to_state TEXT,                   -- NORMAL, DEGRADED, AI_HALT, RECOVERING
    severity TEXT NOT NULL,          -- INFO, WARN, ERROR, CRITICAL
    detail TEXT,                     -- 사유/지표: 연속실패횟수, HTTP status(401/429/5xx), 지속시간 등
    context_json TEXT,               -- 전환 시점 상태 스냅샷 (보유 포지션 수, 미체결 주문, 서킷 상태)
    alerted INTEGER DEFAULT 0,       -- 긴급 알림 발송 여부 (0/1)
    created_at TEXT NOT NULL
);

-- 혼합형 프로세스 핸드오프 (3-7): cron 잡 실행 이력·중복방지 + 데이터 신선도 계약.
CREATE TABLE job_runs (
    id INTEGER PRIMARY KEY,
    job_name TEXT NOT NULL,          -- COLLECT_KRX, COLLECT_US, COLLECT_NEWS, ECON, BRIEFING
    status TEXT NOT NULL,            -- RUNNING, DONE, FAILED
    lock_token TEXT,                 -- 동시 실행 차단용 토큰(또는 파일락 대체)
    started_at TEXT NOT NULL,
    finished_at TEXT,
    rows_written INTEGER,
    error TEXT
);

CREATE TABLE data_freshness (
    source TEXT PRIMARY KEY,          -- macro_daily, krx, us, news ...
    last_collected_at TEXT,           -- 마지막 수집 완료 시각 (단일 writer 잡이 갱신)
    expected_by TEXT,                 -- 기대 시각 (이후 +30분 초과 시 stale)
    is_stale INTEGER DEFAULT 0        -- 코어가 소비 전 검사
);
```

### 5-4. 보관 주기

| 데이터 | 보관 주기 | 근거 |
|--------|-----------|------|
| 주문/체결 내역 | **영구** | 세금 신고(5년), 감사 추적. 용량 미미 |
| 포지션 이력 | **영구** | 주문과 세트 |
| 승인 목표비중·리밸런싱 원장 (approved_target_snapshots/target_items/rebalance_runs/rebalance_ledger) | **영구** | 리밸런싱 감사·소명(12-3) |
| 포트폴리오 일별 스냅샷 (portfolio_snapshots) | **5년** | 비중 추적·백테스트 재현 |
| 일일 스냅샷 | **영구** | 성과 추적, 수익 곡선. 일 1건 |
| 시그널 이력 | **2년** | 전략 정확도 분석. 이후 월별 집계만 보관 |
| AI 응답/합의 이력 | **1년** | AI별 정확도 비교 분석. 이후 월별 집계만 보관 |
| Guard 차단 로그 | **1년** | Guard 규칙 정확도 분석. 이후 월별 집계만 보관 |
| 시스템/AI 상태 이벤트 (system_events) | **2년** | SPOF/장애 사후분석, 월 1회 SPOF 드릴 기록, 소명 보조 |
| cron 잡 실행 이력 (job_runs) | **90일** | 수집 잡 모니터링·중복방지·디버깅 |
| 거시지표 (일별) | **5년** | 백테스팅 재현용 |
| KRX/US 수집 데이터 (clean) | **5년** | 백테스팅 재현용 |
| raw JSON 파일 | **3개월** | 디버깅/재처리용 |
| 실시간 체결가 (WebSocket) | **7일** | 당일 모니터링용. 장기는 일봉으로 대체 |
| 브리핑 원문 | **1년** | 참고용 |
| 경제 학습카드 (econ_cards) | **영구** | 학습 진도·복습 큐 상태. 용량 미미 |
| 로그 (zap) | **30일** | 장애 분석용 |
| 월간/연간 리포트 | **영구** | 전략 개선 참고 |

**자동 정리:** 매일 15:30 장 마감 정산 시 retention 정책에 따라 오래된 데이터 삭제.

---
