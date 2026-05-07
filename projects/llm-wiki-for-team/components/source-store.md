# Source Store

원본 파일 저장 및 위키에서의 참조 설계.

> 저장 위치 결정: [ADR-0002 — Source Store git 비추적](../decisions/0002-source-store-not-tracked.md)
> MVP 파일 형식: [ADR-0003 — PDF/TXT/MD만 지원](../decisions/0003-mvp-file-formats.md)

## 저장

- **Docker 배포**: Docker Volume
- **설치형**: Local Path (예: `./source-store/`)
- 두 경우 모두 git 추적 대상 아님

## 업로드

- 허용 확장자: PDF, TXT, MD (화이트리스트 검증)
- Admin만 업로드 가능

## Source 참조 설계

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

## API

| Method | Path | 설명 | 권한 |
|--------|------|------|------|
| GET | `/api/sources/` | 원본 목록 | 인증 |
| POST | `/api/sources/` | 원본 업로드 | Admin |
| GET | `/api/sources/{filename}` | 원본 파일 서빙 | 인증 |

## 보류 (v0.2+)

- 원본 삭제 (soft-delete to `_trash/`)
- URL 소화 (크롤링)
- DOCX 지원

## 관련 SC

SC-6, SC-7, SC-8, SC-9 (v0.2+), SC-10, SC-14, SC-21, SC-22, SC-23
