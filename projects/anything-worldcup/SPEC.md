# Anything Worldcup — SPEC

## 요건 (Requirements)

**서비스 개요**
"Anything Worldcup"은 N개의 아이템을 토너먼트 형식으로 2개씩 비교해 최선호 1위를 결정하는 웹 서비스다. 어드민이 주제와 아이템을 등록하고, 일반 사용자는 주제를 선택해 토너먼트를 진행한다.

**사용자 유형**
- 일반 사용자: 주제 선택 → 토너먼트 진행 → 우승자 확인. 편집 불가.
- 어드민: 주제와 아이템 CRUD. 토너먼트 없음.

**사용자 페이지 (모바일 우선)**
- 아이콘 이미지와 함께 등록된 주제 목록 표시
- 주제 선택 → 해당 주제의 아이템으로 토너먼트 시작
- 아이템 수가 2^n이 아닌 경우, 가장 가까운 낮은 2^n개를 무작위 선택 (예: 10개 → 8개)
- 2개씩 비교 → 클릭으로 선택 → 우승자 결정까지 반복
- 최종 우승자 화면 표시 및 재시작 가능
- 데이터 편집 불가

**어드민 페이지**
- 로그인: id/pw (admin/admin)
- 주제 CRUD: 이름 + 아이콘 이미지 업로드
- 아이템 CRUD: 이름 + 이미지 업로드 + 랜덤 폴백 이모지
- 토너먼트 없음

**이미지 처리**
- 업로드된 이미지는 서버에서 리사이즈 후 파일로 저장
  - 아이템 이미지: 400×400px (모바일 카드 크기)
  - 주제 아이콘: 100×100px
- 이미지 미제공 시 랜덤 이모지 폴백 사용
- 이미지는 백엔드 API를 통해 사용자 페이지에 제공

**기술 요건**
- 프론트엔드: React (포트 8800)
- 백엔드: Python + venv (포트 8801)
- DB: SQLite, 향후 마이그레이션을 위한 추상화 적용
- 이미지 저장: 파일 시스템
- 로컬 실행 전용

**비기능 요건**
- 사용자 페이지: 모바일 우선 UI/UX
- 토너먼트 결과 미저장 (세션 한정)
- 데이터 편집은 어드민으로 제한

---

## 기술 설계 (Technical Design)

**디렉토리 구조**
```
anything-worldcup/
├── frontend/          # React (Vite)
│   ├── src/
│   │   ├── pages/user/     # 주제 목록, 토너먼트, 우승자
│   │   ├── pages/admin/    # 로그인, 주제, 아이템
│   │   ├── components/
│   │   └── api/            # API 클라이언트
│   └── package.json
└── backend/           # Python FastAPI
    ├── app/
    │   ├── main.py
    │   ├── models.py       # SQLAlchemy 모델
    │   ├── database.py     # DB 엔진 (SQLite → 교체 가능)
    │   ├── routers/
    │   │   ├── topics.py   # 공개 주제/아이템 엔드포인트
    │   │   └── admin.py    # 어드민 CRUD + 인증 엔드포인트
    │   └── utils/
    │       └── images.py   # Pillow 리사이즈 로직
    ├── uploads/            # 저장 이미지 (gitignored)
    ├── requirements.txt
    └── venv/               # gitignored
```

**API 계약**

공개:
- `GET /api/topics` — 주제 목록 (id, name, icon_url, emoji)
- `GET /api/topics/{id}/items` — 주제별 아이템 목록

어드민 (인증 토큰 필요):
- `POST /api/admin/login` → `{token}`
- `POST/PUT/DELETE /api/admin/topics/{id?}`
- `POST/PUT/DELETE /api/admin/topics/{id}/items / /api/admin/items/{id}`
- `POST /api/admin/topics/{id}/items/{iid}/image`

정적:
- `GET /uploads/{filename}` — 이미지 파일 제공

**DB 스키마 (SQLAlchemy)**
```
topics: id, name, icon_path, icon_emoji, created_at
items:  id, topic_id, name, image_path, emoji, created_at
```

**인증**
- `POST /api/admin/login` 시 JWT 토큰 발급
- 하드코딩 크리덴셜: admin/admin
- 어드민 라우트는 토큰 검증 미들웨어로 보호

**프론트엔드 라우트**
- `/` — 주제 목록 (사용자)
- `/tournament/:topicId` — 토너먼트 (사용자)
- `/admin` — 로그인
- `/admin/topics` — 주제 관리
- `/admin/topics/:id/items` — 아이템 관리

**이미지 처리**
- 정사각형으로 크롭 (중앙 기준) 후 Pillow로 리사이즈
- 아이템: 400×400px JPEG/WebP
- 아이콘: 100×100px JPEG/WebP

---

## 성공 기준 (Success Criteria)

### SC-1: 사용자 페이지 — 주제 목록
- Given 백엔드가 실행 중이고 DB에 주제가 존재할 때,
  When 사용자가 `/`에 접속하면,
  Then 각 주제의 아이콘 이미지(또는 폴백 이모지)와 함께 주제 목록이 표시된다.

### SC-2: 2^n 강제 적용으로 토너먼트 시작
- Given 아이템이 정확히 2^n개(예: 8개)인 주제가 있을 때,
  When 사용자가 해당 주제를 선택하면,
  Then 8개 아이템 전체로 토너먼트가 시작된다.

