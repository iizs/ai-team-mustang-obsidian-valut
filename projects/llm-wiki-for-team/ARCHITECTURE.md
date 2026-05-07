# Architecture

## 시스템 큰 그림

```
[Auth / User Mgmt UI] → [User DB (SQLite)]

[Source Input UI] → [Ingest API] → [Source Store (Docker Volume / Local Path)]
                                          ↓
                                   [LLM Pipeline]
                                     ├── Ingest: 원본 → 위키 작성/업데이트
                                     └── Edit: 사용자 수정 요청 처리
                                          ↓
                                   [Wiki Store (Git repo, Normal, Obsidian MD)]
                                          ↓
                              ┌──────────┴──────────┐
                         [Wiki UI]            [Local Sync]
                      + [Source UI]         git clone/pull
                         (조회 전용)          zip 다운로드
```

## 기술 스택

| 영역 | 선택 | 결정 근거 |
|------|------|-----------|
| Backend | Python (FastAPI) | — |
| Frontend | Next.js + shadcn/ui | — |
| 위키 저장소 | Git (normal repo, working tree 있음, Obsidian MD 포맷) | [ADR-0001](decisions/0001-normal-vs-bare-repo.md) |
| 원본 저장소 | Docker Volume (Docker 배포) / Local Path (설치형) — git 비추적 | [ADR-0002](decisions/0002-source-store-not-tracked.md) |
| LLM | LiteLLM (Anthropic API / Ollama 등 교체 가능) | [ADR-0004](decisions/0004-litellm-abstraction.md) |
| 사용자/계정 DB | SQLite (self-hosted 기본), 추후 Postgres 지원 | — |
| 인증 | 이메일+패스워드 (JWT), v0.2+ SSO | — |
| 비동기 처리 | FastAPI `BackgroundTasks` (단일 워커) | [ADR-0006](decisions/0006-fastapi-backgroundtasks.md) |
| 배포 | Docker Compose (backend + frontend + nginx git server) | — |
| git 서버 | nginx + `git http-backend` (별도 컨테이너) | — |

## 컴포넌트 인덱스

상세 명세는 각 컴포넌트 파일 참조.

- [Auth](components/auth.md) — 인증, 사용자 관리, 역할
- [Job Queue](components/job-queue.md) — 비동기 처리, INGEST/EDIT 타입
- [Source Store](components/source-store.md) — 원본 파일 저장 + 참조 설계
- [LLM Pipeline](components/llm-pipeline.md) — Ingest/Edit 흐름 + LLM 설정/프롬프트
- [Wiki Store](components/wiki-store.md) — git repo, index.md, log.md, _sheska.yaml
- [Wiki UI](components/wiki-ui.md) — 조회 UI + Edit 입력창 + Jobs 탭
- [Local Sync](components/local-sync.md) — git clone/pull, zip

## 데이터 모델 (위키 페이지 frontmatter)

```yaml
---
type: [concept | process | policy | reference | glossary]
created: YYYY-MM-DD HH:MM:SS
last_updated: YYYY-MM-DD HH:MM:SS
tags: []
sources:
  - "원본파일.pdf"
---
```

## 핵심 설계 원칙

> "**LLM은 축적하지, 삭제 판단은 Lint에서**"
>
> Karpathy llm-wiki의 핵심. 위키는 LLM이 새 정보를 추가/보강만 하고, 오래되거나 모순된 내용의 정리는 별도 Lint 단계(v0.2+)에서 처리한다.
>
> [ADR-0008](decisions/0008-llm-accumulates-lint-cleans.md) 참조.
