## Requirements

Karpathy의 llm-wiki 컨셉을 팀 협업 도구로 확장한 오픈소스 솔루션.
사람이 원본 자료를 투입하면 LLM이 위키를 생성·관리하며, 팀 에이전트가 로컬에서 연결해 사용할 수 있다.

핵심 사용자 요건:
- 원본 자료(파일/URL) 투입 UI 제공
- LLM이 원본을 소화해 Obsidian 호환 Markdown 위키 자동 생성·업데이트
- 관리자가 수동 트리거하는 Lint (위키 health-check)
- 사용자가 위키 수정을 LLM에게 요청하는 방식 (직접 편집 불가)
- 위키 조회 UI
- 로컬 연동 방법 가이드 + 기본 프롬프트 제공 (docs)
- 로컬 sync: git clone/pull, zip 다운로드 지원

비기능 요건:
- 오픈소스
- Self-hosted (직접 설치 + Docker)

## Technical Design

### 아키텍처 개요

```
[Auth / User Mgmt UI] → [User DB (SQLite)]

[Source Input UI] → [Ingest API] → [Source Store (Git repo)]
                                          ↓
                                   [LLM Pipeline]
                                     ├── Ingest: 원본 → 위키 작성/업데이트
                                     ├── Edit: 사용자 수정 요청 처리
                                     └── Lint: 관리자 트리거 health-check
                                          ↓
                                   [Wiki Store (Git repo, Obsidian MD)]
                                          ↓
                              ┌──────────┴──────────┐
                         [Wiki UI]            [Local Sync]
                         (조회 전용)        git clone/pull
                                             zip 다운로드
```

### 기술 스택 (안)

| 영역 | 선택 |
|------|------|
| Backend | Python (FastAPI) |
| Frontend | Next.js + shadcn/ui |
| 위키 저장소 | Git (bare repo, Obsidian MD 포맷) |
| 원본 저장소 | Git (별도 repo 또는 동일 repo 내 `_sources/`) |
| LLM | LiteLLM (Anthropic API / Ollama 등 교체 가능) |
| 사용자/계정 DB | SQLite (self-hosted 기본), 추후 Postgres 지원 |
| 인증 | 이메일+패스워드 (JWT), v0.2+ SSO |
| 배포 | Docker Compose (backend + frontend + git server) |

### 주요 컴포넌트

#### 1. Source Ingest
- 파일 업로드 (PDF, DOCX, TXT, MD) 및 URL 입력 지원
- 원본은 `_sources/` 하위에 보존 (soft-delete: `_sources/_trash/`)
- 원본 교체: 기존 파일 replace → LLM re-ingest 트리거

#### 2. LLM Pipeline

**Ingest flow:**
1. 원본 파싱 (파일 → 텍스트)
2. LLM에 위키 작성/업데이트 요청
3. 결과를 Obsidian MD 형식으로 위키 저장소에 커밋

**Edit flow:**
1. 사용자가 UI에서 수정 요청 텍스트 입력
2. LLM이 관련 위키 페이지 찾아 수정 후 커밋

**Lint flow:**
1. 관리자가 UI에서 Lint 실행 트리거
2. LLM이 위키 전체를 순회하며 health-check
3. 결함 항목 리포트 + 자동 수정 옵션

#### 3. Wiki Store
- Git 기반 Obsidian MD 포맷
- 페이지 간 `[[백링크]]` 사용
- YAML frontmatter: `type`, `created`, `last_updated`, `tags`, `sources` (원본 파일 참조)

#### 4. Wiki UI
- 위키 페이지 조회 (읽기 전용)
- 수정 요청 입력창
- 관리자: 원본 투입, Lint 실행

#### 5. Local Sync
- **git**: `git clone <wiki-repo-url>` / `git pull`
- **zip**: UI에서 현재 위키 스냅샷 다운로드
- **가이드 문서**: 에이전트 연동 방법 + 기본 프롬프트 예시

### 데이터 모델 (위키 페이지 frontmatter)

```yaml
---
type: [concept | process | policy | reference | glossary]
created: YYYY-MM-DD HH:MM:SS
last_updated: YYYY-MM-DD HH:MM:SS
tags: []
sources: ["[[_sources/원본파일.pdf]]"]
---
```

### 사용자 역할 (Role)

| 역할 | 권한 |
|------|------|
| Admin | 원본 투입, 원본 삭제/교체, Lint 트리거, 사용자 관리 |
| Member | 위키 조회, 수정 요청 제출, 로컬 sync |

### MVP 범위 (v0.1)

1. 사용자 관리 / 인증 (로그인, 계정 관리, role 지정)
2. 원본 투입 UI (파일 업로드, URL)
3. LLM 위키 생성/업데이트
4. 위키 조회 UI
5. 로컬 sync (git + zip)
6. Lint (관리자 수동 트리거)
7. 로컬 연동 가이드 문서

### 보류 (v0.2+)

- 멀티 테넌시
- 사용자 권한 세분화 (role-based)
- 원본 출처 추적 가시화 (source graph)

## Success Criteria

_Hawkeye 작성 예정_

## Open Issues

- [x] [Advisory] LLM 추상화 레이어 범위 — LiteLLM으로 v0.1부터 추상화, Anthropic API + Ollama 지원으로 결정

## Changelog

- v0.1: 초기 SPEC (Requirements + Technical Design)
