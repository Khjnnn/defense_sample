# 로그라이크 타워디펜스 — 프로토타입 기획서

- **문서 버전**: v1.0
- **작성일**: 2026-09-01
- **목표**: 재미 검증용 플레이 가능한 프로토타입 (iOS/Android)
- **상태**: 설계 확정, 구현 착수 전

---

## 0. 배경 & 이 문서의 목적

예전에 같은 장르로 기획했던 프로젝트가 있었으나 폐기했다. 폐기 원인(사용자 확인):

1. **컨셉·세계관 매력 부족** — 아서왕/원피스 캐릭터 짬뽕 네이밍, 세계관 얕음.
2. **시스템 비대** — 캐릭터 111종 + 퓨전 102레시피 + 슬롯 + 메타 + 계정 + 에너지. MVP 불가 규모.
3. **코어 재미 미검증** — 랜덤 소환 + 머지 루프 자체의 재미가 확인 안 됨.
4. **기술 난맥** — React Native → 웹 프로토타입 → Capacitor 로 오락가락, 아키텍처 붕괴.

**근본 원인**: 다른 게임의 유닛 리스트를 베껴와 컨셉만 교체했는데, 원본 게임의 맵 크기·진행 구조·조합 의도가 새로 만들 게임과 맞지 않아 이어붙이기가 불가능했고 구현이 왜곡됐다.

**이번 원칙**:
- 게임 규칙(맵·진행·전투)을 먼저 확정하고, 유닛·조합은 거기에 맞춰 설계한다. 유닛 리스트는 "숫자 채우기 참고용"이지 뼈대가 아니다.
- 스코프를 재미 검증에 필요한 최소로 고정한다. 메타/수익화/대량 콘텐츠는 전부 프로토타입 이후.
- 엔진·언어를 하나로 확정하고 바꾸지 않는다.

예전 프로젝트의 참고 자산: `C:\dev_auto_plan\claude-harness-main` — "아르카디아 대륙" 세계관, 캐릭터 111종 데이터시트(`docs/plans/09-character-stats.md`), 퓨전 레시피(`configs/recipes.json`). 이번 프로토타입에서는 **참고만** 하고 직접 이식하지 않는다.

---

## 1. 확정 사항

| 항목 | 결정 |
|---|---|
| 장르 | 경로 방어형 타워디펜스 + 로그라이크 |
| 런 구조 | 단일 맵 연속 방어 (웨이브 1→∞, 웨이브 사이 랜덤 상점, 죽으면 종료, 한 판 15~30분) |
| 톤 | 다크 판타지 / 로그라이크 정서 (침식되는 세계, 유물, 저주받은 방어선) |
| 핵심 재미 축 | 매 런 다른 타워 조합·시너지 빌딩 |
| 확장 방향 | (이후) 메타 수집·성장 → (더 이후) PvP. **프로토타입 범위 아님.** |
| 엔진 | Godot 4.x |
| 언어 | GDScript (데이터 주도 설계) |
| 리소스 | 1인 개발, 아트 = AI 생성, 게임개발 첫 경험, 웹/TS 배경 |
| 협업 | Claude + codex 2인 1조. Claude = 아키텍처·스키마·상태머신·통합·리뷰. codex = 계약 기반 알고리즘 덩어리 위임. |

### 언어 선택 근거 (GDScript)

- Godot TD 튜토리얼·문서의 대다수가 GDScript → 막혔을 때 검색이 즉시 됨. 첫 게임개발의 마찰 최소화.
- C# 대안은 정적 타입이 TS와 가깝지만 예제를 매번 번역해야 하고 모바일 익스포트 설정이 무겁다.
- 웹(Phaser/Pixi + TS) 대안은 예전에 아키텍처가 붕괴된 바로 그 경로라 배제.
- 데이터 주도로 설계하면 언어의 느슨한 타입 문제를 완화하고, 이후 유닛 100종을 넣어도 코드는 그대로다.

---

## 2. 프로토타입 스코프

### 포함 (IN)

