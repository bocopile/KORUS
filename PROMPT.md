# AlphaRadar (KORUS)

## AI 기반 한미 주식 자동매매 + 모닝 브리핑 시스템

> **프로젝트 목표**:
> 매일 아침 경제 뉴스·지표 브리핑을 자동으로 받고,
> 한국(KRX) + 미국(NYSE/NASDAQ) 시장 데이터를 수집·분석하여
> 3-Tier 포트폴리오 전략으로 AI 기반 자동매매를 구현한다.

---

## 1. 문서 안내

> **이 문서는 KORUS 프로젝트의 마스터 프롬프트이며, 설계 기준·구현 지침·운영 원칙을 통합 정의한다.**
> 규칙 충돌 시 우선순위: **운영/안전 규칙(12장) > Guard 규칙(7-6-5) > 전략 규칙(6장) > 프롬프트(7장)**

| 항목 | 내용 |
|------|------|
| 문서 목적 | 마스터 프롬프트: 설계 기준 + 구현 지침 + 운영 원칙 통합 |
| 대상 독자 | 개발자(본인), LLM(Claude/Gemini/ChatGPT — 코드 생성/분석 시 입력) |
| 사용 방식 | 전체 설계 참고용으로 읽되, LLM에 입력 시 해당 단계 블록만 발췌 사용 |

**문서 구성:**

| 층 | 장 | 성격 |
|---|---|---|
| 프로젝트 개요 | 1~2장 | 문서 안내, 프로젝트 개요/로드맵 |
| 아키텍처/설계 | 3~5장 | 디렉토리, 데이터 흐름, 3-AI, 스케줄링, 데이터/DB |
| 투자 전략 | 6장 | 3-Tier 배분, 리스크 상수, Tier별 진입/청산 조건 |
| LLM 실행 프롬프트 | 7장 | 단계별 AI 입력용 (발췌 사용). 3-AI 공통 프롬프트 |
| 리포트/알림 | 8~9장 | 일간/월간/연간 리포트, 알림 트리거 |
| 구현/운영 | 10~12장 | Go 코드 원칙, 디버깅, 운영 수칙/보안/장애 대응 |
| 부록 | 13장 | 참고 링크, 변경 이력 |

---

## 2. 프로젝트 개요

| 항목 | 내용 |
|------|------|
| 프로젝트명 | AlphaRadar (코드명: KORUS) |
| 개발 언어 | Go 1.22+ |
| 모듈명 | github.com/bhshin/alpharadar |
| 자동매매 API | 한국투자증권 KIS Developers (REST + WebSocket) |
| AI 엔진 | 3-AI 합의 시스템: Claude + Gemini + ChatGPT |
| 스케줄링 | Claude Cowork (0단계) → Go 내장 cron + systemd (이후) |
| 알림 채널 | Slack (1차) / 텔레그램 (2차) |
| 데이터 저장 | SQLite (초기) → PostgreSQL (운영 안정화 후) |
| 배포 환경 | Docker + Linux 서버 |

**자동매매 대상 범위:**
- **국내 주식**: KIS REST + WebSocket API를 통한 완전 자동매매
- **미국 주식/ETF**: 국내 상장 해외 ETF(TIGER 미국나스닥100 등)로 간접 투자 → 7단계에서 KIS 해외주식 API 직접 연동 예정

**개발 로드맵:**

```
0단계: 모닝 브리핑 시스템       ← 매일 아침 경제/뉴스 자동 발송
1단계: KRX 특이 지표 수집기     ← 한국 시장 이상 신호 감지
2단계: 미장(US) 지표 수집기     ← 미국 거시지표 수집
3단계: 한미 통합 분석 리포트    ← 3-AI 합의 기반 자동 분석
4단계: 기술적 분석 엔진         ← 매매 시그널 생성
5단계: 백테스팅 시스템          ← 전략 사전 검증
6단계: KIS 자동매매 연동        ← 실전 자동매매 (국내)
7단계: KIS 해외주식 연동        ← 미국 주식 자동매매 (예정)
```

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

## 6. 투자 전략 — 3-Tier 포트폴리오

### 6-1. 전체 자산 배분

```
총 투자금 100%

┌─────────────────────────────────────────┐
│  [40%] Tier1 — 장기 코어 포지션          │
│  국내 대형주 + ETF (해외 ETF 포함)        │
│  → 분기 리밸런싱                          │
├─────────────────────────────────────────┤
│  [40%] Tier2 — 중기 스윙 포지션           │
│  섹터 순환매 + 성장주                     │
│  → AlphaRadar 시그널 기반 (1~3개월)       │
├─────────────────────────────────────────┤
│  [20%] Tier3 — 단기 모멘텀 포지션         │
│  뉴스/수급 모멘텀 종목                    │
│  → AlphaRadar 단기 시그널 (1~10일)        │
└─────────────────────────────────────────┘
```

### 6-2. 공통 규칙

```
[리스크 상수 — 절대 변경 불가]
※ 이 상수들은 Guard 계층(7-6-5)에서 HARD_BLOCK으로 강제됩니다.
※ 값 정의는 여기 1곳. Guard는 이 값을 참조만 합니다.

MAX_TIER1_PCT   = 0.40    // Tier1 최대 40%
MAX_TIER2_PCT   = 0.40    // Tier2 최대 40%
MAX_TIER3_PCT   = 0.20    // Tier3 최대 20%
MAX_SINGLE_PCT  = 0.10    // 단일 종목 최대 10%
MAX_DAILY_LOSS  = -0.02   // 일일 손실 한도 -2%
VIX_BLOCK       = 35.0    // VIX 초과 시 매수 차단
VIX_WARN        = 30.0    // VIX 초과 시 Tier3 신규 진입 경고
TRADING_START   = "09:05"
TRADING_END     = "14:50"
MIN_TRADE_VALUE = 100_000_000   // 거래대금 최소 1억원 (소액 운용 기준, 향후 상향 가능)
SPREAD_WARN     = 0.02    // 호가 스프레드 2% 초과 시 경고
PRICE_DIVERGE   = 0.02    // 주문 가격 괴리 2% 초과 시 차단

[현금 비중]
- Tier2+Tier3 동시 최대 보유: Tier2 5개 + Tier3 4개 = 총 9개
- Tier2 포지션: 총 자산의 8% (5개 만석 시 40%)
- Tier3 포지션: 총 자산의 5% (4개 만석 시 20%)
- VIX > VIX_WARN(30): Tier3 신규 진입 금지 → 현금 비중 자동 확대

[Tier 간 우선순위]
1. Tier1 리밸런싱 (분기별) — 최우선
2. Tier2 진입/청산 — Tier1 완료 후
3. Tier3 진입/청산 — Tier1·2와 자금 충돌 없을 때만

[시그널 랭킹 (동시 다발 시)]
Strength 높은 순 → VolumeRatio 높은 순 → 시가총액 큰 순
동일 Tier 내 최대 2개/일 신규 진입

[매수 금지 필터]
→ Guard 계층(7-6-5)에서 강제 적용. 상세 규칙은 Guard Layer 3~4 참조.
```

