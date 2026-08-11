# Camera System v2 — Architecture & Implementation

Target: Godot 4.7 · 2D Action-Platformer / Metroidvania · Mobile-First
Reference feel: Hollow Knight / Nine Sols / Silksong

---

## 0) Changelog from v1 — what broke and why

| Problem in v1 | Root Cause | Fix in v2 |
|---|---|---|
| Forward Offset shared the same smoothing speed as Follow | No dedicated variable | New `forward_offset_smoothing_speed`, fully independent |
| `CameraArea.limits` values were wrong / imprecise | `limit_left/top/right/bottom` are **absolute world-pixel coordinates from the scene's (0,0) origin** — not a size, not relative to the node. A hand-typed `Rect2` has **no visual reference in the editor**, so you were guessing blind. | Limits are now **derived automatically from a child `CollisionShape2D` + `RectangleShape2D`** — a shape you can see and drag in the 2D viewport, same as any collision shape. No manual numbers, ever. |
| No way to stop Forward Offset while falling | Not implemented | New `set_falling(bool)`, smoothly retargets offset to 0 via the same dedicated smoothing speed |
| No vertical-lock / "horizontal-only" follow | Not implemented | New Vertical Behavior system (§4) |
| Future Extensions §8 from v1 were just ideas | Not implemented | Shake presets, `focus_on()`, dash zoom pulse, and fall look-ahead are now implemented (§7) |

---

## 1) Architecture (unchanged core principle)

Godot's `Camera2D` gives two different ways to move the view:

- **`global_position`** (the real transform) — clamped by `limit_*`, smoothed by `position_smoothing_speed`.
- **`offset`** (draw-time visual nudge) — **not** clamped by `limit_*`. Meant for shake.

**v2 change:** everything that isn't shake — base follow, forward offset, vertical lock, fall look-ahead, cinematic focus — now funnels through **one single calculation per physics frame** that produces one `Vector2` target, assigned once to `global_position`. This replaced v1's mix of "let the parent transform do it" + separate manual offsets, which is exactly what made forward-offset's smoothing impossible to isolate. One funnel point, one source of truth, only shake stays separate (on `offset`, as intended).

```
   Player state (facing / grounded / falling)
                │
                ▼
   PlayerCamera._physics_process()
        ├─ forward offset (own smoothing) ──┐
        ├─ vertical lock / fall look-ahead ─┼──► target Vector2 ──► global_position
        └─ focus override (bypasses both) ──┘         (engine smooths + clamps)

        └─ shake (independent) ──────────────────► offset (engine does NOT clamp this)
```

---

## 2) Setup Requirements

- Player root node must be in the **`"player"`** group.
- Player root should **not** be mirrored via `scale.x = -1` on the root itself — flip a child sprite/pivot instead, or the Forward Offset direction will double-invert.
- `PlayerCamera` must be a **direct child** of the Player node (`get_parent()` is used to find it — no manual wiring).
- Each `CameraArea` needs exactly one `CollisionShape2D` child using a `RectangleShape2D`, sized/dragged visually in the 2D editor to match the room.

---

## 3) `PlayerCamera.gd`

