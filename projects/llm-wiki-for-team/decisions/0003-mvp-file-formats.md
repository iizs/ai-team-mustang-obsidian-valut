# ADR-0003 — MVP 지원 파일 형식

- **상태**: Accepted
- **결정일**: 2026-05-06
- **결정자**: Kirin, Roy, Breda

## 컨텍스트

원본 파일 업로드의 허용 형식 범위. 초기 SPEC은 PDF/DOCX/TXT/MD + URL 크롤링까지 포함했음.

## 결정

MVP(v0.1)는 **PDF, TXT, MD만 지원.** DOCX, URL은 v0.2+.

## 근거

- DOCX 파싱은 `python-docx` 의존성과 추가 테스트 부담
- URL 크롤링은 robots.txt, 인증, 동적 콘텐츠 등 별도 영역
- MVP 핵심 가치(원본 → 위키)는 PDF/TXT/MD만으로도 검증 가능
- 업로드 확장자 화이트리스트로 명시적 제한
