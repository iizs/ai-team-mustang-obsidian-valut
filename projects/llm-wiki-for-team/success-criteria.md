# Success Criteria

검증 기준 목록. **ID는 영구**(번호 재사용 금지). 추가는 끝에 붙인다.

## 인증 / 사용자 관리

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

## 원본 투입

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

**SC-9** [Advisory / v0.2+] 원본 soft-delete
- Given: Admin 계정, 기존 원본 파일
- When: 삭제 요청
- Then: `_trash/` 하위로 이동, 원본 활성 목록에서 제거
- Note: MVP 제외, v0.2에서 구현

**SC-10** [Blocking] Member 원본 투입 차단
- Given: Member 계정
- When: 원본 파일 업로드 시도
- Then: 권한 오류 반환, 업로드 불가

## LLM Ingest

**SC-11** [Blocking] 원본 업로드 후 Job 등록 응답
- Given: Admin 계정, PDF/TXT/MD 파일
- When: 원본 파일 업로드
- Then: Source Store에 저장 성공 후 INGEST Job 큐 등록, 사용자에게 "큐 등록됨" 응답 즉시 반환

**SC-11-b** [Blocking] Ingest 완료 후 위키 페이지 생성 및 commit
- Given: INGEST Job 처리
- When: Ingest flow 완료
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

## LLM Edit

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

**SC-16-b** [Blocking] index.md Ingest/Edit 후 자동 갱신
- Given: INGEST 또는 EDIT Job 완료
- When: Wiki Store index.md 확인
- Then: 갱신된 `Last updated` 타임스탬프와 변경된 페이지 목록이 반영됨

**SC-16-c** [Blocking] log.md append 및 형식 준수
- Given: Job 완료 (INGEST 또는 EDIT, 성공/실패 모두)
- When: log.md 확인
- Then: 최신 항목이 상단에 추가되며 `job_id` + 결과(페이지 목록 또는 error)만 포함; 요청 전문(edit_text)은 기록되지 않음

## Jobs 탭

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

## 위키 조회 UI

**SC-18** [Blocking] 위키 페이지 목록 및 조회
- Given: Member 이상 계정으로 로그인
- When: Wiki UI 접근
- Then: 위키 페이지 목록 표시, 페이지 클릭 시 내용 조회 가능

**SC-19** [Blocking] 직접 편집 불가 (읽기 전용)
- Given: 위키 페이지 조회 중
- When: 텍스트 직접 편집 시도
- Then: 편집 불가, 수정 요청 입력창만 제공

**SC-20** [Blocking] 내부 링크 클릭 이동
- Given: 위키 페이지에 `[[다른페이지]]` 링크 존재
- When: 링크 클릭
- Then: 해당 위키 페이지로 이동
- Note: v0.1에서 Advisory였으나 v0.2에서 Blocking 승격

## 원본 조회 UI

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

## 로컬 Sync

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

## 로컬 연동 가이드

**SC-27** [Advisory] 가이드 문서 내용 완비
- Given: Sheska 배포
- When: 가이드 문서 페이지 접근
- Then: 에이전트 연동 방법, 기본 프롬프트 예시, `source_base_url` 설정 방법이 모두 포함

## 배포

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

## 자가 가입 / 사용자 관리 (v0.2 신규)

**SC-31** [Blocking] 자가 가입 (signup)
- Given: 비로그인 상태, `SIGNUP_ENABLED=true`
- When: `/signup`에서 이메일/패스워드 입력 후 등록
- Then:
  - 이메일 형식 검증 통과 (e.g., `not-an-email` 거부) 및 중복이 아닐 것 (중복 시 409)
  - 패스워드 최소 8자 이상 (미달 시 422)
  - 즉시 활성(`is_active=true`) 계정으로 생성, 기본 `member` role 부여
  - 가입 직후 **자동 로그인**(JWT 발급) 후 wiki 메인으로 이동
- Note: 이메일 인증 없음 (SMTP 미설정). Rate limit / CAPTCHA는 v0.3+ 보안 이터레이션에서 일괄 처리.

**SC-32** [Blocking] Admin 사용자 관리 메뉴 (목록 / role / 비활성화)
- Given: Admin 계정으로 로그인
- When: `/admin/users` 화면 접근
- Then:
  - 사용자 목록 조회 (이메일, role, `is_active`, 가입일)
  - 다른 사용자의 role 변경 (Member ↔ Admin) 가능
  - 다른 사용자의 `is_active` 토글 가능 (비활성 ↔ 재활성)
  - **자기 자신은 role 변경/비활성화 불가** (UI에서 비활성, API에서도 거부)