```gdscript
class_name PlayerCamera
extends Camera2D

## PlayerCamera
## ------------------------------------------------------------------
## Child of the Player scene. Every physics frame computes ONE target
## position and assigns it to global_position once:
##   X = player.x + forward_offset (own smoothing speed)
##   Y = normal follow, OR vertical-locked + fall-lookahead while airborne
## The engine's position_smoothing_speed + limit_* then smooth AND
## clamp that final result automatically. Shake stays on `offset`
## (not limit-clamped by the engine — see the Known Trade-off note
## at the bottom of the accompanying doc; intentionally left as-is).
## ------------------------------------------------------------------

enum ShakeType { RANDOM, HORIZONTAL, VERTICAL }

const SHAKE_PRESETS := {
	"HIT_LIGHT":     {"power": 4.0, "duration": 0.15, "frequency": 35.0, "type": ShakeType.RANDOM},
	"HIT_HEAVY":     {"power": 9.0, "duration": 0.35, "frequency": 25.0, "type": ShakeType.RANDOM},
	"PARRY_SUCCESS": {"power": 3.0, "duration": 0.0,  "frequency": 0.0,  "type": ShakeType.RANDOM},
	"BOSS_INTRO":    {"power": 6.0, "duration": 0.6,  "frequency": 15.0, "type": ShakeType.RANDOM},
}

@export_group("Zoom")
@export var default_zoom: Vector2 = Vector2(0.6, 0.6)

@export_group("Follow / Smoothing")
@export_range(7.0, 10.0, 0.1) var follow_smoothing_speed: float = 8.0

@export_group("Forward Offset")
@export var forward_offset_distance: float = 80.0
@export var forward_offset_smoothing_speed: float = 5.0  # independent from follow_smoothing_speed

@export_group("Vertical Behavior")
@export var vertical_lock_enabled: bool = true
@export var fall_lookahead_distance: float = 60.0
@export var fall_lookahead_smoothing_speed: float = 3.0

@export_group("Shake Defaults")
@export var default_shake_frequency: float = 30.0

var _player: Node2D = null

var _facing_direction: int = 1
var _forward_offset_enabled: bool = true
var _target_forward_offset: float = 0.0
var _current_forward_offset: float = 0.0

var _is_grounded: bool = true
var _is_falling: bool = false
var _locked_y: float = 0.0
var _current_fall_lookahead: float = 0.0

var _is_focusing: bool = false
var _focus_target: Node2D = null

var _trauma: float = 0.0
var _trauma_decay_rate: float = 2.0
var _shake_type: ShakeType = ShakeType.RANDOM
var _shake_frequency: float = 30.0
var _shake_seed: float = 0.0

var _active_limits: Dictionary = {}  # area.instance_id -> Rect2


func _ready() -> void:
	_player = get_parent() as Player
	if _player == null:
		push_error("'PlayerCamera': must be a direct child of a Node2D (the Player). Reparent it.")

	zoom = default_zoom
	position_smoothing_enabled = true
	position_smoothing_speed = follow_smoothing_speed
	limit_smoothed = true
	_shake_seed = randf() * 1000.0
	_locked_y = global_position.y
	make_current()
	_connect_to_camera_manager()


func _physics_process(delta: float) -> void:
	if not is_instance_valid(_player):
		return

	_update_forward_offset(delta)
	_update_vertical_behavior(delta)
	_update_trauma(delta)

	var follow_node: Node2D = _focus_target if (_is_focusing and is_instance_valid(_focus_target)) else _player

	var target_x: float = follow_node.global_position.x
	var target_y: float = follow_node.global_position.y

	if not _is_focusing:
		target_x += _current_forward_offset
		target_y = _compute_target_y(follow_node)

	global_position = Vector2(target_x, target_y)
	offset = _compute_shake_offset()


# ---------------------------------------------------------------
# Forward Offset
# ---------------------------------------------------------------

## Call whenever the Player's facing flips (once per flip, not every frame).
func set_facing_direction(dir: int) -> void:
	if dir == 0:
		return
	_facing_direction = 1 if dir > 0 else -1
	_recompute_forward_offset_target()


## Call on Fall-state enter/exit. Stopping/resuming is smoothed via
## forward_offset_smoothing_speed like any other change to the offset.
func set_forward_offset_enabled(enabled: bool) -> void:
	_forward_offset_enabled = enabled
	_recompute_forward_offset_target()


func _recompute_forward_offset_target() -> void:
	_target_forward_offset = (forward_offset_distance * _facing_direction) if _forward_offset_enabled else 0.0


func _update_forward_offset(delta: float) -> void:
	_current_forward_offset = lerpf(
		_current_forward_offset, _target_forward_offset, forward_offset_smoothing_speed * delta
	)


# ---------------------------------------------------------------
# Vertical Behavior — grounded follow / vertical lock / fall look-ahead
# ---------------------------------------------------------------
# Interpretation of the referenced video's "horizontal-only follow":
# while airborne, Y stops tracking the player and holds at the height
# captured the moment they left the ground (X keeps following normally
# the whole time — that's the "horizontal-only" part). During an actual
# Fall (not the rising half of a jump), the locked Y is allowed to
# gradually ease downward toward the player by up to
# fall_lookahead_distance, so a long fall reveals more of what's below
# before landing. On landing, Y smoothly re-syncs to the player via the
# normal follow_smoothing_speed. If this isn't quite what the video
# showed, only this block needs to change — nothing else depends on it.

## Call whenever is_on_floor() changes.
func set_grounded(grounded: bool) -> void:
	if grounded == _is_grounded:
		return
	_is_grounded = grounded
	if not grounded:
		_locked_y = global_position.y


## Call on Fall-state enter/exit (distinct from Jump: Jump = locked with
## no look-ahead, Fall = locked + gradual downward look-ahead).
func set_falling(falling: bool) -> void:
	_is_falling = falling
	set_forward_offset_enabled(not falling)


func _update_vertical_behavior(delta: float) -> void:
	var target_lookahead: float = fall_lookahead_distance if (_is_falling and not _is_grounded) else 0.0
	_current_fall_lookahead = lerpf(
		_current_fall_lookahead, target_lookahead, fall_lookahead_smoothing_speed * delta
	)


func _compute_target_y(follow_node: Node2D) -> float:
	if _is_grounded or not vertical_lock_enabled:
		return follow_node.global_position.y
	return _locked_y + _current_fall_lookahead


# ---------------------------------------------------------------
# Focus (cinematic retarget — boss intros, cutscene beats)
# ---------------------------------------------------------------

## Temporarily follows `target` directly (no forward-offset / vertical
## lock — those are player-specific concepts) for `duration` seconds,
## then reverts to the Player automatically.
func focus_on(target: Node2D, duration: float = 1.0) -> void:
	if not is_instance_valid(target):
		return
	_focus_target = target
	_is_focusing = true
	await get_tree().create_timer(duration).timeout
	_is_focusing = false
	_focus_target = null


# ---------------------------------------------------------------
# Zoom
# ---------------------------------------------------------------

func zoom_to(target_zoom: Vector2, duration: float = 0.3) -> void:
	var tw := create_tween()
	tw.set_trans(Tween.TRANS_SINE)
	tw.tween_property(self, "zoom", target_zoom, duration)


func reset_zoom(duration: float = 0.3) -> void:
	zoom_to(default_zoom, duration)


## Quick FOV widen while dashing (reinforces the Speed pillar). Call on
## dash start; zoom eases back to default_zoom on its own.
func dash_zoom_pulse(zoom_multiplier: float = 1.12, duration: float = 0.15) -> void:
	var pulse_zoom: Vector2 = default_zoom * zoom_multiplier
	var tw := create_tween()
	tw.tween_property(self, "zoom", pulse_zoom, duration * 0.4)
	tw.tween_property(self, "zoom", default_zoom, duration * 0.6)


# ---------------------------------------------------------------
# Shake
# ---------------------------------------------------------------

## duration <= 0.0 triggers an INSTANT shake (single sharp kick, snaps
## back almost immediately) instead of the continuous trauma version.
func shake(power: float = 8.0, duration: float = 0.3,
		frequency: float = default_shake_frequency,
		type: ShakeType = ShakeType.RANDOM) -> void:
	if duration <= 0.0:
		_shake_instant(power, type)
		return
	_shake_type = type
	_shake_frequency = frequency
	_trauma = clampf(_trauma + power / 10.0, 0.0, 1.0)
	_trauma_decay_rate = 1.0 / maxf(duration, 0.01)


## Named presets: HIT_LIGHT, HIT_HEAVY, PARRY_SUCCESS (instant),
## BOSS_INTRO. Add more entries to SHAKE_PRESETS as needed.
func shake_preset(preset_name: StringName) -> void:
	if not SHAKE_PRESETS.has(preset_name):
		push_warning("PlayerCamera: unknown shake preset '%s'" % preset_name)
		return
	var p: Dictionary = SHAKE_PRESETS[preset_name]
	shake(p["power"], p["duration"], p["frequency"], p["type"])


func _shake_instant(power: float, type: ShakeType) -> void:
	offset = _direction_for_type(type) * power
	var tw := create_tween()
	tw.tween_property(self, "offset", Vector2.ZERO, 0.08).set_trans(Tween.TRANS_EXPO)


func _update_trauma(delta: float) -> void:
	_trauma = maxf(_trauma - _trauma_decay_rate * delta, 0.0)


func _compute_shake_offset() -> Vector2:
	if _trauma <= 0.0:
		return Vector2.ZERO
	var amount: float = _trauma * _trauma
	var t: float = Time.get_ticks_msec() / 1000.0
	var dir := _direction_for_type(_shake_type)
	var noise_x := sin(t * _shake_frequency + _shake_seed)
	var noise_y := sin(t * (_shake_frequency * 0.9) + _shake_seed * 2.0)
	return Vector2(noise_x * dir.x, noise_y * dir.y) * amount * 16.0


func _direction_for_type(type: ShakeType) -> Vector2:
	match type:
		ShakeType.HORIZONTAL:
			return Vector2(1.0, 0.0)
		ShakeType.VERTICAL:
			return Vector2(0.0, 1.0)
		_:
			return Vector2(1.0, 1.0)


# ---------------------------------------------------------------
# CameraArea stacking — "smallest active area wins"
# ---------------------------------------------------------------

func _connect_to_camera_manager() -> void:
	if not has_node("/root/CameraManager"):
		push_warning("PlayerCamera: CameraManager autoload not found yet.")
		return
	var mgr := get_node("/root/CameraManager")
	mgr.camera_area_entered.connect(_on_camera_area_entered)
	mgr.camera_area_exited.connect(_on_camera_area_exited)


func _on_camera_area_entered(area: Area2D, limits: Rect2) -> void:
	_active_limits[area.get_instance_id()] = limits
	_apply_smallest_limit()


func _on_camera_area_exited(area: Area2D) -> void:
	_active_limits.erase(area.get_instance_id())
	_apply_smallest_limit()


func _apply_smallest_limit() -> void:
	if _active_limits.is_empty():
		_reset_limits_to_default()
		return
	var smallest: Rect2 = Rect2()
	var smallest_size: float = INF
	for rect in _active_limits.values():
		var size: float = rect.size.x * rect.size.y
		if size < smallest_size:
			smallest_size = size
			smallest = rect
	limit_left = int(smallest.position.x)
	limit_top = int(smallest.position.y)
	limit_right = int(smallest.position.x + smallest.size.x)
	limit_bottom = int(smallest.position.y + smallest.size.y)


func _reset_limits_to_default() -> void:
	limit_left = -10000000
	limit_top = -10000000
	limit_right = 10000000
	limit_bottom = 10000000
```

