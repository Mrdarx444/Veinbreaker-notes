نعم. بما أن الـ Built-in Joystick في Godot 4.7 يكتب قيمه في الـ InputMap، فأنت تحصل في النهاية على Vector2 عادي من:

```gdscript
var aim := Input.get_vector(
	"move_left",
	"move_right",
	"move_up",
	"move_down"
)
```

بعدها يمكنك تحويل الـ Vector إلى زاوية واستنتاج أي Zone ينتمي إليها.

---

## الطريقة 1: باستخدام angle()

```gdscript
var angle := rad_to_deg(aim.angle())
```

ستحصل على:

```text
Right      = 0°
Up         = -90°
Left       = 180°
Down       = 90°
```

---

## مثال تقسيم 8 اتجاهات

```gdscript
func get_direction_zone(v: Vector2) -> String:

	if v.length() < 0.2:
		return "IDLE"

	var angle := rad_to_deg(v.angle())

	if angle < 0:
		angle += 360

	if angle >= 337.5 or angle < 22.5:
		return "RIGHT"

	elif angle < 67.5:
		return "UP_RIGHT"

	elif angle < 112.5:
		return "UP"

	elif angle < 157.5:
		return "UP_LEFT"

	elif angle < 202.5:
		return "LEFT"

	elif angle < 247.5:
		return "DOWN_LEFT"

	elif angle < 292.5:
		return "DOWN"

	else:
		return "DOWN_RIGHT"
```

---

## لكن في حالتك

أنت لا تريد 8 اتجاهات متساوية.

من الرسم:

```text
UP
|
UP_RIGHT
|
RIGHT
|
DOWN_RIGHT
|
DOWN
```

أي أن الأعلى والأسفل Zones صغيرة ومخصصة للثبات فقط.

يمكنك فعل هذا:

```gdscript
func get_zone(v: Vector2) -> String:

	if v.length() < 0.2:
		return "IDLE"

	var angle := rad_to_deg(v.angle())

	if angle < 0:
		angle += 360

	# UP
	if angle > 250 and angle < 290:
		return "UP"

	# DOWN
	if angle > 70 and angle < 110:
		return "DOWN"

	# RIGHT
	if angle >= 330 or angle <= 30:
		return "RIGHT"

	# LEFT
	if angle >= 150 and angle <= 210:
		return "LEFT"

	# UP RIGHT
	if angle > 290 and angle < 330:
		return "UP_RIGHT"

	# UP LEFT
	if angle > 210 and angle < 250:
		return "UP_LEFT"

	# DOWN RIGHT
	if angle > 30 and angle < 70:
		return "DOWN_RIGHT"

	# DOWN LEFT
	if angle > 110 and angle < 150:
		return "DOWN_LEFT"

	return "UNKNOWN"
```

---

## طريقة أفضل (أنصح بها)

بدل مقارنة الزوايا مباشرة، استخدم Dot Product.

عرّف الاتجاهات المسموح بها:

```gdscript
const DIRECTIONS := {
	"RIGHT": Vector2.RIGHT,
	"LEFT": Vector2.LEFT,
	"UP": Vector2.UP,
	"DOWN": Vector2.DOWN,
	"UP_RIGHT": Vector2(1,-1).normalized(),
	"UP_LEFT": Vector2(-1,-1).normalized(),
	"DOWN_RIGHT": Vector2(1,1).normalized(),
	"DOWN_LEFT": Vector2(-1,1).normalized(),
}
```

ثم:

```gdscript
func get_zone(v: Vector2) -> String:

	if v.length() < 0.2:
		return "IDLE"

	v = v.normalized()

	var best_dot := -999.0
	var best_name := ""

	for name in DIRECTIONS:

		var dot := v.dot(DIRECTIONS[name])

		if dot > best_dot:
			best_dot = dot
			best_name = name

	return best_name
```

ميزات هذه الطريقة:

- لا تهتم بالزوايا.
    
- يمكن إضافة اتجاهات جديدة بسهولة.
    
- أكثر مرونة إذا غيرت تقسيم الـ Zones لاحقًا.
    
- مستخدمة كثيرًا في ألعاب الـ 8-way shooters.
    

لو كنت سأبني نظام التصويب الذي رسمته، فسأختار طريقة الـ Dot Product لأنها أسهل في التعديل والصيانة من عشرات مقارنات الزوايا.