# حلول هندسية لمخاطر Pitfalls.md

> **منهجية هذا الملف**: لكل pitfall نتبع التسلسل التالي:
> 1. **إعادة صياغة المشكلة** (لفهم الجذر وليس الأعراض).
> 2. **تحليل**: لماذا المشكلة مهمة؟ ما الذي يكسر لو تُرِكت؟
> 3. **3 بدائل تصميمية** (A/B/C) مع إيجابيات/سلبيات كل واحد.
> 4. **التوصية + الهندسة التقنية**: الكود + state diagrams + data structures.
> 5. **الـ Plot integration**: كيف يربط الحل بالقصة (لو ممكن).

---

## Pitfall #1: كيف يطوّر اللاعب الـ Skill Tree؟

> [!Missing] How does the player upgrade The Skill Tree System?
> - Under what circumstances does a player earn skill points?
> - What is the skill points even called and what's the plot for them?
> - Under what circumstances can a player Open the Tree Menu and upgrade?

### 1.1 إعادة صياغة المشكلة
المشكلة ليست فقط "كيف يكسب اللاعب نقاط" — هي **أعمق**: كيف نربط نظام الـ progression بنظام القصة بنظام الـ combat بنظام الاستكشاف، بحيث:
- اللاعب يحس أن كل نقطة skill **مكسب** وليست grinding.
- الـ plot يبرر ميكانيكياً لماذا اللاعب يتحسن.
- الـ upgrade moment يكون **ritual** وليس قائمة جافة.

### 1.2 تحليل المشكلة
- **المشكلة الجذرية**: كل الأنظمة الأخرى (Combat، Movement، Stealth) تعتمد على Skill Tree، فبدون تصميم واضح لـ "كيف تكسب points"، كل الـ balance معلّق.
- **لماذا يكسر اللعبة لو تُرِك**: لو الـ points سهلة جداً → اللاعب يفتح كل شيء في 2 ساعة → يفقد الإحساس بالـ progression. لو صعبة جداً → grinding → ملل.
- **في ألعاب مشابهة**:
  - **Hollow Knight**: تدفع بـ Geo (currency) لـ charms + slots. الفصل بين "اشتراء" و "تجهيز" يخلق layer of choice.
  - **Dead Cells**: cells تُكسب من الأعداء، تُدفع لـ upgrades دائمة. مفهوم "metaprogression".
  - **Ori & Will of the Wisps**: نقاط من إكمال الـ combat + exploration، تُدفع في شجرة بسيطة.
  - **Salt and Sanctuary**: salt = XP + currency معاً (موحّد).

### 1.3 البدائل الثلاثة

#### 🅰️ بديل A: Sanguis System (الموصى به) — دمج القصة + الميكانيك
- **الاسم**: `Sanguis` (لاتيني = دم، يربط باسم Veinbreaker).
- **المصدر**: 
  - كل عدو مقتول يعطي `1-3 Sanguis`.
  - كل boss يعطي `50-100 Sanguis`.
  - كل checkpoint "تنفّس فيه اللاعب" يعطي `+10 Sanguis` (مكافأة على الاستكشاف).
  - كل secret area يعطي `+25 Sanguis`.
- **الإنفاق**: عند **Vein Altars** (مذابح وردية اللون منتشرة في الخريطة، كل ~3 غرف).
- **القصة**: الـ Veinbreakers هم محاربون قديمون شربوا دم آلهة قديمة ليكسبوا قوة. كل عدو مقتول "يطلق" دمه القديم، واللاعب يمتصه. الـ Altars هي بقايا معابدهم.
- **الـ Menu**: 
  - يفتح فقط عند Vein Altar (لا upgrade في أي مكان).
  - عند الـ Altar: full menu شجري مع preview.
  - هذا يخلق **ritual feel** — يجب أن تعود للمذبح لـ "drink the blood of your enemies".
- **الإيجابيات**:
  - يربط القصة + الميكانيك + الخريطة في نظام واحد.
  - يحفّز الاستكشاف (altars منتشرة).
  - يمنح الـ checkpoint معنى مزدوج.
- **السلبيات**:
  - اللاعب لا يقدر يـ upgrade في أي لحظة = إحباط لو مات قبل ما يصل altar.
  - الحل: عند الموت، يرجع آخر altar زاره (فيمحفظ نقطته + يمكنه upgrade قبل retry).

#### 🅱️ بديل B: Skill Points تقليدية (XP)
- **الاسم**: `Resonance` (تناغم).
- **المصدر**: XP من القتل + quests.
- **الإنفاق**: في أي وقت من menu (paused).
- **الإيجابيات**: مألوف، سهل.
- **السلبيات**: لا يربط بالقصة، يفقد ritual feel.

#### 🅲️ بديل C: Multi-Currency (مثل Dead Cells)
- عملة للأشياء الدائمة (Skill Tree).
- عملة للأشياء المؤقتة (consumables).
- **الإيجابيات**: عمق اقتصادي.
- **السلبيات**: معقد لمطور indie + يشتت اللاعب.

