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

## v0.3.1 — Phase B 액션 실행 + Wiki Command + Page Delete

> ADR-0012 (Phase B)에 따른 검증 기준. Phase A에서 skip이었던 `merge_into`/`supersede`를 정상 실행으로 전환하고, 신규 `delete` 액션 + WIKI_COMMAND Job 타입 도입.

**SC-52** [Blocking] `merge_into` 액션 정상 실행
- Given: Plan에 `action="merge_into"`, `target`이 wiki-store에 실제 존재하는 페이지
- When: Phase B 디스패치
- Then:
  - 기존 target 페이지 read → **Plan의 `merged_content`를 그대로 적용**(재호출 없음, Kirin 결정 옵션 a) + git commit
  - SC-12 frontmatter 5필드 보존 (LLM이 빠뜨리면 시스템 보강)
  - **frontmatter 보존 정책 (시스템 적용, LLM 출력보다 우선):**
    - `created`: **기존 보존** (변경 금지)
    - `last_updated`: **갱신** (현재 시각)
    - `tags`: 기존 ∪ 새 content tags (set union, 중복 제거)
    - `type`: **기존 우선 유지** (SC-12 enum 안정성)
    - `sources`: 기존 ∪ 새 content sources (set union, 중복 제거). **INGEST 컨텍스트일 때만** 시스템이 source filename 강제 주입(SC-14 보존). WIKI_COMMAND는 source 개념 없으므로 강제 주입 없음.
  - SC-51 type enum 보정 적용 (위반 값일 때만)
  - log.md에 `executed: merge_into → [[target]]` 라인 추가 (Phase A의 `skipped` 대체)
- Note: target invalid(존재하지 않는 페이지)는 SC-44 패턴 그대로 skip + log "(target not found)". 통합 품질 부족이 실측에서 드러나면 v0.4+에서 옵션 b(재호출 보강)로 진화 검토.

**SC-53** [Blocking] `supersede` 액션 정상 실행
- Given: Plan에 `action="supersede"`, `target`/`reason`/`new_content` 모두 유효
- When: Phase B 디스패치
- Then:
  - 기존 target 페이지 read → **Plan의 `new_content`를 그대로 적용**(재호출 없음, Kirin 결정 옵션 a)으로 완전 대체 + git commit
  - log.md에 `executed: supersede → [[target]] (reason: <reason>)` 라인 추가
  - reason은 LLM이 표현한 갈아엎기 사유로 그대로 보존 (요약/변형 X)
  - frontmatter는 SC-52의 보존 정책(created 보존, last_updated 갱신, type 기존 유지) 동일 적용. supersede는 본문 갈아엎기지만 메타 일관성을 위해 동일 규칙.
  - SC-51 type 보정 적용
- Note: target invalid 시 SC-44 패턴 동일 (skip + log "target not found")

**SC-54** [Blocking] `delete` 액션 정상 실행 (WIKI_COMMAND 전용)
- Given: **WIKI_COMMAND** Job의 Plan에 `action="delete"`, `target`이 일반 위키 페이지 (예약 파일 X), `reason` 포함
- When: Phase B 디스패치
- Then:
  - target 파일이 git index와 working tree에서 제거됨 + git commit ("delete: <target> [job:<id>]")
  - 구현 방식은 `git rm` 또는 `unlink + repo.index.remove(working_tree=True)` 모두 허용 (결과 동등). 실제 구현은 후자.
  - log.md에 `executed: delete → [[target]] (reason: <reason>)` 라인 추가
- Note:
  - 삭제 후 다른 페이지의 `[[삭제된페이지]]` 링크는 SC-39 안내 페이지로 자연 노출 (Lint(B-6)가 사후 정리)
  - **delete는 WIKI_COMMAND에서만 허용** — INGEST plan에서 delete 등장 시 거부 (SC-63 참조). Kirin 결정: "INGEST는 축적 원칙, 삭제는 사용자 명시 명령일 때만"

**SC-55** [Blocking] `create` 충돌 자동 강등의 Phase B 정상 실행
- Given: Plan의 `create` 항목이 기존 페이지명과 충돌 → SC-43에 따라 시스템이 자동 `merge_into`로 강등
- When: Phase B 디스패치 (Phase A의 skip 제거)
- Then:
  - 강등된 `merge_into`가 SC-52에 따라 정상 실행
  - log.md에 `executed: merge_into ← create-conflict → [[target]]` 라인 (Phase A `skipped` 형식 대체)

