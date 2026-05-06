## Requirements

> 서비스명: **Sheska** — llm-wiki for team
> 원칙 문서: https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f
> 방향성이 모호한 경우 이 문서를 따른다.

Karpathy의 llm-wiki 컨셉을 팀 협업 도구로 확장한 오픈소스 솔루션.
사람이 원본 자료를 투입하면 LLM이 위키를 생성·관리하며, 팀 에이전트가 로컬에서 연결해 사용할 수 있다.

핵심 사용자 요건:
- 원본 자료(파일) 투입 UI 제공 (PDF, TXT, MD; 업로드 확장자 제한)
- LLM이 원본을 소화해 Obsidian 호환 Markdown 위키 자동 생성·업데이트
- 사용자가 위키 수정을 LLM에게 요청하는 방식 (직접 편집 불가)
- 위키 조회 UI
- 원본 파일 조회 UI (Web UI 내 별도 메뉴)
- 로컬 연동 방법 가이드 + 기본 프롬프트 제공 (docs)
- 로컬 sync: git clone/pull, zip 다운로드 지원

비기능 요건:
- 오픈소스
- Self-hosted (직접 설치 + Docker)

## Technical Design

### 아키텍처 개요

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

### 기술 스택

| 영역 | 선택 |
|------|------|
| Backend | Python (FastAPI) |
| Frontend | Next.js + shadcn/ui |
| 위키 저장소 | Git (normal repo, working tree 있음, Obsidian MD 포맷) |
| 원본 저장소 | Docker Volume (Docker 배포) / Local Path (설치형) — git 비추적 |
| LLM | LiteLLM (Anthropic API / Ollama 등 교체 가능) |
| 사용자/계정 DB | SQLite (self-hosted 기본), 추후 Postgres 지원 |
| 인증 | 이메일+패스워드 (JWT), v0.2+ SSO |
| 배포 | Docker Compose (backend + frontend + git server) |

### 주요 컴포넌트

#### 1. Source Ingest
- 파일 업로드: PDF, TXT, MD 허용 (확장자 화이트리스트 검증)
- 원본은 Source Store(볼륨/로컬 경로)에 보존
- soft-delete: `_trash/` 하위 이동
- 원본 교체: 기존 파일 replace → LLM re-ingest 트리거

#### 2. LLM Pipeline

**Ingest flow:**
1. 원본 파싱 (파일 → 텍스트)
2. LLM에 위키 작성/업데이트 요청
3. 결과를 Obsidian MD 형식으로 Wiki Store에 커밋

**Edit flow:**
1. 사용자가 UI에서 수정 요청 텍스트 입력
2. LLM에 페이지 목록 + frontmatter 전달 → 대상 페이지 선택 (1단계)
3. LLM이 선택된 페이지를 수정 후 커밋 (2단계)

#### 3. Wiki Store
- Normal Git repo (working tree 있음), gitpython으로 read/write/commit
- Obsidian MD 포맷: 페이지 간 내부 링크는 `[[페이지명]]` 형식
- YAML frontmatter: `type`, `created`, `last_updated`, `tags`, `sources`

#### 4. Source 참조 설계

위키 페이지의 `sources` frontmatter는 원본 파일에 대한 접근 경로를 포함한다.
원본은 git에 없으므로 위키를 로컬에 받아간 쪽도 원본에 접근할 수 있어야 한다.

**방식:**
- `_sheska.yaml` (위키 repo root에 포함): `source_base_url` 설정값 보관
- 위키 페이지 frontmatter: 원본 파일명(상대경로)만 기재
- 원본 접근 URL = `source_base_url` + 파일명

```yaml
# _sheska.yaml (wiki repo root, git 추적)
source_base_url: "https://sheska.yourcompany.com/api/sources"
```

```yaml
# 위키 페이지 frontmatter
sources:
  - "product_spec_v2.pdf"
  - "api_reference.md"
```

→ 실제 접근 URL: `https://sheska.yourcompany.com/api/sources/product_spec_v2.pdf`

**Backend API:**
- `GET /api/sources/{filename}` — 원본 파일 서빙 (Auth 필요)
- `GET /api/sources/` — 원본 목록

#### 5. Wiki UI
- 위키 페이지 조회 (읽기 전용)
- 수정 요청 입력창
- 관리자: 원본 투입, 원본 목록/조회/삭제 메뉴
- Member: 원본 조회 가능 (읽기 전용)

#### 6. Local Sync
- **git**: `git clone <wiki-repo-url>` / `git pull`
- **zip**: UI에서 현재 위키 스냅샷 다운로드 (`_sheska.yaml` 포함)
- **가이드 문서**: 에이전트 연동 방법 + 기본 프롬프트 예시 + `source_base_url` 설정 방법

### 데이터 모델 (위키 페이지 frontmatter)

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

### 사용자 역할 (Role)

| 역할 | 권한 |
|------|------|
| Admin | 원본 투입/교체/삭제, 사용자 관리 |
| Member | 위키 조회, 수정 요청 제출, 원본 조회, 로컬 sync |

### MVP 범위 (v0.1)

1. 사용자 관리 / 인증 (로그인, 계정 관리, role 지정)
2. 원본 투입 UI (PDF, TXT, MD 파일 업로드; 확장자 검증)
3. LLM 위키 생성/업데이트 (Ingest flow)
4. LLM 수정 요청 처리 (Edit flow, 2단계)
5. 위키 조회 UI
6. 원본 조회 UI
7. 로컬 sync (git + zip, `_sheska.yaml` 포함)
8. 로컬 연동 가이드 문서

### 보류 (v0.2+)

- URL 소화 (크롤링)
- DOCX 지원
- Lint (위키 health-check)
- SSO
- 멀티 테넌시
- 원본 출처 추적 가시화 (source graph)

## Success Criteria

_Hawkeye 작성 예정_

## Open Issues

- [x] [Advisory] LLM 추상화 레이어 범위 — LiteLLM으로 v0.1부터 추상화, Anthropic API + Ollama 지원으로 결정
- [x] [Advisory] Git 저장 방식 — normal repo (working tree 있음) 채택, bare repo 제외
- [x] [Advisory] 원본 파일 저장 — git 비추적, Docker Volume / Local Path 분리
- [x] [Advisory] MVP 파일 형식 — PDF, TXT, MD만 지원 (DOCX, URL 제외)
- [x] [Advisory] Lint — v0.2+로 보류

## Changelog

- v0.1: 초기 SPEC (Requirements + Technical Design)
- v0.2: Breda 기술 검토 반영 — 저장 구조, Edit flow, Source 참조 설계, MVP 범위 조정
