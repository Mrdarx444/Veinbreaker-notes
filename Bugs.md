- [ ] Fix The Repeated Code in the **`States`**
- [ ] Area For Detecting Player Exit in Camera System
- [ ] The First Camera Area Transition Will Reveal The Full Camera Screen Before it transits to the limits

> [!NOTE] Solution suggested
> From CameraArea to Camera Area (Implement Transition time)
> From Open Area (Default Limits) to CameraArea (Don't Implement Transition time (`0.0`))


```GDscript
class_name PlayerCamera
extends Camera2D
#...
@export_group("Camera Area Transitions")
@export var limit_transition_duration: float = 1.5
var current_limit_transition_duration: float = 0.0
#...
func _tween_limits_to(rect: Rect2) -> void:
	#...
	if current_limit_transition_duration == 0:
		current_limit_transition_duration = limit_transition_duration
```
- ***ADD:*** **Hitstop / time-scale hook alongside shake.** Your own `Combos.md` already lists "time scale for heavy blows" as a TODO. Since `shake_preset()` already fires on hit events, it's a natural place to also trigger a brief `Engine.time_scale` dip — same call site, reinforces the same impact, no new event wiring needed.