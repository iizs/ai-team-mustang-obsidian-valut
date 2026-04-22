# server-setup

맥미니 홈서버 셋업 관련 아이디어 및 설정 메모.

---

## Cloudflare Tunnel — 외부 웹훅 수신 방안 (2026-04-21)

### 배경
- 홈 맥미니(가정용 공유기 환경)에서 GitHub 등 외부 서비스의 웹훅을 받아야 함
- 포트 포워딩은 보안 우려 (인바운드 포트 노출)

### 옵션 비교

| 방식 | 장점 | 단점 |
|------|------|------|
| 포트 포워딩 | 단순 | 인바운드 포트 노출, 보안 취약 |
| **Cloudflare Tunnel** ✅ | 포트 개방 불필요, 보안 우수, 무료 | cloudflared 데몬 상시 실행 필요 |
| ngrok | 설정 간단 | 무료 플랜은 URL 매번 변경됨 |
| 클라우드 VPS 중계 | 유연함 | 비용 발생 |

### Cloudflare Tunnel 동작 원리

```
GitHub → Cloudflare (공인 URL) → 터널 → Mac mini 로컬 서버
```

- Mac mini에 `cloudflared` 데몬이 Cloudflare 서버에 **아웃바운드 연결 유지**
- 인바운드 포트 개방 없음 → 공유기 설정 불필요
- 고정 URL → GitHub 웹훅 설정 한 번만
- launchd로 재시작 시 자동 연결 가능

### 설정 흐름

1. Cloudflare 계정 + 도메인 연결 (도메인 이미 있음)
2. `brew install cloudflare/cloudflare/cloudflared`
3. `cloudflared tunnel create <이름>` — 터널 생성
4. 로컬 포트를 `webhook.yourdomain.com`에 연결
5. GitHub 웹훅 URL로 `https://webhook.yourdomain.com` 등록
6. GitHub webhook secret 검증 추가 → 요청 위변조 방지

### 다중 웹훅 수신

**방법 1 (추천): 경로별 라우팅 — 단일 터널**

로컬에 웹서버 (FastAPI, Express 등) 하나 띄우고 경로별 핸들러 분기:
```
https://webhook.yourdomain.com/github  → GitHub
https://webhook.yourdomain.com/notion  → Notion
https://webhook.yourdomain.com/slack   → Slack
```
터널 1개 + 로컬 포트 1개로 관리 단순.

**방법 2: 서브도메인별 터널**

서비스마다 다른 서브도메인 + 다른 로컬 포트:
```
github.yourdomain.com  → localhost:8081
notion.yourdomain.com  → localhost:8082
```
cloudflared ingress 규칙으로 멀티 라우팅 지원.

### 미결 사항
- 어떤 서비스들의 웹훅을 받을지 목록 확정
- 로컬 웹서버 언어/프레임워크 선택 (FastAPI, Express 등)

---

## launchd & 자동 로그인 — 부팅 자동화 (2026-04-21)

### launchd 종류

| 종류               | 위치                        | 실행 조건              | 권한                             |
| ---------------- | ------------------------- | ------------------ | ------------------------------ |
| **LaunchDaemon** | `/Library/LaunchDaemons/` | 시스템 부팅 시 (로그인 불필요) | root (UserName 키로 특정 계정 지정 가능) |
| **LaunchAgent**  | `~/Library/LaunchAgents/` | 해당 계정 GUI 로그인 후    | 해당 계정                          |

- LaunchDaemon에서 특정 계정 권한으로 실행:
  ```xml
  <key>UserName</key>
  <string>kirinchoi</string>
  ```

### 서비스별 적용

- **cloudflared** → LaunchDaemon (로그인 없이 터널 유지)
- **팀 시작 스크립트 (tmux)** → LaunchAgent (로그인 후 에이전트 자동 기동)
  - tmux는 사용자 GUI 세션 환경 필요 → LaunchDaemon에서는 동작 안 함

### 자동 로그인 (Auto Login)

- **시스템 설정 → 일반 → 사용자 및 그룹**에서 설정
- 부팅 후 자동으로 GUI 세션 시작 → LaunchAgent 트리거
- SSH 로그인으로는 LaunchAgent 트리거 안 됨 (GUI 콘솔 로그인 기준)

### 완전 자동화 흐름

```
맥미니 부팅
  → LaunchDaemon: cloudflared 터널 자동 시작 (로그인 불필요)
  → Auto Login: GUI 세션 자동 시작
  → LaunchAgent: 팀 시작 스크립트 (tmux) 자동 실행
```

사람 손 없이 부팅 → 웹훅 터널 + 에이전트 팀 전체 자동 기동 가능.
