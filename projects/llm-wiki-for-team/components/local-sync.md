# Local Sync

위키를 로컬로 받아 에이전트가 직접 사용하는 경로.

## 동기화 방법

- **git**: `git clone <wiki-repo-url>` / `git pull`
  - nginx + `git http-backend` 컨테이너가 위키 repo를 외부에 노출
  - 익명 read 가능 (위키 자체가 공개 정보 가정)
- **zip**: UI에서 현재 위키 스냅샷 다운로드 (`_sheska.yaml` 포함)

## 받은 후 원본 파일 접근

`_sheska.yaml`의 `source_base_url`을 통해 원본 API에 접근 가능. [Source Store](source-store.md) 참조.

## 가이드 문서 (`/guide`)

- 에이전트 연동 방법
- 기본 프롬프트 예시
- `source_base_url` 설정 방법

## 관련 SC

SC-24 (git), SC-25 (zip), SC-26 (`_sheska.yaml`), SC-27 (가이드)