- 타일맵 맵 1개 + 고정 적 경로 1개
- 적 4종 + 보스, 무한 웨이브 스폰 + HP/속도 스케일링
- 타워 8종 (역할 분리), 그리드 스냅 배치, 골드 구매, 3레벨 업그레이드, 판매
- 웨이브 사이 랜덤 상점 (3슬롯, 리롤), 타워 해금 게이팅
- 유물 12종 (런 다양성 엔진), 5웨이브마다 3중 택1
- 기지 체력, 게임오버, 결과 화면(도달 웨이브/처치/생존 시간)
- 배속 x1/x2, 일시정지, 재시작
- 조기 웨이브 시작 버튼 (탐욕 결정 장치)
- 최소 UI + AI 아트 플레이스홀더

### 제외 (OUT — 프로토타입 이후)

- 계정/로그인/서버, 에너지 시스템, 메타 progression, 타워 카드
- 광고 / IAP
- 캐릭터 대량 콘텐츠(111종), 퓨전 레시피
- 여러 맵 / 챕터 / 스토리 연출
- PvP
- 사운드 폴리싱, 세이브/로드

---

## 3. 코어 루프 & 상태 머신

### 게임 상태

```
BOOT → MAIN_MENU → PLAYING → GAME_OVER → RESULTS → MAIN_MENU
```

### PLAYING 내부 페이즈

```
        ┌──────────────────────────────────────────────┐
        ▼                                              │
   BUILD_PHASE ──[웨이브 시작 버튼]──► WAVE_ACTIVE ──────┘
   - 적 없음 (또는 이전 낙오병)          - 스포너가 이번 웨이브 적 방출
   - 상점 3슬롯 오픈 (리롤 = 골드)       - 적 경로 이동, 타워 발사
   - 타워 구매/업글/판매                 - 처치 시 골드, 기지 도달 시 피해
   - 유물 선택 (해당 웨이브)             - 스폰 종료 + 잔적 0 → WAVE_CLEARED
```

`WAVE_CLEARED` → 웨이브 보상 골드 → 매 5웨이브 클리어 시 유물 3중 택1 → `BUILD_PHASE`, 웨이브 인덱스 +1.

### 무한 진행

- 웨이브 끝 없음. 적 HP·수량·속도는 공식 스케일링(§5).
- 10웨이브마다 보스 웨이브.
- 기지 체력 0 → `GAME_OVER`. 결과 화면에 도달 웨이브 / 총 처치 / 생존 시간 표시 = 재미 검증 지표.

### 조기 시작 버튼 (탐욕 결정 장치, 프로토타입 포함)

`WAVE_ACTIVE` 중에도 "다음 웨이브 조기 시작" 버튼 활성. 누르면 다음 웨이브가 현재 웨이브에 겹쳐 스폰되고, 대가로 보너스 골드 지급 (`남은 스폰 예정 수 × 2G`). 위험을 감수하고 경제를 가속하는 로그라이크식 선택.

### 아키텍처 (Godot 노드 + 시그널)

- **`Game`** (루트 오케스트레이터) — 상태 머신 소유, 페이즈 전환 제어.
- 하위 서비스 노드: `MapService`, `WaveDirector`, `EconomyService`, `ShopService`, `RelicService`, `TowerContainer`, `ProjectileContainer`.
- **`EventBus`** (autoload 싱글턴) — 전역 시그널 허브. 서비스 간 직접 참조 대신 시그널로 소통.
  - 시그널: `enemy_killed(enemy, bounty)`, `base_damaged(amount)`, `base_hp_changed(value)`, `gold_changed(value)`, `wave_started(index)`, `wave_cleared(index)`, `tower_placed(tower)`, `tower_sold(tower)`, `relic_acquired(relic)`.
- **`RunState`** (autoload) — 현재 런의 휘발 데이터: 골드, 기지 HP, 웨이브 인덱스, 보유 유물, 배치 타워 배열, 해금된 타워 id 집합, `run_seed`. `GAME_OVER` 시 초기화.
- 시뮬레이션은 `_physics_process(delta)` 기준 (60fps 고정). 렌더링 분리.
- 일시정지 = `get_tree().paused = true`. 배속 = `Engine.time_scale = 1.0 / 2.0` (전체 균일 적용).

---

## 4. 맵 / 경로 / 그리드

### 방향 & 크기

- **가로(landscape) 고정.** 경로 방어 장르 표준, 경로 가독성 우위. (예전 기획의 세로 9×12 고민 폐기.)
- 셀 64px, 플레이필드 약 20×11셀 (1280×704). 카메라 고정 — 스크롤/줌 없음 (줌은 이후 스트레치).

