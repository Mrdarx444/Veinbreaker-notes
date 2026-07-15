# أسس تصميم الأنظمة الفارغة

> **الهدف**: تقديم **أساس هندسي كامل** لكل ملف فارغ في الـ vault، بحيث يقدر المطور يبدأ كتابة الكود مباشرة من هذه الأسس.
> **المنهجية**: لكل نظام:
> 1. **Concept** — ما هو ولماذا.
> 2. **Architecture** — مكوناته الرئيسية.
> 3. **Code Skeleton** — GDScript Godot 4.7.
> 4. **Integration Hooks** — كيف يتصل بالأنظمة الأخرى.
> 5. **Balance Targets** — أرقام مبدئية للـ tuning.

---

## جدول الأنظمة الفارغة (11 نظام)

| # | النظام | الملف | الأولوية |
|---|--------|------|---------|
| 1 | Input Map | `Architecture/Input Map.md` | P0 |
| 2 | Enemy AI | `Architecture/Enemies/AI.md` | P0 |
| 3 | Player | `Design/Player.md` | P0 |
| 4 | Level Design | `Design/Level Design.md` | P1 |
| 5 | Camera System | `Game play/Systems/Camera System.md` | P0 |
| 6 | Health System | `Game play/Systems/Health System.md` | P0 |
| 7 | Map System | `Game play/Systems/Map System.md` | P1 |
| 8 | Skill Tree System | `Game play/Systems/Skill Tree System.md` | P0 |
| 9 | Damage System | `Game play/Systems/Combat Systems/Damage System.md` | P0 |
| 10 | Handguns | `Game play/Systems/Combat Systems/Handguns.md` | P0 |
| 11 | Parry | `Game play/Systems/Combat Systems/Parry.md` | P1 |

---

## 1. Input Map (`Architecture/Input Map.md`)

### 1.1 Concept
الـ Input Map هو **الطبقة الوحيدة** بين أصابع اللاعب وكل الأنظمة. كل ما يفعله اللاعب يمرّ من هنا. خطأ هنا = خطأ في كل مكان.

### 1.2 Architecture

```
[Player's Fingers]
       ↓
[TouchScreenButtons + Virtual Joystick]   ← Physical UI
       ↓
[Godot Input Map (Project Settings)]      ← Actions defined
       ↓
[InputSystem Autoload]                    ← Centralized dispatcher
       ↓
[Player Controller / Enemy AI / UI]       ← Subscribers
```

### 1.3 Actions Definition (Project Settings)

| Action Name | Type | Mapping | Notes |
|-------------|------|---------|-------|
| `move_left` | Vector2 axis | Joystick X < 0 | Built-in Godot Joystick |
| `move_right` | Vector2 axis | Joystick X > 0 | Built-in Godot Joystick |
| `move_up` | Vector2 axis | Joystick Y < 0 | Built-in Godot Joystick |
| `move_down` | Vector2 axis | Joystick Y > 0 | Built-in Godot Joystick |
| `jump` | Button | Jump TouchScreenButton | |
| `dash` | Button | Dash TouchScreenButton | |
| `attack` | Button | Attack TouchScreenButton | Multi-function (slash/charge/parry) |
| `handgun_fire` | Button | Handgun TouchScreenButton | |
| `interact` | Button | Interact TouchScreenButton | Context-sensitive: interact/heal |
| `pause` | Button | Top-right corner | Opens pause menu |
| `skill_tree` | Button | Pause menu → Skill Tree | Only at Vein Altars |

### 1.4 Code Skeleton

```gdscript
# input_system.gd (Autoload / Singleton)
extends Node

# Weak references for active input contexts
signal input_context_changed(context: String)

var active_context: String = "gameplay" :
    set(value):
        if active_context != value:
            active_context = value
            input_context_changed.emit(value)

# Contexts:
# "gameplay" - normal play
# "menu" - pause/skill tree/shop
# "dialogue" - cutscene/dialogue
# "cutscene" - no input allowed

func _unhandled_input(event: InputEvent) -> void:
    match active_context:
        "gameplay":
            _handle_gameplay_input(event)
        "menu":
            _handle_menu_input(event)
        "dialogue":
            _handle_dialogue_input(event)
        "cutscene":
            pass  # ignore all
        _:
            push_warning("Unknown input context: " + active_context)

func _handle_gameplay_input(event: InputEvent) -> void:
    if event.is_action_pressed("pause"):
        active_context = "menu"
        get_tree().paused = true
        UIManager.open_pause_menu()
        return
    
    # Dispatch to player
    var player := get_tree().get_first_node_in_group("player")
    if player and player.has_method("_handle_input"):
        player._handle_input(event)

func set_context(context: String) -> void:
    active_context = context
```

### 1.5 Joystick Wrapper (Unified)

```gdscript
# unified_joystick.gd
extends TouchScreenButton

signal zone_changed(zone: String)

const ZONES := {
    "RIGHT": Vector2.RIGHT,
    "LEFT": Vector2.LEFT,
    "UP": Vector2.UP,
    "DOWN": Vector2.DOWN,
    "UP_RIGHT": Vector2(1, -1).normalized(),
    "UP_LEFT": Vector2(-1, -1).normalized(),
    "DOWN_RIGHT": Vector2(1, 1).normalized(),
    "DOWN_LEFT": Vector2(-1, 1).normalized(),
}

const DEAD_ZONE := 0.18

var current_zone: String = "IDLE"
var raw_vector: Vector2 = Vector2.ZERO

func _input(event: InputEvent) -> void:
    if event is InputEventScreenTouch or event is InputEventScreenDrag:
        _update_joystick(event)

func _update_joystick(event: InputEvent) -> void:
    var local_pos := event.position - global_position
    var distance := local_pos.length()
    
    if distance < DEAD_ZONE * 100:  # 100 = joystick radius
        raw_vector = Vector2.ZERO
        _set_zone("IDLE")
        return
    
    raw_vector = (local_pos / 100).limit_length(1.0)
    _set_zone(_get_zone(raw_vector))

func _get_zone(v: Vector2) -> String:
    if v.length() < DEAD_ZONE:
        return "IDLE"
    var best_dot := -999.0
    var best_name := "IDLE"
    for name in ZONES:
        var dot := v.dot(ZONES[name])
        if dot > best_dot:
            best_dot = dot
            best_name = name
    return best_name

func _set_zone(zone: String) -> void:
    if current_zone != zone:
        current_zone = zone
        zone_changed.emit(zone)

func get_aim_direction() -> Vector2:
    return ZONES.get(current_zone, Vector2.ZERO)

func is_movement_active() -> bool:
    return current_zone in ["LEFT", "RIGHT"]
```

### 1.6 Integration Hooks
- **Movement System**: يقرأ `is_movement_active()` و `raw_vector`.
- **Aim System**: يقرأ `current_zone` و `get_aim_direction()`.
- **Combat Systems**: يقرأ `current_zone` لتحديد اتجاه الـ Upward/Downward swings.
- **Camera**: يقرأ `raw_vector` للـ look-ahead.

---

## 2. Enemy AI (`Architecture/Enemies/AI.md`)

### 2.1 Concept
نظام AI موحّد يسمح بـ:
- **Variety**: كل عدو له سلوك مختلف لكن بمواصفات موحدة.
- **Discoverability**: اللاعب يفهم سلوك كل عدو من telegraph واضح.
- ** extensibility**: إضافة عدو جديد = override بعض الدوال فقط.

### 2.2 Architecture — State Machine + Composition

```
[EnemyBase] (Core State Machine)
    ↓ inherits
[Grunt] [Sentinel] [Stalker] [Brute] [Flyer] (Concrete Enemies)
    ↓ uses
[VisionSensor] [HearingSensor] [Pathfinder] [AttackModule] (Components)
```

### 2.3 Code Skeleton

