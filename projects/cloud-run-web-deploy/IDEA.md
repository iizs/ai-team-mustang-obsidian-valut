# cloud-run-web-deploy

Mustang 팀 웹 프로젝트의 GCP 배포 표준. 2026-07-24 초안, hello-world-nextjs로 end-to-end 실증 완료.

Contributors: Breda (검증/스펙), Kirin (요구사항/승인)

---

## 배경

로컬 실행에 머물던 웹 프로젝트를 GCP에 올려 상시 접근 가능하게 하고, agent 팀이 push-to-deploy 파이프라인을 자율적으로 운용할 수 있는 표준을 만든다.

- 대상: **stateless 경량 웹 프로젝트** (저장소 이슈 없음)
- 비대상: Sheska류의 상태 있는 앱 (SQLite/git 저장소 필요) → 별도 검토 필요
- 목표: 새 프로젝트 하나 세팅에 사람 손 최소화 (`cp -r` 템플릿 + 스크립트 몇 줄)

---

## 표준 결정 사항 (2026-07-24)

### GCP 레이아웃

| 항목 | 값 | 비고 |
|---|---|---|
| Project | `mustang-web-apps` | 신규, 권한 분리를 위해 기존 프로젝트와 별개 |
| Project Number | `703759707702` | |
| Region | **`us-west1`** (Oregon) | 사용자 거주지 미국 서부 기준 |
| Artifact Registry repo | `web-apps` (Docker, us-west1) | 모든 웹 앱 이미지 공유 |
| Cloud Build 연결 | `github-mustang` (2nd gen, us-west1) | GitHub App 재사용 |
| Cloud Build 실행 SA | `cloudbuild-runner@mustang-web-apps.iam.gserviceaccount.com` | user-managed, 최소 롤 |
| Cloud Run 런타임 SA | Compute default (`<NUM>-compute@developer.gserviceaccount.com`) | 기본값 유지 |

### 활성화된 API

- `cloudbuild.googleapis.com`
- `run.googleapis.com`
- `artifactregistry.googleapis.com`
- `secretmanager.googleapis.com`
- `firebasehosting.googleapis.com` (Pattern A 준비용)

### IAM (cloudbuild-runner SA)

- `roles/artifactregistry.writer` — 이미지 push
- `roles/run.admin` — Cloud Run 서비스 배포/업데이트
- `roles/iam.serviceAccountUser` — 런타임 SA impersonation
- `roles/logging.logWriter` — 빌드 로그 기록

### 네이밍 규칙

- **repo명 = Cloud Run 서비스명 = Artifact Registry 이미지명** (kebab-case)
- Cloud Build 트리거명: `<서비스명>-main`
- 예: `hello-world-nextjs`, `hello-world-nextjs-main`

### 접근 제어 기본값

- Cloud Run 서비스: `--allow-unauthenticated` (공개)
- 도메인: `*.run.app` 기본 URL 사용, 필요 시 커스텀 도메인 매핑
- 사내 전용 앱은 IAP 별도 세팅 (표준 문서에 옵션으로 기록)

### 리소스 기본값 (Cloud Run)

- CPU: 1
- Memory: 512 MiB
- min-instances: 0 (콜드스타트 감수, 저렴)
- max-instances: 3

### 브랜치 전략

- `main` push → 자동 build → 자동 deploy
- PR preview 서비스: 표준에 포함 안 됨 (필요 시 프로젝트별 확장)

---

## 3-패턴 지원 계획

| 패턴 | 유즈케이스 | 배포 타겟 | 상태 |
|---|---|---|---|
| A. Static Frontend | Vite SPA, Next.js static export | Firebase Hosting | 🚧 미구현 |
| **B. SSR Frontend** | Next.js SSR (App Router) | Cloud Run | ✅ 2026-07-24 검증 완료 |
| C. API + Frontend | FastAPI/Express + Next.js | Cloud Run 서비스 2개 | 🚧 미구현 |

A/C는 첫 사용 프로젝트가 등장할 때 실증 + 템플릿화. 지금 없는 것을 미리 만들지 않는다.

---

## 템플릿 리포

`~/Projects/ai-teams/templates/web-deploy-template/`

- `nextjs-ssr/Dockerfile` — multi-stage node:20-alpine standalone
- `nextjs-ssr/cloudbuild.yaml` — `_SERVICE` 만 바꾸면 재사용 가능
- `nextjs-ssr/.dockerignore`
- `nextjs-ssr/src/app/api/health/route.ts`
- `nextjs-ssr/next.config.ts.snippet` — `output: "standalone"`
- `nextjs-ssr/SETUP.md` — 신규 프로젝트 부트스트랩 10-step

---

## 실증 결과 (hello-world-nextjs, 2026-07-24)

- GitHub repo: `iizs/hello-world-nextjs` (private)
- Cloud Build trigger: `hello-world-nextjs-main` (auto-fire on main push)
- Cloud Run URL: `https://hello-world-nextjs-ycqc4dhniq-uw.a.run.app`
- 검증 통과:
  - 로컬 `npm run build` ✅
  - 수동 트리거 실행 ✅
  - `git push` → 자동 트리거 → 배포 ✅
  - `GET /` → 200 (SSR 서버 시각 렌더링) ✅
  - `GET /api/health` → 200, `{"status":"ok"}` ✅
- 검증 후 리소스 정리 (인스턴스/AR/트리거) 예정

---

## 알려진 이슈 & 결정 기록

### Cloud Build 2nd-gen 서비스 계정 강제

**증상**: `gcloud builds triggers create` 를 `--service-account` 없이 실행하면 `INVALID_ARGUMENT` 에러.
**원인**: 2nd-gen 트리거는 user-managed SA 지정을 요구. Legacy `<NUM>@cloudbuild.gserviceaccount.com` SA는 명시적으로 지정할 수 없음.
**결정**: user-managed SA `cloudbuild-runner` 를 프로젝트 단위로 하나 만들고 모든 트리거가 공유. SA에 최소 롤만 부여.

### GitHub App 재사용

**관찰**: Cloud Build 2nd-gen connection 생성 시, 사용자 GitHub 계정에 이미 설치된 "Google Cloud Build" App을 자동 재사용. OAuth 재승인 불필요.
**의미**: 새 GCP 프로젝트를 추가해도 GitHub 쪽 세팅 반복 불필요.

### Firebase Hosting은 별도 파이프라인

Pattern A는 Cloud Build 대신 `firebase deploy` 를 CD로 쓰는 게 자연스러움. 별도 검증 필요.

---

## 다음 단계 (Roy에게 넘길 것)

1. 이 문서를 팀 표준으로 승인 (또는 수정 요청)
2. `web-deploy-template` 리포를 팀 저장소로 승격할지 결정 (지금은 `ai-teams/templates/` 내부에 존재)
3. Pattern A / C 실증은 첫 사용 프로젝트가 나올 때 착수
4. 사내용 IAP 앱이 등장하면 접근 제어 옵션 문서화

---

## 참조

- 실증 프로젝트: `~/Projects/hello-world-nextjs` (검증 후 삭제 예정)
- 템플릿: `~/Projects/ai-teams/templates/web-deploy-template/`
- Discord 대화: 2026-07-24, Breda ↔ Kirin
