# ADR-0001 — Wiki Store: Normal vs Bare Git Repo

- **상태**: Accepted
- **결정일**: 2026-05-06
- **결정자**: Roy (Lead & Architect), Breda (Developer)

## 컨텍스트

위키 저장소를 git repo로 둔다는 것까지는 합의됐는데, 형태(normal vs bare)가 미정이었음. Breda가 SPEC v0.1 검토 중 이슈로 제기.

## 옵션

| | Normal repo | Bare repo |
|--|------------|-----------|
| 구조 | `.git/` + working tree (실제 파일) | git 메타데이터만 |
| 파일 직접 접근 | O | X |
| git push/pull 수신 | 보통 안 함 | 서버 역할에 적합 |

## 결정

**Normal repo 채택.**

## 근거

- LLM Pipeline(backend)이 위키 파일을 직접 읽고 쓴다
  - Ingest: LLM이 생성한 MD를 파일로 저장 → git commit
  - Edit: 특정 페이지 파일을 read → LLM 수정 → 파일 저장 → git commit
- Bare repo였으면 `git show`, `git cat-file` 같은 명령으로 우회 필요 — 복잡하고 비효율적
- gitpython으로 `repo.index.commit()` 같은 단순 API 쓰려면 working tree 필요

## 트레이드오프

- git clone/pull로 외부에 노출하려면 별도 git 서버(또는 HTTP git endpoint) 필요 → nginx + `git http-backend` 컨테이너로 해결
