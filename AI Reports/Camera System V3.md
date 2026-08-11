# Camera System v3 — Architecture & Implementation

Target: Godot 4.7 · 2D Action-Platformer / Metroidvania · Mobile-First

---

## 0) Report Diagnosis — direct answers first

| Your report | Root cause | Fix |
|---|---|---|
| Fall Look-ahead "seems to not work" → you added `_locked_y = _player.global_position.y` every frame as a fix | **That line was the bug, not the fix.** It overwrote `_locked_y` to the player's live Y every single frame, so the lock could never hold — it always equaled the player's current height. The *actual* cause: your `Player._physics_process()` never calls `camera.set_falling(...)` anywhere. Without it, `_is_falling` stays `false` forever, so look-ahead never activates. What you saw (Y frozen while airborne, snap on landing) was Vertical Lock working *correctly* — just without the look-ahead easing on top, because that half of the feature was never told a fall was happening. | Remove your `_locked_y` line (§3). Add one line to `Player._physics_process()` (§7): `camera.set_falling(not on_floor and velocity.y > 0.0)`. |
| `shake()` "lacks a speed argument, frequency isn't enough" | `frequency` already **is** the speed control — the real bug is upstream: `power` was divided by 10 and fed into a trauma value hard-clamped to `1.0`. Any `power` above ~10 collapsed to the *same* intensity. Your `BIG_FALL` preset (`power: 70`) was rendering at the exact same amplitude as `HIT_HEAVY` (`power: 9`) — that's why it read as "not enough," it wasn't a frequency problem at all. | `power` is now used directly as **pixel amplitude**, no hidden division, no cap. `BIG_FALL` now actually shakes ~70px, `HIT_LIGHT` ~4px. No new parameter needed — rejecting that suggestion, the fix is the scaling bug. |
| "shake should have instant mode if `duration <= 0`" | Already implemented in the code you have — but it used **raw `power` as pixels** while the continuous path used the buggy `/10` scale. Same preset, two different amplitudes depending on `duration`. | Fixed as a side effect of the power-scaling fix above: both paths now use the same direct-pixel meaning of `power`. |
| Presets should live in an autoload with editor autocompletion | `StringName` doesn't autocomplete — Godot has nothing to suggest from a loose string. | New `GameConstants` autoload with a `ShakePreset` **enum**. `GameConstants.ShakePreset.` now autocompletes in the editor. |
| CameraArea transitions feel instant / depend on follow speed | Two separate issues, both real: (1) `limit_*` were snapped instantly, no tween of their own; (2) the detection trigger and the limit bounds were the same shape, so the limit change happened exactly at the moment the player crossed into the new room — no buffer to hide the transition in. | (1) `limit_left/top/right/bottom` now tween over `limit_transition_duration`. (2) `CameraArea` now supports a separate, smaller **DetectionShape** so the trigger fires before the player reaches the room's true edge, giving the tween time to finish unnoticed. |
| `focus_on()` needs its own transition time, not tied to follow speed | Confirmed correct ask — in v2 the pan-in rode on `position_smoothing_speed`, which is tuned for tight gameplay follow, not deliberate cinematic pacing. | Rewritten: takes a `Vector2` (not a `Node2D` — removes a dangling-reference risk if the focused node gets freed mid-focus) plus an explicit `transition_duration`, fully Tween-driven, `position_smoothing_enabled` temporarily disabled so nothing fights the tween. |

---

## 1) Architecture

```
   Player (facing / grounded / falling)
                │
                ▼
   PlayerCamera._physics_process()
        ├─ forward offset (own smoothing) ──┐
        ├─ vertical lock / fall look-ahead ─┼──► target Vector2 ──► global_position
        └─ focus_on() ───────────────────────► (Tween takes exclusive control, bypasses the above)
                                                     (engine smooths + clamps, except during focus)

        └─ shake (independent) ──────────────────► offset (still not limit-clamped — unchanged, deliberate)

   CameraArea (per room)
        DetectionShape (enabled, smaller)  → triggers entry/exit
        LimitsShape (disabled, WYSIWYG)    → defines the actual bound
                │
                ▼
   CameraManager (signal bus, your implementation)
                │
                ▼
   PlayerCamera applies smallest overlapping limit, tweened in

   GameConstants (autoload) → ShakePreset enum + SHAKE_PRESETS data
```