---

## 4) `CameraArea.gd`

```gdscript
class_name CameraArea
extends Area2D

## CameraArea
## ------------------------------------------------------------------
## Placed per-room / boss arena / special room. Detects ONLY the Player
## (via the "player" group) and broadcasts this room's camera limits
## through the CameraManager signal bus.
##
## WHY v1 broke: Camera2D's limit_left/top/right/bottom are ABSOLUTE
## WORLD PIXEL COORDINATES from the scene's (0,0) origin — not a size,
## not relative to this node. Typing a Rect2 by hand has zero visual
## reference for where (0,0) even is, so the numbers were guesses.
##
## Fix: limits are read from a CHILD CollisionShape2D using a
## RectangleShape2D. That shape IS visible and draggable in the 2D
## viewport — resize it to match the room, and correct absolute world
## coordinates are computed automatically from its real transform.
## No manual numbers, ever.
## ------------------------------------------------------------------

var _cached_limits: Rect2
var _has_valid_limits: bool = false


func _ready() -> void:
	body_entered.connect(_on_body_entered)
	body_exited.connect(_on_body_exited)
	_cache_limits_from_shape()


func _cache_limits_from_shape() -> void:
	var shape_node: CollisionShape2D = null
	for child in get_children():
		if child is CollisionShape2D:
			shape_node = child
			break

	if shape_node == null or shape_node.shape == null:
		push_error("CameraArea '%s': needs a child CollisionShape2D with a RectangleShape2D." % name)
		return

	var rect_shape := shape_node.shape as RectangleShape2D
	if rect_shape == null:
		push_error("CameraArea '%s': the CollisionShape2D's shape must be a RectangleShape2D." % name)
		return

	var half_size: Vector2 = rect_shape.size / 2.0
	var world_center: Vector2 = shape_node.global_position
	_cached_limits = Rect2(world_center - half_size, rect_shape.size)
	_has_valid_limits = true


func _on_body_entered(body: Node2D) -> void:
	if not _has_valid_limits or not body.is_in_group("player"):
		return
	if has_node("/root/CameraManager"):
		get_node("/root/CameraManager").camera_area_entered.emit(self, _cached_limits)


func _on_body_exited(body: Node2D) -> void:
	if not _has_valid_limits or not body.is_in_group("player"):
		return
	if has_node("/root/CameraManager"):
		get_node("/root/CameraManager").camera_area_exited.emit(self)
```