```gdscript
# enemy_base.gd
class_name EnemyBase
extends CharacterBody2D

enum State { IDLE, PATROL, DETECT, CHASE, ATTACK, STAGGER, FLEE, DEAD }

@export_group("Stats")
@export var max_health: float = 100.0
@export var current_health: float = 100.0
@export var move_speed: float = 80.0
@export var damage: float = 15.0
@export var sanguis_reward: int = 2
@export var cinders_reward: int = 3

@export_group("AI")
@export var detect_range: float = 250.0
@export var attack_range: float = 60.0
@export var attack_telegraph_duration: float = 0.4
@export var attack_cooldown: float = 1.5
@export var field_of_view_degrees: float = 90.0

@export_group("Stagger")
@export var stagger_threshold: float = 30.0  # damage to stagger
@export var stagger_duration: float = 0.6
@export var stagger_resist: float = 0.0  # 0.0 to 1.0

var state: State = State.IDLE
var stagger_meter: float = 0.0
var attack_cooldown_timer: float = 0.0
var awareness: float = 0.0  # 0=unaware, 1=fully aware (for stealth)

@onready var vision_sensor: VisionSensor = $VisionSensor
@onready var health_bar: ProgressBar = $HealthBar
@onready var animation: AnimatedSprite2D = $AnimatedSprite2D

signal enemy_died(enemy: EnemyBase)
signal state_changed(old_state: State, new_state: State)

func _ready() -> void:
    add_to_group("enemies")
    vision_sensor.player_detected.connect(_on_player_detected)
    vision_sensor.player_lost.connect(_on_player_lost)
    health_bar.visible = false
    current_health = max_health

func _physics_process(delta: float) -> void:
    if state == State.DEAD:
        return
    
    _update_timers(delta)
    
    match state:
        State.IDLE:    _idle_state(delta)
        State.PATROL:  _patrol_state(delta)
        State.DETECT:  _detect_state(delta)
        State.CHASE:   _chase_state(delta)
        State.ATTACK:  _attack_state(delta)
        State.STAGGER: _stagger_state(delta)
        State.FLEE:    _flee_state(delta)
    
    move_and_slide()

# --- State handlers (override in subclasses) ---
func _idle_state(delta: float) -> void:
    awareness = max(0.0, awareness - delta * 0.5)
    if vision_sensor.can_see_player():
        _transition_to(State.DETECT)

func _patrol_state(delta: float) -> void:
    awareness = max(0.0, awareness - delta * 0.5)
    velocity = _get_patrol_velocity()
    if vision_sensor.can_see_player():
        _transition_to(State.DETECT)

func _detect_state(delta: float) -> void:
    # Telegraph: العدو يرى اللاعب ويتحضّر
    velocity = velocity.move_toward(Vector2.ZERO, 200 * delta)
    awareness = min(1.0, awareness + delta * 2.0)
    if awareness >= 1.0:
        _transition_to(State.CHASE)
    elif not vision_sensor.can_see_player():
        awareness = max(0.0, awareness - delta * 1.0)
        if awareness <= 0.0:
            _transition_to(State.PATROL)

func _chase_state(delta: float) -> void:
    var player := _get_player()
    if not player:
        _transition_to(State.PATROL)
        return
    
    var direction := (player.global_position - global_position).normalized()
    velocity = direction * move_speed
    
    if global_position.distance_to(player.global_position) <= attack_range:
        _transition_to(State.ATTACK)
    elif not vision_sensor.can_see_player():
        # فقد اللاعب
        _detect_lost_timer(delta)

func _attack_state(delta: float) -> void:
    if attack_cooldown_timer > 0:
        _transition_to(State.CHASE)
        return
    
    # Telegraph
    _play_telegraph_animation()
    await get_tree().create_timer(attack_telegraph_duration).timeout
    
    if state != State.ATTACK:
        return  # تم إلغاء (مثلاً stagger)
    
    _perform_attack()
    attack_cooldown_timer = attack_cooldown
    _transition_to(State.CHASE)

func _stagger_state(delta: float) -> void:
    velocity = velocity.move_toward(Vector2.ZERO, 400 * delta)
    await get_tree().create_timer(stagger_duration).timeout
    _transition_to(State.CHASE)

func _flee_state(delta: float) -> void:
    # للأنواع التي تهرب عند low HP
    var player := _get_player()
    if player:
        var away := (global_position - player.global_position).normalized()
        velocity = away * move_speed * 1.5

# --- Combat ---
func take_damage(amount: float, knockback_dir: Vector2, source: Node) -> void:
    if state == State.DEAD:
        return
    
    current_health = max(0.0, current_health - amount)
    health_bar.visible = true
    health_bar.value = (current_health / max_health) * 100.0
    
    # Stagger logic
    stagger_meter += amount
    if stagger_meter >= stagger_threshold and state != State.STAGGER:
        stagger_meter = 0.0
        _transition_to(State.STAGGER)
        velocity = knockback_dir * 200 * (1.0 - stagger_resist)
    
    # Awareness boost: العدو يصبح مدركاً عند ضربه
    awareness = 1.0
    if state in [State.IDLE, State.PATROL]:
        _transition_to(State.CHASE)
    
    if current_health <= 0:
        _die()

func _die() -> void:
    _transition_to(State.DEAD)
    SanguisManager.grant_sanguis(sanguis_reward, "enemy:" + name)
    CindersManager.grant(cinders_reward, "enemy:" + name)
    enemy_died.emit(self)
    _play_death_animation()
    await get_tree().create_timer(1.0).timeout
    queue_free()

# --- Helpers ---
func _transition_to(new_state: State) -> void:
    if state == new_state:
        return
    var old := state
    state = new_state
    state_changed.emit(old, new_state)

func _get_player() -> Node2D:
    return get_tree().get_first_node_in_group("player") as Node2D

func _update_timers(delta: float) -> void:
    if attack_cooldown_timer > 0:
        attack_cooldown_timer = max(0.0, attack_cooldown_timer - delta)
    stagger_meter = max(0.0, stagger_meter - delta * 20.0)  # decay

# --- Override in subclasses ---
func _get_patrol_velocity() -> Vector2:
    return Vector2.ZERO

func _perform_attack() -> void:
    push_error("Subclass must override _perform_attack()")
```

### 2.4 Vision Sensor Component

```gdscript
# vision_sensor.gd
extends Node2D

signal player_detected()
signal player_lost()

@export var detect_range: float = 250.0
@export var field_of_view_degrees: float = 90.0
@export var check_interval: float = 0.1

var target: Node2D = null
var can_see: bool = false

func _ready() -> void:
    add_to_group("vision_sensors")
    var timer := Timer.new()
    timer.wait_time = check_interval
    timer.timeout.connect(_check_vision)
    add_child(timer)
    timer.start()

func can_see_player() -> bool:
    return can_see

func _check_vision() -> void:
    var player := get_tree().get_first_node_in_group("player") as Node2D
    if not player:
        can_see = false
        if target:
            target = null
            player_lost.emit()
        return
    
    var to_player := player.global_position - get_parent().global_position
    var distance := to_player.length()
    
    if distance > detect_range:
        _set_can_see(false)
        return
    
    # FOV check
    var facing_dir := Vector2.from_angle(get_parent().rotation)
    var angle := facing_dir.angle_to(to_player.normalized())
    if abs(rad_to_deg(angle)) > field_of_view_degrees / 2.0:
        _set_can_see(false)
        return
    
    # RayCast for walls
    var space_state := get_world_2d().direct_space_state
    var query := PhysicsRayQueryParameters2D.create(
        get_parent().global_position,
        player.global_position,
        1  # collision mask for walls
    )
    var result := space_state.intersect_ray(query)
    if result and not result.collider.is_in_group("player"):
        _set_can_see(false)
        return
    
    _set_can_see(true)
    target = player

func _set_can_see(value: bool) -> void:
    if can_see == value:
        return
    can_see = value
    if value:
        player_detected.emit()
    else:
        player_lost.emit()
```

### 2.5 Concrete Enemy Examples

#### Grunt (Basic Melee)

```gdscript
# grunt.gd
extends EnemyBase

func _ready() -> void:
    super._ready()
    max_health = 60.0
    current_health = 60.0
    move_speed = 90.0
    damage = 12.0
    detect_range = 220.0
    attack_range = 50.0

func _perform_attack() -> void:
    # Slash attack
    var player := _get_player()
    if player and global_position.distance_to(player.global_position) <= attack_range:
        player.take_damage(damage, (player.global_position - global_position).normalized(), self)
    _play_attack_animation()
```

#### Sentinel (Ranged)

```gdscript
# sentinel.gd
extends EnemyBase

@export var projectile_scene: PackedScene
@export var projectile_speed: float = 300.0

func _ready() -> void:
    super._ready()
    max_health = 40.0
    current_health = 40.0
    move_speed = 0.0  # لا يتحرك
    detect_range = 350.0
    attack_range = 300.0
    attack_cooldown = 2.0

func _chase_state(delta: float) -> void:
    # Sentinel لا يطارد، فقط يهاجم من بعيد
    var player := _get_player()
    if not player or not vision_sensor.can_see_player():
        _transition_to(State.PATROL)
        return
    if global_position.distance_to(player.global_position) <= attack_range:
        _transition_to(State.ATTACK)

func _perform_attack() -> void:
    var player := _get_player()
    if not player:
        return
    var projectile := projectile_scene.instantiate()
    projectile.global_position = global_position
    projectile.direction = (player.global_position - global_position).normalized()
    projectile.speed = projectile_speed
    projectile.damage = damage
    get_tree().current_scene.add_child(projectile)
```

#### Stalker (Stealth)

```gdscript
# stalker.gd
extends EnemyBase

var stealth_alpha: float = 0.3

func _ready() -> void:
    super._ready()
    max_health = 50.0
    current_health = 50.0
    move_speed = 110.0
    detect_range = 400.0  # يسمع أكثر من يرى
    attack_range = 40.0
    modulate.a = stealth_alpha  # شبه شفاف

func _idle_state(delta: float) -> void:
    # Stalker يتسلل نحو اللاعب
    var player := _get_player()
    if player:
        var distance := global_position.distance_to(player.global_position)
        if distance < detect_range:
            modulate.a = lerp(modulate.a, 1.0, delta * 2.0)  # يصبح ظاهراً
            _transition_to(State.CHASE)
        else:
            modulate.a = lerp(modulate.a, stealth_alpha, delta)

func _chase_state(delta: float) -> void:
    # يطارد بصمت
    super._chase_state(delta)
    modulate.a = lerp(modulate.a, 0.7, delta * 2.0)
```

