# 미로 TD 코어 (M0–M5) 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 플레이어가 타워로 미로를 짜서 적을 우회시키는 코어 루프를, 웨이브 40 + 보스 + 상점 없는 최소 형태로 플레이 가능하게 만든다 (재미 중간 점검이 가능한 상태).

**Architecture:** Godot 4.3+ / GDScript. 적 경로는 `AStarGrid2D` 공유 경로 1개로 산출하고 타워 배치·판매마다 재탐색한다. 타워는 그리드 셀을 solid로 만들어 경로를 우회시키되, 스폰→기지 경로가 사라지는 배치는 거부한다. **M0–M5는 순진하게 구현한다**: EventBus/RunState autoload 없이 노드 간 직접 시그널·직접 참조를 쓰고, 상태는 `game.gd`가 보유한다. `MapService`/`EconomyService`/`WaveDirector`/`RelicService`는 `Game`의 자식 노드(plain `Node`)이며 `@onready`로 참조한다. 재미 중간 점검 통과 후 autoload 구조로 리팩터링한다.

**Tech Stack:** Godot 4.3+, GDScript, GUT 9.x (`addons/gut`, 유닛 테스트), `AStarGrid2D`, `TileMapLayer`, `CPUParticles2D`.

**Spec:** `docs/design/2026-09-01-roguelite-td-prototype-design.md` (v2.0)

## Global Constraints

- 엔진: **Godot 4.3 이상** (`TileMapLayer`, `AStarGrid2D.region` API 필요). C# 사용 금지 — GDScript만.
- **데이터 주도**: 밸런스 수치(HP, 골드, 사거리, 예산 공식 계수 등)는 전부 `res://data/**/*.tres`에 둔다. 시스템 스크립트(`scripts/services/`, `scripts/entities/`)에 매직 넘버 금지. 상수는 해당 `*Tuning` 리소스에서 읽는다.
- **필드**: 16×12 셀, 셀 64px. 스폰 셀 = 좌측 가장자리 세로 중앙, 기지 셀 = 우측 가장자리 세로 중앙.
- **적은 타워를 공격하지 않는다** (타워 = 무적 벽).
- **배치 합법성**: 배치 후 스폰→기지 경로가 반드시 존재해야 한다. 스폰/기지 셀, 이미 점유된 셀, 현재 적이 서 있는 셀에는 배치 불가.
- **진행**: 웨이브 1~40. 보스 웨이브 = 10, 20, 30 (미니보스), 40 (최종 보스). 웨이브 40 최종 보스 처치 = `RUN_WON`. 기지 HP 0 = `GAME_OVER`.
- **시드**: `WaveDirector`는 `run_seed + wave_index`로 시드해 **웨이브 구성**을 재현한다 (전투 결과 재현은 기대하지 않음).
- **결정론 주의**: 시뮬레이션은 `_physics_process(delta)` 기준 60fps. 배속은 `Engine.time_scale`.
- 스크립트는 한 파일 한 책임. 200줄을 넘으면 분리를 검토한다.
- 각 태스크 끝에서 커밋한다. 커밋 메시지 접두사: `feat:`, `test:`, `chore:`, `fix:`.
- 테스트 실행: `godot --headless -s addons/gut/gut_cmdln.gd -gdir=res://test/unit -gexit -glog=1`
  (`godot`가 PATH에 없으면 Godot 4 실행 파일 전체 경로로 대체.)

---

## File Structure

### 생성할 파일

**부트스트랩 / 설정**
- `project.godot` — 프로젝트 설정, 입력 맵, 오토로드 없음(M5까지), 렌더러 `gl_compatibility`(모바일)
- `addons/gut/` — GUT 9.x 애드온 (다운로드)
- `.gitattributes` — `* text=auto eol=lf`
- `test/unit/` — GUT 유닛 테스트 디렉터리

**데이터 클래스 (`scripts/data/`)** — 전부 `class_name … extends Resource`
- `enemy_type.gd`, `tower_level.gd`, `tower_type.gd`, `relic_type.gd`, `map_config.gd`
- `wave_tuning.gd`, `economy_tuning.gd`, `shop_tuning.gd`, `run_config.gd`

**데이터 인스턴스 (`data/`)**
- `data/enemies/{shambler,hound,swarmling,brute,miniboss,finalboss}.tres`
- `data/towers/{bolt_tower,frost_totem,mortar,arc_coil,war_horn}.tres`
- `data/tuning/{wave_tuning,economy_tuning}.tres`
- `data/maps/map_field.tres` (`MapConfig`)
- `data/run/default_run.tres` (`RunConfig`)

**유틸 (`scripts/util/`)**
- `grid_util.gd` — `class_name GridUtil`, 정적 함수만 (world↔cell 변환)

**서비스 (`scripts/services/`)**
- `map_service.gd` — `AStarGrid2D` 소유, solid 집합, `recompute_path()`, `is_placement_legal()`
- `economy_service.gd` — 골드, 기지 HP
- `wave_director.gd` — `build_wave(n)` + 스폰 루프
- `attack_resolver.gd` — `class_name AttackResolver`, 정적 함수 (attack_kind → 효과)
- `relic_service.gd` — 유물 풀 / `grant()` / `has_relic()` (효과는 M8, 여기선 스켈레톤)

**엔티티 (`scripts/entities/`, `scenes/entities/`)**
- `enemy.gd` + `enemy.tscn`
- `tower.gd` + `tower.tscn`
- `projectile.gd` + `projectile.tscn`
- `projectile_pool.gd` — `class_name ProjectilePool`

**맵 / 게임 (`scenes/`)**
- `scenes/map/map_field.tscn` — `TileMapLayer` + `SpawnMarker` + `BaseZone`
- `scenes/game/game.tscn` + `scripts/game.gd` — 상태 머신, 서비스 노드 조립
- `scenes/game/hud.tscn` + `scripts/hud/*.gd` — `TopBar`, `BuildMenu`, `EndScreen`

**컨테이너 노드** — 씬 내 plain `Node2D`: `EnemyContainer`, `TowerContainer`, `ProjectileContainer`

---

## Task 1: 프로젝트 부트스트랩 + GUT

**Files:**
- Create: `project.godot`, `.gitattributes`, `icon.svg`
- Create: `addons/gut/` (다운로드 압축 해제)
- Create: `test/unit/test_smoke.gd`

**Interfaces:**
- Consumes: (없음)
- Produces: 실행 가능한 Godot 프로젝트. GUT CLI 러너 동작. 이후 모든 태스크가 `test/unit/`에 테스트를 추가한다.

- [ ] **Step 1: Godot 4.3+로 프로젝트 생성**

Godot 에디터에서 `C:/Users/Pro/Documents/defense_project`를 프로젝트로 열거나(이미 `.gitignore`/`docs` 존재), "Import" 후 이 경로 지정. 렌더링 방식: **Compatibility**. 이러면 `project.godot`, `icon.svg`, `.godot/`가 생성된다.

- [ ] **Step 2: 프로젝트 설정 편집**

`project.godot`에 다음이 들어가도록 에디터 Project Settings에서 설정:
- Application > Config > Name: `defense_sample`
- Display > Window > Stretch > Mode: `canvas_items`, Aspect: `expand`
- Display > Window > Handheld > Orientation: `landscape`
- Display > Window > Size > Viewport Width: `1024`, Height: `768`

- [ ] **Step 3: `.gitattributes` 생성**

```
* text=auto eol=lf
*.png binary
*.svg binary
*.ttf binary
*.wav binary
*.ogg binary
```

- [ ] **Step 4: GUT 설치**

https://github.com/bitwes/Gut 의 최신 4.x 호환 릴리스(9.x) zip을 받아 `addons/gut/`로 압축 해제. Godot Project Settings > Plugins 에서 **GUT** 활성화.

- [ ] **Step 5: 스모크 테스트 작성**

`test/unit/test_smoke.gd`:

```gdscript
extends GutTest

func test_gut_runs():
	assert_true(true, "GUT 러너가 동작한다")

func test_godot_version_is_4_3_plus():
	var info := Engine.get_version_info()
	assert_true(info.major == 4 and info.minor >= 3, "Godot 4.3 이상이어야 함, 현재 %s" % Engine.get_version_info().string)
```

- [ ] **Step 6: 테스트 실행 → 통과 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gdir=res://test/unit -gexit -glog=1`
Expected: 2 passing, 0 failing.

- [ ] **Step 7: 커밋**

```bash
git add -A
git commit -m "chore: Godot 4.3 프로젝트 부트스트랩 + GUT"
```

---

## Task 2: 데이터 리소스 클래스

**Files:**
- Create: `scripts/data/enemy_type.gd`, `scripts/data/tower_level.gd`, `scripts/data/tower_type.gd`, `scripts/data/relic_type.gd`, `scripts/data/map_config.gd`, `scripts/data/wave_tuning.gd`, `scripts/data/economy_tuning.gd`, `scripts/data/shop_tuning.gd`, `scripts/data/run_config.gd`
- Test: `test/unit/test_data_classes.gd`

**Interfaces:**
- Consumes: (없음)
- Produces:
  - `EnemyType` — `id: StringName`, `display_name: String`, `sprite: Texture2D`, `max_hp: float`, `base_speed: float`, `armor: float`, `bounty: int`, `leak_damage: int`, `slow_immune_threshold: float` (기본 1.0), `point_cost: int`, `min_wave: int`
  - `TowerLevel` — `damage: float`, `attack_rate: float`, `range_px: float`, `upgrade_cost: int`, `splash_radius: float`, `slow_pct: float`, `slow_dur: float`, `chain_count: int`, `chain_falloff: float`, `buff_pct: float`, `buff_radius: float`
  - `TowerType` — `id: StringName`, `display_name: String`, `description: String`, `attack_kind: TowerType.Kind` (enum `SINGLE, SPLASH, SLOW, CHAIN, BUFF`), `projectile: PackedScene`, `sprite: Texture2D`, `levels: Array[TowerLevel]`, `sell_refund_rate: float` (기본 0.6), `starter_unlocked: bool`
  - `RelicType` — `id: StringName`, `display_name: String`, `description: String`, `icon: Texture2D`, `hooks: Array[StringName]`, `params: Dictionary`, `weight: int` (기본 10)
  - `MapConfig` — `grid_size: Vector2i` (기본 `Vector2i(16, 12)`), `cell_px: int` (기본 64), `spawn_cell: Vector2i`, `base_cell: Vector2i`, `blocked_cells: Array[Vector2i]`
  - `WaveTuning` — `budget_a: float`, `budget_b: float`, `budget_c: float`, `hp_scale_a: float`, `hp_scale_b: float`, `miniboss_hp_base: float`, `miniboss_hp_growth: float`, `finalboss_hp: float`, `endless_boss_growth: float`, `speed_per_wave: float`, `speed_cap: float`, `spawn_interval_start: float`, `spawn_interval_floor: float`, `spawn_interval_decay: float`, `total_waves: int` (기본 40), `boss_waves: Array[int]` (기본 `[10,20,30,40]`), `unlock_table: Array[Dictionary]`
  - `EconomyTuning` — `starting_gold: int`, `starting_base_hp: int`, `wave_reward_base: int`, `wave_reward_per_wave: int`, `early_call_bonus_per_pending: int`, `airstrike_base_cost: int`, `airstrike_cost_step: int`, `airstrike_radius: float`, `airstrike_damage: float`
  - `ShopTuning` — `slot_count: int` (기본 3), `reroll_base: int` (기본 10), `reroll_step: int` (기본 5), `content_weights: Dictionary`, `gold_cache_amount: int`, `base_heal_amount: int`, `relic_pick_every: int` (기본 5), `relic_pick_count: int` (기본 3), `starting_relic_pick_count: int` (기본 3)
  - `RunConfig` — `map_scene: PackedScene`, `map_config: MapConfig`, `tower_pool: Array[TowerType]`, `relic_pool: Array[RelicType]`, `enemy_set: Array[EnemyType]`, `wave_tuning: WaveTuning`, `economy_tuning: EconomyTuning`, `shop_tuning: ShopTuning`

- [ ] **Step 1: 실패하는 테스트 작성**

`test/unit/test_data_classes.gd`:

```gdscript
extends GutTest

func test_enemy_type_defaults():
	var e := EnemyType.new()
	assert_eq(e.slow_immune_threshold, 1.0, "감속 면역 임계 기본 1.0")
	e.max_hp = 100.0
	assert_eq(e.max_hp, 100.0)

func test_tower_type_kind_enum():
	assert_eq(TowerType.Kind.SINGLE, 0)
	assert_true(TowerType.Kind.has("BUFF") if TowerType.Kind is Dictionary else true)
	var t := TowerType.new()
	t.attack_kind = TowerType.Kind.SPLASH
	assert_eq(t.attack_kind, TowerType.Kind.SPLASH)
	assert_eq(t.sell_refund_rate, 0.6, "판매 환급 기본 0.6")

func test_tower_type_levels_typed_array():
	var t := TowerType.new()
	var lv := TowerLevel.new()
	lv.damage = 12.0
	t.levels.append(lv)
	assert_eq(t.levels[0].damage, 12.0)

func test_map_config_defaults():
	var m := MapConfig.new()
	assert_eq(m.grid_size, Vector2i(16, 12))
	assert_eq(m.cell_px, 64)

func test_wave_tuning_boss_waves_default():
	var w := WaveTuning.new()
	assert_eq(w.total_waves, 40)
	assert_eq(w.boss_waves, [10, 20, 30, 40])

func test_run_config_holds_pools():
	var r := RunConfig.new()
	r.tower_pool.append(TowerType.new())
	assert_eq(r.tower_pool.size(), 1)
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_data_classes.gd -gexit`
Expected: FAIL — `Identifier "EnemyType" not declared` 등.

- [ ] **Step 3: 리소스 클래스 구현**

`scripts/data/enemy_type.gd`:

```gdscript
class_name EnemyType
extends Resource

@export var id: StringName
@export var display_name: String
@export var sprite: Texture2D
@export var max_hp: float = 100.0
@export var base_speed: float = 60.0
@export var armor: float = 0.0
@export var bounty: int = 5
@export var leak_damage: int = 1
@export var slow_immune_threshold: float = 1.0
@export var point_cost: int = 10
@export var min_wave: int = 1
```

`scripts/data/tower_level.gd`:

```gdscript
class_name TowerLevel
extends Resource

@export var damage: float = 0.0
@export var attack_rate: float = 1.0
@export var range_px: float = 128.0
@export var upgrade_cost: int = 0
@export_group("kind-specific")
@export var splash_radius: float = 0.0
@export var slow_pct: float = 0.0
@export var slow_dur: float = 0.0
@export var chain_count: int = 0
@export var chain_falloff: float = 0.7
@export var buff_pct: float = 0.0
@export var buff_radius: float = 0.0
```

`scripts/data/tower_type.gd`:

```gdscript
class_name TowerType
extends Resource

enum Kind { SINGLE, SPLASH, SLOW, CHAIN, BUFF }

@export var id: StringName
@export var display_name: String
@export var description: String
@export var attack_kind: Kind = Kind.SINGLE
@export var projectile: PackedScene
@export var sprite: Texture2D
@export var levels: Array[TowerLevel] = []
@export var sell_refund_rate: float = 0.6
@export var starter_unlocked: bool = false
```

`scripts/data/relic_type.gd`:

```gdscript
class_name RelicType
extends Resource

@export var id: StringName
@export var display_name: String
@export var description: String
@export var icon: Texture2D
@export var hooks: Array[StringName] = []
@export var params: Dictionary = {}
@export var weight: int = 10
```

`scripts/data/map_config.gd`:

```gdscript
class_name MapConfig
extends Resource

@export var grid_size: Vector2i = Vector2i(16, 12)
@export var cell_px: int = 64
@export var spawn_cell: Vector2i = Vector2i(0, 6)
@export var base_cell: Vector2i = Vector2i(15, 6)
@export var blocked_cells: Array[Vector2i] = []
```

`scripts/data/wave_tuning.gd`:

```gdscript
class_name WaveTuning
extends Resource

@export_group("budget: a + b*N + c*N*N")
@export var budget_a: float = 40.0
@export var budget_b: float = 14.0
@export var budget_c: float = 0.5
@export_group("hp scale: 1 + a*(N-1) + b*(N-1)^2")
@export var hp_scale_a: float = 0.075
@export var hp_scale_b: float = 0.0035
@export_group("bosses")
@export var miniboss_hp_base: float = 1200.0
@export var miniboss_hp_growth: float = 0.6
@export var finalboss_hp: float = 12000.0
@export var endless_boss_growth: float = 0.08
@export_group("speed")
@export var speed_per_wave: float = 0.004
@export var speed_cap: float = 0.35
@export_group("cadence")
@export var spawn_interval_start: float = 0.9
@export var spawn_interval_floor: float = 0.45
@export var spawn_interval_decay: float = 0.01
@export_group("structure")
@export var total_waves: int = 40
@export var boss_waves: Array[int] = [10, 20, 30, 40]
## 각 항목: {"enemy_id": StringName, "min_wave": int, "weight_early": float, "weight_late": float}
@export var unlock_table: Array[Dictionary] = []
```

`scripts/data/economy_tuning.gd`:

```gdscript
class_name EconomyTuning
extends Resource

@export var starting_gold: int = 150
@export var starting_base_hp: int = 20
@export var wave_reward_base: int = 25
@export var wave_reward_per_wave: int = 5
@export var early_call_bonus_per_pending: int = 2
@export var airstrike_base_cost: int = 60
@export var airstrike_cost_step: int = 20
@export var airstrike_radius: float = 96.0
@export var airstrike_damage: float = 400.0
```

`scripts/data/shop_tuning.gd`:

```gdscript
class_name ShopTuning
extends Resource

@export var slot_count: int = 3
@export var reroll_base: int = 10
@export var reroll_step: int = 5
@export var content_weights: Dictionary = {
	"locked_tower": 40, "relic": 25, "gold_cache": 20, "base_heal": 15,
}
@export var gold_cache_amount: int = 60
@export var base_heal_amount: int = 3
@export var relic_pick_every: int = 5
@export var relic_pick_count: int = 3
@export var starting_relic_pick_count: int = 3
```

`scripts/data/run_config.gd`:

```gdscript
class_name RunConfig
extends Resource

@export var map_scene: PackedScene
@export var map_config: MapConfig
@export var tower_pool: Array[TowerType] = []
@export var relic_pool: Array[RelicType] = []
@export var enemy_set: Array[EnemyType] = []
@export var wave_tuning: WaveTuning
@export var economy_tuning: EconomyTuning
@export var shop_tuning: ShopTuning
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_data_classes.gd -gexit`
Expected: 6 passing.

- [ ] **Step 5: 커밋**

```bash
git add scripts/data test/unit/test_data_classes.gd
git commit -m "feat: 데이터 리소스 클래스 9종"
```

---

## Task 3: 그리드 좌표 유틸

**Files:**
- Create: `scripts/util/grid_util.gd`
- Test: `test/unit/test_grid_util.gd`

**Interfaces:**
- Consumes: (없음)
- Produces: `GridUtil` (정적 함수만)
  - `GridUtil.cell_to_world(cell: Vector2i, cell_px: int) -> Vector2` — 셀 중심의 월드 좌표
  - `GridUtil.world_to_cell(pos: Vector2, cell_px: int) -> Vector2i`
  - `GridUtil.in_bounds(cell: Vector2i, grid_size: Vector2i) -> bool`
  - `GridUtil.manhattan(a: Vector2i, b: Vector2i) -> int`