---

## 2) Setup Requirements

- Player root in the **`"player"`** group, and typed `class_name Player`.
- Player root not mirrored via `scale.x = -1` on the root itself.
- `PlayerCamera` is a direct child of Player.
- `CameraArea` needs:
  - a **`DetectionShape`** child (`CollisionShape2D`, enabled, `RectangleShape2D`) — the trigger. Keep it inset from the room's true edges.
  - an optional **`LimitsShape`** child (`CollisionShape2D`, **disabled**, `RectangleShape2D`) — the true camera bound. Disabled shapes are still visible/draggable in the 2D viewport, just excluded from physics. If omitted, `DetectionShape` is used for both (fine for small rooms).
- Register **`GameConstants.gd`** as an autoload, same as `CameraManager.gd`.

---

## 3) `PlayerCamera.gd`

```gdscript
class_name PlayerCamera
extends Camera2D

## PlayerCamera (v3)
## ------------------------------------------------------------------
## Child of the Player scene. Every physics frame computes ONE target
## position and assigns it to global_position once (X = follow + own-
## smoothing forward offset, Y = normal follow OR vertical-locked +
## fall-lookahead while airborne). Engine smoothing + limit_* clamp
## the result automatically, except during focus_on() where a Tween
## takes exclusive control. Shake stays on `offset` (still not limit-
## clamped by the engine — unchanged trade-off, still deliberate,
## see §Known Trade-off at the bottom of the doc).
##
## v3 fixes (see "Camera System 3.md" §0 for full diagnosis):
## - shake() power is now a DIRECT pixel amplitude (was silently
##   capped above power≈10 by a /10 + trauma-clamp bug).
## - Shake presets moved to GameConstants (enum, autocompletes).
## - focus_on() takes a Vector2 + its own transition_duration.
## - Do NOT add a per-frame `_locked_y = _player.global_position.y`
##   line — that defeats Vertical Lock entirely. Call set_falling()
##   from Player instead (see Player.gd snippet, §7).
## ------------------------------------------------------------------

enum ShakeType { RANDOM, HORIZONTAL, VERTICAL }

@export_group("Zoom")
@export var default_zoom: Vector2 = Vector2(0.6, 0.6)

@export_group("Follow / Smoothing")
@export_range(2.0, 12.0, 0.1) var follow_smoothing_speed: float = 8.0

@export_group("Forward Offset")
@export var forward_offset_distance: float = 70.0
@export var forward_offset_smoothing_speed: float = 5.0

@export_group("Vertical Behavior")
@export var vertical_lock_enabled: bool = true
@export var fall_lookahead_distance: float = 200.0
@export var fall_lookahead_smoothing_speed: float = 3.0

@export_group("Camera Area Transitions")
@export var limit_transition_duration: float = 0.45

@export_group("Shake Defaults")
@export var default_shake_frequency: float = 30.0

var _player: Player = null

var _facing_direction: int = 1
var _forward_offset_enabled: bool = true
var _target_forward_offset: float = 0.0
var _current_forward_offset: float = 0.0

var _is_grounded: bool = true
var _is_falling: bool = false
var _locked_y: float = 0.0
var _current_fall_lookahead: float = 0.0

var _is_focusing: bool = false

var _trauma: float = 0.0
var _trauma_decay_rate: float = 2.0
var _shake_power: float = 0.0   # direct pixel amplitude, not /10-scaled
var _shake_type: ShakeType = ShakeType.RANDOM
var _shake_frequency: float = 30.0
var _shake_seed: float = 0.0

var _active_limits: Dictionary = {}  # area.instance_id -> Rect2
var _limit_tween: Tween = null


func _ready() -> void:
	_player = get_parent() as Player
	if _player == null:
		push_error("PlayerCamera: must be a direct child of a Player node. Reparent it.")

	zoom = default_zoom
	position_smoothing_enabled = true
	position_smoothing_speed = follow_smoothing_speed
	limit_smoothed = true
	_shake_seed = randf() * 1000.0
	_locked_y = global_position.y
	make_current()
	_connect_to_camera_manager()


func _connect_to_camera_manager() -> void:
	if not has_node("/root/CameraManager"):
		push_warning("PlayerCamera: CameraManager autoload not found yet.")
		return
	CameraManager.change_facing_direction.connect(set_facing_direction)
	CameraManager.camera_shake.connect(shake)
	CameraManager.camera_shake_preset.connect(shake_preset)
	CameraManager.camera_area_entered.connect(_on_camera_area_entered)
	CameraManager.camera_area_exited.connect(_on_camera_area_exited)


func _physics_process(delta: float) -> void:
	if not is_instance_valid(_player):
		return

	_update_trauma(delta)
	offset = _compute_shake_offset()

	if _is_focusing:
		return  # a Tween owns global_position exclusively during focus_on()

	_update_forward_offset(delta)
	_update_vertical_behavior(delta)

	var target_x: float = _player.global_position.x + _current_forward_offset
	var target_y: float = _compute_target_y()
	global_position = Vector2(target_x, target_y)


# ---------------------------------------------------------------
# Forward Offset
# ---------------------------------------------------------------

func set_facing_direction(dir: int) -> void:
	if dir == 0:
		return
	_facing_direction = 1 if dir > 0 else -1
	_recompute_forward_offset_target()


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
# Vertical Behavior
# ---------------------------------------------------------------

## Call every physics frame from Player with is_on_floor().
func set_grounded(grounded: bool) -> void:
	if grounded == _is_grounded:
		return
	_is_grounded = grounded
	if not grounded:
		_locked_y = global_position.y


## Call every physics frame from Player. See §7 for the exact line —
## this is the call that was missing and caused the reported bug.
func set_falling(falling: bool) -> void:
	if falling == _is_falling:
		return
	_is_falling = falling
	set_forward_offset_enabled(not falling)


func _update_vertical_behavior(delta: float) -> void:
	var target_lookahead: float = fall_lookahead_distance if (_is_falling and not _is_grounded) else 0.0
	_current_fall_lookahead = lerpf(
		_current_fall_lookahead, target_lookahead, fall_lookahead_smoothing_speed * delta
	)


func _compute_target_y() -> float:
	if _is_grounded or not vertical_lock_enabled:
		return _player.global_position.y
	return _locked_y + _current_fall_lookahead


# ---------------------------------------------------------------
# Focus (cinematic — boss intros, cutscene beats)
# ---------------------------------------------------------------

## Pans to a world-space point over `transition_duration`, holds for
## `hold_duration`, then pans back to the Player over the same
## transition_duration. Takes a Vector2 (not a Node2D) so there's no
## dangling-reference risk if whatever you were focusing gets freed
## mid-focus — read its global_position once before calling this.
func focus_on(target_position: Vector2, transition_duration: float = 0.6, hold_duration: float = 1.0) -> void:
	_is_focusing = true
	position_smoothing_enabled = false  # tween owns the motion, no double-smoothing on top

	var in_tween := create_tween()
	in_tween.set_trans(Tween.TRANS_SINE).set_ease(Tween.EASE_IN_OUT)
	in_tween.tween_property(self, "global_position", target_position, transition_duration)
	await in_tween.finished

	await get_tree().create_timer(hold_duration).timeout

	var return_position: Vector2 = _player.global_position if is_instance_valid(_player) else global_position
	var out_tween := create_tween()
	out_tween.set_trans(Tween.TRANS_SINE).set_ease(Tween.EASE_IN_OUT)
	out_tween.tween_property(self, "global_position", return_position, transition_duration)
	await out_tween.finished

	position_smoothing_enabled = true
	_is_focusing = false


# ---------------------------------------------------------------
# Zoom
# ---------------------------------------------------------------

func zoom_to(target_zoom: Vector2, duration: float = 0.3) -> void:
	var tw := create_tween()
	tw.set_trans(Tween.TRANS_SINE)
	tw.tween_property(self, "zoom", target_zoom, duration)


func reset_zoom(duration: float = 0.3) -> void:
	zoom_to(default_zoom, duration)


func dash_zoom_pulse(zoom_multiplier: float = 1.05, duration: float = 0.5) -> void:
	var pulse_zoom: Vector2 = default_zoom * zoom_multiplier
	var tw := create_tween()
	tw.tween_property(self, "zoom", pulse_zoom, duration * 0.4)
	tw.tween_property(self, "zoom", default_zoom, duration * 0.6)


# ---------------------------------------------------------------
# Shake
# ---------------------------------------------------------------

## power is now a DIRECT pixel amplitude — see §0 diagnosis for why
## v2's power scaling silently broke anything above ~power=10.
func shake(power: float = 8.0, duration: float = 0.3,
		frequency: float = default_shake_frequency,
		type: ShakeType = ShakeType.RANDOM) -> void:
	if duration <= 0.0:
		_shake_instant(power, type)
		return
	_shake_type = type
	_shake_frequency = frequency
	_shake_power = maxf(_shake_power, power)  # overlapping shakes: keep the stronger one
	_trauma = 1.0
	_trauma_decay_rate = 1.0 / maxf(duration, 0.01)


## Looks up GameConstants.ShakePreset (enum) — autocompletes in the
## editor, unlike the old raw StringName version.
func shake_preset(preset: GameConstants.ShakePreset) -> void:
	if not GameConstants.SHAKE_PRESETS.has(preset):
		push_warning("PlayerCamera: unknown shake preset '%s'" % preset)
		return
	var p: Dictionary = GameConstants.SHAKE_PRESETS[preset]
	shake(p["power"], p["duration"], p["frequency"], p["type"])


func _shake_instant(power: float, type: ShakeType) -> void:
	offset = _direction_for_type(type) * power
	var tw := create_tween()
	tw.tween_property(self, "offset", Vector2.ZERO, 0.08).set_trans(Tween.TRANS_EXPO)


func _update_trauma(delta: float) -> void:
	_trauma = maxf(_trauma - _trauma_decay_rate * delta, 0.0)
	if _trauma <= 0.0:
		_shake_power = 0.0


func _compute_shake_offset() -> Vector2:
	if _trauma <= 0.0:
		return Vector2.ZERO
	var envelope: float = _trauma * _trauma
	var t: float = Time.get_ticks_msec() / 1000.0
	var dir := _direction_for_type(_shake_type)
	var noise_x := sin(t * _shake_frequency + _shake_seed)
	var noise_y := sin(t * (_shake_frequency * 0.9) + _shake_seed * 2.0)
	return Vector2(noise_x * dir.x, noise_y * dir.y) * envelope * _shake_power


func _direction_for_type(type: ShakeType) -> Vector2:
	match type:
		ShakeType.HORIZONTAL:
			return Vector2(1.0, 0.0)
		ShakeType.VERTICAL:
			return Vector2(0.0, 1.0)
		_:
			return Vector2(1.0, 1.0)


# ---------------------------------------------------------------
# CameraArea stacking + smoothed limit transitions
# ---------------------------------------------------------------

func _on_camera_area_entered(area: Area2D, limits: Rect2) -> void:
	_active_limits[area.get_instance_id()] = limits
	_apply_smallest_limit()


func _on_camera_area_exited(area: Area2D) -> void:
	_active_limits.erase(area.get_instance_id())
	_apply_smallest_limit()


func _apply_smallest_limit() -> void:
	if _active_limits.is_empty():
		_tween_limits_to(Rect2(Vector2(-10000000, -10000000), Vector2(20000000, 20000000)))
		return
	var smallest: Rect2 = Rect2()
	var smallest_size: float = INF
	for rect in _active_limits.values():
		var size: float = rect.size.x * rect.size.y
		if size < smallest_size:
			smallest_size = size
			smallest = rect
	_tween_limits_to(smallest)


## v3: limits tween into place instead of snapping — combined with an
## inset DetectionShape (see CameraArea.gd), the player crosses into
## the new room before the bound finishes moving, hiding the switch.
func _tween_limits_to(rect: Rect2) -> void:
	if _limit_tween and _limit_tween.is_valid():
		_limit_tween.kill()
	_limit_tween = create_tween()
	_limit_tween.set_parallel(true)
	_limit_tween.tween_property(self, "limit_left", int(rect.position.x), limit_transition_duration)
	_limit_tween.tween_property(self, "limit_top", int(rect.position.y), limit_transition_duration)
	_limit_tween.tween_property(self, "limit_right", int(rect.position.x + rect.size.x), limit_transition_duration)
	_limit_tween.tween_property(self, "limit_bottom", int(rect.position.y + rect.size.y), limit_transition_duration)
```

