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