### 6-3. Tier별 상세

| 항목 | Tier1 (장기) | Tier2 (중기) | Tier3 (단기) |
|------|-------------|-------------|-------------|
| 보유 기간 | 6개월~수년 | 1~3개월 | 1~10거래일 |
| 목표 수익 | 연 +15~30% | +15~25% | +5~10% |
| 손절 기준 | -15% | -7% | -3% |
| 매매 빈도 | 분기 1~2회 | 월 2~5회 | 주 2~5회 |
| 전략 | 분할매수 (DCA) | 섹터 순환매 + 실적 모멘텀 | 뉴스/수급 + 기술적 돌파 |
| 포지션 크기 | - | 총 자산 8% | 총 자산 5% |
| 최대 보유 | - | 5개 | 4개 |

**Tier1 종목 유니버스:**
```
국내 대형주: 삼성전자(005930), SK하이닉스(000660), NAVER(035420), 삼성바이오로직스
국내 ETF:   KODEX 200, KODEX 반도체, TIGER 미국나스닥100, TIGER 미국S&P500
※ 미국 직접 투자(QQQ, SPY)는 7단계 이후
```

**Tier1 리밸런싱 기준:**
```
트리거: 분기 말(3/6/9/12월) 또는 목표 비중 대비 ±5%p 이상 편차 발생 시
방법:   비중 초과 종목 일부 매도 → 비중 부족 종목 추가 매수
종목 교체: 3-AI 합의(majority)로 유니버스 재평가. 2/3 이상 "제외" 시 교체.
손절:   개별 종목 -15% 도달 시 AI 합의로 보유/손절 판단 (자동 강제 손절 아님)
```

**Tier2 진입 조건 (3개 이상):**
```
수급:  외국인 5일 연속 순매수 / 기관 동반 / 거래대금 20일 평균 150%↑
기술:  SMA20 위 거래 / RSI 45~65 / MACD 골든크로스
펀더:  영업이익 증가 / PER 합리적 / 실적 서프라이즈
추가:  실적발표 D-3 ~ D+1 신규 진입 금지
```

**Tier2 청산 조건 (1개 이상):**
```
손절:  -7% 도달 → 자동 매도 (예외 없음)
익절:  +20% 도달 → 50% 매도 + 잔량 추적 손절 (고점 대비 -5%에서 청산)
기간:  보유 3개월 초과 → AI 합의로 보유 연장/청산 판단
수급:  외국인 5일 연속 순매도 + 기관 동반 이탈
기술:  SMA20 하향 이탈 + RSI 30 미만 + MACD 데드크로스
펀더:  실적 미스(컨센서스 -10% 이상) 발생 시 즉시 청산 검토
```

**Tier3 진입 조건 (3개 이상):**
```
뉴스:  긍정적 공시 / 테마 이슈 부각
수급:  거래량 전일 대비 300%↑ / 외국인·기관 당일 순매수
기술:  52주 신고가 돌파 / 볼린저밴드 상단+거래량 / 갭상승 눌림목
규칙:  진입 즉시 -3% 손절 설정 / 목표가 50% 익절+추적 / 하루 최대 2종목
       당일 -2% 도달 시 매매 중단
```

**Tier3 청산 조건 (1개 이상):**
```
손절:  -3% 도달 → 자동 매도 (예외 없음)
익절:  +7% 도달 → 50% 매도 + 잔량 추적 손절 (고점 대비 -2%에서 청산)
기간:  보유 10거래일 초과 → 자동 청산 (장기 보유 금지)
모멘텀 이탈: 거래량 전일 대비 50% 미만으로 급감 시 청산 검토
당일:  장 종료 15분 전(14:35) 미익절 Tier3 → 익일 시초 청산 검토
```

**섹터 우선순위 (VIX 기반):**
```
Risk-On  (VIX < 20):   반도체 > AI/SW > 2차전지 > 바이오
Risk-Off (VIX 20~30):  방어주 > 배당주 > 음식료 > 유틸리티
Extreme  (VIX > 30):   현금 비중 확대, 신규 매수 자제
```

---

## 7. 단계별 프롬프트 (LLM 입력용)

> LLM(Claude/Gemini/ChatGPT)에 입력할 때 해당 단계 블록만 발췌하여 사용.
> 3-AI 합의가 필요한 태스크(7-3 등)는 동일 프롬프트를 3개 AI에 병렬 전송.

**데이터 체이닝 (Context Chaining):**
각 단계의 Output이 다음 단계의 Input으로 연결됩니다. `[Input from]`과 `[Output to]`로 흐름을 추적하세요.
```
7-0-1(거시지표) ──┐
7-0-2(뉴스)     ──┼→ 7-0-4(브리핑) → Slack
7-0-3(캘린더)   ──┘       ↓
                    7-1(KRX) ──┐
                    7-2(US)  ──┼→ 7-3(통합분석) → 7-4(기술분석) → signals → 7-6(자동매매)
                    7-0-2    ──┘                                    ↓
                    7-0-3    ──┘                               Guard(7-6-5)
```

**단계별 I/O 계약 표:**

| 단계 | Input | Output | 저장 위치 | 실패 시 |
|------|-------|--------|-----------|---------|
| 7-0-1 거시지표 | 외부 API | macro JSON | DB(macro_daily) + raw JSON | null 필드 + stale 플래그 |
| 7-0-2 뉴스 | 크롤링/RSS | news JSON[] (최대 10개) | raw JSON | 빈 배열 |
| 7-0-3 캘린더 | 외부 캘린더 | events JSON | raw JSON | 빈 배열 + 경고 |
| 7-0-4 브리핑 | 0-1 + 0-2 + 0-3 | Slack 마크다운 | DB(briefings) + .md 파일 | skip + 알림 |
| 7-1 KRX | KIS/KRX/DART API | KRXSignal[] | DB + raw JSON | stale + 알림 (fallback 금지) |
| 7-2 US | Yahoo/FRED/CNN | USMarketData | DB(macro_daily) + raw JSON | 캐시 fallback + IsStale |
| 7-3 통합분석 | 0-2 + 0-3 + 7-1 + 7-2 | 합의 JSON | DB(ai_responses, ai_consensus) | AI fallback (12장) |
| 7-4 기술분석 | 7-3 + KIS 일봉 | TechnicalSignal[] | DB(signals) | 데이터 부족 시 skip |
| 7-6 자동매매 | signals | 주문 체결 | DB(orders, positions) | Guard 차단 or 주문 실패 |

