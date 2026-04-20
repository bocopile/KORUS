# AlphaRadar (KORUS)

## AI 기반 한미 주식 자동매매 + 모닝 브리핑 시스템

> **프로젝트 목표**:
> 매일 아침 경제 뉴스·지표 브리핑을 자동으로 받고,
> 한국(KRX) + 미국(NYSE/NASDAQ) 시장 데이터를 수집·분석하여
> 3-Tier 포트폴리오 전략으로 AI 기반 자동매매를 구현한다.

---

## 1. 문서 안내

> **이 문서는 KORUS 프로젝트의 요구사항 개요입니다.**
> 상세 설계는 `docs/` 디렉토리의 개별 문서에 정의되어 있습니다.
> 규칙 충돌 시 우선순위: **운영/안전(OPERATIONS.md) > Guard(AI_PROMPTS.md 7-6-5) > 전략(STRATEGY.md) > 프롬프트(AI_PROMPTS.md)**

| 항목 | 내용 |
|------|------|
| 문서 목적 | 프로젝트 요구사항 개요 (상세 설계는 `docs/*.md`) |
| 대상 독자 | 개발자(본인), LLM(Claude/Gemini/ChatGPT — 코드 생성/분석 시 입력) |
| Source of Truth | `PROMPT.md`(요구사항) → `CLAUDE.md`(인덱스/라우터) → `docs/*.md`(상세 설계) |
| 사용 방식 | 전체 설계 참고용. LLM 입력 시 `CLAUDE.md` 매핑표를 보고 필요한 `docs/*.md`만 발췌 전달 |

**문서 구조:**

| 문서 | 내용 | 원본 장 |
|------|------|---------|
| `PROMPT.md` (이 문서) | 프로젝트 개요, 로드맵, 요구사항 | 1~2장 |
| `CLAUDE.md` | 인덱스/라우터 + 단계별 문서 매핑 | - |
| `docs/ARCHITECTURE.md` | 디렉토리, 데이터 흐름, 3-AI, 스케줄링, DB | 3~5장 |
| `docs/STRATEGY.md` | 3-Tier 배분, 리스크 상수, Tier별 진입/청산 | 6장 |
| `docs/AI_PROMPTS.md` | 단계별 AI 프롬프트, Guard 계층, 자동매매 | 7장 |
| `docs/REPORTS.md` | 일간/월간/연간 리포트, 알림 트리거 | 8~9장 |
| `docs/DEVELOPMENT.md` | Go 코드 원칙, 디버깅/코드리뷰 | 10~11장 |
| `docs/OPERATIONS.md` | 운영 수칙, 보안, 장애 대응, 참고 링크 | 12~13장 |

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

**핵심 운영 원칙:**
- **Confidence Score**: 3-AI 합의 결과는 `confidence_score: 0.0~1.0` float64로 수치화. `≥0.75` 실행 / `0.60~0.74` SOFT_WARN / `<0.60` skip.
- **Graceful Degradation**: AI 1개 장애 시 나머지 2개로 합의 진행. 2개 이상 장애 시 신규 매수 중단, 매도/손절은 지속(fail-open).
- **Swarm 합의**: MiroFish 엔진 미도입. 역할별 관점 분리(fundamental/news/risk) + 2차 반박 라운드는 Go `consensus.go`에 이식.
- **Knowledge Graph**: 현 단계 보류. PostgreSQL 안정화 후 Neo4j/Graphiti 사이드카 재검토 (8단계~).

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


> **상세 설계는 `docs/` 디렉토리에 분리되어 있습니다.**
> `CLAUDE.md`가 인덱스/라우터 역할을 하며, 단계별로 필요한 문서를 안내합니다.
> Claude Code CLI는 `CLAUDE.md`를 자동 로드합니다.
>
> LLM에 작업을 지시할 때는 이 문서 전체를 넣지 말고,
> `CLAUDE.md`의 매핑표를 참고하여 **해당 단계에 필요한 `docs/*.md`만 선택 전달**하세요.