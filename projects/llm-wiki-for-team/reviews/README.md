# Reviews

Breda(Developer) / Hawkeye(Evaluator) 리뷰의 영구 보관소.
Discord에서 휘발될 리뷰 결과를 합의된 시점에 vault로 옮긴다.

## 작성 규칙

- 파일명: `YYYY-MM-DD-<who>-<scope>.md`
  - 예: `2026-05-06-hawkeye-phase1-validation.md`, `2026-05-07-breda-phase3-completion.md`
- `who`: `breda` | `hawkeye` | `breda+hawkeye`
- `scope`: 짧은 한 단어 또는 두 단어 조합 (`phase1-validation`, `spec-v0.6`, `parser-hardening`)

## 템플릿

```markdown
# <Title>

- **일자**: YYYY-MM-DD
- **주체**: Hawkeye (또는 Breda)
- **대상**: <대상 SPEC/SC/커밋>
- **상태**: PASS | PARTIAL | FAILED | Advisory only

## 요약

핵심 결과 1~2줄.

## 결과

대상별 상세. 표 또는 항목 리스트.

## Blocking 이슈

(있다면 항목 + 권고)

## Advisory 관찰

(비블로킹 항목, 향후 검토 메모)

## 후속 작업

- 패스 받은 사람 / 다음 액션
```

## v0.1 (요약 보존)

- [v0.1 통합 요약](2026-05-07-v0.1-completion-summary.md) — 3 Phase 검증 + 핫픽스 검증 통합

상세 Discord 원본은 휘발됐으나, 핵심 결정과 결과는 위 요약에 정리되어 있다.

## v0.2

- [v0.2 통합 검증](2026-05-08-hawkeye-v0.2-validation.md) — Hawkeye, SC-20+SC-31~40 PASS, Advisory 1건, 사후 발견 1건(DB 마이그레이션)

## v0.3 (Phase A) + Postmortem

- [Prompt 로드 버그 + LiteLLM Ollama 호환성 Postmortem](2026-05-08-prompt-load-bug-postmortem.md) — Roy, v0.1부터 잠재된 silent prompt failure가 v0.3 실사용에서 드러남. v0.1~v0.2 회고의 "weak 모델 instruction following" 평가 무효화. B-21 백로그 신설 근거.

## v0.3.1 (Phase B)

- [v0.3.1 통합 검증](2026-05-09-hawkeye-v0.3.1-validation.md) — Hawkeye, SC-52~63 PASS, Advisory 2건 (SC-54/SC-58 표현 차이) → SC 본문 정리로 흡수

## v0.4 (Agentic + Backlinks)

- [v0.4 통합 검증 + 핫픽스 시리즈](2026-05-14-hawkeye-v0.4-validation.md) — Hawkeye, SC-64~80 PASS (1차 SC-71/78 Blocking → 2차 PASS), 후속 핫픽스 7건 모두 회귀 영향 없이 흡수
- [v0.4 실사용 피드백 (Kirin)](2026-05-15-v0.4-empirical-feedback.md) — INGEST 큰 source 처리 + WIKI_COMMAND 시나리오 1~5 실측, prompt caching/tier limit/rename backlinks 등 발견 + ADR-0016 도출
