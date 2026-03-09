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
│   ├── briefing/              # 0단계: 모닝 브리핑
│   ├── krx/                   # 1단계: KRX 수집
│   ├── usmarket/              # 2단계: 미장 수집
│   ├── ai/                    # AI 엔진 추상화 계층
│   │   ├── provider.go        # AIProvider 인터페이스
│   │   ├── claude.go          # Claude (Anthropic) 클라이언트
│   │   ├── gemini.go          # Gemini (Google) 클라이언트
│   │   ├── openai.go          # ChatGPT (OpenAI) 클라이언트
│   │   └── consensus.go       # 3-AI 합의 엔진
│   ├── analysis/              # 3단계: 통합 분석 (AI 합의 결과 소비)
│   ├── technical/             # 4단계: 기술적 분석 (Go 직접 계산, AI 불필요)
│   ├── backtest/              # 5단계: 백테스팅
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
거시지표 수집 ──┼→ 3-AI 합의 분석 → 시그널 → Guard 계층 → 주문 실행 → 알림
KRX 수집 ──────┤   ├─ Claude       (go-talib)  ↓           ↓         ↓
US 수집 ───────┘   ├─ Gemini             PASS/BLOCK   체결 모니터링  리포트
                   └─ ChatGPT               ↓
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

### 3-4. 3-AI 합의 아키텍처

```
[AI 엔진]
┌─────────────────────────────────────────────────────────┐
│                    AIProvider 인터페이스                   │
│  Analyze(ctx, prompt, data) → AnalysisResult            │
├──────────┬──────────┬──────────────────────────────────┤
│  Claude  │  Gemini  │  ChatGPT                          │
│  Sonnet  │  2.5 Pro │  GPT-4o                           │
│  (1차)   │  (2차)   │  (3차)                            │
└──────────┴──────────┴──────────────────────────────────┘
         ↓           ↓           ↓
    ┌─────────────────────────────────────┐
    │       ConsensusEngine                │
    │  3개 응답 비교 → 투표 → 최종 판정    │
    └─────────────────────────────────────┘

[합의 모드]
- majority (기본):    2/3 다수결. 시그널 방향(BUY/SELL/HOLD) 기준.
- unanimous:          3/3 만장일치만 실행. 가장 보수적.
- primary_with_check: Claude 메인 + 나머지 1개 이상 동의 시 실행.

[태스크별 AI 활용]
태스크                     방식                        근거
─────────────────────────────────────────────────────────
시그널 생성 (매수/매도)    3-AI 합의 (majority)        안전 최우선. 단일 AI 환각 방지.
모닝 브리핑               Primary(Claude) + 1개 검증   비용 절감. 브리핑은 참고용.
뉴스 감성 분석            3-AI 병렬 → 평균             감성 판단은 AI마다 편차 큼.
기술적 분석               Go 코드 직접 계산            AI 불필요. 수학적 지표.
섹터/종목 추천            3-AI 합의 (majority)        종목 선정은 AI 편향 위험 큼.

[합의 프로세스]
1. 동일 프롬프트 + 동일 데이터를 3개 AI에 병렬 전송 (goroutine)
2. 각 AI 응답을 동일 JSON 스키마로 정규화 (AIProvider.Analyze)
3. ConsensusEngine이 응답 비교:
   - signal 방향: BUY/SELL/HOLD 투표 (2/3 이상 → 채택)
   - strength: 3개 평균값 (소수점 반올림)
   - confidence: 합의 여부에 따라 조정
     - 3/3 일치 → confidence +1
     - 2/3 일치 → confidence 유지
     - 1/3 또는 0/3 → HOLD 강제 (판단 보류)
   - reason: 합의된 측 AI들의 reason 병합
4. 합의 결과 + 개별 AI 응답 모두 DB 저장 (추후 정확도 분석)

[Fallback 전략]
→ 상세 장애 대응은 12장 참조. 아래는 요약:
- AI 1개 장애 → 2개 합의 / 2개 장애 → SOFT_WARN / 3개 장애 → skip + 알림

[비용 최적화]
- 시그널 생성: 하루 10~20회 호출 × 3 AI ≈ 30~60회/일
- 브리핑: 1회/일 × 2 AI = 2회/일
- 예상 일일 비용: 약 $1~3/일 (모델별 가격 변동)
- 모의투자 기간: 비용 절감을 위해 primary_with_check 모드 권장
```