### 경로 = Godot `Path2D` + `PathFollow2D`

- 맵 씬(`map_forsaken_vale.tscn`) 에디터에서 TileMap 위에 `Path2D` 곡선을 직접 그린다.
- 적은 `PathFollow2D`를 루트로 하는 씬. 매 프레임 `progress += current_speed * delta`. `progress_ratio >= 1.0` 이면 기지 도달.
- 이유: Godot TD 튜토리얼 대다수가 이 방식 → 검색·디버그 즉시. 웨이포인트 좌표 배열 수동 관리 불필요.

### 맵 씬 구성물

| 노드 | 역할 |
|---|---|
| `TileMapLayer` | 바닥 + 경로 장식 타일 |
| `Path2D` | 적 이동 곡선 (에디터 작성). 적(PathFollow2D)의 부모가 됨. |
| `SpawnPoint` (`Marker2D`) | 스폰 위치 (경로 시작과 동일, 명시용) |
| `BaseZone` (`Area2D`) | 경로 끝. 적 진입 감지 → 기지 피해 |
| `MapConfig.tres` | `grid_size`, `cell_px`, `blocked_cells` (장식 차단 셀만 수동 지정) |

### 배치 가능 판정 — `MapService`가 단일 권한

- 그리드 좌표 = `floor(world_pos / cell_px)`.
- 로드 시 `Path2D` 곡선을 그리드 셀로 래스터화 → `path_cells` 집합 자동 생성 → 배치 불가에 자동 포함. 디자이너는 장식 차단 셀만 수동 지정.
- 배치 가능 조건: `not in path_cells` ∧ `not in blocked_cells` ∧ `not in occupied_cells`(런타임) ∧ 그리드 범위 내.
- 타워 풋프린트 1×1 (프로토타입).
- 배치 프리뷰: 그리드 스냅된 고스트 스프라이트 + 유효/무효 녹/적 틴트 + 사거리 원.

---

## 5. 적 / 웨이브

### 적 엔티티 (`enemy.tscn`, 루트 = `PathFollow2D`)

- 자식: `Sprite2D`, `HealthComponent`, `Area2D`(피격 판정 hurtbox), `DebuffComponent`(감속/도트).
- 이동: `current_speed = base_speed * slow_mult`. 감속 중첩은 최강값으로 갱신, 하한 클램프(예: 0.2).
- 기지 도달: `EventBus.base_damaged.emit(leak_damage)` → `queue_free()`.
- 사망: HP ≤ 0 → `EventBus.enemy_killed.emit(self, bounty)` → 사망 연출 → `queue_free()`.
- 스탯: `max_hp`, `base_speed`(px/s), `armor`(고정 피해 감산, 최소 1 보장), `bounty`, `leak_damage`, `slow_immune_threshold`.

### 적 4종 + 보스

| id | 역할 | HP | 속도 | armor | bounty | leak | 특이 |
|---|---|---|---|---|---|---|---|
| `shambler` | 기본 | 100 | 60 | 0 | 6 | 1 | 밸런스 기준점 |
| `hound` | 속공 | 55 | 130 | 0 | 8 | 1 | 감속 타워 가치 부여 |
| `swarmling` | 물량 | 30 | 85 | 0 | 3 | 1 | 큰 무리로 스폰, 범위 타워 가치 부여 |
| `brute` | 탱커 | 480 | 38 | 6 | 20 | 3 | armor로 저단일딜 무력화, 관통/도트/고뎀 요구 |
| `boss` | 10웨이브마다 | 전용 곡선 | 30 | 10 | 120 | 10 | 감속 50% 이상 면역 (`slow_immune_threshold = 0.5`) |

### 웨이브 = 절차적 생성 + 시드 고정

무한이라 손으로 못 짜므로 `WaveDirector`가 공식으로 웨이브 N을 조립한다.

- **예산제**: `budget(N) = 40 + 14*N + 0.5*N*N`. 예산으로 적 "카드"를 구매(타입별 `point_cost`), 해금 스케줄 준수:
  - W1+ `shambler` / W3+ `hound` / W5+ `swarmling` / W8+ `brute` / 매 10웨이브 `boss`(예산 ~50% 소비, 나머지는 잡몹).