**공통 에러 행동 강령:**
모든 역할에 적용되는 실패 시 행동 원칙:
- 데이터 수집 실패 → 해당 필드 `null` 처리 + Slack 알림. **절대 임의 값 생성 금지.**
- API 타임아웃 → 3회 재시도 후 실패 확정. 후속 단계에 `stale_warning` 전달.
- AI 응답 파싱 실패 → 원본 저장 후 해당 AI 결과 제외, 나머지 AI로 합의 진행.
- AI JSON 스키마 검증: 디코딩 성공 후 필수 필드(signal, confidence, ticker) 존재+타입 검증. 누락 시 해당 AI 결과 무효.
- 치명적 오류 → 해당 세션 매수 로직 중단 + 긴급 알림. **매도/손절은 중단하지 않음.**

### 7-0. 모닝 브리핑 시스템

**0-1. 거시경제 지표 수집**

```
[역할] 글로벌 거시경제 전문 애널리스트
[Input from] 외부 API (FRED, Yahoo Finance, ECOS 등)
[Output to] 7-0-4(브리핑), 7-3(통합분석), DB(macro_daily)
[에러 시] 수집 실패 필드는 null. 전체 실패 시 전일 데이터 fallback + stale 플래그.

[수집 데이터]
국내: 코스피/코스닥 등락률, 원달러 환율, 국고채 금리, 외국인/기관 순매수
미국: S&P500/NASDAQ/DOW/Russell2000, VIX, 미국 국채 2/10년, DXY
원자재: WTI, 브렌트유, 금, 은
거시지표: 미국 CPI/PCE/실업률/GDP, 연준 금리, CME FedWatch 확률
아시아: 닛케이225, 상해종합, 항셍, 대만 가권

[출력 - JSON]
{
  "date": "YYYY-MM-DD",
  "collected_at": "YYYY-MM-DDTHH:MM:SS+09:00",
  "kr_market": {
    "kospi": float, "kospi_change_pct": float,
    "kosdaq": float, "kosdaq_change_pct": float,
    "usdkrw": float, "kr_bond_3y": float,
    "foreign_net_buy": float, "inst_net_buy": float
  },
  "us_market": {
    "sp500": float, "sp500_change_pct": float,
    "nasdaq": float, "nasdaq_change_pct": float,
    "dow": float, "dow_change_pct": float,
    "russell2000": float, "russell2000_change_pct": float,
    "vix": float, "dxy": float
  },
  "bonds": {"us10y": float, "us2y": float, "spread": float},
  "commodities": {"wti": float, "brent": float, "gold": float, "silver": float},
  "macro": {
    "cpi": float, "cpi_date": "YYYY-MM-DD",
    "pce": float, "pce_date": "YYYY-MM-DD",
    "unemployment": float, "gdp": float, "fed_rate": float
  },
  "fed_watch": {"hike": float, "cut": float, "hold": float},
  "asia": {"nikkei": float, "shanghai": float, "hangseng": float, "taiex": float}
}

[출력 규칙]
- 확정값만 사용. 추정값은 _est 접미사.
- 수집 불가 필드는 null (임의 값 금지).
- JSON 외 텍스트 출력 금지.
```

**0-2. 경제 뉴스 수집 + AI 요약**

```
[역할] 경제/금융 전문 기자이자 애널리스트
[Input from] 외부 뉴스 소스 (크롤링/RSS)
[Output to] 7-0-4(브리핑), 7-3(통합분석)
[에러 시] 뉴스 수집 불가 시 "뉴스 수집 실패" 명시. 빈 배열 반환. 루머/미확인 절대 포함 금지.

[소스] 국내: 한경, 매경, 연합인포맥스, 네이버 금융 / 해외: Reuters, Bloomberg, CNBC, WSJ

[분류] 1.중앙은행/금리 2.경제지표 3.기업실적/M&A 4.지정학 5.반도체/기술주 6.에너지/원자재 7.환율 8.국내 정책

[출력 - JSON 배열, 최대 10개]
{
  "category": "카테고리",
  "headline": "헤드라인 (30자 이내)",
  "summary": "핵심 2~3줄",
  "market_impact": "긍정 | 부정 | 중립",
  "affected_sectors": ["반도체", "금융"],
  "importance": "high | medium",
  "source": "출처명",
  "source_url": "URL (확인 가능 시)",
  "published_at": "YYYY-MM-DDTHH:MM"
}

[규칙] low 제외, 중복 제거, 루머 제외, JSON만 출력.
```

**0-3. 경제 이벤트 캘린더**

```
[역할] 경제 캘린더 전문가
[Input from] 외부 캘린더 소스
[Output to] 7-0-4(브리핑), 7-3(통합분석)
[에러 시] 캘린더 수집 불가 시 빈 배열 반환. "캘린더 미확인" 경고 포함.
[소스] Investing.com, FRED, 한국은행, 실적 발표 일정

[출력 - JSON]
{
  "today_events": [
    {
      "time_kst": "HH:MM",
      "event": "이벤트명",
      "country": "US | KR | EU | JP | CN",
      "forecast": "예상치 (미발표 시 null)",
      "previous": "이전치",
      "impact": 1~3,
      "affected_assets": ["채권", "달러", "주식"]
    }
  ],
  "this_week_highlights": ["핵심 이벤트 3개"]
}

[규칙] JSON만 출력.
```

**0-4. 모닝 브리핑 최종 리포트**

```
[역할] AI 모닝 브리핑 어시스턴트
[방식] Primary(Claude) + 1개 검증 (3-4 참조)
[Input from] 7-0-1(거시지표 JSON), 7-0-2(뉴스 JSON), 7-0-3(캘린더 JSON)
[Output to] Slack 발송, DB(briefings), data/briefings/{날짜}.md
[에러 시] 입력 데이터 일부 누락 시 해당 섹션 "N/A" 표시. 전체 실패 시 브리핑 skip + 알림.

[입력] 거시지표(0-1), 뉴스(0-2), 캘린더(0-3)
※ KRX 데이터는 1단계 구축 후 추가 예정.

[출력 - Slack 마크다운]
*AlphaRadar 모닝 브리핑* | {date} {요일}
━━━━━━━━━━━━━━━━━━━━
*글로벌 시장*
- S&P500 {등락} | NASDAQ {등락} | DOW {등락}
- VIX: {수치} ({레벨}) | DXY: {수치} | 원달러: {환율}원

*시장 분위기*
{Risk-On/Off 판단 + 근거 2~3줄}

*핵심 뉴스 Top5*
1~5. {헤드라인} [{영향}]

*경제 이벤트*
{시간} | {이벤트} | 예상 {값} | 이전 {값}

*국내 시장 주목*
- 코스피/코스닥 전망, 주목 섹터

*3-Tier 스탠스*
- T1: {유지/리밸런싱} | T2: {공격/보통/방어} | T3: {가능/자제/금지}

*리스크* {2~3가지}

*한 줄 전략* "{메시지}"
━━━━━━━━━━━━━━━━━━━━

[조건] 50줄 이내, 팩트 중심, 근거 부족 시 "판단 보류", 수집 실패 시 "N/A".
```

**0-5. Cowork 태스크 프롬프트**