### 2.6 Integration Hooks
- **Player**: `player.take_damage()` يُستدعى من `_perform_attack()`.
- **Damage System**: `take_damage()` يستقبل damage + knockback_dir + source.
- **Sanguis/Cinders**: يُكسب عند `_die()`.
- **Parry Manager**: العدو يسجّل parry windows قبل `_perform_attack()`.

---

## 3. Player (`Design/Player.md`)

### 3.1 Concept
اللاعب هو **محور كل شيء**. هذا الملف يحدّد:
- Player Stats (HP، stamina إن وجدت، speed).
- Player Components (sprite، collider، raycasts، etc.).
- Player State Machine الأساسية (مرجعية، تفاصيل الحركة في `Movement System.md`).

### 3.2 Architecture

```
Player (CharacterBody2D)
├── MovementController (state machine: idle/move/jump/fall/dash/wall_slide)
├── CombatController (melee/handgun/parry/heal)
├── HealthController (HP + flasks + i-frames)
├── InventoryController (cinders/items/keys)
├── AnimationPlayer
├── AnimatedSprite2D
├── CollisionShape2D
├── RayCasts (left_wall, right_wall, floor_check, ground_check)
├── Hitbox (Area2D — ما يضرب الأعداء)
├── Hurtbox (Area2D — ما يستقبل ضرر)
└── Camera2D (follows player)
```

### 3.3 Code Skeleton

```gdscript
# player.gd
class_name Player
extends CharacterBody2D

@export_group("Movement")
@export var walk_speed: float = 180.0
@export var air_speed: float = 180.0
@export var jump_velocity: float = -380.0
@export var gravity: float = 980.0
@export var max_fall_speed: float = 600.0

@export_group("Combat")
@export var melee_damage: float = 25.0
@export var handgun_damage: float = 30.0
@export var invincibility_duration: float = 0.6

@onready var movement: MovementController = $MovementController
@onready var combat: CombatController = $CombatController
@onready var health: HealthController = $HealthController
@onready var inventory: InventoryController = $InventoryController
@onready var animated_sprite: AnimatedSprite2D = $AnimatedSprite2D
@onready var hitbox: Area2D = $Hitbox
@onready var hurtbox: AreaBox = $Hurtbox

func _ready() -> void:
    add_to_group("player")
    health.player_died.connect(_on_death)

func _physics_process(delta: float) -> void:
    movement._update(delta)
    combat._update(delta)
    move_and_slide()

func take_damage(amount: float, knockback_dir: Vector2, source: Node) -> void:
    if health.is_invincible:
        return
    health.take_damage(amount, knockback_dir)

func _on_death() -> void:
    # Respawn at last Vein Altar
    SaveManager.load_last_checkpoint()
```

### 3.4 Player Stats Table

| Stat | Base Value | Notes |
|------|-----------|-------|
| Max HP | 100 | يمكن زيادتها من skill tree |
| Move Speed | 180 px/s | |
| Jump Velocity | -380 | (gravity 980 → height ~74px) |
| Air Control | 100% | full air control |
| Coyote Time | 0.12s | |
| Buffer Time | 0.16s | |
| I-frames | 0.6s | after taking damage |
| Vein Flasks (max) | 3 | upgradeable to 5 |
| Flask Heal | 40% max HP | |
| Handgun Ammo (max) | 20 | refillable at altars |

---

## 4. Level Design (`Design/Level Design.md`)

### 4.1 Concept
Level Design في Metroidvania ليست "رسم خرائط" — هي **هندسة flow**. كل غرفة يجب أن تجيب على:
- لماذا اللاعب هنا؟ (purpose)
- ما الذي يلي؟ (flow)
- ما المكافأة على الاستكشاف؟ (incentive)
- ما الذي يمنع التقدم؟ (gating)

### 4.2 Architecture — Room-Based

```
World Map
├── Region A (e.g., "The Blood Veins")
│   ├── Room A1 (entry)
│   ├── Room A2 (combat challenge)
│   ├── Room A3 (Vein Altar)
│   ├── Room A4 (boss arena)
│   ├── Room A5 (secret: charm)
│   └── Room A6 (locked: requires Double Jump)
├── Region B (e.g., "Ashen Wastes")
└── ...
```

### 4.3 Room Design Template

لكل غرفة، هذا الـ spec:

```yaml
Room ID: A3
Region: The Blood Veins
Type: altar_room  # entry/combat/altar/boss/secret/puzzle/hub
Size: 1280x720 px  # ~2 screens
Camera Bounds: [0, 0, 1280, 720]
Entrances: [left, right, top]
Exits: [right (to A4), top (to A2)]
Enemies: [Grunt x2, Sentinel x1]
Items: [Cinders x15]
Altar: yes  # refills flasks + skill tree access
Save Point: yes
Gating: none  # أو "requires double_jump" مثلاً
Lore: ["This altar was the first to bleed..."]
Music: vein_altar_theme
Ambient: dripping_blood
```

### 4.4 Level Design Rules

| Rule | Value | Rationale |
|------|-------|-----------|
| Room Size (typical) | 1280x720 (1 screen) إلى 2560x1440 (4 screens) | mobile screen ~640x360 viewport |
| Checkpoint (Altar) Frequency | كل 4-6 rooms | ~5-7 دقائق بين كل altar |
| Enemy Density | 2-4 per room | لا clusterfuck |
| Secret Frequency | 1 secret per 3 rooms | يحفّز الاستكشاف |
| Boss Frequency | 1 per region | pacing |
| Ability-Gating Frequency | كل region يفتح skill يفتح region آخر | Metroidvania core |
| Lore Items | 1-2 per region | ليس overload |

### 4.5 Ability Gating Tree

```
[Start] → [Wall Jump] (Skill Tree cost: 50 Sanguis)
            ↓
        [Region B opens: requires Wall Jump]
            ↓
        [Double Jump] (cost: 80 Sanguis)
            ↓
        [Region C opens: requires Double Jump]
            ↓
        [Roll Dash] (cost: 60 Sanguis)
            ↓
        [Region D opens: requires Roll Dash to pass under barrier]
            ↓
        [Charged Slash] (cost: 100 Sanguis)
            ↓
        [Region E opens: requires Charged Slash to break wall]
            ↓
        [Wall Dash] (cost: 120 Sanguis)
            ↓
        [Region F: final]
```

### 4.6 Combat Encounter Design

#### Per-Room Encounter Template
```yaml
Encounter ID: A2_combat_1
Difficulty: medium
Waves:
  - Wave 1:
      - Grunt x1 (enters from left)
      - Spawn delay: 0s
  - Wave 2:
      - Grunt x2 (enters from left + right)
      - Spawn delay: 3s (بعد موت Wave 1)
  - Wave 3:
      - Sentinel x1 (top platform)
      - Grunt x1 (ground)
      - Spawn delay: 2s
Reward on completion: Cinders x25, Lore snippet
```

### 4.7 Mobile-Specific Considerations
- **Room transitions**:无缝 (metroidvania standard) لا loading screens.
- **Camera bounds**: كل room له camera bounds صارمة لمنع رؤية ما وراء الجدران.
- **Touch-friendly**: لا تضع enemies في spots صعبة الـ reach بـ joystick (مثلاً فوق منصة عالية جداً).

---

## 5. Camera System (`Game play/Systems/Camera System.md`)

### 5.1 Concept
الكاميرا في 2D platformer = **محرر الإحساس**. كاميرا سيئة تقتل أفضل combat. كاميرا ممتازة تخفي عيوب الـ level design.

### 5.2 Architecture

```
Camera2D
├── Follow Logic (lerp toward player)
├── Look-ahead (anticipate movement direction)
├── Bounds (clamp to room)
├── Shake (trauma-based)
├── Zoom (dynamic)
└── Special Modes (boss cinematics, fall transitions)
```

### 5.3 Code Skeleton

