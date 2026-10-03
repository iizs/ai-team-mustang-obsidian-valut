# 핸드북 (Handbook)

팀과 상관없이 모든 에이전트가 알아야 할 공통 도구와 요청 창구. 현재 적용 대상은 Roy · Breda · Hawkeye · Falman. 행동 원칙은 `CONSTITUTION.md`, 팀 안의 일하는 방식은 `teams/<팀>/CONSTITUTION.md` 에 둔다.

## 조직

| 누가 | 하는 일 | 창구 (Dooray 프로젝트) |
|---|---|---|
| Roy — 운영 총괄 (Operations) | 에이전트 구성 · 계정 · 권한, 헌장 · 핸드북 관리, 공통 도구 · 실행 환경 운영 | `AI-Team-Management` |
| Atelier — 파일럿 개발팀 (Breda · Hawkeye) | 아이디어를 빠르게 만들어 가치 확인. 헌장 `teams/atelier/CONSTITUTION.md` | `Atelier` |
| Falman | SNS 발행 | — |

파일럿 시작은 Kirin이 정한다.

## 공통 도구

모든 에이전트가 함께 쓰는 스킬·플러그인·실행 환경. 아래 도구의 버그 신고, 개선 요청, 건의사항은 [요청 창구](#요청-창구)로 보낸다. 담당은 Roy.

**에이전트가 직접 쓰는 것**

| 도구 | 하는 일 | 버전 | 위치 · 문서 |
|---|---|---|---|
| `dooray` 스킬 | Dooray API 호출 (태스크 · 댓글 · 첨부파일) | 0.6.0 | `~/Projects/dooray-skill` · `projects/dooray-skill/README.md` |
| `journal` 스킬 | vault 저널 기록 | 버전 관리 없음 | `~/.claude/skills/journal/SKILL.md` |
| mustang-hub-agent 플러그인 | `hub` 스킬 (Dooray 태스크 처리 절차) + 알림 수신 consumer | 0.1.0 | `~/Projects/mustang-hub-agent` · `projects/mustang-hub-agent/README.md` |

**에이전트를 돌리는 실행 환경**

| 도구 | 하는 일 | 위치 · 문서 |
|---|---|---|
| mustang-hub 리시버 | Dooray webhook · 폴링을 받아 담당 에이전트에게 알림 전달 | `~/Projects/mustang-hub` · `projects/mustang-hub/README.md` |
| 런처 스크립트 | 에이전트 세션 기동 (`start.sh`, `start-team-*.sh`), 세션 env 주입 | `~/Projects/ai-teams/scripts` |
| discord-agent-runner | Discord로 운영하는 에이전트의 메시지 ↔ Claude 세션 중계 | `~/Projects/ai-teams/discord-agent-runner` · `projects/discord-agent-runner/README.md` |
| 공통 Claude Code 설정 | 사용자 단위 권한 허용 목록 등 (`~/.claude/settings.json`) | — |

**외부 도구**

| 도구 | 비고 |
|---|---|
| claude-in-chrome MCP | 우리가 고칠 수 없다. 요청하면 원인 분류와 우회 방법만 안내한다. |

에이전트 한 명만 쓰는 스킬(각 에이전트 세션 디렉토리에 있는 것)은 공통 도구가 아니다. 해당 에이전트와 Kirin이 관리한다.

버전은 공통 도구가 릴리스될 때 이 표에서 갱신한다. 무엇이 바뀌었는지는 각 문서의 상태·변경 기록에서 확인한다.

## 요청 창구

**Dooray 프로젝트 `AI-Team-Management` 에 태스크를 만들고 담당자를 Roy로 지정한다.**

- 프로젝트 id: `4426732997356166358`
- Roy memberId: `4414791632567480970`

```bash
dooray task-create --project-id 4426732997356166358 \
  --to 4414791632567480970 \
  --subject "[dooray-skill] 버그: task-get 이 500 을 반복" \
  --body-file request.md
```

### 제목

`[대상] 종류: 요약`

- 대상: 위 표의 도구 이름 (`dooray-skill`, `journal`, `hub-agent`, `mustang-hub`, `launcher`, `discord-agent-runner`, `settings`, `chrome-mcp`). 어느 도구 문제인지 모르겠으면 `[미상]`.
- 종류: `버그` · `개선` · `건의`
  - 버그: 문서에 적힌 대로 동작하지 않는다.
  - 개선: 있는 기능을 바꾸거나 새 기능이 필요하다.
  - 건의: 도구 운영 방식에 대한 의견, 새 공통 도구 제안.

### 본문

```md
## 대상
도구 이름과 버전

## 현상
무엇을 했고 무엇이 일어났는지. 실행한 명령, 오류 출력(dooray 는 stderr JSON 한 줄)을 그대로 붙인다.

## 기대한 동작
어떻게 되어야 하는지, 또는 무엇이 필요한지

## 급한 정도
지금 막혀 있는지. 우회해서 진행 중이면 그 방법.
```

건의는 `현상` 대신 `배경` 으로 써도 된다.

### 요청하기 전에

- 같은 건이 열려 있는지 확인한다 (`dooray tasks-list --project-id 4426732997356166358`). 있으면 새로 만들지 말고 그 태스크에 댓글을 단다.
- 공통 도구의 코드나 설정을 직접 고치지 않는다. 우회해서 일을 진행하는 건 괜찮지만, 그 우회 방법을 요청서에 적는다.

### 요청한 뒤에

- Roy가 검토하고 Kirin 승인을 받은 뒤 진행한다. 승인 전까지 시간이 걸릴 수 있다.
- 처리가 끝나면 결과 댓글과 함께 담당자가 요청자로 돌아온다. 직접 확인한 뒤 완료 처리한다. 해결되지 않았으면 댓글을 달고 담당자를 다시 Roy로 바꾼다.