**SC-56** [Blocking] `delete` 안전장치 — 예약 파일 보호
- Given: Plan의 `delete` 항목 `target`이 예약 파일(`index.md`, `log.md`, `_sheska.yaml`) 또는 `_`로 시작하는 시스템 파일
- When: Phase B 디스패치
- Then:
  - 해당 항목 거부 (파일 변경 없음)
  - log.md에 `rejected: delete → [[target]] (reserved file)` 라인 추가
  - 같은 plan의 다른 액션은 정상 처리 (전체 Job FAILED는 아님). plan_summary에 `rejected` outcome 보존

**SC-57** [Blocking] `WIKI_COMMAND` Job 등록 + 즉시 응답
- Given: 인증된 사용자 — **Member 또는 Admin 모두 허용** (Kirin 결정: 옵션 a, Edit 권한과 일관). 비인증 호출은 401.
- When: `POST /api/wiki/commands` 본문 `{"command_text": "<자연어>"}`
- Then:
  - `WIKI_COMMAND` 타입 Job을 큐에 등록 (payload `{command_text}`, `created_by=현재 user`)
  - 즉시 `{"job_id": "...", "status": "queued"}` 응답 (HTTP 202)
  - SC-17-a/b 회귀 — Member는 본인 WIKI_COMMAND Job만, Admin은 전체 조회 가능
- Note: command_text 빈 문자열/공백만 입력 시 422 (사전 검증). v0.4+ 권한 세분화 도입 시 별도 정책 검토.

**SC-58** [Blocking] `WIKI_COMMAND` 실행 → Plan 생성 → 액션 순차 실행
- Given: WIKI_COMMAND Job이 워커에 픽업됨
- When: `run_wiki_command()` 호출
- Then:
  - LLM에 `command_text` + `index.md` 컨텍스트 + `prompts/wiki_command.txt` 전달
  - LLM이 Plan(JSON, INGEST 동일 스키마) 출력
  - Plan 액션을 순차 실행 (create/merge_into/supersede/delete 모두 SC-43/52/53/54 적용)
  - log.md/index.md 갱신 (SC-16-b/c 동일 동작)
  - **JSON 파싱 실패 시 즉시 Job FAILED + LLM 출력 snippet `error_msg` 보존**. legacy fallback 미적용 (INGEST와 다른 정책 — WIKI_COMMAND는 prompt를 우리가 통제하므로 legacy 형식 출현 가능성이 의미 없음)

**SC-59** [Blocking] `WIKI_COMMAND` 빈 plan 처리
- Given: LLM이 유효 JSON이지만 `actions=[]` (해석 실패 또는 변경 불필요로 판단)
- When: 디스패치
- Then:
  - 파일 변경 없음, Job SUCCESS 종료
  - log.md에 `WIKI_COMMAND | SUCCESS | job_id: <id>` + `- note: no actions in plan` 라인
  - **FAILED 처리하지 않음** — LLM이 "할 일 없음"으로 판단하는 케이스를 정상 동작으로 인정

**SC-60** [Blocking] Wiki Command UI 동작
- Given: 인증된 사용자가 `/command` 페이지 또는 nav 진입점에 접근
- When: 자연어 입력창에 명령 입력 후 Submit
- Then:
  - `POST /api/wiki/commands` 호출 → "queued" 응답 표시
  - Jobs 탭에서 해당 WIKI_COMMAND Job 진행 상태 추적 가능 (SC-17-a/b 회귀)
  - 빈 입력은 클라이언트에서 비활성/거부

**SC-61** [Blocking] log.md `executed:` / `rejected:` / `skipped:` 라인 형식 통일
- Given: Phase B 액션 처리 완료
- When: log.md 확인
- Then: 라인 형식이 일관되게 다음 패턴 중 하나:
  - `- executed: merge_into → [[target]]`
  - `- executed: merge_into ← create-conflict → [[target]]`
  - `- executed: supersede → [[target]] (reason: ...)`
  - `- executed: delete → [[target]] (reason: ...)`
  - `- rejected: delete → [[target]] (reserved file)` — 예약 파일 보호 (SC-56)
  - `- rejected: delete → [[target]] (forbidden in INGEST)` — INGEST에서 delete 시도 (SC-63)
  - `- skipped: merge_into → [[target]] (target not found)` — invalid target (SC-44)
  - `- failed: <action> → [[target]] (error: ...)` — 액션 실행 중 예외 (다른 액션은 계속 진행)
