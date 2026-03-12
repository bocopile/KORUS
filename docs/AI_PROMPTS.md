# AlphaRadar — 단계별 프롬프트 / LLM 입력용 (7장)

> 원본: CLAUDE.md 7장에서 분리
> 관련 문서: [ARCHITECTURE.md](ARCHITECTURE.md), [STRATEGY.md](STRATEGY.md)
> 용도: LLM(Claude/Gemini/ChatGPT)에 입력 시 해당 단계 블록만 발췌 사용

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

**단계별 완료 기준 (DoD — Definition of Done):**

| 단계 | 완료 조건 | 필수 테스트 | 실패 허용 범위 |
|------|-----------|-------------|----------------|
| 7-0-1 거시지표 | JSON 전체 필드 수집 + DB 저장 | API 응답 파싱 테스트, null 필드 처리 테스트 | 개별 필드 null 허용, 전체 실패 시 stale fallback |
| 7-0-2 뉴스 | 최소 3개 이상 뉴스 수집 + JSON 배열 반환 | 중복 제거 테스트, importance 필터 테스트 | 0개 수집 시 빈 배열 (에러 아님) |
| 7-0-3 캘린더 | 당일 이벤트 JSON 반환 | 시간대(KST) 변환 테스트 | 빈 배열 허용 + 경고 로그 |
| 7-0-4 브리핑 | Slack 마크다운 50줄 이내 + .md 저장 + DB 저장 | 입력 일부 null 시 "N/A" 표시 테스트 | 전체 입력 실패 시 skip |
| 7-1 KRX | KRXSignal[] 반환 + raw JSON 저장 | Rate Limit 준수 테스트, 재시도 3회 테스트 | 전일 fallback 금지 — 실패 시 stale |
| 7-2 US | USMarketData 반환 + IsStale 플래그 정확 | 캐시 fallback 테스트, stale 감지 테스트 | 개별 API 실패 시 캐시 허용 |
| 7-3 통합분석 | 3-AI 응답 + 합의 JSON + 개별 원본 DB 저장 | 2/3 합의 테스트, 전원 불일치→HOLD 테스트 | AI 1개 장애 시 2개 합의 |
| 7-4 기술분석 | TechnicalSignal[] 반환 + 지표 계산 정확 | RSI/MACD/BB 수치 검증 (기준 데이터 대비) | 일봉 120개 미만 시 skip |
| 7-6 자동매매 | Guard 통과 후 주문 체결 + DB 저장 | Guard 4-Layer 전체 통과/차단 테스트 | Guard HARD_BLOCK 시 주문 불가 |

**산출물 체크리스트 (모든 단계 공통):**
- [ ] 출력 JSON이 정의된 스키마와 일치하는가
- [ ] DB 저장이 정상적으로 완료되었는가
- [ ] 실패 시 Slack 알림이 발송되었는가
- [ ] `go test -race` 통과하는가
- [ ] slog 로그에 민감 데이터가 노출되지 않는가

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

[정답 샘플 — 2026-03-06 금요일 기준]
{
  "date": "2026-03-06",
  "collected_at": "2026-03-07T06:15:00+09:00",
  "kr_market": {
    "kospi": 2645.32, "kospi_change_pct": -0.83,
    "kosdaq": 812.45, "kosdaq_change_pct": -1.12,
    "usdkrw": 1382.50, "kr_bond_3y": 3.15,
    "foreign_net_buy": -285000000000, "inst_net_buy": 120000000000
  },
  "us_market": {
    "sp500": 5104.76, "sp500_change_pct": 0.52,
    "nasdaq": 16205.34, "nasdaq_change_pct": 0.78,
    "dow": 39142.89, "dow_change_pct": 0.23,
    "russell2000": 2067.45, "russell2000_change_pct": -0.15,
    "vix": 18.32, "dxy": 103.85
  },
  "bonds": {"us10y": 4.25, "us2y": 4.62, "spread": -0.37},
  "commodities": {"wti": 78.52, "brent": 82.14, "gold": 2158.30, "silver": 24.85},
  "macro": {
    "cpi": 3.1, "cpi_date": "2026-02-12",
    "pce": 2.8, "pce_date": "2026-02-28",
    "unemployment": 3.7, "gdp": 2.1, "fed_rate": 5.25
  },
  "fed_watch": {"hike": 0.02, "cut": 0.68, "hold": 0.30},
  "asia": {"nikkei": 39856.12, "shanghai": 3027.45, "hangseng": 16589.34, "taiex": 19872.56}
}
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

