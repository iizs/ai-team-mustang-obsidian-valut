# Architecture Decision Records (ADR)

주요 기술/설계 결정과 그 배경을 보존한다. **결정 시점의 컨텍스트와 트레이드오프**를 기록해 6개월 후에도 "왜 이렇게 했나"를 이해할 수 있게 한다.

## 작성 규칙

- 파일명: `NNNN-slug.md` (예: `0001-normal-vs-bare-repo.md`)
- 번호는 **영구 ID**. 폐기 결정은 상태를 `Superseded`로 바꾸고 새 ADR을 추가
- 한 결정 = 한 파일

## 템플릿

```markdown
# ADR-NNNN — <제목>

- **상태**: Accepted | Proposed | Superseded by ADR-XXXX | Rejected
- **결정일**: YYYY-MM-DD
- **결정자**: <이름들>

## 컨텍스트
무엇을 결정해야 했나. 어떤 제약/요구사항 하에서.

## (옵션) 검토한 대안
A안 / B안 / C안 비교.

## 결정
무엇을 선택했나. 단호하게 한 줄.

## 근거
왜 그 선택인지. 핵심 이유 2~5개.

## 트레이드오프
무엇을 포기했나. 알려진 한계.

## (선택) 영향 / 후속 작업
이 결정으로 인해 추가로 해야 할 일.
```

## 목록

| ID | 제목 | 상태 |
|----|------|------|
| [0001](0001-normal-vs-bare-repo.md) | Wiki Store: Normal vs Bare Git Repo | Accepted |
| [0002](0002-source-store-not-tracked.md) | Source Store: git 비추적 | Accepted |
| [0003](0003-mvp-file-formats.md) | MVP 지원 파일 형식 | Accepted |
| [0004](0004-litellm-abstraction.md) | LLM 추상화: LiteLLM | Accepted |
| [0005](0005-lint-deferred.md) | Lint: v0.2+로 보류 | Accepted |
| [0006](0006-fastapi-backgroundtasks.md) | 백그라운드 워커: FastAPI BackgroundTasks | Accepted |
| [0007](0007-casbin-future-rbac.md) | 권한 관리: MVP 직접 구현, v0.2+ Casbin | Accepted (방향성) |
| [0008](0008-llm-accumulates-lint-cleans.md) | "LLM은 축적, 삭제는 Lint" 원칙 | Accepted |
| [0009](0009-edit-single-page.md) | Edit Flow: 단일 페이지 컨텍스트 | Accepted |
| [0010](0010-log-md-no-request-text.md) | log.md 기록 범위: 요청 전문 미포함 | Accepted |
| [0011](0011-staged-llm-pipeline.md) | Ingest 파이프라인 다단계화 (Phase A) | Accepted |
| [0012](0012-phase-b-actions-and-wiki-command.md) | Phase B: Plan 액션 실행 + Wiki Command + Page Delete | Proposed |
