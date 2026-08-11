### Current Code:
```GDscript
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
@export var forward_offset_smoothing_speed: float = 4.5

@export_group("Vertical Behavior")
@export var vertical_lock_enabled: bool = true
@export var fall_lookahead_distance: float = 200.0
@export var fall_lookahead_smoothing_speed: float = 4.0

@export_group("Camera Area Transitions")
@export var limit_transition_duration: float = 1.5

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
@onready var boss_area: CollisionShape2D = $"../../CameraAreas/BossArea/DetectionShape"


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
	_locked_y = _player.global_position.y # MY CODE - REMOVEABLE ?

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

```GDscript
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
	#...
	facing_direction = 1
	#...

func _physics_process(delta: float) -> void:
	camera.set_grounded(is_on_floor())
	#...
#...
```

```GDscript
extends PlayerState
class_name PlayerFallState # MADE UP
#...
func enter(state_owner: Node2D, state_machine: StateMachine) -> void:
	var player: Player = state_owner as Player
	player.camera.set_falling(true)

func physics_update(delta: float, state_owner: Node2D, state_machine: StateMachine) -> void:
	#...
	if player.velocity.y >= player.max_fall_speed:
		CameraManager.apply_camera_shake_preset(GameConstants.ShakePreset.FAST_FALL)
#...
func exit(state_owner: Node2D, state_machine: StateMachine) -> void:
	var player: Player = state_owner as Player
	player.camera.set_falling(false)
	#...
```

```GDscript
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
	DODGE,
	HIT_LIGHT,
	HIT_HEAVY,
	PARRY_SUCCESS,
	BOSS_INTRO,
}

const SHAKE_PRESETS: Dictionary = {
	ShakePreset.FAST_FALL:     {"power": 4.0,  "duration": 0.05,  "frequency": 60.0, "type": PlayerCamera.ShakeType.RANDOM},
	ShakePreset.FORCED_FALL:   {"power": 6.0,  "duration": 0.2, "frequency": 20.0, "type": PlayerCamera.ShakeType.RANDOM},
	ShakePreset.BIG_FALL:      {"power": 20.0, "duration": 0.5,  "frequency": 70.0, "type": PlayerCamera.ShakeType.RANDOM},
	ShakePreset.DODGE:         {"power": 5.5, "duration": 0.2,  "frequency": 30.0, "type": PlayerCamera.ShakeType.RANDOM},
	#ShakePreset.HIT_LIGHT:     {"power": 4.0,  "duration": 0.15, "frequency": 35.0, "type": PlayerCamera.ShakeType.RANDOM},
	#ShakePreset.HIT_HEAVY:     {"power": 9.0,  "duration": 0.35, "frequency": 25.0, "type": PlayerCamera.ShakeType.RANDOM},
	#ShakePreset.PARRY_SUCCESS: {"power": 3.0,  "duration": 0.0,  "frequency": 0.0,  "type": PlayerCamera.ShakeType.RANDOM},
	#ShakePreset.BOSS_INTRO:    {"power": 6.0,  "duration": 0.6,  "frequency": 15.0, "type": PlayerCamera.ShakeType.RANDOM},
}
```

```GDscript
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

```GDscript
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

> [!NOTE] About The Code
> The `PlayerCamera` and `CameraArea` and `CameraManager` Code Is Done By you *Claude*.
> The Camera System Was Vibe Coded with You and we will work on the new Version of it.

### BUGs and Future Modifications and additional Features:
- ***Fix:*** The First Camera Area Transition Will Reveal The Full Camera Screen Before it transits to the new limits
> [!NOTE] My Answer to Your suggestion to fix the BUG:
> You suggest that setting initial limits for the Player Camera Will fix the problem from it's root but the problem that the PlayerCamera is inside the Player Scene and I can't predict every level size so That's why I used CameraArea system in the first place *Claude*.

- ***ADD:*** Separate Collision that detects player entrance and the collision that detects the player exit in `CameraArea` beside the Limits Collision that sets the limit.
- ***Modify: (Mabye)*** Limits Changing system still looks more like Celeste than like hollow knight I don't remember the details about Hollow knight and silksong camera system so I depend on you *Claude* to search and Fix :) .    
- ***ADD:*** **`CameraArea` priority override.** An optional `@export var priority_override: int = -1` that, when set, wins regardless of size — an escape hatch for the rare room where "smallest wins" picks the wrong one.
