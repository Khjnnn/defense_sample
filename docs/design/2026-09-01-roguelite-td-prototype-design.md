# 로그라이크 미로 타워디펜스 — 프로토타입 기획서

- **문서 버전**: v2.0
- **작성일**: 2026-09-01 (v1), 개정 2026-09-01 (v2)
- **목표**: 재미 검증용 플레이 가능한 프로토타입 (iOS/Android)
- **상태**: 설계 확정, 구현 착수 전

### v2 변경 요약

v1(고정 경로 TD)을 확장 검토 후 다음을 반영:

1. **핵심 차별점 확정** — 플레이어가 타워를 세워 **경로 자체를 만든다**(미로 TD). 적은 스폰→기지를 경로탐색으로 이동하며, 타워는 장애물이 되어 적을 빙빙 돌게 만든다. **단, 출구를 완전히 막을 수는 없다.** ← 이게 프로토타입이 검증할 가설.
2. **PvP 삭제** — 먼 미래의 별개 프로젝트로 격하. 이 문서·아키텍처에서 고려하지 않음.
3. 타워 8→5종, 유물 12→6종 (밸런싱 부담 감축).
4. 무한 진행 → **웨이브 40 최종 보스 = 런 승리** (원하면 이후 무한 계속).
5. 런 시작 시 **시작 유물 3중 택1**.
6. M1~M5는 **순진하게(naive) 구현** 후 재미 확인 시 리팩터링.
7. M9에 **게임필(juice) 최소 세트** 명시.
8. **터치 조작 명세** 섹션 추가 + 온보딩 + 골드 소비처.
9. 견적 3주 → **5~7주**로 정정, 오픈소스 포크 옵션 병기.

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
- **프로토타입은 하나의 가설을 검증한다**: "타워로 미로를 만들어 적을 농락하는 것"이 재밌는가.

예전 프로젝트 참고 자산: `C:\dev_auto_plan\claude-harness-main` (캐릭터 데이터시트, 퓨전 레시피). 이번엔 **참고만** 하고 이식하지 않는다.

---

## 1. 확정 사항

| 항목 | 결정 |
|---|---|
| 장르 | **미로형** 로그라이크 타워디펜스 (플레이어가 타워로 경로를 설계) |
| 핵심 차별점 (검증 가설) | 타워 = 장애물. 적은 경로탐색으로 스폰→기지 이동. 타워를 세워 적을 우회·순환시켜 더 오래 때린다. **출구를 완전히 막는 배치는 불가.** |
| 런 구조 | 단일 맵, 웨이브 1→40, 웨이브 사이 랜덤 상점. 웨이브 40 최종 보스 처치 = 런 승리(이후 무한 계속 가능). 기지 HP 0 = 런 종료. 한 판 20~35분. |
| 톤 | 다크 판타지 / 로그라이크 정서 |
| 확장 방향 | (이후) 메타 수집·성장. PvP는 격하 — 이 프로젝트 범위 아님. |
| 엔진 | Godot 4.x |
| 언어 | GDScript (데이터 주도 설계) |
| 리소스 | 1인 개발, 아트 = AI 생성, 게임개발 첫 경험, 웹/TS 배경 |
| 협업 | Claude + codex 2인 1조. Claude = 아키텍처·스키마·상태머신·통합·리뷰. codex = 계약 기반 알고리즘 위임. |

### 언어 선택 근거 (GDScript)

- Godot TD·미로 TD 튜토리얼 대다수가 GDScript → 막혔을 때 검색 즉시.
- 미로 TD의 핵심인 경로탐색은 Godot 내장 `AStarGrid2D`로 해결 → 외부 라이브러리 불필요.
- 웹(Phaser/Pixi + TS) 대안은 예전에 아키텍처가 붕괴된 경로라 배제.
- 데이터 주도로 설계하면 느슨한 타입 문제를 완화.

---

## 2. 프로토타입 스코프

### 포함 (IN)

- 그리드 필드 맵 1개 (고정 경로 없음), 스폰 1곳 · 기지 1곳
- 적 4종 + 미니보스(웨이브 10/20/30) + 최종 보스(웨이브 40)
- 웨이브 절차적 생성 + HP/속도 스케일링, 40웨이브 구조
- 타워 5종 (역할 분리), **그리드 배치 = 장애물화**, "출구 봉쇄 불가" 검증
- 적 경로탐색 (`AStarGrid2D`, 공유 경로) + 미로 변경 시 재탐색
- 3레벨 업그레이드, 판매
- 웨이브 사이 랜덤 상점 (3슬롯, 리롤), 타워 해금 게이팅
- 유물 6종, 런 시작 시 시작 유물 3중 택1, 이후 5웨이브마다 3중 택1
- 웨이브 중 골드 소비 소모품 1종 (골드 싱크)
- 기지 체력, 승리/패배 화면 (도달 웨이브 / 처치 / 생존 시간 / 최종 경로 길이)
- 배속 x1/x2, 일시정지(일시정지 중 배치 허용), 재시작
- 조기 웨이브 시작 버튼 (탐욕 결정 장치)
- 터치 조작, 첫 3웨이브 온보딩 툴팁
- 게임필 최소 세트 (M9), AI 아트 스프라이트