- 라인 prefix(`executed`/`rejected`/`skipped`/`failed`)로 outcome 분리 추적 가능

**SC-62** [Advisory] 삭제된 페이지 `[[link]]` 클릭 시 SC-39 안내 (회귀)
- Given: 페이지 P 삭제 후, 다른 페이지에 `[[P]]` 링크가 잔존
- When: 사용자가 `[[P]]` 클릭
- Then: SC-39 "This page does not yet exist" 안내 페이지로 자연 노출 (회귀 보장)
- Note: Lint(B-6)가 도입되면 잔존 링크 정리. v0.3.1에서는 자연 동작에만 의존.

**SC-63** [Blocking] INGEST plan에서 `delete` 액션 거부
- Given: INGEST Job의 LLM 출력 Plan에 `action="delete"` 항목이 포함됨
- When: 시스템이 plan을 처리
- Then:
  - 해당 항목 거부 (파일 변경 없음)
  - log.md에 `rejected: delete → [[target]] (forbidden in INGEST)` 라인 추가
  - 같은 plan의 다른 액션(create/merge_into/supersede)은 정상 처리 — 전체 Job FAILED 아님
  - plan_summary에 outcome `rejected_forbidden_in_ingest`로 보존 (Jobs 탭 가시성)
- Note: Kirin 결정 — INGEST는 축적 원칙(LLM은 축적, 삭제는 사용자 명시 명령), delete는 WIKI_COMMAND에서만 허용. Plan validator 또는 dispatcher 단계에서 차단.

## v0.4 — Agentic Wiki Operations + Backlink Index

> ADR-0013 / ADR-0014 / ADR-0015에 따른 검증 기준. 정적 Plan 모델에서 agentic loop로 전환, Anthropic 우선, Ollama는 v0.3.1 흐름 유지.

### Agentic Loop 기본

**SC-64** [Blocking] Agentic loop 기본 동작 + 자연 종료
- Given: Anthropic provider 환경, 사용자가 INGEST 또는 WIKI_COMMAND 트리거
- When: `run_agentic_loop()` 실행
- Then:
  - LLM에 system prompt(`prompts/agent_system.txt`) + user request + tool 카탈로그 전달
  - LLM이 tool 호출 → 시스템이 dispatch → 결과를 `tool_result` 메시지로 다시 LLM에 전달
  - LLM이 tool 호출 없는 응답을 반환하면 **자연 종료** → Job SUCCESS
  - 한 번이라도 tool 호출 발생하면 모든 step이 `Job.payload.steps`에 기록됨

**SC-65** [Blocking] Loop 안전장치 5종
- Given: Agentic loop 실행 중
- When: 다음 조건 중 하나가 트리거됨
- Then:
  - **`max_iterations` 초과** (기본 20회): loop abort + Job FAILED + `error_msg`에 "max_iterations reached"
  - **`max_tool_calls` 초과** (기본 50회): 동일
  - **`timeout` 초과** (기본 300초): 동일
  - **이상 패턴 감지** — 같은 tool + 동일 인자(hash)가 연속 3회 호출되면 abort + Job FAILED + `error_msg`에 "stuck pattern detected: <tool>(<args_hash>)"
  - **`stop_reason == "max_tokens"`** — Anthropic API가 context limit 도달로 응답을 잘랐을 때: loop abort + Job FAILED + `error_msg`에 "LLM hit context limit (max_tokens stop_reason)"
- 임계값(`MAX_ITERATIONS`, `MAX_TOOL_CALLS`, `LOOP_TIMEOUT_SEC`, `STUCK_PATTERN_THRESHOLD`)은 `.env`로 조정 가능

**SC-66** [Blocking] 사용자 취소 (graceful abort)
- Given: Agentic loop 진행 중인 Job (`status=processing`)
- When: 사용자가 **`POST /api/jobs/{id}/cancel`** 호출 (UI Cancel 버튼이 이 엔드포인트 사용)
- Then:
  - API가 `Job.status = cancelling`으로 설정 + 즉시 응답 (loop abort는 비동기)
  - 다음 iteration 진입 전 시스템이 상태 체크 → loop abort
  - `Job.status = cancelled` 최종 마킹 (**`JobStatus.cancelled` 신규 enum**, `failed`와 구분)
  - **이미 commit된 step은 롤백 안 함** (매 step commit이므로 부분 변경 유지)
  - `Job.payload.steps`에 마지막 step + `{aborted: true, reason: "user_cancelled"}` 기록
  - `log.md`에 `cancelled: job <id> at step <n>` 라인 추가
