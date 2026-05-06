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

#### 1. Job Queue (비동기 처리)

모든 LLM 처리 작업은 즉시 실행이 아닌 Job으로 등록되어 백그라운드 워커가 순차 처리한다.

**Job 종류:**
| Type | 트리거 |
|------|--------|
| `INGEST` | 원본 파일 업로드 (신규/교체 구분 없음) |
| `EDIT` | 사용자 위키 수정 요청 |

> 원칙: "LLM은 축적하지, 삭제 판단은 Lint에서" — 원본 교체 시도 단순 INGEST와 동일하게 처리. 기존 위키 내용을 삭제하지 않고 새 원본 내용을 반영·보강한다.

**Job 상태:** `Pending → Processing → Done | Failed`

**데이터 모델 (SQLite `jobs` 테이블):**
```
job_id | type | payload | status | created_by | created_at | updated_at | error_msg
```
- `payload`: INGEST — `{source_path}`, EDIT — `{edit_text}`

**동작 규칙:**
- 사용자는 업로드/요청 후 즉시 "큐 등록됨" 응답 받음
- **Jobs 탭 (별도 UI)**: Job 목록 및 처리 상태(Pending/Processing/Done/Failed) 조회. Admin: 전체, Member: 본인 요청만
- 실패 시 `error_msg` 저장, Admin 재시도 가능
- MVP: 워커 1개 (순차 FIFO)

**Known Limitation (v0.1):**
원본 교체 시 원본에서 제거된 내용이 위키에 잔존할 수 있다. Lint(v0.2+) 도입 전까지 사용자가 직접 Edit 요청으로 수동 정리해야 한다.

#### 2. Source Ingest
- 파일 업로드: PDF, TXT, MD 허용 (확장자 화이트리스트 검증)
- 원본은 Source Store(볼륨/로컬 경로)에 보존
- 원본 삭제: v0.2+ (MVP 제외)

#### 3. LLM Pipeline

**Ingest flow:**
1. 원본 파싱 (파일 → 텍스트)
2. LLM에 위키 작성/업데이트 요청 (축적 방식, 삭제 없음)
3. 결과를 Obsidian MD 형식으로 Wiki Store에 커밋
4. `index.md` 갱신 (LLM)
5. `log.md` 항목 append (시스템)

**Edit flow:**
1. 사용자가 특정 위키 페이지 조회 화면 하단 입력창에 수정 요청 텍스트 입력 (단일 페이지 컨텍스트)
2. 해당 페이지를 대상으로 LLM에 수정 요청 전달 (페이지 선택 단계 생략)
3. LLM이 해당 페이지를 수정 후 커밋
4. `index.md` 갱신 (LLM)
5. `log.md` 항목 append — `job_id` + 수정된 페이지만 기록 (요청 전문은 jobs 테이블에만 보관)

#### 4. Wiki Store
- Normal Git repo (working tree 있음), gitpython으로 read/write/commit
- Obsidian MD 포맷: 페이지 간 내부 링크는 `[[페이지명]]` 형식
- YAML frontmatter: `type`, `created`, `last_updated`, `tags`, `sources`

**예약 파일 (repo root):**

`index.md` — 위키 전체 지도. LLM이 모든 Ingest/Edit 완료 후 자동 갱신.
```markdown
# Wiki Index
*Last updated: YYYY-MM-DD HH:MM:SS*

| 페이지 | 타입 | 한줄 요약 | 최종 수정 |
|--------|------|-----------|-----------|
| [[페이지명]] | concept | ... | YYYY-MM-DD |
```
- 에이전트가 위키 구조 파악 시 첫 번째로 읽는 파일

