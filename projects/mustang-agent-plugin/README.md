# mustang-agent-plugin (설계 초안)

_2026-09-15 초안 · 미구현_

Claude Code 플러그인 + 세션-측 skill 조합으로 **에이전트가 task-hub 이벤트를 자동 수신하고 Dooray truth에 따라 처리하는 runtime**. [[projects/team-operations-rework/README]] Track A4/A5의 실체.

## 목적

**세션 시작 즉시**, 모델 개입 없이 결정론적으로:
- Dooray 이벤트 실시간 수신 채널 활성화 (WebSocket monitor via mustang-task-hub receiver).
- 이후 세션-측 skill이 lifecycle(drain → monitor → triggered re-check)을 진행.

## 결정 사항 (확정)

- **Queue 도구 도입 X** — Dooray가 곧 큐 (source of truth). 우리는 이벤트 배달 채널 + 재확인만.
- **Plugin scope = Monitor auto-start 만** (얇게). Queue 조작 도구 · MCP 서버 X.
- **세션-측 로직은 skill 로 캡슐화** (매니페스트 X, 지시 지향). CLAUDE.md 는 skill을 언제 호출할지만 안내.
- **Agent 이름** = env `$JOURNAL_AGENT_NAME` (launcher 주입) — Dooray userCode와 일치 규약.
- **Tunnel URL** = env `$TASK_HUB_WS_URL` (launcher 주입) — Quick Tunnel면 URL 바뀔 때마다 갱신 필요.

## Plugin 매니페스트 설계

**공식 문서 요약** (`code.claude.com/docs/en/plugins-reference` `#monitors`, experimental):
- 위치: `monitors/monitors.json` 또는 `plugin.json` 인라인.
- 각 monitor: `{name, command, description, when?}`.
- `command` 만 지원 (WebSocket `ws:` 지원 여부는 문서에 명시 없음 — Monitor tool 자체는 지원하나 plugin 매니페스트에선 미확인).
- Env 확장 지원: `${TASK_HUB_WS_URL}` 등.
- 시작 시점: `when=always` (default) — 세션 시작 · `/reload-plugins` 시.
- 여러 monitor 선언 가능.
- 실패 시 동작(fail-load vs silent)은 미문서 — 실증 필요.

### 확정: Path Y (command + WS consumer 스크립트)

**Path X 는 실증 결과 스키마 거부**:
```
[ERROR] Failed to load monitors for test-ws-monitor …
  { "expected": "string", "code": "invalid_type", "path": [0, "command"] }
  { "code": "unrecognized_keys", "keys": ["ws"] }
```
Plugin monitor 매니페스트는 `command` 만 인식. `ws:` 는 unrecognized_keys. 실패 시 monitor 만 조용히 drop (플러그인 다른 요소는 계속 로드) — 확인 방법은 `--debug-file` 로 로그 fetch.

**Path Y 확정**:
```json
[{
  "name": "task-hub-events",
  "command": "python3 \"${CLAUDE_PLUGIN_ROOT}/scripts/ws-consumer.py\" \"${TASK_HUB_WS_URL}\"",
  "description": "task-hub events for me"
}]
```
번들 스크립트가 WS 클라이언트로 붙어 각 프레임을 **stdout 한 줄로 emit**. Monitor tool이 각 stdout 라인을 notification으로 세션 컨텍스트에 삽입.

**의존성**: 호스트에 Python 3.10+ + `websockets` 패키지. Plugin 설치 시 요구사항 안내 (README).

**Consumer 스크립트 실증 (2026-09-15)**: 별도 `ws_stdout_consumer.py` 로 로컬 receiver의 WS에 붙어 시뮬 payload → 한 줄 요약 JSON stdout emit 성공. Plugin 안에 이 스크립트를 담고 `command`로 지정하면 완결.

**자동 재연결**: consumer가 예외 시 5초 sleep 후 재접속 loop. Receiver 재시작에도 자연 복구.