- Note: cancel API는 Job 소유자(`created_by == current_user`) 또는 Admin만 호출 가능 (SC-17-a/b 권한 정책 유지). 이미 종료된 Job(`done/failed/cancelled`)에 cancel 호출 시 409 또는 idempotent 무시

**SC-67** [Blocking] Progress checkpoint — `Job.payload.steps`
- Given: Agentic loop 실행 중
- When: 매 tool 호출 후
- Then:
  - `Job.payload.steps` 배열에 항목 push:
    - `{step_no: int, tool_name: str, args_snippet: str, result_snippet: str, ts: ISO8601, status: "ok"|"error"}`
  - **args_snippet / result_snippet은 500자 truncate** (큰 content는 git log/wiki-store에서 풀 확인)
  - DB commit (매 step 후) — 도중 중단되어도 진행상황 보존
  - Jobs 탭 polling으로 step 목록 + 진행 상태 표시 가능

### Tool 카탈로그 (5건)

**SC-68** [Blocking] Read 계열 tool 동작 (read-only, 자유 호출)
- Given: Agentic loop 중 LLM이 read-only tool 호출
- When: 다음 tool 중 하나가 호출됨
- Then:
  - **`get_index()`** — `wiki-store/index.md` 내용 + 표제어/한줄요약 표 반환
  - **`read_page(path)`** — frontmatter dict + body str 반환. 페이지 부재 시 `null`
  - **`read_source(filename)`** — Source Store에서 원본 텍스트 추출 (.pdf는 pypdf로 추출). 부재 시 명시 오류
  - **`search_pages(query)`** — 매칭 페이지 목록 + 매칭 라인 snippet (frontmatter+body 모두 매칭)
  - 모두 부작용 없음 (파일 변경 X, commit X). Job.payload.steps에 기록만.

**SC-69** [Blocking] `get_backlinks(page)` tool 동작
- Given: Agentic loop 중 LLM이 `get_backlinks(page)` 호출
- When: tool 실행
- Then:
  - target 페이지의 frontmatter `backlinks` 필드 조회 (없으면 빈 리스트)
  - 페이지 부재 시 명시 오류 (null이 아닌 error 응답으로 LLM이 인지)
  - ADR-0015 자동 갱신 결과가 정확히 반영됨

**SC-70** [Blocking] `write_page` 동작 + 안전장치
- Given: Agentic loop 중 LLM이 `write_page(path, content)` 호출
- When: tool 실행
- Then:
  - 신규 또는 전체 덮어쓰기로 `wiki-store/<path>` 저장
  - **예약 파일(`index.md`/`log.md`/`_sheska.yaml`/`_`로 시작) 거부** + tool 결과에 명시
  - path traversal 거부 (`../`/절대경로/non-kebab-case + `.md` 외 확장자)
  - `content`의 frontmatter `type` enum 위반 시 SC-51처럼 `reference`로 보정
  - 매 호출 = 1 git commit (SC-78)
  - backlinks 자동 갱신 (SC-75) 한 commit에 묶음

**SC-71** [Blocking] `patch_page` 동작 — Unified Diff 패턴
- Given: Agentic loop 중 LLM이 `patch_page(path, diff)` 호출 (`diff`는 표준 **unified diff** 형식, 5-line context 권장)
- When: tool 실행
- Then:
  - 시스템이 기존 페이지를 읽고, diff의 각 hunk를 **context fuzzy match**로 적용 (line number는 무시, context line으로 위치 찾기)
  - 결과 응답: `{"applied": bool, "hunks_applied": int, "hunks_failed": [{hunk_idx, reason, context_excerpt}], "error": str|null}`
  - **각 hunk별로 독립 적용** — 한 hunk fail이 다른 hunk 적용을 막지 않음
  - **hunks_applied == 0** → 파일 변경 없음, commit 안 함 (FAILED 아님; LLM이 결과 보고 재시도 가능)
  - **hunks_applied >= 1** → 변경 적용 + 1 commit (SC-78 패턴) + backlinks 자동 갱신
  - 적용 후 frontmatter `last_updated`는 시스템이 자동 갱신 (LLM이 diff에 포함 안 해도 됨)
  - 예약 파일(`index.md`/`log.md`/`_sheska.yaml`/`_*`) / path traversal 거부 → `error="reserved file"` 또는 `"invalid path"`
  - 새 페이지(파일 부재) 시 patch_page 거부 → LLM이 `write_page` 사용하도록 유도. `error="page not found; use write_page for new pages"`
