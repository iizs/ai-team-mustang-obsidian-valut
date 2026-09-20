# mustang-hub-agent (에이전트 runtime plugin)

_2026-09-15 착수 · v0.1 실증 완료 · 2026-09-20 드레인 전략 변경 (agent-side → receiver-side)_

소스: [github.com/iizs/mustang-hub-agent](https://github.com/iizs/mustang-hub-agent) (private)

_기존 초안 이름 `mustang-agent-plugin` → 2026-09-15 receiver(`mustang-hub`) 리네임과 함께 `mustang-hub-agent` 로 정합화. Skill 이름도 `task-hub-loop` → `hub`._

Claude Code 플러그인 + 세션-측 skill(`hub`) 조합으로 **에이전트가 mustang-hub 이벤트를 자동 수신하고 Dooray truth에 따라 처리하는 runtime**. [[projects/team-operations-rework/README]] Track A4/A5 실체.

## 목적

**세션 시작 즉시**, 모델 개입 없이 결정론적으로:
- Dooray 이벤트 실시간 수신 채널 활성화 (WebSocket monitor via [[projects/mustang-hub/README|mustang-hub]] receiver).
- 이후 세션-측 `hub` skill 이 notification 을 소비해 태스크를 처리.

**드레인 담당 위치**: 서버(mustang-hub receiver)의 폴 드레인. 에이전트 skill 은 notification-driven only — 세션 시작 드레인 · 안전망 wake-up 개념 없음.

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
mustang-hub-agent/
├── .claude-plugin/
│   └── plugin.json                 # 매니페스트 (experimental.monitors 경로 지정)
├── monitors/
│   └── monitors.json               # WS consumer 실행 정의
├── scripts/
│   └── ws-consumer.py              # 각 event 프레임을 line으로 emit
├── skills/
│   └── hub/
│       └── SKILL.md                # 세션-측 lifecycle skill
├── README.md
└── LICENSE
```

## 세션-측 skill: `hub`

### 언제 호출되나

**Trigger — Monitor notification 도착**. Claude Code 가 자동으로 컨텍스트에 삽입한 `<task-notification>` 을 본 다음 turn 에 모델이 이벤트 확인 → `hub` skill 호출.

Notification 은 두 종류:
1. **Webhook 이벤트** — Dooray 실 이벤트 실시간.
2. **폴 드레인** (`hook_event=pollDrain`, `request_origin=poll`) — receiver 가 주기적으로 pending 을 훑어서 push. Webhook 누락 · 세션 다운 사이 발생분 회복 용도. Skill 관점에선 처리 로직 동일.

**세션 시작 드레인 · 주기적 wake-up 안전망 없음** — 드레인은 서버가 담당.

### 처리 흐름 (skill 안)

```
notification 도착
  ↓
 (컨텍스트에 <task-notification> 삽입)
  ↓
 다음 turn: hub skill 호출
  ↓
 project_id / post_id 로 dooray task-get (Dooray = truth)
  ↓
 "액션 필요" 룰 판단
  ↓                  ↓
액션 없음 (자기 활동  액션 필요
흔적 있음 등)         ↓
  ↓                  처리 → 코멘트 · workflow 이동 · assignee 재할당
 짧게 종료             ↓
                    turn 종료 (다음 notification 대기)
```

**Idempotency 는 skill 안에서 자동** — 매 호출마다 Dooray 재fetch → 자기 마지막 코멘트 · workflow 이동 흔적이 있으면 액션 필요 룰이 자동 skip. 폴 드레인이 동일 태스크를 반복 push 해도 이 지점에서 걸러진다.

### "액션 필요" 판단 룰

담당자 필드에 나(agent)가 있는 task 중:

1. `workflow_class == registered` (신규 할당): **액션 필요** (초기 반응 · 접수 코멘트 등).
2. `workflow_class == working`:
   - 마지막 코멘트/변경이 **내가 아닌 다른 사용자** → **액션 필요** (응답 필요).
   - 마지막 코멘트/변경이 나 자신 → **액션 없음** (대기 상태).
3. `workflow_class == closed`: **액션 없음**.

**개별 담당자 상태(assignee-specific workflowId)** 도 별도 확인 필요:
- 내가 아직 특정 workflowId를 안 완료했다면 (예: 다른 담당자만 완료) 여전히 액션 필요일 수 있음. 실제 payload에서 `post.users.to[].workflowId` 로 판단.

### 우선순위 규칙 (참고)

Notification 은 도착 순으로 처리되므로 강한 정렬 개념이 skill 안에 필요치는 않다. 다만 여러 notification 이 짧은 시간에 몰려 있을 때 어느 것부터 볼지의 상식적 기준:

1. **Overdue 여부** (dueDate < now).
2. **Priority 필드**: `highest > high > normal > low > lowest > none`.
3. **Tie-break — 생성 시각 오름차순** (FIFO).

미래 확장 옵션 (초기엔 skip): tag / assigner / milestone 기반 부스트.

### Idempotency

Dooray truth 원칙 그대로:
- 처리 전에 항상 최신 Dooray 상태 fetch.
- 이미 반응한 task 는 Dooray 코멘트 이력에 나 자신 마지막 활동이 있어 액션 필요 룰에서 자동 skip.
- 폴 드레인이 반복 push 해도 이 지점에서 걸러진다.
- 명시적 "이미 처리한 event_id" 로컬 캐시 X.

### 실패 처리 (원 지시자에게 반환)

Task 처리 중 예외 발생 시:

1. Dooray 코멘트 (정형): `[<agent>] 처리 실패 · <에러 요약> · <원인 힌트> · 반환합니다.`
2. **Assignee 를 원 지시자로 재할당** — `tasks.update(to=[post.users.from])`. 반환하지 않으면 에이전트 큐에 남아 무한 실패 loop.
3. Loop 계속 — 다음 drain에서 자신이 assignee 가 아니라 자동 skip (idempotency 확보).

**Edge case 방어**

| 조건 | 대응 |
|---|---|
| 지시자 == 나 (내가 나에게 만든 subtask 등) | 반환하면 무한 loop. **반환 skip + 코멘트만** 남기고 `workflowClass` 를 backlog 로 되돌려 사람 검토 요청. 코멘트에 "지시자==실행자라 반환 불가" 명시. |
| 지시자가 emailUser (외부 이메일이 task 생성) | `to=[{type: emailUser, emailUser: {emailAddress, name}}]` 형식으로 재할당 (Dooray API 지원). 코멘트에 "이메일 지시자에게 반환" 명시. |
| `post.users.from` 정보 부재 | 반환 불가 → 코멘트만 + backlog 이동 + 로그 warning. |
| 재할당 자체가 실패 (권한 · 네트워크) | 코멘트만 남기고 loop 계속. 재할당 실패는 별도 warning log. 재시도 없음 (무한 loop 방지). |

**자동 재시도는 하지 않는다** — 같은 원인으로 반복 실패 확률 높음. 사람 판단 후 재실행/폐기.

세션 rate limit · context 압박은 별도 세션에서 재개 (사람이 판단).

이 정책의 조직 층 규범은 [[projects/team-operations-rework/README|team-operations-rework]] "실패 처리 방침" 소절 참고.

## 환경변수

**Agent session 쪽 (launcher가 세팅)**:

| 변수 | 필수 | 설명 |
|---|---|---|
| `TASK_HUB_WS_URL` | ✓ | Plugin monitor가 붙을 WS URL. 예: `wss://<tunnel>/events/roy`. |
| `JOURNAL_AGENT_NAME` | ✓ | 이미 존재. Dooray userCode와 일치 규약. |
| `DOORAY_API_KEY` | ✓ | 이미 존재. Dooray API 호출용. |

Skill 안에서 agent 이름 필요 시 `$JOURNAL_AGENT_NAME` 참조.

## Env 미설정 시 정책

- `TASK_HUB_WS_URL` 없음 → plugin monitor 시작 실패 (또는 즉시 exit). Skill 은 여전히 CLAUDE.md 지시로 호출 가능하지만 notification 도착 채널이 없으니 실질 무용. Launcher 가 env 를 확실히 주입해야 함.

## 결정 기록

- **2026-09-15** 초안 작성. Plugin scope 얇게 (Monitor only). Queue 도구 기각. Lifecycle 4단계, 우선순위 3축(overdue > priority > 생성시각) 정의.
- **2026-09-15** Path Y 확정. Plugin monitor 스키마가 `ws:` 를 unrecognized_keys 로 거부하는 실증 완료 (`--debug-file` 로그로 확인). 실패 시 monitor 만 조용히 drop.
- **2026-09-15** `ws_stdout_consumer.py` 프로토타입 실증. WS 붙어서 매 프레임 → stdout 한 줄 요약. 재접속 loop 포함.
- **2026-09-15** 사람 계정 필터는 receiver 층에서 처리 확정 (Dooray가 사람에게 이메일/앱 알림). Receiver env `TASK_HUB_HUMAN_AGENTS=kirin` 도입. Plugin 쪽엔 관련 로직 없음.
- **2026-09-15** 리네임 · v0.1 스켈레톤 완성 (`iizs/mustang-hub-agent`). plugin manifest · monitors.json (Path Y) · scripts/ws-consumer.py · skills/hub/SKILL.md · README. `claude plugin validate` 통과, standalone consumer 실행으로 receiver `/events/roy` 연결 + fake payload 배달 + stdout 한 줄 요약 emit 실증. 실 세션에서 Monitor 자동 기동은 세션 재시작 필요.
- **2026-09-15** 실패 처리 정형 확정 (skill 안): Dooray 코멘트 + assignee 재할당(원 지시자) + edge case 4종 방어 (지시자==실행자, emailUser, from 부재, 재할당 실패).
- **2026-09-16** 실 세션 e2e 최초 실증. Kirin 웹UI 태스크 생성 → hub skill 자동 호출 → 처리 완결 확인. Payload assignee 위치 불일치 버그(4~5건 dropped_no_target) 발견 → parser fallback + API fallback 로 수정.
- **2026-09-16** Dooray 사용 규범 확정 — 라우팅은 assignee 만 사용, 반응 필요한 코멘트는 반드시 assignee 도 함께 변경 (mention 라우팅 미도입).
- **2026-09-20** **드레인 전략을 receiver-side 로 이관** (mustang-hub v0.2.1). 배경: `ScheduleWakeup` 이 `/loop dynamic` 모드 전용이라 `--continue` 세션에선 60분 안전망 wake-up 이 작동 안 함. Skill 을 lifecycle 관리 주체에서 notification 소비자로 축소 — 세션 시작 드레인 · 60분 안전망 · CLAUDE.md 강한 트리거 문구 세 개가 다 사라짐. Skill 은 notification-driven only.

## 미결 · 확인 대기

- **`hub` skill 위치**: 확정 — 플러그인 내부 (`skills/hub/SKILL.md`) 로 함께 배포.
- **처리 예외 시 Dooray 코멘트 포맷 표준화** (에러 유형 · 재시도 안 함 안내 등).
- **폴 드레인 주기 fine-tune** — 현재 receiver-side default 600s. 관찰 후 조정 ([[projects/mustang-hub/README]]).
