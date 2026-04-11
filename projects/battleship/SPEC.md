# Battleship — SPEC

## 요건 (Requirements)

**프로젝트 목적**
Flutter 앱 개발 가능성 검증:
- Flutter 기반 iOS/Android 동시 개발 가능 여부
- 로컬 파일 처리 (SQLite) 가능 여부

**게임 개요**
표준 Battleship 게임. 플레이어 vs COM (랜덤 AI) 1:1 대결.

**표준 룰**
- 10×10 그리드
- 함선 5종: Carrier(5칸), Battleship(4칸), Cruiser(3칸), Submarine(3칸), Destroyer(2칸)
- 턴제: 플레이어 → COM → 플레이어 → … (먼저 상대 함선 전부 격침 시 승리)

**초기 화면 메뉴**
- New Game: 기존 저장 게임이 있으면 삭제 후 신규 게임 시작
- Continue: 자동 저장된 게임이 있으면 이어하기 (없으면 비활성화)
- Stats: 전적 통계 조회
- Credits: 제작 정보

**전적 통계 (상세)**
- 전체 게임 수, 승/패, 승률(%)
- 총 발사 횟수, 명중 횟수, 명중률(%)
- 내가 격침한 적 함선 수 (누적)
- COM이 격침한 내 함선 수 (누적)

**자동 저장**
- 게임 진행 중 매 턴 후 자동 저장
- 앱이 백그라운드로 전환 시 즉시 저장
- Continue 선택 시 저장 상태 복원

**COM AI**
- 랜덤으로 미공격 셀 선택 (전략 불필요)

**에셋 방향**
- 미니멀 스타일 (심플한 선, 제한된 색상)
- 직접 생성 불가한 이미지는 `assets/prompts/` 하위에 생성 프롬프트 작성 → Kirin이 외부 도구로 생성

**플랫폼**
- iOS + Android (Flutter 공유 코드베이스)
- 로컬 전용 (서버 없음)

---

## 기술 설계 (Technical Design)

**디렉토리 구조**
```
battleship/
├── lib/
│   ├── main.dart
│   ├── models/
│   │   ├── game_state.dart       # 전체 게임 상태
│   │   ├── board.dart            # 10×10 그리드 상태
│   │   └── ship.dart             # 함선 정의 및 상태
│   ├── screens/
│   │   ├── home_screen.dart      # 메인 메뉴
│   │   ├── placement_screen.dart # 함선 배치
│   │   ├── battle_screen.dart    # 전투 화면
│   │   ├── game_over_screen.dart # 게임 종료
│   │   ├── stats_screen.dart     # 전적 통계
│   │   └── credits_screen.dart   # 크레딧
│   ├── services/
│   │   ├── database_service.dart # SQLite CRUD
│   │   ├── game_service.dart     # 게임 로직
│   │   └── com_ai_service.dart   # COM 랜덤 AI
│   └── widgets/
│       ├── grid_widget.dart      # 10×10 그리드 UI
│       └── ship_widget.dart      # 함선 UI
├── assets/
│   ├── images/                   # 생성된 이미지 에셋
│   └── prompts/                  # 이미지 생성 프롬프트 (Kirin 전달용)
│       ├── app_icon.md
│       ├── hit_marker.md
│       ├── miss_marker.md
│       └── ships.md
├── test/
│   └── widget_test.dart
└── pubspec.yaml
```

**기술 스택**
- Flutter (최신 stable)
- `sqflite`: SQLite 로컬 DB
- `path_provider`: 앱 로컬 디렉토리 접근
- `provider`: 상태 관리
- `flutter_test`: 단위 + 위젯 테스트

**화면 흐름**
```
Home
├── New Game → Placement → Battle → Game Over → Home
├── Continue → Battle (저장 상태 복원) → Game Over → Home
├── Stats → Home
└── Credits → Home
```