- Note: SC-3(Admin 직접 사용자 생성)은 별개로 유지됨 — 자가 가입과 별도로 Admin이 직접 생성하는 시나리오 보존.

## 위키 페이지 레이아웃 (v0.2 신규)

**SC-33** [Blocking] frontmatter는 페이지 속성 영역으로 분리
- Given: 위키 페이지 조회 중
- When: 페이지 렌더링 확인
- Then:
  - frontmatter 내용은 본문과 분리된 "Page Properties" 영역에 표(2-column key/value) 형식으로 표시
  - **라벨은 영문 키 그대로** (`type`, `created`, `last_updated`, `tags`, `sources` 등; v0.2 기준 i18n 미적용)
  - list 값(`tags`, `sources` 등)은 **콤마 구분** 단일 셀 (빈 list는 `—`)
  - 알 수 없는 frontmatter 키도 동일한 표 행으로 표시 (관용적)
  - 기본 접힌 상태이며 사용자가 펼칠 수 있음. 펼침/접힘 상태는 localStorage에 저장되어 다음 방문 시 유지

**SC-34** [Blocking] 위키 페이지 노출 순서
- Given: 위키 페이지 조회 중
- When: 페이지 렌더링 확인
- Then: 위에서 아래로 본문 → 페이지 속성(접힘) → 수정 요청 입력창 순서로 노출됨

**SC-35** [Blocking] 페이지 속성 sources 링크
- Given: 페이지 속성 영역에 `sources` 항목이 있는 위키 페이지 (Web UI 컨텍스트)
- When: `sources`의 항목 클릭
- Then:
  - Frontend가 `GET /api/sources/{filename}`을 **JWT 헤더와 함께 fetch**, 응답을 Blob으로 받아 다운로드 트리거 (단순 `<a href>` 링크 미사용 — SC-23 인증 충돌 방지)
  - 별도 미리보기 페이지 없음
  - 원본 파일 부재 시 사용자에게 오류 메시지 표시

## Jobs 탭 (v0.2 신규)

**SC-36** [Blocking] Jobs 탭 페이징
- Given: Jobs 탭 접근 (Admin 또는 Member)
- When: Job 개수가 페이지 크기를 초과
- Then:
  - 백엔드: `GET /api/jobs/?page=1&size=20` 쿼리 파라미터 지원, 응답은 `{items, total, page, size}`
  - 정렬: `created_at desc` 고정 (최신 우선)
  - 클라이언트 default 페이지 크기 20, 옵션 20 / 50 / 100 (selector)
  - 네비게이션: Prev / Next 버튼 + "Page N of M" 표시
  - 결과 0건 시 "No jobs" 안내 메시지 노출

## v0.2 신규 SC (보안·안전장치)

**SC-37** [Blocking] 마지막 Admin 보호
- Given: 시스템에 활성(`is_active=true`) Admin이 1명만 있는 상태
- When: 그 Admin을 Member로 강등 시도 또는 비활성화 시도 (UI 또는 API 직접 호출)
- Then: 거부 + 명시적 오류 메시지 ("Cannot demote/deactivate the last active admin"). 시스템에 Admin이 0명이 되는 상태는 항상 차단됨.

**SC-38** [Blocking] 비활성화 사용자 JWT 즉시 무효
- Given: 사용자가 Admin에 의해 비활성화됨(`is_active=false`)
- When: 해당 사용자가 만료 전 JWT로 보호된 API 호출
- Then: 인증 미들웨어가 `is_active` 체크 후 401 반환 (JWT TTL 만료를 기다리지 않음). 로그인 시도도 401.

**SC-39** [Blocking] 존재하지 않는 wikilink 동작
- Given: 위키 페이지 본문에 `[[NonExistentPage]]` 링크가 있음
- When: 사용자가 해당 링크 클릭
- Then: "This page does not yet exist" 안내 페이지로 이동 (404 페이지 구분 가능). v0.1 Karpathy 원칙 — 사용자 직접 생성 불가, LLM 경유로만 생성. 안내 페이지는 빈 페이지 생성을 유도하지 않고 정보만 제공.

**SC-40** [Blocking] `SIGNUP_ENABLED` 환경변수
- Given: 백엔드 환경변수 `SIGNUP_ENABLED=false`
- When:
  - (a) 비로그인 상태에서 `/signup` 페이지 접근
  - (b) `POST /api/auth/signup` 직접 호출