- Note:
  - LLM은 diff format에 정확한 line number를 못 맞춰도 됨 — context line만 정확하면 위치 fuzzy match로 안전 적용
  - hunk fail 발생 시 LLM은 `read_page`로 최신 상태 확인 후 새 diff로 재시도 가능 (loop 안전장치 SC-65에 의해 최대 시도 횟수 제한)
  - 위키 본문은 짧은 줄 + 자연어가 많아 context 3줄로는 모자랄 수 있어 5줄 권장

**SC-72** [Blocking] `delete_page` 동작 + 안전장치
- Given: Agentic loop 중 LLM이 `delete_page(path, reason)` 호출
- When: tool 실행
- Then:
  - 일반 위키 페이지 삭제 (git에서 제거) + commit
  - **예약 파일 거부** (SC-56 패턴 유지)
  - **INGEST job 컨텍스트에서는 거부** (SC-63 정신 유지) + 결과에 "forbidden in INGEST"
  - target 부재 시 skip + 결과 "page not found"
  - 삭제 후 backlinks 정리 자동 (SC-77)

### Workflow 통합 / Provider 분기 (2건)

**SC-73** [Blocking] INGEST + WIKI_COMMAND 통합 agentic 흐름 (Anthropic)
- Given: Anthropic provider 환경, INGEST 또는 WIKI_COMMAND Job
- When: 워커가 픽업
- Then:
  - 둘 다 동일 `run_agentic_loop()` 호출 (내부 흐름 동일)
  - INGEST Job은 user request에 `source_filename` + 원본 텍스트 hint 포함, WIKI_COMMAND는 `command_text`만
  - 정적 `Plan`/`PlanAction` Pydantic 폐기 — `Job.payload.steps` 누적이 plan 역할 대체
  - log.md/index.md 갱신 동작은 기존 SC-16-b/c 유지

**SC-74** [Blocking] Provider 제한 — Anthropic only (v0.4)
- Given: `.env`의 `LITELLM_PROVIDER`가 Anthropic이 아님 (e.g., `ollama`, `openai`, `gemini` 등)
- When: INGEST 또는 WIKI_COMMAND Job 트리거
- Then:
  - **즉시 Job FAILED** + `error_msg`에 명시: "This provider (<provider>) does not support agentic mode (v0.4). Use Anthropic, or downgrade to v0.3.1 for legacy provider support."
  - `/guide` 페이지에 "v0.4 requires Anthropic provider" 안내 추가
  - v0.3.1 단순 chat/Plan 흐름은 v0.4 코드베이스에서 **제거됨** (silent fallback 없음, Kirin 결정)
- Note:
  - v0.4 Anthropic-only 단순화 결정 (ADR-0013)
  - Ollama/기타 provider는 v0.3.1 인스턴스로 운영하거나 v0.5+ OllamaNativeAdapter(B-21) 도입 대기
  - Anthropic API 자체 오류(인증 실패 / rate limit 등)는 일반 LLM 호출 오류로 처리

### Backlink Index (3건)

**SC-75** [Blocking] Backlink frontmatter 자동 갱신
- Given: `write_page`/`patch_page`/`delete_page` tool 실행 완료
- When: 시스템 hook이 변경된 페이지의 outgoing 링크 분석
- Then:
  - 변경된 페이지의 body에서 `[[X]]` 패턴 추출 (outgoing)
  - 이전 outgoing과 비교해 추가/삭제 식별
  - 추가된 X: X 페이지의 frontmatter `backlinks`에 현재 페이지 stem 추가 (set union, 중복 제거)
  - 삭제된 X: X 페이지의 frontmatter `backlinks`에서 현재 페이지 stem 제거
  - 모든 backlinks 갱신은 같은 tool 호출의 commit에 묶여 atomic 처리
  - LLM은 갱신에 관여하지 않음 (시스템 자동, 결정적)