- [ ] **Step 1: 실패하는 테스트 작성**

`test/unit/test_grid_util.gd`:

```gdscript
extends GutTest

func test_cell_to_world_is_center():
	assert_eq(GridUtil.cell_to_world(Vector2i(0, 0), 64), Vector2(32, 32))
	assert_eq(GridUtil.cell_to_world(Vector2i(2, 3), 64), Vector2(160, 224))

func test_world_to_cell_floors():
	assert_eq(GridUtil.world_to_cell(Vector2(32, 32), 64), Vector2i(0, 0))
	assert_eq(GridUtil.world_to_cell(Vector2(127, 5), 64), Vector2i(1, 0))
	assert_eq(GridUtil.world_to_cell(Vector2(0, 0), 64), Vector2i(0, 0))

func test_round_trip():
	for c in [Vector2i(0, 0), Vector2i(15, 11), Vector2i(7, 4)]:
		assert_eq(GridUtil.world_to_cell(GridUtil.cell_to_world(c, 64), 64), c)

func test_in_bounds():
	var g := Vector2i(16, 12)
	assert_true(GridUtil.in_bounds(Vector2i(0, 0), g))
	assert_true(GridUtil.in_bounds(Vector2i(15, 11), g))
	assert_false(GridUtil.in_bounds(Vector2i(16, 0), g))
	assert_false(GridUtil.in_bounds(Vector2i(-1, 0), g))

func test_manhattan():
	assert_eq(GridUtil.manhattan(Vector2i(0, 0), Vector2i(3, 4)), 7)
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_grid_util.gd -gexit`
Expected: FAIL — `Identifier "GridUtil" not declared`.

- [ ] **Step 3: 구현**

`scripts/util/grid_util.gd`:

```gdscript
class_name GridUtil
extends RefCounted

static func cell_to_world(cell: Vector2i, cell_px: int) -> Vector2:
	return Vector2(cell.x * cell_px + cell_px / 2.0, cell.y * cell_px + cell_px / 2.0)

static func world_to_cell(pos: Vector2, cell_px: int) -> Vector2i:
	return Vector2i(floori(pos.x / cell_px), floori(pos.y / cell_px))

static func in_bounds(cell: Vector2i, grid_size: Vector2i) -> bool:
	return cell.x >= 0 and cell.y >= 0 and cell.x < grid_size.x and cell.y < grid_size.y

static func manhattan(a: Vector2i, b: Vector2i) -> int:
	return abs(a.x - b.x) + abs(a.y - b.y)
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_grid_util.gd -gexit`
Expected: 5 passing.

- [ ] **Step 5: 커밋**

```bash
git add scripts/util/grid_util.gd test/unit/test_grid_util.gd
git commit -m "feat: 그리드 좌표 변환 유틸"
```

---

## Task 4: MapService — 경로탐색 코어

**Files:**
- Create: `scripts/services/map_service.gd`
- Test: `test/unit/test_map_service_path.gd`

**Interfaces:**
- Consumes: `MapConfig` (Task 2), `GridUtil` (Task 3)
- Produces: `MapService` (extends `Node`)
  - `setup(config: MapConfig) -> void` — `AStarGrid2D` 초기화, 차단 셀 반영, 최초 경로 산출
  - `set_solid(cell: Vector2i, solid: bool) -> void` — 셀 통행 가능 여부 설정 (`blocked_cells`는 항상 solid, 덮어쓰기 불가)
  - `recompute_path() -> void` — `current_path` 갱신, `path_recomputed` 시그널 발신
  - `current_path: PackedVector2Array` — 셀 중심 월드 좌표 배열 (스폰→기지). 경로 없음이면 빈 배열 (배치 규칙상 정상 상황에선 발생 안 함)
  - `path_cells() -> Array[Vector2i]` — 현재 경로가 지나는 셀
  - `path_length_cells() -> int` — `current_path.size()`
  - `is_solid(cell: Vector2i) -> bool`
  - 시그널: `path_recomputed(path: PackedVector2Array)`

- [ ] **Step 1: 실패하는 테스트 작성**

`test/unit/test_map_service_path.gd`:

```gdscript
extends GutTest

var svc: MapService

func before_each():
	svc = MapService.new()
	add_child_autofree(svc)
	var cfg := MapConfig.new()
	cfg.grid_size = Vector2i(6, 5)
	cfg.cell_px = 64
	cfg.spawn_cell = Vector2i(0, 2)
	cfg.base_cell = Vector2i(5, 2)
	svc.setup(cfg)

func test_initial_path_is_straight_shortest():
	# 6칸 폭 → 스폰~기지 사이 경로 셀 수는 6 (스폰·기지 포함)
	assert_eq(svc.path_length_cells(), 6, "장애물 없으면 직선 최단 경로")

func test_path_routes_around_single_solid():
	svc.set_solid(Vector2i(3, 2), true)
	svc.recompute_path()
	var cells := svc.path_cells()
	assert_false(cells.has(Vector2i(3, 2)), "solid 셀을 지나지 않음")
	assert_gt(svc.path_length_cells(), 6, "우회로가 더 길다")

func test_path_recomputed_signal_fires():
	watch_signals(svc)
	svc.set_solid(Vector2i(3, 1), true)
	svc.recompute_path()
	assert_signal_emitted(svc, "path_recomputed")

func test_blocked_cells_are_always_solid():
	var cfg := MapConfig.new()
	cfg.grid_size = Vector2i(6, 5)
	cfg.spawn_cell = Vector2i(0, 2)
	cfg.base_cell = Vector2i(5, 2)
	cfg.blocked_cells = [Vector2i(2, 2)]
	var s2 := MapService.new()
	add_child_autofree(s2)
	s2.setup(cfg)
	assert_true(s2.is_solid(Vector2i(2, 2)))
	s2.set_solid(Vector2i(2, 2), false)  # 무시되어야 함
	assert_true(s2.is_solid(Vector2i(2, 2)), "blocked_cell은 해제 불가")

func test_no_path_returns_empty():
	# (3,y) 열 전체를 막아 완전 봉쇄
	for y in range(5):
		svc.set_solid(Vector2i(3, y), true)
	svc.recompute_path()
	assert_eq(svc.current_path.size(), 0, "경로 없으면 빈 배열")
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_map_service_path.gd -gexit`
Expected: FAIL — `Identifier "MapService" not declared`.

- [ ] **Step 3: 구현**

`scripts/services/map_service.gd`:

```gdscript
class_name MapService
extends Node

signal path_recomputed(path: PackedVector2Array)

var current_path: PackedVector2Array = PackedVector2Array()

var _astar: AStarGrid2D
var _cfg: MapConfig
var _blocked: Dictionary = {}      # Vector2i -> true (해제 불가)
var _tower_solid: Dictionary = {}  # Vector2i -> true (타워)

func setup(config: MapConfig) -> void:
	_cfg = config
	_blocked.clear()
	_tower_solid.clear()
	for c in config.blocked_cells:
		_blocked[c] = true
	_astar = AStarGrid2D.new()
	_astar.region = Rect2i(Vector2i.ZERO, config.grid_size)
	_astar.cell_size = Vector2(config.cell_px, config.cell_px)
	_astar.diagonal_mode = AStarGrid2D.DIAGONAL_MODE_NEVER
	_astar.default_compute_heuristic = AStarGrid2D.HEURISTIC_MANHATTAN
	_astar.update()
	for c in _blocked:
		_astar.set_point_solid(c, true)
	recompute_path()

func set_solid(cell: Vector2i, solid: bool) -> void:
	if _blocked.has(cell):
		return
	if solid:
		_tower_solid[cell] = true
	else:
		_tower_solid.erase(cell)
	_astar.set_point_solid(cell, solid)

func is_solid(cell: Vector2i) -> bool:
	return _blocked.has(cell) or _tower_solid.has(cell)

func recompute_path() -> void:
	var raw := _astar.get_id_path(_cfg.spawn_cell, _cfg.base_cell)
	if raw.is_empty():
		current_path = PackedVector2Array()
	else:
		var pts := PackedVector2Array()
		for cell in raw:
			pts.append(GridUtil.cell_to_world(cell, _cfg.cell_px))
		current_path = pts
	path_recomputed.emit(current_path)

func path_cells() -> Array:
	var out: Array = []
	for p in current_path:
		out.append(GridUtil.world_to_cell(p, _cfg.cell_px))
	return out

func path_length_cells() -> int:
	return current_path.size()
```

> **AStarGrid2D 주의**: `get_id_path`는 시작·끝 점이 solid면 빈 배열을 준다. 스폰/기지 셀은 절대 solid로 만들지 말 것 (Task 5의 검증이 이를 보장).

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_map_service_path.gd -gexit`
Expected: 5 passing.

- [ ] **Step 5: 커밋**

```bash
git add scripts/services/map_service.gd test/unit/test_map_service_path.gd
git commit -m "feat: MapService 경로탐색 코어 (AStarGrid2D)"
```

---

## Task 5: MapService — 배치 합법성 검증 (차별점의 심장) — codex 위임 대상

**Files:**
- Modify: `scripts/services/map_service.gd`
- Test: `test/unit/test_map_service_placement.gd`

**codex 계약서**
- 입력: `cell: Vector2i`, 현재 `MapService` 상태 (`_cfg`, `_blocked`, `_tower_solid`, `_astar`, 적 점유 셀 집합은 인자로 받음).
- 출력: `is_placement_legal(cell, occupied_by_enemy: Array) -> bool`, `placement_block_reason(cell, occupied_by_enemy: Array) -> StringName`.
- 부작용 금지: 검증 중 `_astar`를 임시 변경하더라도 반드시 원상 복구. `current_path`·시그널 건드리지 않음.
- 성능: 배치 시도당 1회 호출. `get_id_path` 1~2회 이내.

**Interfaces:**
- Consumes: `MapService` 내부 상태 (Task 4)
- Produces:
  - `is_placement_legal(cell: Vector2i, occupied_by_enemy: Array) -> bool`
  - `placement_block_reason(cell: Vector2i, occupied_by_enemy: Array) -> StringName` — `&"ok"`, `&"out_of_bounds"`, `&"spawn_or_base"`, `&"blocked"`, `&"occupied"`, `&"enemy_present"`, `&"would_seal_exit"` 중 하나

- [ ] **Step 1: 실패하는 테스트 작성**

`test/unit/test_map_service_placement.gd`:

```gdscript
extends GutTest

var svc: MapService

func before_each():
	svc = MapService.new()
	add_child_autofree(svc)
	var cfg := MapConfig.new()
	cfg.grid_size = Vector2i(6, 5)
	cfg.spawn_cell = Vector2i(0, 2)
	cfg.base_cell = Vector2i(5, 2)
	cfg.blocked_cells = [Vector2i(1, 0)]
	svc.setup(cfg)

func test_normal_cell_is_legal():
	assert_true(svc.is_placement_legal(Vector2i(2, 1), []))
	assert_eq(svc.placement_block_reason(Vector2i(2, 1), []), &"ok")

func test_spawn_and_base_illegal():
	assert_false(svc.is_placement_legal(Vector2i(0, 2), []))
	assert_eq(svc.placement_block_reason(Vector2i(0, 2), []), &"spawn_or_base")
	assert_eq(svc.placement_block_reason(Vector2i(5, 2), []), &"spawn_or_base")

func test_out_of_bounds_illegal():
	assert_eq(svc.placement_block_reason(Vector2i(6, 2), []), &"out_of_bounds")

func test_blocked_cell_illegal():
	assert_eq(svc.placement_block_reason(Vector2i(1, 0), []), &"blocked")

func test_occupied_by_tower_illegal():
	svc.set_solid(Vector2i(2, 2), true)
	assert_eq(svc.placement_block_reason(Vector2i(2, 2), []), &"occupied")

func test_enemy_present_illegal():
	assert_eq(svc.placement_block_reason(Vector2i(2, 2), [Vector2i(2, 2)]), &"enemy_present")

func test_would_seal_exit_illegal():
	# (2,y) 열의 y=0,1,3,4를 막고 마지막 (2,2)를 시도하면 완전 봉쇄
	svc.set_solid(Vector2i(2, 0), true)
	svc.set_solid(Vector2i(2, 1), true)
	svc.set_solid(Vector2i(2, 3), true)
	svc.set_solid(Vector2i(2, 4), true)
	assert_true(svc.is_placement_legal(Vector2i(2, 2), []) == false)
	assert_eq(svc.placement_block_reason(Vector2i(2, 2), []), &"would_seal_exit")

func test_legality_check_does_not_mutate_state():
	var before := svc.path_length_cells()
	svc.is_placement_legal(Vector2i(2, 1), [])
	svc.recompute_path()
	assert_eq(svc.path_length_cells(), before, "검증이 경로를 바꾸지 않음")
	assert_false(svc.is_solid(Vector2i(2, 1)), "임시 solid가 복구됨")
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_map_service_placement.gd -gexit`
Expected: FAIL — `Invalid call ... is_placement_legal`.

- [ ] **Step 3: `map_service.gd`에 추가**

```gdscript
func is_placement_legal(cell: Vector2i, occupied_by_enemy: Array) -> bool:
	return placement_block_reason(cell, occupied_by_enemy) == &"ok"

func placement_block_reason(cell: Vector2i, occupied_by_enemy: Array) -> StringName:
	if not GridUtil.in_bounds(cell, _cfg.grid_size):
		return &"out_of_bounds"
	if cell == _cfg.spawn_cell or cell == _cfg.base_cell:
		return &"spawn_or_base"
	if _blocked.has(cell):
		return &"blocked"
	if _tower_solid.has(cell):
		return &"occupied"
	if occupied_by_enemy.has(cell):
		return &"enemy_present"
	# 임시로 solid 설정 → 경로 존재 확인 → 복구
	_astar.set_point_solid(cell, true)
	var path := _astar.get_id_path(_cfg.spawn_cell, _cfg.base_cell)
	_astar.set_point_solid(cell, false)
	if path.is_empty():
		return &"would_seal_exit"
	return &"ok"
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_map_service_placement.gd -gexit`
Expected: 8 passing.

- [ ] **Step 5: 전체 스위트 회귀 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gdir=res://test/unit -gexit -glog=1`
Expected: 이전 태스크 테스트 전부 여전히 통과.

- [ ] **Step 6: 커밋**

```bash
git add scripts/services/map_service.gd test/unit/test_map_service_placement.gd
git commit -m "feat: 타워 배치 합법성 검증 (출구 봉쇄 불가)"
```

---

## Task 6: 적 엔티티 — 공유 경로 이동 + 체력

**Files:**
- Create: `scenes/entities/enemy.tscn`, `scripts/entities/enemy.gd`
- Test: `test/unit/test_enemy_movement.gd`

**Interfaces:**
- Consumes: `EnemyType` (Task 2)
- Produces: `Enemy` (extends `Node2D`, `enemy.tscn` 루트)
  - `setup(type: EnemyType, path: PackedVector2Array) -> void` — 스탯 적용, `global_position`을 `path[0]`으로, 경로 인덱스 0
  - `take_damage(amount: float) -> void` — `max(1.0, amount - armor)` 적용. 0 이하 → `died` 발신 후 `queue_free()`
  - `hp: float`, `max_hp: float`, `armor: float`, `bounty: int`, `path_index: int` (읽기용)
  - `current_cell(cell_px: int) -> Vector2i`
  - `effective_speed() -> float` — 이후 Task 7에서 감속 반영. 지금은 `base_speed`
  - 시그널: `died(enemy: Enemy)`, `reached_base(enemy: Enemy, leak_damage: int)`
  - `_physics_process(delta)`에서 `path[path_index]`로 `effective_speed()*delta` 이동. 도달 시 `path_index += 1`. 끝 도달 시 `reached_base` 발신 후 `queue_free()`

- [ ] **Step 1: 실패하는 테스트 작성**

`test/unit/test_enemy_movement.gd`:

```gdscript
extends GutTest

const EnemyScene := preload("res://scenes/entities/enemy.tscn")

func _make_type() -> EnemyType:
	var t := EnemyType.new()
	t.max_hp = 100.0
	t.base_speed = 64.0
	t.armor = 10.0
	t.bounty = 7
	t.leak_damage = 2
	return t

func test_setup_places_at_path_start():
	var e := EnemyScene.instantiate()
	add_child_autofree(e)
	e.setup(_make_type(), PackedVector2Array([Vector2(32, 32), Vector2(160, 32)]))
	assert_eq(e.global_position, Vector2(32, 32))
	assert_eq(e.hp, 100.0)

func test_moves_toward_next_node():
	var e := EnemyScene.instantiate()
	add_child_autofree(e)
	e.setup(_make_type(), PackedVector2Array([Vector2(0, 0), Vector2(640, 0)]))
	e._physics_process(1.0)  # 64 px 이동
	assert_almost_eq(e.global_position.x, 64.0, 0.01)

func test_advances_path_index_on_arrival():
	var e := EnemyScene.instantiate()
	add_child_autofree(e)
	e.setup(_make_type(), PackedVector2Array([Vector2(0, 0), Vector2(32, 0), Vector2(200, 0)]))
	e._physics_process(1.0)  # 64px → (32,0) 통과, 인덱스 1로
	assert_eq(e.path_index, 1)

func test_reached_base_signal_and_freed():
	var e := EnemyScene.instantiate()
	add_child_autofree(e)
	watch_signals(e)
	e.setup(_make_type(), PackedVector2Array([Vector2(0, 0), Vector2(10, 0)]))
	e._physics_process(1.0)
	assert_signal_emitted_with_parameters(e, "reached_base", [e, 2])

func test_take_damage_applies_armor():
	var e := EnemyScene.instantiate()
	add_child_autofree(e)
	e.setup(_make_type(), PackedVector2Array([Vector2(0, 0), Vector2(9999, 0)]))
	e.take_damage(30.0)  # 30 - 10 armor = 20
	assert_eq(e.hp, 80.0)
	e.take_damage(5.0)   # max(1, 5-10) = 1
	assert_eq(e.hp, 79.0)

func test_death_emits_died():
	var e := EnemyScene.instantiate()
	add_child_autofree(e)
	watch_signals(e)
	e.setup(_make_type(), PackedVector2Array([Vector2(0, 0), Vector2(9999, 0)]))
	e.take_damage(9999.0)
	assert_signal_emitted(e, "died")
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_enemy_movement.gd -gexit`
Expected: FAIL — `enemy.tscn` 없음.

- [ ] **Step 3: `enemy.gd` 구현**

`scripts/entities/enemy.gd`:

```gdscript
class_name Enemy
extends Node2D

signal died(enemy: Enemy)
signal reached_base(enemy: Enemy, leak_damage: int)

const ARRIVE_EPSILON := 2.0

var hp: float = 1.0
var max_hp: float = 1.0
var armor: float = 0.0
var bounty: int = 0
var slow_immune_threshold: float = 1.0
var path_index: int = 0

var _base_speed: float = 60.0
var _leak_damage: int = 1
var _path: PackedVector2Array = PackedVector2Array()
var _slow_mult: float = 1.0
var _slow_until_ms: int = 0

func setup(type: EnemyType, path: PackedVector2Array) -> void:
	max_hp = type.max_hp
	hp = type.max_hp
	armor = type.armor
	bounty = type.bounty
	slow_immune_threshold = type.slow_immune_threshold
	_base_speed = type.base_speed
	_leak_damage = type.leak_damage
	_path = path
	path_index = 0
	if _path.size() > 0:
		global_position = _path[0]

func effective_speed() -> float:
	return _base_speed * _slow_mult

func current_cell(cell_px: int) -> Vector2i:
	return GridUtil.world_to_cell(global_position, cell_px)

func take_damage(amount: float) -> void:
	hp -= maxf(1.0, amount - armor)
	if hp <= 0.0:
		died.emit(self)
		queue_free()

func _physics_process(delta: float) -> void:
	if _path.size() < 2:
		return
	var target_idx := path_index + 1
	if target_idx >= _path.size():
		reached_base.emit(self, _leak_damage)
		queue_free()
		return
	var target := _path[target_idx]
	var step := effective_speed() * delta
	var to_target := target - global_position
	if to_target.length() <= step + ARRIVE_EPSILON:
		global_position = target
		path_index = target_idx
	else:
		global_position += to_target.normalized() * step
```

- [ ] **Step 4: `enemy.tscn` 생성**

에디터에서: 루트 `Node2D`(스크립트 `enemy.gd`, 이름 `Enemy`) → 자식 `Sprite2D`(이름 `Sprite`, 텍스처는 임시로 `icon.svg`, scale 0.25) → 자식 `Area2D`(이름 `Hurtbox`, 콜리전 레이어 2, 마스크 0) + `CollisionShape2D`(원, radius 16). 저장 `res://scenes/entities/enemy.tscn`.

- [ ] **Step 5: 테스트 실행 → 통과 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_enemy_movement.gd -gexit`
Expected: 6 passing.

- [ ] **Step 6: 커밋**

```bash
git add scenes/entities/enemy.tscn scripts/entities/enemy.gd test/unit/test_enemy_movement.gd
git commit -m "feat: 적 엔티티 - 공유 경로 이동 + 체력"
```

---

## Task 7: 적 — 경로 재탐색 재스냅 + 감속

**Files:**
- Modify: `scripts/entities/enemy.gd`
- Test: `test/unit/test_enemy_repath_slow.gd`

**Interfaces:**
- Consumes: `Enemy` (Task 6)
- Produces:
  - `on_path_recomputed(new_path: PackedVector2Array) -> void` — `new_path`에서 자신의 현재 위치 기준 "가장 가까운 노드"를 찾아 `path_index`로 삼고 `_path` 교체. 동률이면 인덱스가 큰 쪽(더 진행한 쪽) 선택.
  - `apply_slow(pct: float, duration_ms: int) -> void` — `pct`가 `slow_immune_threshold` 이상이면 `pct`를 `slow_immune_threshold` 바로 아래로 클램프. 기존 감속보다 강할 때만 갱신(`_slow_mult` 최소값). `duration_ms` 후 자동 해제.
  - `_slow_mult` 만료 처리를 `_physics_process` 앞에서 수행

- [ ] **Step 1: 실패하는 테스트 작성**

`test/unit/test_enemy_repath_slow.gd`:

```gdscript
extends GutTest

const EnemyScene := preload("res://scenes/entities/enemy.tscn")

func _type(speed := 100.0, immune := 1.0) -> EnemyType:
	var t := EnemyType.new()
	t.max_hp = 100.0
	t.base_speed = speed
	t.slow_immune_threshold = immune
	return t

func test_repath_snaps_to_nearest_forward_node():
	var e := EnemyScene.instantiate()
	add_child_autofree(e)
	e.setup(_type(), PackedVector2Array([Vector2(0,0), Vector2(100,0), Vector2(200,0)]))
	e.global_position = Vector2(140, 0)  # 두 번째 구간 중간
	var new_path := PackedVector2Array([Vector2(0,0), Vector2(100,0), Vector2(150,0), Vector2(150,80)])
	e.on_path_recomputed(new_path)
	assert_eq(e.path_index, 2, "가장 가까운 노드(150,0)의 인덱스")

func test_apply_slow_reduces_effective_speed():
	var e := EnemyScene.instantiate()
	add_child_autofree(e)
	e.setup(_type(100.0), PackedVector2Array([Vector2(0,0), Vector2(9999,0)]))
	e.apply_slow(0.4, 2000)
	assert_almost_eq(e.effective_speed(), 60.0, 0.01)

func test_slow_immunity_clamps():
	var e := EnemyScene.instantiate()
	add_child_autofree(e)
	e.setup(_type(100.0, 0.5), PackedVector2Array([Vector2(0,0), Vector2(9999,0)]))
	e.apply_slow(0.9, 2000)  # 면역 임계 0.5 → 최대 감속 0.5 미만
	assert_gt(e.effective_speed(), 50.0, "감속 50%에서 면역이라 그 이상 안 느려짐")

func test_stronger_slow_overrides_weaker():
	var e := EnemyScene.instantiate()
	add_child_autofree(e)
	e.setup(_type(100.0), PackedVector2Array([Vector2(0,0), Vector2(9999,0)]))
	e.apply_slow(0.2, 2000)
	e.apply_slow(0.5, 2000)
	assert_almost_eq(e.effective_speed(), 50.0, 0.01)
	e.apply_slow(0.1, 2000)  # 약한 감속은 무시
	assert_almost_eq(e.effective_speed(), 50.0, 0.01)
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_enemy_repath_slow.gd -gexit`
Expected: FAIL — `on_path_recomputed` 없음.

- [ ] **Step 3: `enemy.gd` 수정**

`_physics_process` 첫 줄에 감속 만료 처리 추가:

```gdscript
func _physics_process(delta: float) -> void:
	if _slow_until_ms > 0 and Time.get_ticks_msec() >= _slow_until_ms:
		_slow_mult = 1.0
		_slow_until_ms = 0
	# ... 기존 이동 로직 ...
```

메서드 추가:

```gdscript
func on_path_recomputed(new_path: PackedVector2Array) -> void:
	if new_path.size() < 2:
		return
	var best_idx := 0
	var best_d := INF
	for i in range(new_path.size()):
		var d := global_position.distance_squared_to(new_path[i])
		if d <= best_d:
			best_d = d
			best_idx = i
	_path = new_path
	path_index = mini(best_idx, new_path.size() - 1)

func apply_slow(pct: float, duration_ms: int) -> void:
	var capped := minf(pct, maxf(0.0, slow_immune_threshold - 0.001))
	var new_mult := 1.0 - capped
	if new_mult < _slow_mult:
		_slow_mult = new_mult
		_slow_until_ms = Time.get_ticks_msec() + duration_ms
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_enemy_repath_slow.gd -gexit`
Expected: 4 passing.

- [ ] **Step 5: 커밋**

```bash
git add scripts/entities/enemy.gd test/unit/test_enemy_repath_slow.gd
git commit -m "feat: 적 경로 재스냅 + 감속(면역 임계 포함)"
```

---

## Task 8: 적 데이터 인스턴스 (.tres ×6)

**Files:**
- Create: `data/enemies/shambler.tres`, `hound.tres`, `swarmling.tres`, `brute.tres`, `miniboss.tres`, `finalboss.tres`
- Test: `test/unit/test_enemy_data.gd`

**Interfaces:**
- Consumes: `EnemyType` (Task 2)
- Produces: 스펙 §5 표의 수치를 담은 리소스 6종. `id`는 파일명과 동일.

- [ ] **Step 1: 실패하는 테스트 작성**

`test/unit/test_enemy_data.gd`:

```gdscript
extends GutTest

func _load(name: String) -> EnemyType:
	return load("res://data/enemies/%s.tres" % name) as EnemyType

func test_all_enemy_files_load():
	for n in ["shambler", "hound", "swarmling", "brute", "miniboss", "finalboss"]:
		var e := _load(n)
		assert_not_null(e, "%s.tres 로드" % n)
		assert_eq(String(e.id), n, "id는 파일명과 동일")

func test_shambler_baseline_stats():
	var e := _load("shambler")
	assert_eq(e.max_hp, 100.0)
	assert_eq(e.base_speed, 60.0)
	assert_eq(e.armor, 0.0)
	assert_eq(e.bounty, 6)
	assert_eq(e.leak_damage, 1)

func test_brute_has_armor():
	var e := _load("brute")
	assert_eq(e.max_hp, 480.0)
	assert_eq(e.armor, 6.0)
	assert_eq(e.bounty, 20)

func test_finalboss_slow_immune():
	var e := _load("finalboss")
	assert_eq(e.slow_immune_threshold, 0.5)
	assert_eq(e.leak_damage, 20)

func test_unlock_waves():
	assert_eq(_load("shambler").min_wave, 1)
	assert_eq(_load("hound").min_wave, 3)
	assert_eq(_load("swarmling").min_wave, 5)
	assert_eq(_load("brute").min_wave, 8)
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_enemy_data.gd -gexit`
Expected: FAIL — 파일 로드 실패.

- [ ] **Step 3: `.tres` 생성**

에디터 FileSystem 패널에서 `data/enemies/` 우클릭 → New Resource → `EnemyType`. 각 파일을 아래 값으로 저장 (인스펙터 편집):

| 파일 | id | max_hp | base_speed | armor | bounty | leak_damage | slow_immune_threshold | point_cost | min_wave |
|---|---|---|---|---|---|---|---|---|---|
| shambler | `shambler` | 100 | 60 | 0 | 6 | 1 | 1.0 | 10 | 1 |
| hound | `hound` | 55 | 130 | 0 | 8 | 1 | 1.0 | 12 | 3 |
| swarmling | `swarmling` | 30 | 85 | 0 | 3 | 1 | 1.0 | 5 | 5 |
| brute | `brute` | 480 | 38 | 6 | 20 | 3 | 1.0 | 45 | 8 |
| miniboss | `miniboss` | 1200 | 34 | 8 | 80 | 6 | 1.0 | 200 | 10 |
| finalboss | `finalboss` | 12000 | 30 | 12 | 300 | 20 | 0.5 | 800 | 40 |

> `max_hp`는 기준값이다. 웨이브별 스케일링은 Task 15의 `WaveDirector.build_wave`가 스폰 시점에 곱한다 (보스는 별도 곡선).

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_enemy_data.gd -gexit`
Expected: 5 passing.

- [ ] **Step 5: 커밋**

```bash
git add data/enemies test/unit/test_enemy_data.gd
git commit -m "feat: 적 데이터 인스턴스 6종"
```

---

## Task 9: EconomyService

**Files:**
- Create: `scripts/services/economy_service.gd`
- Test: `test/unit/test_economy_service.gd`

**Interfaces:**
- Consumes: `EconomyTuning` (Task 2)
- Produces: `EconomyService` (extends `Node`)
  - `setup(tuning: EconomyTuning) -> void` — `gold = starting_gold`, `base_hp = starting_base_hp`, `max_base_hp = starting_base_hp`, `airstrike_uses = 0`
  - `gold: int`, `base_hp: int`, `max_base_hp: int` (읽기용)
  - `add_gold(amount: int) -> void` — `gold_changed` 발신
  - `can_afford(cost: int) -> bool`
  - `try_spend(cost: int) -> bool` — 충분하면 차감·`true`·`gold_changed`, 아니면 `false`
  - `damage_base(amount: int) -> void` — `base_hp` 감소(0 하한), `base_hp_changed` 발신. 0 도달 시 `base_destroyed` 발신(1회)
  - `heal_base(amount: int) -> void` — `max_base_hp` 상한
  - `set_max_base_hp(value: int) -> void` — 유물용. `base_hp`도 상한 클램프
  - `airstrike_cost() -> int` — `airstrike_base_cost + airstrike_cost_step * airstrike_uses`
  - `consume_airstrike() -> void` — `airstrike_uses += 1`
  - 시그널: `gold_changed(gold: int)`, `base_hp_changed(hp: int, max_hp: int)`, `base_destroyed()`

- [ ] **Step 1: 실패하는 테스트 작성**

`test/unit/test_economy_service.gd`:

```gdscript
extends GutTest

var eco: EconomyService

func before_each():
	eco = EconomyService.new()
	add_child_autofree(eco)
	var t := EconomyTuning.new()
	t.starting_gold = 150
	t.starting_base_hp = 20
	t.airstrike_base_cost = 60
	t.airstrike_cost_step = 20
	eco.setup(t)

func test_initial_state():
	assert_eq(eco.gold, 150)
	assert_eq(eco.base_hp, 20)
	assert_eq(eco.max_base_hp, 20)

func test_try_spend_success_and_fail():
	assert_true(eco.try_spend(100))
	assert_eq(eco.gold, 50)
	assert_false(eco.try_spend(100))
	assert_eq(eco.gold, 50)

func test_add_gold_emits():
	watch_signals(eco)
	eco.add_gold(30)
	assert_eq(eco.gold, 180)
	assert_signal_emitted_with_parameters(eco, "gold_changed", [180])

func test_damage_base_floors_at_zero_and_emits_destroyed_once():
	watch_signals(eco)
	eco.damage_base(5)
	assert_eq(eco.base_hp, 15)
	eco.damage_base(999)
	assert_eq(eco.base_hp, 0)
	eco.damage_base(1)
	assert_signal_emit_count(eco, "base_destroyed", 1)

func test_heal_capped_at_max():
	eco.damage_base(10)  # 10
	eco.heal_base(99)
	assert_eq(eco.base_hp, 20)

func test_set_max_base_hp_clamps_current():
	eco.set_max_base_hp(15)
	assert_eq(eco.max_base_hp, 15)
	assert_eq(eco.base_hp, 15)

func test_airstrike_cost_scales():
	assert_eq(eco.airstrike_cost(), 60)
	eco.consume_airstrike()
	assert_eq(eco.airstrike_cost(), 80)
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_economy_service.gd -gexit`
Expected: FAIL — `Identifier "EconomyService" not declared`.

- [ ] **Step 3: 구현**

`scripts/services/economy_service.gd`:

```gdscript
class_name EconomyService
extends Node

signal gold_changed(gold: int)
signal base_hp_changed(hp: int, max_hp: int)
signal base_destroyed()

var gold: int = 0
var base_hp: int = 0
var max_base_hp: int = 0

var _tuning: EconomyTuning
var _airstrike_uses: int = 0
var _destroyed_emitted: bool = false

func setup(tuning: EconomyTuning) -> void:
	_tuning = tuning
	gold = tuning.starting_gold
	base_hp = tuning.starting_base_hp
	max_base_hp = tuning.starting_base_hp
	_airstrike_uses = 0
	_destroyed_emitted = false

func add_gold(amount: int) -> void:
	gold += amount
	gold_changed.emit(gold)

func can_afford(cost: int) -> bool:
	return gold >= cost

func try_spend(cost: int) -> bool:
	if gold < cost:
		return false
	gold -= cost
	gold_changed.emit(gold)
	return true

func damage_base(amount: int) -> void:
	base_hp = maxi(0, base_hp - amount)
	base_hp_changed.emit(base_hp, max_base_hp)
	if base_hp == 0 and not _destroyed_emitted:
		_destroyed_emitted = true
		base_destroyed.emit()

func heal_base(amount: int) -> void:
	base_hp = mini(max_base_hp, base_hp + amount)
	base_hp_changed.emit(base_hp, max_base_hp)

func set_max_base_hp(value: int) -> void:
	max_base_hp = maxi(1, value)
	base_hp = mini(base_hp, max_base_hp)
	base_hp_changed.emit(base_hp, max_base_hp)

func airstrike_cost() -> int:
	return _tuning.airstrike_base_cost + _tuning.airstrike_cost_step * _airstrike_uses

func consume_airstrike() -> void:
	_airstrike_uses += 1
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_economy_service.gd -gexit`
Expected: 7 passing.

- [ ] **Step 5: 커밋**

```bash
git add scripts/services/economy_service.gd test/unit/test_economy_service.gd
git commit -m "feat: EconomyService (골드/기지 HP/화염폭격 비용)"
```

---

## Task 10: WaveDirector — 고정 리스트 스포너 (M2 최소)

**Files:**
- Create: `scripts/services/wave_director.gd`
- Test: `test/unit/test_wave_director_spawn.gd`

**Interfaces:**
- Consumes: `Enemy` scene (Task 6), `EnemyType` (Task 2), `MapService.current_path` (Task 4)
- Produces: `WaveDirector` (extends `Node`)
  - `configure(enemy_scene: PackedScene, enemy_container: Node, get_path: Callable) -> void` — `get_path`는 `func() -> PackedVector2Array` (항상 최신 공유 경로 반환)
  - `run_spawn_list(entries: Array[Dictionary]) -> void` — 각 항목 `{"type": EnemyType, "delay": float, "hp_mult": float}`. `delay`(초) 뒤에 적 1기 스폰. 스폰 시 `type` 복제 후 `max_hp *= hp_mult` 적용해 `Enemy.setup`. `enemy_container`에 추가. `died`/`reached_base` 시그널을 자기 핸들러에 연결.
  - `alive_count: int` (읽기용)
  - `is_wave_running() -> bool` — 스폰 진행 중이거나 생존 적 있음
  - 시그널: `enemy_spawned(enemy: Enemy)`, `enemy_killed(enemy: Enemy, bounty: int)`, `enemy_leaked(enemy: Enemy, leak_damage: int)`, `wave_finished()` (스폰 완료 + `alive_count == 0`)

- [ ] **Step 1: 실패하는 테스트 작성**

`test/unit/test_wave_director_spawn.gd`:

```gdscript
extends GutTest

const EnemyScene := preload("res://scenes/entities/enemy.tscn")

var wd: WaveDirector
var container: Node2D
var _path := PackedVector2Array([Vector2(0, 0), Vector2(64, 0)])

func _type() -> EnemyType:
	var t := EnemyType.new()
	t.max_hp = 100.0
	t.base_speed = 32.0
	t.bounty = 5
	return t

func before_each():
	container = Node2D.new()
	add_child_autofree(container)
	wd = WaveDirector.new()
	add_child_autofree(wd)
	wd.configure(EnemyScene, container, func(): return _path)

func test_spawns_after_delay():
	wd.run_spawn_list([{"type": _type(), "delay": 0.0, "hp_mult": 1.0}] as Array[Dictionary])
	await wait_frames(3)
	assert_eq(wd.alive_count, 1)
	assert_eq(container.get_child_count(), 1)

func test_hp_mult_applied():
	wd.run_spawn_list([{"type": _type(), "delay": 0.0, "hp_mult": 2.5}] as Array[Dictionary])
	await wait_frames(3)
	var e := container.get_child(0) as Enemy
	assert_eq(e.max_hp, 250.0)
	assert_eq(e.hp, 250.0)

func test_enemy_killed_signal_carries_bounty():
	watch_signals(wd)
	wd.run_spawn_list([{"type": _type(), "delay": 0.0, "hp_mult": 1.0}] as Array[Dictionary])
	await wait_frames(3)
	var e := container.get_child(0) as Enemy
	e.take_damage(9999.0)
	assert_signal_emitted_with_parameters(wd, "enemy_killed", [e, 5])

func test_wave_finished_after_all_dead():
	watch_signals(wd)
	wd.run_spawn_list([{"type": _type(), "delay": 0.0, "hp_mult": 1.0}] as Array[Dictionary])
	await wait_frames(3)
	(container.get_child(0) as Enemy).take_damage(9999.0)
	await wait_frames(2)
	assert_signal_emitted(wd, "wave_finished")
	assert_eq(wd.alive_count, 0)
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_wave_director_spawn.gd -gexit`
Expected: FAIL — `Identifier "WaveDirector" not declared`.