- **구성 가중치**가 N에 따라 이동: 초반 `shambler` 위주 → 후반 혼합.
- **HP 스케일**: `hp_mult(N) = 1 + 0.08*(N-1) + 0.004*(N-1)^2` (boss 제외).
- **보스 HP 곡선**: 별도. `boss_hp(N) = 2500 * (1 + 0.15 * (N/10 - 1))` (10웨이브째 = 2500 기준, 이후 보스마다 +15%p).
- **속도 스케일**: 웨이브당 +0.5%, +40% 상한.
- **스폰 간격**: `0.9s → 0.45s` 하한으로 N 따라 축소. 타입별 버스트로 방출.
- **결정성**: `WaveDirector`를 `run_seed + N` 으로 시드 → 같은 시드는 같은 웨이브 재현 (밸런싱 디버그 필수).
- **웨이브 보상**: `reward(N) = 25 + 5*N` 골드.

### 데이터 주도

- `EnemyType.tres` — 타입별 스탯 + 씬 참조 + 스프라이트.
- `WaveTuning.tres` — 예산/HP/속도/보스 공식 계수, 해금 테이블, 스폰 케이던스 전부 `@export`. 밸런싱 = 리소스 편집, 코드 무수정.

---

## 6. 타워 / 투사체 / 타게팅

### 타워 엔티티 (`tower.tscn`, 루트 `Node2D`)

- 자식: `Sprite2D`, `RangeArea`(`Area2D` + `CircleShape2D`, 반지름 = 사거리), `AttackTimer`(`Timer`), `TurretPivot`(타겟 방향 회전, 선택).
- 타게팅: `RangeArea`가 `area_entered`/`area_exited`로 `in_range` 배열 관리. `AttackTimer.timeout` 시 정책으로 타겟 선정 후 발사.
- 타겟 정책: `FIRST`(경로 진행도 최대) / `CLOSEST` / `STRONGEST`(최대 HP). 타워 패널에서 순환 토글. 기본 `FIRST`.

### 공격 종류 (`attack_kind` enum)

| kind | 동작 |
|---|---|
| `single` | 투사체 1발, 단일 타겟 |
| `splash` | 착탄 폭발, `splash_radius` 범위딜 |
| `slow` | 감속 디버프 부여 (딜 미미) |
| `chain` | 타겟 → 인근 `chain_count`체 연쇄, 점프마다 `chain_falloff` 감쇠 |
| `pierce` | 직선 투사체, 경로상 `pierce_count`체 관통 |
| `dot` | 화상/중독: `dot_dps` × `dot_dur`. **armor 무시** |
| `buff` | 공격 안 함. `buff_radius` 내 아군 타워에 `+buff_pct`(공속/딜) 오라 |

### 프로토타입 타워 8종

| id | kind | starter | 역할 |
|---|---|---|---|
| `bolt_tower` | single | O | 저가 단일 딜러, 밸런스 기준점 |
| `frost_totem` | slow | O | 감속 특화 |
| `mortar` | splash | X | 대물량, 느린 발사 |
| `arc_coil` | chain | X | 밀집 적 연쇄 |
| `ballista` | pierce | X | 직선 관통, brute·정렬 적 |
| `ember_brazier` | dot | X | 화상 지속딜, armor 무시 |
| `war_horn` | buff | X | 주변 타워 공속 오라 |
| `hex_obelisk` | single (고뎀·저속·고가) | X | 대 boss/brute 처형 (`damage_type = true`) |

### 투사체 (`projectile.tscn`, 루트 `Area2D`)

- 타겟 추적(호밍) 또는 직선(pierce). 피격 시(`area_entered` with hurtbox) 효과 적용 → `queue_free()` (pierce는 카운터 감소).
- `ProjectileContainer` **오브젝트 풀**로 재사용 (무한 진행이라 instantiate 부하 방지).
- **피해 계산**: `damage_type == physical` → `final = max(1, damage - armor)`. `damage_type == true`(`dot`, `hex_obelisk`) → armor 무시. brute 카운터 설계 레버.

### 업그레이드 / 판매

