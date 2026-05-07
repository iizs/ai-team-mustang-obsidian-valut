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
