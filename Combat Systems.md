# Combat System — Veinbreaker
**Status:** Draft v1
**Depends On:** `Movement System.md` · `Stamina System.md` · `Ability Tree System.md`
**Mobile-First:** ✅ | **Engine:** Godot 4.x

---

## المبدأ الأساسي: Forced Synergy

لا سلاح واحد أقوى من الثلاثة مجتمعين. هذا **قيد ميكانيكي** لا مجرد تصميم جمالي.

```
Fists ──[Stun]──► Blade ──[Launch]──► Gun ──[Finish]
  ▲                                           │
  └─────────────[Reset / New Enemy]───────────┘
```

| السلاح    | الدور الهجومي           | الدور الدفاعي            | الدور الاقتصادي |
| --------- | ----------------------- | ------------------------ | --------------- |
| **Fists** | يفتح الـ Chain (Stun)   | Counter Attack بعد Parry | يولّد Stamina   |
| **Blade** | يضخّم الدمج على Stunned | Riposte + I-frames       | يستهلك Stamina  |
| **Gun**   | ينهي الـ Chain جواً     | لا Parry                 | يستهلك Stamina  |

---

## النظام 1 — Synergy Chain

### جدول مضاعفات الدمج

| حالة العدو | Fists  | Blade  | Gun    |
| ---------- | ------ | ------ | ------ |
| `NORMAL`   | `1.0×` | `0.8×` | `1.0×` |
| `STUNNED`  | `1.0×` | `1.5×` | `1.2×` |
| `JUGGLED`  | `1.2×` | `1.3×` | `2.0×` |
| `PARRIED`  | `2.0×` | `2.0×` | `2.0×` |

> ⚠️ **Blade على `NORMAL` = 0.8×** — أضعف من الـ Fists. هذا القيد يُجبر اللاعب على Stun أولاً بدون عقاب مباشر.
> **UX مطلوب:** اللاعب يحتاج feedback واضح (اهتزاز خفيف + لون مختلف للرقم) يخبره أن الـ Blade الآن ضعيف. بدون هذا، القيد غير مفهوم.

### Reset Conditions

| الحالة | ماذا يحدث |
|--------|----------|
| Enemy يموت | Chain تنتهي |
| `stun_duration` تنتهي | Enemy يرجع لـ `NORMAL` |
| اللاعب يتوقف عن الضرب `[FILL: combo_timeout]` ثانية | Enemy يرجع لـ `NORMAL` |

---

## النظام 2 — Enemy Hit State Machine

### States

```GDscript
enum HitState {
    NORMAL,   # الحالة الافتراضية
    STUNNED,  # بعد Fists combo
    JUGGLED,  # بعد uppercut / launcher
    PARRIED   # بعد Parry ناجح — نافذة قصيرة جداً
}
```

### Transitions

| من | المُحفّز | إلى |
|----|---------|-----|
| `NORMAL` | Fists combo × `[FILL: hits_to_stun]` | `STUNNED` |
| `NORMAL` | Parry ناجح | `PARRIED` |
| `STUNNED` | Blade launcher | `JUGGLED` |
| `STUNNED` | Timer ينتهي | `NORMAL` |
| `JUGGLED` | Gun aerial / Timer ينتهي | `NORMAL` أو Dead |
| `PARRIED` | Timer ينتهي | `NORMAL` |

### Godot Implementation

```gdscript
# BaseEnemy.gd
class_name BaseEnemy
extends CharacterBody2D

@export var stun_duration: float = 1.5
@export var juggle_duration: float = 2.0
@export var parried_duration: float = 0.4
@export var base_health: float = 100.0

enum HitState { NORMAL, STUNNED, JUGGLED, PARRIED }

var hit_state: HitState = HitState.NORMAL
var _state_timer: float = 0.0
var current_health: float

func _ready() -> void:
    current_health = base_health

func apply_hit_state(new_state: HitState) -> void:
    hit_state = new_state
    match new_state:
        HitState.STUNNED:  _state_timer = stun_duration
        HitState.JUGGLED:  _state_timer = juggle_duration
        HitState.PARRIED:  _state_timer = parried_duration
        HitState.NORMAL:   _state_timer = 0.0

func take_damage(amount: float, multiplier: float = 1.0) -> void:
    current_health -= amount * multiplier
    if current_health <= 0.0:
        die()

func die() -> void:
    queue_free()  # يُستبدل بـ death animation لاحقاً

func _physics_process(delta: float) -> void:
    if hit_state != HitState.NORMAL:
        _state_timer -= delta
        if _state_timer <= 0.0:
            hit_state = HitState.NORMAL
```

الـ Concrete enemies ترث وتعدّل القيم فقط:

```gdscript
# HeavyEnemy.gd
class_name HeavyEnemy
extends BaseEnemy

func _ready() -> void:
    super()
    stun_duration = 0.8   # أصعب في الـ Stun
    juggle_duration = 0.5 # يسقط أسرع
    base_health = 250.0
```

---

## النظام 3 — Stamina Economy

### المبدأ