- 3레벨. `TowerType.tres` 안에 `levels: Array[TowerLevel]` 서브리소스 (레벨별 스탯 + `upgrade_cost`).
- L1 = `levels[0].upgrade_cost`(= 초기 배치 비용)로 배치. L1→L2 = `levels[1].upgrade_cost`. L2→L3 = `levels[2].upgrade_cost`.
- 판매 환급 = `sell_refund_rate(0.7) × 총 투자액`.
- **최종 스탯 = 기본 × 버프배율 × 유물배율** (곱연산). 매 배치·판매·유물 획득 시 재계산.

---

## 7. 경제 / 상점 / 유물

### 경제

- 수입: 적 `bounty`, 웨이브 보상 `25 + 5*N`, 조기 시작 보너스, 유물 효과.
- 지출: 타워 구매·업글, 상점 리롤.
- 시작 골드 150, 시작 기지 HP 20. 골드 상한 없음.
- 튜닝은 `EconomyTuning.tres`.

### 상점 (매 `BUILD_PHASE` 오픈)

- 슬롯 3칸. 웨이브 사이에만 조작. 웨이브 중 리롤 불가.
- **타워 해금 게이팅 (로그라이크 핵심)**: `bolt_tower` + `frost_totem`만 처음부터 빌드 메뉴에 있음. 나머지 6종은 상점에서 뽑아야 이번 런에 해금된다. → "이번 런은 어떤 빌드가 되나"의 변수. 특정 타워를 런 내내 못 볼 수도 있음.
- 슬롯 내용: 가중 랜덤 = {미해금 타워, 유물, 골드 뭉치, 기지 회복}. 이미 해금된 타워가 뽑히면 그 자리는 유물/골드 뭉치로 대체.
- **리롤 비용**: `10 + 5*(이번 페이즈 리롤 횟수)`. 페이즈마다 0으로 리셋.

### 유물 12종 (런 다양성 엔진)

획득: (a) 매 5웨이브 클리어 시 무료 3중 택1, (b) 상점 슬롯에 가중치로 등장.
같은 유물 중복 획득 불가 (획득 시 풀에서 제거).

| id | 유물 | 효과 |
|---|---|---|
| `blood_pact` | 피의 계약 | 처치 골드 +25%, 기지 최대 HP −5 |
| `frost_heart` | 서리 심장 | 모든 타워에 약한 감속 부여, 감속 타워 효과 +30% |
| `overload_core` | 과부하 코어 | 타워 공속 +20%, 사거리 −10% |
| `double_load` | 이중 장전 | 15% 확률 즉시 재발사 |
| `piercing_rune` | 관통의 룬 | 모든 투사체 `pierce_count` +1 |
| `usurer` | 고리대금업자 | 웨이브 보상 +40%, 리롤 비용 2배 |
| `vuln_mark` | 취약 표식 | 첫 피격 적 받는 피해 +15% (10초) |
| `blast_legacy` | 폭발 유산 | 적 사망 시 소형 범위 폭발 |
| `time_cog` | 시간의 톱니 | 배속 x2에서 타워만 x2.5로 작동 |
| `last_bastion` | 마지막 보루 | 기지 HP ≤ 5일 때 전 타워 딜 +50% |
| `harvest_rite` | 수확 의식 | 10킬마다 골드 +15 |
| `echo_arrow` | 메아리 화살 | `single` 타워 20% 확률로 2번째 투사체 |

- `RelicType.tres`에 `@export` 효과 파라미터(`params: Dictionary`) + `hooks: Array[StringName]`.
- 효과 로직은 `RelicService` 안에서 `id`로 분기(`match`), `params`가 숫자 공급. 순수 데이터로는 임의 로직 표현 불가하므로 **id 디스패치 + params** 가 실용적 타협.
- `RelicService`가 EventBus 시그널 구독해 모디파이어 적용. 스탯 모디파이어는 타워 최종 스탯 계산에 곱연산으로 합류.

---

## 8. 씬 트리 & 노드 구조