**Setup in the editor:** add a `CollisionShape2D` as a child of `CameraArea`, give it a `RectangleShape2D`, then drag its handles directly in the 2D viewport until it matches the room's visual bounds. That's it — no numbers to type anywhere.

---

## 5) Signals Contract — `CameraManager` (you implement this, unchanged from v1)

```gdscript
# CameraManager.gd (autoload) — skeleton only, you own the implementation
extends Node

signal camera_area_entered(area: Area2D, limits: Rect2)
signal camera_area_exited(area: Area2D)
```

---

## 6) Player Integration Guide

Hook these calls into your existing FSM (`Idle → Move → Jump → Fall → ...` per your Movement System doc):

```gdscript
# on facing flip (Move state, whenever direction changes)
player_camera.set_facing_direction(input_direction_x)

# on is_on_floor() transition (any state)
player_camera.set_grounded(is_on_floor())

# on Fall state enter
player_camera.set_falling(true)
# on Fall state exit (landing)
player_camera.set_falling(false)

# on Dash state enter
player_camera.dash_zoom_pulse()

# on successful parry (example — call from wherever that resolves)
player_camera.shake_preset("PARRY_SUCCESS")

# on boss room CameraArea entered + boss intro cutscene
player_camera.focus_on(boss_node, 1.5)
```

