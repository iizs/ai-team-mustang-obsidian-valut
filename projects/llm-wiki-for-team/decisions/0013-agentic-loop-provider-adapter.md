# ADR-0013 — Agentic Loop + Provider Adapter 전환 전략

- **상태**: Accepted (v0.4 킥오프 — 2026-05-14)
- **결정일**: 2026-05-14
- **결정자**: Kirin, Roy (+ Breda/Hawkeye 합동 검토)

## 컨텍스트

v0.3.1까지 만든 Plan 모델은 "LLM에 한 번 보내고 한 번에 받은 plan을 실행"하는 정적 구조다. v0.3.1 실측과 Kirin이 별도로 Claude Code 콘솔로 위키 구축을 시도한 경험에서 다음 한계가 드러났다:

1. **거의 모든 위키 작업이 다중 페이지 영향** — 간단한 편집을 제외하면 분리/통합/표제어 변경 등은 페이지 라이프사이클을 건드림
2. **정적 plan은 의도 misread에 취약** — LLM이 사전 정보(`index.md` 한 줄 요약만)에 의존해 짠 plan은 실제 페이지 내용을 모르고 결정한 것
3. **현 흐름은 "action → LLM 생성 → 저장"의 단순 패턴** — Claude Code처럼 "분석 → 계획 → 행동 → 반복"이 본질에 더 가까움