---

## 4) `CameraArea.gd`

```gdscript
class_name CameraArea
extends Area2D

## CameraArea (v3)
## ------------------------------------------------------------------
## Two SEPARATE shapes, both WYSIWYG (draggable in the 2D viewport):
##   - "DetectionShape": a normal, ENABLED CollisionShape2D. Triggers
##     body_entered/exited. Keep it INSET (smaller than) the room's
##     true bounds — the player then crosses into the new area a bit
##     before the camera limits finish tweening, hiding the switch.
##   - "LimitsShape": a DISABLED CollisionShape2D (disabled = true, so
##     it never affects physics) whose size/position define the actual
##     camera bounds. Disabled shapes are still fully visible/draggable
##     in the editor — you keep exact, independent visual control over
##     both rects.
## If no "LimitsShape" child exists, DetectionShape is used for both
## (fine for small rooms where the distinction doesn't matter).
## ------------------------------------------------------------------

@export var active: bool = true

var _cached_limits: Rect2
var _has_valid_limits: bool = false


func _ready() -> void:
	if not active:
		return
	body_entered.connect(_on_body_entered)
	body_exited.connect(_on_body_exited)
	_cache_limits_from_shape()


func _cache_limits_from_shape() -> void:
	var shape_node: CollisionShape2D = get_node_or_null("LimitsShape") as CollisionShape2D
	if shape_node == null:
		shape_node = _find_any_collision_shape()

	if shape_node == null or shape_node.shape == null:
		push_error("CameraArea '%s': needs a CollisionShape2D (named 'LimitsShape', or any CollisionShape2D) with a RectangleShape2D." % name)
		return

	var rect_shape := shape_node.shape as RectangleShape2D
	if rect_shape == null:
		push_error("CameraArea '%s': the limits shape must be a RectangleShape2D." % name)
		return

	var half_size: Vector2 = rect_shape.size / 2.0
	var world_center: Vector2 = shape_node.global_position
	_cached_limits = Rect2(world_center - half_size, rect_shape.size)
	_has_valid_limits = true


func _find_any_collision_shape() -> CollisionShape2D:
	for child in get_children():
		if child is CollisionShape2D:
			return child
	return null


func _on_body_entered(body: Node2D) -> void:
	if not active or not _has_valid_limits:
		return
	if not (body.is_in_group("player") and body is Player):
		return
	if has_node("/root/CameraManager"):
		CameraManager.camera_area_entered.emit(self, _cached_limits)


func _on_body_exited(body: Node2D) -> void:
	if not active or not _has_valid_limits:
		return
	if not (body.is_in_group("player") and body is Player):
		return
	if has_node("/root/CameraManager"):
		CameraManager.camera_area_exited.emit(self)
```