```
[태스크명] AlphaRadar 모닝 브리핑
[실행 주기] 평일 매일 오전 7:00

[태스크 내용]
매일 아침 주식 투자를 위한 모닝 브리핑을 생성해줘.

1. 웹 검색으로 수집: 미국 증시 마감(S&P500, NASDAQ, DOW), VIX, 원달러 환율,
   경제 뉴스 5개(한국+미국), 오늘 경제 지표 발표 예정

2. 0-4 형식으로 브리핑 작성

3. Slack 웹훅으로 발송 (URL: {SLACK_WEBHOOK_URL})

4. ~/AlphaRadar/briefings/{날짜}-morning.md 파일로 저장
```

---

### 7-1. KRX 특이 지표 수집기

```
[역할] 한국 주식 데이터 수집 Go 개발자
[Input from] KIS Open API, KRX Open API, OpenDART
[Output to] 7-3(통합분석), 7-4(기술분석), DB, data/raw/{날짜}/krx.json
[에러 시] API 실패 시 3회 재시도 → 실패 확정 시 stale 플래그 + Slack 알림. 전일 데이터 fallback 금지(오래된 수급 데이터는 위험).
[소스] KIS Open API(REST), KRX Open API, OpenDART

[수집] 거래량 300%↑, 외국인/기관 순매수 Top20, 52주 신고가/신저가,
       급등(+5%)/급락(-5%), 시총 Top100

[Go 조건] goroutine+WaitGroup, context.WithTimeout(30s), 재시도 3회,
          slog 로깅, uber-go/ratelimit

[출력]
type KRXSignal struct {
    Ticker      string    `json:"ticker"`
    Name        string    `json:"name"`
    Price       int64     `json:"price"`          // 원 단위
    ChangeRate  float64   `json:"change_rate"`     // 비율
    Volume      int64     `json:"volume"`
    VolumeRatio float64   `json:"volume_ratio"`    // 비율
    ForeignNet  int64     `json:"foreign_net"`     // 원 단위
    InstNet     int64     `json:"inst_net"`        // 원 단위
    SignalType  string    `json:"signal_type"`
    CollectedAt time.Time `json:"collected_at"`
}
```

---

### 7-2. 미장(US) 지표 수집기

```
[역할] 글로벌 매크로 데이터 수집 Go 개발자
[Input from] Yahoo Finance, Alpha Vantage, FRED API, CNN/alternative.me
[Output to] 7-3(통합분석), DB(macro_daily), data/raw/{날짜}/us.json
[에러 시] 개별 API 실패 시 캐시 fallback(5분 TTL) + IsStale 플래그. 전체 실패 시 수집 skip + 알림.
[소스] Yahoo Finance/Alpha Vantage, FRED API, CNN/alternative.me(Fear&Greed)

[수집] 지수(SPY,QQQ,DIA,IWM), 변동성(VIX,VVIX), 채권(DGS2,DGS10),
       환율(KRW=X,DXY), 원자재(WTI,브렌트,금,은), 섹터ETF 8개, 빅테크 9개

[Go 조건] WaitGroup 병렬, ratelimit(초당2회), 캐시 fallback(5분 TTL),
          stale 감지(30분 초과 시 플래그)

[출력]
type USMarketData struct {
    Indices     map[string]MarketIndex `json:"indices"`
    Volatility  map[string]float64     `json:"volatility"`
    Bonds       BondData               `json:"bonds"`
    Forex       map[string]float64     `json:"forex"`
    Commodities map[string]float64     `json:"commodities"`
    Sectors     map[string]float64     `json:"sectors"`
    BigTech     map[string]float64     `json:"bigtech"`
    FearGreed   int                    `json:"fear_greed"`
    IsStale     bool                   `json:"is_stale"`
    CollectedAt time.Time              `json:"collected_at"`
}
```

---

### 7-3. 한미 통합 분석 (3-AI 합의)

```
[역할] 글로벌 매크로 분석가 + 한국 주식 트레이더
[방식] 3-AI 합의 (3-4 참조) — Claude, Gemini, ChatGPT에 동일 프롬프트 병렬 전송
[Input from] 7-2(USMarketData JSON), 7-1(KRXSignal JSON[]), 7-0-2(뉴스 JSON[]), 7-0-3(캘린더 JSON)
[Output to] 7-4(기술분석 입력), DB(ai_responses, ai_consensus, signals)
[에러 시] AI 1개 실패 → 나머지 2개 합의. 2개 실패 → SOFT_WARN. 3개 실패 → 시그널 skip + 알림.

[환각 방지 제약 — 필수]
- 모든 수치는 제공된 Input JSON 데이터 내에서만 인용할 것.
- 추측성 데이터와 팩트를 엄격히 구분: 추측에는 반드시 "추정" 또는 "_est" 표기.
- Input에 없는 종목코드, 가격, 지표를 생성하지 말 것.
- 불확실하면 confidence: "low" + "판단 보류"로 명시.

[분석] 시장 분위기, VIX 경고, 달러 영향, 미장→한국 예측,
       Tier별 스탠스, 섹터 추천, 리스크

[Tier별 분석 관점 — 같은 데이터를 봐도 요약 축이 다르다]

Tier1 (장기) 관점: "6개월~수년 보유 근거가 강화됐는가 약화됐는가?"
- 산업 구조 변화: AI, 반도체, 에너지 전환 등 1~3년 이상 지속될 테마
- 기업 체력: 매출 성장, 영업이익률, ROE, FCF, 부채, CAPEX
- 경쟁 우위 (Moat): 점유율, 기술력, 고객 락인, 규제 진입장벽
- 밸류에이션 위치: PER/PBR/EV/EBITDA의 역사적 범위(5~10년 Band) 대비 위치
- 자본 배분 정책: R&D, M&A, 자사주 매입/소각, 배당 → 주주 환원 의지
- 거시 체제 변화: 금리 방향, 달러 추세, 정책 변화가 장기적으로 유리한지
- 리스크: 사업 모델 훼손 가능성, 규제, 공급망, 지정학

Tier2 (중기) 관점: "앞으로 1~3개월 안에 주가를 움직일 촉매가 있는가?"
- 실적 모멘텀: 최근 분기 서프라이즈, 다음 분기 컨센서스 상향/하향 추이
- 수급 변화: 외국인/기관 순매수의 지속성과 매수 주체 성격 (연기금, 패시브 등)
- 섹터 순환: 시장이 어떤 업종으로 이동 중인지, 대장주 vs 후발주자 구분
- 이벤트 촉매: 실적 발표, 정책, 신제품, 수주, 임상, FOMC 등
- 기술적 흐름: 추세 유지, 박스권 돌파, 눌림목 가능성
- 단기 악재 지속성: 일회성인지, 1~3개월 이어질 구조적 악재인지

[AI 분석 유도 질문 — 프롬프트에 포함]
Tier1 (장기):
  Q1. "이 기업의 3년 후 시장 점유율은 지금보다 높아질 근거가 있는가?"
  Q2. "경쟁사 대비 압도적 강점(Economic Moat)은 무엇인가?"
  Q3. "매크로 환경(금리, 환율)이 최악일 때 이익 방어력은?"
Tier2 (중기):
  Q1. "향후 3개월 내 주가를 움직일 가장 큰 단일 이벤트는?"
  Q2. "애널리스트 목표가 추이가 상향 중인가 하향 중인가?"
  Q3. "업황 피크아웃(Peak-out) 논란이 있는가, 이제 시작 구간인가?"

[AI 공통 프롬프트]
3개 AI에 동일한 프롬프트와 데이터를 전송. 출력 스키마를 강제하여 비교 가능하게 함.

[출력 - JSON] (각 AI 동일 스키마)
{
  "date": "YYYY-MM-DD",
  "provider": "claude | gemini | chatgpt",
  "us_mood": "Risk-On | Risk-Off | Neutral",
  "vix_level": "Low | Medium | High | Extreme",
  "kr_outlook": "긍정 | 부정 | 중립",
  "tier1_analysis": {
    "action": "유지 | 리밸런싱 | 추가매수",
    "thesis_strength": "강화 | 유지 | 약화",
    "valuation": "저평가 | 적정 | 고평가",
    "moat_assessment": "근거 요약"
  },
  "tier2_analysis": {
    "stance": "공격적 | 보통 | 방어적",
    "earnings_revision": "상향 | 유지 | 하향",
    "key_catalyst": "촉매 요약",
    "sector_rotation": "유입 | 유지 | 유출"
  },
  "tier3_stance": "매매가능 | 자제 | 금지",
  "focus_sectors": [{"sector": "반도체", "reason": "근거", "sentiment_score": 0.85}],
  "risk_factors": [{"risk": "내용", "level": "high | medium"}],
  "watch_list": ["005930"],
  "avoid_list": ["종목코드"],
  "key_levels": {"kospi_support": float, "kospi_resistance": float},
  "confidence": "high | medium | low",
  "stale_warning": "stale 항목 명시"
}

[합의 처리]
1. 3개 AI 응답 수신 (각각 타임아웃 30초)
2. tier1_analysis.action, tier2_analysis.stance, tier3_stance 등 핵심 필드 다수결
3. focus_sectors: 2개 이상 AI가 언급한 섹터만 채택
4. watch_list: 합집합 (하나라도 추천하면 포함)
5. avoid_list: 합집합 (하나라도 경고하면 포함)
6. 합의 결과 + 개별 AI 원본 응답 모두 DB 저장

[규칙]
- 근거 부족 시 confidence "low" + 판단 보류
- stale 데이터 별도 경고
- JSON만 출력
- AI 간 stance가 완전히 엇갈리면 (예: 공격적 vs 금지) → HOLD 강제 + 알림
```