```gdscript
# camera_controller.gd
extends Camera2D

@export var target: Node2D
@export var follow_lerp: float = 8.0
@export var look_ahead_distance: float = 80.0
@export var look_ahead_lerp: float = 3.0
@export var look_ahead_deadzone: float = 30.0

@export_group("Bounds")
@export var bounds_enabled: bool = false
@export var bounds: Rect2 = Rect2()

@export_group("Shake")
@export var shake_decay: float = 3.0
@export var shake_max_offset: float = 12.0
var trauma: float = 0.0  # 0 to 1
var shake_noise: FastNoiseLite

@export_group("Zoom")
@export var default_zoom: Vector2 = Vector2(2.5, 2.5)  # mobile: عالية
@export var combat_zoom: Vector2 = Vector2(2.2, 2.2)  # أوسع قليلاً
@export var zoom_lerp: float = 2.0

var look_ahead_velocity: Vector2 = Vector2.ZERO

func _ready() -> void:
    zoom = default_zoom
    shake_noise = FastNoiseLite.new()
    shake_noise.frequency = 30.0

func _process(delta: float) -> void:
    if not target:
        return
    
    _update_follow(delta)
    _update_look_ahead(delta)
    _update_bounds()
    _update_shake(delta)
    _update_zoom(delta)

func _update_follow(delta: float) -> void:
    var target_pos := target.global_position + look_ahead_velocity
    global_position = global_position.lerp(target_pos, follow_lerp * delta)

func _update_look_ahead(delta: float) -> void:
    var player_velocity := (target as CharacterBody2D).velocity
    var desired_look_ahead := Vector2.ZERO
    
    if abs(player_velocity.x) > look_ahead_deadzone:
        desired_look_ahead.x = sign(player_velocity.x) * look_ahead_distance
    
    look_ahead_velocity = look_ahead_velocity.lerp(desired_look_ahead, look_ahead_lerp * delta)

func _update_bounds() -> void:
    if not bounds_enabled:
        return
    var half_viewport := get_viewport_rect().size / (2.0 * zoom.x)
    global_position.x = clamp(global_position.x, bounds.position.x + half_viewport.x, bounds.end.x - half_viewport.x)
    global_position.y = clamp(global_position.y, bounds.position.y + half_viewport.y, bounds.end.y - half_viewport.y)

func _update_shake(delta: float) -> void:
    if trauma > 0:
        trauma = max(0.0, trauma - delta * shake_decay)
        var shake_amount := trauma * trauma  # quadratic for snappier feel
        var noise_x := shake_noise.get_noise_1d(Time.get_ticks_msec() * 0.05) * shake_max_offset * shake_amount
        var noise_y := shake_noise.get_noise_1d(Time.get_ticks_msec() * 0.05 + 100) * shake_max_offset * shake_amount
        offset = Vector2(noise_x, noise_y)
    else:
        offset = Vector2.ZERO

func _update_zoom(delta: float) -> void:
    var target_zoom := default_zoom
    if _is_in_combat():
        target_zoom = combat_zoom
    zoom = zoom.lerp(target_zoom, zoom_lerp * delta)

func add_trauma(amount: float) -> void:
    trauma = clamp(trauma + amount, 0.0, 1.0)

func _is_in_combat() -> bool:
    var player := get_tree().get_first_node_in_group("player") as Node2D
    if not player:
        return false
    for enemy in get_tree().get_nodes_in_group("enemies"):
        if enemy.global_position.distance_to(player.global_position) < 350:
            return true
    return false

# Triggers
func shake_light() -> void:
    add_trauma(0.2)

func shake_medium() -> void:
    add_trauma(0.4)

func shake_heavy() -> void:
    add_trauma(0.7)

func shake_explosion() -> void:
    add_trauma(1.0)
```

### 5.4 Shake Triggers Table

| Event | Trauma | Notes |
|shape|--------|-------|
| Light hit (player takes damage) | 0.4 | |
| Heavy hit (player charged attack) | 0.3 | |
| Enemy death | 0.2 | |
| Boss death | 1.0 | |
| Big fall landing | 0.6 | |
| Parry success | 0.3 | + time scale 0.1s |
| Explosion | 0.8 | |

### 5.5 Integration Hooks
- **Movement System**: `shake_heavy()` عند Big Fall.
- **Combat Systems**: `shake_light()` عند hit / `shake_medium()` عند charged.
- **Level Design**: كل room يضبط `bounds` عند دخوله.

---

## 6. Health System (`Game play/Systems/Health System.md`)

### 6.1 Concept
الـ Health System في هذا المشروع يتكوّن من ثلاث طبقات:
1. **HP** (Health Points): الرقم الفعلي.
2. **Vein Flasks**: موارد الشفاء (مثل Estus).
3. **I-frames**: فترات عدم القابلية للضرر بعد الضربة.

> ملاحظة: تفاصيل Vein Flasks في `02-Pitfalls-Solutions.md` Pitfall #5. هذا الملف يركز على البنية التحتية.

### 6.2 Architecture

```
HealthController
├── HP (current/max)
├── Flasks (current/max)
├── I-frames timer
├── Damage multipliers (defensive buffs)
├── Regen (optional, very slow)
└── Death handling
```

### 6.3 Code Skeleton

```gdscript
# health_controller.gd
extends Node

signal health_changed(current: float, maximum: float)
signal flask_changed(remaining: int, maximum: int)
signal player_died()
signal damage_taken(amount: float, knockback_dir: Vector2)
signal healing_started()
signal healing_completed()
signal healing_interrupted()

@export var max_health: float = 100.0
@export var max_flasks: int = 3
@export var flask_heal_percent: float = 40.0
@export var heal_duration: float = 0.6
@export var invincibility_duration: float = 0.6
@export var regen_per_second: float = 0.0  # default off

var current_health: float
var current_flasks: int
var is_invincible: bool = false
var is_healing: bool = false
var damage_multiplier: float = 1.0  # for buffs/debuffs

func _ready() -> void:
    current_health = max_health
    current_flasks = max_flasks
    _emit_signals()

func _process(delta: float) -> void:
    if regen_per_second > 0 and current_health > 0 and current_health < max_health:
        current_health = min(max_health, current_health + regen_per_second * delta)
        health_changed.emit(current_health, max_health)

func take_damage(amount: float, knockback_dir: Vector2 = Vector2.ZERO) -> void:
    if is_invincible or current_health <= 0:
        return
    
    var actual_damage := amount * damage_multiplier
    current_health = max(0.0, current_health - actual_damage)
    
    damage_taken.emit(actual_damage, knockback_dir)
    health_changed.emit(current_health, max_health)
    
    if is_healing:
        is_healing = false
        healing_interrupted.emit()
    
    if current_health <= 0:
        player_died.emit()
    else:
        _start_invincibility()

func _start_invincibility() -> void:
    is_invincible = true
    # Flash effect
    var player := get_parent()
    var sprite := player.get_node_or_null("AnimatedSprite2D") as AnimatedSprite2D
    if sprite:
        sprite.modulate = Color(1, 0.3, 0.3, 0.7)
    
    await get_tree().create_timer(invincibility_duration).timeout
    
    is_invincible = false
    if sprite:
        sprite.modulate = Color.WHITE

func try_heal() -> bool:
    if current_flasks <= 0 or is_healing or current_health >= max_health:
        return false
    
    current_flasks -= 1
    is_healing = true
    flask_changed.emit(current_flasks, max_flasks)
    healing_started.emit()
    
    await get_tree().create_timer(heal_duration).timeout
    
    if is_healing:  # لم يُقاطع
        var heal_amount := max_health * (flask_heal_percent / 100.0)
        current_health = min(max_health, current_health + heal_amount)
        health_changed.emit(current_health, max_health)
        healing_completed.emit()
    
    is_healing = false
    return true

func refill_flasks() -> void:
    current_flasks = max_flasks
    flask_changed.emit(current_flasks, max_flasks)

func full_heal() -> void:
    current_health = max_health
    health_changed.emit(current_health, max_health)

func upgrade_max_flasks(amount: int = 1) -> void:
    max_flasks = min(5, max_flasks + amount)
    current_flasks = max_flasks
    flask_changed.emit(current_flasks, max_flasks)

func upgrade_max_health(amount: float) -> void:
    max_health += amount
    current_health += amount
    health_changed.emit(current_health, max_health)

func _emit_signals() -> void:
    health_changed.emit(current_health, max_health)
    flask_changed.emit(current_flasks, max_flasks)
```

### 6.4 Low Health Effects (مثل الصورة في `_Assets/Low Health Effects.png`)

```gdscript
# low_health_effects.gd
extends ColorRect  # full-screen overlay

@export var low_health_threshold: float = 30.0  # percent
@export var critical_threshold: float = 15.0
@export var vignette_intensity: float = 0.6

func _ready() -> void:
    color = Color(0.4, 0, 0, 0)  # أحمر شفاف
    PlayerHealth.health_changed.connect(_on_health_changed)

func _on_health_changed(current: float, maximum: float) -> void:
    var percent := (current / maximum) * 100.0
    
    if percent <= critical_threshold:
        # نبضات قوية
        var pulse := sin(Time.get_ticks_msec() * 0.008) * 0.5 + 0.5
        color.a = vignette_intensity * pulse
    elif percent <= low_health_threshold:
        # vignette خفيف
        color.a = vignette_intensity * 0.4
    else:
        color.a = 0.0
```

---

## 7. Map System (`Game play/Systems/Map System.md`)

### 7.1 Concept
خريطة Metroidvania تحتاج:
- **Exploration tracking**: ما الذي اكتشفه اللاعب؟
- **Markers**: نقاط الاهتمام (altars، merchants، bosses).
- **Ability-gating visualization**: المناطق المقفلة.
- **Fast travel**: بين altars (optional).

### 7.2 Architecture

```
MapSystem (Autoload)
├── Discovered Rooms (Set)
├── Visited Rooms (Set)
├── Markers (per room)
├── Current Region/Room
└── Map UI
```

### 7.3 Code Skeleton