### 제외 (OUT — 프로토타입 이후)

- 계정/로그인/서버, 에너지, 메타 progression, 타워 카드, 광고/IAP
- 캐릭터 대량 콘텐츠, 퓨전
- 여러 맵/챕터, 스토리 연출, 멀티 스폰/멀티 기지
- PvP (먼 미래 별개 프로젝트)
- 사운드 폴리싱, 세이브/로드

---

## 3. 코어 루프 & 상태 머신

### 게임 상태

```
BOOT → MAIN_MENU → RUN_START(시작 유물 택1) → PLAYING → (RUN_WON | GAME_OVER) → RESULTS → MAIN_MENU
```

### PLAYING 내부 페이즈

```
        ┌──────────────────────────────────────────────┐
        ▼                                              │
   BUILD_PHASE ──[웨이브 시작 버튼]──► WAVE_ACTIVE ──────┘
   - 적 없음 (또는 이전 낙오병)          - 스포너가 이번 웨이브 적 방출
   - 상점 3슬롯 오픈 (리롤 = 골드)       - 적이 현재 공유 경로를 따라 이동
   - 타워 구매/배치/업글/판매           - 타워 발사, 처치 시 골드, 기지 도달 시 피해
   - (미로 자유 재설계)                  - 스폰 종료 + 잔적 0 → WAVE_CLEARED
   - 유물 선택 (5의 배수 웨이브 후)
```

`WAVE_CLEARED` → 웨이브 보상 골드 → (웨이브 % 5 == 0 이면) 유물 3중 택1 → `BUILD_PHASE`, 웨이브 인덱스 +1.

### 진행 & 종료

- 웨이브 1~40. 10/20/30 = 미니보스 포함 웨이브, 40 = 최종 보스 단독+호위.
- **최종 보스 처치 → `RUN_WON`.** 결과 화면에서 "계속하기(무한 모드)" 선택 시 웨이브 41+ 무한 진행(스케일링 지속).
- 기지 HP 0 → `GAME_OVER` (어느 웨이브에서든).
- 결과 화면 지표(재미 검증용): 도달/클리어 웨이브, 총 처치, 생존 시간, **최종 경로 길이(셀 수)**, 보유 유물, 최종 타워 구성.

### 조기 시작 버튼 (탐욕 결정 장치)

`WAVE_ACTIVE` 중에도 "다음 웨이브 조기 시작" 버튼 활성. 누르면 다음 웨이브가 겹쳐 스폰되고 보너스 골드(`남은 스폰 예정 수 × 2G`). 위험 감수로 경제 가속.

### 아키텍처 방침 (중요)

- **M1~M5는 순진하게 구현한다.** 하드코딩, 직접 노드 참조, 배열 순회. 목표는 "미로 만들기가 재밌는가" 확인이지 아키텍처 완성도가 아니다.
- 재미가 확인되면(§12 기준) 아래 목표 구조로 리팩터링:
  - `EventBus` (autoload 싱글턴) 전역 시그널 허브 — `enemy_killed`, `base_damaged`, `gold_changed`, `wave_started`, `wave_cleared`, `maze_changed`, `path_recomputed`, `relic_acquired`.
  - `RunState` (autoload) 런 휘발 데이터 — 골드, 기지 HP, 웨이브 인덱스, 보유 유물, 배치 타워 배열, 해금 타워 id 집합, `run_seed`.
  - 서비스 노드 분리 (`MapService` / `WaveDirector` / `EconomyService` / `ShopService` / `RelicService`).
- 시뮬레이션 `_physics_process(delta)` 기준(60fps). 일시정지 = `get_tree().paused`. 배속 = `Engine.time_scale`.

---

## 4. 맵 / 그리드 / 경로탐색 (핵심 시스템)

### 필드

- **가로(landscape) 고정.** 셀 64px, 필드 약 **16×12셀** (1024×768). 미로를 짤 공간이 필요하므로 v1보다 열려 있고 정사각형에 가깝게.
- 사전 경로 없음. 스폰 셀 = 좌측 가장자리 중앙, 기지 셀 = 우측 가장자리 중앙.
- 사전 차단 셀(`blocked_cells`)은 최소 (0~4개, 장식용 바위). 미로는 플레이어가 만든다.
- 카메라 고정, 스크롤/줌 없음 (줌은 이후).

### 경로탐색 — Godot 내장 `AStarGrid2D`

- `MapService`가 `AStarGrid2D` 하나를 소유. 각 셀은 `solid`(타워 있음 또는 `blocked_cells`) 또는 통행 가능.
- **공유 경로**: 스폰·기지가 각 1개이므로 모든 적이 같은 경로를 쓴다. `MapService.recompute_path()` → `astar.get_point_path(spawn, base)` → `current_path: PackedVector2Array` (셀 중심 좌표 배열).
- 재탐색 트리거: 타워 배치·판매 시 1회 (`maze_changed` → `recompute_path` → `path_recomputed`). 프레임마다 하지 않음 (배치는 드묾).
- 적은 `current_path`의 인덱스를 추적. 재탐색 시 각 적은 "자기 위치에서 가장 가까운, 진행 방향상의 경로 노드"로 스냅해 이어감.