---

## 5) `GameConstants.gd` (new autoload)

```gdscript
extends Node

# GameConstants — AUTOLOAD
# Project-wide constants that benefit from editor autocompletion.
# Shake presets live here (not in PlayerCamera) specifically so typing
# `GameConstants.ShakePreset.` autocompletes the preset names — a
# StringName gives the editor nothing to suggest.

enum ShakePreset {
	FAST_FALL,
	FORCED_FALL,
	BIG_FALL,
	HIT_LIGHT,
	HIT_HEAVY,
	PARRY_SUCCESS,
	BOSS_INTRO,
}

const SHAKE_PRESETS: Dictionary = {
	ShakePreset.FAST_FALL:     {"power": 2.0,  "duration": 0.1,  "frequency": 10.0, "type": PlayerCamera.ShakeType.RANDOM},
	ShakePreset.FORCED_FALL:   {"power": 6.0,  "duration": 0.35, "frequency": 20.0, "type": PlayerCamera.ShakeType.RANDOM},
	ShakePreset.BIG_FALL:      {"power": 70.0, "duration": 0.7,  "frequency": 70.0, "type": PlayerCamera.ShakeType.RANDOM},
	ShakePreset.HIT_LIGHT:     {"power": 4.0,  "duration": 0.15, "frequency": 35.0, "type": PlayerCamera.ShakeType.RANDOM},
	ShakePreset.HIT_HEAVY:     {"power": 9.0,  "duration": 0.35, "frequency": 25.0, "type": PlayerCamera.ShakeType.RANDOM},
	ShakePreset.PARRY_SUCCESS: {"power": 3.0,  "duration": 0.0,  "frequency": 0.0,  "type": PlayerCamera.ShakeType.RANDOM},
	ShakePreset.BOSS_INTRO:    {"power": 6.0,  "duration": 0.6,  "frequency": 15.0, "type": PlayerCamera.ShakeType.RANDOM},
}
```