### 3-5. 환경변수 (.env.example)

```env
# KIS API
KIS_APP_KEY=
KIS_APP_SECRET=
KIS_ACCOUNT_NO=
IS_PAPER_TRADING=true

# AI (3-AI 합의 시스템)
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
GOOGLE_AI_API_KEY=
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
  half_days: ["2026-01-26", "2026-09-15"]  # 반장일 (설/추석 전일 등)

kis:
  tps: 2                            # KIS REST API 초당 호출 한도

ai:
  consensus_mode: "majority"        # majority | unanimous | primary_with_check
  timeout: "30s"
  models:
    claude: "claude-sonnet-4-6"
    gemini: "gemini-2.5-pro"
    openai: "gpt-4o"

# ※ 이 파일은 Git 추적 대상 (시크릿 없음). .env는 .gitignore.
```

> **config.yaml 실제 구현 시**: 6-2 리스크 상수와 6-3 Tier 파라미터의 모든 값을 YAML 키로 풀어서 작성.
> 문서에서는 중복 방지를 위해 6-2/6-3을 원본으로 하고 여기서는 구조만 표시.

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
08:00       일일 리포트 발송 (전일 결과)        → 8장 참조
08:30       KRX 장전 특이 지표 수집
08:50       오늘의 매매 후보 종목 발송
09:05       장 시작 → WebSocket 모니터링         상주 프로세스
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
4. API별 Rate Limit 준수 (uber-go/ratelimit)
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
```

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
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);

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
    consensus_id INTEGER,            -- ai_consensus.id (3-AI 합의 결과 참조)
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

CREATE TABLE ai_responses (
    id INTEGER PRIMARY KEY,
    task_type TEXT NOT NULL,         -- SIGNAL, BRIEFING, NEWS_SENTIMENT, SECTOR
    provider TEXT NOT NULL,          -- CLAUDE, GEMINI, CHATGPT
    prompt_hash TEXT NOT NULL,       -- 동일 프롬프트 그룹핑용
    response_json TEXT NOT NULL,     -- AI 원본 응답
    signal_direction TEXT,           -- BUY, SELL, HOLD (시그널 태스크 시)
    strength INTEGER,
    confidence TEXT,
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
    final_confidence TEXT,
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
```

### 5-4. 보관 주기

| 데이터 | 보관 주기 | 근거 |
|--------|-----------|------|
| 주문/체결 내역 | **영구** | 세금 신고(5년), 감사 추적. 용량 미미 |
| 포지션 이력 | **영구** | 주문과 세트 |
| 일일 스냅샷 | **영구** | 성과 추적, 수익 곡선. 일 1건 |
| 시그널 이력 | **2년** | 전략 정확도 분석. 이후 월별 집계만 보관 |
| AI 응답/합의 이력 | **1년** | AI별 정확도 비교 분석. 이후 월별 집계만 보관 |
| Guard 차단 로그 | **1년** | Guard 규칙 정확도 분석. 이후 월별 집계만 보관 |
| 거시지표 (일별) | **5년** | 백테스팅 재현용 |
| KRX/US 수집 데이터 (clean) | **5년** | 백테스팅 재현용 |
| raw JSON 파일 | **3개월** | 디버깅/재처리용 |
| 실시간 체결가 (WebSocket) | **7일** | 당일 모니터링용. 장기는 일봉으로 대체 |
| 브리핑 원문 | **1년** | 참고용 |
| 로그 (slog) | **30일** | 장애 분석용 |
| 월간/연간 리포트 | **영구** | 전략 개선 참고 |

**자동 정리:** 매일 15:30 장 마감 정산 시 retention 정책에 따라 오래된 데이터 삭제.

---