### 1.4 التوصية + الهندسة التقنية

**التوصية**: 🅰️ (Sanguis System).

#### Data Structure

```gdscript
# sanguis_manager.gd (Autoload / Singleton)
extends Node

signal sanguis_changed(new_amount: int)
signal skill_unlocked(skill_id: String)

var sanguis: int = 0 :
    set(value):
        sanguis = value
        sanguis_changed.emit(sanguis)

const SKILL_DATABASE := {
    "wall_jump": {
        "name": "Wall Jump",
        "description": "Jump off walls to reach new areas.",
        "cost": 50,
        "prerequisites": [],
        "type": "movement"
    },
    "double_jump": {
        "name": "Double Jump",
        "description": "Perform a second jump in mid-air.",
        "cost": 80,
        "prerequisites": ["wall_jump"],
        "type": "movement"
    },
    "roll_dash": {
        "name": "Roll Dash",
        "description": "Dash under enemies, ignoring collision.",
        "cost": 60,
        "prerequisites": [],
        "type": "dash"
    },
    # ... كل الـ skills
}

var unlocked_skills: PackedStringArray = []

func grant_sanguis(amount: int, source: String) -> void:
    sanguis += amount
    print("[Sanguis] +%d from %s (total: %d)" % [amount, source, sanguis])

func can_unlock(skill_id: String) -> bool:
    if not SKILL_DATABASE.has(skill_id):
        return false
    var skill = SKILL_DATABASE[skill_id]
    if skill_id in unlocked_skills:
        return false
    if sanguis < skill.cost:
        return false
    for prereq in skill.prerequisites:
        if prereq not in unlocked_skills:
            return false
    return true

func unlock_skill(skill_id: String) -> bool:
    if not can_unlock(skill_id):
        return false
    sanguis -= SKILL_DATABASE[skill_id].cost
    unlocked_skills.append(skill_id)
    skill_unlocked.emit(skill_id)
    return true
```

#### VeinAltar Interaction

```gdscript
# vein_altar.gd
extends Area2D

@onready var prompt: Label = $PromptLabel
@onready var menu: Control = $SkillTreeMenu

func _ready() -> void:
    body_entered.connect(_on_body_entered)
    body_exited.connect(_on_body_exited)

func _on_body_entered(body: Node) -> void:
    if body.is_in_group("player"):
        prompt.text = "Press [Interact] to drink at the Vein Altar"
        prompt.visible = true

func _on_body_exited(body: Node) -> void:
    if body.is_in_group("player"):
        prompt.visible = false
        menu.close()

func _unhandled_input(event: InputEvent) -> void:
    if event.is_action_pressed("interact") and _player_in_range():
        get_tree().paused = true
        menu.open(SanguisManager.sanguis, SanguisManager.unlocked_skills)
```

#### Skill Tree Menu UI

```gdscript
# skill_tree_menu.gd
extends Control

@onready var tree_graph: GraphEdit = $TreeGraph
@onready var sanguis_label: Label = $SanguisLabel

func open(current_sanguis: int, unlocked: PackedStringArray) -> void:
    sanguis_label.text = "Sanguis: %d" % current_sanguis
    _render_tree(unlocked)
    visible = true

func _render_tree(unlocked: PackedStringArray) -> void:
    for skill_id in SanguisManager.SKILL_DATABASE:
        var node := SkillNode.new()
        node.skill_id = skill_id
        node.unlocked = skill_id in unlocked
        node.can_unlock = SanguisManager.can_unlock(skill_id)
        node.unlock_requested.connect(_on_unlock_requested)
        tree_graph.add_child(node)

func _on_unlock_requested(skill_id: String) -> void:
    if SanguisManager.unlock_skill(skill_id):
        sanguis_label.text = "Sanguis: %d" % SanguisManager.sanguis
        _render_tree(SanguisManager.unlocked_skills)
```

#### Sanguis Earning Hooks

```gdscript
# enemy_base.gd
func _die() -> void:
    SanguisManager.grant_sanguis(sanguis_reward, "enemy:" + enemy_name)
    queue_free()

# boss.gd
func _die() -> void:
    SanguisManager.grant_sanguis(sanguis_reward, "boss:" + boss_name)
    # cutscene trigger
    queue_free()

# checkpoint.gd
func _on_player_rest() -> void:
    SanguisManager.grant_sanguis(10, "rest_checkpoint")
    PlayerManager.full_heal()

# secret_area_trigger.gd
func _on_entered_first_time() -> void:
    SanguisManager.grant_sanguis(25, "secret_discovery")
```

### 1.5 Plot Integration
- **Vein Altars**: بقايا معابد قديمة. كل altar له اسم (مثل "Altar of the First Drink"). تفتح lore snippets.
- **Sanguis**: ليس مجرد نقطة — هو "الدم القديم" الذي يجري في عروق الـ Veinbreakers. كل skill = استعادة لقدرة قديمة.
- **الـ Bosses**: كل boss هو "Veinbreaker سابق فشل" — قتله يطلق دمه فيستعيد اللاعب جزءاً من قوته.