`log.md` — 모든 Job 이력. append-only (최신 항목이 상단). 시스템이 자동 기록.
- **기록 범위**: `job_id` + 결과(생성/수정된 페이지 목록)만 포함. 요청 전문은 기록하지 않음 (jobs 테이블에만 보관).
- git에 포함되어 clone 시 전체 공개됨 — 민감한 요청 내용 노출 방지 목적.
```markdown
# Sheska Operation Log

## 2026-05-06T19:00:00Z | INGEST | SUCCESS | job_id: abc123
- source: product_spec_v2.pdf
- created: [[제품개요]], [[기능정책]]
- updated: [[용어집]]

## 2026-05-06T18:30:00Z | EDIT | SUCCESS | job_id: def456
- page: [[기능정책]]
- modified: [[기능정책]]

## 2026-05-06T18:00:00Z | INGEST | FAILED | job_id: ghi789
- source: old_spec.pdf
- error: LLM context limit exceeded
```

`_sheska.yaml` — Sheska 서버 설정. 원본 접근 base URL 보관.

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
- **수정 요청 입력창**: 각 위키 페이지 하단에 고정. 해당 페이지 단일 컨텍스트로 수정 요청 제출. 여러 페이지에 걸친 요청은 MVP 범위 외.
- **Jobs 탭**: Job 목록 및 처리 상태 조회 (Admin: 전체 / Member: 본인 요청만). 요청 전문, 상태, 에러 메시지 포함.
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
2. 비동기 Job Queue (INGEST / EDIT, 상태 조회 UI 포함)
3. 원본 투입 UI (PDF, TXT, MD 파일 업로드; 확장자 검증)
4. LLM 위키 생성/업데이트 (Ingest flow, index.md + log.md 자동 관리)
5. LLM 수정 요청 처리 (Edit flow, 2단계, index.md + log.md 자동 관리)
5. 위키 조회 UI
6. 원본 조회 UI
7. 로컬 sync (git + zip, `_sheska.yaml` 포함)
8. 로컬 연동 가이드 문서

### 보류 (v0.2+)

- 원본 삭제 (soft-delete)
- URL 소화 (크롤링)
- DOCX 지원
- Lint (위키 health-check, 스테일 페이지 정리)
- Job 병렬 처리 (워커 N개)
- SSO
- 멀티 테넌시
- 원본 출처 추적 가시화 (source graph)

## Success Criteria

### 인증 / 사용자 관리

**SC-1** [Blocking] 이메일+패스워드 로그인
- Given: 등록된 이메일과 패스워드를 가진 사용자
- When: 로그인 시도
- Then: JWT 토큰 발급 및 대시보드 진입 성공

**SC-2** [Blocking] 미인증 접근 차단
- Given: 비로그인 상태
- When: 보호된 페이지 URL 직접 접근
- Then: 로그인 화면으로 리다이렉트

**SC-3** [Blocking] Admin 사용자 관리
- Given: Admin 계정으로 로그인
- When: 사용자 관리 메뉴 접근
- Then: 사용자 목록 조회, 신규 계정 생성, role(Admin/Member) 지정 가능

**SC-4** [Blocking] Member 권한 제한
- Given: Member 계정으로 로그인
- When: 원본 투입 또는 사용자 관리 기능 접근 시도
- Then: 접근 거부 및 권한 오류 응답 반환

**SC-5** [Advisory] 비밀번호 변경
- Given: 로그인된 사용자
- When: 비밀번호 변경 요청 제출
- Then: 새 비밀번호 적용, 기존 JWT 무효화

### 원본 투입

**SC-6** [Blocking] 허용 확장자 파일 업로드
- Given: Admin 계정, PDF/TXT/MD 파일
- When: 원본 투입 UI에서 업로드
- Then: Source Store(볼륨/로컬 경로)에 저장 성공, 목록에 표시

**SC-7** [Blocking] 허용 외 확장자 차단
- Given: Admin 계정, .docx/.xlsx 등 비허용 파일
- When: 업로드 시도
- Then: 업로드 거부, 허용 확장자 안내 오류 메시지 표시

**SC-8** [Blocking] 원본 교체 및 re-ingest
- Given: 기존 원본 파일이 Source Store에 존재
- When: 동일 파일명으로 재업로드
- Then: 기존 파일 replace, re-ingest 자동 트리거