```
Fists attacks  →  +Stamina
Blade attacks  →  -Stamina
Gun shots      →  -Stamina
Dashes         →  -Stamina
Parry success  →  +Stamina (bonus)
Passive        →  +Stamina/sec (دائماً)
```

> لا عقاب مباشر. لو Stamina وصلت 0: الـ Normal Dash لا تزال متاحة — لكن Shadow Dash ومهارات الـ Ability Tree المتقدمة تُغلق. السرعة محفوظة.

### جدول الاستهلاك والتوليد

| الفعل | Stamina |
|-------|---------|
| Fists light attack | `+[FILL: fists_light_regen]` |
| Fists heavy attack | `+[FILL: fists_heavy_regen]` |
| Parry success (أي حالة) | `+[FILL: parry_stamina_bonus]` |
| Blade attack | `-[FILL: blade_cost]` |
| Gun shot | `-[FILL: gun_cost]` |
| Normal Dash | `-[FILL: normal_dash_cost]` |
| Slide Dash | `-[FILL: slide_dash_cost]` |
| Dodge Dash | `-[FILL: dodge_dash_cost]` |
| Shadow Dash | `-[FILL: shadow_dash_cost]` |
| Passive regen | `+[FILL: passive_regen]` /sec |

### Shadow Dash Condition (مرتبط بـ Movement System)

```gdscript
# في DashState.gd
var can_shadow_dash: bool:
    get: return stamina_component.get_ratio() >= 0.75
```

### Godot Implementation — StaminaComponent (Composition)

```gdscript
# StaminaComponent.gd
class_name StaminaComponent
extends Node

signal stamina_changed(current: float, ratio: float)

@export var max_stamina: float = 100.0
@export var passive_regen_rate: float = 5.0

var current_stamina: float

func _ready() -> void:
    current_stamina = max_stamina

func consume(amount: float) -> bool:
    if current_stamina < amount:
        return false  # رفض — المستدعي يقرر ماذا يفعل
    current_stamina = maxf(0.0, current_stamina - amount)
    stamina_changed.emit(current_stamina, get_ratio())
    return true

func generate(amount: float) -> void:
    current_stamina = minf(max_stamina, current_stamina + amount)
    stamina_changed.emit(current_stamina, get_ratio())

func get_ratio() -> float:
    return current_stamina / max_stamina

func _physics_process(delta: float) -> void:
    generate(passive_regen_rate * delta)
```

الـ Player يضيفه كـ Child Node:

```
Player
└── StaminaComponent  ← $StaminaComponent
```

---

## النظام 4 — Weapon-Specific Parry

### ⚠️ قرار: زر Parry مستقل (الزر الـ 5)

**"Hold Hit = Parry" مرفوض.**

السبب: على موبايل، threshold التمييز بين tap وhold هو ~250ms. في لعبة بـ Speed pillar، اللاعب سيتجاوز الـ threshold أثناء combo spam → Parry accidental → combo ينكسر. هذا يكسر الـ Speed pillar مباشرة.

