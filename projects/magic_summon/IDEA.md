
마법진을 그려서 소환수를 생성하는 앱


터치 등의 인터페이스를 제공해서 마법진을 그릴 수 있게 제공. -> 결국 (0,1) 의 n x n 크기의 데이터 배열 -> 암호화, 난수화 하면 속성을 추출할 수 있을 것

예전 md5 배틀 같은 개념

이 값에 따라 모양(이미지), 속성, 능력치 부여

여기에 소환 시간, 장소, 주문에 따라서도 소환 결과에 영향을 미칠 수 있도록 하면 좋겠다. 
시간은 1시간 또는 2시간 단위
장소는 GPS 기반 혹은 지형 고려?

---

## 브레인스토밍 메모 (2026-04-17)

### 핵심 재미 요소
- **의외성**: 뭐가 나올지 모르는 두근두근함 (md5 배틀의 DNA)
- 소환 행위 자체에 의미가 있어야 함

### "소환 후 뭐할건데" 문제 검토

| 방향 | 핵심 | 판단 |
|------|------|------|
| 공유형 | 소환하고 SNS 공유가 목적 | 일러스트 부담 크고, 루프 약함 |
| **자동 배틀형** | 능력치로 결판, 비동기 대전 | ✅ 유력 방향 |
| 비대칭 배틀형 | 공격측에 선택지 부여 | 밸런싱 골치, 보류 |

### 비동기 배틀 구조
- CoC, 포켓몬Go 체육관 방식 참고
- 사용자가 소환수를 등록해두면, 다른 유저가 언제든 도전 가능
- 사용자 풀이 적어도 각자의 소환수가 쌓이며 콘텐츠 형성
- 배틀 결과는 능력치 자동 계산 (비동기 특성상 수동 개입 불가)

### 소환수 이미지 방향
- AI 이미지 생성으로 "나만의 유일한 소환수" 구현
- on-the-fly 생성은 시간 문제 → **큐잉 방식**으로 해결
  - 소환 의뢰 접수 → 몇 분 후 푸시 알림 "소환수가 강림했습니다"
  - 기다림 자체가 소환 의식의 일부가 될 수 있음

### AI 이미지 프롬프트 생성 방식 (2026-04-18 추가)
- 마법진 → 해시 → 능력치 분배와 동일한 로직으로 이미지 프롬프트 키워드 선택
- 해시값을 구간별로 쪼개어 각 집합에서 키워드 선택:
  - `{색상 집합}` × `{종족 집합}` × `{크기 집합}` × `{분위기 집합}` ...
  - 예: `"a colossal crimson dragon, ethereal, fantasy creature..."`
- 같은 키워드 조합이어도 LLM 특성상 매번 다른 이미지 생성 → "소환할 때마다 다른 모습" 연출 가능
- **희귀도 구현**: 종족 집합에 가중치 부여로 자연스러운 등급 형성
  - 예: `{slime:3, goblin:2, phoenix:1}` → 확률적 희귀도
- **컨텍스트 변수 연동** (원래 스펙의 시간/장소 아이디어와 연결):
  - 기본 속성은 마법진 해시로 결정
  - 시간/장소가 키워드 가중치에 보정값 부여 (예: 자정 → undead/shadow 계열 상승, 해안 GPS → aqua/sea 계열 상승)

### 이미지 생성 키워드 그룹 (2026-04-19)

프롬프트 구조: `{외형} {색상} {종족}, {속성} element, {이펙트}, {배경}`

#### 1. 외형 (form) — 크기+체형 통합
tiny, small, petite, compact, sleek, swift, lithe, lean, wiry,
medium, sturdy, stocky, broad, muscular,
large, massive, towering, hulking, armored, plated,
colossal, enormous, gargantuan, titanic,
wispy, ghostly, formless, amorphous, ethereal

#### 2. 색상 (color)
crimson, scarlet, ruby, vermillion,
amber, golden, saffron, ochre,
emerald, jade, olive, chartreuse,
azure, cobalt, sapphire, cerulean, teal,
violet, indigo, lavender, magenta,
silver, platinum, pearl, ivory,
obsidian, ebony, charcoal, onyx,
pale, ashen, ghostly white,
bronze, copper, rust,
iridescent, prismatic, opalescent

#### 3. 종족 (species) — 희귀도 가중치 적용 예정
**일반 (common)**
slime, rat, frog, beetle, moth, bat, crow, worm, crab, jellyfish

**비일반 (uncommon)**
wolf, fox, bear, eagle, shark, octopus, snake, lizard, cat, deer

**희귀 (rare)**
golem, wraith, fairy, sprite, imp, harpy, mermaid, centaur, minotaur, kappa

**매우 희귀 (epic)**
dragon, phoenix, griffin, leviathan, behemoth, basilisk, chimera, hydra, manticore

**전설 (legendary)**
celestial, void walker, elder god fragment, ancient titan, primordial serpent

#### 4. 속성 (element)
fire, water, earth, wind, lightning, ice,
light, shadow, void, arcane, poison, nature,
metal, crystal, gravity, time, sound, dream

#### 5. 배경 (background) — 희귀할수록 배경 등장
**(없음)** — plain / transparent (일반 등급)
misty meadow, sunlit garden, forest clearing,
dark forest, ancient ruins, mountain peak, desert dunes,
volcanic crater, frozen tundra, deep ocean abyss, stormy cliffside,
moonlit temple, celestial realm, void rift, crystal cavern,
twilight sky, aurora borealis, cosmic nebula

#### 6. 이펙트 (effect) — 희귀할수록 이펙트 추가
**(없음)** — (일반 등급)
faint glow, soft sparkle,
glowing aura, lightning sparks, swirling embers, floating ice crystals,
radiant halo, shadow tendrils, ethereal shimmer, star particles,
cosmic energy burst, divine light pillar, void distortion, time fracture effect

---

### 미결 사항
- 현실적인 비용 문제 (AI 이미지 생성 API 비용)
- WORKFLOW 반영 시기