**SC-9** [Blocking] 원본 soft-delete
- Given: Admin 계정, 기존 원본 파일
- When: 삭제 요청
- Then: `_trash/` 하위로 이동, 원본 활성 목록에서 제거

**SC-10** [Blocking] Member 원본 투입 차단
- Given: Member 계정
- When: 원본 파일 업로드 시도
- Then: 권한 오류 반환, 업로드 불가

### LLM Ingest

**SC-11** [Blocking] 원본 업로드 후 위키 페이지 생성
- Given: PDF/TXT/MD 파일이 Source Store에 저장됨
- When: Ingest flow 실행
- Then: LLM이 Obsidian MD 위키 페이지를 생성하여 Wiki Store에 git commit

**SC-12** [Blocking] 위키 페이지 frontmatter 준수
- Given: Ingest 완료된 위키 페이지
- When: 파일 내용 확인
- Then: `type`, `created`, `last_updated`, `tags`, `sources` 필드가 SPEC 형식에 맞게 존재

**SC-13** [Blocking] 내부 링크 형식 준수
- Given: LLM이 생성한 위키 페이지
- When: 다른 페이지를 참조하는 경우
- Then: `[[페이지명]]` Obsidian 형식 사용 (일반 markdown 링크 불가)

**SC-14** [Blocking] Source 참조 설계 준수
- Given: Ingest 완료
- When: 위키 페이지 frontmatter의 `sources` 필드 확인
- Then: 원본 파일명(상대경로)만 존재하며, `_sheska.yaml`에 `source_base_url` 기재

### LLM Edit

**SC-15** [Blocking] 수정 요청 입력 위치
- Given: Member/Admin이 특정 위키 페이지 조회 중
- When: 페이지 하단 입력창에 수정 요청 텍스트 입력 후 제출
- Then: 해당 페이지를 컨텍스트로 EDIT Job 생성, "큐 등록됨" 응답 반환

**SC-16** [Blocking] 단일 페이지 Edit 처리
- Given: EDIT Job 생성 (특정 페이지 컨텍스트)
- When: Edit flow 실행
- Then: LLM이 해당 페이지만 수정 후 git commit. 다른 페이지는 변경되지 않음.

**SC-17** [Blocking] 수정 결과 git commit
- Given: Edit flow 완료
- When: Wiki Store git log 확인
- Then: 변경된 페이지가 새 commit으로 기록됨

### Jobs 탭

**SC-17-a** [Blocking] Jobs 탭 — Member 본인 요청 조회
- Given: Member 로그인, Edit 또는 Ingest Job 제출 후
- When: Jobs 탭 접근
- Then: 본인이 제출한 Job 목록 및 처리 상태(Pending/Processing/Done/Failed) 조회 가능, 요청 전문 확인 가능

**SC-17-b** [Blocking] Jobs 탭 — Admin 전체 조회
- Given: Admin 로그인
- When: Jobs 탭 접근
- Then: 전체 사용자의 Job 목록 및 상태 조회 가능

**SC-17-c** [Advisory] Jobs 탭 — 수정 결과 페이지 표시
- Given: Edit flow 완료
- When: Jobs 탭에서 해당 Job 확인
- Then: 수정된 페이지 목록이 명시적으로 표시됨

### 위키 조회 UI

**SC-18** [Blocking] 위키 페이지 목록 및 조회
- Given: Member 이상 계정으로 로그인
- When: Wiki UI 접근
- Then: 위키 페이지 목록 표시, 페이지 클릭 시 내용 조회 가능

**SC-19** [Blocking] 직접 편집 불가 (읽기 전용)
- Given: 위키 페이지 조회 중
- When: 텍스트 직접 편집 시도
- Then: 편집 불가, 수정 요청 입력창만 제공

**SC-20** [Advisory] 내부 링크 클릭 이동
- Given: 위키 페이지에 `[[다른페이지]]` 링크 존재
- When: 링크 클릭
- Then: 해당 위키 페이지로 이동