- Then:
  - (a) `/signup` 페이지가 표시되지 않거나 "Signup is disabled" 안내로 대체
  - (b) API는 403 Forbidden 반환
- Default: `SIGNUP_ENABLED=true` (자가 가입 허용)

## v0.3a — LLM 파이프라인 다단계화 (Phase A)

> ADR-0011 (Ingest 파이프라인 다단계화)에 따른 검증 기준. Phase A는 `create` 액션만 실행하고 `merge_into`/`supersede`는 log 표시만 한다.

**SC-41** [Blocking] Ingest LLM 컨텍스트에 `index.md` 주입
- Given: Wiki Store에 1개 이상 페이지가 존재하고 `index.md`가 존재
- When: 신규 INGEST Job이 처리되어 LLM 호출이 발생
- Then: LLM에 전달되는 컨텍스트(system 또는 user 메시지)에 `index.md` 내용이 포함됨. 빈 wiki(첫 ingest) 시에는 빈 index 표시 또는 "no existing pages" 안내가 포함됨.

**SC-42** [Blocking] LLM 출력 = JSON Plan 구조 + Pydantic 검증
- Given: Ingest 호출
- When: LLM 호출 시 `response_format={"type":"json_object"}` 인자를 일관되게 전달 (provider 무관, LiteLLM이 추상화)
- Then:
  - LLM 응답을 파싱하면 최상위 JSON 배열이며 각 항목은 `action ∈ {"create", "merge_into", "supersede"}` 필드를 포함
  - `action="create"` 시 `page_path`(kebab-case `.md`, path traversal 거부), `content`(frontmatter 포함 markdown 전체) 필수
  - `action="merge_into"` 시 `target`(.md), `merged_content` 필수
  - `action="supersede"` 시 `target`(.md), `reason`, `new_content` 필수
  - **Pydantic 모델로 스키마 검증** (action enum, 필드 존재, 타입). 검증 실패 시 SC-45로 위임

**SC-43** [Blocking] `create` 액션 즉시 실행 + 충돌 시 자동 강등
- Given: Plan에 `action="create"` 항목 N개
- When: Phase A 디스패치
- Then:
  - **신규(충돌 없음)**: `wiki-store/<page_path>`에 파일 생성 + git commit. `index.md`/`log.md` 갱신은 기존(SC-16-b/16-c)과 동일.
  - **충돌(같은 page_path 페이지가 이미 존재)**: 해당 항목을 **자동으로 `merge_into` 액션으로 강등**(`{action: "merge_into", target: <기존 page_path>, merged_content: <요청된 content>}`로 재구성). Phase A에서는 SC-44에 따라 skip + log 명시 (Phase B에서 실행). 사용자에게 데이터 손실 없음 보장.
  - 시스템이 강제 주입할 항목: `sources` frontmatter에 원본 파일명 보장 (LLM 누락 시 시스템이 자동 추가, SC-14 보존)

**SC-44** [Blocking] `merge_into` / `supersede` 액션 skip + log 표시 (Phase A)
- Given: Plan에 `action ∈ {"merge_into", "supersede"}` 항목이 포함됨 — LLM이 직접 출력했거나, SC-43의 충돌 강등으로 시스템이 자동 변환했거나, target invalid 케이스 모두 포함
- When: Phase A 디스패치
- Then:
  - 해당 액션은 **실행하지 않음** (대상 페이지 변경 없음)
  - `log.md`의 해당 Job 항목에 일관된 라인 형식으로 명시:
    - LLM 직접: `skipped (phase A): merge_into → [[target]]`
    - 충돌 강등: `skipped (phase A): merge_into ← create-conflict → [[target]]`
    - target invalid: `skipped (phase A): merge_into → [[target]] (target not found)`
    - supersede도 같은 패턴으로 표시
  - **Plan 전체 JSON을 `Job.payload`에 보존** (Jobs 탭에서 사용자 확인 + Phase B 도입 시 재실행/재검증 활용)
  - Job은 SUCCESS로 종료 가능 (skip만으로 FAILED 처리 X). `create` 액션이 함께 있고 정상 실행됐으면 그 부분은 적용됨