- [ ] **Step 3: 구현**

`scripts/services/wave_director.gd`:

```gdscript
class_name WaveDirector
extends Node

signal enemy_spawned(enemy: Enemy)
signal enemy_killed(enemy: Enemy, bounty: int)
signal enemy_leaked(enemy: Enemy, leak_damage: int)
signal wave_finished()

var alive_count: int = 0

var _enemy_scene: PackedScene
var _container: Node
var _get_path: Callable
var _spawning: bool = false

func configure(enemy_scene: PackedScene, enemy_container: Node, get_path: Callable) -> void:
	_enemy_scene = enemy_scene
	_container = enemy_container
	_get_path = get_path

func is_wave_running() -> bool:
	return _spawning or alive_count > 0

func run_spawn_list(entries: Array[Dictionary]) -> void:
	_spawn_loop(entries)

func _spawn_loop(entries: Array[Dictionary]) -> void:
	_spawning = true
	for entry in entries:
		var delay: float = entry.get("delay", 0.0)
		if delay > 0.0:
			await get_tree().create_timer(delay).timeout
		_spawn_one(entry["type"], entry.get("hp_mult", 1.0))
	_spawning = false
	_check_wave_finished()

func _spawn_one(type: EnemyType, hp_mult: float) -> void:
	var scaled: EnemyType = type.duplicate()
	scaled.max_hp = type.max_hp * hp_mult
	var e: Enemy = _enemy_scene.instantiate()
	_container.add_child(e)
	e.setup(scaled, _get_path.call())
	e.died.connect(_on_enemy_died)
	e.reached_base.connect(_on_enemy_reached_base)
	alive_count += 1
	enemy_spawned.emit(e)

func _on_enemy_died(enemy: Enemy) -> void:
	alive_count -= 1
	enemy_killed.emit(enemy, enemy.bounty)
	_check_wave_finished()

func _on_enemy_reached_base(enemy: Enemy, leak_damage: int) -> void:
	alive_count -= 1
	enemy_leaked.emit(enemy, leak_damage)
	_check_wave_finished()

func _check_wave_finished() -> void:
	if not _spawning and alive_count <= 0:
		wave_finished.emit()
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_wave_director_spawn.gd -gexit`
Expected: 4 passing.

- [ ] **Step 5: 커밋**

```bash
git add scripts/services/wave_director.gd test/unit/test_wave_director_spawn.gd
git commit -m "feat: WaveDirector 고정 리스트 스포너"
```

---

## Task 11: 타게팅 로직

**Files:**
- Create: `scripts/util/targeting.gd`
- Test: `test/unit/test_targeting.gd`

**Interfaces:**
- Consumes: `Enemy` (`path_index`, `global_position`, `hp` 노출 — Task 6)
- Produces: `Targeting` (정적 함수, `extends RefCounted`)
  - enum `Policy { FIRST, CLOSEST, STRONGEST }`
  - `Targeting.pick(candidates: Array, from_pos: Vector2, policy: int) -> Object` — 후보 중 정책에 맞는 1기 반환. 비어 있으면 `null`.
    - `FIRST`: `path_index` 최대 (동률 시 `from_pos`에 더 가까운 쪽)
    - `CLOSEST`: `from_pos`와의 거리 최소
    - `STRONGEST`: `hp` 최대
  - `Targeting.policy_name(policy: int) -> String` — HUD 표시용 (`"선두"`, `"근접"`, `"강적"`)

- [ ] **Step 1: 실패하는 테스트 작성**

`test/unit/test_targeting.gd`:

```gdscript
extends GutTest

const T := preload("res://scripts/util/targeting.gd")

class FakeEnemy:
	extends RefCounted
	var path_index: int
	var global_position: Vector2
	var hp: float
	func _init(pi: int, pos: Vector2, h: float):
		path_index = pi; global_position = pos; hp = h

func test_empty_returns_null():
	assert_null(T.pick([], Vector2.ZERO, T.Policy.FIRST))

func test_first_picks_highest_path_index():
	var a := FakeEnemy.new(2, Vector2(10, 0), 100.0)
	var b := FakeEnemy.new(5, Vector2(500, 0), 100.0)
	assert_eq(T.pick([a, b], Vector2.ZERO, T.Policy.FIRST), b)

func test_closest_picks_nearest():
	var a := FakeEnemy.new(2, Vector2(10, 0), 100.0)
	var b := FakeEnemy.new(5, Vector2(500, 0), 100.0)
	assert_eq(T.pick([a, b], Vector2(20, 0), T.Policy.CLOSEST), a)

func test_strongest_picks_highest_hp():
	var a := FakeEnemy.new(2, Vector2(10, 0), 40.0)
	var b := FakeEnemy.new(5, Vector2(500, 0), 250.0)
	assert_eq(T.pick([a, b], Vector2.ZERO, T.Policy.STRONGEST), b)

func test_first_tiebreak_by_distance():
	var a := FakeEnemy.new(5, Vector2(100, 0), 100.0)
	var b := FakeEnemy.new(5, Vector2(30, 0), 100.0)
	assert_eq(T.pick([a, b], Vector2(0, 0), T.Policy.FIRST), b)

func test_policy_name():
	assert_eq(T.policy_name(T.Policy.FIRST), "선두")
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_targeting.gd -gexit`
Expected: FAIL — 파일 없음.

- [ ] **Step 3: 구현**

`scripts/util/targeting.gd`:

```gdscript
class_name Targeting
extends RefCounted

enum Policy { FIRST, CLOSEST, STRONGEST }

static func pick(candidates: Array, from_pos: Vector2, policy: int) -> Object:
	if candidates.is_empty():
		return null
	var best: Object = null
	var best_key := -INF
	var best_dist := INF
	for c in candidates:
		if c == null or not is_instance_valid(c):
			continue
		var key: float
		var dist: float = from_pos.distance_squared_to(c.global_position)
		match policy:
			Policy.FIRST:
				key = float(c.path_index)
			Policy.CLOSEST:
				key = -dist
			Policy.STRONGEST:
				key = c.hp
			_:
				key = float(c.path_index)
		if key > best_key or (is_equal_approx(key, best_key) and dist < best_dist):
			best_key = key
			best_dist = dist
			best = c
	return best

static func policy_name(policy: int) -> String:
	match policy:
		Policy.FIRST: return "선두"
		Policy.CLOSEST: return "근접"
		Policy.STRONGEST: return "강적"
		_: return "선두"
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_targeting.gd -gexit`
Expected: 6 passing.

- [ ] **Step 5: 커밋**

```bash
git add scripts/util/targeting.gd test/unit/test_targeting.gd
git commit -m "feat: 타게팅 정책 (선두/근접/강적)"
```

---

## Task 12: 투사체 + 오브젝트 풀

**Files:**
- Create: `scenes/entities/projectile.tscn`, `scripts/entities/projectile.gd`, `scripts/util/projectile_pool.gd`
- Test: `test/unit/test_projectile.gd`, `test/unit/test_projectile_pool.gd`

**Interfaces:**
- Consumes: `Enemy` (`take_damage`, `global_position` — Task 6)
- Produces:
  - `Projectile` (extends `Node2D`, `projectile.tscn` 루트)
    - `launch(from: Vector2, target: Enemy, speed: float, damage: float, on_hit: Callable) -> void` — `on_hit`은 `func(hit_enemy: Enemy) -> void` (스플래시/체인 후처리 훅). 타겟이 사라지면 마지막 위치로 직진 후 소멸.
    - 명중 시 `target.take_damage(damage)` → `on_hit.call(target)` → `deactivate()`
    - `active: bool`
    - 시그널: `finished(projectile: Projectile)` — 풀 반환 신호
  - `ProjectilePool` (extends `Node`)
    - `_init(scene: PackedScene, parent: Node, prewarm: int)`
    - `acquire() -> Projectile` — 비활성 재사용 또는 신규 생성
    - `release(p: Projectile) -> void` — 비활성화 후 보관
    - `active_count() -> int`, `pool_size() -> int`

- [ ] **Step 1: 실패하는 테스트 작성 (투사체)**

`test/unit/test_projectile.gd`:

```gdscript
extends GutTest

const ProjScene := preload("res://scenes/entities/projectile.tscn")
const EnemyScene := preload("res://scenes/entities/enemy.tscn")

func _enemy(pos: Vector2) -> Enemy:
	var e := EnemyScene.instantiate()
	add_child_autofree(e)
	var t := EnemyType.new()
	t.max_hp = 100.0
	e.setup(t, PackedVector2Array([pos, pos + Vector2(9999, 0)]))
	e.global_position = pos
	return e

func test_projectile_hits_and_damages():
	var enemy := _enemy(Vector2(50, 0))
	var p := ProjScene.instantiate()
	add_child_autofree(p)
	var hit := []
	p.launch(Vector2.ZERO, enemy, 1000.0, 25.0, func(e): hit.append(e))
	for i in range(20):
		p._physics_process(1.0 / 60.0)
	assert_eq(enemy.hp, 75.0, "명중 시 데미지")
	assert_eq(hit.size(), 1, "on_hit 콜백 1회")
	assert_false(p.active)

func test_projectile_survives_target_loss():
	var enemy := _enemy(Vector2(50, 0))
	var p := ProjScene.instantiate()
	add_child_autofree(p)
	p.launch(Vector2.ZERO, enemy, 1000.0, 10.0, func(_e): pass)
	enemy.queue_free()
	await wait_frames(2)
	for i in range(30):
		p._physics_process(1.0 / 60.0)
	assert_false(p.active, "타겟 소실 시 직진 후 비활성")
```

- [ ] **Step 2: 실패하는 테스트 작성 (풀)**

`test/unit/test_projectile_pool.gd`:

```gdscript
extends GutTest

const ProjScene := preload("res://scenes/entities/projectile.tscn")

func test_acquire_reuses_released():
	var parent := Node.new()
	add_child_autofree(parent)
	var pool := ProjectilePool.new(ProjScene, parent, 0)
	var a := pool.acquire()
	assert_eq(pool.active_count(), 1)
	pool.release(a)
	assert_eq(pool.active_count(), 0)
	var b := pool.acquire()
	assert_eq(b, a, "반환된 인스턴스 재사용")
	assert_eq(pool.pool_size(), 1, "새 인스턴스 안 만듦")

func test_prewarm_creates_inactive():
	var parent := Node.new()
	add_child_autofree(parent)
	var pool := ProjectilePool.new(ProjScene, parent, 5)
	assert_eq(pool.pool_size(), 5)
	assert_eq(pool.active_count(), 0)
```

- [ ] **Step 3: 테스트 실행 → 실패 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_projectile.gd -gexit`
Expected: FAIL — `projectile.tscn` 없음.

- [ ] **Step 4: `projectile.gd` 구현**

`scripts/entities/projectile.gd`:

```gdscript
class_name Projectile
extends Node2D

signal finished(projectile: Projectile)

const MAX_LIFETIME := 3.0

var active: bool = false

var _target: Enemy
var _last_target_pos: Vector2
var _speed: float = 800.0
var _damage: float = 0.0
var _on_hit: Callable
var _life: float = 0.0

func launch(from: Vector2, target: Enemy, speed: float, damage: float, on_hit: Callable) -> void:
	global_position = from
	_target = target
	_last_target_pos = target.global_position if is_instance_valid(target) else from
	_speed = speed
	_damage = damage
	_on_hit = on_hit
	_life = 0.0
	active = true
	visible = true

func deactivate() -> void:
	active = false
	visible = false
	_target = null
	finished.emit(self)

func _physics_process(delta: float) -> void:
	if not active:
		return
	_life += delta
	if _life >= MAX_LIFETIME:
		deactivate()
		return
	var dest: Vector2
	if is_instance_valid(_target):
		_last_target_pos = _target.global_position
		dest = _last_target_pos
	else:
		dest = _last_target_pos
	var to_dest := dest - global_position
	var step := _speed * delta
	if to_dest.length() <= step:
		global_position = dest
		if is_instance_valid(_target):
			_target.take_damage(_damage)
			if _on_hit.is_valid():
				_on_hit.call(_target)
		deactivate()
	else:
		global_position += to_dest.normalized() * step
```

- [ ] **Step 5: `projectile.tscn` 생성**

에디터: 루트 `Node2D`(스크립트 `projectile.gd`, 이름 `Projectile`) → 자식 `Sprite2D`(임시 `icon.svg`, scale 0.1). 저장 `res://scenes/entities/projectile.tscn`.

- [ ] **Step 6: `projectile_pool.gd` 구현**

`scripts/util/projectile_pool.gd`:

```gdscript
class_name ProjectilePool
extends RefCounted

var _scene: PackedScene
var _parent: Node
var _all: Array[Projectile] = []

func _init(scene: PackedScene, parent: Node, prewarm: int) -> void:
	_scene = scene
	_parent = parent
	for i in range(prewarm):
		_all.append(_make())

func _make() -> Projectile:
	var p: Projectile = _scene.instantiate()
	p.active = false
	p.visible = false
	_parent.add_child(p)
	return p

func acquire() -> Projectile:
	for p in _all:
		if not p.active:
			return p
	var np := _make()
	_all.append(np)
	return np

func release(p: Projectile) -> void:
	p.active = false
	p.visible = false

func active_count() -> int:
	var n := 0
	for p in _all:
		if p.active:
			n += 1
	return n

func pool_size() -> int:
	return _all.size()
```

- [ ] **Step 7: 테스트 실행 → 통과 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_projectile.gd -gexit`
Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_projectile_pool.gd -gexit`
Expected: 2 + 2 passing.

- [ ] **Step 8: 커밋**

```bash
git add scenes/entities/projectile.tscn scripts/entities/projectile.gd scripts/util/projectile_pool.gd test/unit/test_projectile.gd test/unit/test_projectile_pool.gd
git commit -m "feat: 투사체 + 오브젝트 풀"
```

---

## Task 13: AttackResolver — 5가지 공격 종류 해석 — codex 위임 대상

**Files:**
- Create: `scripts/services/attack_resolver.gd`
- Test: `test/unit/test_attack_resolver.gd`

**codex 계약서**
- 순수 함수. 노드·씬·시그널 접근 금지. 입력만으로 "무엇을 할지" 기술한 결과 구조를 반환.
- 입력: `kind: TowerType.Kind`, `level: TowerLevel`, `primary: Enemy`, `in_range: Array[Enemy]`, `all_towers: Array` (buff 대상 조회용, `global_position`·`apply_buff` 노출).
- 출력: `AttackPlan` (`Dictionary`):
  - `{"projectiles": Array[Dictionary], "instant_hits": Array[Dictionary], "buff": Dictionary}`
  - `projectiles` 항목: `{"target": Enemy, "damage": float, "splash_radius": float}`
  - `instant_hits` 항목: `{"target": Enemy, "damage": float}` (chain 각 대상)
  - `buff`: `{"targets": Array, "pct": float}` 또는 `{}`
- 부작용 금지: `Enemy.take_damage`를 직접 부르지 않는다. 호출자(Task 14)가 plan을 실행한다.

**Interfaces:**
- Consumes: `TowerType.Kind`, `TowerLevel` (Task 2), `Enemy` (Task 6)
- Produces: `AttackResolver.resolve(kind, level, primary, in_range, all_towers) -> Dictionary` (위 `AttackPlan`)

- [ ] **Step 1: 실패하는 테스트 작성**

`test/unit/test_attack_resolver.gd`:

```gdscript
extends GutTest

const AR := preload("res://scripts/services/attack_resolver.gd")

class FakeEnemy:
	extends RefCounted
	var global_position: Vector2
	var hp := 100.0
	var path_index := 0
	func _init(pos: Vector2): global_position = pos

func _lv(d := 10.0) -> TowerLevel:
	var l := TowerLevel.new()
	l.damage = d
	return l

func test_single_emits_one_projectile():
	var p := FakeEnemy.new(Vector2(10, 0))
	var plan := AR.resolve(TowerType.Kind.SINGLE, _lv(15.0), p, [p], [])
	assert_eq(plan["projectiles"].size(), 1)
	assert_eq(plan["projectiles"][0]["damage"], 15.0)
	assert_eq(plan["projectiles"][0]["splash_radius"], 0.0)

func test_splash_carries_radius():
	var l := _lv(20.0)
	l.splash_radius = 48.0
	var p := FakeEnemy.new(Vector2(10, 0))
	var plan := AR.resolve(TowerType.Kind.SPLASH, l, p, [p], [])
	assert_eq(plan["projectiles"][0]["splash_radius"], 48.0)

func test_slow_projectile_zero_damage_by_default():
	var l := _lv(0.0)
	l.slow_pct = 0.4
	l.slow_dur = 2000.0
	var p := FakeEnemy.new(Vector2(10, 0))
	var plan := AR.resolve(TowerType.Kind.SLOW, l, p, [p], [])
	assert_eq(plan["projectiles"].size(), 1)
	assert_eq(plan["projectiles"][0]["damage"], 0.0)
	assert_eq(plan["projectiles"][0].get("slow_pct", 0.0), 0.4)
	assert_eq(plan["projectiles"][0].get("slow_dur", 0.0), 2000.0)

func test_chain_hits_n_with_falloff():
	var l := _lv(30.0)
	l.chain_count = 3
	l.chain_falloff = 0.5
	var a := FakeEnemy.new(Vector2(0, 0))
	var b := FakeEnemy.new(Vector2(20, 0))
	var c := FakeEnemy.new(Vector2(40, 0))
	var d := FakeEnemy.new(Vector2(9999, 0))  # 사거리 밖 취급 (in_range 미포함)
	var plan := AR.resolve(TowerType.Kind.CHAIN, l, a, [a, b, c], [])
	var hits: Array = plan["instant_hits"]
	assert_eq(hits.size(), 3)
	assert_eq(hits[0]["damage"], 30.0)
	assert_eq(hits[1]["damage"], 15.0)
	assert_almost_eq(hits[2]["damage"], 7.5, 0.01)

func test_buff_lists_towers_in_radius():
	var l := _lv(0.0)
	l.buff_pct = 0.25
	l.buff_radius = 100.0
	var self_tower := FakeEnemy.new(Vector2(0, 0))
	var near := FakeEnemy.new(Vector2(50, 0))
	var far := FakeEnemy.new(Vector2(500, 0))
	var plan := AR.resolve(TowerType.Kind.BUFF, l, null, [], [self_tower, near, far])
	assert_eq(plan["buff"]["pct"], 0.25)
	assert_true(plan["buff"]["targets"].has(near))
	assert_false(plan["buff"]["targets"].has(far))
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_attack_resolver.gd -gexit`
Expected: FAIL — 파일 없음.

- [ ] **Step 3: 구현**

`scripts/services/attack_resolver.gd`:

```gdscript
class_name AttackResolver
extends RefCounted

## kind별로 "무엇을 할지"만 기술한 plan을 반환한다. 실제 피해 적용은 호출자가 한다.
static func resolve(kind: int, level: TowerLevel, primary, in_range: Array, all_towers: Array) -> Dictionary:
	var plan := {"projectiles": [], "instant_hits": [], "buff": {}}
	match kind:
		TowerType.Kind.SINGLE:
			if primary != null:
				plan["projectiles"].append({
					"target": primary, "damage": level.damage, "splash_radius": 0.0,
				})
		TowerType.Kind.SPLASH:
			if primary != null:
				plan["projectiles"].append({
					"target": primary, "damage": level.damage, "splash_radius": level.splash_radius,
				})
		TowerType.Kind.SLOW:
			if primary != null:
				plan["projectiles"].append({
					"target": primary, "damage": level.damage, "splash_radius": 0.0,
					"slow_pct": level.slow_pct, "slow_dur": level.slow_dur,
				})
		TowerType.Kind.CHAIN:
			var ordered := in_range.duplicate()
			ordered.sort_custom(func(x, y):
				return primary.global_position.distance_squared_to(x.global_position) \
					< primary.global_position.distance_squared_to(y.global_position))
			var dmg := level.damage
			var n := mini(level.chain_count, ordered.size())
			for i in range(n):
				plan["instant_hits"].append({"target": ordered[i], "damage": dmg})
				dmg *= level.chain_falloff
		TowerType.Kind.BUFF:
			var self_pos: Vector2 = all_towers[0].global_position if all_towers.size() > 0 else Vector2.ZERO
			var targets := []
			for t in all_towers:
				if self_pos.distance_to(t.global_position) <= level.buff_radius:
					targets.append(t)
			plan["buff"] = {"targets": targets, "pct": level.buff_pct}
	return plan
```

> codex 주의: `CHAIN`은 primary를 포함해 가까운 순으로 `chain_count`개를 때린다. primary가 항상 첫 대상. `BUFF`는 `all_towers[0]`을 시전자로 가정(호출자가 자기 타워를 배열 맨 앞에 넣어 전달).

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_attack_resolver.gd -gexit`
Expected: 5 passing.

- [ ] **Step 5: 커밋**

```bash
git add scripts/services/attack_resolver.gd test/unit/test_attack_resolver.gd
git commit -m "feat: AttackResolver 5종 공격 해석 (순수 함수)"
```

---

## Task 14: 타워 엔티티 + 배치 플로우

**Files:**
- Create: `scenes/entities/tower.tscn`, `scripts/entities/tower.gd`, `scripts/services/tower_container.gd`
- Test: `test/unit/test_tower_container.gd`

**Interfaces:**
- Consumes: `TowerType`/`TowerLevel` (Task 2), `MapService` (Task 4, 5), `EconomyService` (Task 9), `GridUtil` (Task 3)
- Produces:
  - `Tower` (extends `Node2D`, `tower.tscn` 루트)
    - `setup(type: TowerType) -> void` — `level_index = 0`, `invested = type.levels[0].upgrade_cost`
    - `cell: Vector2i`, `type: TowerType`, `level_index: int`, `invested: int` (읽기용)
    - `current_level() -> TowerLevel` — `type.levels[level_index]`
    - `can_upgrade() -> bool` — `level_index < type.levels.size() - 1`
    - `next_upgrade_cost() -> int` — `type.levels[level_index + 1].upgrade_cost`
    - `apply_upgrade() -> void` — `level_index += 1`, `invested += next cost`
    - `sell_value() -> int` — `floori(invested * type.sell_refund_rate)`
    - `buff_mult: float` (기본 1.0, 매 프레임 재계산 대상 — Task 15에서 사용)
  - `TowerContainer` (extends `Node2D`)
    - `configure(tower_scene: PackedScene, map: MapService, economy: EconomyService, cell_px: int) -> void`
    - `try_place(type: TowerType, cell: Vector2i, enemy_cells: Array) -> bool` — 순서: (1) `map.is_placement_legal(cell, enemy_cells)` (2) `economy.can_afford(type.levels[0].upgrade_cost)` (3) `economy.try_spend(...)` (4) 타워 인스턴스화·`setup`·`global_position` 설정·자식 추가 (5) `map.set_solid(cell, true)` (6) `map.recompute_path()` (7) `tower_placed` 발신 → `true`. 하나라도 실패 시 `false`(부작용 없음).
    - `sell(tower: Tower) -> void` — `economy.add_gold(tower.sell_value())`, `map.set_solid(cell, false)`, `map.recompute_path()`, 타워 제거, `tower_sold` 발신
    - `towers() -> Array[Tower]`
    - 시그널: `tower_placed(tower: Tower)`, `tower_sold(cell: Vector2i)`

- [ ] **Step 1: 실패하는 테스트 작성**

`test/unit/test_tower_container.gd`:

```gdscript
extends GutTest

const TowerScene := preload("res://scenes/entities/tower.tscn")

var map: MapService
var eco: EconomyService
var tc: TowerContainer

func _tower_type(cost := 50) -> TowerType:
	var t := TowerType.new()
	t.id = &"test"
	var l := TowerLevel.new()
	l.upgrade_cost = cost
	l.damage = 10.0
	var l2 := TowerLevel.new()
	l2.upgrade_cost = 80
	l2.damage = 22.0
	t.levels = [l, l2] as Array[TowerLevel]
	t.sell_refund_rate = 0.6
	return t

func before_each():
	map = MapService.new(); add_child_autofree(map)
	var cfg := MapConfig.new()
	cfg.grid_size = Vector2i(6, 5)
	cfg.spawn_cell = Vector2i(0, 2)
	cfg.base_cell = Vector2i(5, 2)
	map.setup(cfg)
	eco = EconomyService.new(); add_child_autofree(eco)
	var et := EconomyTuning.new(); et.starting_gold = 100; et.starting_base_hp = 20
	eco.setup(et)
	tc = TowerContainer.new(); add_child_autofree(tc)
	tc.configure(TowerScene, map, eco, 64)

func test_place_success_spends_and_makes_solid():
	assert_true(tc.try_place(_tower_type(50), Vector2i(2, 1), []))
	assert_eq(eco.gold, 50)
	assert_true(map.is_solid(Vector2i(2, 1)))
	assert_eq(tc.towers().size(), 1)

func test_place_denied_when_would_seal_exit():
	map.set_solid(Vector2i(3, 0), true)
	map.set_solid(Vector2i(3, 1), true)
	map.set_solid(Vector2i(3, 3), true)
	map.set_solid(Vector2i(3, 4), true)
	var before_gold := eco.gold
	assert_false(tc.try_place(_tower_type(50), Vector2i(3, 2), []))
	assert_eq(eco.gold, before_gold, "실패 시 골드 미차감")
	assert_false(map.is_solid(Vector2i(3, 2)))

func test_place_denied_when_too_expensive():
	assert_false(tc.try_place(_tower_type(500), Vector2i(2, 1), []))
	assert_eq(tc.towers().size(), 0)

func test_place_triggers_recompute():
	watch_signals(map)
	tc.try_place(_tower_type(50), Vector2i(3, 2), [])
	assert_signal_emitted(map, "path_recomputed")

func test_sell_refunds_and_frees_cell():
	tc.try_place(_tower_type(50), Vector2i(2, 1), [])
	var t := tc.towers()[0]
	tc.sell(t)
	assert_eq(eco.gold, 80, "50 남았다가 +30(50*0.6)")
	assert_false(map.is_solid(Vector2i(2, 1)))
	assert_eq(tc.towers().size(), 0)

func test_upgrade_math():
	tc.try_place(_tower_type(50), Vector2i(2, 1), [])
	var t := tc.towers()[0]
	assert_eq(t.level_index, 0)
	assert_true(t.can_upgrade())
	assert_eq(t.next_upgrade_cost(), 80)
	t.apply_upgrade()
	assert_eq(t.level_index, 1)
	assert_eq(t.invested, 130)
	assert_eq(t.sell_value(), 78)  # floor(130 * 0.6)
	assert_false(t.can_upgrade())
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_tower_container.gd -gexit`
Expected: FAIL — `tower.tscn` 없음.

- [ ] **Step 3: `tower.gd` 구현**

`scripts/entities/tower.gd`:

```gdscript
class_name Tower
extends Node2D

var type: TowerType
var cell: Vector2i
var level_index: int = 0
var invested: int = 0
var buff_mult: float = 1.0

func setup(tower_type: TowerType) -> void:
	type = tower_type
	level_index = 0
	invested = tower_type.levels[0].upgrade_cost

func current_level() -> TowerLevel:
	return type.levels[level_index]

func can_upgrade() -> bool:
	return level_index < type.levels.size() - 1

func next_upgrade_cost() -> int:
	return type.levels[level_index + 1].upgrade_cost

func apply_upgrade() -> void:
	if not can_upgrade():
		return
	invested += next_upgrade_cost()
	level_index += 1

func sell_value() -> int:
	return floori(invested * type.sell_refund_rate)

func effective_damage() -> float:
	return current_level().damage * buff_mult
```

- [ ] **Step 4: `tower.tscn` 생성**

에디터: 루트 `Node2D`(스크립트 `tower.gd`, 이름 `Tower`) → 자식 `Sprite2D`(임시 `icon.svg`, scale 0.28, 이름 `Sprite`) → 자식 `Area2D`(이름 `RangeArea`, 모니터링 on, 콜리전 마스크 = 레이어 2[적 hurtbox]) + `CollisionShape2D`(원, radius 128, 이름 `RangeShape`). 저장 `res://scenes/entities/tower.tscn`.

- [ ] **Step 5: `tower_container.gd` 구현**

`scripts/services/tower_container.gd`:

```gdscript
class_name TowerContainer
extends Node2D

signal tower_placed(tower: Tower)
signal tower_sold(cell: Vector2i)

var _tower_scene: PackedScene
var _map: MapService
var _economy: EconomyService
var _cell_px: int = 64

func configure(tower_scene: PackedScene, map: MapService, economy: EconomyService, cell_px: int) -> void:
	_tower_scene = tower_scene
	_map = map
	_economy = economy
	_cell_px = cell_px

func try_place(type: TowerType, cell: Vector2i, enemy_cells: Array) -> bool:
	if not _map.is_placement_legal(cell, enemy_cells):
		return false
	var cost: int = type.levels[0].upgrade_cost
	if not _economy.can_afford(cost):
		return false
	_economy.try_spend(cost)
	var t: Tower = _tower_scene.instantiate()
	add_child(t)
	t.setup(type)
	t.cell = cell
	t.global_position = GridUtil.cell_to_world(cell, _cell_px)
	_map.set_solid(cell, true)
	_map.recompute_path()
	tower_placed.emit(t)
	return true

func sell(tower: Tower) -> void:
	var c := tower.cell
	_economy.add_gold(tower.sell_value())
	_map.set_solid(c, false)
	_map.recompute_path()
	tower.queue_free()
	tower_sold.emit(c)

func towers() -> Array:
	var out: Array = []
	for ch in get_children():
		if ch is Tower:
			out.append(ch)
	return out
```

- [ ] **Step 6: 테스트 실행 → 통과 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_tower_container.gd -gexit`
Expected: 6 passing.

- [ ] **Step 7: 커밋**

```bash
git add scenes/entities/tower.tscn scripts/entities/tower.gd scripts/services/tower_container.gd test/unit/test_tower_container.gd
git commit -m "feat: 타워 엔티티 + 배치/판매/업그레이드 플로우"
```

---

## Task 15: 타워 공격 루프 (통합)

**Files:**
- Modify: `scripts/entities/tower.gd`, `scenes/entities/tower.tscn`
- Modify: `scripts/entities/projectile.gd` (감속·스플래시 훅 지원)
- Test: `test/unit/test_tower_attack_loop.gd`

**Interfaces:**
- Consumes: `Targeting` (Task 11), `AttackResolver` (Task 13), `ProjectilePool` (Task 12), `Enemy` (Task 6)
- Produces:
  - `Tower`:
    - `arm(pool: ProjectilePool, get_all_towers: Callable) -> void` — 공격 루프 활성화. `get_all_towers`는 `func() -> Array[Tower]` (buff 대상).
    - `target_policy: int` (기본 `Targeting.Policy.FIRST`), `cycle_policy() -> void`
    - `_physics_process`에서 `_cooldown` 누적 → `1.0 / current_level().attack_rate` 도달 시 `_fire()`
    - `_fire()`: `RangeArea`의 겹친 적 목록 수집 → `Targeting.pick` → `AttackResolver.resolve` → plan 실행:
      - `projectiles`: `pool.acquire().launch(global_position, target, PROJ_SPEED, damage * buff_mult, splash/slow 후처리 콜백)`
      - `instant_hits`: `target.take_damage(damage * buff_mult)` 즉시
      - `buff`: 각 대상 타워의 `buff_mult`를 `max(1.0, 1.0 + pct)`로 설정 (이 타워가 매 프레임 재적용; buff 타워 제거 시 다음 프레임 자연 소멸 위해 Task 15 노트 참조)
    - 시그널: `fired(tower: Tower)`
  - `Projectile.launch` 확장: `on_hit` 콜백이 splash/slow 처리를 담당하므로 시그니처 유지. 단 `launch`에 옵셔널 `extra: Dictionary = {}` 인자 추가해 `slow_pct`/`slow_dur` 전달 → 명중 시 `target.apply_slow(...)` 호출.

- [ ] **Step 1: 실패하는 테스트 작성**

`test/unit/test_tower_attack_loop.gd`:

```gdscript
extends GutTest

const TowerScene := preload("res://scenes/entities/tower.tscn")
const ProjScene := preload("res://scenes/entities/projectile.tscn")
const EnemyScene := preload("res://scenes/entities/enemy.tscn")

func _single_tower(dmg := 50.0, rate := 2.0) -> Tower:
	var tt := TowerType.new()
	tt.attack_kind = TowerType.Kind.SINGLE
	var l := TowerLevel.new()
	l.damage = dmg; l.attack_rate = rate; l.range_px = 200.0; l.upgrade_cost = 10
	tt.levels = [l] as Array[TowerLevel]
	var t: Tower = TowerScene.instantiate()
	add_child_autofree(t)
	t.setup(tt)
	t.global_position = Vector2.ZERO
	return t

func _enemy_at(pos: Vector2, hp := 100.0) -> Enemy:
	var e := EnemyScene.instantiate()
	add_child_autofree(e)
	var et := EnemyType.new(); et.max_hp = hp; et.base_speed = 0.0
	e.setup(et, PackedVector2Array([pos, pos + Vector2(9999, 0)]))
	e.global_position = pos
	return e

func test_tower_fires_and_damages_after_cooldown():
	var parent := Node.new(); add_child_autofree(parent)
	var pool := ProjectilePool.new(ProjScene, parent, 2)
	var t := _single_tower(50.0, 2.0)  # 0.5s 쿨다운
	var e := _enemy_at(Vector2(60, 0))
	t.arm(pool, func(): return [t])
	# RangeArea가 적을 감지하도록 물리 프레임 대기
	await wait_frames(3)
	for i in range(40):  # ~0.66s
		t._physics_process(1.0 / 60.0)
		for p in parent.get_children():
			p._physics_process(1.0 / 60.0)
	assert_lt(e.hp, 100.0, "쿨다운 후 발사되어 적이 피해를 입음")

func test_slow_tower_applies_slow_on_hit():
	var parent := Node.new(); add_child_autofree(parent)
	var pool := ProjectilePool.new(ProjScene, parent, 2)
	var tt := TowerType.new()
	tt.attack_kind = TowerType.Kind.SLOW
	var l := TowerLevel.new()
	l.damage = 0.0; l.attack_rate = 4.0; l.range_px = 200.0; l.upgrade_cost = 10
	l.slow_pct = 0.5; l.slow_dur = 3000.0
	tt.levels = [l] as Array[TowerLevel]
	var t: Tower = TowerScene.instantiate(); add_child_autofree(t); t.setup(tt); t.global_position = Vector2.ZERO
	var e := _enemy_at(Vector2(50, 0))
	e._base_speed = 100.0
	t.arm(pool, func(): return [t])
	await wait_frames(3)
	for i in range(40):
		t._physics_process(1.0 / 60.0)
		for p in parent.get_children():
			p._physics_process(1.0 / 60.0)
	assert_almost_eq(e.effective_speed(), 50.0, 1.0, "감속 적용됨")

func test_buff_tower_multiplies_neighbor_damage():
	var buff_tt := TowerType.new()
	buff_tt.attack_kind = TowerType.Kind.BUFF
	var bl := TowerLevel.new()
	bl.attack_rate = 4.0; bl.range_px = 10.0; bl.upgrade_cost = 10
	bl.buff_pct = 0.5; bl.buff_radius = 150.0
	buff_tt.levels = [bl] as Array[TowerLevel]
	var buff_t: Tower = TowerScene.instantiate(); add_child_autofree(buff_t); buff_t.setup(buff_tt); buff_t.global_position = Vector2.ZERO
	var dps_t := _single_tower(20.0, 2.0)
	dps_t.global_position = Vector2(80, 0)
	var parent := Node.new(); add_child_autofree(parent)
	var pool := ProjectilePool.new(ProjScene, parent, 1)
	buff_t.arm(pool, func(): return [buff_t, dps_t])
	for i in range(10):
		buff_t._physics_process(1.0 / 60.0)
	assert_almost_eq(dps_t.buff_mult, 1.5, 0.01)
	assert_almost_eq(dps_t.effective_damage(), 30.0, 0.01)
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_tower_attack_loop.gd -gexit`
Expected: FAIL — `arm` 없음.

- [ ] **Step 3: `projectile.gd`에 `extra` 인자 추가**

`launch` 시그니처를 다음으로 변경하고, 명중 처리에 감속을 추가:

```gdscript
var _extra: Dictionary = {}

func launch(from: Vector2, target: Enemy, speed: float, damage: float, on_hit: Callable, extra: Dictionary = {}) -> void:
	_extra = extra
	# ... 기존 대입 유지 ...

# 명중 블록 (target 유효 시):
	if is_instance_valid(_target):
		if _damage > 0.0:
			_target.take_damage(_damage)
		var sp: float = _extra.get("slow_pct", 0.0)
		if sp > 0.0:
			_target.apply_slow(sp, int(_extra.get("slow_dur", 0.0)))
		if _on_hit.is_valid():
			_on_hit.call(_target)
```

- [ ] **Step 4: `tower.gd`에 공격 루프 추가**

```gdscript
signal fired(tower: Tower)

const PROJ_SPEED := 900.0

var target_policy: int = Targeting.Policy.FIRST
var _pool: ProjectilePool
var _get_all_towers: Callable
var _cooldown: float = 0.0
var _armed: bool = false
@onready var _range_area: Area2D = $RangeArea
@onready var _range_shape: CollisionShape2D = $RangeArea/RangeShape

func arm(pool: ProjectilePool, get_all_towers: Callable) -> void:
	_pool = pool
	_get_all_towers = get_all_towers
	_armed = true
	_apply_range_radius()

func _apply_range_radius() -> void:
	if _range_shape and _range_shape.shape is CircleShape2D:
		(_range_shape.shape as CircleShape2D).radius = current_level().range_px

func cycle_policy() -> void:
	target_policy = (target_policy + 1) % 3

func _enemies_in_range() -> Array:
	var out: Array = []
	for a in _range_area.get_overlapping_areas():
		var owner := a.get_parent()
		if owner is Enemy:
			out.append(owner)
	return out