---

## Pitfall #2: كثرة الأزرار (دمج Attack + Parry؟)

> [!warning] الكثير من الأزرار
> - خيار الدمج بين زر Attack و Parry؟

### 2.1 إعادة صياغة المشكلة
المشكلة ليست "كيف ندمج زرّين" — هي **كيف نقلل الـ cognitive load على الـ thumb اليمين في شاشة 6 إنش**. عدد الأزرار المطلوبة نظرياً:

| الزر | الوظيفة | ضروري؟ |
|------|---------|--------|
| Jump | القفز | ✅ نعم |
| Dash | الـ dash بأنواعه | ✅ نعم |
| Melee Attack | الضرب | ✅ نعم |
| Handgun Fire | الإطلاق | ✅ نعم |
| Parry | التصدي | ✅ نعم |
| Heal | العلاج | ⚠️ ممكن |
| Interact | الحوار/التفاعل | ⚠️ ممكن |
| Skill Tree | فتح القائمة | ❌ يمكن من pause |

= **6-7 أزرار** على الـ bottom-right. **Dead Cells Mobile** يكتفي بـ 4. **Hollow Knight Mobile** يكتفي بـ 3.

### 2.2 تحليل المشكلة
- **المشكلة الجذرية**: كل زر ميكانيك مستقل، لكن بعضها **context-dependent** (Parry فقط عند هجوم عدو، Heal فقط عند low HP).
- **لماذا يكسر اللعبة**: 
  - false-positive taps (لاعب يضغط Parry بدون هجوم → يخسر فرصة attack).
  - thumb fatigue بعد 30 دقيقة.
  - **uninstall** بسبب "controls are clunky".
- **في ألعاب مشابهة**:
  - **Bloodborne**: زر الـ gun هو نفسه زر الـ parry (tap = shoot، tap في parry window = parry).
  - **Sekiro**: زر L1 = Block/Parry بناءً على timing.
  - **Nine Sols**: زر الـ Parry منفصل لكنه على الكتف.

### 2.3 البدائل الثلاثة

#### 🅰️ بديل A: Context-Sensitive Multi-Function Button (الموصى به)
**الأزرار النهائية = 5**:
1. **Jump** (مستقل، لأنه الأكثر استخداماً).
2. **Dash** (مستقل، لأنه ينقذ الحياة).
3. **Attack/Parry** (مدمج، context-aware):
   - Tap = Slash combo.
   - Hold 0.5s = Charged.
   - Tap في **enemy attack parry window** (±0.15s قبل الـ hit) = Parry.
4. **Handgun** (مستقل، لأنه secondary).
5. **Interact/Heal** (مدمج):
   - قرب Vein Altar/chest = Interact.
   - في القتال + عند وجود consumable = Heal.

**الإيجابيات**:
- 5 أزرار فقط = معيار Dead Cells.
- كل وظيفة محفوظة.
- الـ Parry يصبح "skill expressive" (timing-based).

**السلبيات**:
- false parry attempts لو اللاعب يضغط Attack في توقيت خاطئ (هذا **feature** وليس bug — يحفّز التعلم).
- يستغرق تعوّد 1-2 ساعة.

#### 🅱️ بديل B: Attack + Parry كـ tap vs hold
- Tap = Attack.
- Hold 0.3s = Parry (يدخل parry stance).
- **الإيجابيات**: أوضح.
- **السلبيات**: Hold في قتال سريع = انتحار (اللاعب لا يقدر أن يحمل 0.3s أمام عدو).

#### 🅲️ بديل C: Parry كـ gesture (swipe)
- Swipe على عدو مهاجم = Parry.
- **الإيجابيات**: native mobile.
- **السلبيات**: يتعارض مع joystick، صعب في القتال السريع.

### 2.4 التوصية + الهندسة التقنية

**التوصية**: 🅰️ (Context-Sensitive Multi-Function).

#### State Machine للزر الموحّد