**القرار: زر مستقل.** 5 أزرار على اليمين قابل للتطبيق (Punishing: Gray Raven, Pascal's Wager, Darkness Rises كلها تتجاوز 4).

```
HUD المقترح:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Joystick]          [Parry] [Switch]
                    [Dash]  [Hit]
                            [Jump]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### نتائج الـ Parry حسب السلاح النشط

| السلاح | نتيجة Parry ناجح | Stamina | ملاحظة |
|--------|-----------------|---------|--------|
| **Fists** | Counter Strike → `STUNNED` + دمج فوري | `+bonus` | يبدأ chain جديدة تلقائياً |
| **Blade** | Riposte → Redirect projectile + I-frames قصيرة | `+bonus` | فقط على projectile — على melee attack = تُغلق فقط |
| **Gun** | لا Parry — input يُهمل | — | Gun وضعها هجومي بحت |

> **Parry Fail = لا عقاب.** اللاعب يتلقى الضربة كالعادة. عدم العقاب مقصود: اللعبة Speed-focused وليست Souls-like.

### Parry Window

```
Parry Input
    │
    ▼
[FILL: parry_window_ms] ms
    │
    ├─ Enemy attack وصل في النافذة → Parry SUCCESS
    └─ لا attack → Parry MISS (لا شيء يحدث)
```

### Godot Implementation — WeaponComponent (Composition)

```gdscript
# WeaponComponent.gd
class_name WeaponComponent
extends Node

enum WeaponType { FISTS, BLADE, GUN }

@export var counter_damage: float = 30.0
@export var parry_stamina_bonus: float = 15.0

var active_weapon: WeaponType = WeaponType.FISTS
var _stamina: StaminaComponent

func _ready() -> void:
    _stamina = get_parent().get_node("StaminaComponent")

func switch_weapon(next: WeaponType) -> void:
    active_weapon = next

func get_damage_multiplier(enemy: BaseEnemy) -> float:
    match active_weapon:
        WeaponType.FISTS:
            match enemy.hit_state:
                BaseEnemy.HitState.JUGGLED:  return 1.2
                _:                            return 1.0
        WeaponType.BLADE:
            match enemy.hit_state:
                BaseEnemy.HitState.NORMAL:   return 0.8
                BaseEnemy.HitState.STUNNED:  return 1.5
                BaseEnemy.HitState.JUGGLED:  return 1.3
                BaseEnemy.HitState.PARRIED:  return 2.0
                _:                            return 1.0
        WeaponType.GUN:
            match enemy.hit_state:
                BaseEnemy.HitState.JUGGLED:  return 2.0
                BaseEnemy.HitState.PARRIED:  return 2.0
                _:                            return 1.0
        _:
            return 1.0

func handle_parry(enemy: BaseEnemy) -> void:
    match active_weapon:
        WeaponType.FISTS:
            enemy.apply_hit_state(BaseEnemy.HitState.STUNNED)
            enemy.take_damage(counter_damage)
            _stamina.generate(parry_stamina_bonus)
        WeaponType.BLADE:
            # Projectile redirect — يُكمل لاحقاً مع Projectile System
            _stamina.generate(parry_stamina_bonus)
        WeaponType.GUN:
            pass
```

---

## Signal Flow الكامل (Godot)

```
Player Input
    │
    ├─[Hit]──► WeaponComponent.get_damage_multiplier(enemy)
    │               └──► enemy.take_damage(base * multiplier)
    │               └──► StaminaComponent.consume(cost)        ← Blade / Gun
    │               └──► StaminaComponent.generate(amount)     ← Fists فقط
    │               └──► enemy.apply_hit_state(STUNNED/JUGGLED)← حسب combo count
    │
    ├─[Switch]──► WeaponComponent.switch_weapon(next)
    │
    ├─[Parry]──► WeaponComponent.handle_parry(enemy)
    │                └──► enemy.apply_hit_state(PARRIED/STUNNED)
    │                └──► StaminaComponent.generate(bonus)
    │
    └─[Dash]──► [Movement System → DashState]
                    └──► StaminaComponent.consume(dash_cost)
                    └──► can_shadow_dash = StaminaComponent.get_ratio() >= 0.75
```

---

## مثال Encounter كامل

```
① Fists combo ×[hits_to_stun]
   → Enemy: NORMAL → STUNNED
   → Stamina: +X

② Switch → Blade
③ Hit (Blade على STUNNED)
   → Damage × 1.5
   → Stamina: -Y

④ Blade launcher
   → Enemy: STUNNED → JUGGLED

⑤ Switch → Gun
⑥ Hit (Gun aerial)
   → Damage × 2.0
   → Stamina: -Z
   → Enemy: Dead أو NORMAL

⑦ عدو جديد يهاجم
⑧ Parry (Fists active)
   → Parry Success
   → Enemy: PARRIED → STUNNED + counter damage
   → Stamina: +Bonus
   → يرجع لـ ①
```

---

## Architecture Summary

```
Player (CharacterBody2D)
│
├── [FSM Inheritance]   PlayerState ← من Movement System
│
├── [Composition]  ──── StaminaComponent (Node)
│                           passive regen
│                           consume / generate / get_ratio
│
├── [Composition]  ──── WeaponComponent (Node)
│                           active_weapon
│                           get_damage_multiplier(enemy)
│                           handle_parry(enemy)
│
└── [Composition]  ──── AimComponent (Node)  ← موجود مسبقاً

BaseEnemy (CharacterBody2D)
│
├── [Inheritance] ───── HeavyEnemy / FastEnemy / RangedEnemy ...
│                           يعدّلون القيم فقط (stun_duration, base_health ...)
│
└── [Built-in] ───────── HitState enum + timer + take_damage()
```

---

## Open Questions

- [ ] هل Blade launcher على JUGGLED يطلق juggle ثانية أم يكمل فقط؟
- [ ] هل Gun تولّد Stamina على Aerial Finish؟ (مكافأة على إكمال الـ Chain)
- [ ] Ability Tree upgrades مرتبطة بالـ Combat: longer stun window، bigger parry window، counter damage+
- [ ] مصادر جمع Sparks (مؤجل لجلسة مستقلة)
- [ ] Second Wind: هل يرتبط بـ Stamina (مثلاً: يُفعّل عند Stamina 0 وHealth منخفضة)؟

---

## الربط مع الأنظمة الأخرى

| النظام | نقطة الربط |
|--------|-----------|
| **Movement System** | `DashState` يستدعي `StaminaComponent.consume()` + يتحقق من `get_ratio()` لـ Shadow Dash |
| **Stamina System** | `StaminaComponent` هو الـ implementation الفعلي |
| **Ability Tree System** | يُفعّل: Wall Jump, Double Jump, Slide Dash, Dodge Dash, Shadow Dash, longer Parry window |
| **Health System** | Parry ناجح = لا تأثير على Health. Parry فاشل = الـ Enemy attack تمر عادي على Health |
| **Enemy System** | `BaseEnemy.HitState` هو bridge بين Combat System وكل enemy |