### 타워 배치 규칙 (차별점의 핵심)

배치 시도 셀 `c`에 대해, 아래 **모두** 만족해야 배치 가능:

1. 그리드 범위 내 ∧ `c` != 스폰 셀 ∧ `c` != 기지 셀
2. `c` not in `blocked_cells`
3. `c` not in `occupied_cells` (이미 타워)
4. `c`에 현재 적이 서 있지 않음
5. **`c`를 solid로 가정했을 때, 스폰→기지 경로가 여전히 존재한다** (`astar` 임시 solid 설정 → `get_id_path` 비어있지 않은지 확인 → 원복). ← "출구 봉쇄 불가"

배치 프리뷰: 그리드 스냅 고스트 + 유효/무효 틴트. 무효 사유가 (5)일 때 = "이 위치는 길을 완전히 막습니다" 툴팁. 배치 확정 순간 미로 경로가 새로 그려지는 애니메이션(§11 juice).

### 적 이동 (v1에서 변경 — PathFollow2D 폐기)

- 적 = `Node2D`(또는 `CharacterBody2D`). `current_path[idx]`를 향해 `current_speed * delta`로 이동, 도달하면 `idx++`.
- `idx`가 경로 끝 → 기지 도달 → `base_damaged`.
- `current_speed = base_speed * slow_mult` (감속 중첩은 최강값, 하한 클램프).
- 적은 타워를 공격하지 않는다 (타워 = 무적 벽). 프로토타입 단순화.

### 성능 주의

- `AStarGrid2D` 재탐색은 배치당 1회라 저렴. 문제는 후반 다수 적 × 다수 타워의 `Area2D` 범위 판정 → §6 참고. 웨이브 40 상한이 이 부담도 제한한다.

---

## 5. 적 / 웨이브

### 적 엔티티 (`enemy.tscn`, 루트 `Node2D`)

- 자식: `Sprite2D`, `HealthComponent`, `Area2D`(피격 hurtbox), `DebuffComponent`(감속/도트).
- 이동: §4 참고 (공유 경로 인덱스 추적).
- 사망: HP ≤ 0 → `enemy_killed(self, bounty)` → 연출 → `queue_free()`.
- 기지 도달: `base_damaged(leak_damage)` → `queue_free()`.
- 스탯: `max_hp`, `base_speed`(px/s), `armor`(고정 감산, 최소 1 보장), `bounty`, `leak_damage`, `slow_immune_threshold`.

### 적 4종 + 보스

| id | 역할 | HP | 속도 | armor | bounty | leak | 특이 |
|---|---|---|---|---|---|---|---|
| `shambler` | 기본 | 100 | 60 | 0 | 6 | 1 | 밸런스 기준점 |
| `hound` | 속공 | 55 | 130 | 0 | 8 | 1 | 감속 가치 부여. 미로가 길수록 오래 노출 |
| `swarmling` | 물량 | 30 | 85 | 0 | 3 | 1 | 큰 무리, 좁은 미로 통로에서 범위딜 극대화 |
| `brute` | 탱커 | 480 | 38 | 6 | 20 | 3 | armor로 저단일딜 무력화 → 더 긴 미로로 처치 시간 확보 강요 |
| 미니보스 | 10/20/30 | 곡선 별도 | 34 | 8 | 80 | 6 | 해당 웨이브에 잡몹과 함께 |
| 최종 보스 | 40 | 곡선 별도 | 30 | 12 | 300 | 20 | 감속 50%↑ 면역. 호위 소수 동반 |

### 웨이브 = 절차적 생성 + 시드

`WaveDirector`가 공식으로 웨이브 N(1~40) 조립. 무한 계속 모드는 N>40에서 같은 공식 연장.

- **예산제**: `budget(N) = 40 + 14*N + 0.5*N*N`. 예산으로 적 카드 구매(타입별 `point_cost`), 해금 준수:
  - W1+ `shambler` / W3+ `hound` / W5+ `swarmling` / W8+ `brute` / W10·20·30 미니보스 / W40 최종 보스.
- **구성 가중치**가 N에 따라 이동 (초반 `shambler` 위주 → 후반 혼합).
- **HP 스케일**: `hp_mult(N) = 1 + 0.075*(N-1) + 0.0035*(N-1)^2` (보스 제외).
- **보스 HP**: 미니보스 `1200 * (1 + 0.6*(N/10 - 1))`. 최종 보스 `12000` 고정(+무한 모드 시 웨이브당 +8%).
- **속도 스케일**: 웨이브당 +0.4%, +35% 상한.
- **스폰 간격**: `0.9s → 0.45s` 하한, N 따라 축소. 타입별 버스트.
- **결정성**: `WaveDirector`를 `run_seed + N`으로 시드 → **웨이브 구성**은 재현됨. (전투 결과는 프레임 타이밍·부동소수점 때문에 완전 결정적이지 않음 — 구성 재현까지만 기대.)
- **웨이브 보상**: `reward(N) = 25 + 5*N` 골드.