func _physics_process(delta: float) -> void:
	if not _armed:
		return
	_cooldown -= delta
	if type.attack_kind == TowerType.Kind.BUFF:
		_fire()  # buff는 매 프레임 재적용
		return
	if _cooldown <= 0.0:
		if _fire():
			_cooldown = 1.0 / maxf(0.01, current_level().attack_rate)

func _fire() -> bool:
	var lv := current_level()
	if type.attack_kind == TowerType.Kind.BUFF:
		var plan_b := AttackResolver.resolve(type.attack_kind, lv, null, [], _prepend_self(_get_all_towers.call()))
		for tw in plan_b["buff"].get("targets", []):
			tw.buff_mult = maxf(tw.buff_mult, 1.0 + plan_b["buff"]["pct"])
		return true
	var candidates := _enemies_in_range()
	var primary := Targeting.pick(candidates, global_position, target_policy)
	if primary == null:
		return false
	var plan := AttackResolver.resolve(type.attack_kind, lv, primary, candidates, [])
	for pj in plan["projectiles"]:
		var p := _pool.acquire()
		var dmg: float = pj["damage"] * buff_mult
		var extra := {}
		if pj.has("slow_pct"):
			extra = {"slow_pct": pj["slow_pct"], "slow_dur": pj["slow_dur"]}
		p.launch(global_position, pj["target"], PROJ_SPEED, dmg,
			_make_splash_cb(pj.get("splash_radius", 0.0), dmg), extra)
	for hit in plan["instant_hits"]:
		if is_instance_valid(hit["target"]):
			hit["target"].take_damage(hit["damage"] * buff_mult)
	fired.emit(self)
	return true

func _prepend_self(arr: Array) -> Array:
	var out := [self]
	for t in arr:
		if t != self:
			out.append(t)
	return out

func _make_splash_cb(radius: float, dmg: float) -> Callable:
	if radius <= 0.0:
		return func(_e): pass
	return func(hit_enemy):
		for a in _range_area.get_overlapping_areas():
			var o := a.get_parent()
			if o is Enemy and o != hit_enemy:
				if hit_enemy.global_position.distance_to(o.global_position) <= radius:
					o.take_damage(dmg)
```

> **buff 소멸 처리 (M5 순진 버전)**: `game.gd`가 매 물리 프레임 **처음에** 모든 타워의 `buff_mult = 1.0`으로 리셋한 뒤, 그 프레임에 buff 타워들이 `_fire()`로 다시 칠한다. 이 리셋은 Task 19의 `game.gd`에 넣는다. 테스트에서는 buff 타워 `_process`를 직접 돌려 검증.

- [ ] **Step 5: 테스트 실행 → 통과 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_tower_attack_loop.gd -gexit`
Expected: 3 passing. (물리 프레임 타이밍이 민감하면 `wait_frames` 수를 늘린다.)

- [ ] **Step 6: 전체 회귀**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gdir=res://test/unit -gexit -glog=1`
Expected: 모든 테스트 통과.

- [ ] **Step 7: 커밋**

```bash
git add scripts/entities/tower.gd scripts/entities/projectile.gd scenes/entities/tower.tscn test/unit/test_tower_attack_loop.gd
git commit -m "feat: 타워 공격 루프 (타게팅+리졸버+투사체 통합)"
```

---

## Task 16: 타워 데이터 인스턴스 (.tres ×5, 각 3레벨)

**Files:**
- Create: `data/towers/bolt_tower.tres`, `frost_totem.tres`, `mortar.tres`, `arc_coil.tres`, `war_horn.tres`
- Test: `test/unit/test_tower_data.gd`

**Interfaces:**
- Consumes: `TowerType`/`TowerLevel` (Task 2)
- Produces: 5개 리소스. `bolt_tower`·`frost_totem`은 `starter_unlocked = true`. 각 `levels` 크기 3. `projectile`은 `res://scenes/entities/projectile.tscn`.

- [ ] **Step 1: 실패하는 테스트 작성**

`test/unit/test_tower_data.gd`:

```gdscript
extends GutTest

func _load(n: String) -> TowerType:
	return load("res://data/towers/%s.tres" % n) as TowerType

func test_all_load_with_three_levels():
	for n in ["bolt_tower", "frost_totem", "mortar", "arc_coil", "war_horn"]:
		var t := _load(n)
		assert_not_null(t, "%s 로드" % n)
		assert_eq(String(t.id), n)
		assert_eq(t.levels.size(), 3, "%s 3레벨" % n)
		assert_gt(t.levels[0].upgrade_cost, 0, "%s L1 배치비용" % n)

func test_starters_flagged():
	assert_true(_load("bolt_tower").starter_unlocked)
	assert_true(_load("frost_totem").starter_unlocked)
	assert_false(_load("mortar").starter_unlocked)
	assert_false(_load("arc_coil").starter_unlocked)
	assert_false(_load("war_horn").starter_unlocked)

func test_kinds():
	assert_eq(_load("bolt_tower").attack_kind, TowerType.Kind.SINGLE)
	assert_eq(_load("frost_totem").attack_kind, TowerType.Kind.SLOW)
	assert_eq(_load("mortar").attack_kind, TowerType.Kind.SPLASH)
	assert_eq(_load("arc_coil").attack_kind, TowerType.Kind.CHAIN)
	assert_eq(_load("war_horn").attack_kind, TowerType.Kind.BUFF)

func test_upgrade_costs_increase():
	for n in ["bolt_tower", "frost_totem", "mortar", "arc_coil", "war_horn"]:
		var lv := _load(n).levels
		assert_gt(lv[1].upgrade_cost, 0)
		assert_gt(lv[2].upgrade_cost, lv[1].upgrade_cost, "%s L3 > L2 비용" % n)

func test_frost_has_slow_values():
	var f := _load("frost_totem")
	assert_gt(f.levels[0].slow_pct, 0.0)
	assert_gt(f.levels[0].slow_dur, 0.0)
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_tower_data.gd -gexit`
Expected: FAIL — 로드 실패.

- [ ] **Step 3: `.tres` 생성 (인스펙터)**

`data/towers/`에 `TowerType` 리소스 5개. 각 `levels`에 `TowerLevel` 서브리소스 3개 추가. 시작 밸런스(1차, Task 20에서 튜닝):

**bolt_tower** — `id=bolt_tower`, `attack_kind=SINGLE`, `starter_unlocked=true`, `projectile=projectile.tscn`, `sell_refund_rate=0.6`
| lv | damage | attack_rate | range_px | upgrade_cost |
|---|---|---|---|---|
| 1 | 8 | 1.6 | 150 | 40 |
| 2 | 16 | 1.8 | 165 | 55 |
| 3 | 30 | 2.0 | 180 | 110 |

**frost_totem** — `id=frost_totem`, `attack_kind=SLOW`, `starter_unlocked=true`, `projectile=projectile.tscn`
| lv | damage | attack_rate | range_px | slow_pct | slow_dur | upgrade_cost |
|---|---|---|---|---|---|---|
| 1 | 2 | 1.2 | 150 | 0.30 | 1800 | 50 |
| 2 | 4 | 1.2 | 165 | 0.40 | 2200 | 70 |
| 3 | 7 | 1.3 | 180 | 0.50 | 2800 | 140 |

**mortar** — `id=mortar`, `attack_kind=SPLASH`, `projectile=projectile.tscn`
| lv | damage | attack_rate | range_px | splash_radius | upgrade_cost |
|---|---|---|---|---|---|
| 1 | 22 | 0.55 | 170 | 56 | 110 |
| 2 | 40 | 0.6 | 185 | 64 | 150 |
| 3 | 70 | 0.7 | 200 | 74 | 320 |

**arc_coil** — `id=arc_coil`, `attack_kind=CHAIN`, `projectile=projectile.tscn`
| lv | damage | attack_rate | range_px | chain_count | chain_falloff | upgrade_cost |
|---|---|---|---|---|---|---|
| 1 | 10 | 1.2 | 160 | 3 | 0.6 | 90 |
| 2 | 18 | 1.3 | 175 | 4 | 0.65 | 130 |
| 3 | 30 | 1.5 | 190 | 5 | 0.7 | 280 |

**war_horn** — `id=war_horn`, `attack_kind=BUFF` (damage/attack_rate 무의미, `attack_rate=4` 권장)
| lv | attack_rate | range_px | buff_pct | buff_radius | upgrade_cost |
|---|---|---|---|---|---|
| 1 | 4 | 10 | 0.15 | 120 | 80 |
| 2 | 4 | 10 | 0.22 | 135 | 120 |
| 3 | 4 | 10 | 0.30 | 150 | 240 |

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_tower_data.gd -gexit`
Expected: 5 passing.

- [ ] **Step 5: 커밋**

```bash
git add data/towers test/unit/test_tower_data.gd
git commit -m "feat: 타워 데이터 5종 (각 3레벨, 1차 밸런스)"
```

---

## Task 17: 절차적 웨이브 생성 — codex 위임 대상

**Files:**
- Modify: `scripts/services/wave_director.gd`
- Create: `data/tuning/wave_tuning.tres`
- Test: `test/unit/test_wave_build.gd`

**codex 계약서**
- 순수 함수 우선. `build_wave`는 노드/씬 접근 없이 `Array[Dictionary]` 스폰 리스트만 계산.
- 입력: `wave_index: int`, `tuning: WaveTuning`, `enemy_pool: Dictionary` (`id: StringName -> EnemyType`), `seed: int`.
- 출력: `Array[Dictionary]` — 각 `{"type": EnemyType, "delay": float, "hp_mult": float}`. `delay`는 **직전 항목으로부터의 간격**(초).
- 규칙:
  - 예산 `budget = tuning.budget_a + tuning.budget_b*N + tuning.budget_c*N*N`.
  - `N in tuning.boss_waves`이면 먼저 보스 1기 추가(`N < 40` → `miniboss`, `N == 40` → `finalboss`). 보스 `hp_mult`: 미니 `1 + tuning.miniboss_hp_growth*(N/10 - 1)`, 최종 `tuning.finalboss_hp / finalboss.max_hp` (즉 절대값 고정), 무한(N>40)이면 최종 보스에 `pow(1+endless_boss_growth, N-40)` 추가. 보스는 예산의 절반을 소비한 것으로 간주.
  - 남은 예산으로, `min_wave <= N`인 잡몹만 뽑아 `point_cost`만큼 차감하며 채운다. 잡몹 `hp_mult = 1 + hp_scale_a*(N-1) + hp_scale_b*(N-1)^2`.
  - 구성 가중치: 각 적의 `weight = lerp(weight_early, weight_late, clamp(N/40, 0, 1))` (`unlock_table`에서). `unlock_table`이 비어 있으면 균등.
  - 스폰 간격: `interval = max(tuning.spawn_interval_floor, tuning.spawn_interval_start - tuning.spawn_interval_decay*(N-1))`. 첫 항목 `delay = 0`, 이후 전부 `interval`. 보스는 잡몹 뒤에 `delay = interval * 2`.
  - **결정성**: 같은 `seed`·`N`이면 완전히 동일한 리스트. `RandomNumberGenerator`를 `seed`로 초기화해서만 난수 사용.
- 부작용 금지.

**Interfaces:**
- Consumes: `WaveTuning`/`EnemyType` (Task 2, 8)
- Produces:
  - `WaveDirector.build_wave(wave_index: int, tuning: WaveTuning, enemy_pool: Dictionary, seed: int) -> Array[Dictionary]` (정적)
  - `WaveDirector.run_wave(wave_index, tuning, enemy_pool, seed) -> void` — `build_wave` 결과를 `run_spawn_list`에 넘김
  - `WaveDirector.is_boss_wave(wave_index, tuning) -> bool` (정적)

- [ ] **Step 1: 실패하는 테스트 작성**

`test/unit/test_wave_build.gd`:

```gdscript
extends GutTest

const WD := preload("res://scripts/services/wave_director.gd")

func _pool() -> Dictionary:
	var d := {}
	for n in ["shambler", "hound", "swarmling", "brute", "miniboss", "finalboss"]:
		d[StringName(n)] = load("res://data/enemies/%s.tres" % n)
	return d

func _tuning() -> WaveTuning:
	return load("res://data/tuning/wave_tuning.tres") as WaveTuning

func test_early_wave_only_shamblers():
	var list := WD.build_wave(1, _tuning(), _pool(), 12345)
	assert_gt(list.size(), 0)
	for e in list:
		assert_eq(String(e["type"].id), "shambler", "웨이브 1은 shambler만")

func test_budget_grows_more_enemies_later():
	var w1 := WD.build_wave(1, _tuning(), _pool(), 1).size()
	var w15 := WD.build_wave(15, _tuning(), _pool(), 1).size()
	assert_gt(w15, w1, "후반 웨이브가 적이 더 많다")

func test_hp_mult_scales_with_wave():
	var w1 := WD.build_wave(1, _tuning(), _pool(), 1)
	var w20 := WD.build_wave(20, _tuning(), _pool(), 1)
	assert_eq(w1[0]["hp_mult"], 1.0)
	# w20의 잡몹(보스 제외) hp_mult > 2.0
	var scrub := w20.filter(func(e): return String(e["type"].id) != "miniboss")
	assert_gt(scrub[0]["hp_mult"], 2.0)

func test_boss_waves():
	assert_true(WD.is_boss_wave(10, _tuning()))
	assert_true(WD.is_boss_wave(40, _tuning()))
	assert_false(WD.is_boss_wave(11, _tuning()))
	var w10 := WD.build_wave(10, _tuning(), _pool(), 1)
	assert_true(w10.any(func(e): return String(e["type"].id) == "miniboss"))
	var w40 := WD.build_wave(40, _tuning(), _pool(), 1)
	assert_true(w40.any(func(e): return String(e["type"].id) == "finalboss"))

func test_finalboss_hp_is_fixed_absolute():
	var w40 := WD.build_wave(40, _tuning(), _pool(), 1)
	var boss: Dictionary = w40.filter(func(e): return String(e["type"].id) == "finalboss")[0]
	# hp_mult * finalboss.max_hp == tuning.finalboss_hp
	assert_almost_eq(boss["hp_mult"] * boss["type"].max_hp, _tuning().finalboss_hp, 1.0)

func test_deterministic_by_seed():
	var a := WD.build_wave(17, _tuning(), _pool(), 999)
	var b := WD.build_wave(17, _tuning(), _pool(), 999)
	assert_eq(a.size(), b.size())
	for i in range(a.size()):
		assert_eq(a[i]["type"].id, b[i]["type"].id, "항목 %d 동일" % i)
	var c := WD.build_wave(17, _tuning(), _pool(), 1000)
	# 다른 시드는 (거의 확실히) 다른 구성
	var same := a.size() == c.size()
	if same:
		var all_eq := true
		for i in range(a.size()):
			if a[i]["type"].id != c[i]["type"].id:
				all_eq = false
				break
		assert_false(all_eq, "다른 시드는 다른 구성")
```

- [ ] **Step 2: `data/tuning/wave_tuning.tres` 생성**

`WaveTuning` 리소스. 기본값 유지 + `unlock_table`을 다음으로 (인스펙터에서 Dictionary 배열 편집):

```
[
  {"enemy_id": &"shambler",  "min_wave": 1, "weight_early": 10.0, "weight_late": 3.0},
  {"enemy_id": &"hound",     "min_wave": 3, "weight_early": 4.0,  "weight_late": 6.0},
  {"enemy_id": &"swarmling", "min_wave": 5, "weight_early": 3.0,  "weight_late": 7.0},
  {"enemy_id": &"brute",     "min_wave": 8, "weight_early": 1.0,  "weight_late": 5.0},
]
```

- [ ] **Step 3: 테스트 실행 → 실패 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_wave_build.gd -gexit`
Expected: FAIL — `build_wave` 없음.

- [ ] **Step 4: `wave_director.gd`에 추가**

```gdscript
static func is_boss_wave(wave_index: int, tuning: WaveTuning) -> bool:
	return tuning.boss_waves.has(wave_index)

static func build_wave(wave_index: int, tuning: WaveTuning, enemy_pool: Dictionary, seed: int) -> Array[Dictionary]:
	var rng := RandomNumberGenerator.new()
	rng.seed = seed
	var n := wave_index
	var budget: float = tuning.budget_a + tuning.budget_b * n + tuning.budget_c * n * n
	var scrub_hp_mult: float = 1.0 + tuning.hp_scale_a * (n - 1) + tuning.hp_scale_b * (n - 1) * (n - 1)
	var interval: float = maxf(tuning.spawn_interval_floor,
		tuning.spawn_interval_start - tuning.spawn_interval_decay * (n - 1))

	var boss_entry: Dictionary = {}
	if is_boss_wave(n, tuning):
		budget *= 0.5
		if n >= 40:
			var fb: EnemyType = enemy_pool[&"finalboss"]
			var mult: float = tuning.finalboss_hp / fb.max_hp
			if n > 40:
				mult *= pow(1.0 + tuning.endless_boss_growth, n - 40)
			boss_entry = {"type": fb, "delay": interval * 2.0, "hp_mult": mult}
		else:
			var mb: EnemyType = enemy_pool[&"miniboss"]
			var mult: float = 1.0 + tuning.miniboss_hp_growth * (float(n) / 10.0 - 1.0)
			boss_entry = {"type": mb, "delay": interval * 2.0, "hp_mult": mult}

	# 잡몹 후보 + 가중치
	var weights: Array = []  # [{type, w}]
	for row in tuning.unlock_table:
		if n >= int(row["min_wave"]) and enemy_pool.has(row["enemy_id"]):
			var t: float = clampf(float(n) / 40.0, 0.0, 1.0)
			var w: float = lerpf(float(row["weight_early"]), float(row["weight_late"]), t)
			weights.append({"type": enemy_pool[row["enemy_id"]], "w": maxf(0.01, w)})
	if weights.is_empty() and enemy_pool.has(&"shambler"):
		weights.append({"type": enemy_pool[&"shambler"], "w": 1.0})

	var out: Array[Dictionary] = []
	var spent: float = 0.0
	var guard := 0
	while spent < budget and guard < 500:
		guard += 1
		var pick: EnemyType = _weighted_pick(weights, rng)
		if pick == null:
			break
		spent += pick.point_cost
		out.append({
			"type": pick,
			"delay": 0.0 if out.is_empty() else interval,
			"hp_mult": scrub_hp_mult,
		})
	if not boss_entry.is_empty():
		out.append(boss_entry)
	if out.is_empty() and enemy_pool.has(&"shambler"):
		out.append({"type": enemy_pool[&"shambler"], "delay": 0.0, "hp_mult": scrub_hp_mult})
	return out

static func _weighted_pick(weights: Array, rng: RandomNumberGenerator) -> EnemyType:
	if weights.is_empty():
		return null
	var total := 0.0
	for e in weights:
		total += e["w"]
	var r := rng.randf() * total
	for e in weights:
		r -= e["w"]
		if r <= 0.0:
			return e["type"]
	return weights[weights.size() - 1]["type"]

func run_wave(wave_index: int, tuning: WaveTuning, enemy_pool: Dictionary, seed: int) -> void:
	run_spawn_list(build_wave(wave_index, tuning, enemy_pool, seed))
```