[정답 샘플 — 2026-03-07 기준, 2개만 예시]
[
  {
    "category": "반도체/기술주",
    "headline": "삼성전자, 2nm GAA 양산 일정 앞당긴다",
    "summary": "삼성전자가 2nm GAA 공정 양산을 당초 2027년에서 2026년 하반기로 앞당기는 방안을 검토 중. TSMC와의 파운드리 격차 축소가 목표.",
    "market_impact": "긍정",
    "affected_sectors": ["반도체", "IT"],
    "importance": "high",
    "source": "한국경제",
    "source_url": null,
    "published_at": "2026-03-07T06:30"
  },
  {
    "category": "중앙은행/금리",
    "headline": "연준 위원 '6월 금리 인하 가능성 열려'",
    "summary": "연준 이사가 인플레이션 둔화 추세가 이어질 경우 6월 FOMC에서 금리 인하를 논의할 수 있다고 발언. 시장은 6월 인하 확률을 68%로 반영.",
    "market_impact": "긍정",
    "affected_sectors": ["금융", "부동산"],
    "importance": "high",
    "source": "Reuters",
    "source_url": null,
    "published_at": "2026-03-06T22:15"
  }
]
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

[정답 샘플 — 2026-03-07 금요일]
*AlphaRadar 모닝 브리핑* | 2026-03-07 금요일
━━━━━━━━━━━━━━━━━━━━
*글로벌 시장*
- S&P500 +0.52% | NASDAQ +0.78% | DOW +0.23%
- VIX: 18.3 (Low) | DXY: 103.85 | 원달러: 1,382원

*시장 분위기*
Risk-On 기조 유지. 연준 위원의 6월 금리 인하 가능성 시사에 기술주 중심 강세.
다만 장단기 금리차 역전(-0.37%p) 지속, 경기 침체 우려는 여전.

*핵심 뉴스 Top5*
1. 삼성전자 2nm GAA 양산 일정 앞당긴다 [긍정]
2. 연준 위원 '6월 금리 인하 가능성 열려' [긍정]
3. 중국 2월 수출 +5.6% 예상 상회 [긍정]
4. 테슬라 유럽 판매 -12% 감소 [부정]
5. 한국 2월 소비자물가 +2.8% [중립]

*경제 이벤트*
21:30 | 미국 2월 고용보고서(NFP) | 예상 20.0만 | 이전 22.9만
22:00 | 미국 2월 ISM 서비스업 PMI | 예상 53.0 | 이전 53.4

*국내 시장 주목*
- 코스피 외국인 순매도 지속(-2,850억), 기관 순매수로 방어
- 반도체 섹터 주목: 삼성전자 2nm 뉴스 + HBM 수요 견조

*3-Tier 스탠스*
- T1: 유지 | T2: 보통 (고용지표 확인 후 판단) | T3: 가능 (VIX 18.3)

*리스크*
- 미국 고용보고서 서프라이즈 시 금리 인하 기대 후퇴 가능
- 원달러 1,380원대 불안정, 외국인 이탈 지속 시 하방 압력

*한 줄 전략* "고용지표 발표 전까지 관망, 서프라이즈 방향에 따라 T2 스탠스 조정"
━━━━━━━━━━━━━━━━━━━━
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