---

### 7-4. 기술적 분석 엔진

```
[역할] 퀀트 전략 Go 개발자 (go-talib)
[Input from] 7-3(통합분석 JSON — focus_sectors, watch_list), KIS API(일봉/분봉 데이터)
[Output to] DB(signals), 7-6(자동매매 → Guard → 주문)
[에러 시] 데이터 부족(일봉 120개 미만) 시 해당 종목 분석 skip. AI 불필요 — 순수 Go 계산.

[지표]
이동평균: SMA(5/20/60/120), EMA(12/26)
모멘텀:   RSI(14), MACD(12,26,9), Stochastic(14,3,3)
변동성:   볼린저밴드(20,2), ATR(14)
거래량:   VolumeRatio, OBV

[시그널 기준] → 6-3 Tier별 진입/매도 조건 참조

[출력]
type TechnicalSignal struct {
    Ticker      string    `json:"ticker"`
    Name        string    `json:"name"`
    Tier        int       `json:"tier"`
    Signal      string    `json:"signal"`       // BUY | SELL | HOLD
    Strength    int       `json:"strength"`      // 1~5
    RSI         float64   `json:"rsi"`           // 비율
    MACDCross   string    `json:"macd_cross"`
    VolumeRatio float64   `json:"volume_ratio"`  // 비율
    Reason      string    `json:"reason"`
    EntryPrice  int64     `json:"entry_price"`   // 원 단위
    StopLoss    int64     `json:"stop_loss"`     // 원 단위
    TargetPrice int64     `json:"target_price"`  // 원 단위
    AnalyzedAt  time.Time `json:"analyzed_at"`
}
```

---

### 7-5. 백테스팅 시스템

```
[역할] Go 기반 백테스팅 개발자
[Input from] DB(clean 시장 데이터 5년), 6장(전략 파라미터), 7-4(시그널 로직)
[Output to] 리포트(CSV, PNG), 전략 파라미터 검증 결과
[에러 시] 데이터 기간 부족 시 가용 기간만으로 실행 + 경고. 결과에 "제한된 기간" 명시.

[설정]
Tier별 파라미터: 6-2(리스크 상수) + 6-3(보유기간, 손절, 익절, 포지션 크기, 최대 보유) 참조
공통: 최근 5년(동적), 초기 1000만원, 수수료 0.015%, 세금 0.20%
      슬리피지: 대형주(시총 상위 100) 0.05%, 중소형주 0.1~0.2%, Tier3 모멘텀 종목 1~2호가 불리 체결 가정

[편향 방지]
- 생존자 편향: 상폐/합병 종목 포함 (point-in-time 유니버스)
- 선행 편향: 시그널 시점 이전 데이터만 (실적은 공시일 기준)
- 과최적화: Walk-forward 검증 필수
- 체결 가정: 시장가 기준 (종가 매매 가정 금지)

[출력] 총수익률, CAGR, MDD, 샤프비율, 승률, Profit Factor,
       Tier별 비교, 거래내역 CSV, 수익곡선 PNG, 코스피 대비 초과수익
```

---

### 7-6. KIS 자동매매

**7-6-1. 인증 모듈**

```
[역할] KIS API Go 클라이언트 개발자

실전: https://openapi.koreainvestment.com:9443
모의: https://openapivts.koreainvestment.com:29443
인증: OAuth2 (24시간)

요구: .env 로드, IS_PAPER_TRADING 모의 전환, 메모리+파일 이중 캐시, API 키 로그 절대 금지

[토큰 갱신]
- 만료 30분 전 자동 갱신 (백그라운드 goroutine)
- 401 응답 시 즉시 재발급 후 원래 요청 재시도 (HTTP 미들웨어)
- 동시 갱신 요청 방지: sync.Once 또는 singleflight 패턴

[Rate Limit]
- KIS REST API: 초당 2건 (uber-go/ratelimit). config.yaml `kis_tps` 참조.
- 주문 API는 별도 제한 확인 필요 (KIS 공지 기준)
```

**7-6-2. 주문 실행 모듈**

```
[역할] KIS REST API 주문 Go 개발자

[기능] 현재가/잔고 조회, 시장가·지정가 매수·매도, 주문 취소, 체결 조회

[흐름] 시그널 → Guard 계층 (7-6-5) → 주문 실행 → 체결 확인
※ 주문 실행 모듈은 Guard를 통과한 주문만 처리. 자체 검증 로직 없음.
```