- [ ] **Step 5: 테스트 실행 → 통과 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_wave_build.gd -gexit`
Expected: 6 passing.

- [ ] **Step 6: 커밋**

```bash
git add scripts/services/wave_director.gd data/tuning/wave_tuning.tres test/unit/test_wave_build.gd
git commit -m "feat: 절차적 웨이브 생성 (예산제/시드/보스 곡선)"
```

---

## Task 18: RelicService 스켈레톤 + 시작 유물 픽 데이터

**Files:**
- Create: `scripts/services/relic_service.gd`
- Create: `data/relics/{blood_pact,overload_core,labyrinth_eng,toll_keeper,long_road,blast_legacy}.tres`
- Test: `test/unit/test_relic_service.gd`

> **범위 주의**: M5에서는 유물 **선택 플로우와 보유 상태**만 구현한다. 실제 효과(스탯 모디파이어)는 M8. `RelicService`는 풀 관리 + `grant` + `has_relic` + `roll_choices`만 한다.

**Interfaces:**
- Consumes: `RelicType` (Task 2)
- Produces: `RelicService` (extends `Node`)
  - `setup(pool: Array[RelicType], seed: int) -> void`
  - `roll_choices(count: int) -> Array[RelicType]` — 미보유 유물 중 가중치 없이(M5는 균등) `count`개 뽑기. 시드 rng 사용. 남은 게 부족하면 있는 만큼.
  - `grant(relic: RelicType) -> void` — 보유 목록에 추가, 풀에서 제거, `relic_acquired` 발신
  - `has_relic(id: StringName) -> bool`
  - `owned() -> Array[RelicType]`
  - 시그널: `relic_acquired(relic: RelicType)`

- [ ] **Step 1: 실패하는 테스트 작성**

`test/unit/test_relic_service.gd`:

```gdscript
extends GutTest

func _pool() -> Array:
	var ids := ["blood_pact", "overload_core", "labyrinth_eng", "toll_keeper", "long_road", "blast_legacy"]
	var out: Array[RelicType] = []
	for i in ids:
		out.append(load("res://data/relics/%s.tres" % i))
	return out

var svc: RelicService

func before_each():
	svc = RelicService.new()
	add_child_autofree(svc)
	svc.setup(_pool(), 42)

func test_all_relic_files_load():
	assert_eq(_pool().size(), 6)
	for r in _pool():
		assert_ne(String(r.id), "")
		assert_ne(r.description, "")

func test_roll_choices_returns_distinct():
	var c := svc.roll_choices(3)
	assert_eq(c.size(), 3)
	assert_eq(c[0], c[0])
	assert_ne(c[0].id, c[1].id)
	assert_ne(c[1].id, c[2].id)

func test_grant_removes_from_pool():
	var c := svc.roll_choices(3)
	svc.grant(c[0])
	assert_true(svc.has_relic(c[0].id))
	assert_eq(svc.owned().size(), 1)
	# 이후 roll에 이미 보유한 건 안 나옴
	for _i in range(20):
		for r in svc.roll_choices(3):
			assert_false(r.id == c[0].id, "보유 유물은 재출현 안 함")

func test_roll_deterministic_by_seed():
	var a := RelicService.new(); add_child_autofree(a); a.setup(_pool(), 7)
	var b := RelicService.new(); add_child_autofree(b); b.setup(_pool(), 7)
	var ra := a.roll_choices(3).map(func(r): return String(r.id))
	var rb := b.roll_choices(3).map(func(r): return String(r.id))
	assert_eq(ra, rb)

func test_grant_emits():
	watch_signals(svc)
	var c := svc.roll_choices(1)
	svc.grant(c[0])
	assert_signal_emitted(svc, "relic_acquired")
