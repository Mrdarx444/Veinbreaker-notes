# Camera System — Architecture & Implementation

Target: Godot 4.7 · 2D Action-Platformer / Metroidvania · Mobile-First
Reference feel: Hollow Knight / Nine Sols / Silksong — tight, responsive, no visible lag on input.

---

## 1) Design Philosophy

The system is split into **3 pieces**, each with one job:

| Piece | Type | Responsibility |
|---|---|---|
| `PlayerCamera` | `Camera2D`, child of Player scene | Zoom, Forward Offset, Shake |
| `CameraArea` | `Area2D`, placed per-room | Owns a `limits` Rect2, broadcasts it |
| `CameraManager` | Autoload (**you implement**) | Pure signal bus connecting the two above |

### The core architectural decision (this is why the old version broke)

Godot's `Camera2D` has **two different ways** to move the view:

1. **`position`** (the node's actual transform) — this IS clamped by `limit_left/top/right/bottom`, and IS smoothed by `position_smoothing_enabled`.
2. **`offset`** (a draw-time visual nudge) — this is **NOT** clamped by limits. The engine's own docs say it's meant for "looking around or camera shake."

Instead of writing manual clamping/lerp code (what broke last time), this design routes each feature through the property that already does what we want, natively:

- **Forward Offset → uses `position`.** It's a real transform change, so it is *automatically* smoothed and *automatically* limit-clamped by the engine. Zero extra code needed for either.
- **Shake → uses `offset`.** Exactly what the engine designed it for. It is *not* limit-clamped, but shake amplitude is small (a few pixels) and short-lived by design — in practice this never causes visible bounds violations. See **§6 Known Trade-off** if you ever need to close this fully.

This is simpler, uses fewer moving parts, and can't fight itself the way the manual version did.

---

## 2) Setup Requirements

- Player root node must be in the **`"player"`** group (used by `CameraArea` for safe detection — no type-checking, no nil risk).
- Player root node should **not** be flipped/mirrored (`scale.x = -1`) for facing — flip a child sprite/pivot instead. `PlayerCamera` flips its own local offset independently; if the parent also flips, the offset direction would double-invert.
- Each room that needs its own bounds gets a `CameraArea` node with a `CollisionShape2D` (for detection) and an authored `limits` Rect2 (for the actual bound — these are intentionally decoupled, see §4).

---

## 3) `PlayerCamera.gd`

```gdscript
class_name PlayerCamera
extends Camera2D

## PlayerCamera
## ------------------------------------------------------------------
## Lives as a child node inside the Player scene. Basic "follow" is
## free — it moves with its parent through the scene tree. This script
## adds: mobile-default zoom, Forward Offset (look-ahead), and Shake.
##
## See "Camera System.md" §1 for why Forward Offset uses `position`
## (limit-clamped, auto-smoothed) while Shake uses `offset` (not
## limit-clamped, but small/short-lived by design).
## ------------------------------------------------------------------

enum ShakeType { RANDOM, HORIZONTAL, VERTICAL }

@export_group("Zoom")
@export var default_zoom: Vector2 = Vector2(0.6, 0.6)

@export_group("Follow / Smoothing")
@export_range(7.0, 10.0, 0.1) var smoothing_speed: float = 8.0

@export_group("Forward Offset")
@export var forward_offset_distance: float = 80.0

@export_group("Shake Defaults")
@export var default_shake_frequency: float = 30.0

var _facing_direction: int = 1  # 1 = right, -1 = left

var _trauma: float = 0.0
var _trauma_decay_rate: float = 2.0
var _shake_type: ShakeType = ShakeType.RANDOM
var _shake_frequency: float = 30.0
var _shake_seed: float = 0.0

# area instance_id (int) -> Rect2, tracks every CameraArea currently overlapping the player
var _active_limits: Dictionary = {}


func _ready() -> void:
	zoom = default_zoom
	position_smoothing_enabled = true
	position_smoothing_speed = smoothing_speed
	limit_smoothed = true
	_shake_seed = randf() * 1000.0
	make_current()
	_connect_to_camera_manager()


func _physics_process(delta: float) -> void:
	_update_trauma(delta)
	offset = _compute_shake_offset()


# ---------------------------------------------------------------
# Forward Offset
# ---------------------------------------------------------------

## Call this whenever the Player's facing direction changes (once per
## flip, not every frame). Updates local `position.x`, which the engine
## smooths AND clamps against the active CameraArea limits automatically.
func set_facing_direction(dir: int) -> void:
	if dir == 0 or dir == _facing_direction:
		return
	_facing_direction = 1 if dir > 0 else -1
	position.x = forward_offset_distance * _facing_direction


# ---------------------------------------------------------------
# Zoom
# ---------------------------------------------------------------

func zoom_to(target_zoom: Vector2, duration: float = 0.3) -> void:
	var tw := create_tween()
	tw.set_trans(Tween.TRANS_SINE)
	tw.tween_property(self, "zoom", target_zoom, duration)


func reset_zoom(duration: float = 0.3) -> void:
	zoom_to(default_zoom, duration)


# ---------------------------------------------------------------
# Shake
# ---------------------------------------------------------------

## Full-featured shake.
## - power:     amplitude, roughly 0–10 (10 = strong hit)
## - duration:  seconds. <= 0.0 triggers INSTANT shake instead (a single
##              sharp kick that snaps back almost immediately — good
##              for hits, parries, impacts, no sustained oscillation).
## - frequency: oscillation speed for the continuous (non-instant) case.
## - type:      RANDOM (both axes), HORIZONTAL, or VERTICAL only.
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


func _shake_instant(power: float, type: ShakeType) -> void:
	offset = _direction_for_type(type) * power
	var tw := create_tween()
	tw.tween_property(self, "offset", Vector2.ZERO, 0.08).set_trans(Tween.TRANS_EXPO)


func _update_trauma(delta: float) -> void:
	_trauma = maxf(_trauma - _trauma_decay_rate * delta, 0.0)


func _compute_shake_offset() -> Vector2:
	if _trauma <= 0.0:
		return Vector2.ZERO
	var amount: float = _trauma * _trauma  # squared falloff feels punchier than linear
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
	var smallest: Rect2
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
	# Godot's own engine defaults — functionally "no limit"
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
## Placed per-room (or per boss arena / special room). Detects ONLY the
## Player (via the "player" group — safe, no type-check nil risk) and
## broadcasts its `limits` through the CameraManager signal bus.
## PlayerCamera listens and applies the SMALLEST currently-overlapping
## limits (handles stacked/nested areas, e.g. a boss room inside a
## bigger hall).
##
## `limits` is authored manually rather than derived from the
## CollisionShape2D on purpose: the detection trigger (where the player
## crosses to activate this area) and the actual camera bound can be
## different shapes/sizes — e.g. a thin trigger at a doorway applying
## a large room-sized limit.
## ------------------------------------------------------------------

@export var limits: Rect2 = Rect2(0, 0, 1000, 1000)


func _ready() -> void:
	body_entered.connect(_on_body_entered)
	body_exited.connect(_on_body_exited)


func _on_body_entered(body: Node2D) -> void:
	if not body.is_in_group("player"):
		return
	if has_node("/root/CameraManager"):
		get_node("/root/CameraManager").camera_area_entered.emit(self, limits)


func _on_body_exited(body: Node2D) -> void:
	if not body.is_in_group("player"):
		return
	if has_node("/root/CameraManager"):
		get_node("/root/CameraManager").camera_area_exited.emit(self)
```

---

## 5) Signals Contract — required for `CameraManager` (you implement this)

`PlayerCamera.gd` and `CameraArea.gd` call these two signals **by exact name**. Your autoload must expose them with this exact signature or both scripts will error on connect:

```gdscript
# CameraManager.gd (autoload) — skeleton only, you own the implementation
extends Node

signal camera_area_entered(area: Area2D, limits: Rect2)
signal camera_area_exited(area: Area2D)

# Room to grow later, once you need other systems to trigger camera
# behavior (e.g. a boss intro cutscene, a parry-success zoom punch):
# signal request_shake(power: float, duration: float, frequency: float, type: int)
# signal request_zoom(target_zoom: Vector2, duration: float)
# signal request_focus(target: Node2D, duration: float)
```

---

## 6) Known Trade-off (read this before you forget it exists)

`shake()`'s continuous mode writes to `offset`, which the engine does **not** clamp against `limit_*`. In a 2D action-platformer this is a non-issue 99% of the time because shake amplitude (`power * 16`, capped by trauma ≤ 1.0) stays in the range of a few pixels to ~16px — nothing close to typical room-limit margins.

If you ever add an extreme "screen-breaking" shake (e.g. a final boss death event) and it visibly pokes past a level edge, the fix is a single added step in `_compute_shake_offset()`: clamp the returned offset to `(limit_bounds - global_position)` before returning it. Not built now — no reason to pay that complexity cost for a case that hasn't happened yet.

---

## 7) Quick API Reference

| Method | On | Purpose |
|---|---|---|
| `set_facing_direction(dir: int)` | PlayerCamera | Update forward look-ahead (call on flip only) |
| `zoom_to(target_zoom: Vector2, duration: float)` | PlayerCamera | Tween to any zoom |
| `reset_zoom(duration: float)` | PlayerCamera | Tween back to `default_zoom` (0.6) |
| `shake(power, duration, frequency, type)` | PlayerCamera | Continuous or instant shake (`duration <= 0` = instant) |

---

## 8) Future Extensions (not built yet, on purpose)

- **Vertical look-ahead during Fall/Jump** — same `position.y` trick as Forward Offset, small downward bias while falling (Silksong does this subtly).
- **Shake presets** — named constants (`HIT_LIGHT`, `HIT_HEAVY`, `PARRY_SUCCESS`, `BOSS_INTRO`) instead of hand-tuning power/duration/frequency at every call site.
- **`focus_on(target, duration)`** — temporary camera retarget for boss intros / cinematic beats. Not built because it needs `CameraManager`'s `request_focus` signal first — add once that autoload exists.
- **Dash-speed FOV widen** — tiny automatic `zoom_to()` pulse tied to the Dash state, reinforcing the *Speed* pillar.
- **Closing the Shake/Limit gap** — see §6, only if it ever becomes a real visible problem.