---

## 6) `CameraManager.gd` (your existing autoload — one required update)

Only the shake-preset type changed (`StringName` → `GameConstants.ShakePreset`) to match §5. Everything else is exactly what you already had and confirmed working.

```gdscript
extends Node

# CameraManager — AUTOLOAD

signal camera_shake(power: float, duration: float, frequency: float, type: PlayerCamera.ShakeType)
signal camera_shake_preset(preset: GameConstants.ShakePreset)
signal change_facing_direction(new_dir: int)
signal camera_area_entered(area: CameraArea, limits: Rect2)
signal camera_area_exited(area: CameraArea)

func set_facing_direction(dir: int) -> void:
	change_facing_direction.emit(dir)

func apply_camera_shake(power: float, duration: float, frequency: float, type: PlayerCamera.ShakeType) -> void:
	camera_shake.emit(power, duration, frequency, type)

func apply_camera_shake_preset(preset: GameConstants.ShakePreset) -> void:
	if GameConstants.SHAKE_PRESETS.has(preset):
		camera_shake_preset.emit(preset)
```

---

## 7) `Player.gd` — the missing integration line

```gdscript
extends CharacterBody2D
class_name Player
#...
var facing_direction: int = 0:
	set(dir):
		if facing_direction != dir:
			facing_direction = dir
			CameraManager.set_facing_direction(dir)
#...
func _ready() -> void:
	set_timers()
	debug_labels_container.visible = DEBUG_MODE
	facing_direction = 1
#...
func _physics_process(delta: float) -> void:
	var on_floor := is_on_floor()
	camera.set_grounded(on_floor)
	camera.set_falling(not on_floor and velocity.y > 0.0)  # <-- this line was missing
	if joystick.move_direction: facing_direction = int(joystick.move_direction)
	if DEBUG_MODE: _debug()
#...
```