```

- [ ] **Step 2: `data/relics/*.tres` 생성**

`RelicType` 6개. `hooks`/`params`는 M8이 소비하므로 지금 채워두되 미사용:

| id | display_name | description | hooks | params |
|---|---|---|---|---|
| `blood_pact` | 피의 계약 | 처치 골드 +25%, 기지 최대 HP −5 | `["on_enemy_killed","stat_mod"]` | `{"gold_bonus":0.25,"max_hp_delta":-5}` |
| `overload_core` | 과부하 코어 | 타워 공속 +20%, 사거리 −10% | `["stat_mod"]` | `{"rate_mult":1.2,"range_mult":0.9}` |
| `labyrinth_eng` | 미로공학 | 타워 판매 100% 환급 | `["stat_mod"]` | `{"sell_rate":1.0}` |
| `toll_keeper` | 관문세 | 적이 지나온 셀 수 × 0.5G 추가 처치 보상 | `["on_enemy_killed"]` | `{"per_cell":0.5}` |
| `long_road` | 길 잃은 자 | 현재 경로 ≥ 24셀이면 전 타워 딜 +30% | `["stat_mod","on_path_recomputed"]` | `{"threshold":24,"dmg_mult":1.3}` |
| `blast_legacy` | 폭발 유산 | 적 사망 시 소형 범위 폭발 | `["on_enemy_killed"]` | `{"radius":64,"damage":25}` |

- [ ] **Step 3: 테스트 실행 → 실패 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_relic_service.gd -gexit`
Expected: FAIL.

- [ ] **Step 4: 구현**

`scripts/services/relic_service.gd`:

```gdscript
class_name RelicService
extends Node

signal relic_acquired(relic: RelicType)

var _pool: Array[RelicType] = []
var _owned: Array[RelicType] = []
var _rng := RandomNumberGenerator.new()

func setup(pool: Array[RelicType], seed: int) -> void:
	_pool = pool.duplicate()
	_owned.clear()
	_rng.seed = seed

func roll_choices(count: int) -> Array[RelicType]:
	var avail := _pool.duplicate()
	# 시드 기반 셔플
	for i in range(avail.size() - 1, 0, -1):
		var j := _rng.randi_range(0, i)
		var tmp = avail[i]; avail[i] = avail[j]; avail[j] = tmp
	var out: Array[RelicType] = []
	for k in range(mini(count, avail.size())):
		out.append(avail[k])
	return out

func grant(relic: RelicType) -> void:
	if has_relic(relic.id):
		return
	_owned.append(relic)
	_pool.erase(relic)
	relic_acquired.emit(relic)

func has_relic(id: StringName) -> bool:
	for r in _owned:
		if r.id == id:
			return true
	return false

func owned() -> Array[RelicType]:
	return _owned.duplicate()
```

- [ ] **Step 5: 테스트 실행 → 통과 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_relic_service.gd -gexit`
Expected: 6 passing.

- [ ] **Step 6: 커밋**

```bash
git add scripts/services/relic_service.gd data/relics test/unit/test_relic_service.gd
git commit -m "feat: RelicService 스켈레톤 + 유물 데이터 6종 (효과는 M8)"
```

---

## Task 19: 게임 상태 머신 (헤드리스 로직)

**Files:**
- Create: `scripts/game_state.gd`
- Test: `test/unit/test_game_state.gd`

> **분리 이유**: 상태 전이·웨이브 인덱스·보상 계산·승패 판정은 씬 없이 테스트 가능한 순수 로직으로 뺀다. Task 20의 `game.gd`가 이 `GameState`를 소유하고 서비스·HUD와 배선한다.

**Interfaces:**
- Consumes: `EconomyTuning` (Task 2), `WaveDirector.is_boss_wave` (Task 17)
- Produces: `GameState` (extends `RefCounted`)
  - enum `Phase { RUN_START, BUILD, WAVE_ACTIVE, RUN_WON, GAME_OVER }`
  - `_init(economy_tuning: EconomyTuning, total_waves: int)`
  - `phase: int`, `wave_index: int` (읽기용, `RUN_START`에서 `wave_index == 0`)
  - `begin_run() -> void` — `phase = BUILD`, `wave_index = 1`
  - `start_wave() -> void` — `BUILD`에서만. `phase = WAVE_ACTIVE`
  - `wave_reward(n: int) -> int` — `economy_tuning.wave_reward_base + economy_tuning.wave_reward_per_wave * n`
  - `early_call_bonus(pending: int) -> int` — `economy_tuning.early_call_bonus_per_pending * pending`
  - `on_wave_finished(final_boss_killed: bool) -> void` — `WAVE_ACTIVE`에서만. `wave_index == total_waves and final_boss_killed` → `phase = RUN_WON`. 아니면 `phase = BUILD`, `wave_index += 1`.
  - `on_base_destroyed() -> void` — `phase = GAME_OVER` (어느 상태에서든)
  - `is_relic_pick_wave(relic_pick_every: int) -> bool` — 방금 끝난 웨이브(`wave_index`가 증가하기 전 값)가 `relic_pick_every`의 배수인지. `on_wave_finished` 내부에서 판정해 `pending_relic_pick: bool`로 노출.
  - `pending_relic_pick: bool` — `on_wave_finished` 후, 유물 픽이 필요하면 `true`. `consume_relic_pick()`로 소비.

- [ ] **Step 1: 실패하는 테스트 작성**

`test/unit/test_game_state.gd`:

```gdscript
extends GutTest

const GameState := preload("res://scripts/game_state.gd")

func _tuning() -> EconomyTuning:
	var t := EconomyTuning.new()
	t.wave_reward_base = 25
	t.wave_reward_per_wave = 5
	t.early_call_bonus_per_pending = 2
	return t

func test_begin_run_sets_build_wave_1():
	var gs := GameState.new(_tuning(), 40)
	assert_eq(gs.phase, GameState.Phase.RUN_START)
	gs.begin_run()
	assert_eq(gs.phase, GameState.Phase.BUILD)
	assert_eq(gs.wave_index, 1)

func test_start_wave_only_from_build():
	var gs := GameState.new(_tuning(), 40)
	gs.begin_run()
	gs.start_wave()
	assert_eq(gs.phase, GameState.Phase.WAVE_ACTIVE)
	gs.start_wave()  # 무시
	assert_eq(gs.phase, GameState.Phase.WAVE_ACTIVE)

func test_wave_finished_advances_and_returns_to_build():
	var gs := GameState.new(_tuning(), 40)
	gs.begin_run()
	gs.start_wave()
	gs.on_wave_finished(false)
	assert_eq(gs.phase, GameState.Phase.BUILD)
	assert_eq(gs.wave_index, 2)

func test_final_wave_boss_kill_wins():
	var gs := GameState.new(_tuning(), 3)
	gs.begin_run()
	for i in range(2):
		gs.start_wave(); gs.on_wave_finished(false)
	assert_eq(gs.wave_index, 3)
	gs.start_wave()
	gs.on_wave_finished(true)
	assert_eq(gs.phase, GameState.Phase.RUN_WON)

func test_final_wave_without_boss_kill_continues():
	var gs := GameState.new(_tuning(), 3)
	gs.begin_run()
	for i in range(2):
		gs.start_wave(); gs.on_wave_finished(false)
	gs.start_wave()
	gs.on_wave_finished(false)
	assert_eq(gs.phase, GameState.Phase.BUILD)
	assert_eq(gs.wave_index, 4, "무한 모드로 계속")

func test_base_destroyed_is_game_over():
	var gs := GameState.new(_tuning(), 40)
	gs.begin_run(); gs.start_wave()
	gs.on_base_destroyed()
	assert_eq(gs.phase, GameState.Phase.GAME_OVER)

func test_reward_and_bonus_math():
	var gs := GameState.new(_tuning(), 40)
	assert_eq(gs.wave_reward(1), 30)
	assert_eq(gs.wave_reward(10), 75)
	assert_eq(gs.early_call_bonus(7), 14)

func test_relic_pick_on_multiples_of_five():
	var gs := GameState.new(_tuning(), 40)
	gs.begin_run()
	for i in range(4):
		gs.start_wave(); gs.on_wave_finished(false)
		assert_false(gs.pending_relic_pick, "웨이브 %d 후엔 픽 없음" % (i + 1))
	gs.start_wave(); gs.on_wave_finished(false)  # 웨이브 5 종료
	assert_true(gs.pending_relic_pick)
	gs.consume_relic_pick()
	assert_false(gs.pending_relic_pick)
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_game_state.gd -gexit`
Expected: FAIL — 파일 없음.

- [ ] **Step 3: 구현**

`scripts/game_state.gd`:

```gdscript
class_name GameState
extends RefCounted

enum Phase { RUN_START, BUILD, WAVE_ACTIVE, RUN_WON, GAME_OVER }

var phase: int = Phase.RUN_START
var wave_index: int = 0
var pending_relic_pick: bool = false

var _eco: EconomyTuning
var _total_waves: int
var _relic_every: int = 5

func _init(economy_tuning: EconomyTuning, total_waves: int) -> void:
	_eco = economy_tuning
	_total_waves = total_waves

func begin_run() -> void:
	phase = Phase.BUILD
	wave_index = 1

func start_wave() -> void:
	if phase != Phase.BUILD:
		return
	phase = Phase.WAVE_ACTIVE

func wave_reward(n: int) -> int:
	return _eco.wave_reward_base + _eco.wave_reward_per_wave * n

func early_call_bonus(pending: int) -> int:
	return _eco.early_call_bonus_per_pending * pending

func on_wave_finished(final_boss_killed: bool) -> void:
	if phase != Phase.WAVE_ACTIVE:
		return
	if wave_index >= _total_waves and final_boss_killed:
		phase = Phase.RUN_WON
		return
	pending_relic_pick = (wave_index % _relic_every == 0)
	wave_index += 1
	phase = Phase.BUILD

func consume_relic_pick() -> void:
	pending_relic_pick = false

func on_base_destroyed() -> void:
	phase = Phase.GAME_OVER
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_game_state.gd -gexit`
Expected: 8 passing.

- [ ] **Step 5: 커밋**

```bash
git add scripts/game_state.gd test/unit/test_game_state.gd
git commit -m "feat: GameState 상태 머신 (헤드리스 로직)"
```

---

## Task 20: 통합 씬 + HUD + 수동 플레이테스트 (M5 재미 중간 점검)

**Files:**
- Create: `scenes/map/map_field.tscn`, `data/maps/map_field.tres`
- Create: `data/tuning/economy_tuning.tres`, `data/run/default_run.tres`
- Create: `scenes/game/game.tscn`, `scripts/game.gd`
- Create: `scenes/game/hud.tscn`, `scripts/hud/hud.gd`
- Modify: `project.godot` (main scene = `game.tscn`)
- Test: `test/unit/test_game_integration.gd`

**Interfaces:**
- Consumes: 모든 이전 태스크의 서비스·엔티티.
- Produces:
  - `Game` (extends `Node2D`, `game.tscn` 루트) — `GameState` 소유, 서비스 노드 조립, HUD 배선.
    - `@onready` 서비스: `MapService`, `EconomyService`, `WaveDirector`, `TowerContainer`, `RelicService`, `ProjectilePool`(코드 생성)
    - `_ready()`: `RunConfig` 로드 → 각 서비스 `setup/configure` → `map_field.tscn` 인스턴스 → `GameState.new(...)` → `RUN_START` 유물 픽 모달 → `begin_run()`
    - `_physics_process(delta)`: **매 프레임 시작에 모든 타워 `buff_mult = 1.0` 리셋** (Task 15 노트), 그다음 엔진이 각 타워 `_physics_process` 실행 (buff 타워가 다시 칠함)
    - 입력: 빈 셀 탭 → 선택된 타워 배치 시도(`TowerContainer.try_place`, 적 셀 목록 = 현재 살아있는 적들의 `current_cell`), 기존 타워 탭 → 인스펙터
    - `WaveDirector.enemy_killed` → `EconomyService.add_gold(bounty)`; `enemy_leaked` → `EconomyService.damage_base(leak)`; `wave_finished` → 보상 지급 + `GameState.on_wave_finished(_final_boss_killed)` + 유물 픽 모달(필요 시) + 상태별 화면
    - `EconomyService.base_destroyed` → `GameState.on_base_destroyed()` → 패배 화면
    - 조기 시작 버튼: `WAVE_ACTIVE`에서 다음 웨이브 미리 `run_wave` + `EconomyService.add_gold(early_call_bonus(남은 스폰 예정 수))`
  - `Hud` (`hud.tscn`, `CanvasLayer`) — 라벨(골드/기지HP/웨이브/경로길이), 빌드 메뉴(해금 타워 버튼), 웨이브 시작·조기 시작 버튼, 유물 픽 모달(3버튼), 승리/패배 패널+재시작. 시그널로 `Game`에 알림.

- [ ] **Step 1: `map_field.tscn` + `map_field.tres` 생성**

- `data/maps/map_field.tres` (`MapConfig`): `grid_size=(16,12)`, `cell_px=64`, `spawn_cell=(0,6)`, `base_cell=(15,6)`, `blocked_cells=[(7,2),(8,9)]`.
- `scenes/map/map_field.tscn`: 루트 `Node2D`(이름 `MapField`) → `TileMapLayer`(바닥 타일; 임시로 `ColorRect` 1024×768 어두운 회색으로 대체 가능) → `Marker2D`(이름 `SpawnMarker`, position `(32, 416)`) → `Area2D`(이름 `BaseZone`, position `(992, 416)`, 콜리전 레이어 4, 마스크 2) + `CollisionShape2D`(사각 64×64).

- [ ] **Step 2: `economy_tuning.tres` + `default_run.tres` 생성**

- `data/tuning/economy_tuning.tres` (`EconomyTuning`): 기본값(시작 골드 150, 기지 HP 20).
- `data/run/default_run.tres` (`RunConfig`):
  - `map_scene` = `map_field.tscn`, `map_config` = `map_field.tres`
  - `tower_pool` = 5개 타워 `.tres`
  - `relic_pool` = 6개 유물 `.tres`
  - `enemy_set` = 6개 적 `.tres`
  - `wave_tuning` = `wave_tuning.tres`, `economy_tuning` = `economy_tuning.tres`, `shop_tuning` = 새 `ShopTuning` 리소스(기본값)

- [ ] **Step 3: 통합 스모크 테스트 작성**

`test/unit/test_game_integration.gd`:

```gdscript
extends GutTest

const GameScene := preload("res://scenes/game/game.tscn")

func test_game_boots_into_build_phase():
	var g := GameScene.instantiate()
	add_child_autofree(g)
	await wait_frames(5)
	assert_eq(g.state.phase, GameState.Phase.BUILD, "부트 후 BUILD")
	assert_eq(g.state.wave_index, 1)
	assert_eq(g.economy.gold, 150)

func test_place_starter_tower_via_api():
	var g := GameScene.instantiate()
	add_child_autofree(g)
	await wait_frames(5)
	var bolt: TowerType = load("res://data/towers/bolt_tower.tres")
	var ok: bool = g.tower_container.try_place(bolt, Vector2i(4, 5), [])
	assert_true(ok)
	assert_lt(g.economy.gold, 150)
	assert_true(g.map_service.is_solid(Vector2i(4, 5)))
	assert_gt(g.map_service.path_length_cells(), 0, "경로 여전히 존재")

func test_full_wave_one_runs_and_returns_to_build():
	var g := GameScene.instantiate()
	add_child_autofree(g)
	await wait_frames(5)
	# 경로 위에 타워 몇 개 심어 딜 확보
	for c in [Vector2i(4, 5), Vector2i(6, 5), Vector2i(8, 5)]:
		g.tower_container.try_place(load("res://data/towers/bolt_tower.tres"), c, [])
	g.start_wave()
	assert_eq(g.state.phase, GameState.Phase.WAVE_ACTIVE)
	# 웨이브가 끝날 때까지 최대 40초 시뮬 (헤드리스는 빠름)
	var guard := 0
	while g.state.phase == GameState.Phase.WAVE_ACTIVE and guard < 2400:
		await wait_frames(1)
		guard += 1
	assert_eq(g.state.phase, GameState.Phase.BUILD)
	assert_eq(g.state.wave_index, 2)
```

- [ ] **Step 4: 테스트 실행 → 실패 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_game_integration.gd -gexit`
Expected: FAIL — `game.tscn` 없음.

- [ ] **Step 5: `game.gd` 구현**

`scripts/game.gd` (핵심 배선; 파일이 200줄 근처면 입력 처리를 `scripts/game_input.gd`로 분리):

```gdscript
class_name Game
extends Node2D

const RUN_CONFIG_PATH := "res://data/run/default_run.tres"

@onready var map_service: MapService = $Services/MapService
@onready var economy: EconomyService = $Services/EconomyService
@onready var wave_director: WaveDirector = $Services/WaveDirector
@onready var relic_service: RelicService = $Services/RelicService
@onready var tower_container: TowerContainer = $TowerContainer
@onready var enemy_container: Node2D = $EnemyContainer
@onready var projectile_parent: Node2D = $ProjectileContainer
@onready var hud: Hud = $Hud

var state: GameState
var _run: RunConfig
var _enemy_pool: Dictionary = {}
var _pool: ProjectilePool
var _selected_tower_type: TowerType
var _final_boss_killed: bool = false
var _run_seed: int = 0

func _ready() -> void:
	_run = load(RUN_CONFIG_PATH)
	_run_seed = randi()
	for e in _run.enemy_set:
		_enemy_pool[e.id] = e
	map_service.setup(_run.map_config)
	economy.setup(_run.economy_tuning)
	relic_service.setup(_run.relic_pool, _run_seed)
	_pool = ProjectilePool.new(load("res://scenes/entities/projectile.tscn"), projectile_parent, 64)
	tower_container.configure(load("res://scenes/entities/tower.tscn"), map_service, economy, _run.map_config.cell_px)
	wave_director.configure(load("res://scenes/entities/enemy.tscn"), enemy_container, func(): return map_service.current_path)

	add_child(load(_run.map_scene.resource_path).instantiate() if _run.map_scene else Node2D.new())

	_connect_signals()
	state = GameState.new(_run.economy_tuning, _run.wave_tuning.total_waves)
	hud.setup(_run)
	_offer_relic_pick(_run.shop_tuning.starting_relic_pick_count)
	state.begin_run()
	_refresh_hud()

func _connect_signals() -> void:
	wave_director.enemy_killed.connect(func(_e, bounty): economy.add_gold(bounty))
	wave_director.enemy_leaked.connect(func(_e, leak): economy.damage_base(leak))
	wave_director.wave_finished.connect(_on_wave_finished)
	economy.base_destroyed.connect(_on_base_destroyed)
	economy.gold_changed.connect(func(_g): _refresh_hud())
	economy.base_hp_changed.connect(func(_h, _m): _refresh_hud())
	map_service.path_recomputed.connect(_on_path_recomputed)
	tower_container.tower_placed.connect(func(t): t.arm(_pool, tower_container.towers))
	hud.build_tower_selected.connect(func(tt): _selected_tower_type = tt)
	hud.start_wave_pressed.connect(_on_start_wave_pressed)
	hud.early_call_pressed.connect(_on_early_call_pressed)
	hud.cell_tapped.connect(_on_cell_tapped)
	hud.restart_pressed.connect(func(): get_tree().reload_current_scene())

func _physics_process(_delta: float) -> void:
	for t in tower_container.towers():
		t.buff_mult = 1.0  # buff 타워가 이 프레임에 다시 칠함

func _on_path_recomputed(new_path: PackedVector2Array) -> void:
	for e in enemy_container.get_children():
		if e is Enemy:
			e.on_path_recomputed(new_path)
	_refresh_hud()

func _enemy_cells() -> Array:
	var out: Array = []
	for e in enemy_container.get_children():
		if e is Enemy:
			out.append(e.current_cell(_run.map_config.cell_px))
	return out

func _on_cell_tapped(cell: Vector2i) -> void:
	if state.phase == GameState.Phase.GAME_OVER or state.phase == GameState.Phase.RUN_WON:
		return
	for t in tower_container.towers():
		if t.cell == cell:
			hud.open_inspector(t)
			return
	if _selected_tower_type != null:
		var ok := tower_container.try_place(_selected_tower_type, cell, _enemy_cells())
		if not ok:
			hud.flash_placement_error(map_service.placement_block_reason(cell, _enemy_cells()))

func start_wave() -> void:
	_on_start_wave_pressed()

func _on_start_wave_pressed() -> void:
	if state.phase != GameState.Phase.BUILD:
		return
	state.start_wave()
	wave_director.run_wave(state.wave_index, _run.wave_tuning, _enemy_pool, _run_seed + state.wave_index)
	_check_final_boss_spawned()
	_refresh_hud()

func _on_early_call_pressed() -> void:
	if state.phase != GameState.Phase.WAVE_ACTIVE:
		return
	var pending := wave_director.alive_count
	economy.add_gold(state.early_call_bonus(pending))
	wave_director.run_wave(state.wave_index + 1, _run.wave_tuning, _enemy_pool, _run_seed + state.wave_index + 1)

func _check_final_boss_spawned() -> void:
	if WaveDirector.is_boss_wave(state.wave_index, _run.wave_tuning) and state.wave_index >= _run.wave_tuning.total_waves:
		# finalboss died → win
		for e in enemy_container.get_children():
			if e is Enemy:
				e.died.connect(func(_x): _final_boss_killed = true, CONNECT_ONE_SHOT)

func _on_wave_finished() -> void:
	economy.add_gold(state.wave_reward(state.wave_index))
	state.on_wave_finished(_final_boss_killed)
	match state.phase:
		GameState.Phase.RUN_WON:
			hud.show_win(state.wave_index)
		GameState.Phase.BUILD:
			if state.pending_relic_pick:
				_offer_relic_pick(_run.shop_tuning.relic_pick_count)
				state.consume_relic_pick()
	_refresh_hud()

func _on_base_destroyed() -> void:
	state.on_base_destroyed()
	hud.show_lose(state.wave_index)

func _offer_relic_pick(count: int) -> void:
	var choices := relic_service.roll_choices(count)
	if choices.is_empty():
		return
	var chosen: RelicType = await hud.prompt_relic_choice(choices)
	if chosen != null:
		relic_service.grant(chosen)

func _refresh_hud() -> void:
	hud.update_stats(economy.gold, economy.base_hp, economy.max_base_hp,
		state.wave_index, map_service.path_length_cells())
```

- [ ] **Step 6: `hud.gd` 구현 (골격) + `hud.tscn` 구성**

`scripts/hud/hud.gd`:

```gdscript
class_name Hud
extends CanvasLayer

signal build_tower_selected(tower_type: TowerType)
signal start_wave_pressed()
signal early_call_pressed()
signal cell_tapped(cell: Vector2i)
signal restart_pressed()

@onready var _stats_label: Label = $Root/TopBar/StatsLabel
@onready var _build_menu: HBoxContainer = $Root/BuildMenu
@onready var _relic_modal: Control = $Root/RelicModal
@onready var _end_panel: Control = $Root/EndPanel
@onready var _end_label: Label = $Root/EndPanel/Label
@onready var _error_label: Label = $Root/ErrorLabel

var _cell_px: int = 64
var _relic_result

func setup(run: RunConfig) -> void:
	_cell_px = run.map_config.cell_px
	for tt in run.tower_pool:
		if tt.starter_unlocked:
			_add_build_button(tt)
	_relic_modal.hide()
	_end_panel.hide()
	_error_label.text = ""

func add_unlocked_tower(tt: TowerType) -> void:  # M7에서 사용
	_add_build_button(tt)

func _add_build_button(tt: TowerType) -> void:
	var b := Button.new()
	b.text = "%s\n%dG" % [tt.display_name, tt.levels[0].upgrade_cost]
	b.pressed.connect(func(): build_tower_selected.emit(tt))
	_build_menu.add_child(b)

func update_stats(gold: int, hp: int, max_hp: int, wave: int, path_len: int) -> void:
	_stats_label.text = "골드 %d   기지 %d/%d   웨이브 %d   경로 %d칸" % [gold, hp, max_hp, wave, path_len]

func _unhandled_input(event: InputEvent) -> void:
	if event is InputEventScreenTouch and event.pressed:
		var cell := GridUtil.world_to_cell(event.position, _cell_px)
		cell_tapped.emit(cell)
	elif event is InputEventMouseButton and event.pressed and event.button_index == MOUSE_BUTTON_LEFT:
		cell_tapped.emit(GridUtil.world_to_cell(event.position, _cell_px))

func open_inspector(tower) -> void:
	# M6에서 완성; 지금은 판매만
	_error_label.text = "%s Lv%d (판매가 %dG)" % [tower.type.display_name, tower.level_index + 1, tower.sell_value()]

func flash_placement_error(reason: StringName) -> void:
	var msg := {
		&"would_seal_exit": "길을 완전히 막을 수 없습니다",
		&"occupied": "이미 타워가 있습니다",
		&"enemy_present": "적이 지나가는 중입니다",
		&"spawn_or_base": "여기엔 지을 수 없습니다",
		&"blocked": "지형이 막혀 있습니다",
		&"out_of_bounds": "",
	}.get(reason, "배치 불가")
	_error_label.text = msg

func prompt_relic_choice(choices: Array) -> RelicType:
	_relic_result = null
	for c in _relic_modal.get_node("Row").get_children():
		c.queue_free()
	for r in choices:
		var b := Button.new()
		b.text = "%s\n%s" % [r.display_name, r.description]
		b.custom_minimum_size = Vector2(240, 120)
		b.pressed.connect(func(): _relic_result = r; _relic_modal.hide())
		_relic_modal.get_node("Row").add_child(b)
	_relic_modal.show()
	while _relic_result == null and _relic_modal.visible:
		await get_tree().process_frame
	return _relic_result

func show_win(wave: int) -> void:
	_end_label.text = "승리! 웨이브 %d 클리어" % wave
	_end_panel.show()

func show_lose(wave: int) -> void:
	_end_label.text = "패배 — 웨이브 %d" % wave
	_end_panel.show()
```

`scenes/game/hud.tscn`: 루트 `CanvasLayer`(스크립트 `hud.gd`, 이름 `Hud`) → `Control`(이름 `Root`, 전체 앵커) 아래:
- `PanelContainer`(`TopBar`) → `Label`(`StatsLabel`)
- `HBoxContainer`(`BuildMenu`, 하단 좌측)
- `HBoxContainer`(하단 우측) → `Button`(`StartWaveButton`, `pressed` → `start_wave_pressed`), `Button`(`EarlyCallButton`, → `early_call_pressed`)
- `Label`(`ErrorLabel`, 중앙 상단)
- `Control`(`RelicModal`, 중앙, 반투명 배경) → `HBoxContainer`(`Row`)
- `Control`(`EndPanel`, 중앙) → `Label`(`Label`) + `Button`(`RestartButton`, → `restart_pressed`)

`hud.gd`의 `_ready`에서 `StartWaveButton`/`EarlyCallButton`/`RestartButton`의 `pressed`를 각 시그널로 연결하는 코드 추가:

```gdscript
func _ready() -> void:
	$Root/BottomRight/StartWaveButton.pressed.connect(func(): start_wave_pressed.emit())
	$Root/BottomRight/EarlyCallButton.pressed.connect(func(): early_call_pressed.emit())
	$Root/EndPanel/RestartButton.pressed.connect(func(): restart_pressed.emit())
```

- [ ] **Step 7: `game.tscn` 조립 + main scene 설정**

`scenes/game/game.tscn`: 루트 `Node2D`(스크립트 `game.gd`, 이름 `Game`):
- `Node`(`Services`) → 자식 `MapService`, `EconomyService`, `WaveDirector`, `RelicService` (각 스크립트 붙인 `Node`)
- `Node2D`(`TowerContainer`, 스크립트 `tower_container.gd`)
- `Node2D`(`EnemyContainer`)
- `Node2D`(`ProjectileContainer`)
- `Camera2D`(position `(512, 384)`)
- `Hud` (인스턴스 `hud.tscn`)

Project Settings > Application > Run > Main Scene = `res://scenes/game/game.tscn`.

- [ ] **Step 8: 통합 테스트 실행 → 통과 확인**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/test_game_integration.gd -gexit`
Expected: 3 passing. (타이밍 이슈 시 `while` 가드/`wait_frames` 조정.)

- [ ] **Step 9: 전체 스위트 회귀**

Run: `godot --headless -s addons/gut/gut_cmdln.gd -gdir=res://test/unit -gexit -glog=1`
Expected: 전 테스트 통과.

- [ ] **Step 10: 수동 플레이테스트 (에디터에서 F5) — M5 재미 중간 점검**

다음을 직접 확인하고 결과를 `docs/design/m5-playtest-notes.md`에 기록:

1. [ ] 부팅 시 시작 유물 3중 택1 모달이 뜨고, 하나 고르면 닫힌다.
2. [ ] 빌드 메뉴에 `bolt_tower`/`frost_totem` 2개만 보인다.
3. [ ] 타워 버튼 탭 → 빈 셀 탭 → 타워가 그리드에 스냅되어 배치되고 골드가 준다.
4. [ ] 타워로 경로를 우회시키면 **상단 "경로 N칸" 숫자가 늘어난다** (핵심 체감 지표).
5. [ ] 스폰~기지를 한 줄로 막으려 하면 배치가 거부되고 "길을 완전히 막을 수 없습니다" 문구가 뜬다.
6. [ ] "웨이브 시작" → 적이 스폰되어 **현재 미로 경로를 따라 돌아간다**.
7. [ ] 웨이브 중 타워를 추가하면 살아있는 적이 **즉시 새 경로로 우회**한다.
8. [ ] 타워가 사거리 내 적을 자동 조준·발사하고, 적이 죽으면 골드가 들어온다.
9. [ ] 적이 기지에 닿으면 기지 HP가 준다. 0이 되면 패배 화면.
10. [ ] 웨이브 클리어 시 보상 골드 + (5·10·… 웨이브면) 유물 픽 모달.
11. [ ] 웨이브 10에 미니보스가 섞여 나온다.
12. [ ] 조기 시작 버튼으로 다음 웨이브를 당기면 보너스 골드가 들어온다.

**재미 판정 (스펙 §12 중간 점검)** — 아래를 노트에 적는다:
- 미로 위치를 고민하게 되는가? (Y/N + 이유)
- 적이 빙 도는 걸 보는 것 자체가 만족스러운가? (Y/N)
- "봉쇄 불가" 제약이 답답한가 퍼즐인가?
→ 3개 중 2개 미달이면 **여기서 멈추고 코어 메커닉 재설계**. M6~M9 계획은 통과 후 별도 작성.

- [ ] **Step 11: 커밋**

```bash
git add scenes scripts data project.godot test/unit/test_game_integration.gd docs/design/m5-playtest-notes.md
git commit -m "feat: M5 통합 씬 + HUD + 플레이테스트 (재미 중간 점검 지점)"
```

---

## Self-Review (계획 작성자 체크)

**1. 스펙 커버리지 (M0–M5 범위)**
- §3 코어 루프·상태 머신 → Task 19 (`GameState`), Task 20 (배선). ✅ (조기 시작 = Task 20 Step 5)
- §4 맵/그리드/경로탐색 → Task 3, 4, 5, 6, 7. ✅ (배치 봉쇄 불가 = Task 5)
- §5 적/웨이브 → Task 6, 7, 8 (적), Task 10, 17 (웨이브). ✅ (보스 곡선 = Task 17)
- §6 타워/투사체/타게팅 → Task 11, 12, 13, 14, 15, 16. ✅ (5종 kind = Task 13, 16)
- §7 경제/상점/유물 → Task 9 (경제), Task 18 (유물 픽만). 상점·화염폭격·유물효과 = **M6~M9로 명시 연기**. ✅
- §9 데이터 스키마 → Task 2. ✅
- §10 터치 조작 → Task 20 (`hud.gd _unhandled_input`, `flash_placement_error`). 온보딩 툴팁 = M9. ✅
- §11 M0 → Task 1. ✅

**2. 플레이스홀더 스캔**: "TBD/적절히 처리" 없음. 모든 코드 스텝에 실제 GDScript. 밸런스 수치는 Task 8/16/17에 구체 표로 명시. ✅

**3. 타입 일관성 체크**:
- `MapService.is_placement_legal(cell, occupied_by_enemy: Array)` — Task 5 정의, Task 14/20에서 동일 시그니처로 호출. ✅
- `WaveDirector.configure(enemy_scene, enemy_container, get_path)` + `run_wave(wave_index, tuning, enemy_pool, seed)` — Task 10/17 정의, Task 20 호출 일치. ✅
- `Enemy.setup(type, path)` / `on_path_recomputed(new_path)` / `apply_slow(pct, duration_ms)` — Task 6/7 정의, Task 10/12/15/20 사용 일치. ✅
- `Tower.arm(pool, get_all_towers)` / `buff_mult` / `effective_damage()` — Task 14/15 정의, Task 20에서 `t.arm(_pool, tower_container.towers)` 및 프레임 리셋 일치. ✅
- `EconomyService` 메서드명 (`add_gold`, `try_spend`, `damage_base`, `airstrike_cost`) — Task 9 정의, Task 20 사용 일치. ✅
- `AttackResolver.resolve(kind, level, primary, in_range, all_towers)` — Task 13 정의, Task 15 호출 일치. `plan["projectiles"]`/`["instant_hits"]`/`["buff"]` 키 일치. ✅
- `GameState` API (`begin_run`, `start_wave`, `on_wave_finished(final_boss_killed)`, `pending_relic_pick`) — Task 19 정의, Task 20 사용 일치. ✅
- `Hud` 시그널 (`build_tower_selected`, `start_wave_pressed`, `early_call_pressed`, `cell_tapped`, `restart_pressed`) — Task 20 내부 정의·연결 일치. ✅

이슈 없음.

---

## Execution Handoff

계획 저장 위치: `docs/superpowers/plans/2026-09-01-maze-td-core-m0-m5.md`

**주의**: 이 계획은 M0–M5까지다. Task 20 Step 10의 재미 중간 점검을 통과하면 M6–M9(업그레이드 UI, 상점+해금, 화염폭격, 유물 효과, juice, 온보딩, 밸런싱) 계획을 별도로 작성한다.

**codex 위임 후보** (계약서 포함): Task 5(배치 합법성), Task 13(AttackResolver), Task 17(절차적 웨이브). 각 태스크의 "codex 계약서" 블록을 그대로 전달하고, 산출물을 해당 테스트로 검증한 뒤 통합.

두 가지 실행 방식:

1. **Subagent-Driven (권장)** — 태스크마다 새 서브에이전트 디스패치, 태스크 사이 리뷰, 빠른 이터레이션.
2. **Inline Execution** — 이 세션에서 `executing-plans`로 체크포인트 배치 실행.

어느 쪽으로 진행할까요?