**7-6-3. WebSocket 실시간 모니터링**

```
[역할] KIS WebSocket Go 개발자

WebSocket: ws://ops.koreainvestment.com:21000
구독: H0STCNT0(체결가), H0STASP0(호가), H0STCNI0(체결통보)

goroutine: WebSocket 수신 → channel → 주문 로직 (producer-consumer)
├── 손절/익절 감시 (1초 주기)
├── 일일 손실 한도 감시
├── 하트비트
└── OS 시그널 감시

재연결: exponential backoff (1s → 60s max)

[Graceful Shutdown]
SIGINT/SIGTERM → 신규 주문 중단 → 미체결 취소 → WS 종료
→ 포지션 DB 저장 → Slack 알림 → 30초 후 강제 종료
```

**7-6-4. 런타임 안전장치**

```
[역할] 트레이딩 런타임 안전장치 Go 개발자
※ Guard(7-6-5)는 주문 전 검증. 여기는 주문 후 + 상시 감시.

1. 부분 체결: 잔량 1시간 미체결 시 취소, 체결분 기준 손익가 재계산
2. Reconciliation: 30분마다 잔고 API vs 내부 상태 비교, 불일치 시 매매 중단+알림
3. Kill Switch: 신규 차단+미체결 취소, 해제는 수동만
   활성화 경로: (1) Slack slash command (2) 파일 플래그 (data/kill_switch) (3) REST API 엔드포인트
   ※ 어떤 경로든 활성화 즉시 Slack 알림. 해제 시에도 알림.
4. API 장애: 3회 연속 실패 → 매매 중단, 5분 후 재시도, 손절은 별도 5회 재시도
   Circuit Breaker: CLOSED → OPEN → HALF-OPEN → CLOSED
```

**7-6-5. Guard 계층 (주문 전 안전장치)**

```
[역할] 주문 전 안전장치 Go 개발자 (internal/guard/)
[원칙] 전략은 "사고 싶다"를 말하고, Guard는 "지금 사도 되나?"를 최종 검증한다.
       전략과 독립적으로 동작 — 전략을 바꿔도 Guard는 공통 재사용.

[구조]
시그널 → GuardEngine.Validate(signal) → PASS / BLOCK / WARN
         ├── Layer 1: MarketGuard    (시장 환경)
         ├── Layer 2: AccountGuard   (계좌/리스크)
         ├── Layer 3: StockGuard     (종목)
         └── Layer 4: OrderGuard     (주문 직전)

[판정 유형]
- HARD_BLOCK: 무조건 차단. 주문 불가.
- SOFT_WARN:  경고 알림만. 주문은 진행. (추후 수동 승인 전환 가능)
- PASS:       통과.

※ 아래 임계값의 정의 원본은 6-2(리스크 상수). Guard는 해당 값을 참조만 합니다.

[Layer 1 — 시장 환경 Guard] (market.go)
HARD  장시간 외 (TRADING_START 전, TRADING_END 후, 반장 12:20 후)
HARD  휴장일 (KRX 휴장 캘린더 API 또는 config.yaml 수동 목록)
      ※ 반장일(설/추석 전일 등): KRX 공지 기반. config.yaml `half_days` 목록으로 관리.
HARD  VIX > VIX_BLOCK → 매수 차단 (매도/손절은 허용)
HARD  Kill Switch 활성 상태
HARD  API 연결 불안정 (Circuit Breaker OPEN)
SOFT  VIX VIX_WARN~VIX_BLOCK → Tier3 신규 진입 경고
SOFT  장 시작 직후 5분

[Layer 2 — 계좌/리스크 Guard] (account.go)
HARD  일일 손실 MAX_DAILY_LOSS 초과
HARD  가용 현금 부족 (주문 금액 > 주문가능금액)
HARD  Tier별 최대 보유 초과 (6-3 max_positions 참조)
HARD  Tier별 비중 한도 초과 (6-2 MAX_TIER*_PCT 참조)
HARD  잔고 불일치 감지 (Reconciliation 실패 상태)
SOFT  일일 손실 MAX_DAILY_LOSS × 75% 접근 → 경고

[Layer 3 — 종목 Guard] (stock.go)
HARD  단일 종목 비중 MAX_SINGLE_PCT 초과
HARD  거래대금 일평균 MIN_TRADE_VALUE 미만
HARD  관리종목/투자주의/거래정지
HARD  Stale 데이터 (stale_threshold 초과) → 매수 차단 (매도/손절은 허용)
SOFT  호가 스프레드 SPREAD_WARN 초과
SOFT  당일 급등 +15% 이상 (추격매수 경고)
SOFT  실적발표 D-3 ~ D+1 (Tier2)

[Layer 4 — 주문 직전 Guard] (order.go)
HARD  중복 주문 (idempotency key: 종목+날짜+시그널+시퀀스)
HARD  미체결 주문 존재 시 동일 종목 신규 주문 차단
HARD  주문 가격 괴리 PRICE_DIVERGE 초과 (현재가 대비)
HARD  동일 Tier 내 당일 신규 진입 max_daily_entries_per_tier 초과
HARD  IS_PAPER_TRADING 모의/실전 불일치
SOFT  주문 금액이 평소 대비 200% 이상

[GuardEngine 구현]
type GuardResult struct {
    Layer   string    // "MARKET", "ACCOUNT", "STOCK", "ORDER"
    Rule    string    // "vix_block", "position_limit" 등
    Result  string    // "PASS", "HARD_BLOCK", "SOFT_WARN"
    Reason  string    // 사람이 읽을 수 있는 차단 사유
    Context map[string]any // 판단 시점 상태값
}

func (e *GuardEngine) Validate(signal Signal) (bool, []GuardResult) {
    // 4개 Layer 순차 평가
    // 하나라도 HARD_BLOCK → 즉시 차단 (이후 Layer skip)
    // SOFT_WARN → 경고 알림 발송, 주문은 계속
    // 모든 결과를 guard_logs 테이블에 저장
}

[로그 예시]
PASS  market_guard.trading_hours    "장중 정상"
PASS  account_guard.daily_loss      "일일 손실 -0.8% (한도 -2%)"
BLOCK stock_guard.position_limit    "삼성전자 비중 12.3% > 한도 10%"
SKIP  order_guard                   "(앞단 차단으로 미실행)"

[Guard 상태 대시보드 (Slack 알림)]
🟢 MarketGuard   PASS  │ VIX 18.3 / 정규장
🟢 AccountGuard  PASS  │ 일손실 -0.8% / 보유 6종목
🔴 StockGuard    BLOCK │ 005930 비중 12.3% 초과
⚪ OrderGuard    SKIP  │ (앞단 차단)
→ 주문 차단: 005930 매수 시그널 [signal_id: 142]

[핵심 원칙]
1. Guard는 상태를 변경하지 않음 — 순수 검증만 수행
2. 하나라도 HARD_BLOCK이면 전체 차단 (AND 조건)
3. 모든 판정 이력 DB 저장 (guard_logs) — 전략 개선/사고 추적용
4. Guard 임계값은 config.yaml(3-6)에서 로드 — 코드 수정 없이 조정 가능. 리스크 상수 원본은 6-2.
5. "왜 주문이 안 나갔는지" 3단계 구분: 시그널 미발생 / Guard 차단 / 주문 실패
```

