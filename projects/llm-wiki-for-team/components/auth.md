# Auth

## 인증 방식

- 이메일 + 패스워드 (로그인)
- JWT 토큰 발급/검증
- 보호된 페이지: 비로그인 시 로그인 화면으로 리다이렉트

## 사용자 모델

- `id`, `email`, `hashed_password`, `role`, `is_active`
- `role`: `admin` 또는 `member`

## 역할별 권한

| 역할 | 권한 |
|------|------|
| Admin | 원본 투입/교체/삭제, 사용자 관리, 위키 조회·수정 요청 |
| Member | 위키 조회, 수정 요청 제출, 원본 조회, 로컬 sync |

## 초기 Admin 시딩

첫 Admin 계정은 CLI 스크립트로 생성한다.

```bash
cd backend
python scripts/seed_admin.py admin@example.com mypassword
```

이미 존재하는 계정이면 admin role로 승격.

## API

| Method | Path | 설명 | 권한 |
|--------|------|------|------|
| POST | `/api/auth/token` | 로그인, JWT 발급 | 공개 |
| GET | `/api/users/me` | 본인 프로필 | 인증 |
| GET | `/api/users/` | 사용자 목록 | Admin |
| POST | `/api/users/` | 사용자 생성 | Admin |
| PATCH | `/api/users/{id}/role` | 역할 변경 | Admin |
| DELETE | `/api/users/{id}` | 비활성화 | Admin |

## 미래 확장

- v0.2+ SSO (별도 ADR 예정)
- v0.2+ 권한 세분화 — Casbin 도입 가능성: [ADR-0007](../decisions/0007-casbin-future-rbac.md)

## 관련 SC

SC-1, SC-2, SC-3, SC-4, SC-5