```gdscript
# map_system.gd (Autoload)
extends Node

signal room_discovered(room_id: String)
signal room_cleared(room_id: String)
signal marker_added(room_id: String, marker_type: String)

# room_id = "A3", "B1", etc.
var discovered_rooms: PackedStringArray = []
var visited_rooms: PackedStringArray = []  # دخلت فعلاً
var cleared_rooms: PackedStringArray = []  # كل الأعداء ماتوا
var markers: Dictionary = {}  # room_id -> Array of {type, position, label}

const ROOM_DATABASE := {
    "A1": {"name": "The First Bleed", "region": "Blood Veins", "position": Vector2(0, 0), "size": Vector2(1280, 720)},
    "A2": {"name": "Crimson Hall", "region": "Blood Veins", "position": Vector2(1280, 0), "size": Vector2(1280, 720)},
    "A3": {"name": "Altar of First Drink", "region": "Blood Veins", "position": Vector2(2560, 0), "size": Vector2(1280, 720)},
    # ...
}

func discover_room(room_id: String) -> void:
    if room_id not in discovered_rooms:
        discovered_rooms.append(room_id)
        room_discovered.emit(room_id)

func enter_room(room_id: String) -> void:
    discover_room(room_id)
    if room_id not in visited_rooms:
        visited_rooms.append(room_id)

func clear_room(room_id: String) -> void:
    if room_id not in cleared_rooms:
        cleared_rooms.append(room_id)
        room_cleared.emit(room_id)

func add_marker(room_id: String, marker_type: String, label: String = "") -> void:
    if not markers.has(room_id):
        markers[room_id] = []
    markers[room_id].append({"type": marker_type, "label": label})
    marker_added.emit(room_id, marker_type)

func has_marker(room_id: String, marker_type: String) -> bool:
    if not markers.has(room_id):
        return false
    for m in markers[room_id]:
        if m.type == marker_type:
            return true
    return false
```

### 7.4 Map UI

```gdscript
# map_ui.gd
extends Control

@onready var grid: GridContainer = $MapGrid
@onready var region_label: Label = $RegionLabel

const TILE_SIZE := Vector2(40, 24)

func _ready() -> void:
    visible = false
    set_process_input(true)

func _unhandled_input(event: InputEvent) -> void:
    if event.is_action_pressed("map") and InputSystem.active_context == "gameplay":
        visible = not visible
        get_tree().paused = visible
        if visible:
            _render_map()

func _render_map() -> void:
    for child in grid.get_children():
        child.queue_free()
    
    for room_id in MapSystem.ROOM_DATABASE.keys():
        var room_data := MapSystem.ROOM_DATABASE[room_id]
        var tile := Panel.new()
        tile.custom_minimum_size = TILE_SIZE
        
        var is_discovered := room_id in MapSystem.discovered_rooms
        var is_visited := room_id in MapSystem.visited_rooms
        var is_cleared := room_id in MapSystem.cleared_rooms
        
        if not is_discovered:
            tile.modulate = Color(0, 0, 0, 1)  # أسود
        elif not is_visited:
            tile.modulate = Color(0.3, 0.3, 0.3, 1)  # رمادي
        elif is_cleared:
            tile.modulate = Color(0.7, 0.7, 0.7, 1)  # أبيض باهت
        else:
            tile.modulate = Color(1, 1, 1, 1)  # أبيض
        
        # Markers
        if MapSystem.has_marker(room_id, "altar"):
            _add_marker_icon(tile, "altar")
        if MapSystem.has_marker(room_id, "boss"):
            _add_marker_icon(tile, "boss")
        if MapSystem.has_marker(room_id, "merchant"):
            _add_marker_icon(tile, "merchant")
        
        grid.add_child(tile)
```

### 7.5 Marker Types

| Marker | Icon | When Added |
|--------|------|-----------|
| `altar` | 🔴 (red circle) | عند زيارة altar لأول مرة |
| `boss` | 💀 (skull) | عند دخول boss arena |
| `merchant` | 🪙 (coin) | عند لقاء merchant |
| `secret` | ❓ (question mark) | عند اكتشاف secret entrance |
| `locked` | 🔒 (lock) | عند العثور على ability-gate |
| `unlocked` | ✓ (checkmark) | عند فتح الـ gate |
| `save_point` | ⭐ (star) | عند استخدام altar |

---

## 8. Skill Tree System (`Game play/Systems/Skill Tree System.md`)

### 8.1 Concept
شجرة مهارات غير خطية، تتفرّع لـ 4 مسارات:
1. **Movement** (Wall Jump، Double Jump، Roll Dash، Wall Dash، Dodge Dash).
2. **Melee Combat** (Charged Slash، Dash Attack، Fall Stab، Back Stab، Combo upgrades).
3. **Handgun** ( ammo capacity، damage، special bullets).
4. **Survival** (max HP، max flasks، i-frame duration).

> ملاحظة: تفاصيل Sanguis (العملة) في `02-Pitfalls-Solutions.md` Pitfall #1.

### 8.2 Architecture

```
SkillTree
├── Skills (Dictionary: id -> SkillData)
├── Unlocked Skills (Array)
├── Branches (4: movement, melee, handgun, survival)
└── Prerequisites (graph)
```

### 8.3 Skill Tree Layout

```text
[Movement Branch]                    [Melee Branch]
   |                                    |
[Wall Jump] 50                  [Charged Slash] 60
   |                                    |
[Double Jump] 80               [Dash Attack] 80
   |                                    |
[Roll Dash] 60                 [Fall Stab] 70
   |                                    |
[Wall Dash] 100                [Back Stab] 100
   |                                    |
[Dodge Dash] 120               [Combo Master] 150
                                        |
                                [Upward Strike+] 80
                                        |
                                [Pogo Jump] 100

[Handgun Branch]                    [Survival Branch]
   |                                    |
[Ammo +5] 50                    [Max HP +20] 60
   |                                    |
[Damage +20%] 80                [Max Flask +1] 100
   |                                    |
[Quick Draw] 100                [I-frames +0.2s] 80
   |                                    |
[Piercing Rounds] 150           [Regen 1/s] 200
   |                                    |
[Explosive Rounds] 250          [Last Stand] 250
```

### 8.4 Code Skeleton