---

## 8. 리포트 체계

### 8-0. 리포트 렌더링

```
[템플릿] VerseKit HTML (templates/reports/)
├── daily.html       일일 리포트
├── monthly.html     월간 리포트
└── annual.html      연간 리포트

[렌더링 흐름]
1. internal/report/ 에서 데이터 수집 + 집계
2. Go html/template으로 VerseKit HTML 렌더링
3. data/reports/{date}-daily.html 등으로 저장
4. Slack에는 텍스트 요약본 발송 (아래 각 섹션 형식)

[디자인 시스템]
- 폰트: Space Grotesk (본문) + JetBrains Mono (데이터)
- 색상: #ff4d00 (강조), #00c853 (수익), #ff1744 (손실), #0f0f0f (다크)
- 레이아웃: stats-grid (3/4/6 col), highlight-box, bar-chart
- 템플릿 변수: Go html/template 문법 — {{.Variable}}, {{range .List}}, {{if .Cond}}
  ※ 현재 HTML 파일은 Handlebars 스타일로 작성됨. Go 구현 시 변환 필요.
```

### 8-1. 일일 리포트 (매일 08:00, 전일 결과)

```
[발송] Slack 텍스트 요약 + HTML 리포트 저장 (data/reports/{date}-daily.html)
[시점] 아침 08:00 — 모닝 브리핑(07:00) 후 연이어 수신

*AlphaRadar 일일 리포트* | {전일 date}
━━━━━━━━━━━━━━━━━━━━

*전일 성과*
- 일일 손익: {daily_pnl}원 ({daily_pct}%)
- 누적 손익: {total_pnl}원 ({total_pct}%)
- 총 자산: {total_asset}원

*Tier별 현황*
- T1: {value}원 | 비중 {pct}% | 평가손익 {pnl}%
- T2: {value}원 | 보유 {count}개 | 평가손익 {pnl}%
- T3: {value}원 | 보유 {count}개 | 평가손익 {pnl}%
- 현금: {cash}원 ({cash_pct}%)

*전일 체결*
- 매수: {list} / 매도: {list} / 손절: {list} / 익절: {list}

*보유 종목*
| 종목 | Tier | 수량 | 평균가 | 현재가 | 손익률 | 손절가 | 목표가 |

*시그널 요약*
- 발생: {count}개 | 실행: {count}개 | 미실행 사유: {reasons}

*시스템 상태*
- API 오류: {n}건 | stale: {n}건 | WS 재연결: {n}회

*오늘 주목*
- 손절 임박: {list} | 목표가 임박: {list} | 예정 이벤트: {list}
━━━━━━━━━━━━━━━━━━━━
```

### 8-2. 월간 리포트 (매월 1일 08:00, 전월 실적)

```
[발송] Slack 텍스트 요약 + HTML 리포트 저장 (data/reports/{YYYY-MM}-monthly.html)
[시점] 매월 1일 — 일일 리포트 대신 발송

*AlphaRadar 월간 리포트* | {year}년 {month}월
━━━━━━━━━━━━━━━━━━━━

*월간 성과*
- 수익률: {pct}% (코스피: {kospi_pct}%, 초과: {alpha}%)
- 월초 {start}원 → 월말 {end}원 | 손익 {pnl}원

*Tier별 성과*
| Tier | 수익률 | 거래 수 | 승률 | 평균수익 | 평균손실 | Profit Factor |

*매매 통계*
- 총 {n}건 (매수 {b} / 매도 {s}) | 승률 {w}%
- 최대 수익: {best} ({pct}%) | 최대 손실: {worst} ({pct}%)
- 최대 연속 손실: {streak}회 | 평균 보유: {days}일

*리스크*
- MDD: {mdd}% | 일별 최대 손실: {worst_day}%
- 손절 {n}회 | VIX 차단 {n}회 | Kill Switch {n}회

*시그널 정확도*
- 매수 적중률: {hit}% (익절 도달 기준)
- 미실행 시그널 사후: "실행했으면 {pnl}%"
- 적중률 추세: 전월 {prev}% → 이번 달 {curr}% ({direction})

*AI별 성적표*
- Claude: 적중률 {hit}% | 수익 기여 {pnl}% | 단독 정답 {n}회
- Gemini: 적중률 {hit}% | 수익 기여 {pnl}% | 단독 정답 {n}회
- ChatGPT: 적중률 {hit}% | 수익 기여 {pnl}% | 단독 정답 {n}회
- 합의 불일치 → HOLD 강제: {n}회

*Guard 사후 분석*
- Guard 차단 총 {n}건 | 차단이 옳았던 비율: {correct}%
- 차단했으나 수익이었을 거래: {missed_profit}건 ({missed_pnl}%)
- 가장 많이 발동된 규칙: {top_rule} ({count}회)

*거래 비용 분석*
- 수수료: {commission}원 | 세금: {tax}원 | 슬리피지(추정): {slip}원
- 비용 합계: {total_cost}원 (수익 대비 {cost_ratio}%)

*시장환경 vs 성과*
- VIX 20 미만 구간: {days}일 | 수익률 {pct}%
- VIX 20~30 구간: {days}일 | 수익률 {pct}%
- VIX 30+ 구간: {days}일 | 수익률 {pct}%
- 최고 성과 섹터: {sector} ({pct}%) | 최저: {sector} ({pct}%)

*비중 변화*
- 월초: T1 {p}% / T2 {p}% / T3 {p}% / 현금 {p}%
- 월말: T1 {p}% / T2 {p}% / T3 {p}% / 현금 {p}%

*시스템*
- 가동률 {uptime}% | API 오류 {n}건 | stale {n}일 | 잔고 불일치 {n}건
━━━━━━━━━━━━━━━━━━━━
```

### 8-3. 연간 리포트 (매년 1/2 08:00, 전년 실적)