```
Game (game.tscn)  ─ scripts/game.gd = 상태 머신
├── Map (map_forsaken_vale.tscn 인스턴스)
│   ├── TileMapLayer
│   ├── Path2D  ← 적(PathFollow2D)의 부모
│   ├── SpawnPoint (Marker2D)
│   └── BaseZone (Area2D)
├── Services
│   ├── MapService       (그리드↔월드, occupied 집합, 배치 검증)
│   ├── WaveDirector     (절차적 웨이브 조립 + 스폰 루프)
│   ├── EconomyService   (골드, 기지 HP)
│   ├── ShopService      (오퍼 3슬롯, 리롤)
│   └── RelicService     (유물 풀, 모디파이어 적용)
├── TowerContainer
├── ProjectileContainer  (+ 오브젝트 풀)
├── Camera2D (고정)
└── HUD (CanvasLayer)
    ├── TopBar (골드 / 기지 HP / 웨이브 #)
    ├── BuildMenu (해금된 타워 버튼)
    ├── ShopPanel (3슬롯 + 리롤)
    ├── TowerInspector (선택 타워: 스탯 · 업글 · 판매 · 타겟정책 토글)
    ├── RelicChoiceModal
    ├── WaveControl (웨이브 시작 / 조기 시작)
    ├── SpeedControls (x1 · x2 · 일시정지)
    └── GameOverScreen
```

- **Autoload**: `EventBus`, `RunState`. (Project Settings > Autoload)
- **적 부모 주의**: `PathFollow2D`는 `Path2D`의 자식이어야 하므로 `WaveDirector`가 `Map/Path2D` 밑에 스폰한다. 별도 `EnemyContainer` 노드 대신 `RunState`의 배열이 논리적 레지스트리.

### 시그널 흐름 (1웨이브 예)

```
WaveControl "시작" → Game.start_wave()
  → EventBus.wave_started(N)
  → WaveDirector.build_and_run(N): Path2D 밑에 적을 시간차 스폰
적 사망 → EventBus.enemy_killed(enemy, bounty)
  → EconomyService.add_gold(bounty) → EventBus.gold_changed → HUD
  → RelicService 훅 (harvest_rite 카운트, blast_legacy 폭발 등)
적 누수 → BaseZone.area_entered → EventBus.base_damaged(leak)
  → EconomyService.damage_base() → EventBus.base_hp_changed → HUD
  → HP ≤ 0 → Game.game_over()
WaveDirector: 스폰 완료 + 잔적 0 → EventBus.wave_cleared(N)
  → EconomyService.add_gold(reward(N))
  → N % 5 == 0 → Game → RelicChoiceModal → RelicService.grant(pick)
  → Game → BUILD_PHASE, ShopService.refresh()
```

---

## 9. 데이터 스키마 (`Resource` / `.tres`)

JSON 대신 **타입 있는 커스텀 `Resource` 클래스 + `.tres`** 사용 — 인스펙터 편집 + 타입 안전 + 에디터 통합. 핫리로드 불필요.

### 클래스 정의

```gdscript
class_name EnemyType extends Resource
@export var id: StringName
@export var display_name: String
@export var sprite: Texture2D
@export var max_hp: float
@export var base_speed: float          # px/s
@export var armor: float
@export var bounty: int
@export var leak_damage: int
@export var slow_immune_threshold: float = 1.0   # boss = 0.5
@export var point_cost: int             # 웨이브 예산 코스트
@export var min_wave: int               # 해금 웨이브
```

```gdscript
class_name TowerLevel extends Resource
@export var damage: float
@export var attack_rate: float          # 초당 발사 수
@export var range_px: float
@export var upgrade_cost: int           # 이 레벨에 "도달"하는 비용 (levels[0] = 초기 배치 비용)
# kind별 (해당하는 것만 채움):
@export var splash_radius: float
@export var slow_pct: float
@export var slow_dur: float
@export var chain_count: int
@export var chain_falloff: float
@export var pierce_count: int
@export var dot_dps: float
@export var dot_dur: float
@export var buff_pct: float
@export var buff_radius: float
```

```gdscript
class_name TowerType extends Resource
enum Kind { SINGLE, SPLASH, SLOW, CHAIN, PIERCE, DOT, BUFF }
enum DamageType { PHYSICAL, TRUE }
@export var id: StringName
@export var display_name: String
@export var description: String
@export var attack_kind: Kind
@export var damage_type: DamageType = DamageType.PHYSICAL
@export var projectile: PackedScene
@export var sprite: Texture2D
@export var levels: Array[TowerLevel]   # size 3
@export var sell_refund_rate: float = 0.7
@export var starter_unlocked: bool = false
```