[Hashkey — 주문 API 보안]
- KIS 공식 문서 기준 현재 "optional" (필수 아님). 단, 데이터 변조 방지를 위해 AlphaRadar는 필수 적용.
- 매수·매도·정정·취소 등 POST 주문 API 호출 시 Body를 Hashkey API로 서명 후 헤더에 포함
- GET(조회) API: Hashkey 불필요
- 흐름: 주문 Body JSON → POST /uapi/hashkey → 응답 hash값 → 주문 요청 헤더 hashkey 필드에 포함
- KIS API가 Hashkey 검증을 강제화할 경우 미적용 시 주문 거부될 수 있음
- hash값 로그 출력 금지

[Rate Limit]
- KIS REST API: 초당 2건 (uber-go/ratelimit). config.yaml `kis_tps` 참조.
- 주문 API는 별도 제한 확인 필요 (KIS 공지 기준)
```

**7-6-2. 주문 실행 모듈**

```
[역할] KIS REST API 주문 Go 개발자

[기능]
- 매수가능금액 조회: 주문 전 가용 현금 확인 → Guard Layer 2(AccountGuard) 연동
- 매도가능수량 조회: 매도 주문 전 보유수량 확인 (내부 상태 불일치 방지)
- 시장가·지정가 매수·매도
- 정정주문: 원주문 ID → `parent_order_id` 연결 필수, cancel_reason='AMEND'
- 취소주문: `cancel_reason` 저장 필수 (소명 Audit Log 연동, 12-3 참조)
- 체결 조회

[슬리피지 관리]
- 시장가 주문: 호가 스프레드 SPREAD_WARN(2%) 초과 시 지정가 전환 검토 → Guard SOFT_WARN
- 지정가 주문: 현재가 ±PRICE_DIVERGE(2%) 이내만 허용 → Guard Layer 4 HARD_BLOCK
- 체결 후 예상가 대비 1% 초과 슬리피지 발생 시 `orders.context_json`에 기록
- 수수료/세금 실전 반영: 매수 0.015% + 매도 0.015% + 세금 0.20% → 손익 계산 시 포함

[흐름] 시그널 → Guard 계층 (7-6-5) → Hashkey 서명 → 주문 실행 → 체결 확인
※ 주문 실행 모듈은 Guard를 통과한 주문만 처리. 자체 검증 로직 없음.
```

**7-6-3. WebSocket 실시간 모니터링**

```
[역할] KIS WebSocket Go 개발자

[WebSocket 접속키 발급/갱신]
- WS 연결 전 접속키 발급 필수: POST /oauth2/Approval
- 접속키 유효기간: 24시간 (Access Token과 별도 관리)
- **발급 제한: 1분당 1회** — 재연결 시 유효기간 내 기존 접속키 재사용 필수. 1분 내 재발급 시도 시 에러
- 만료 30분 전 자동 갱신 (백그라운드 goroutine, 토큰 갱신 goroutine과 독립 실행)
- exponential backoff 재연결 시 접속키 재사용 가능 여부 확인 후 재발급 결정
- 접속키 로그 출력 금지

WebSocket:
  실전: ws://ops.koreainvestment.com:21000
  모의: ws://ops.koreainvestment.com:31000  ← 실전과 다름. IS_PAPER_TRADING으로 분기 필수
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
HARD  VI(변동성완화장치) 발동 중 → 신규 주문 차단 (매도/손절은 허용)
HARD  Stale 데이터 (stale_threshold 초과) → 매수 차단 (매도/손절은 허용)
SOFT  호가 스프레드 SPREAD_WARN 초과
SOFT  당일 급등 +15% 이상 (추격매수 경고)
SOFT  상/하한가 ±3% 이내 (상/하한가 관여 경고)
SOFT  실적발표 D-3 ~ D+1 (Tier2)