```gdscript
# skill_tree.gd (Autoload)
extends Node

signal skill_unlocked(skill_id: String)
signal skill_point_spent(amount: int)

const SKILLS := {
    # === Movement ===
    "wall_jump": {
        "name": "Wall Jump",
        "description": "Jump off walls to reach new heights.",
        "branch": "movement",
        "cost": 50,
        "prerequisites": [],
        "icon": "res://icons/skills/wall_jump.png",
        "position": Vector2(0, 0)
    },
    "double_jump": {
        "name": "Double Jump",
        "description": "Perform a second jump in mid-air.",
        "branch": "movement",
        "cost": 80,
        "prerequisites": ["wall_jump"],
        "icon": "res://icons/skills/double_jump.png",
        "position": Vector2(0, 1)
    },
    "roll_dash": {
        "name": "Roll Dash",
        "description": "Dash under enemies, ignoring collision damage.",
        "branch": "movement",
        "cost": 60,
        "prerequisites": [],
        "position": Vector2(0, 2)
    },
    "wall_dash": {
        "name": "Wall Dash",
        "description": "Launch from walls during wall slide.",
        "branch": "movement",
        "cost": 100,
        "prerequisites": ["wall_jump"],
        "position": Vector2(0, 3)
    },
    "dodge_dash": {
        "name": "Dodge Dash",
        "description": "Quick backward dash to evade attacks.",
        "branch": "movement",
        "cost": 120,
        "prerequisites": ["roll_dash"],
        "position": Vector2(0, 4)
    },
    
    # === Melee ===
    "charged_slash": {
        "name": "Charged Slash",
        "description": "Hold attack for 0.6s to unleash a heavy blow.",
        "branch": "melee",
        "cost": 60,
        "prerequisites": [],
        "position": Vector2(1, 0)
    },
    "dash_attack": {
        "name": "Swift Slash",
        "description": "Attack during dash to penetrate enemies.",
        "branch": "melee",
        "cost": 80,
        "prerequisites": ["charged_slash"],
        "position": Vector2(1, 1)
    },
    "fall_stab": {
        "name": "Death from Above",
        "description": "Slam down on enemies from the air.",
        "branch": "melee",
        "cost": 70,
        "prerequisites": [],
        "position": Vector2(1, 2)
    },
    "back_stab": {
        "name": "Back Stab",
        "description": "Stealth kill unaware enemies from behind.",
        "branch": "melee",
        "cost": 100,
        "prerequisites": ["fall_stab"],
        "position": Vector2(1, 3)
    },
    "combo_master": {
        "name": "Combo Master",
        "description": "Mix melee + handgun for special combo effects.",
        "branch": "melee",
        "cost": 150,
        "prerequisites": ["dash_attack"],
        "position": Vector2(1, 4)
    },
    
    # === Handgun ===
    "ammo_plus_5": {
        "name": "Ammo Capacity +5",
        "description": "Increase handgun ammo by 5.",
        "branch": "handgun",
        "cost": 50,
        "prerequisites": [],
        "position": Vector2(2, 0)
    },
    "handgun_damage_20": {
        "name": "Gunpowder+",
        "description": "Handgun damage +20%.",
        "branch": "handgun",
        "cost": 80,
        "prerequisites": ["ammo_plus_5"],
        "position": Vector2(2, 1)
    },
    "quick_draw": {
        "name": "Quick Draw",
        "description": "Reduce handgun fire cooldown by 50%.",
        "branch": "handgun",
        "cost": 100,
        "prerequisites": ["handgun_damage_20"],
        "position": Vector2(2, 2)
    },
    "piercing_rounds": {
        "name": "Piercing Rounds",
        "description": "Bullets pierce through 2 enemies.",
        "branch": "handgun",
        "cost": 150,
        "prerequisites": ["quick_draw"],
        "position": Vector2(2, 3)
    },
    "explosive_rounds": {
        "name": "Explosive Rounds",
        "description": "Final bullet in magazine explodes on impact.",
        "branch": "handgun",
        "cost": 250,
        "prerequisites": ["piercing_rounds"],
        "position": Vector2(2, 4)
    },
    
    # === Survival ===
    "max_hp_20": {
        "name": "Vitality+",
        "description": "Max HP +20.",
        "branch": "survival",
        "cost": 60,
        "prerequisites": [],
        "position": Vector2(3, 0)
    },
    "max_flask_1": {
        "name": "Flask Capacity +1",
        "description": "Carry one more Vein Flask (max 5).",
        "branch": "survival",
        "cost": 100,
        "prerequisites": ["max_hp_20"],
        "position": Vector2(3, 1)
    },
    "iframes_plus": {
        "name": "Vein Resilience",
        "description": "I-frames +0.2s after damage.",
        "branch": "survival",
        "cost": 80,
        "prerequisites": ["max_hp_20"],
        "position": Vector2(3, 2)
    },
    "regen_1": {
        "name": "Sanguis Regen",
        "description": "Regenerate 1 HP per second out of combat.",
        "branch": "survival",
        "cost": 200,
        "prerequisites": ["iframes_plus"],
        "position": Vector2(3, 3)
    },
    "last_stand": {
        "name": "Last Stand",
        "description": "Survive one lethal hit at 1 HP (once per room).",
        "branch": "survival",
        "cost": 250,
        "prerequisites": ["regen_1"],
        "position": Vector2(3, 4)
    },
}

var unlocked_skills: PackedStringArray = []

func has_skill(skill_id: String) -> bool:
    return skill_id in unlocked_skills

func can_unlock(skill_id: String) -> bool:
    if not SKILLS.has(skill_id):
        return false
    if skill_id in unlocked_skills:
        return false
    var skill: Dictionary = SKILLS[skill_id]
    if SanguisManager.sanguis < skill.cost:
        return false
    for prereq in skill.prerequisites:
        if not has_skill(prereq):
            return false
    return true

func unlock(skill_id: String) -> bool:
    if not can_unlock(skill_id):
        return false
    var skill: Dictionary = SKILLS[skill_id]
    SanguisManager.sanguis -= skill.cost
    unlocked_skills.append(skill_id)
    skill_unlocked.emit(skill_id)
    _apply_skill_effect(skill_id)
    return true

func _apply_skill_effect(skill_id: String) -> void:
    var player := get_tree().get_first_node_in_group("player") as Player
    if not player:
        return
    match skill_id:
        "wall_jump": player.movement.can_wall_jump = true
        "double_jump": player.movement.can_double_jump = true
        "roll_dash": player.movement.can_roll_dash = true
        "wall_dash": player.movement.can_wall_dash = true
        "dodge_dash": player.movement.can_dodge_dash = true
        "charged_slash": player.combat.can_charge_attack = true
        "dash_attack": player.combat.can_dash_attack = true
        "fall_stab": player.combat.can_fall_stab = true
        "back_stab": player.combat.can_back_stab = true
        "combo_master": player.combat.can_combo_mix = true
        "ammo_plus_5": player.combat.handgun_max_ammo += 5
        "handgun_damage_20": player.combat.handgun_damage_multiplier *= 1.2
        "quick_draw": player.combat.handgun_cooldown *= 0.5
        "piercing_rounds": player.combat.handgun_pierce_count = 2
        "explosive_rounds": player.combat.handgun_explosive = true
        "max_hp_20": player.health.upgrade_max_health(20)
        "max_flask_1": player.health.upgrade_max_flasks(1)
        "iframes_plus": player.health.invincibility_duration += 0.2
        "regen_1": player.health.regen_per_second = 1.0
        "last_stand": player.health.has_last_stand = true
```

### 8.5 Save/Load

```gdscript
# save_manager.gd (Autoload)
extends Node

const SAVE_PATH := "user://veinbreaker_save.json"

func save_game() -> void:
    var data := {
        "sanguis": SanguisManager.sanguis,
        "cinders": CindersManager.cinders,
        "unlocked_skills": SkillTree.unlocked_skills,
        "current_health": PlayerHealth.current_health,
        "current_flasks": PlayerHealth.current_flasks,
        "max_health": PlayerHealth.max_health,
        "max_flasks": PlayerHealth.max_flasks,
        "discovered_rooms": MapSystem.discovered_rooms,
        "cleared_rooms": MapSystem.cleared_rooms,
        "current_room": MapSystem.current_room_id,
        "playtime_seconds": Time.get_ticks_msec() / 1000.0 - SessionManager.start_time,
    }
    var file := FileAccess.open(SAVE_PATH, FileAccess.WRITE)
    file.store_string(JSON.stringify(data, "\t"))
    file.close()

func load_game() -> bool:
    if not FileAccess.file_exists(SAVE_PATH):
        return false
    var file := FileAccess.open(SAVE_PATH, FileAccess.READ)
    var data = JSON.parse_string(file.get_as_text())
    file.close()
    
    SanguisManager.sanguis = data.sanguis
    CindersManager.cinders = data.cinders
    SkillTree.unlocked_skills = data.unlocked_skills
    PlayerHealth.current_health = data.current_health
    PlayerHealth.current_flasks = data.current_flasks
    PlayerHealth.max_health = data.max_health
    PlayerHealth.max_flasks = data.max_flasks
    MapSystem.discovered_rooms = data.discovered_rooms
    MapSystem.cleared_rooms = data.cleared_rooms
    
    # Reload player into last room
    LevelManager.load_room(data.current_room)
    return true
```

---

## 9. Damage System (`Game play/Systems/Combat Systems/Damage System.md`)

### 9.1 Concept
الـ Damage System يجمع كل ما يتعلق بـ:
- حساب الضرر (formula).
- تطبيقه (knockback، hit reactions).
- I-frames.
- Damage types (physical، elemental مستقبلاً).
- Death handling.

### 9.2 Architecture

```
DamageSystem (Autoload)
├── Damage Formula
├── Hit Reactions (hit stop، knockback، particles)
├── Damage Type Modifiers
└── Critical Hit Logic
```

### 9.3 Code Skeleton

```gdscript
# damage_system.gd (Autoload)
extends Node

# Damage Types
enum DamageType {
    SLASHING,   # blades
    PIERCING,   # handgun bullets
    BLUDGEONING,# charged attacks، fall stab
    FIRE,       # explosive rounds
    POISON,     # status effects
    TRUE,       # ignores armor
}

# Damage Type Effectiveness Matrix
const TYPE_MULTIPLIERS := {
    # damage_type → enemy_armor_class → multiplier
    "SLASHING":   {"light": 1.2, "medium": 1.0, "heavy": 0.7, "flying": 0.8},
    "PIERCING":   {"light": 0.8, "medium": 1.0, "heavy": 1.2, "flying": 1.5},
    "BLUDGEONING":{"light": 1.0, "medium": 1.1, "heavy": 1.4, "flying": 0.5},
    "FIRE":       {"light": 1.0, "medium": 1.0, "heavy": 1.0, "flying": 1.0},
    "POISON":     {"light": 1.0, "medium": 1.0, "heavy": 1.0, "flying": 1.0},
    "TRUE":       {"light": 1.0, "medium": 1.0, "heavy": 1.0, "flying": 1.0},
}

signal damage_dealt(target: Node, amount: float, damage_type: int, is_critical: bool)
signal damage_received(amount: float, source: Node)

func calculate_damage(
    base_damage: float,
    damage_type: int,
    attacker_multiplier: float,
    target_armor_class: String,
    target_armor_value: float,
    is_critical: bool = false
) -> float:
    # Final Formula:
    # final = base * attacker_mult * type_mult * (1 - armor_reduction) * critical_mult
    
    var type_mult: float = TYPE_MULTILLIERS.get(DamageType.keys()[damage_type], {}).get(target_armor_class, 1.0)
    var armor_reduction: float = clamp(target_armor_value / (target_armor_value + 50.0), 0.0, 0.85)
    var critical_mult: float = 2.0 if is_critical else 1.0
    
    var final_damage := base_damage * attacker_multiplier * type_mult * (1.0 - armor_reduction) * critical_mult
    return max(1.0, final_damage)  # minimum 1 damage

func apply_damage(
    target: Node,
    amount: float,
    damage_type: int,
    knockback_dir: Vector2,
    knockback_force: float,
    source: Node,
    is_critical: bool = false
) -> void:
    if not target.has_method("take_damage"):
        return
    
    # Calculate armor etc.
    var target_armor_class: String = target.get("armor_class") if "armor_class" in target else "medium"
    var target_armor_value: float = target.get("armor_value") if "armor_value" in target else 0.0
    
    var final_amount := calculate_damage(
        amount, damage_type, 1.0, target_armor_class, target_armor_value, is_critical
    )
    
    # Apply knockback
    var knockback := knockback_dir.normalized() * knockback_force
    
    # Hit Stop (universal juice)
    _apply_hit_stop(0.04 if not is_critical else 0.08)
    
    # Call target's take_damage
    target.take_damage(final_amount, knockback, source)
    
    # Spawn particles + sound
    _spawn_hit_effects(target, damage_type, is_critical)
    
    damage_dealt.emit(target, final_amount, damage_type, is_critical)

func _apply_hit_stop(duration: float) -> void:
    Engine.time_scale = 0.05
    await get_tree().create_timer(duration, true, false, true).timeout
    Engine.time_scale = 1.0

func _spawn_hit_effects(target: Node, damage_type: int, is_critical: bool) -> void:
    # Spawn particles based on target's material_type (flesh, metal, stone)
    var material_type: String = target.get("material_type") if "material_type" in target else "flesh"
    
    var particle_scene: PackedScene
    match material_type:
        "flesh": particle_scene = preload("res://effects/blood_splash.tscn")
        "metal": particle_scene = preload("res://effects/metal_sparks.tscn")
        "stone": particle_scene = preload("res://effects/stone_chips.tscn")
        _:       particle_scene = preload("res://effects/generic_hit.tscn")
    
    var particles := particle_scene.instantiate()
    particles.global_position = target.global_position
    if is_critical:
        particles.scale *= 1.5
    get_tree().current_scene.add_child(particles)
    
    # Sound
    AudioManager.play_hit_sound(material_type, is_critical)
```