`velocity.y > 0.0` = descending (Godot's Y-axis points down) — this is exactly your Fall state's condition, so it doubles as a drop-in check without needing to know your FSM's internals. If you already flag Fall state entry/exit elsewhere, calling `camera.set_falling(true/false)` directly from those transitions instead is equally correct — just make sure it's called from *somewhere*.

---

## 8) Quick API Reference (v3 changes marked)

| Method | Change |
|---|---|
| `set_facing_direction(dir: int)` | unchanged |
| `set_grounded(grounded: bool)` | unchanged |
| `set_falling(falling: bool)` | now guards against redundant calls (safe to call every frame) |
| `focus_on(target_position: Vector2, transition_duration: float, hold_duration: float)` | **changed** — was `focus_on(target: Node2D, duration: float)` |
| `shake(power, duration, frequency, type)` | **power is now direct pixel amplitude** (was `/10`-scaled and silently capped) |
| `shake_preset(preset: GameConstants.ShakePreset)` | **changed** — was `shake_preset(preset_name: StringName)` |
| `zoom_to(...)`, `reset_zoom(...)`, `dash_zoom_pulse(...)` | unchanged |

---

## 9) Additional Suggestions (not built — proposals only)

- **In-editor debug draw for CameraArea.** A `@tool` `_draw()` override on `CameraArea` that paints its cached limit rect with a label directly in the 2D viewport, visible without selecting the child `LimitsShape`. Given how much the "invisible limits" problem cost you already, this is the highest-value next addition — worth doing before you place many more rooms.
- **`CameraArea` priority override.** An optional `@export var priority_override: int = -1` that, when set, wins regardless of size — an escape hatch for the rare room where "smallest wins" picks the wrong one.
- **Hitstop / time-scale hook alongside shake.** Your own `Combos.md` already lists "time scale for heavy blows" as a TODO. Since `shake_preset()` already fires on hit events, it's a natural place to also trigger a brief `Engine.time_scale` dip — same call site, reinforces the same impact, no new event wiring needed.

---

## 10) Known Trade-off (unchanged, still deliberate, still not built)

`shake()` writes to `offset`, which the engine does not clamp against `limit_*`. With `power` now a direct pixel value, a `BIG_FALL`-scale shake (~70px) is large enough that this is worth keeping in mind near tight room edges — though still not built, per your call in the v2 pass. Flagging again now that amplitudes are bigger and real than they were with the old scaling bug.