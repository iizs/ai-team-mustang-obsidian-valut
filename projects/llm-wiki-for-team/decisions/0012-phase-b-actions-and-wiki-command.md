# ADR-0012 — Phase B: Plan 액션 실행 + Wiki Command + Page Delete

- **상태**: Proposed (v0.3.1 킥오프 검토 중)
- **결정일**: 2026-05-09 (예정)
- **결정자**: Kirin, Roy (+ Breda/Hawkeye 합동 검토)

## 컨텍스트

[ADR-0011 Phase A](0011-staged-llm-pipeline.md) 도입 후:
- Ingest LLM이 plan을 출력하고 `create` 액션은 즉시 실행됨
- `merge_into`/`supersede`는 skip + log 표시만 (Phase A 범위)
- 사용자 분리/통합 요청은 단일 페이지 Edit으로 표현 불가능 (다중 페이지 영향)

Phase B는 (A) Plan 액션을 실제 실행하고, (B) 사용자가 자연어 명령으로 다중 페이지 작업을 요청할 수 있는 진입점을 제공하며, (C) 페이지 삭제 라이프사이클을 도입한다.

## 결정

### A. Plan 액션 실행

- **`merge_into`**: 기존 페이지 read → LLM이 `merged_content`를 통합본으로 반영 → 덮어쓰기 + git commit + log "executed: merge_into → [[target]]"
- **`supersede`**: 기존 페이지 read → LLM이 새 본문으로 대체 → 덮어쓰기 + git commit + log "executed: supersede → [[target]] (reason: ...)"
- **`create` 충돌 자동 강등**: Phase A에서는 `merge_into`로 강등 후 skip이었으나, Phase B에서는 강등된 `merge_into`를 정상 실행

### B. Wiki Command (다중 페이지 작업 진입점)

- 신규 Job 타입 **`WIKI_COMMAND`**
- payload: `{command_text: <자연어 명령>}`
- 처리 흐름:
  1. LLM에 명령 + `index.md` 컨텍스트 전달
  2. LLM이 Plan(JSON, INGEST와 동일 스키마) 생성
  3. Plan의 액션들을 순차 실행 (create/merge_into/supersede/delete)
  4. log.md/index.md 갱신
- UI: 별도 탭 또는 페이지 (예: `/command`) — nav 진입점, 자연어 입력창
- 사용자 승인 절차: **도입 안 함** (v0.4+ 검토; 도입 시 웹소켓/이슈트래킹 수준 복잡도 필요)

### C. Page Delete

- Plan 액션 신규 **`delete`**: payload `{action: "delete", target: "<existing-page>.md", reason: "..."}`
- 처리: 파일 삭제 + git commit + log "executed: delete → [[target]] (reason: ...)"
- 삭제 후 `[[삭제된페이지]]` 클릭은 v0.2 SC-39 안내 페이지로 자연 처리 (404 톤)
- redirect: 도입 안 함 (Kirin 결정)
- Lint(B-6)가 사후 정리: 삭제된 페이지 참조하는 다른 페이지의 `[[링크]]` 정리는 Lint 단계

### D. Job/Plan 모델 (단일 job + 다중 액션 구조)

- 한 job이 plan(다중 액션)을 보유 (현재 INGEST와 동일 구조 확장)
  - sub-job 모델 도입 안 함 (Kirin 결정 — 운영 필요성 발생 시 v0.4+에서 검토)
- `Job.payload`에 plan + 액션별 status (executed/failed/skipped) 보존 (현재 구조 그대로)
- Job 상태는 기존(`pending → processing → done | failed`) 유지
  - `awaiting_approval` 도입 안 함

## 근거

- **현재 Phase A 구조 일관성**: INGEST가 이미 1 job : N pages로 동작. Wiki Command/Phase B도 같은 추상화 재사용해 변경 최소화.
- **원자성**: 진짜 원자성은 git이 보장 못 함. plan_summary의 액션별 status로 디버깅/추적 충분.
- **사용자 승인 보류**: 다이어로그/플로우 복잡도가 매우 큼. 자연어 명령 → 즉시 실행이 v0.3.1 가치 검증에 충분.
- **redirect 미도입**: 데이터 모델 단순 유지. 안내 페이지는 이미 SC-39로 존재.

## 트레이드오프

- **사용자 실수 시 영향 범위**: 자연어 명령이 다중 페이지를 변경/삭제하면 즉시 반영. 되돌리려면 git 직접 복구. 운영 초기엔 `log.md`로 추적성 확보가 안전망.
- **단일 job 다중 액션의 원자성 부족**: 5개 액션 중 3번째 실패 시 1~2번 액션은 이미 commit. 부분 재시도는 plan_summary 기반 manual.
- **Wiki Command JSON output 신뢰성**: Ingest와 동일하게 LiteLLM Ollama bug로 우회 적용. 안전망(prompt + Pydantic + graceful degrade) 동일하게 작동.

## 영향 / 후속 작업

- `models/job.py` — `JobType.wiki_command` 추가
- `routes/wiki.py` 또는 신규 `routes/command.py` — POST `/api/wiki/commands` 엔드포인트
- `services/pipeline.py` — `run_wiki_command(command_text, db, job_id)` 신규, `_resolve_action`이 merge/supersede/delete 정상 실행 분기
- `services/plan.py` — `DeleteAction` Pydantic 모델 추가
- `prompts/wiki_command.txt` 신규 (자연어 명령 → Plan 변환 가이드)
- `frontend/src/app/command/page.tsx` 신규 또는 nav input
- `frontend/src/lib/api.ts` — `submitWikiCommand(text)` 추가
- 신규 SC들 — Phase B 액션 정상 실행, WIKI_COMMAND 흐름, delete 동작, log 라인 통일

## 보류 (v0.4+)

- 사용자 승인 절차 (Plan Review 화면) — 웹소켓/이슈트래킹 수준 복잡도 도달 시점 재검토
- redirect 기능
- Sub-job 모델 (액션 단위 재시도/병렬화 필요성 발생 시)
