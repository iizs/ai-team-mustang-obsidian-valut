# ADR-0002 — Source Store: git 비추적

- **상태**: Accepted
- **결정일**: 2026-05-06
- **결정자**: Kirin, Roy, Breda

## 컨텍스트

원본 파일(PDF, TXT, MD)을 어디에 저장할지. 초기 SPEC은 `_sources/` 디렉토리를 위키 git repo에 함께 두는 방향이었음.

## 옵션

- **(A)** `_sources/`를 git으로 추적 + Git LFS (PDF/DOCX 등 바이너리)
- **(B)** `_sources/`는 Docker 볼륨/Local Path로 분리, git은 MD 위키만 추적

## 결정

**(B) 분리 채택.**

## 근거

- PDF 등 바이너리를 git에 두면 repo 용량이 빠르게 비대화
- LFS 도입 시 추가 설정 부담 (self-hosted 사용자 환경 가정)
- 위키 자체는 텍스트만이라 git으로 가볍게 유지 가능
- 로컬 sync 받는 쪽도 위키만 받으면 충분 (원본은 `_sheska.yaml`의 `source_base_url`로 접근)

## 트레이드오프

- 로컬 환경에서 원본을 직접 보려면 서버 API 호출 필요 (Self-contained 단점)
- `_sheska.yaml` 메커니즘으로 보완