```gdscript
# attack_parry_button.gd
extends TouchScreenButton

signal slash_pressed()
signal charged_released(hold_duration: float)
signal parry_attempted()
signal backstab_attempted()

enum ButtonState { IDLE, TAPPING, CHARGING, BACKSTAB_HOLD, PARRY_WINDOW }
var state: ButtonState = ButtonState.IDLE

var tap_duration: float = 0.0
var press_start_time: float = 0.0
const TAP_THRESHOLD := 0.18      # أقل من هذا = tap
const CHARGE_THRESHOLD := 0.6    # أكثر من هذا = charged
const BACKSTAB_THRESHOLD := 0.5  # أكثر من هذا + عدو غير مدرك = backstab
const PARRY_WINDOW := 0.15       # ثانية قبل الـ hit land

func _on_pressed() -> void:
    press_start_time = Time.get_ticks_msec() / 1000.0
    state = ButtonState.TAPPING
    
    # فحص فوري: هل نحن في parry window؟
    if ParryManager.is_in_parry_window():
        parry_attempted.emit()
        state = ButtonState.PARRY_WINDOW
        return
    
    # فحص: هل نحن في backstab range؟
    if BackstabManager.is_in_backstab_range():
        state = ButtonState.BACKSTAB_HOLD
        return
    
    state = ButtonState.CHARGING

func _on_released() -> void:
    var hold_duration := (Time.get_ticks_msec() / 1000.0) - press_start_time
    
    match state:
        ButtonState.PARRY_WINDOW:
            pass  # تم معالجته في _on_pressed
        ButtonState.BACKSTAB_HOLD:
            if hold_duration >= BACKSTAB_THRESHOLD:
                backstab_attempted.emit()
            else:
                slash_pressed.emit()  # tap على عدو غير مدرك = slash عادي
        ButtonState.CHARGING:
            if hold_duration >= CHARGE_THRESHOLD and SkillManager.has_skill("charged_slash"):
                charged_released.emit(hold_duration)
            else:
                slash_pressed.emit()
        ButtonState.TAPPING:
            slash_pressed.emit()
    
    state = ButtonState.IDLE
```

#### Parry Window Manager

```gdscript
# parry_manager.gd (Autoload)
extends Node

# كل عدو يهاجم يسجل parry window هنا
var active_parry_windows: Array[Dictionary] = []

func register_parry_window(enemy: Node, start_time: float, end_time: float) -> void:
    active_parry_windows.append({
        "enemy": enemy,
        "start": start_time,
        "end": end_time
    })

func is_in_parry_window() -> bool:
    var now := Time.get_ticks_msec() / 1000.0
    for window in active_parry_windows:
        if now >= window.start and now <= window.end:
            return true
    return false

func consume_parry_window() -> Node:
    """يسترجع العدو الذي تم parry له (لتنفيذ counter)"""
    var now := Time.get_ticks_msec() / 1000.0
    for i in range(active_parry_windows.size()):
        var window = active_parry_windows[i]
        if now >= window.start and now <= window.end:
            var enemy = window.enemy
            active_parry_windows.remove_at(i)
            return enemy
    return null
```

#### Enemy Telegraph Hook

```gdscript
# enemy_base.gd
func _start_attack_telegraph() -> void:
    # مدة telegraph = 0.4s (مثال)
    # parry window = آخر 0.15s قبل hit land
    var telegraph_duration := 0.4
    var parry_window_start := Time.get_ticks_msec() / 1000.0 + telegraph_duration - 0.15
    var parry_window_end := parry_window_start + 0.15
    
    ParryManager.register_parry_window(self, parry_window_start, parry_window_end)
    
    # بصرياً: العدو يومض بالأحمر
    _flash_red()
    
    await get_tree().create_timer(telegraph_duration).timeout
    _perform_attack()  # لو لم يُـ parry
```

### 2.5 تكامل القصة
- **Veinbreaker Reflexes**: القصة تبرر الـ context-sensitive بـ "الـ Veinbreakers تتدفّق في عروقهم دم قديم يفتح لهم 'eye moments' يرون فيها هجمات العدو ببطء". الـ parry window = هذه الـ eye moments.

---

## Pitfall #3: هل يوجد نظام Stamina؟

> [!Question] هل يوجد نظام Stamina?
> - ماهي فائدته ان قررت وضعه؟

### 3.1 إعادة صياغة المشكلة
ليست "هل أضع Stamina" — هي **"ما الذي تحلّه الـ Stamina ولماذا تحتاج حلاً؟"**. 

في تصميمك الحالي، الـ Stamina مذكورة 4 مرات في جدول الـ Dash لكنها لا توجد فعلياً. هذا يعني أنك **تشعر بمشكلة** لكن لم تُسمِّها بعد.

#### ما المشكلة التي قد تحلّها الـ Stamina؟
1. **Spam Prevention**: منع اللاعب من dash indefinitely (يكسر combat tension).
2. **Resource Management**: إضافة layer tactic (متى dash؟ متى أحفظ؟).
3. **Pacing Control**: إجبار اللاعب على slower moments بين الـ bursts.
4. **Skill Expression**: لاعبون ماهرون يديرون stamina أفضل = تمايز.

#### ما المشكلات التي تخلقها الـ Stamina؟
1. **Frustration**: نفاد stamina في لحظة حرجة = موت غير مستحق.
2. **Cognitive Load**: مراقبة bar إضافي = تشتيت.
3. **Mobile UX**: bar على الشاشة = مساحة بصرية مأخوذة.
4. **Balancing Nightmare**: كل ميكانيك يحتاج tuning مع stamina.