위 문제를 해결하려면 **tool-use agentic loop**로 전환이 필요하다. 동시에 v0.3에서 드러난 LiteLLM Ollama provider 호환성 이슈([B-21](../backlog.md#b-21-ollama-llm-호환성-우회-제거-경로))는 provider 추상화 자체에 대한 재검토를 요구한다.

## 검토한 옵션

### A. 멀티 provider 유지 + LiteLLM 위에 직접 agentic loop
- v0.3과 일관, lock-in 없음
- Ollama tool calling 호환성 검증/우회 부담이 큼 (v0.3 JSON 강제 우회 패턴 재현 우려)
- v0.4 핵심 가치(agentic loop)와 무관한 작업에 시간 잡아먹음

### B. Anthropic-only + Claude Agent SDK
- 가장 빠른 v0.4 (loop + 도구 + 세션·hook·permission 인프라 무료)
- SDK lock-in 강함, Ollama/멀티 provider 일시적 포기
- v0.5+에서 Ollama로 다시 가려면 거의 v0.4 재작성

### C. Anthropic-first + LiteLLM 추상화 유지
- LiteLLM tool calling 위 직접 loop, Anthropic만 검증
- 단기 효율적
- LiteLLM 추상화 한계는 그대로 (Anthropic best practice 손실)

### D. Provider별 Adapter 분기 (처음부터)
- `LLMAdapter` 인터페이스 + `AnthropicNativeAdapter` + `LiteLLMAdapter`
- 장기 깨끗함, lock-in 없음, best practice 활용
- v0.4 작업량 +α, 인터페이스 설계 시 잘못 잡으면 v0.5+에서 깨질 위험 (Kirin 경험)

## 결정

**B로 시작 → 학습 후 D로 refactor (Kirin 결정).**

단, **Claude Agent SDK는 채택하지 않고 Anthropic SDK 직접 사용 + agentic loop 직접 구현**. Agent SDK lock-in을 처음부터 피하고 D로의 refactor를 용이하게 함.

### v0.4 범위 (Anthropic-only, Kirin 결정 2026-05-14)
- `anthropic` 패키지 직접 사용 (Messages API + tool use)
- Agentic loop 직접 작성 (`call → tool_calls 처리 → 결과 append → 반복`)
- **Anthropic 외 provider 호출 시 Job FAILED + 명시 에러** ("This provider does not support agentic mode (v0.4). Use Anthropic."). v0.3.1의 LiteLLM 단순 chat 흐름은 v0.4 코드베이스에서 **제거**.
- `pipeline.py`/`llm_client.py`는 *나중에* Adapter 인터페이스를 추출할 수 있도록 응집도 높게 작성. 단 v0.4에 추상화 인터페이스 도입은 안 함.

> **Kirin 결정 근거**: v0.4는 agentic 흐름의 가치 검증이 핵심. fallback 코드 유지는 dual-path 부담만 추가하고, Ollama 사용자는 v0.3.1 인스턴스를 유지하는 편이 깨끗함. v0.5+에서 OllamaNativeAdapter 도입 시 진짜 multi-provider 복귀.

### v0.5+ 범위 (D로 refactor)
- `LLMAdapter` 인터페이스 추출
- `AnthropicNativeAdapter` (현 v0.4 코드를 클래스화)
- `OllamaNativeAdapter` (httpx 직접 또는 LiteLLM 우회 — B-21 결정 반영)
- `LiteLLMAdapter` (기타 provider fallback)

## 근거

- **B 시작은 학습 효과**: agentic 패턴은 처음 만들 때 인터페이스를 잘못 잡기 쉬움. Anthropic 단일 케이스로 구현하면서 발견한 abstraction을 v0.5에서 D로 정착시키는 게 안전.
- **Claude Agent SDK 회피 이유**: SDK가 제공하는 풀스택(Bash/Read/Edit 일반 도구, MCP, hooks 등)은 우리 도메인에 거의 무용 + v0.5 refactor 시 의존성 제거 비용 큼.
- **v0.4 Anthropic-only**: fallback 코드 유지의 dual-path 부담을 피하고 agentic 흐름 가치 검증에 집중. Ollama 사용자는 v0.3.1 인스턴스를 유지. v0.5+에서 진짜 multi-provider 복귀.

## 추가 결정 (셋이 합의 + Kirin 확정)

### Loop 종료 / 안전장치 (확정 임계값 — Kirin 결정 2026-05-14)
- 자연 종료: LLM이 tool 호출 안 함 (`stop_reason == "end_turn"`)
- 강제 종료:
  - `max_iterations = 20`
  - `max_tool_calls = 50`
  - `timeout = 300s` (5분)
  - `stop_reason == "max_tokens"` → abort + Job FAILED (context limit)
- 사용자 취소: `POST /api/jobs/{id}/cancel` → Job.status = `cancelling` → 다음 iteration 진입 전 graceful abort → `cancelled`
- **이상 패턴 감지**: 같은 (tool_name, args hash) 연속 3회 호출 시 abort (무한 루프 방지)
- **parallel tool_use**: Anthropic은 동시 호출 가능하지만 v0.4에선 **순차 처리**로 단순화 (race 회피)

### Job 모델 확장
- **`JobStatus.cancelled` 신규** — `failed`와 구분. `cancelling`은 진행 중 마킹용 (DB에는 즉시 반영 → 다음 iteration 진입 시 reader가 보고 abort)

### INGEST + WIKI_COMMAND 통합
- 내부적으로는 모두 동일한 agentic loop ("wiki command"). UI/UX 측면에서 INGEST 진입점(파일 업로드)은 분리 유지.
- v0.3.1의 정적 Plan 모델 (`PlanAction` Pydantic 등)은 제거됨. `Job.payload.steps`에 tool 호출 history 누적.

### Lint
- **v0.4 제외**. 같은 agentic loop를 별도 트리거로 실행하면 되는 구조라 v0.5+에서 작은 추가로 도입 가능.

### UI 진행 가시화 / 취소
- 매 step DB checkpoint (`Job.payload.steps`에 `{step_no, tool_name, args, result_snippet, ts}` push)
- Jobs 탭 polling으로 진행 표시
- 취소 버튼 → `status=cancelling` → 다음 iteration 진입 시 graceful abort

### Dry-run 모드
- **v0.5+로 미룸**. 도중 취소가 있어서 안전망 충분.

## 트레이드오프

- **LLM 비용 ↑↑**: agentic loop는 N회 호출. Anthropic 비용 부담.
- **결정성 ↓**: agent가 매번 같은 입력에 같은 액션 안 할 수 있음 → 자동 회귀 테스트는 mock 기반 + Hawkeye 정적 검증 중심
- **v0.4 동안 Ollama는 단순 흐름**: 사용자 일부 기능(다중 페이지 작업) 미지원. 가이드로 명시.

## 영향 / 후속 작업

- `services/llm_client.py` — agentic loop 구현 (Anthropic SDK 직접)
- `services/pipeline.py` — `run_wiki_command()` 재작성 (Plan 폐기, agentic loop 호출)
- `services/plan.py` — 폐기 또는 step 모델로 진화
- `routes/wiki.py`, `routes/sources.py` — Job 등록 시 agentic 흐름으로 분기 (Anthropic provider인 경우)
- `models/job.py` — `payload.steps` 구조 정의
- 신규 ADR-0014 (Tool 카탈로그 + Commit/Log)
- 신규 ADR-0015 (Backlink Frontmatter Index)
- 신규 SC — agentic loop 안전장치, INGEST 통합, progress checkpoint, 취소, provider별 분기