```
[발송] Slack 텍스트 요약 + HTML 리포트 저장 (data/reports/{YYYY}-annual.html) + CSV + PNG
[시점] 매년 1월 2일 — 월간 리포트와 함께 발송

*AlphaRadar 연간 리포트* | {year}년
━━━━━━━━━━━━━━━━━━━━

*연간 성과*
- 수익률: {pct}% (코스피 {kospi}%, S&P500 {sp500}%, Alpha {alpha}%)
- 연초 {start}원 → 연말 {end}원 | 순손익 {pnl}원

*월별 추이*
| 월 | 수익률 | 거래 수 | 승률 | MDD | 코스피 |

*Tier별 연간*
| Tier | 수익률 | 총 거래 | 승률 | 샤프 | MDD | PF | 목표 달성 |

*리스크 분석*
- MDD: {mdd}% ({date}) | 회복: {days}일
- 샤프비율: {sharpe} | 소르티노: {sortino}
- 최대 연속 손실: {streak}회 | VaR(95%): {var}%

*세금 기초자료*
- 국내 양도차익: {kr_gain}원
- 해외 ETF 양도차익: {os_gain}원 (공제 후 {taxable}원, 예상 세금 {tax}원)
- 배당 수익: {div}원 | 총 수수료/세금: {fees}원

*전략 평가* (목표 수익률은 6-3 참조)
- T1 목표 → 실제 {actual}%
- T2 목표 → 실제 {actual}%
- T3 목표 → 실제 {actual}%
- 시그널 연간 적중률: {hit}%
- 최고 섹터: {sector} ({pct}%) | 최저: {sector} ({pct}%)

*AI 연간 성적표*
- Claude: 적중률 {hit}% | 수익 기여 {contrib}% | 단독 정답 {n}회
- Gemini: 적중률 {hit}% | 수익 기여 {contrib}% | 단독 정답 {n}회
- ChatGPT: 적중률 {hit}% | 수익 기여 {contrib}% | 단독 정답 {n}회
- 합의 모드 효과: 3/3 일치 시 적중률 {pct}% vs 2/3 시 {pct}%
- 내년 합의 모드 권장: {recommendation}

*Guard 연간 효과*
- 연간 차단 총 {n}건 | 차단 정확률: {correct}%
- Guard가 방어한 추정 손실: {saved}원
- Guard가 놓친 추정 수익: {missed}원
- 규칙별 발동: {rule1} {n}회, {rule2} {n}회, ...
- 규칙 튜닝 권장: {recommendations}

*시장 레짐별 성과*
- 상승장 (코스피 +5%↑ 월): {months}개월 | 수익률 {pct}%
- 횡보장 (코스피 ±5%): {months}개월 | 수익률 {pct}%
- 하락장 (코스피 -5%↓ 월): {months}개월 | 수익률 {pct}%
- 전천후 지수: {score} (상승장 수익률 ÷ 하락장 방어력)

*거래 효율성*
- 총 거래 {n}건 중 수익 {w}건 ({win_rate}%)
- 평균 보유기간: T2 {days}일 (목표 30~90일) | T3 {days}일 (목표 1~10일)
- 불필요 거래(churning) 추정: {n}건 (진입 후 3일 이내 손절)
- 연간 거래 비용: {total_cost}원 (수수료+세금+슬리피지)

*전략 파라미터 드리프트* (설정값은 6-3 참조)
- 실제 평균 손절: T2 {actual}% (vs 설정) | T3 {actual}% (vs 설정)
- 실제 평균 익절: T2 {actual}% (vs 설정) | T3 {actual}% (vs 설정)
- 실제 평균 보유: T2 {actual}일 (vs 설정) | T3 {actual}일 (vs 설정)
- 괴리 큰 파라미터 → 다음 해 조정 검토 대상

*시스템 연간*
- 가동일: {days} | 가동률: {uptime}% | 총 거래: {trades}건
- API 장애: {incidents}건 | Kill Switch: {ks}회

*첨부*: {year}_trades.csv, {year}_monthly.csv, {year}_equity.png
━━━━━━━━━━━━━━━━━━━━
```

---

## 9. 알림 시스템

```
[지원] Slack (Webhook) / 텔레그램 (Bot API)

[실전/모의 시각적 구분 — 필수]
- IS_PAPER_TRADING=true  → 모든 알림 메시지 앞에 [PAPER] 접두사. 색상: 파란색.
- IS_PAPER_TRADING=false → 모든 알림 메시지 앞에 [REAL] 접두사. 색상: 붉은색.
- 시스템 시작 시 Slack에 "[REAL] AlphaRadar 실전 모드 시작" 또는 "[PAPER] 모의투자 모드 시작" 발송.
- 실전 모드 최초 전환 시 수동 확인 필수 (Slack 버튼 또는 CLI 확인).

[트리거]
07:00  모닝 브리핑                     → 전문 발송
08:00  일일/월간/연간 리포트             → 8장 참조
08:50  매매 후보                        → 종목별 Tier/강도/진입/손절/목표
체결   매수/매도 체결 즉시               → 종목, 수량, 가격, 금액
손익   손절/익절 발생 즉시               → 종목, 손익률
긴급   API 오류, stale, 잔고 불일치      → 즉시 발송
시스템 Kill Switch, Graceful Shutdown     → 상태 변경 알림
```

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

**장애 대응:**
- KIS API 장애 → Fail-Closed (신규 매매 자동 중단)
- AI 1개 장애 → 나머지 2개로 합의 진행 (2/2 일치 시 실행)
- AI 2개 장애 → 남은 1개 결과를 SOFT_WARN 처리 (자동 실행 안 함, 알림만)
- AI 3개 장애 → 시그널 생성 skip + 브리핑 skip + 긴급 알림
- 네트워크 장애 → WebSocket 자동 재연결 + 실패 시 알림
- 데이터 불일치 → 매매 중단 + 수동 확인

---

## 13. 참고 링크

| 항목 | URL |
|------|-----|
| KIS Developers | https://apiportal.koreainvestment.com |
| KIS GitHub 샘플 | https://github.com/koreainvestment/open-trading-api |
| OpenDART API | https://opendart.fss.or.kr |
| 한국은행 ECOS | https://ecos.bok.or.kr/api |
| KRX Open API | https://data.krx.co.kr |
| FRED API | https://fred.stlouisfed.org/docs/api |
| SEC EDGAR API | https://efts.sec.gov/LATEST/search-index |
| Claude Cowork 스케줄링 | https://support.claude.com/en/articles/13854387 |
| go-talib | https://github.com/markcheno/go-talib |
| gorilla/websocket | https://github.com/gorilla/websocket |
| uber-go/ratelimit | https://github.com/uber-go/ratelimit |
| robfig/cron | https://github.com/robfig/cron |
| Claude API (Anthropic) | https://docs.anthropic.com |
| OpenAI API (ChatGPT) | https://platform.openai.com/docs |
| Google AI (Gemini) | https://ai.google.dev/docs |

---

## 변경 이력

| 버전 | 날짜 | 주요 변경 |
|------|------|-----------|
| v1.0 | 2026-03-09 | 초기 설계 완료. 전체 아키텍처, 3-AI 합의, 4-Layer Guard, 3-Tier 전략, Tier별 분석 관점(장기=구조/Moat, 중기=촉매/수급), 데이터 체이닝, DB 스키마(KRW int64), VerseKit 리포트(일간/월간/연간), 관측성, 운영 수칙 |

---

*Last Updated: 2026-03-09*
*Project: AlphaRadar (KORUS)*