Note: `set_falling(true)` already stops Forward Offset internally — you don't need to call `set_forward_offset_enabled()` separately from the Fall state; it's wired in.

---

## 7) Future Extensions — implemented this pass

| Extension (from v1 §8) | Status |
|---|---|
| Vertical look-ahead during Fall | ✅ Implemented — merged into Vertical Behavior (§3, `fall_lookahead_distance`) |
| Shake presets (HIT_LIGHT, HIT_HEAVY, PARRY_SUCCESS, BOSS_INTRO) | ✅ Implemented — `SHAKE_PRESETS` const + `shake_preset()` |
| `focus_on(target, duration)` | ✅ Implemented — works standalone, doesn't require `CameraManager` |
| Dash-speed FOV widen | ✅ Implemented — `dash_zoom_pulse()` |
| Closing the Shake/Limit gap | ⏸️ Skipped per your instruction this pass |

## 8) Deferred (explicitly postponed by you)

- **Ledge Detection** camera bias — not started, picking this back up later.

---

## 9) Known Trade-off (unchanged, still true, still deliberately left alone)

`shake()`'s continuous mode writes to `offset`, which the engine does **not** clamp against `limit_*`. Amplitude stays small (`power * 16`, trauma-capped at 1.0) — a few pixels to ~16px — well inside normal room-limit margins, so this isn't visibly an issue. If an extreme "screen-breaking" shake ever needs it, the fix is one added clamp step in `_compute_shake_offset()`. Not built now, per your call in §7.
