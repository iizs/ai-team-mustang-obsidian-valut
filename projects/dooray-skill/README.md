# dooray-skill

_2026-09-09 착수 · v0.3 완료_

Mustang 팀 에이전트가 Dooray를 사용하기 위한 Claude Code skill + 지원 라이브러리. [[projects/team-operations-rework/README]] 의 협업 환경 축(② 업무 채널) 을 실체화한다.

## 목표

- 각 에이전트(Roy · Breda · Hawkeye · ...) 가 자기 Dooray 계정으로 API 호출을 수행할 수 있게 한다.
- 시작은 미니멀 — 사람↔에이전트 협업의 최소 세트만.
- Skill과 라이브러리는 **별도 GitHub 리포**(`iizs/dooray-skill`) 로 관리하며, 필요에 따라 업데이트한다.

## 대상 사용자 / 범위

- **주 사용자**: Mustang 팀 에이전트 (Claude Code 세션).
- **부 사용자**: 필요 시 사람(Kirin) 도 CLI로 직접 호출 가능.
- **범위 밖**: Webhook 수신·라우팅 (별도 Task Hub 프로젝트에서 다룰 예정).

## Dooray 개념 정리

Dooray는 **프로젝트** 안에 태스크가 존재하는 구조. 즉:

- **워크스페이스(테넌트)** — 예: `<workspace>.dooray.com`
- **프로젝트** — 팀 · 업무 단위. 여러 태스크 · 멤버 · 설정을 담는다.
- **태스크(post)** — 프로젝트 안의 개별 작업 · 이슈. assignee · status · comments 를 가진다.

이 계층 그대로 API 인터페이스에 반영한다.

## 미니멀 기능 (첫 릴리스 목표)

**v0.1 — 완료 (2026-09-09)**
- ✅ 인증 (토큰 · base URL 로드; `Authorization: dooray-api <token>` 헤더)
- ✅ `me` — `GET /common/v1/members/me` (auth 확인 겸 organizationMemberId 확보)
- ✅ 프로젝트 목록 조회 — `GET /project/v1/projects` (default `member=me`)
- ✅ 태스크 목록 조회 — `GET /project/v1/projects/{project-id}/posts` (default `assignee=me` → 자동으로 `toMemberIds=<my id>` 로 해석)

**v0.2 — 완료 (2026-09-09)**
Kirin이 지정한 범위 전부 커버:
- ✅ 태스크 조회 필터 확장 — `assignee/from/cc/tag/parent/workflow-class/workflow-id/milestone/subject/createdAt/updatedAt/dueAt/order` 등 API의 filter를 모두 노출
- ✅ 태스크 상세 조회 — project 스코프(`GET /project/v1/projects/{id}/posts/{pid}`) + 비스코프(`GET /project/v1/posts/{pid}`, webhook 회신용)
- ✅ 태스크 생성 · 수정 (`POST` · `PUT`)
- ✅ Task actions: `set-workflow` / `set-done` / `set-assignee-workflow` (`/to/{member-id}`) / `set-parent-post` / `move`
- ✅ Workflow 관리 · 컨트롤 — `list` / `create` / `update` / `delete`(w/ toBeWorkflowId 이관)
- ✅ Tag 관리 · 컨트롤 — `list` / `get` / `create` / `tag-group update`(mandatory · selectOne)
- ✅ 댓글(log) — `list` / `get` / `create` / `update` / `delete`

**"Project > Posts" vs "Project > Projects > Posts"**: 전자는 `project-id` 없이 `post-id`로 직접 조회 (`GET /project/v1/posts/{pid}`, `POST /post-drafts` 임시 업무). 후자는 프로젝트 스코프 CRUD + actions. 둘 다 커버 (drafts는 skip).

**v0.3+ 후보**
- 다른 봇 유저(Breda/Hawkeye 등) 계정 부여 및 토큰 파일 세팅
- 파일 첨부(POST files, multipart) — 지금은 skip
- 스코프 밖: 마일스톤 · 템플릿 · 훅 관리 · 멤버 관리 (Kirin이 명시적으로 skip)
- (Task Hub 도입 시) Webhook payload 파서 · 라우터 헬퍼

## 인증 · 설정

**v0.3에서 재설계.** 세션 launcher가 env로 토큰을 주입하는 것이 정합이라는 판단.

토큰 우선순위:
1. **env `$DOORAY_API_KEY`** (권장) — launcher가 세팅. 봇 identity는 이 값(토큰)이 곧 정한다. skill 호출에 별도 `agent` 인자 불필요.
2. **파일 `~/.dooray/tokens/key`** — 로컬 dev fallback (한 줄에 토큰).
3. 둘 다 없으면 명시적 에러.

Base URL은 `~/.dooray/base_url` 파일 (optional). 없으면 민간 클라우드(`https://api.dooray.com`) default. 다른 클라우드:
- 민간: `https://api.dooray.com`
- 공공: `https://api.gov-dooray.com`
- 공공 업무망: `https://api.gov-dooray.co.kr`
- 금융: `https://api.dooray.co.kr`

**Dooray에서 personal token 발급**: 각 봇 유저(및 Kirin)로 웹 UI 로그인 → **개인 설정 → API → 개인 인증 토큰** 메뉴에서 발급.

> ⚠️ 웹 UI 주소(`<workspace>.dooray.com`)와 API endpoint(`api.dooray.com`)는 다르다. `base_url` 파일엔 API endpoint를 넣는다.

## 아키텍처 (초안)

