# AlphaRadar (KORUS) — 프로젝트 컨텍스트

> 이 파일은 Claude Code CLI가 자동으로 로드하는 프로젝트 컨텍스트입니다.
> 상세 설계는 `docs/` 디렉토리의 개별 문서를 참조하세요.
> 요구사항 개요는 PROMPT.md를 참조하세요.

## 프로젝트 요약

- **프로젝트명**: AlphaRadar (코드명: KORUS)
- **개발 언어**: Go 1.22+
- **모듈명**: github.com/bhshin/alpharadar
- **자동매매 API**: 한국투자증권 KIS Developers (REST + WebSocket)
- **AI 엔진**: 3-AI 합의 시스템 (Claude + Gemini + ChatGPT)
- **규칙 우선순위**: OPERATIONS.md(운영/안전) > AI_PROMPTS.md 7-6-5(Guard) > STRATEGY.md(전략) > AI_PROMPTS.md(프롬프트)
- **Source of Truth**: `PROMPT.md`(요구사항) → `CLAUDE.md`(인덱스) → `docs/*.md`(상세 설계)

## 현재 구현 범위

| 항목 | 현재 범위 | 비고 |
|------|-----------|------|
| 대상 시장 | **국내주식 (KRX) 중심** | 해외주식은 7단계 예정 |
| 매매 방식 | **장중 매매** (09:05~14:50) | 종가/시간외 매매 없음 |
| 주문 유형 | 시장가 + 지정가 | 예약주문 미포함 |
| 분할매수 | Tier1 DCA, Tier2 익절 후 추적 손절 | 7-6-2 주문 모듈 수준 상세 명세 |
| 종목 범위 | 국내 대형주 + ETF (Tier1), 섹터 성장주 (Tier2), 모멘텀 종목 (Tier3) | MIN_TRADE_VALUE 1억원 이상 |
| 운용 주체 | **본인 자금만** | 타인 자금 운용 금지 (자본시장법) |

## 시스템 범위 결정 (단계별)

```
[현재] 최소 운영형
  Go 자동매매 엔진 + KIS REST/WS + SQLite + 알림
  → 전략 설계 UI 없음. 백테스트는 Go 내장(7-5).

[운영 검증 후] 확장형
  + PostgreSQL + Redis + Prometheus/Grafana

[보류] 제품형
  + Python 백테스트 계층 + Next.js 전략 빌더 + FastAPI
  → 범위가 3개 언어(Go+Python+JS)로 늘어남. 지금 결정 안 함.
```
→ 상세 기술 스택 결정표: [DEVELOPMENT.md 10-1](docs/DEVELOPMENT.md)

---

## 문서 구조

| 문서 | 장 | 내용 | 용도 |
|------|-----|------|------|
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | 3~5장 | 디렉토리, 데이터 흐름, 3-AI, 스케줄링, DB 스키마 | 구조 설계 전반 |
| [docs/STRATEGY.md](docs/STRATEGY.md) | 6장 | 3-Tier 배분, 리스크 상수, Tier별 진입/청산 조건 | 투자 전략 |
| [docs/AI_PROMPTS.md](docs/AI_PROMPTS.md) | 7장 | 단계별 AI 입력 프롬프트, Guard 계층, 자동매매 | LLM 발췌 입력용 |
| [docs/REPORTS.md](docs/REPORTS.md) | 8~9장 | 일간/월간/연간 리포트, 알림 트리거 | 리포트/알림 |
| [docs/DEVELOPMENT.md](docs/DEVELOPMENT.md) | 10~11장 | Go 코드 원칙, 디버깅/코드리뷰 | 개발 가이드 |
| [docs/OPERATIONS.md](docs/OPERATIONS.md) | 12~13장 | 운영 수칙, 보안, 장애 대응, 참고 링크 | 운영/배포 |

---

## 단계별 필요 문서 매핑

orchestrator 등에서 단계별로 실행할 때, 필요한 문서만 선택적으로 전달하세요.

| 실행 단계 | 필요 문서 |
|-----------|-----------|
| 0단계 (모닝 브리핑) | `ARCHITECTURE.md` + `AI_PROMPTS.md` |
| 1~2단계 (수집기) | `ARCHITECTURE.md` + `DEVELOPMENT.md` |
| 3단계 (통합 분석) | `ARCHITECTURE.md` + `STRATEGY.md` + `AI_PROMPTS.md` |
| 4~5단계 (기술분석/백테스트) | `STRATEGY.md` + `DEVELOPMENT.md` |
| 6단계 (자동매매) | `STRATEGY.md` + `AI_PROMPTS.md` + `OPERATIONS.md` |

---

## 개발 로드맵

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

## AI 실무 운영 가이드

### 추천 방식 (이렇게 하면 잘 작동합니다)

| 작업 유형 | 문서 조합 | 예시 지시 |
|-----------|-----------|-----------|
| 설계 검토 | `CLAUDE.md` + 해당 `docs/` 1~2개 | "3-AI 합의 아키텍처를 검토해줘" |
| 코드 생성 | `ARCHITECTURE.md` + `DEVELOPMENT.md` + 해당 모듈 문서 | "KRX 수집기 Go 코드를 작성해줘" |
| 분석/브리핑 | `AI_PROMPTS.md` 해당 블록만 발췌 | "0-4 브리핑 프롬프트로 브리핑 생성해줘" |
| 자동매매 | `STRATEGY.md` + `OPERATIONS.md` + `AI_PROMPTS.md` | "Guard 계층 코드를 구현해줘" |
| 코드리뷰 | `DEVELOPMENT.md` + 해당 모듈 문서 | "이 코드가 리스크 규칙을 위반하는지 검토해줘" |

### 비추천 방식 (이렇게 하면 성능이 떨어집니다)

- PROMPT.md 전체를 통째로 넣고 아무 작업이나 시키기 (토큰 낭비 + 잡음)
- CLAUDE.md만 주고 상세 구현까지 기대하기 (인덱스일 뿐, 상세 없음)
- 전략 문서 없이 주문 로직 작성시키기 (리스크 상수 누락 위험)
- 운영 문서 없이 자동매매 실행 로직 작성시키기 (장애 대응 누락)
- "전체 시스템 다 구현해" 같은 광범위 요청 (AI가 규칙을 놓침)

### 핵심 원칙

> 문서를 잘 쓴 것만으로는 부족하다.
> **작업별로 필요한 문서만 골라서 주는 운영 방식**이 같이 따라가야 AI가 잘 작동한다.

---

## 변경 이력

| 버전 | 날짜 | 주요 변경 |
|------|------|-----------|
| v1.0 | 2026-03-09 | 초기 설계 완료 |
| v1.1 | 2026-03-09 | 문서 분리: CLAUDE.md → docs/*.md (6개 파일) |
| v1.2 | 2026-03-09 | Source of Truth 명확화, 실무 운영 가이드 추가, DoD/샘플 데이터 보강 |
| v1.4 | 2026-03-12 | 기술 스택 필수/선택/보류 3단 재정의, 시스템 범위 결정(최소운영형/확장형/제품형), KIS 공식 레포 활용 방침 추가 |
| v1.3 | 2026-03-12 | 현재 구현 범위 명확화, KRX 이상거래 감시 대응, Hashkey/WS접속키, 이벤트 기반 매매, 법적 주의사항, 소명 Audit Log 규격 추가 |

---

*Last Updated: 2026-03-09*
*Project: AlphaRadar (KORUS)*