**DB 스키마 (SQLite)**
```sql
-- 전적 (단일 행 유지)
CREATE TABLE stats (
  id INTEGER PRIMARY KEY,
  games_played INTEGER DEFAULT 0,
  wins INTEGER DEFAULT 0,
  losses INTEGER DEFAULT 0,
  total_shots INTEGER DEFAULT 0,
  total_hits INTEGER DEFAULT 0,
  enemy_ships_sunk INTEGER DEFAULT 0,
  own_ships_sunk INTEGER DEFAULT 0
);

-- 저장 게임 (1개만 유지, 신규 게임 시 덮어쓰기)
CREATE TABLE saved_game (
  id INTEGER PRIMARY KEY,
  updated_at TEXT NOT NULL,
  player_ships TEXT NOT NULL,    -- JSON: 함선 위치 + 피격 여부
  com_ships TEXT NOT NULL,       -- JSON: COM 함선 위치 + 피격 여부
  player_shots TEXT NOT NULL,    -- JSON: 플레이어가 공격한 셀 (hit/miss)
  com_shots TEXT NOT NULL,       -- JSON: COM이 공격한 셀 (hit/miss)
  is_player_turn INTEGER NOT NULL -- 1=플레이어 턴, 0=COM 턴
);
```

**게임 JSON 구조 예시**
```json
{
  "ships": [
    {
      "type": "carrier",
      "cells": [[0,0],[0,1],[0,2],[0,3],[0,4]],
      "isVertical": false,
      "hitCells": []
    }
  ]
}
```

**함선 배치 (Placement Screen)**
- 배치 순서: Carrier → Battleship → Cruiser → Submarine → Destroyer (고정 순서)
- 현재 배치 중인 함선 및 크기를 화면 상단에 표시
- 가로/세로 방향 토글 버튼 제공
- 그리드 탭 → 선택 셀부터 함선 길이만큼 미리보기 표시 → 재탭(또는 확인 버튼)으로 배치 확정
- 무효 위치(겹침, 범위 초과) 시 미리보기를 빨간색으로 표시, 배치 불가
- **"이전 단계" 버튼**: 직전에 배치한 함선 취소 후 해당 단계로 복귀 (첫 번째 함선 배치 중에는 비활성화)
- 전체 배치 완료 후 "전투 시작" 버튼 활성화
- COM: 랜덤 자동 배치

**함선 렌더링**
- 이미지 에셋 미사용. Flutter `Container` + 색상/테두리로 구현
- 함선 셀: 어두운 회색 배경 + 함선 종류별 구분 색상 테두리
- 격침된 함선: 빨간 배경으로 전환

**전투 (Battle Screen)**
- 내 보드 (COM의 공격 결과 표시): 함선 위치 공개, 명중/빗나감 표시
- COM 보드 (플레이어 공격 결과 표시): 함선 숨김, 명중/빗나감만 표시
- 플레이어 공격 → COM 공격 순서 (순차 처리, 중간 피드백 표시)
- 함선 전부 격침 시 → Game Over Screen

**백그라운드 처리**
- `WidgetsBindingObserver`로 `AppLifecycleState.inactive` + `paused` 양쪽에서 저장 (iOS 강제 종료 대응)
- 앱 재시작 시 저장 게임 존재 여부 확인 → Home Screen에 Continue 활성화 여부 결정

**이미지 에셋 목록 및 사이즈**
| 에셋 | 사이즈 | 용도 |
|---|---|---|
| 앱 아이콘 | 1024×1024px | iOS/Android 앱 아이콘 |
| 명중 마커 | 256×256px | 공격 성공 셀 표시 |
| 빗나감 마커 | 256×256px | 공격 실패 셀 표시 |

(함선 본체는 Flutter 위젯으로 구현, 별도 이미지 불필요)

**테스트 구조**
```
test/
├── unit/
│   ├── game_service_test.dart      # 턴 처리, 격침 판정, 승패 결정
│   ├── com_ai_service_test.dart    # COM AI 랜덤 미공격 셀 선택
│   └── board_test.dart            # 배치 유효성 (겹침, 범위 초과, 이전 단계 취소)
└── widget/
    └── grid_widget_test.dart
```

---

## 성공 기준 (Success Criteria)

*Hawkeye가 작성 예정*

---

## 미결 이슈 (Open Issues)

*(없음)*

---

## 변경 이력 (Changelog)

- v0.1: 초기 스펙 — Flutter Battleship 앱, iOS/Android 동시, SQLite 로컬 저장
- v0.1.1: Roy↔Breda 구현 검토 반영 — 배치 UI 명확화(고정 순서+이전 단계 취소), 함선 Flutter 위젯 렌더링, iOS 백그라운드 저장 보완, 테스트 구조 추가