```
dooray-skill/
├── SKILL.md               # Claude Code skill 정의 (매뉴얼 호출)
├── src/
│   └── dooray_skill/      # Python 패키지
│       ├── __init__.py
│       ├── auth.py        # 토큰·base URL 로드
│       ├── client.py      # 얇은 HTTP wrapper (requests)
│       ├── projects.py    # 프로젝트 API
│       ├── tasks.py       # 태스크 API
│       └── cli.py         # 사람 사용용 얇은 CLI
├── requirements.txt
├── README.md
├── .gitignore
└── .python-version        # (optional) pyenv
```

- Python 3.10+ (system python 3.9 있으니 pyenv 로 3.11+ 고정 검토)
- venv 격리는 필수 (Kirin 원칙)
- 의존성 최소: `requests` 정도

## 사용 예시 (예상)

```python
from dooray_skill import Dooray

dr = Dooray(agent="roy")

# 프로젝트 목록
for p in dr.projects.list():
    print(p.id, p.code, p.name)

# 특정 프로젝트의 내 태스크
tasks = dr.tasks.list(project_id="123", assignee="me")
```

Skill 호출:
```
Skill(skill="dooray", args="agent=roy op=projects.list")
Skill(skill="dooray", args="agent=roy op=tasks.list project_id=123 assignee=me")
```

_(op 명명 규칙은 v0.1 구현하며 확정)_

## 진행 순서 (오늘 세션)

1. ✅ Vault 프로젝트 문서 뼈대 (이 파일)
2. GitHub 리포 생성: `gh repo create iizs/dooray-skill --private`
3. 로컬 clone + 초기화 (README, .gitignore, requirements.txt, venv 지침, src 골격)
4. Access key 저장 위치·명세 문서화 → Kirin 입력
5. 인증 + `projects.list` + `tasks.list` 구현 · 실증

## 결정 기록 (연대순)

- **2026-09-09** 프로젝트 착수. 이름 `dooray-skill` · 리포 `iizs/dooray-skill` (private) · 문서 위치 `~/Vaults/team-mustang/projects/dooray-skill/`.
- **2026-09-09** 언어 Python (venv 격리 필수). CLI 배제 (기능 부족).
- **2026-09-09** 첫 목표 기능 = 인증 + 프로젝트 목록 + 태스크 목록.
- **2026-09-09** 인증 파일 배치: `~/.dooray/base_url` (optional, default `https://api.dooray.com`) + `~/.dooray/tokens/<agent>` (`chmod 600`).
- **2026-09-09** API base URL은 워크스페이스 URL이 아닌 클라우드별 고정 endpoint (`api.dooray.com` 등). Auth 스키마: `Authorization: dooray-api <token>`.
- **2026-09-09** "assignee=me" 축약이 API에 직접 없어서 `GET /common/v1/members/me` 로 `id`를 얻어 `toMemberIds` 로 전달. Members.me() 캐시.
- **2026-09-09** macOS 시스템 Python(3.9 + LibreSSL) 지원 위해 `urllib3<2` 핀. requires-python = 3.9.
- **2026-09-09** v0.1 실증 완료: Sandbox 프로젝트에서 `me`/`projects-list`/`tasks-list` 모두 정상 응답.
- **2026-09-09** v0.2 완료: 4개 신규 모듈(`workflows.py` / `tags.py` / `logs.py` / `members.py`) + `tasks.py` 확장. Task CRUD + workflow · tag · log · task actions 전부 실증 성공 (Sandbox 프로젝트에서 create → set-workflow → log-create → log-update → set-done → log-delete까지 e2e). 파일 첨부는 스코프 밖으로 유예.
- **2026-09-09** 이용 시나리오 커버: `assignee=me` 자동 해석 외에도 `from_member`/`cc_member`에도 `me` shortcut 도입. `set-assignee-workflow`도 API가 `me`를 그대로 지원.
- **2026-09-10** 유닛 테스트 59건 (auth · client · members · projects · tasks · workflows · tags · logs) + Sandbox e2e 통합 테스트 12건 도입. 통합 테스트는 `~/.dooray/tokens/roy` 없으면 auto-skip. `pytest -q` 로 전체 71건 12초 정도.
- **2026-09-10** 관찰: `tasks.list(subjects="[...] ...")` 처럼 대괄호가 포함된 subject 필터를 보내면 Dooray 서버가 500을 반환하는 경우 있음. 통합 테스트에선 subject 필터 없이 대체.
- **2026-09-10** 인증 방식 재설계 (v0.3). Kirin 지시: skill 스코프 좁게 (`DOORAY_API_KEY` 명확한 env 변수), 파일 기본 경로 `~/.dooray/tokens/key`, 둘 다 없으면 에러. `agent` 인자 완전 제거 — identity는 token 자체가 정하고, 봇마다 launcher env로 주입되는 방식으로 정착. `~/.dooray/tokens/<agent>` 개별 파일 관례는 폐기.

## 미결 / 다음 단계

- 다른 봇 유저(Breda/Hawkeye) 계정 신설 및 토큰 배포.
- v0.2 기능(태스크 상세/생성/상태 변경/댓글) 추가 시점 판단 — 실제 워크플로우 실증 후.
- Skill 호출 인자 명명 규칙 (`op=projects.list` 형태 유지 vs 세분화된 여러 skill).
- `SKILL.md` 를 `~/.claude/skills/dooray/` 로 배치(심볼릭 링크 등) 후 실제 Skill invoke 시연.