```gdscript
class_name RelicType extends Resource
@export var id: StringName
@export var display_name: String
@export var description: String
@export var icon: Texture2D
@export var hooks: Array[StringName]    # 예: ["on_enemy_killed", "stat_mod"]
@export var params: Dictionary          # 효과별 숫자
@export var weight: int = 10            # 상점 등장 가중치
```

```gdscript
class_name MapConfig extends Resource
@export var grid_size: Vector2i
@export var cell_px: int = 64
@export var blocked_cells: Array[Vector2i]   # 장식 차단만; 경로는 자동 도출
```

### 튜닝 리소스 (각 1개 `.tres`)

- `WaveTuning` — `budget_a/b/c`, `hp_scale_a/b`, `boss_hp_base`, `boss_hp_growth`, `speed_per_wave`, `speed_cap`, `spawn_interval_start/floor/decay`, `unlock_table: Array[Dictionary]`(`{enemy_id, min_wave, weight_early, weight_late}`), `boss_every: int`.
- `EconomyTuning` — `starting_gold`, `starting_base_hp`, `wave_reward_base`, `wave_reward_per_wave`, `early_call_bonus_per_pending`.
- `ShopTuning` — `slot_count = 3`, `reroll_base = 10`, `reroll_step = 5`, `content_weights: Dictionary`(`locked_tower / relic / gold_cache / base_heal`), `gold_cache_amount`, `base_heal_amount`, `relic_pick_every = 5`, `relic_pick_count = 3`.
- `RunConfig` — `map_scene: PackedScene`, `tower_pool: Array[TowerType]`, `relic_pool: Array[RelicType]`, `enemy_set: Array[EnemyType]`.

### 폴더 레이아웃

```
res://
  data/
    enemies/*.tres
    towers/*.tres
    relics/*.tres
    tuning/{wave_tuning,economy_tuning,shop_tuning}.tres
    run/default_run.tres
  scenes/
    game/{game,hud}.tscn
    map/map_forsaken_vale.tscn
    entities/{enemy,tower,projectile}.tscn
  scripts/
    autoload/{event_bus,run_state}.gd
    services/{map_service,wave_director,economy_service,shop_service,relic_service}.gd
    entities/{enemy,tower,projectile}.gd
    data/{enemy_type,tower_type,tower_level,relic_type,map_config, *_tuning}.gd
    game.gd
```

---

## 10. 구현 순서 (마일스톤)

사용자 제시 순서 반영: 타일맵 맵+경로 → 적 스폰·경로 이동 → 타워 배치(그리드 스냅) → 타워 감지·발사 → 웨이브/골드/체력 → UI·밸런싱. 각 마일스톤은 독립 검증 가능.

| M | 내용 | 검증 | 일수 | codex |
|---|---|---|---|---|
| M0 | 프로젝트 뼈대, 폴더, autoload 스텁, Resource 클래스, 상태 enum, 고정 Camera2D | 실행됨, 상태 출력 | 0.5 | |
| M1 | 타일맵 맵 + `Path2D` + `SpawnPoint` + `BaseZone` + `MapConfig.tres` + `MapService`(그리드/경로셀 래스터화/배치검증) + 디버그 오버레이 | 배치가능 셀 정확, 경로 자동 제외 | 1 | |
| M2 | `enemy.tscn`(PathFollow2D 루트) + `EnemyType`×4 + `HealthComponent` + `DebuffComponent` 스텁 + `WaveDirector` 최소(고정 리스트 스폰·이동·누수→시그널) | 적 시작→기지 이동, despawn, 누수 시그널 | 1 | |
| M3 | `tower.tscn` + `TowerType`×8(러프 스탯) + `BuildMenu`(스타터 2종) + 고스트 프리뷰(녹/적 틴트, 사거리 원) + `MapService.occupied` + `EconomyService` 골드 + `TopBar` | 배치/거부, 골드 차감, 셀 점유 | 1 | |
| M4 | 타게팅(FIRST/CLOSEST/STRONGEST) + `AttackTimer` + `projectile.tscn` + `ProjectileContainer` 풀 + **7가지 attack_kind** + armor·damage_type 피해 계산 | 타워별 거동 차이 확인, 적 사망·bounty 지급 | 2 | O (attack_kind 해석, 풀) |
| M5 | `Game` 상태머신 완성(BUILD↔WAVE_ACTIVE↔WAVE_CLEARED) + 조기 시작 보너스 + **절차적 웨이브**(`WaveTuning` 예산·해금·스케일·시드) + 보스(10웨이브) + `GAME_OVER` + 결과 화면 | 웨이브 15까지 난이도 상승, 시드 재현, 게임오버 동작 | 1.5 | O (예산 솔버) |
| M6 | 3레벨 업그레이드 + 판매 환급 + `TowerInspector` + 타겟정책 토글 | 스탯 변화·환급 정확 | 1 | |
| M7 | `ShopService`(가중 3슬롯) + 리롤 공식 + **타워 해금 게이팅** + 골드 뭉치/기지 회복 | 상점으로 타워 해금, 리롤 비용 증감·페이즈 리셋 | 1.5 | O (가중 롤) |
| M8 | `RelicService` 유물 12종 + `RelicChoiceModal`(5웨이브마다) + 상점 유물 슬롯 + 모디파이어 합성(기본×버프×유물) | 유물별 체감 변화, 중복 획득 없음 | 2 | O (id 디스패치 핸들러) |
| M9 | 배속 x1/x2 + 일시정지 + 재시작 + AI 아트 스프라이트 교체 + 시드 런 1차 밸런싱 패스 | 전체 루프 일관, 웨이브 20+ 도달 가능 (잘하면), 못하면 이른 패배 | 2 | |