- Given 아이템이 2^n개가 아닌(예: 10개) 주제가 있을 때,
  When 사용자가 해당 주제를 선택하면,
  Then 8개를 무작위 선택하여 토너먼트가 시작된다.

### SC-3: 토너먼트 진행
- Given 토너먼트가 진행 중일 때,
  When 매치 화면이 표시되면,
  Then 정확히 2개의 아이템이 이미지(또는 이모지 폴백)와 함께 나란히 표시된다.

- Given 매치가 표시되었을 때,
  When 사용자가 아이템 하나를 클릭하면,
  Then 선택된 아이템이 다음 라운드로 진출하고 다음 매치가 표시된다.

- Given 2개 아이템만 남았을 때(결승),
  When 사용자가 하나를 선택하면,
  Then 우승 아이템을 보여주는 우승자 화면이 표시된다.

- Given 우승자 화면이 표시되었을 때,
  When 사용자가 재시작을 선택하면,
  Then 같은 주제로 새 토너먼트가 시작된다(재무작위화).

### SC-4: 사용자 페이지 읽기 전용
- Given 사용자 페이지에서,
  When 모든 사용자 인터랙션을 테스트하면,
  Then 주제나 아이템의 생성, 수정, 삭제가 불가능하다.

### SC-5: 어드민 인증
- Given `/admin`의 어드민 로그인 페이지에서,
  When admin/admin 크리덴셜을 제출하면,
  Then JWT 토큰이 발급되고 `/admin/topics`로 리디렉션된다.

- Given `/api/admin/*` 엔드포인트에 인증되지 않은 요청이 올 때,
  When 유효한 토큰 없이 요청하면,
  Then 401 Unauthorized 응답이 반환된다.

### SC-6: 어드민 주제 CRUD
- Given 로그인한 어드민이,
  When 이름 + 아이콘 이미지로 새 주제를 생성하면,
  Then 어드민 주제 목록과 사용자 페이지(`GET /api/topics`)에 주제가 나타난다.

- Given 기존 주제가 있을 때,
  When 어드민이 이름이나 아이콘을 수정하면,
  Then 어드민과 사용자 뷰 모두에 변경 사항이 반영된다.

- Given 아이템이 있는 기존 주제가 있을 때,
  When 어드민이 주제를 삭제하면,
  Then 주제와 모든 아이템이 삭제되고 사용자 페이지에서 더 이상 표시되지 않는다.

### SC-7: 어드민 아이템 CRUD
- Given 어드민에서 주제가 선택되었을 때,
  When 이름과 이미지로 아이템을 추가하면,
  Then 해당 주제의 아이템 목록에 아이템이 나타난다.

- Given 이미지가 업로드되지 않은 아이템이 있을 때,
  When 사용자 페이지 토너먼트에서 표시되면,
  Then 랜덤 이모지가 폴백으로 표시된다.

- Given 기존 아이템이 있을 때,
  When 어드민이 삭제하면,
  Then 주제의 아이템 목록과 토너먼트에서 더 이상 나타나지 않는다.

### SC-8: 이미지 처리
- Given 어드민이 아이템 이미지를 업로드할 때,
  When 업로드가 처리되면,
  Then 저장된 파일은 정확히 400×400px (중앙 크롭 및 리사이즈)이다.

- Given 어드민이 주제 아이콘 이미지를 업로드할 때,
  When 업로드가 처리되면,
  Then 저장된 파일은 정확히 100×100px (중앙 크롭 및 리사이즈)이다.

- Given 저장된 이미지가 있는 아이템이 있을 때,
  When 사용자 페이지가 요청하면,
  Then `GET /uploads/{filename}`을 통해 이미지가 정상 제공된다.

### SC-9: 모바일 우선 레이아웃
- Given 390px 너비 뷰포트(iPhone 크기)에서 사용자 페이지가 로드될 때,
  When 주제 목록과 토너먼트 화면이 렌더링되면,
  Then 가로 스크롤이나 오버플로우 없이 레이아웃이 사용 가능하다.

### SC-10: 토너먼트 결과 미저장
- Given 사용자가 토너먼트를 완료했을 때,
  When 세션이 종료되거나 사용자가 다른 페이지로 이동하면,
  Then DB에 토너먼트 결과가 저장되지 않는다.

### SC-11: 어두운 배경 위 텍스트 가독성
- Given 사용자 페이지(주제 목록, 토너먼트 매치 카드, 우승자 화면)에서,
  When 어두운 배경 위에 텍스트가 렌더링되면,
  Then 텍스트 색상이 충분한 대비를 제공해 명확히 읽힌다 (WCAG AA 기준: 일반 텍스트 4.5:1 이상).

### SC-12: 아이템 부족 주제 숨김
- Given DB에 아이템이 0개 또는 1개인 주제가 있을 때,
  When 사용자 페이지가 주제 목록을 렌더링하면,
  Then 해당 주제는 목록에 표시되지 않는다.

- Given 아이템이 2개 이상인 주제가 있을 때,
  When 사용자 페이지가 주제 목록을 렌더링하면,
  Then 해당 주제는 표시되고 선택 가능하다.

---

## 미결 이슈 (Open Issues)

*(없음)*

---

## 변경 이력 (Changelog)

- v0.1: 초기 스펙 — React+Python+SQLite 아키텍처, 사용자/어드민 분리
- v0.2: UI 텍스트 가독성 수정, 아이템 2개 미만 주제 숨김