**SC-76** [Blocking] Backlink 마이그레이션 (`init_backlinks.py`)
- Given: 기존 위키에 `backlinks` 필드가 없는 페이지들이 존재
- When: `python scripts/init_backlinks.py` 실행 또는 `init_env.py --reset` 후 자동 트리거
- Then:
  - 모든 위키 페이지의 body에서 `[[X]]` 추출 → 역색인 구축
  - 각 페이지의 frontmatter에 `backlinks: [...]` 작성 (기존 필드 보존)
  - 단일 commit으로 모든 변경 묶기 (`migrate: backlinks index initial build`)
  - 멱등성 — 다시 실행해도 결과 동일

**SC-77** [Blocking] 페이지 삭제 시 backlinks 자동 정리
- Given: 페이지 P가 `delete_page`로 삭제됨
- When: 시스템 hook 실행
- Then:
  - P가 frontmatter.backlinks에 등록된 모든 페이지에서 P stem 제거
  - **본문의 `[[P]]` 링크 자체는 보존** (Lint v0.5+가 정리, SC-39 안내로 자연 노출)
  - 정리는 같은 commit에 묶임

### Commit / UI (3건)

**SC-78** [Blocking] Write tool 호출당 1 commit + message 패턴
- Given: `write_page`/`patch_page`/`delete_page` tool 실행 + 실제 변경 발생
- When: 시스템이 git commit
- Then:
  - 매 호출당 1 commit (v0.4 정책; 1 액션 = 1 commit는 v0.5+)
  - commit message: `[job:<id> step:<n>] <tool>: <path>` (예: `[job:abc123 step:3] patch_page: payment.md`)
  - 같은 commit에 backlinks 갱신 변경분도 포함 (atomic)
  - `log.md`에는 job 단위 요약만 기록 (시작/종료/요약, 매 step 기록 X)

**SC-79** [Blocking] Frontend — Properties 표 backlinks 클릭 가능
- Given: 위키 페이지 조회 시 frontmatter에 `backlinks: [페이지A, 페이지B]`
- When: Page Properties 영역 펼침
- Then:
  - backlinks 값이 클릭 가능한 wikilink로 렌더 (`/wiki/<stem>`)
  - 빈 리스트는 `—`로 표시 (SC-33 정책)
  - 클릭 시 해당 페이지로 이동 (SC-20)

**SC-80** [Blocking] Frontend — Jobs 탭 진행 표시 + 취소
- Given: 진행 중 또는 완료된 Agentic Job (INGEST/WIKI_COMMAND)
- When: Jobs 탭에서 해당 Job 항목 확인
- Then:
  - `Job.payload.steps` 배열 노출 (step_no / tool_name / args_snippet / result_snippet / ts / status)
  - 진행 중인 Job(`status=processing`)에 **Cancel 버튼** 노출 → 클릭 시 **`POST /api/jobs/{id}/cancel`** 호출
  - 완료/실패/취소 Job(`status ∈ {done, failed, cancelled}`)은 Cancel 버튼 비활성 (또는 미노출)
  - 취소 후 Job 표시 상태가 자연스럽게 `cancelled` 라벨로 갱신 (polling)
  - SC-17-a/b 권한 회귀 (Member 본인 / Admin 전체)

## v0.5 — Lint Issue Tracking System

> ADR-0017 (Lint Tier Model) + ADR-0018 (Lint Finding Tracker)에 따른 검증 기준. v0.5는 **인프라만** — scanner/agent 없음, 사용자 신고가 유일한 active source.

**SC-81** [Blocking] 사용자 신고 (POST `/api/lint/findings`) — 권한 + 자동 채움
- Given: 인증된 사용자(Member 또는 Admin)
- When: `POST /api/lint/findings` 본문 `{page_path?: str, description: str, category?: str}`
- Then:
  - 신규 finding 생성 + 201 응답
  - 자동 채움 필드:
    - `source = "user:web"` (강제 — 클라이언트 입력 무시)
    - `status = "open"` (default)
    - `reported_by = current_user.email`
    - `category = user_reported` (지정 안 했을 때 default)
    - `created_at = now`
  - `description` 필수 — 누락 또는 빈 문자열 시 422
  - `page_path`는 **optional** (ADR-0018 데이터 모델 정합 — 위키 전체 통계 finding 등 페이지 비종속 케이스 수용). 사용자 신고 UI에서는 항상 현재 page_path를 첨부하지만 API 차원에서는 nullable.
  - 비인증 호출은 401