**합계 ≈ 3주 (1인).** M1–M5 = "코어 작동", M6–M8 = "로그라이크 성립", M9 = "재미 판정".

### codex 협업 모델

- Claude: 아키텍처, 데이터 스키마, 상태 머신, 시그널 배선, 통합, 코드 리뷰 소유.
- codex: 계약서(입출력 / 시그널 / 부작용 명시)와 함께 자기완결적 알고리즘 덩어리를 위임받음 — attack_kind 해석(M4), 오브젝트 풀, 가중 상점 롤(M7), 유물 효과 핸들러(M8), 웨이브 예산 솔버(M5).
- 각 codex 산출물은 통합 전 전용 테스트 씬에서 검증.

---

## 11. 재미 검증 기준 (M9 이후 판단)

프로토타입이 "다음 단계로 갈 가치가 있는가"를 아래로 판단:

1. **빌드 분기 체감** — 서로 다른 시드 3개를 플레이했을 때, 상점·유물 때문에 실제로 다른 전략을 강요받는가? (같은 플레이가 반복되면 실패.)
2. **의사결정 긴장** — 조기 시작 버튼 / 리롤 / 유물 선택에서 "고민"이 생기는가?
3. **난이도 곡선** — 잘하면 웨이브 20+, 못하면 웨이브 8~12에서 패배. 무지성 배치로 무한 진행이 되면 실패.
4. **"한 판 더"** — 게임오버 후 재시작을 누르고 싶은가?

이 중 3개 이상 충족 시 메타 progression 설계로 진행. 미달 시 코어 루프 재설계 (엔진·스키마는 유지).

---

## 부록 A: 명시적으로 미룬 결정

| 항목 | 미룬 이유 | 다시 볼 시점 |
|---|---|---|
| 세계관·네이밍·아트 스타일 확정 | 프로토타입은 러프 아트로 재미만 검증 | 재미 검증 통과 후 |
| 카메라 줌/팬 (핀치 줌) | 고정 카메라로 충분 | 맵이 커질 때 |
| 사운드 / BGM | 재미 판단에 필수 아님 | 세로 슬라이스 단계 |
| 세이브/로드, 이어하기 | 한 판이 짧고 휘발적 | 메타 progression 도입 시 |
| 데미지 타입 다양화 (물리/마법/참 등 3종+) | brute 카운터엔 physical/true 2종이면 충분 | 적 종류 확장 시 |
| 멀티 레인 / 경로 분기 | 단일 경로로 코어 검증 | 맵 다양화 단계 |
| 캐릭터 대량 콘텐츠, 퓨전 | 스코프 폭발의 주범 (예전 실패 원인) | 코어가 재밌다고 확인된 후 |
| PvP | 최후순위 확장 | 정식 버전 로드맵 |