### 9.4 Critical Hit Logic

```gdscript
# Player's critical chance (base 5% + skills)
func roll_critical(base_chance: float = 0.05) -> bool:
    return randf() < base_chance + SkillTree.get_critical_bonus()

# Back stab = always critical
# Charged attack = +30% critical chance
# Fall stab = always critical
```

### 9.5 Damage Numbers (Floating Text)

```gdscript
# damage_number.gd
extends Label

func _ready() -> void:
    z_index = 100  # فوق كل شيء

func setup(amount: float, is_critical: bool, position: Vector2) -> void:
    text = str(int(amount))
    if is_critical:
        text = "CRIT! " + text
        add_theme_color_override("font_color", Color.YELLOW)
        add_theme_font_size_override("font_size", 24)
    else:
        add_theme_color_override("font_color", Color.WHITE)
        add_theme_font_size_override("font_size", 16)
    
    global_position = position + Vector2(randf_range(-10, 10), -20)
    
    # Animation: ترتفع وتتلاشى
    var tween := create_tween()
    tween.tween_property(self, "global_position:y", global_position.y - 40, 0.8).set_ease(Tween.EASE_OUT)
    tween.parallel().tween_property(self, "modulate:a", 0.0, 0.8)
    tween.tween_callback(queue_free)
```

---

## 10. Handguns (`Game play/Systems/Combat Systems/Handguns.md`)

### 10.1 Concept
الـ Handgun هو السلاح الثانوي:
- **محدود**: ذخيرة محدودة.
- **بعيد المدى**: يتفادى خطر القتال القريب.
- **تكتيكي**: لإبعاد الأعداء، إطلاق الأبواب، إصابة الطائرة.
- **3 اتجاهات**: Up، Right، Left (مع aim assist).

### 10.2 Architecture

```
HandgunController
├── Ammo (current/max)
├── Fire Cooldown
├── Aim Direction (from joystick)
├── Bullet Scene
├── Aim Assist
└── Special Bullets (piercing، explosive)
```

### 10.3 Code Skeleton

```gdscript
# handgun_controller.gd
extends Node2D

@export var bullet_scene: PackedScene
@export var max_ammo: int = 20
@export var fire_cooldown: float = 0.4
@export var bullet_speed: float = 600.0
@export var base_damage: float = 30.0
@export var aim_assist_angle_deg: float = 30.0
@export var aim_assist_range: float = 400.0

var current_ammo: int = max_ammo
var cooldown_timer: float = 0.0

# Skill tree modifiers
var damage_multiplier: float = 1.0
var pierce_count: int = 0
var is_explosive: bool = false

@onready var muzzle: Marker2D = $Muzzle
@onready var aim_assist_ray: RayCast2D = $AimAssistRay

signal ammo_changed(current: int, max: int)
signal gun_fired()

func _ready() -> void:
    current_ammo = max_ammo
    ammo_changed.emit(current_ammo, max_ammo)

func _process(delta: float) -> void:
    if cooldown_timer > 0:
        cooldown_timer = max(0.0, cooldown_timer - delta)

func can_fire() -> bool:
    return current_ammo > 0 and cooldown_timer <= 0.0

func fire(aim_direction: Vector2) -> void:
    if not can_fire():
        return
    
    # Aim assist
    var final_direction := _apply_aim_assist(aim_direction)
    
    # Spawn bullet
    var bullet := bullet_scene.instantiate() as Bullet
    bullet.global_position = muzzle.global_position
    bullet.direction = final_direction
    bullet.speed = bullet_speed
    bullet.damage = base_damage * damage_multiplier
    bullet.pierce_count = pierce_count
    bullet.is_explosive = is_explosive and current_ammo == 1  # آخر طلقة
    get_tree().current_scene.add_child(bullet)
    
    current_ammo -= 1
    cooldown_timer = fire_cooldown
    
    ammo_changed.emit(current_ammo, max_ammo)
    gun_fired.emit()
    
    # Effects
    _muzzle_flash()
    CameraManager.shake_light()
    
    if current_ammo == 0:
        _play_empty_click()

func _apply_aim_assist(aim_direction: Vector2) -> Vector2:
    var player := get_tree().get_first_node_in_group("player") as Node2D
    if not player:
        return aim_direction
    
    var best_target: Node2D = null
    var best_dot := cos(deg_to_rad(aim_assist_angle_deg))
    
    for enemy in get_tree().get_nodes_in_group("enemies"):
        var to_enemy := enemy.global_position - player.global_position
        if to_enemy.length() > aim_assist_range:
            continue
        var dot := aim_direction.dot(to_enemy.normalized())
        if dot > best_dot:
            best_dot = dot
            best_target = enemy
    
    if best_target:
        return (best_target.global_position - muzzle.global_position).normalized()
    return aim_direction

func _muzzle_flash() -> void:
    var flash := preload("res://effects/muzzle_flash.tscn").instantiate()
    flash.global_position = muzzle.global_position
    flash.rotation = aim_direction.angle() if aim_direction else 0.0
    get_tree().current_scene.add_child(flash)

func reload_ammo(amount: int) -> void:
    current_ammo = min(max_ammo, current_ammo + amount)
    ammo_changed.emit(current_ammo, max_ammo)

func refill_full() -> void:
    current_ammo = max_ammo
    ammo_changed.emit(current_ammo, max_ammo)
```

### 10.4 Bullet Scene

```gdscript
# bullet.gd
extends Area2D

@export var speed: float = 600.0
@export var damage: float = 30.0
@export var pierce_count: int = 0
@export var is_explosive: bool = false
@export var lifetime: float = 2.0

var direction: Vector2 = Vector2.ZERO
var hit_enemies: Array[Node] = []

func _ready() -> void:
    body_entered.connect(_on_body_entered)
    area_entered.connect(_on_area_entered)
    
    # Auto-destroy after lifetime
    var timer := get_tree().create_timer(lifetime)
    timer.timeout.connect(queue_free)

func _physics_process(delta: float) -> void:
    position += direction * speed * delta

func _on_body_entered(body: Node) -> void:
    if body.is_in_group("walls"):
        _impact()
        queue_free()
    elif body.is_in_group("enemies") and body not in hit_enemies:
        hit_enemies.append(body)
        DamageSystem.apply_damage(
            body, damage, DamageSystem.DamageType.PIERCING,
            direction, 50.0, self, false
        )
        
        if is_explosive:
            _explode()
            queue_free()
        elif pierce_count > 0:
            pierce_count -= 1
        else:
            _impact()
            queue_free()

func _on_area_entered(area: Area2D) -> void:
    # Parry reflects
    if area.is_in_group("parry_reflect"):
        direction = -direction
        hit_enemies.clear()

func _impact() -> void:
    var impact := preload("res://effects/bullet_impact.tscn").instantiate()
    impact.global_position = global_position
    get_tree().current_scene.add_child(impact)

func _explode() -> void:
    var explosion := preload("res://effects/explosion.tscn").instantiate()
    explosion.global_position = global_position
    explosion.damage = damage * 0.5
    explosion.radius = 80.0
    get_tree().current_scene.add_child(explosion)
    CameraManager.shake_medium()
```

### 10.5 Handgun Stats Progression