### 데이터 주도

- `EnemyType.tres` — 타입별 스탯 + 씬 + 스프라이트.
- `WaveTuning.tres` — 예산/HP/속도/보스 계수, 해금 테이블, 케이던스 전부 `@export`.

---

## 6. 타워 / 투사체 / 타게팅

### 타워 엔티티 (`tower.tscn`, 루트 `Node2D`)

- 자식: `Sprite2D`, `RangeArea`(`Area2D`+`CircleShape2D`), `AttackTimer`(`Timer`), `TurretPivot`(선택).
- 배치 순간 `MapService`에 셀을 solid로 등록 → `maze_changed` 발신.
- 타게팅: `RangeArea`의 `in_range` 배열에서 정책으로 선정 → `AttackTimer.timeout` 시 발사.
- 타겟 정책: `FIRST`(공유 경로 인덱스 최대 = 기지에 가장 가까움) / `CLOSEST` / `STRONGEST`(최대 HP). 기본 `FIRST`.

### 공격 종류 (`attack_kind`) — 5종으로 축소

| kind | 동작 | 미로에서의 의미 |
|---|---|---|
| `single` | 투사체 1발, 단일 타겟 | 저가 딜러 겸 저가 벽. 미로의 기본 블록 |
| `splash` | 착탄 폭발, `splash_radius` 범위딜 | 미로 통로에 뭉친 적 정리, brute 카운터 조립 |
| `slow` | 감속 디버프 | 적 순환 시간을 늘려 다른 타워 딜 누적 |
| `chain` | 인근 `chain_count`체 연쇄, `chain_falloff` 감쇠 | 좁은 미로 라인에 밀집한 적에게 폭발적 |
| `buff` | 공격 안 함. `buff_radius` 내 아군 공속/딜 오라 | 벽 슬롯 하나를 비딜 지원에 쓰는 트레이드오프 결정 |

> 제외(프로토타입 후 검토): `pierce`(긴 직선 통로 설계 시 강력하나 5종 제약), `dot`/`hex`(brute 전용 카운터였으나 미로 길이로 대체 가능).

### 프로토타입 타워 5종

| id | kind | starter | 역할 |
|---|---|---|---|
| `bolt_tower` | single | O | 저가 단일 딜러 + 기본 벽 |
| `frost_totem` | slow | O | 감속 특화 |
| `mortar` | splash | X | 대물량 / brute 조립 |
| `arc_coil` | chain | X | 밀집 연쇄 |
| `war_horn` | buff | X | 주변 공속 오라 |

### 투사체 (`projectile.tscn`, 루트 `Area2D`)

- 타겟 추적(호밍). 피격 시 효과 적용 → `queue_free()`.
- `ProjectileContainer` **오브젝트 풀** (재사용).
- **피해 계산**: `final = max(1, damage - armor)`. (프로토타입은 damage_type 단일 — 물리만. brute는 미로 길이 + 집중포화로 상대.)

### 업그레이드 / 판매

- 3레벨. `TowerType.tres`의 `levels: Array[TowerLevel]` (레벨별 스탯 + `upgrade_cost`). `levels[0].upgrade_cost` = 초기 배치 비용.
- 판매 환급 = `sell_refund_rate(0.6) × 총 투자액`. (유물 `미로공학` 획득 시 1.0)
- 판매 시 셀을 통행 가능으로 되돌리고 `maze_changed` 발신 → 경로 재탐색.
- **최종 스탯 = 기본 × 버프배율 × 유물배율** (곱연산). 배치·판매·유물 획득·`war_horn` 변화 시 재계산.

---

## 7. 경제 / 상점 / 유물

### 경제

- 수입: 적 `bounty`, 웨이브 보상 `25 + 5*N`, 조기 시작 보너스, 유물 효과.
- 지출: 타워 구매·업글, 상점 리롤, **웨이브 중 소모품**(골드 싱크).
- 시작 골드 150, 시작 기지 HP 20. 골드 상한 없음.
- **골드 싱크 — 소모품 "화염 폭격"**: `WAVE_ACTIVE` 중 버튼, 골드 `60 + 20*사용횟수` 지불, 지정 지점에 즉시 대형 범위딜. 후반 골드 인플레이션 흡수 + 위기 대응 결정.
- 튜닝은 `EconomyTuning.tres`.

### 상점 (매 `BUILD_PHASE` 오픈)

- 슬롯 3칸. 웨이브 사이에만 조작.
- **타워 해금 게이팅**: `bolt_tower` + `frost_totem`만 시작부터 빌드 메뉴에. 나머지 3종(`mortar`/`arc_coil`/`war_horn`)은 상점에서 뽑아야 이번 런에 해금. 특정 타워를 런 내내 못 볼 수도 있음.
- 슬롯 내용: 가중 랜덤 = {미해금 타워, 유물, 골드 뭉치, 기지 회복}. 이미 해금된 타워가 뽑히면 유물/골드로 대체.
- 리롤 비용: `10 + 5*(이번 페이즈 리롤 횟수)`. 페이즈마다 0으로 리셋.