**SC-82** [Blocking] 목록 조회 (GET `/api/lint/findings`)
- Given: 인증된 사용자 (Member/Admin)
- When: `GET /api/lint/findings?status=&category=&source=&page=&size=`
- Then:
  - 응답 `{items, total, page, size}` (Jobs 탭과 일관)
  - 정렬: `created_at desc` 고정
  - 필터 미지정 시 전체 조회
  - 클라이언트 default 페이지 크기 20, 옵션 20/50/100 (selector)
  - 결과 0건 시 빈 items 배열 + total=0
  - 단건 조회 `GET /api/lint/findings/{id}` 존재, 부재 시 404

**SC-83** [Blocking] 상태 변경 (PATCH `/api/lint/findings/{id}`) + 권한 + Audit
- Given: **Admin 계정**으로 로그인 (Kirin 결정 — 처리 권한은 Admin 전용, 거대화 회피)
- When: `PATCH /api/lint/findings/{id}` 본문 `{status: "acknowledged"|"wont_fix", resolution_reason: str}`
- Then:
  - status 변경 + `decided_by = current_user.email` + `decided_at = now` 시스템 자동 채움
  - `resolution_reason` 필수 (audit 추적성 보장) — 빈 문자열 거부 (422)
  - Member가 호출 시 403
  - 비인증 호출은 401

**SC-84** [Blocking] Enum 검증 + status 한 방향 흐름
- Given: 인증된 Admin
- When: PATCH 요청에서 잘못된 값 전달
- Then:
  - 정의되지 않은 `category`/`source`/`status` 값 → 422
  - **status 흐름은 한 방향**: `open → acknowledged` 또는 `open → wont_fix`만 허용
  - `acknowledged → wont_fix`, `wont_fix → acknowledged`, `* → open` 등 역방향/측면 전이 → 409 또는 422
  - 동일 status 재설정 (`open → open`) → 409 또는 idempotent 무시 (구현 일관)

**SC-85** [Blocking] DELETE 미지원 (Audit Trail 보존)
- Given: 임의 finding `id`
- When: `DELETE /api/lint/findings/{id}` 호출
- Then:
  - 405 Method Not Allowed (또는 라우트 미정의로 404)
  - finding은 어떤 status에서도 삭제 불가
  - 이력은 영구 보존 — 결정자/시점/이유 추적 가능

**SC-86** [Blocking] 사용자 신고 UI — wiki 페이지 "문제 신고" 버튼
- Given: 인증 사용자가 `/wiki/[...slug]` 페이지 조회 중
- When: "문제 신고" 버튼 클릭 → modal 표시 → category 선택 (default `user_reported`) + description text area + 제출
- Then:
  - `POST /api/lint/findings` 호출 (현재 page_path + description + category)
  - 성공 시 "신고가 접수되었습니다" 안내 + modal 닫힘
  - 빈 description은 클라이언트에서 거부
  - 비로그인 상태에서 버튼 비활성 또는 미노출 (SC-2 회귀)

**SC-87** [Blocking] `/lint` 페이지 — 목록 + 필터 + 페이지네이션
- Given: 인증된 사용자가 `/lint` 페이지 접근 (nav menu 진입점 존재)
- When: 페이지 로드
- Then:
  - finding 목록 표 노출 (created_at desc 정렬)
  - 필터: status (default `open`), category, source — Member/Admin 모두 조회 가능
  - 페이지네이션: Prev/Next + "Page N of M" + 페이지 크기 selector 20/50/100
  - 결과 0건 시 "No findings" 안내 메시지
  - 항목 클릭 시 상세 view 또는 인라인 확장

**SC-88** [Blocking] `/lint` 상세 + 처리 UI
- Given: Admin이 `/lint` 페이지에서 finding 항목 클릭 (상세 진입)
- When: 상세 view 노출
- Then:
  - finding 메타 표시 (id/category/source/status/page_path/description/details/created_at/reported_by)
  - status가 `open`이면 **"Acknowledge" / "Won't Fix" 버튼 + reason 입력창** 노출
  - status가 `acknowledged`/`wont_fix`면 결정 메타(decided_by/at/resolution_reason) 표시 + 버튼 비활성/미노출
  - Member 사용자는 상세 view에 접근하지만 처리 버튼 미노출 (SC-83 권한 일관)
  - reason 빈 입력 시 클라이언트에서 거부