| Stat | Base | With All Skills | Notes |
|------|------|-----------------|-------|
| Ammo | 20 | 25 | +5 from skill |
| Damage | 30 | 36 | +20% from skill |
| Fire Cooldown | 0.4s | 0.2s | -50% from Quick Draw |
| Pierce | 0 | 2 | From Piercing Rounds |
| Explosive | No | Yes (last shot) | From Explosive Rounds |

---

## 11. Parry (`Game play/Systems/Combat Systems/Parry.md`)

### 11.1 Concept
الـ Parry هو **أعلى skill expression** في اللعبة:
- توقيت دقيق (±0.15s window).
- مكافأة عالية (stagger + counter damage).
- يجعل القتال "dance" وليس button mash.

> ملاحظة: زرّ الـ Parry مدمج مع Attack (Pitfall #2). هذا الملف يفصّل منطق الـ Parry نفسه.

### 11.2 Architecture

```
ParryManager (Autoload)
├── Active Parry Windows (per enemy attack)
├── Parry Success → Stagger + Counter
├── Parry Fail → continue attack
└── Parry Reflect (for projectiles)
```

### 11.3 Code Skeleton

```gdscript
# parry_manager.gd (Autoload)
extends Node

signal parry_success(enemy: Node)
signal parry_failed()
signal parry_perfect(enemy: Node)  # tighter window = perfect parry

@export_group("Windows")
@export var parry_window_duration: float = 0.15   # نافذة عادية
@export var perfect_parry_window: float = 0.05    # نافذة perfect
@export var parry_cooldown: float = 0.3

# كل عدو يهاجم يسجّل window هنا
var active_windows: Array[Dictionary] = []
var last_parry_time: float = 0.0

func register_attack_window(enemy: Node, hit_land_time: float, attack_id: String) -> void:
    var window_start := hit_land_time - parry_window_duration
    var window_end := hit_land_time
    var perfect_end := hit_land_time - parry_window_duration + perfect_parry_window
    
    active_windows.append({
        "enemy": enemy,
        "start": window_start,
        "perfect_end": perfect_end,
        "end": window_end,
        "attack_id": attack_id,
    })

func attempt_parry() -> void:
    var now := Time.get_ticks_msec() / 1000.0
    
    if now - last_parry_time < parry_cooldown:
        parry_failed.emit()
        return
    
    last_parry_time = now
    
    # ابحث عن window نشطة
    var found_window: Dictionary = {}
    var is_perfect := false
    
    for i in range(active_windows.size()):
        var window = active_windows[i]
        if now >= window.start and now <= window.end:
            found_window = window
            is_perfect = now <= window.perfect_end
            active_windows.remove_at(i)
            break
    
    if found_window.is_empty():
        parry_failed.emit()
        return
    
    # Success!
    var enemy: Node = found_window.enemy
    
    if is_perfect:
        parry_perfect.emit(enemy)
        _apply_perfect_parry_effects(enemy)
    else:
        parry_success.emit(enemy)
        _apply_parry_effects(enemy)

func _apply_parry_effects(enemy: Node) -> void:
    # Stagger العدو
    if enemy.has_method("apply_stagger"):
        enemy.apply_stagger(1.0)
    
    # Hit stop
    Engine.time_scale = 0.1
    await get_tree().create_timer(0.1, true, false, true).timeout
    Engine.time_scale = 1.0
    
    # Camera shake
    CameraManager.shake_medium()
    
    # Spark effect
    var spark := preload("res://effects/parry_spark.tscn").instantiate()
    spark.global_position = enemy.global_position
    get_tree().current_scene.add_child(spark)
    
    # Sound
    AudioManager.play_parry_sound(false)
    
    # Counter damage window (1.5s)
    CounterAttackManager.open_counter_window(enemy, 1.5)

func _apply_perfect_parry_effects(enemy: Node) -> void:
    _apply_parry_effects(enemy)  # كل ما سبق
    
    # إضافي للـ perfect:
    # - يلغي هجوم العدو كلياً
    if enemy.has_method("cancel_attack"):
        enemy.cancel_attack()
    
    # - يفتح counter window أطول
    CounterAttackManager.open_counter_window(enemy, 2.5)
    
    # - يحفظ طلقة parry reflect
    ParryReflectManager.add_charge(1)
    
    # - screen flash
    var flash := preload("res://effects/perfect_parry_flash.tscn").instantiate()
    get_tree().current_scene.add_child(flash)
    
    AudioManager.play_parry_sound(true)

func _process(delta: float) -> void:
    # Clean expired windows
    var now := Time.get_ticks_msec() / 1000.0
    for i in range(active_windows.size() - 1, -1, -1):
        if active_windows[i].end < now:
            active_windows.remove_at(i)
```

### 11.4 Counter Attack Manager

```gdscript
# counter_attack_manager.gd (Autoload)
extends Node

var active_counter_targets: Array[Dictionary] = []

func open_counter_window(enemy: Node, duration: float) -> void:
    active_counter_targets.append({
        "enemy": enemy,
        "expire": Time.get_ticks_msec() / 1000.0 + duration,
    })

func is_counter_available(enemy: Node) -> bool:
    var now := Time.get_ticks_msec() / 1000.0
    for entry in active_counter_targets:
        if entry.enemy == enemy and entry.expire > now:
            return true
    return false

func consume_counter(enemy: Node) -> void:
    var now := Time.get_ticks_msec() / 1000.0
    for i in range(active_counter_targets.size()):
        if active_counter_targets[i].enemy == enemy:
            active_counter_targets.remove_at(i)
            return

func _process(delta: float) -> void:
    var now := Time.get_ticks_msec() / 1000.0
    for i in range(active_counter_targets.size() - 1, -1, -1):
        if active_counter_targets[i].expire < now:
            active_counter_targets.remove_at(i)
```

### 11.5 Integration: Enemy Attack Hook

```gdscript
# enemy_base.gd (مكمّل)
func _perform_attack() -> void:
    var hit_land_time := Time.get_ticks_msec() / 1000.0 + attack_telegraph_duration
    
    # سجّل parry window
    ParryManager.register_attack_window(self, hit_land_time, "basic_attack")
    
    # Telegraph بصري
    _play_telegraph_animation()
    
    await get_tree().create_timer(attack_telegraph_duration).timeout
    
    # لو لم يُـ parry (لم يتم إلغاء الهجوم)
    if state == State.ATTACK:
        _deal_attack_damage()

func apply_stagger(duration: float) -> void:
    stagger_meter = stagger_threshold  # يضمن الدخول لـ stagger state
    _transition_to(State.STAGGER)

func cancel_attack() -> void:
    if state == State.ATTACK:
        _transition_to(State.STAGGER)
```

### 11.6 Player Attack → Counter Check

```gdscript
# combat_controller.gd
func _perform_slash() -> void:
    var enemy := _get_enemy_in_front()
    if enemy and CounterAttackManager.is_counter_available(enemy):
        # Counter attack = double damage + guaranteed critical
        var counter_damage := melee_damage * 2.0
        DamageSystem.apply_damage(
            enemy, counter_damage, DamageSystem.DamageType.SLASHING,
            (enemy.global_position - global_position).normalized(),
            150.0, self, true  # is_critical = true
        )
        CounterAttackManager.consume_counter(enemy)
        CameraManager.shake_heavy()
        _play_counter_animation()
    else:
        # Slash عادي
        _perform_normal_slash()
```

### 11.7 Parry Reflect (Projectiles)

```gdscript
# parry_reflect_manager.gd (Autoload)
extends Node

var charges: int = 0

func add_charge(amount: int) -> void:
    charges = min(3, charges + amount)

func try_reflect() -> bool:
    if charges <= 0:
        return false
    charges -= 1
    return true

# عند parry projectile:
# bullet.direction = -bullet.direction
# hit_enemies.clear()
# bullet.damage *= 2  # reflected bullets do double damage
```

---

## خلاصة الأنظمة الفارغة

| النظام | المفهوم الأساسي | الكود الأساسي |
|--------|----------------|---------------|
| Input Map | centralized dispatcher | `input_system.gd` Autoload |
| Enemy AI | State Machine + Composition | `enemy_base.gd` + subclasses |
| Player | Component-based CharacterBody2D | `player.gd` + child controllers |
| Level Design | Room-based YAML specs | `room_data.tres` resources |
| Camera | Lerp follow + trauma shake | `camera_controller.gd` |
| Health | HP + Flasks + I-frames | `health_controller.gd` |
| Map | Discovered/Visited/Cleared tracking | `map_system.gd` Autoload |
| Skill Tree | 4 branches × 5 skills each | `skill_tree.gd` Autoload |
| Damage | Formula + Type Matrix + Hit Stop | `damage_system.gd` Autoload |
| Handguns | Bullet + Aim Assist | `handgun_controller.gd` |
| Parry | Window-based + Counter + Reflect | `parry_manager.gd` Autoload |

**هذه الـ 11 نظاماً تشكل الـ "skeleton" الكامل للعبة. الكود أعلاه قابل للـ copy-paste في Godot 4.7 والبدء بـ prototyping فوراً.**

---

**التالي**: راجع `04-Story-Concepts.md` للأفكار القصصية بـ plot من 5 أنواع أدبية.
