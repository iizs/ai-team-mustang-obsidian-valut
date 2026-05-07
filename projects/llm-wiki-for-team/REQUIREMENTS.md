# Requirements

> 서비스명: **Sheska** — llm-wiki for team
> 원칙 문서: https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f
> 방향성이 모호한 경우 이 문서를 따른다.

## 개요

Karpathy의 llm-wiki 컨셉을 팀 협업 도구로 확장한 오픈소스 솔루션.
사람이 원본 자료를 투입하면 LLM이 위키를 생성·관리하며, 팀 에이전트가 로컬에서 연결해 사용할 수 있다.

## 사용자 요건

- 원본 자료(파일) 투입 UI 제공 (PDF, TXT, MD; 업로드 확장자 제한)
- LLM이 원본을 소화해 Obsidian 호환 Markdown 위키 자동 생성·업데이트
- 사용자가 위키 수정을 LLM에게 요청하는 방식 (직접 편집 불가)
- 위키 조회 UI
- 원본 파일 조회 UI (Web UI 내 별도 메뉴)
- 로컬 연동 방법 가이드 + 기본 프롬프트 제공 (docs)
- 로컬 sync: git clone/pull, zip 다운로드 지원

## 비기능 요건

- 오픈소스
- Self-hosted (직접 설치 + Docker)

## 사용자 역할 (Role)

| 역할 | 권한 |
|------|------|
| Admin | 원본 투입/교체/삭제, 사용자 관리 |
| Member | 위키 조회, 수정 요청 제출, 원본 조회, 로컬 sync |

> 권한 세분화의 미래: [decisions/0007-casbin-future-rbac.md](decisions/0007-casbin-future-rbac.md)
