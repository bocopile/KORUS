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
- **규칙 우선순위**: 운영/안전 규칙(12장) > Guard 규칙(7-6-5) > 전략 규칙(6장) > 프롬프트(7장)

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

## 변경 이력

| 버전 | 날짜 | 주요 변경 |
|------|------|-----------|
| v1.0 | 2026-03-09 | 초기 설계 완료 |
| v1.1 | 2026-03-09 | 문서 분리: CLAUDE.md → docs/*.md (6개 파일) |

---

*Last Updated: 2026-03-09*
*Project: AlphaRadar (KORUS)*