### 유물 6종 (런 다양성 엔진 — 미로 메커닉과 연동)

획득: (a) **런 시작 시 3중 택1**, (b) 매 5웨이브 클리어 시 3중 택1, (c) 상점 슬롯 가중 등장. 중복 획득 불가.

| id | 유물 | 효과 | 미로 연동 |
|---|---|---|---|
| `blood_pact` | 피의 계약 | 처치 골드 +25%, 기지 최대 HP −5 | 경제 가속 |
| `overload_core` | 과부하 코어 | 타워 공속 +20%, 사거리 −10% | 사거리↓ → 미로를 더 촘촘히 짜야 함 |
| `labyrinth_eng` | 미로공학 | 타워 판매 100% 환급 | 미로 자유 재설계 |
| `toll_keeper` | 관문세 | 적이 지나온 셀 수 × 0.5G 추가 처치 보상 | 긴 미로 = 더 큰 수입 |
| `long_road` | 길 잃은 자 | 현재 경로 길이 ≥ 24셀이면 전 타워 딜 +30% | 긴 미로 유지 보상 |
| `blast_legacy` | 폭발 유산 | 적 사망 시 소형 범위 폭발 | 미로 통로 연쇄 처치 |

- `RelicType.tres` — `@export` 효과 파라미터(`params: Dictionary`) + `hooks: Array[StringName]`.
- 효과 로직은 `RelicService` 안에서 `id`로 분기(`match`), `params`가 숫자 공급 (순수 데이터로 임의 로직 불가 → id 디스패치 + params 타협).
- `RelicService`가 EventBus 시그널 구독, 모디파이어를 타워 최종 스탯/경제 계산에 곱연산 합류.

---

## 8. 씬 트리 & 노드 구조 (목표 구조 — M1~M5는 축약본)

```
Game (game.tscn)  ─ scripts/game.gd = 상태 머신
├── Map (map_field.tscn 인스턴스)
│   ├── TileMapLayer (바닥)
│   ├── SpawnMarker (Marker2D)
│   └── BaseZone (Area2D)
├── Services
│   ├── MapService       (AStarGrid2D 소유, solid 집합, 경로 재탐색, 배치 합법성 검증)
│   ├── WaveDirector     (절차적 웨이브 조립 + 스폰 루프)
│   ├── EconomyService   (골드, 기지 HP, 소모품)
│   ├── ShopService      (오퍼 3슬롯, 리롤)
│   └── RelicService     (유물 풀, 모디파이어)
├── EnemyContainer       (적 부모 — v2에서 부활. Path2D 없으므로 일반 노드)
├── TowerContainer
├── ProjectileContainer  (+ 오브젝트 풀)
├── Camera2D (고정)
└── HUD (CanvasLayer)
    ├── TopBar (골드 / 기지 HP / 웨이브 # / 현재 경로 길이)
    ├── BuildMenu (해금된 타워 버튼)
    ├── ShopPanel (3슬롯 + 리롤)
    ├── TowerInspector (스탯 · 업글 · 판매 · 타겟정책 토글)
    ├── RelicChoiceModal
    ├── WaveControl (웨이브 시작 / 조기 시작 / 화염 폭격)
    ├── SpeedControls (x1 · x2 · 일시정지)
    ├── OnboardingTips (첫 3웨이브)
    └── EndScreen (승리 / 패배)
```

- **Autoload**: `EventBus`, `RunState` (리팩터링 후).
- **적 부모**: v1의 Path2D 제약이 사라져 `EnemyContainer` 일반 노드로 관리. `RunState`가 논리 레지스트리 겸용.

### 시그널 흐름 (미로 변경 + 1웨이브)

```
타워 배치 → MapService.try_place(cell)
  → 합법성 검증(§4-5) 통과 시 solid 등록
  → EventBus.maze_changed → MapService.recompute_path()
  → EventBus.path_recomputed(new_path) → 적들 경로 인덱스 재스냅 + HUD 경로 길이 갱신

WaveControl "시작" → Game.start_wave() → EventBus.wave_started(N)
  → WaveDirector.build_and_run(N): EnemyContainer에 시간차 스폰
적 사망 → EventBus.enemy_killed(enemy, bounty)
  → EconomyService.add_gold(bounty) → gold_changed → HUD
  → RelicService 훅 (toll_keeper 정산, blast_legacy 폭발)
적 누수 → BaseZone.area_entered → EventBus.base_damaged(leak)
  → EconomyService.damage_base() → base_hp_changed → HUD
  → HP ≤ 0 → Game.game_over()
WaveDirector: 스폰 완료 + 잔적 0 → EventBus.wave_cleared(N)
  → EconomyService.add_gold(reward(N))
  → N == 40 ∧ 보스 처치 → Game.run_won()
  → N % 5 == 0 → RelicChoiceModal → RelicService.grant(pick)
  → Game → BUILD_PHASE, ShopService.refresh()
```