### 플러그인 파일 구조 (Path Y 기준)

```
mustang-agent-plugin/
├── .claude-plugin/
│   └── plugin.json                 # 매니페스트 (experimental.monitors 경로 지정)
├── monitors/
│   └── monitors.json               # WS consumer 실행 정의
├── scripts/
│   └── ws-consumer.py              # 각 event 프레임을 line으로 emit
├── skills/
│   └── task-hub-loop/
│       └── SKILL.md                # 세션-측 lifecycle skill
├── README.md
└── LICENSE
```

## 세션-측 skill: `task-hub-loop`

### 언제 호출되나

**Trigger 1 — 세션 시작 직후** (CLAUDE.md 안내): 세션 열리자마자 skill 호출 → drain phase 실행.

**Trigger 2 — Monitor notification 도착** (Claude Code가 자동으로 컨텍스트에 삽입): 다음 turn 에서 모델이 이벤트 확인 → skill 호출 → triggered re-check.

**Trigger 3 — 주기적 wake-up** (안전망, `/loop` dynamic 20~60분): Monitor가 조용할 때 fallback으로 drain phase 재실행.

### Lifecycle (Kirin 초안 정련)

```
[start / plugin activation]
  ↓
 (a) [drain phase]
  1. Dooray API 호출: 내가 assignee인 task 목록
     filter: workflow_class in {registered, working}
  2. 각 task를 "액션 필요" 판단 (아래 룰)
  3. 액션 필요 task 없음 → (b) monitor phase
  4. 액션 필요 task 있음 → 우선순위 정렬 → 첫 번째 처리 → step 1로 반복
     (매 처리 후 재조회 = 새 이벤트 자연 흡수)
  ↓
 (b) [monitor phase]
  Plugin이 auto-start한 Monitor가 wss://.../events/<me> 리스닝.
  세션은 idle 상태. 새 프롬프트도 없고 처리할 task도 없음.
  ↓ (Monitor notification 도착 OR loop wake-up)
 (c) [triggered re-check]
  · notification 이 온 경우:
      payload의 post_id로 특정 task 조회 → 액션 필요? → 처리
      (여러 태스크가 동시 관여된 이벤트면 각각 판단)
  · loop wake-up 인 경우 (안전망):
      (a)와 동일한 drain phase 재실행
  ↓
 처리 후 → (a)로 복귀 (drain phase = 잔여 확인)
```

**드레인 phase 를 매 처리 후 반복**하는 이유: task 처리 사이에 새 이벤트가 오면 자동 흡수 (별도 큐 없이 Dooray가 truth).

### "액션 필요" 판단 룰

담당자 필드에 나(agent)가 있는 task 중:

1. `workflow_class == registered` (신규 할당): **액션 필요** (초기 반응 · 접수 코멘트 등).
2. `workflow_class == working`:
   - 마지막 코멘트/변경이 **내가 아닌 다른 사용자** → **액션 필요** (응답 필요).
   - 마지막 코멘트/변경이 나 자신 → **액션 없음** (대기 상태).
3. `workflow_class == closed`: **액션 없음**.

**개별 담당자 상태(assignee-specific workflowId)** 도 별도 확인 필요:
- 내가 아직 특정 workflowId를 안 완료했다면 (예: 다른 담당자만 완료) 여전히 액션 필요일 수 있음. 실제 payload에서 `post.users.to[].workflowId` 로 판단.

### 우선순위 규칙 (신규)

다중 task drain 시 정렬 축, 상단이 우선:

1. **Overdue 여부** (dueDate < now): overdue → 최우선.
2. **Priority 필드**: `highest > high > normal > low > lowest > none`.
3. **Tie-break — 생성 시각 오름차순** (오래된 것 먼저, FIFO).

미래 확장 옵션 (초기엔 skip):
- 특정 tag(예: `urgent`)가 붙으면 부스트.
- 특정 assigner가 보낸 task 부스트 (예: Kirin의 지시 우선).
- Milestone 기반 정렬.

