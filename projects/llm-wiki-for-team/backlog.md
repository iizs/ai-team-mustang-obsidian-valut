# Backlog

정렬되지 않은 아이디어/작업 풀. 이터레이션 킥오프 시 여기서 끌어와 우선순위·담당·DoD 부여한다.

> 이터레이션에서 작업 중인 항목은 [iterations/](iterations/) 참조.
> ID는 영구. 완료된 항목은 [완료된 항목](#완료된-항목-reference) 섹션 참조.

---

## 운영 안정화

### B-14. DB 스키마 마이그레이션 자동화 ⭐

- **배경**: v0.2에서 `User.created_at` 컬럼 추가했으나 기존 DB는 v0.1 스키마. SQLAlchemy `create_all`만 쓰는 구조 → 기존 인스턴스 로그인 깨짐. Kirin 회피: DB 삭제 + reseed.
- **방향 후보**:
  - alembic 도입 (Python 표준)
  - 기동 시 `inspect → ALTER` 자동 적용 (단순)
  - 마이그레이션 스크립트 폴더 + 수동 실행
- **부수 작업**:
  - SC 작성 가이드 보강: "기존 인스턴스 업그레이드" 시나리오를 SC에 포함하도록

### B-15. Race condition 명시 가드 (마지막 Admin 보호)

- **배경**: `_active_admin_count()` 호출과 commit 사이 시간차로 동시 다중 admin demote 시 우회 가능
- **현재 상태**: SQLite 직렬화로 사실상 보호. 내부 운영 위주라 실제 위험 거의 없음
- **개선**: `SELECT ... FOR UPDATE` 또는 명시적 트랜잭션 lock으로 명시화
- **참조**: [reviews/2026-05-08-hawkeye-v0.2-validation.md](reviews/2026-05-08-hawkeye-v0.2-validation.md)

---

## LLM Pipeline

### B-4. 단일 소스 → 다중 위키 페이지 분리 ⭐

- **배경**: 큰 원본(예: 30페이지 PDF)을 1개 위키 페이지로 합치면 가독성/검색성 저하. 주제 단위 분리가 위키 본질
- **현 상태 (설계상)**:
  - `=== FILE: name.md ===` 마커가 다중 페이지 출력 포맷
  - `parse_llm_pages`가 마커 단위 split
- **현 상태 (실측)**:
  - `prompts/ingest.txt` 예시가 단일 페이지뿐이라 LLM이 분리 시도 안 함
  - 결과적으로 1소스 → 1페이지 패턴 고착
- **필요 작업**:
  - 프롬프트에 "분리 기준" 명시 (주제 2개 이상 / 길이 상한)
  - 다중 페이지 예시 추가
  - 파일명 충돌 처리 강화
  - 페이지 간 자동 `[[링크]]` 가이드
- **열린 질문**:
  - 분리 결정을 LLM에 맡길지, 시스템 후처리로 분리할지
  - "축적 원칙" 하에서 기존 페이지가 있을 때의 분리 전략

### B-5. 프롬프트 다듬기 (weak 모델 호환)

- Ollama gemma 등 instruction following이 약한 모델 대응 강화
- few-shot 예시 1개 → 2~3개로 확장 (긴 문서, 한국어, 다중 페이지 케이스)
- 부정 예시 ("이렇게 답하지 마세요") 추가
- LiteLLM `response_format` / Ollama `format: json` 활용 검토
- 모델별 프롬프트 분기 가능성 검토 (운영 복잡도 트레이드오프)

---

## 보안 / 계정

### B-16. Rate limiting (signup 봇 방어)

- **배경**: SC-31 자가 가입은 v0.2에서 무방어. Public 운영 시 봇 가입 가능 — Kirin 결정으로 v0.3+ 보안 이터레이션 일괄 처리.
- **방향 후보**:
  - IP 기반 단순 rate limit (예: 1분 5회 가입 시도)
  - FastAPI 미들웨어 (slowapi 등)
  - Reverse proxy(nginx) 단계에서 처리
- **부수**: login 시도, source 업로드 등 다른 엔드포인트도 같이 검토

### B-17. 이메일 인증 (SMTP 도입)

- **배경**: v0.2 자가 가입은 즉시 활성. SMTP 미설정 회피 결정. 운영 단계에서는 이메일 인증 필요
- **필요 작업**:
  - SMTP 설정(`.env` 변수)
  - 가입 후 인증 메일 발송 → 링크 클릭 시 활성화
  - 미인증 사용자 처리 정책 (제한된 사용 vs 완전 차단)
  - 인증 토큰 모델 (DB) + 만료 정책

### B-18. `INITIAL_ADMIN_*` 환경변수 자동 시딩

- **배경**: v0.1에서 `seed_admin.py` CLI를 사후 추가. 첫 부팅 시 자동 시딩 옵션 있으면 운영 편의 ↑
- **방향**: `.env`의 `INITIAL_ADMIN_EMAIL`, `INITIAL_ADMIN_PASSWORD` 있고 사용자 0명일 때 자동 시딩 (한 번만)
- **주의**: 초기 부팅 후 .env 변경 → 재시딩 방지 로직 필요

---

## 컴포넌트 보강

### B-6. Lint 도입 ([ADR-0005](decisions/0005-lint-deferred.md) 해소)

- 위키 health-check, 스테일/모순 페이지 정리
- Job Queue에 LINT 타입 추가 가능성
- 범위: 전체 vs 변경분 vs 선택 페이지 — 비용 대비 가치 검토 필수

### B-7. 원본 soft-delete (SC-9)

- `_trash/` 디렉토리로 이동
- 활성 목록에서 제거
- Admin 권한
- Lint 도입 시 `_trash/` 원본을 참조하는 위키 페이지 정리 트리거 가능

### B-8. Job 병렬 처리

- v0.1: 워커 1개 (FIFO)
- v0.2+: 워커 N개 검토
- 동일 페이지 동시 수정 충돌 처리 정책 필요
- 실현 시 [ADR-0006](decisions/0006-fastapi-backgroundtasks.md) Superseded 가능성

### B-19. Sources 목록 페이징

- **배경**: Sources 목록도 시간 지나면서 누적되어 페이징 필요 (Kirin 2026-05-08 검토 중 발견)
- **방향**: Jobs 페이징(SC-36)과 동일한 패턴 — `?page=&size=`, 응답 `{items, total, page, size}`
- **필요 작업**:
  - 백엔드 `GET /api/sources/` 페이징 지원
  - 프론트 `/sources` UI에 페이지 크기 selector + Prev/Next + "Page N of M" + "No sources" 메시지
  - 정렬 기준 결정 (filename asc 또는 uploaded_at desc 등)

### B-20. UI 폴리싱 (B-3 이월)

- **상태**: B-3 본체는 v0.2에서 핵심만 완료. 아래는 자연 보강 또는 별도 이터레이션
- **항목**:
  - 페이지 정렬, 타이포그래피, 여백 일관성
  - 에러 표시 개선 (현재 `alert()` 기반 다수)
  - 로딩 상태 표시
  - 모바일 반응형
  - 시간 포맷 ko-KR (또는 i18n)

---

## 큰 기능 (가치 검증 후)

### B-9. SSO

- v0.1: 이메일/패스워드 + JWT
- v0.2+: SAML/OIDC 지원
- Keycloak 도입 검토

### B-10. 멀티 테넌시

- 한 Sheska 인스턴스에서 여러 팀(테넌트) 운영
- 위키 repo, 사용자 DB, Job Queue 격리

### B-11. 원본 출처 추적 가시화 (source graph)

- 위키 페이지 ↔ 원본 파일 의존 그래프 시각화
- 원본 변경 시 영향받는 위키 페이지 식별

### B-12. URL 소화 (크롤링)

- v0.1: 파일 업로드만
- v0.2+: URL 입력 → 크롤링 → INGEST
- robots.txt, 인증, 동적 콘텐츠 등 별도 영역

### B-13. DOCX 지원

- v0.1: PDF/TXT/MD만
- v0.2+: `python-docx` 도입 검토

### B-21. Ollama LLM 호환성 우회 제거 경로

- **배경**: v0.3 실사용에서 LiteLLM Ollama provider가 `response_format` / `format="json"` 어떤 JSON flag든 응답을 function call 구조로 해석 → `KeyError: 'arguments'` 발생 (`ollama.py:566`). 임시 조치로 **Ollama에서는 JSON 강제 자체를 미사용**, prompt 강제 + Pydantic + graceful degrade만으로 운영 중.
- **현 상태**: 임시 우회. ADR-0011 §결정.8 운영 변경 흐름 명시.
- **참조**: [reviews/2026-05-08-prompt-load-bug-postmortem.md](reviews/2026-05-08-prompt-load-bug-postmortem.md)
- **제거 경로 후보**:
  1. **LiteLLM 업그레이드** — Ollama provider line 566 버그 fix 확인 후 JSON flag 재도입 (가장 단순, 검증 비용 작음)
  2. **Ollama API 직접 호출** — httpx로 `/api/chat` 직접 사용. LiteLLM 우회. Ollama-native `format: "json"` 정상 사용 가능
  3. **OpenAI-compatible 서버 호스팅** — vLLM, Together 등 OpenAI-compatible 서버에 모델 호스팅. LiteLLM의 OpenAI 경로 사용. 운영 부담 ↑
- **결정 기준**: 우선 1번 시도 (비용 작음). 안 되면 2번 (Ollama 종속이지만 우리 도메인에선 큰 비용 X). 3번은 최후 카드.

### B-22. LLM 비용 telemetry

- **배경**: v0.4 agentic loop는 단일 명령에 수십 LLM 호출 가능. token 사용량/비용 추적 부재 시 운영 비용 폭발 위험을 사용자가 인지 못 함.
- **방향**:
  - 매 LLM 호출 후 `usage` (input/output tokens) 수집 → `Job.payload`에 누적
  - Jobs 탭에 job 단위 token 사용량 + 추정 비용 표시 (model별 단가 테이블)
  - 누적 통계 대시보드 (선택적, v0.5+)
- **참고**: Anthropic API 응답에 `usage` 필드 포함. 단가는 model별 다름.
- **확장**: 다른 provider도 동일 패턴 (OpenAI usage 등)

---

## 메타

### M-1. v0.3+ 킥오프 체크리스트

- [ ] v0.3 테마 결정 (운영 안정화 vs LLM 품질 vs 보안 베이스 등)
- [ ] B-6 Lint 도입 시기 결정
- [ ] GitHub Issue 도입 여부 재검토 (vault 내부 관리 vs 외부 추적)

---

## 완료된 항목 (Reference)

ID 영속성을 위해 보존. 상세는 해당 iteration 회고 참조.

### B-1. Web UI `[[wikilink]]` 클릭 이동 — Done (v0.2)

SC-20을 Advisory에서 Blocking으로 승격. `remark-wiki-link` 플러그인 + 미존재 페이지 안내 (SC-39).
- 회고: [iterations/v0.2.md](iterations/v0.2.md)

### B-2. 신규 계정 생성 UI — Done (v0.2)

자가 가입 (`/signup`, SC-31) + Admin 권한 부여 메뉴 (`/admin/users`, SC-32) + `SIGNUP_ENABLED` 환경변수 (SC-40).
이월 항목은 B-17(이메일 인증), B-18(INITIAL_ADMIN_* 시딩)로 분리.
- 회고: [iterations/v0.2.md](iterations/v0.2.md)

### B-3. UI 정돈 — Done (v0.2, 핵심만)

위키 페이지 레이아웃 (본문 → Properties 접힘 → Edit Request, SC-33/34/35), Jobs 페이징 (SC-36).
이월 폴리싱 항목은 B-20으로 분리.
- 회고: [iterations/v0.2.md](iterations/v0.2.md)