---

## 9. 데이터 스키마 (`Resource` / `.tres`)

JSON 대신 **타입 있는 커스텀 `Resource` 클래스 + `.tres`** — 인스펙터 편집 + 타입 안전 + 에디터 통합.

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
@export var slow_immune_threshold: float = 1.0   # 최종 보스 = 0.5
@export var point_cost: int             # 웨이브 예산 코스트
@export var min_wave: int               # 해금 웨이브
```

```gdscript
class_name TowerLevel extends Resource
@export var damage: float
@export var attack_rate: float          # 초당 발사 수
@export var range_px: float
@export var upgrade_cost: int           # 이 레벨 도달 비용 (levels[0] = 초기 배치 비용)
# kind별 (해당하는 것만):
@export var splash_radius: float
@export var slow_pct: float
@export var slow_dur: float
@export var chain_count: int
@export var chain_falloff: float
@export var buff_pct: float
@export var buff_radius: float
```

```gdscript
class_name TowerType extends Resource
enum Kind { SINGLE, SPLASH, SLOW, CHAIN, BUFF }
@export var id: StringName
@export var display_name: String
@export var description: String
@export var attack_kind: Kind
@export var projectile: PackedScene
@export var sprite: Texture2D
@export var levels: Array[TowerLevel]   # size 3
@export var sell_refund_rate: float = 0.6
@export var starter_unlocked: bool = false
```

```gdscript
class_name RelicType extends Resource
@export var id: StringName
@export var display_name: String
@export var description: String
@export var icon: Texture2D
@export var hooks: Array[StringName]    # 예: ["on_enemy_killed", "stat_mod", "on_path_recomputed"]
@export var params: Dictionary
@export var weight: int = 10
```

```gdscript
class_name MapConfig extends Resource
@export var grid_size: Vector2i = Vector2i(16, 12)
@export var cell_px: int = 64
@export var spawn_cell: Vector2i
@export var base_cell: Vector2i
@export var blocked_cells: Array[Vector2i]   # 장식 차단만
```

### 튜닝 리소스 (각 1개 `.tres`)

- `WaveTuning` — `budget_a/b/c`, `hp_scale_a/b`, `miniboss_hp_base/growth`, `finalboss_hp`, `endless_boss_growth`, `speed_per_wave`, `speed_cap`, `spawn_interval_start/floor/decay`, `unlock_table: Array[Dictionary]`, `total_waves = 40`, `boss_waves = [10,20,30,40]`.
- `EconomyTuning` — `starting_gold`, `starting_base_hp`, `wave_reward_base`, `wave_reward_per_wave`, `early_call_bonus_per_pending`, `airstrike_base_cost`, `airstrike_cost_step`, `airstrike_radius`, `airstrike_damage`.
- `ShopTuning` — `slot_count = 3`, `reroll_base = 10`, `reroll_step = 5`, `content_weights: Dictionary`, `gold_cache_amount`, `base_heal_amount`, `relic_pick_every = 5`, `relic_pick_count = 3`, `starting_relic_pick_count = 3`.
- `RunConfig` — `map_scene`, `tower_pool: Array[TowerType]`, `relic_pool: Array[RelicType]`, `enemy_set: Array[EnemyType]`.

### 폴더 레이아웃

```
res://
  data/{enemies,towers,relics,tuning,run}/*.tres
  scenes/game/{game,hud}.tscn
  scenes/map/map_field.tscn
  scenes/entities/{enemy,tower,projectile}.tscn
  scripts/autoload/{event_bus,run_state}.gd
  scripts/services/{map_service,wave_director,economy_service,shop_service,relic_service}.gd
  scripts/entities/{enemy,tower,projectile}.gd
  scripts/data/{enemy_type,tower_type,tower_level,relic_type,map_config,*_tuning}.gd
  scripts/game.gd
```

---

## 10. 터치 조작 명세

| 제스처 | 대상 | 동작 |
|---|---|---|
| 탭 | 빌드 메뉴 타워 버튼 | 해당 타워 "배치 모드" 진입 (버튼 하이라이트 유지) |
| 탭 | 배치 모드에서 빈 셀 | 그 셀에 배치 (골드 충분 ∧ 합법). 배치 후 모드 유지 → 연속 배치로 미로 빠르게 |
| 탭 | 배치 모드에서 무효 셀 | 무효 사유 툴팁 ("길을 완전히 막습니다" 등), 배치 안 됨 |
| 탭 | 빈 공간 / 다시 버튼 | 배치 모드 해제 |
| 탭 | 기존 타워 | `TowerInspector` 열림 (업글 / 판매 / 타겟정책) |
| 탭 | 화염 폭격 버튼 → 지점 | 소모품 발동 |
| 드래그 | 없음 | 미사용 (카메라 고정) |
| 롱프레스 | — | 미사용 (오조작 방지) |

- **사거리 원**: 배치 모드·타워 선택 시 표시. 손가락 가림 방지 위해 프리뷰 고스트를 탭 지점보다 살짝 위로 오프셋.
- **판매**: `TowerInspector` 내 버튼. L2 이상은 확인 팝업. 스와이프 판매 없음.
- **일시정지 중 배치 허용** (전략성 — 킹덤러쉬 방식). 배속과 별개.
- **가로 강제**: 세로로 들면 "기기를 가로로 돌려주세요" 안내 오버레이.
- **온보딩 (첫 런 웨이브 1~3)**: 순차 툴팁 — ① "타워를 세워 적의 길을 늘려보세요 (완전히 막을 순 없습니다)" ② "적이 빙 돌아가는 동안 타워가 더 오래 공격합니다" ③ "웨이브 사이 상점에서 새 타워를 해금하세요". 경로 라인 강조 표시.

---

## 11. 구현 순서 (마일스톤)

사용자 제시 순서 반영, 미로 메커닉에 맞게 조정. 각 마일스톤 독립 검증. **M1~M5 순진하게 구현.**

| M | 내용 | 검증 | 일수 | codex |
|---|---|---|---|---|
| M0 | Godot 4 프로젝트, 폴더, Resource 클래스, 상태 enum, 고정 Camera2D, 빈 씬 | 실행됨, 상태 출력 | 1 | |
| M1 | 그리드 필드 + `AStarGrid2D` + 스폰/기지 + `MapService`(solid 집합, `recompute_path`, `is_placement_legal` = 임시 solid 후 경로 존재 확인) + 현재 경로 디버그 라인 | 임의 셀 solid 토글 시 경로가 우회함, 봉쇄 시 "불가" 반환 | 2 | O (배치 합법성 검증) |
| M2 | `enemy.tscn`(Node2D, 공유 경로 인덱스 추적) + `EnemyType`×4 + 미로 변경 시 재스냅 + 누수→시그널 + `WaveDirector` 최소(고정 리스트 스폰) | 적이 현재 경로 따라 이동, 배치로 경로 바뀌면 즉시 우회, 기지 도달 시 누수 | 2 | |
| M3 | `tower.tscn` 배치 = solid 등록 + "봉쇄 불가" 검증 UI + 빌드 메뉴(스타터 2종) + 고스트 프리뷰 + `EconomyService` 골드 + `TopBar`(골드/경로길이) | 배치/거부 정확, 골드 차감, 경로 길이 실시간 갱신 | 2 | |
| M4 | 타게팅(경로 인덱스 기준 FIRST/CLOSEST/STRONGEST) + `AttackTimer` + `projectile.tscn` + 풀 + **5가지 attack_kind** + armor 피해 계산 | 타워별 거동 차이, 적 사망·bounty | 2.5 | O (attack_kind 해석, 풀) |
| M5 | `Game` 상태머신(RUN_START→BUILD↔WAVE→WON/OVER) + 시작 유물 택1 + 조기 시작 + **절차적 웨이브**(`WaveTuning`, 40웨이브, 미니/최종보스) + 승리/패배 화면 | 웨이브 40까지 난이도 상승, 보스 웨이브 동작, 승리·패배 분기, 시드로 구성 재현 | 3 | O (예산 솔버) |
| — | **재미 중간 점검** (§12 기준) — 미달 시 여기서 코어 재설계 | | | |
| M6 | 3레벨 업그레이드 + 판매(경로 재탐색 포함) + `TowerInspector` + 타겟정책 토글 + 터치 조작 정리 | 스탯 변화·환급 정확, 판매 시 경로 복원 | 2 | |
| M7 | `ShopService`(가중 3슬롯) + 리롤 공식 + 타워 해금 게이팅 + 골드 뭉치/기지 회복 + **화염 폭격 소모품** | 상점으로 타워 해금, 리롤 비용 증감·리셋, 소모품 골드 싱크 동작 | 2 | O (가중 롤) |
| M8 | `RelicService` 유물 6종 + `RelicChoiceModal`(시작 + 5웨이브마다) + 상점 유물 + 모디파이어 합성 + 미로 연동 훅(`toll_keeper`, `long_road`) | 유물별 체감 변화, 중복 없음, 경로 길이 연동 유물 정상 | 2.5 | O (id 디스패치 핸들러) |
| M9 | 배속/일시정지(배치 허용)/재시작 + **게임필 최소 세트** + AI 아트 스프라이트 교체 + 온보딩 툴팁 + 시드 런 1차 밸런싱 | 전체 루프 일관, 미로 짜는 맛이 나는가, 웨이브 40 클리어가 잘하면 가능 | 3 | |
| (리팩터) | 재미 확인 시 EventBus/RunState/서비스 분리 리팩터링 | 회귀 없음 | 2~3 | |

### 게임필 최소 세트 (M9, 명시)

- 적 피격: 1~2프레임 흰색 `modulate` 플래시
- 적 사망: `CPUParticles2D` 파편 버스트
- 기지 피해: 화면 흔들림(카메라 offset noise 0.15초)
- 미로 변경: 새 경로 라인이 쓸리듯 그려지는 트윈 + 짧은 "재배선" SFX
- SFX 2개: 타워 발사, 적 사망
- 처치 시 골드 숫자가 적 위치에서 TopBar로 튀는 트윈

### 일정

- **합계 ≈ 5~7주 (1인, 첫 Godot).** M1~M5(코어·약 2.5주) → 재미 점검 → M6~M9(로그라이크·재미·약 2.5주) → 리팩터링(선택).
- **오픈소스 포크 옵션**: M1~M4에 해당하는 완성 Godot TD/미로 TD 프로젝트가 GitHub에 다수 존재. 하나를 포크하거나 레퍼런스로 병행하면 1~2주 단축 가능. 목표가 "재미 검증"이므로 바닥부터 고집할 이유는 없음.

### codex 협업 모델

- Claude: 아키텍처, 데이터 스키마, 상태 머신, 시그널 배선, 통합, 리뷰.
- codex: 계약서(입출력/시그널/부작용 명시)와 함께 자기완결 알고리즘 위임 — 배치 합법성 검증(M1), attack_kind 해석·오브젝트 풀(M4), 웨이브 예산 솔버(M5), 가중 상점 롤(M7), 유물 효과 핸들러(M8).
- 각 codex 산출물은 통합 전 전용 테스트 씬에서 검증.

---

## 12. 재미 검증 기준

### 중간 점검 (M5 직후 — 코어만)

1. **미로 짜는 맛** — 타워 위치를 고민하게 되는가? "여기 세우면 적이 이만큼 더 돈다"가 눈에 보이고 즐거운가?
2. **경로 우회의 체감** — 적이 빙 도는 걸 보는 것 자체가 만족스러운가? (이게 없으면 차별점 실패 → 재설계.)
3. **봉쇄 불가 제약이 답답하지 않고 퍼즐로 느껴지는가?**

3개 중 2개 미달 → M6 진행 중단, 코어 메커닉 재설계 (엔진·스키마는 유지).

### 최종 판단 (M9 이후 — 로그라이크 포함)

4. **빌드 분기** — 시드 3개를 플레이했을 때 상점·유물이 실제로 다른 미로·전략을 강요하는가?
5. **의사결정 긴장** — 조기 시작 / 리롤 / 유물 선택 / 화염 폭격 사용에 "고민"이 생기는가?
6. **난이도 곡선** — 잘하면 웨이브 40 클리어, 못하면 웨이브 15~25 패배.
7. **"한 판 더"** — 게임오버·승리 후 재시작을 누르고 싶은가?

1~3 + 4~7 중 5개 이상 충족 시 메타 progression 설계로 진행. 미달 시 원인 분석 후 코어 루프 조정.

---

## 부록 A: 명시적으로 미룬 결정

| 항목 | 미룬 이유 | 다시 볼 시점 |
|---|---|---|
| 세계관·네이밍·아트 스타일 확정 | 프로토타입은 러프 아트로 재미만 검증 | 재미 검증 통과 후 |
| `pierce` / `dot` 타워, 데미지 타입 다양화 | 5종 + 물리 단일로 충분 | 타워 확장 시 |
| 멀티 스폰 / 멀티 기지 / 경로 분기 | 단일 스폰·기지로 공유 경로 = 구현 단순 | 맵 다양화 시 |
| 플로우 필드 경로탐색 | 단일 스폰·기지라 공유 A* 경로 1개로 충분 | 멀티 스폰 도입 시 |
| 적이 타워를 공격 / 벽 파괴 | 타워 = 무적 벽으로 단순화 | 코어 검증 후 밸런스 옵션 |
| 카메라 줌/팬 | 고정으로 충분 | 맵이 커질 때 |
| 사운드/BGM (SFX 2개 초과) | 재미 판단에 필수 아님 | 세로 슬라이스 단계 |
| 세이브/로드, 이어하기 | 한 판이 짧고 휘발적 | 메타 progression 도입 시 |
| 캐릭터 대량 콘텐츠, 퓨전 | 스코프 폭발의 주범 (예전 실패 원인) | 코어가 재밌다고 확인된 후 |
| PvP | 별개 프로젝트로 격하 | 이 프로젝트 로드맵 밖 |

## 부록 B: v1 대비 폐기/변경된 설계

- **고정 `Path2D` + `PathFollow2D` 폐기** → `AStarGrid2D` 동적 경로탐색.
- **`MapConfig`의 수동 경로 정의 폐기** → 스폰/기지 셀만 정의, 경로는 런타임 산출.
- 필드 크기 20×11 → 16×12 (미로 공간 확보).
- 타워 8종 → 5종, 유물 12종 → 6종.
- 무한 진행 → 40웨이브 + 최종 보스 + 선택적 무한.
- `EnemyContainer` 노드 부활 (Path2D 자식 제약 소멸).
- 판매 환급률 0.7 → 0.6 (유물 `미로공학`으로 1.0 달성 여지).
- 견적 3주 → 5~7주.
