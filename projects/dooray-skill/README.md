# dooray-skill

_2026-09-09 착수 · Planning_

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

**v0.1 (지금 세션 목표)**
- 인증 (토큰 · base URL 로드)
- 프로젝트 목록 조회
- 태스크 목록 조회 (프로젝트 지정, `assignee=me` 필터 포함)

**v0.2 이후 후보 (실증 후 추가)**
- 태스크 상세 조회
- 태스크 생성 (project · title · body · assignee)
- 태스크 상태 변경
- 댓글 작성 · 조회
- (Task Hub 도입 시) Webhook payload 파서 · 라우터 헬퍼

## 인증 · 설정 (초안)

파일 방식. Git 밖. `chmod 600`.

- `~/.dooray/base_url` — 워크스페이스 URL 한 줄. 예: `https://mustang.dooray.com`
- `~/.dooray/tokens/<agent>` — 에이전트별 personal access token 한 줄. 예: `~/.dooray/tokens/roy`

Skill/라이브러리는 `agent=<이름>` 인자를 받아 해당 파일에서 토큰을 로드한다. 사람이 CLI로 쓸 때도 `--agent kirin` 형태로 통일.

**Dooray에서 personal token 발급**: 각 봇 유저(및 Kirin)로 웹 UI 로그인 → 개인 설정 → API access token 생성 → 위 경로에 저장.

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

- **2026-09-09** 프로젝트 착수. 이름 `dooray-skill` · 리포 `iizs/dooray-skill` · 문서 위치 `~/Vaults/team-mustang/projects/dooray-skill/`.
- **2026-09-09** 언어 Python (venv 격리 필수). CLI 배제 (기능 부족).
- **2026-09-09** 첫 목표 기능 = 인증 + 프로젝트 목록 + 태스크 목록.
- **2026-09-09** 인증 파일 배치: `~/.dooray/base_url` + `~/.dooray/tokens/<agent>` (`chmod 600`).

## 미결 / 다음 단계

- Dooray API 문서 확보 (URL이 SPA라 fetch 안 됨 — PDF 요청 대기).
- Skill 호출 인자 명명 규칙 (`op=projects.list` 형태 vs 개별 skill 여러 개).
- Python 버전 pin (3.9 vs 3.11+).