**SC-45** [Blocking] JSON 파싱 실패 시 graceful degrade + 명시 오류
- Given: LLM 출력이 유효한 JSON Plan이 아님 (parse 실패 또는 schema mismatch)
- When: 파이프라인이 응답 처리
- Then:
  - 1순위: 기존 4단계 fallback (`=== FILE: ===` 마커, frontmatter, dangling `---`, `#` 제목) 시도 → 페이지 생성 가능 시 정상 처리하되 `log.md`에 `legacy_fallback_used: <reason>` 표시
  - 2순위: fallback도 실패 시 Job FAILED + `error_msg`에 LLM 출력 첫 200~500자 보존 (SC-11-b silent SUCCESS 방지 원칙 유지)

**SC-46** [Advisory] 동일 주제 중복 ingest 시 plan에 `merge_into` 등장
- Given: 동일/유사 주제 원본을 두 번째로 ingest (첫 번째 ingest로 관련 페이지가 wiki에 존재)
- When: LLM이 plan을 출력
- Then: plan에 `action="merge_into"` 또는 `action="supersede"`가 등장하고 `target`이 기존 페이지 하나를 가리킴
- Note:
  - LLM 비결정성으로 인해 **자동 테스트 대상 아님** (회귀 판단 불가)
  - 운영 시 **실측 관측 후 대응** — 의미 있는 사례는 `reviews/YYYY-MM-DD-llm-merge-behavior.md`에 영구 보관 (Phase A 가치 검증 신호로 사용)
  - Phase A에서는 SC-44에 따라 skip + log로 가시성만 확보. Phase B 도입 후 본격 검증으로 승격 검토.

## v0.3a — 환경 초기화 스크립트

**SC-47** [Blocking] `init_env.py` 기본 동작 (interactive)
- Given: 새 환경에서 `python scripts/init_env.py` 실행, 옵션/환경변수 모두 미지정
- When: 사용자가 prompt에 admin 이메일/패스워드 응답
- Then:
  - DB 스키마 생성 (`create_all`, 기존 DB 있으면 보존)
  - `source_store_path` 디렉토리 생성 (없을 때만)
  - `wiki_store_path` 디렉토리 생성 + git init + `index.md`/`log.md`/`_sheska.yaml` 초기화 (없을 때만)
  - admin 계정 생성 (`role=admin`, `is_active=true`)
  - 같은 이메일 사용자가 이미 있으면 스크립트가 명시적 안내 후 중단 (덮어쓰기 안전장치)

**SC-48** [Blocking] `init_env.py --reset` 동작
- Given: 기존 DB/source-store/wiki-store가 존재하는 환경
- When: `python scripts/init_env.py --reset` 실행
- Then:
  - DB 파일, source store, wiki store **모두 삭제 후 재생성**
  - admin 계정 새로 생성
  - 이전 데이터 영구 손실 — 실행 전 사용자 확인 prompt 필수 (`Are you sure? [y/N]`)

**SC-49** [Blocking] `init_env.py --non-interactive` 모드
- Given: `INITIAL_ADMIN_EMAIL`, `INITIAL_ADMIN_PASSWORD` 환경변수 또는 `--admin-email`/`--admin-password` 인자 제공
- When: `python scripts/init_env.py --non-interactive` 실행
- Then:
  - prompt 없이 즉시 실행 (CI 등 자동화 가능)
  - 필수 값 미지정 시 명시적 오류 후 비-zero exit code (silent fail X)

**SC-50** [Blocking] `init_env.py` 환경변수 default 채우기
- Given: 환경변수 `INITIAL_ADMIN_EMAIL=admin@example.com` 설정 + interactive 실행
- When: `python scripts/init_env.py` 실행 (옵션 미지정)
- Then: prompt에 환경변수 값이 default로 미리 채워짐 (사용자가 Enter만 눌러도 수용 가능). password도 동일.

**SC-51** [Blocking] Plan frontmatter `type` enum 위반 시 시스템 보정
- Given: LLM이 출력한 plan의 `create`/`merge_into`/`supersede` 항목 `content`/`merged_content`/`new_content` 안의 frontmatter `type` 값이 SC-12 enum (`concept | process | policy | reference | glossary`)에 포함되지 않음
- When: 시스템이 plan을 처리
- Then:
  - 해당 항목의 `type`을 **`reference`로 자동 보정** (v0.2 자동 frontmatter 정책과 일관)
  - `log.md`에 `frontmatter_type_normalized: <원래 값> → reference` 표시 (가시성)
  - 항목 자체는 정상 처리 (FAILED 아님). 데이터 보존 우선.
