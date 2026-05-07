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
