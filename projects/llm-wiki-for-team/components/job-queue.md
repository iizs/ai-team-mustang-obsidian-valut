# Job Queue (비동기 처리)

모든 LLM 처리 작업은 즉시 실행이 아닌 Job으로 등록되어 백그라운드 워커가 순차 처리한다.

> 워커 구현 결정: [ADR-0006 — FastAPI BackgroundTasks](../decisions/0006-fastapi-backgroundtasks.md)

## Job 종류

| Type | 트리거 | payload |
|------|--------|---------|
| `INGEST` | 원본 파일 업로드 (신규/교체 구분 없음) | `{source_path}` |
| `EDIT` | 사용자 위키 수정 요청 | `{edit_text, page_path}` |

> 신규/교체 동일 처리 원칙: [ADR-0008](../decisions/0008-llm-accumulates-lint-cleans.md) — "LLM은 축적하지, 삭제 판단은 Lint에서"

## Job 상태

```
Pending → Processing → Done | Failed
```

## 데이터 모델 (SQLite `jobs` 테이블)

```
job_id | type | payload | status | created_by | created_at | updated_at | error_msg
```

## 동작 규칙

- 사용자는 업로드/요청 후 즉시 "큐 등록됨" 응답 받음 (HTTP 202)
- **Jobs 탭** (별도 UI): Job 목록 및 처리 상태 조회. Admin: 전체, Member: 본인 요청만
- 실패 시 `error_msg` 저장, Admin 재시도 가능
- MVP: 워커 1개 (순차 FIFO). FastAPI `BackgroundTasks` — 외부 의존성 없이 단일 프로세스로 처리
- LLM 출력이 파싱 불가하거나 평론형이면 명시적 FAILED 처리 (silent SUCCESS 금지)

## Known Limitation (v0.1)

- 원본 교체 시 원본에서 제거된 내용이 위키에 잔존할 수 있다. Lint(v0.2+) 도입 전까지 사용자가 직접 Edit 요청으로 수동 정리해야 한다.
- 위키 규모 증가 시 `index.md` 갱신에 LLM 비용이 매 Job마다 발생한다. v0.2에서 incremental 갱신 검토 예정.

## 관련 SC

SC-11 (등록 응답), SC-11-b (완료 후 commit), SC-17-a/b/c (Jobs 탭)