### 3.2 تحليل المشكلة
- **في ألعاب مشابهة**:
  - **Hollow Knight**: لا stamina. بدلاً منها، cooldown على spells فقط.
  - **Dark Souls**: stamina شاملة (attack + dash + block). عميقة لكنها frustrating للمبتدئين.
  - **Sekiro**: لا stamina (بـ posture بدلاً منها، وهو different concept).
  - **Dead Cells**: لا stamina، cooldown على skills فقط.
  - **Hades**: لا stamina، dash له cooldown قصير (1s).
  - **Ori & Will of the Wisps**: لا stamina (energy للأشياء السحرية فقط).
- **الاستنتاج**: في الـ indie 2D platformers، الـ stamina نادرة. الـ cooldowns أفضل على mobile.

### 3.3 البدائل الثلاثة

#### 🅰️ بديل A: لا Stamina، Cooldowns فقط (الموصى به للمشروع الحالي)
- كل dash له cooldown قصير مستقل:
  - Normal Dash: 0.5s cooldown.
  - Roll Dash: 1.0s cooldown.
  - Wall Dash: 0.8s cooldown.
  - Dodge Dash: 1.5s cooldown (لأنه أقوى).
- **الإيجابيات**: 
  - بدون bar إضافي = شاشة أنظف.
  - لا frustration من نفاد.
  - balancing أبسط (tune per ability).
- **السلبيات**: 
  - اللاعب يقدر يـ spam Normal Dash.
  - الحل: 0.5s cooldown كافٍ لمنع الـ infinite spam.

#### 🅱️ بديل B: Stamina بسيطة لـ Dash فقط
- stamina bar صغير (50pt) بجانب الـ health.
- كل dash = -10 stamina.
- regeneration 20/s.
- **الإيجابيات**: tactical layer.
- **السلبيات**: bar على الشاشة + balancing معقد.

#### 🅲️ بديل C: Sekiro-style Posture (للـ Player)
- لا stamina، لكن **posture meter**: يتعب من dash + parry + attack.
- يمتلئ → stagger (يفقد السيطرة 1s).
- يفرّغ بـ not doing anything.
- **الإيجابيات**: عميق جداً، تكامل مع combat.
- **السلبيات**: معقد جداً للمشروع indie mobile.

### 3.4 التوصية + الهندسة التقنية

**التوصية**: 🅰️ (لا Stamina، Cooldowns فقط).

#### Data Structure

```gdscript
# dash_cooldown_manager.gd
extends Node

# كل نوع dash له cooldown منفصل
const DASH_COOLDOWNS := {
    "normal": 0.5,
    "roll": 1.0,
    "wall": 0.8,
    "dodge": 1.5,
}

var cooldowns: Dictionary = {
    "normal": 0.0,
    "roll": 0.0,
    "wall": 0.0,
    "dodge": 0.0,
}

func _process(delta: float) -> void:
    for key in cooldowns.keys():
        if cooldowns[key] > 0.0:
            cooldowns[key] = max(0.0, cooldowns[key] - delta)

func can_dash(dash_type: String) -> bool:
    return cooldowns.get(dash_type, 0.0) <= 0.0

func trigger_cooldown(dash_type: String) -> void:
    cooldowns[dash_type] = DASH_COOLDOWNS.get(dash_type, 0.5)

func get_cooldown_progress(dash_type: String) -> float:
    """0.0 = جاهز، 1.0 =刚 بدأ"""
    var cd := DASH_COOLDOWNS.get(dash_type, 0.5)
    if cd <= 0.0:
        return 0.0
    return cooldowns.get(dash_type, 0.0) / cd
```

#### HUD Integration (Radial Cooldown Indicator)

```gdscript
# dash_button.gd
extends TouchScreenButton

@onready var cooldown_overlay: Sprite2D = $CooldownOverlay

func _process(delta: float) -> void:
    var progress := DashCooldownManager.get_cooldown_progress("normal")
    cooldown_overlay.visible = progress > 0.0
    if progress > 0.0:
        # radial wipe effect (مثل ألعاب MOBA)
        var shader_mat := cooldown_overlay.material as ShaderMaterial
        shader_mat.set_shader_parameter("progress", progress)
```

#### Shader (radial_cooldown.gdshader)

```glsl
shader_type canvas_item;

uniform float progress : hint_range(0.0, 1.0) = 0.0;
uniform vec4 color : source_color = vec4(0.0, 0.0, 0.0, 0.6);

void fragment() {
    vec4 tex = texture(TEXTURE, UV);
    float angle = atan(UV.y - 0.5, UV.x - 0.5);
    angle = (angle + 3.14159) / (2.0 * 3.14159);  // 0..1
    if (angle < progress) {
        COLOR = mix(tex, color, color.a);
    } else {
        COLOR = tex;
    }
}
```

### 3.5 تكامل القصة (اختياري)
- لو اخترت 🅰️، لا تحتاج تبرير شعري. الـ cooldowns معطاة بشكل طبيعي.
- لو اخترت 🅱️ لاحقاً، الـ Stamina يمكن تسميتها `Adrenaline` وتبرَّر بـ "الـ Veinbreaker يستهلك دمه الخاص لإجراء حركات خارقة".

