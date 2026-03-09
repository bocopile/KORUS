# AlphaRadar — 리포트 체계 / 알림 시스템 (8~9장)

> 원본: CLAUDE.md 8~9장에서 분리
> 관련 문서: [ARCHITECTURE.md](ARCHITECTURE.md), [STRATEGY.md](STRATEGY.md)

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