### Idempotency

Dooray truth 원칙 그대로:
- 처리 전에 항상 최신 Dooray 상태 fetch.
- 이미 반응한 task는 Dooray 코멘트 이력에 나 자신 마지막 활동이 있어 액션 필요 룰 2에서 자동 skip.
- 명시적 "이미 처리한 event_id" 로컬 캐시 X.

### 실패 처리

- Task 처리 중 예외 발생 → Dooray에 실패 코멘트 남기고 loop 계속. 자동 재시도 X (같은 원인으로 반복 실패 우려). 사람 escalation은 실패 코멘트가 사람 눈에 띔.
- 처리 시간이 세션 rate limit · context window 압박 → 별도 세션에서 재개.

## 환경변수

**Agent session 쪽 (launcher가 세팅)**:

| 변수 | 필수 | 설명 |
|---|---|---|
| `TASK_HUB_WS_URL` | ✓ | Plugin monitor가 붙을 WS URL. 예: `wss://<tunnel>/events/roy`. |
| `JOURNAL_AGENT_NAME` | ✓ | 이미 존재. Dooray userCode와 일치 규약. |
| `DOORAY_API_KEY` | ✓ | 이미 존재. Dooray API 호출용. |

Skill 안에서 agent 이름 필요 시 `$JOURNAL_AGENT_NAME` 참조.

## Env 미설정 시 정책

- `TASK_HUB_WS_URL` 없음 → plugin monitor 시작 실패 (또는 즉시 exit). Skill은 여전히 CLAUDE.md 지시로 호출됨 — monitor phase 진입 못하고 계속 drain-only 동작.
- 실증으로 정확한 실패 시나리오 관찰 필요.

## 실증 · 검증 계획

1. **Path X 시도 → 실패 시 Path Y로 fallback**.
2. Roy 세션에서 plugin 활성화 → `/health` 로 receiver 쪽에 `ws_connections: {roy: 1}` 확인.
3. Kirin이 웹UI에서 Roy 태스크에 코멘트 → Monitor notification → skill이 액션 판단 → Dooray에 응답 코멘트.
4. Session 재시작 → drain phase가 backlog 흡수 확인.
5. `/loop` dynamic wake-up 실증 (안전망).

## 결정 기록

- **2026-09-15** 초안 작성. Plugin scope 얇게 (Monitor only). Queue 도구 기각. Lifecycle 4단계, 우선순위 3축(overdue > priority > 생성시각) 정의.
- **2026-09-15** Path Y 확정. Plugin monitor 스키마가 `ws:` 를 unrecognized_keys 로 거부하는 실증 완료 (`--debug-file` 로그로 확인). 실패 시 monitor 만 조용히 drop.
- **2026-09-15** `ws_stdout_consumer.py` 프로토타입 실증. WS 붙어서 매 프레임 → stdout 한 줄 요약. 재접속 loop 포함.
- **2026-09-15** 사람 계정 필터는 receiver 층에서 처리 확정 (Dooray가 사람에게 이메일/앱 알림). Receiver env `TASK_HUB_HUMAN_AGENTS=kirin` 도입 (mustang-task-hub `c9e587f`). Plugin 쪽엔 관련 로직 없음.

## 미결 · 확인 대기

- **`task-hub-loop` skill 위치**: 플러그인 내부 (`skills/task-hub-loop/SKILL.md`) vs 기존 `~/.claude/skills/`. 플러그인 내부가 정합 (일괄 배포). 최종 확정 대기.
- **Skill 호출 트리거를 CLAUDE.md에서 얼마나 명시하는지**: "세션 시작 시 반드시 호출" 강제 문구 필요할지, 아니면 Monitor notification 도착 자체가 자연스러운 트리거인지.
- **처리 예외 시 Dooray 코멘트 포맷 표준화** (에러 유형 · 재시도 안 함 안내 등).