---

## Pitfall #4: لا يوجد نظام Economy / Currency

> [!Missing] There is still no Economy / Currency system.
> - How it works?
> - From where the Player will get it?
> - Where can he spend it?
> - What can he buy with it?

### 4.1 إعادة صياغة المشكلة
ليست "كيف أضيف currency" — هي **"ما الذي لا يفعله Sanguis ويحتاج currency منفصلة؟"**.

#### لماذا قد تحتاج currency منفصلة عن Sanguis؟
- **Sanguis** = نقاط الـ Skill Tree (دائمة، تستثمر مرة واحدة).
- **Currency** = للأشياء القابلة للاستهلاك (consumables) أو الترقيات المؤقتة.
- بدون فصل، اللاعب يضطر للاختيار بين skill دائم و heal فوري = قرارات غير ممتعة.

### 4.2 تحليل المشكلة
- **في ألعاب مشابهة**:
  - **Hollow Knight**: Geo لكل شيء (charms + upgrades + consumables). لا فصل. لكن الـ charms هي التي تحدد الـ build.
  - **Dead Cells**: Gold للمؤقت (in-run)، Cells للدائم (meta). فصل واضح.
  - **Hades**: Gems للـ meta، Gold للـ run. فصل.
  - **Salt and Sanctuary**: Salt = XP + currency معاً. موحّد لكنه problematic (تفقد الاثنين عند الموت).
- **الخلاصة**: الفصل بين "نقاط الترقية الدائمة" و "نقاط الاستهلاك" هو المعيار الذهبي.

### 4.3 البدائل الثلاثة