### 원본 조회 UI

**SC-21** [Blocking] 원본 파일 목록 접근
- Given: Admin 또는 Member 로그인
- When: 원본 목록 메뉴 접근
- Then: 업로드된 원본 파일 목록 표시

**SC-22** [Blocking] Member 원본 조회 권한
- Given: Member 로그인
- When: 원본 파일 조회/다운로드
- Then: 조회 및 다운로드 가능, 업로드/삭제 버튼 비활성 또는 미노출

**SC-23** [Blocking] 원본 서빙 API 인증 필수
- Given: 비인증 클라이언트
- When: `GET /api/sources/{filename}` 호출
- Then: 401 Unauthorized 반환

### 로컬 Sync

**SC-24** [Blocking] git clone/pull 동작
- Given: 배포된 Sheska 인스턴스
- When: 노출된 wiki repo URL로 `git clone` 또는 `git pull`
- Then: 위키 MD 파일 + `_sheska.yaml` 정상 다운로드

**SC-25** [Blocking] zip 다운로드
- Given: 로그인된 사용자
- When: UI에서 zip 다운로드 요청
- Then: 현재 위키 스냅샷 + `_sheska.yaml` 포함된 zip 파일 다운로드

**SC-26** [Blocking] `_sheska.yaml` 존재 및 형식
- Given: git clone 또는 zip 다운로드 완료
- When: 파일 구성 확인
- Then: `source_base_url` 필드가 담긴 `_sheska.yaml`이 repo 루트에 존재

### 로컬 연동 가이드

**SC-27** [Advisory] 가이드 문서 내용 완비
- Given: Sheska 배포
- When: 가이드 문서 페이지 접근
- Then: 에이전트 연동 방법, 기본 프롬프트 예시, `source_base_url` 설정 방법이 모두 포함

### 배포

**SC-28** [Blocking] Docker Compose 기동
- Given: 제공된 `docker-compose.yml`
- When: `docker compose up` 실행
- Then: backend + frontend + git server 모두 정상 기동, 로그인 화면 접근 가능

**SC-29** [Advisory] 설치형 직접 실행
- Given: Python/Node 환경 구성 완료
- When: 제공된 설치 지침대로 실행
- Then: Docker 없이도 서비스 정상 동작

**SC-30** [Advisory] LiteLLM 공급자 교체
- Given: Anthropic API 대신 Ollama URL로 설정 변경
- When: 서비스 재시작 후 Ingest/Edit 실행
- Then: Ollama 모델 경유로 위키 생성/수정 정상 동작

## Open Issues

- [x] [Advisory] LLM 추상화 레이어 범위 — LiteLLM으로 v0.1부터 추상화, Anthropic API + Ollama 지원으로 결정
- [x] [Advisory] Git 저장 방식 — normal repo (working tree 있음) 채택, bare repo 제외
- [x] [Advisory] 원본 파일 저장 — git 비추적, Docker Volume / Local Path 분리
- [x] [Advisory] MVP 파일 형식 — PDF, TXT, MD만 지원 (DOCX, URL 제외)
- [x] [Advisory] Lint — v0.2+로 보류
- [ ] [Advisory] 권한 관리 세분화 — MVP는 Admin/Member 2-role 직접 구현. v0.2+ 권한 원자화 시 Casbin 확장 가능성을 염두에 둔 설계.

## Changelog

- v0.1: 초기 SPEC (Requirements + Technical Design)
- v0.2: Breda 기술 검토 반영 — 저장 구조, Edit flow, Source 참조 설계, MVP 범위 조정
- v0.3: Hawkeye SC-1~SC-30 추가 (Blocking 22개 / Advisory 8개)
- v0.4: Job Queue, index.md/log.md, Known Limitation 추가; 삭제 v0.2+로 보류
- v0.5: Edit flow 확정 — 단일 페이지 컨텍스트 입력창, Jobs 탭 분리, log.md 기록 범위 한정 (job_id + 결과만, 요청 전문 제외)