[Layer 4 — 주문 직전 Guard] (order.go)
HARD  중복 주문 (idempotency key: 종목+날짜+시그널+시퀀스)
HARD  미체결 주문 존재 시 동일 종목 신규 주문 차단
HARD  주문 가격 괴리 PRICE_DIVERGE 초과 (현재가 대비)
HARD  동일 Tier 내 당일 신규 진입 max_daily_entries_per_tier 초과
HARD  IS_PAPER_TRADING 모의/실전 불일치
HARD  최소 주문 간격 미준수 (동일 계좌 직전 주문 후 3초 이내 연속 주문 차단) ← KRX 감시 대응
HARD  15:20 이후 신규 매수 차단 (동시호가/종가 관여 방지) ← KRX 감시 대응
      ※ 기존 TRADING_END(14:50) 이후 전체 차단과 별개로, 15:20부터 매수만 선차단
HARD  허수호가 패턴 감지: 동일 종목 당일 미체결 취소 3회 연속 → 해당 종목 당일 신규 주문 차단 ← KRX 감시 대응
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

### 7-6-6. 이벤트 기반 매매 (뉴스 → 주문 파이프라인)

```
[역할] 실시간 뉴스/이벤트 → 매매 신호 변환 Go 개발자
[Input from] 7-0-2(뉴스 JSON[]), 7-3(통합분석 JSON)
[Output to] DB(signals) → 7-6-2(주문 실행) via Guard
[에러 시] 중복 이벤트 → 무시(idempotency). 낮은 신뢰도 → 시그널 생성 skip + 로그.

[이벤트 → 시그널 변환 규칙]

1. 키워드 가중치 (config.yaml event_keywords에서 관리)
   HIGH_POSITIVE: ["어닝서프라이즈", "FDA 승인", "수주 계약", "자사주 매입", "합병"]
   HIGH_NEGATIVE: ["대규모 적자", "CFO 사임", "공시 정정", "검찰 조사", "거래정지 예고"]
   트리거 조건: importance=high + market_impact=긍정/부정 + affected_sectors 보유 종목 1개 이상 일치

2. 신뢰도 점수 산출
   - AI 3개 감성 분석 평균 (7-3 뉴스 감성 분석 활용)
   - confidence < 0.6 → 시그널 생성 skip
   - 단일 매체 단독 보도 → confidence -0.1 (미확인 리스크)

3. 중복 뉴스 처리
   - 동일 ticker + event_type + 24시간 이내 → 중복 판정, 추가 시그널 생성 금지
   - idempotency key: SHA256(ticker + event_type + date)

4. 이벤트 → Tier 매핑
   긍정 공시(수주/계약) → Tier3: 당일 즉시 매매 시그널
   실적 서프라이즈     → Tier2: 다음 거래일 진입 검토
   거시지표 서프라이즈  → Tier1: 리밸런싱 트리거 검토
   부정 뉴스           → 전 Tier: 보유 종목 청산 검토

5. 제약 사항
   - 장 종료 1시간 이내(13:50 이후) 이벤트 시그널 → 익일 처리 (당일 진입 금지)
   - 실적발표 D-3~D+1 Tier2 진입 금지 (Guard Layer 3 SOFT_WARN과 연동)
   - 동일 종목 당일 이벤트 시그널 최대 1회 (중복 주문 방지)

[출력]
type EventSignal struct {
    Ticker      string    `json:"ticker"`
    EventType   string    `json:"event_type"`    // "EARNINGS_BEAT", "CONTRACT", "NEGATIVE_NEWS"
    Tier        int       `json:"tier"`
    Signal      string    `json:"signal"`         // BUY | SELL | HOLD
    Confidence  float64   `json:"confidence"`     // 0.0~1.0
    NewsIDs     []string  `json:"news_ids"`       // 근거 뉴스 ID 목록 (소명 Audit Log 연동)
    Reason      string    `json:"reason"`
    CreatedAt   time.Time `json:"created_at"`
}
```

---