#### 🅰️ بديل A: Sanguis (permanent) + Cinders (consumable) (الموصى به)
- **Sanguis**: للـ Skill Tree (كما في Pitfall #1).
- **Cinders** (رماد): للـ consumables والترقيات المؤقتة.
  - **المصادر**:
    - الأعداء (1-5 cinders لكل عدو).
    - Breakable items (5-15 cinders).
    - Selling items (variable).
    - Side quests (50-200 cinders).
  - **أين تنفق**:
    - **Wandering Merchants** (تجار متنقلون في الخريطة).
    - **Blacksmiths** (لترقية الأسلحة).
  - **ماذا تشتري**:
    - Healing vials (consumable).
    - Ammo for Handgun.
    - Throwing knives (consumable ranged).
    - Weapon upgrades (+damage %).
    - Charms/trinkets (passive bonuses).

#### 🅱️ بديل B: عملة واحدة لكل شيء
- Sanguis لكل شيء (skill + consumables).
- **الإيجابيات**: بساطة.
- **السلبيات**: قرارات غير ممتعة (skill vs heal).

#### 🅲️ بديل C: ثلاث عملات (Meta + Run + Consumable)
- معقد جداً لمطور indie.

### 4.4 التوصية + الهندسة التقنية

**التوصية**: 🅰️ (Sanguis + Cinders).

#### Cinders Manager

```gdscript
# cinders_manager.gd (Autoload)
extends Node

signal cinders_changed(new_amount: int)

var cinders: int = 0 :
    set(value):
        cinders = value
        cinders_changed.emit(cinders)

func grant(amount: int, source: String) -> void:
    cinders += amount
    print("[Cinders] +%d from %s (total: %d)" % [amount, source, cinders])

func spend(amount: int) -> bool:
    if cinders < amount:
        return false
    cinders -= amount
    return true
```

#### Merchant System

```gdscript
# merchant.gd
extends Node2D

@export var merchant_inventory: Array[Dictionary] = [
    {"item_id": "healing_vial", "name": "Healing Vial", "price": 30, "stock": -1},  # -1 = infinite
    {"item_id": "handgun_ammo", "name": "Handgun Ammo x10", "price": 20, "stock": -1},
    {"item_id": "throwing_knife", "name": "Throwing Knife x5", "price": 25, "stock": 10},
    {"item_id": "blade_sharpening", "name": "Blade Sharpening (+10% melee damage)", "price": 100, "stock": 1, "permanent": true},
]

@onready var interaction_area: Area2D = $InteractionArea
@onready var shop_ui: Control = $ShopUI

func _ready() -> void:
    interaction_area.body_entered.connect(_on_body_entered)
    interaction_area.body_exited.connect(_on_body_exited)

func _on_body_entered(body: Node) -> void:
    if body.is_in_group("player"):
        $PromptLabel.visible = true

func _on_body_exited(body: Node) -> void:
    if body.is_in_group("player"):
        $PromptLabel.visible = false
        shop_ui.close()

func _unhandled_input(event: InputEvent) -> void:
    if event.is_action_pressed("interact") and _player_in_range():
        shop_ui.open(merchant_inventory, CindersManager.cinders)
```

#### Shop UI Logic

```gdscript
# shop_ui.gd
extends Control

signal item_purchased(item_id: String)

@onready var item_list: VBoxContainer = $ScrollContainer/ItemList
@onready var cinders_label: Label = $CindersLabel

func open(inventory: Array, current_cinders: int) -> void:
    cinders_label.text = "Cinders: %d" % current_cinders
    _populate_items(inventory)
    visible = true
    get_tree().paused = true

func _populate_items(inventory: Array) -> void:
    for child in item_list.get_children():
        child.queue_free()
    
    for item in inventory:
        var button := Button.new()
        var stock_text := "∞" if item.stock == -1 else str(item.stock)
        button.text = "%s - %d cinders (x%s)" % [item.name, item.price, stock_text]
        button.disabled = CindersManager.cinders < item.price or item.stock == 0
        button.pressed.connect(func(): _purchase(item))
        item_list.add_child(button)

func _purchase(item: Dictionary) -> void:
    if not CindersManager.spend(item.price):
        return
    if item.stock > 0:
        item.stock -= 1
    InventoryManager.add_item(item.item_id, 1)
    cinders_label.text = "Cinders: %d" % CindersManager.cinders
    _populate_items(get_meta("inventory"))  # re-render
```

### 4.5 ماذا تشتري بالـ Cinders؟ (Table)

| # | العنصر | السعر | التأثير | النوع |
|---|--------|------|---------|-------|
| 1 | Healing Vial | 30 cinders | يعالج 40% HP | Consumable |
| 2 | Handgun Ammo ×10 | 20 cinders | ذخيرة مسدس | Consumable |
| 3 | Throwing Knife ×5 | 25 cinders | ضرر بعيد سريع | Consumable |
| 4 | Smoke Bomb ×3 | 50 cinders | يهرب من المعركة | Consumable |
| 5 | Blade Sharpening | 100 cinders | +10% melee damage | Permanent upgrade |
| 6 | Gunpowder Mod | 150 cinders | +20% handgun damage | Permanent upgrade |
| 7 | Stamina Vial | 60 cinders | (لو وُجد stamina) يعالج 50% stamina | Consumable |
| 8 | Vein Map | 200 cinders | يكشف منطقة على الخريطة | Permanent |
| 9 | Lore Scroll | 40 cinders | يفتح lore entry | Permanent |
| 10 | Charm Slot | 300 cinders | +1 slot للـ charms | Permanent |

### 4.6 تكامل القصة
- **Cinders** = رماد الموتى. في عالم Veinbreaker، الموتى يحترقون ورمادهم يُتداول كعملة (لأنه يحتوي بقايا دم قديم).
- **Merchants** = "Ash Bearers" — طائفة تجارية تحترم الرماد.
- هذا يربط الـ economy بـ lore اللعبة بشكل عميق.

---

## Pitfall #5: لا يوجد نظام Healing أو زر

> [!Missing] There is no healing system or button

### 5.1 إعادة صياغة المشكلة
ليست "كيف أضع زر heal" — هي **"كيف أشفى اللاعب بطريقة تحافظ على إحساس Soulslike Challenged دون إحباط mobile"**.

#### الأسئلة الجوهرية:
1. هل الـ healing حر (في أي وقت) أم مقيد (في checkpoint)؟
2. هل هو consumable أم rechargeable؟
3. ما cooldown العلاج؟
4. هل يفتح animation تفتح نافذة ضرب؟
5. كم يعالج؟

### 5.2 تحليل المشكلة
- **في ألعاب مشابهة**:
  - **Hollow Knight**: healing حر لكن يأخذ 1.5s + يوقف اللاعب. tactical decision.
  - **Dark Souls**: Estus Flask — charges محدودة، refill عند rest.
  - **Sekiro**: healing potions محدودة (9 max)، refill عند rest.
  - **Dead Cells**: healing من вып drops عشوائية + bottles.
  - **Hades**: healing نادر، من drops + shops.
  - **Blasphemous**: healing من flasks محدودة.
- **الخلاصة**: في Soulslike، الـ healing **مورد محدود** يجب إدارته. هذا هو "Challenged".

### 5.3 البدائل الثلاثة

#### 🅰️ بديل A: Vein Flasks (Estus-style) (الموصى به)
- اللاعب يبدأ بـ **3 Vein Flasks**.
- كل flask يعالج **40% HP**.
- refill عند كل Vein Altar (نفس مكان الـ Skill Tree upgrade — يجمع الـ rituals).
- يمكن شراء flasks إضافية من merchants (max 5).
- **زر**: Heal button منفصل (لكنه ضمن الـ context-sensitive button #5 في Pitfall #2).
- **animation**: 0.6s drink animation، اللاعب يمكن أن يُقاطع (يُضرب أثناءه → flask تُستلك بدون healing).
- **الإيجابيات**:
  - يحافظ على إحساس Soulslike.
  - tactical (متى تشرب؟).
  - يربط بـ Vein Altars (lore + mechanic).
- **السلبيات**:
  - لاعب قد يموت بـ 3 flasks فارغة → إحباط.
  - الحل: refill عند كل checkpoint يخفف هذا.

#### 🅱️ بديل B: Regeneration (Hollow Knight style بدون items)
- HP يتجدد ببطء خارج القتال (5 HP/s بعد 3s بدون قتال).
- **الإيجابيات**: لا إدارة موارد.
- **السلبيات**: يقتل إحساس Soulslike.

#### 🅲️ بديل C: Healing بـ Sanguis
- اللاعب يدفع Sanguis ليشفى.
- **الإيجابيات**: يربط الأنظمة.
- **السلبيات**: يخلق conflict (هل أحفظ لـ skill أم أشفى؟).

### 5.4 التوصية + الهندسة التقنية

**التوصية**: 🅰️ (Vein Flasks).

#### Player Health Manager

```gdscript
# player_health.gd
extends Node

signal health_changed(current: float, maximum: float)
signal flask_changed(remaining: int, maximum: int)
signal player_died()
signal healing_started()
signal healing_interrupted()

@export var max_health: float = 100.0
@export var max_flasks: int = 3
@export var flask_heal_amount: float = 40.0  # 40% of max
@export var heal_duration: float = 0.6

var current_health: float = max_health
var current_flasks: int = max_flasks
var is_healing: bool = false

func take_damage(amount: float) -> void:
    if is_healing:
        healing_interrupted.emit()
        is_healing = false
    current_health = max(0.0, current_health - amount)
    health_changed.emit(current_health, max_health)
    if current_health <= 0.0:
        player_died.emit()

func try_heal() -> bool:
    if current_flasks <= 0 or is_healing or current_health >= max_health:
        return false
    current_flasks -= 1
    flask_changed.emit(current_flasks, max_flasks)
    is_healing = true
    healing_started.emit()
    
    # animation + timer
    var timer := get_tree().create_timer(heal_duration)
    timer.timeout.connect(func():
        if is_healing:  # لم يُقاطع
            current_health = min(max_health, current_health + (max_health * flask_heal_amount / 100.0))
            health_changed.emit(current_health, max_health)
            is_healing = false
    )
    return true

func refill_flasks() -> void:
    current_flasks = max_flasks
    flask_changed.emit(current_flasks, max_flasks)

func increase_max_flasks(amount: int = 1) -> void:
    max_flasks = min(5, max_flasks + amount)  # hard cap at 5
    current_flasks = max_flasks
    flask_changed.emit(current_flasks, max_flasks)
```

#### Vein Altar Refill Hook

```gdscript
# vein_altar.gd (مكمّل للـ code في Pitfall #1)
func _on_player_rest() -> void:
    PlayerHealth.refill_flasks()
    PlayerHealth.current_health = PlayerHealth.max_health
    SanguisManager.grant_sanguis(10, "rest_altar")
    SaveManager.save_game()
```

#### HUD: Flask Display

```gdscript
# flask_hud.gd
extends Control

@onready var flask_icons: HBoxContainer = $FlaskIcons

func _ready() -> void:
    PlayerHealth.flask_changed.connect(_update_display)
    _update_display(PlayerHealth.current_flasks, PlayerHealth.max_flasks)

func _update_display(remaining: int, maximum: int) -> void:
    for i in range(flask_icons.get_child_count()):
        var icon := flask_icons.get_child(i) as TextureRect
        icon.visible = i < maximum
        icon.modulate = Color.WHITE if i < remaining else Color(0.3, 0.3, 0.3, 0.5)
```

### 5.5 تكامل القصة
- **Vein Flasks** = قوارير زجاجية تحتوي على دم مخفّف من آلهة قديمة. شربها يعيد بناء الجروح بسرعة.
- **الـ Altar refill**: عند الـ Vein Altar، اللاعب "يعيد ملء" قواريره من الدم المتراكم.
- **الـ animation**: يشرب ببطء ويرى رؤى قصيرة (lores snippets).

---

## خلاصة الحلول المقترحة لـ Pitfalls

| Pitfall | التوصية | الـ Plot |
|---------|---------|---------|
| #1 Skill Tree upgrade | Sanguis System + Vein Altars | دم الآلهة القديمة |
| #2 كثرة الأزرار | Context-Sensitive Multi-Function Button (5 أزرار) | Veinbreaker Reflexes |
| #3 Stamina | لا Stamina، Cooldowns فقط | لا يحتاج تبرير |
| #4 Economy | Sanguis (permanent) + Cinders (consumable) | الرماد كعملة |
| #5 Healing | Vein Flasks (Estus-style) | قوارير الدم المخفّف |

**هذه الحلول الخمسة تشكل طبقة الميكانيك الأساسية للعبة. بدونها لا يمكن بدء prototype.**

---

**التالي**: راجع `03-Empty-Systems-Foundations.md` لتأسيس الأنظمة الفارغة (Player، Level Design، Camera، Health، Map، Skill Tree، Damage، Handguns، Parry، AI، Input Map).
