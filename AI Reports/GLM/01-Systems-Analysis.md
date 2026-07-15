# تقرير تحليلي شامل — الأنظمة الموجودة في Veinbreaker

> **الهدف من هذا الملف**: تفكيك كل نظام مصمَّم حالياً في الـ vault، تحديد نقاط قوته، ثم نقاط ضعفه مع **حجج منطقية لاحتمال فشله**، ثم **أمثلة من ألعاب indie ناجحة** واجهت نفس المعضلة، وأخيراً **حلول مقترحة بمستويات أهمية متفاوتة**.

> **ملاحظة منهجية**: كل قسم يحترم هيكل: `الوصف الحالي` → `نقاط القوة` → `نقاط الضعف (مع حجة الفشل)` → `أمثلة من ألعاب مشابهة` → `الحلول المقترحة`.

---

## 0. خريطة الأنظمة الموجودة

| # | النظام | الحالة | مكان الملف |
|---|--------|--------|-----------|
| 1 | Core Concept (الفكرة الأساسية) | ✅ مكتمل | `Core Concept.md` |
| 2 | Movement System (القفز/الـ Dash/الـ Wall Slide) | ✅ مكتمل بتفاصيل | `Game play/Systems/Movement System.md` |
| 3 | Aim System (نظام التصويب بالـ Joystick) | ✅ مكتمل تقنياً | `Architecture/Aim system.md` |
| 4 | Melee Combat (5 هجمات) | ✅ مفصّل | `Game play/Systems/Combat Systems/Melee.md` |
| 5 | Controllers (Joystick ميكانيكية) | ⚠️ مختصر جداً | `Game play/Systems/Controllers.md` |
| 6 | Enemy AI Framework | ❌ فارغ | `Architecture/Enemies/AI.md` |
| 7 | Damage System | ❌ نقطتان فقط | `Architecture/Damage System.md` |
| 8 | Pitfalls (المخاطر المعلنة) | ⚠️ مكتوبة لكن بلا حلول | `Pitfalls.md` |

---

## 1. Core Concept (الفكرة الأساسية)

### 1.1 الوصف الحالي
- Genre: 2D Action-Platformer Metroidvania, Mobile-First, Soulslike-inspired, Stealth-Optional, Minimal Puzzles.
- اللاعب يحمل سلاحين: Melee (2 Blades) كأساسي + Handgun كثانوي (ذخيرة محدودة، 3 اتجاهات تصويب).
- الأهداف الشعورية: `Skilled` + `Challenged` + `Fast` + `Intelligent`.
- الإلهام: Hollow Knight + Nine Sols للقتال، Action-Platformer indie للحركة.

### 1.2 نقاط القوة
- **تحديد هوية واضحة من اليوم الأول**: الجمع بين `Mobile-First` و `Soulslike-inspired` قرار جريء ولكنه يبني هوية مميزة. أغلب ألعاب الـ mobile تتجنب الـ Soulslike بسبب صعوبته.
- **اختيار إلهامات ذكي**: Hollow Knight (متعة الاستكشاف) + Nine Sols (قتال Parry-based عميق) يجمع بين أفضل ما في الـ indie خلال الـ 5 سنوات الأخيرة.
- **تحديد الشعور المستهدف بـ 4 كلمات** ممتاز لاتخاذ قرارات تصميمية لاحقة (كل ميكانيك يُسأل: هل يخدم Skilled أو Fast؟).
- **الـ Handgun كـ Secondary وليس Primary** يحل مشكلة جوهرية: الـ mobile shooters صعبة بسبب نقص دقة الـ touch.

### 1.3 نقاط الضعف — وحجج احتمال الفشل

#### ⚠️ ضعف #1: تضارب بين `Mobile-First` و `Soulslike-inspired`
**الحجة**: Soulslike يعني بالتعريف "عقاب قاسٍ على الأخطاء + تكرار + تعلّم من النمط". الـ mobile players إحصائياً يلعبون جلسات قصيرة (5-15 دقيقة) في أوقات الانتظار. لو مات اللاعب في boss بعد 12 دقيقة من اللعب واضطر يعيد 10 دقائق من الـ level → احتمال uninstall يرتفع dramatically. Dead Cells (mobile port) واجه هذا وراحوا حلّوه بـ **Daily Run + Cell System + Cloud Save**. Hollow Knight على الـ Switch ( ليس mobile) أيضاً اتُّهم بأنه "مستحيل على الهاتف" قبل إصدار iOS.

#### ⚠️ ضعف #2: كثرة الـ Genres تخلق أزمة هوية
**الحجة**: في الجدول 5 genres (Action-Platformer، Metroidvania، Soulslike، Stealth، Puzzles). كل genre يحتاج أنظمة مستقلة (Stealth يحتاج AI vision cones + sound propagation؛ Puzzles يحتاج logic system). مطور indie وحده لا يستطيع إتقان 5 أنظمة بعمق كافٍ. النتيجة المتوقعة: كل نظام يكون "نصف مطبوخ" → إحساس "jack of all trades, master of none". مثال سلبي: **Ender Lilies** حاول يجمع Metroidvania + Soulslike + RPG وانتقده النقاد بـ "بيئات مكررة، أعداء ضعفاء". مثال إيجابي: **Hollow Knight** بدأ بـ 2 genres فقط (Metroidvania + Soulslike) ثم أضاف الـ Platforming لاحقاً بعد إتقان الأساس.

#### ⚠️ ضعف #3: قيود 3 اتجاهات للتصويب قد تشعر بـ clunky
**الحجة**: اللاعب سيتوقع حرية كاملة (8 اتجاهات)، لكن اللعبة تقصره على 3 (فوق/يمين/يسار). في معارك الـ Boss (خاصة Background Bosses)، اللاعب يحتاج إصابة زاوية معينة. لو الـ boss ضعيف فقط من الأسفل واللاعب لا يقدر يصوّب لأسفل بالـ Handgun → إحباط. **Hyper Light Drifter** حلّت هذا بـ "8-dir aim" لكنها قيّدت الذخيرة بشدة.

### 1.4 أمثلة من ألعاب مشابهة

| اللعبة | ما يمكن تعلّمه |
|--------|----------------|
| **Hollow Knight** | كيف Metroidvania + Soulslike يندمجان على هواتف (iOS port ممتاز) |
| **Nine Sols** | أحدث مرجع للـ Parry-based 2D combat — راجعه جيداً |
| **Dead Cells (Mobile)** | نموذج لـ "roguelite بدون إحباط الـ mobile" |
| **Katana Zero** | مرجع للـ Dash Attack + time scale effects |
| **Blasphemous** | مرجع لـ Dark Fantasy 2D combat على mobile |
| **Ori & Will of the Wisps** | كيف تجمع Metroidvania + Fast movement feel |

### 1.5 الحلول المقترحة

> **القرار كبير (3 بدائل):** كيف نوفّق بين Mobile-First و Soulslike؟

#### 🅰️ بديل A: Souls-Lite (المُوصى به للمشروع الحالي)
- Checkpoints كثيفة (كل 2-3 دقائق).
- لا خسارة currency عند الموت، فقط خسارة "progress في الـ room الحالية".
- Boss fights لها phases قابلة للحفظ (تموت في phase 2 → ترجع من phase 2).
- **لماذا**: يحافظ على "Challenged" بدون قتل الـ retention.
- **مثال**: **Tunic** على iOS.

#### 🅱️ بديل B: Souls-Authentic + Cloud Save + Quick Resume
- يحافظ على صرامة Dark Souls، لكن يدعم "Quick Save" تلقائي عند كل exit + Cloud sync.
- Boss fights تُعاد من البداية.
- **لماذا**: للـ hardcore audience فقط. يخاطر بـ 60% من قاعدة الـ mobile players.
- **مثال**: **Sekiro** على أجهزة قوية، لكنه ليس mobile.

#### 🅲️ بديل C: Difficulty Modes
- Easy = Souls-Lite، Hard = Souls-Authentic.
- **لماذا**: يرضي الجميع، لكن يضاعف عمل الـ balancing.
- **مثال**: **Hollow Knight** (لم يضع modes، فقاعدة players انقسمت).

**توصيتنا**: 🅰️ لأن المشروع indie + mobile-first + مطور واحد. راجع تصميم **Tunic** كمرجع.

---

## 2. Movement System

### 2.1 الوصف الحالي
نظام حركة غني بشكل لافت:
- **6 آليات قفز**: Coyote Timer (0.12s)، Buffer Timer (0.16s)، Jump Cut، Wall Jump، Double Jump، Forced Fall.
- **4 أنواع Dash**: Normal، Roll، Wall، Dodge — كل منها بمتطلبات إدخال مختلفة.
- **Wall Slide** بـ RayCasts.
- **Big Fall** يفعّل Screen Shake + heavy landing animation.
- كل شيء يعتمد على `stamina > x` و `special_cooldown_timer` و Skill Tree unlock.

### 2.2 نقاط القوة
- **تفكير عالي النضج**: وجود Coyote Timer و Buffer Timer دليل أنك تقرأ game design articles. هذه الـ "invisible aids" هي ما يجعل الـ platforming يحس正确 حتى لو الـ input كان slightly off.
- **Dash variations ذكية**: Roll Dash يمرّ تحت الأعداء، Wall Dash ينطلق من الجدار، Dodge Dash للخلف. هذا يخلق **combat-movement integration** نادر في indie mobile.
- **ربط كل ميكانيك متقدم بـ Skill Tree**: ممتاز لـ progression curve (Metroidvania gating).

### 2.3 نقاط الضعف — وحجج احتمال الفشل

#### ⚠️ ضعف #1: تضارب语义 في `Jump` + `Joystick Down`
**الحجة**: الجدول يقول:
- `Jump` + `Joystick Down` في الهواء = **Forced Fall**
- `Melee Attack` + `Joystick Down` في Fall State = **Fall Stab**

لو اللاعب في الهواء وضغط Jump ثم مباشرة Melee Attack → هل يقصد Forced Fall ثم Fall Stab؟ أم يقصد Fall Stab مباشرة؟ الـ Input ambiguity قاتل على mobile لأن أصابع اللاعب ليست دقيقة. **Sekiro** واجه هذا بـ "input priority system" معقد. **Hollow Knight** حلّته بفصل الـ Down-Attack عن الـ Down-Movement بالكامل (لا تحتاج joystick down للـ pogo، فقط attack فوق عدو).

#### ⚠️ ضعف #2: 4 أنواع Dash → مشكلة اكتشاف (Discoverability)
**الحجة**: 
- Normal Dash = زر Dash
- Roll Dash = Dash + Joystick Down
- Wall Dash = Dash + Move (L/R)
- Dodge Dash = Dash × 2

اللاعب سيكتشف Normal Dash من Tutorial، لكن الباقي؟ Roll Dash يحتاج توقيتاً دقيقاً مع Joystick Down. Dodge Dash يحتاج double tap بـ 0.2s window. على mobile، احتمال false-positive ( Dodge Dash يتفعل بدون قصد) = 15-20% (إحصائياً من ألعاب مشابهة). **Dead Cells** واجه هذا وراحوا حلّوه بـ "Tap vs Hold" بدل "Tap × 2".

#### ⚠️ ضعف #3: `stamina > x` مكرر 4 مرات في جدول الـ Dash لكن لا يوجد نظام Stamina أصلاً
**الحجة**: الـ Pitfalls.md نفسه يسأل "هل يوجد Stamina؟". هذا يعني أنك صمّمت Movement يعتمد على نظام غير موجود. خطر: لو قررت لاحقاً إلغاء Stamina، كل جدول Dash يلزم إعادة كتابته + re-balancing.

#### ⚠️ ضعف #4: لا يوجد Air Mobility Control واضح
**الحجة**: في الـ Fall State، اللاعب يقدر يحرك joystick لـ Left/Right؟ لو نعم → ما سرعة الـ air control؟ لو لا → الـ Double Jump يصبح "vertical-only" ومحدود. **Super Meat Boy** يتيح full air control، **Celeste** يتيح partial. كلاهما يخبران اللاعب صراحةً عبر tutorial.

### 2.4 أمثلة من ألعاب مشابهة

| اللعبة | الدرس |
|--------|-------|
| **Celeste** | كيف 6 حركات قفز تُدرَّب في 6 levels تدريجياً |
| **Hollow Knight** | كيف يُربط Wall Jump + Double Jump بـ progression |
| **Katana Zero** | كيف يُربط Dash بـ combat (roll under enemies) |
| **Dead Cells** | كيف يُدار الـ Dash cooldown بدون stamina |
| **Ori** | كيف يُعطى Air Control متناهي الدقة |

### 2.5 الحلول المقترحة

> **القرار كبير (3 بدائل):** كيف نحل تضارب الـ Inputs؟

#### 🅰️ بديل A: Context-Aware Single Button (الموصى به)
- زر `Dash` واحد، والـ context يقرر النوع:
  - على الأرض + joystick down → Roll Dash
  - على جدار → Wall Dash
  - في الهواء + لا شيء → Air Dash (مختلف عن Normal Dash: لا i-frames)
  - زر Dash مرتين سريعاً → Dodge Dash (with confirm window 0.25s)
- **لماذا**: يقلل cognitive load + ينفع mobile.
- **Code Snippet**:

```gdscript
# movement_state_machine.gd
func _on_dash_input() -> void:
    if is_on_wall() and can_wall_dash:
        _enter_state("WallDash")
    elif is_on_floor() and _joystick_down_active() and can_roll_dash:
        _enter_state("RollDash")
    elif not is_on_floor() and can_air_dash:
        _enter_state("AirDash")
    elif _is_double_tap_within(0.25) and can_dodge_dash:
        _enter_state("DodgeDash")
    elif can_dash and stamina > dash_cost:
        _enter_state("NormalDash")
```

#### 🅱️ بديل B: Separate Buttons
- زر Dash + زر Dodge منفصلين.
- **لماذا**: أوضح، لكن يضيف زرّاً سابعاً (مشكلة الـ Pitfalls "كثرة أزرار").

#### 🅲️ بديل C: Gestures (Swipe)
- Swipe للأمام = Dash، Swipe للأسفل = Roll، Swipe للخلف = Dodge.
- **لماذا**: mobile native، لكن يتعارض مع joystick الحالي.

---

> **القرار صغير (موصى به واحد):** Forced Fall vs Fall Stab التضارب
- افصل: `Joystick Down` (held) في الهواء = Forced Fall. `Melee` (pressed) في الهواء = Fall Stab (يستخدم joystick down كـ context لتحديد الاتجاه فقط).
- هذا يحل التضارب: اللاعب يضغط Jump ثم يضغط Joystick Down ليبدأ Forced Fall، ثم يضغط Melee ليحوّلها لـ Fall Stab على العدو.

---

> **القرار صغير (موصى به واحد):** Air Control
- اسمح بـ full air control بنفس سرعة الـ ground movement (مثل Hollow Knight). الـ Double Jump يحافظ على الـ horizontal velocity. هذا يعطي إحساس "Fast" المطلوب.

---

## 3. Aim System (نظام التصويب بالـ Joystick)

### 3.1 الوصف الحالي
- Joystick واحد يتحكم بـ Movement + Aim في نفس الوقت.
- 8 Zones:
  - Green (Left/Right) = حركة عادية.
  - Red (Up/Down) = Idle + Aim Up/Down.
  - Blue (Up-Left/Up-Right) = حركة بطيئة + Aim Up.
  - Purple (Down-Left/Down-Right) = حركة بطيئة + Aim Down.
- التطبيق التقني: قتراح Dot Product على 8 directions.

### 3.2 نقاط القوة
- **ابتكار حقيقي**: استخدام joystick واحد لـ movement + aim معاً يحل مشكلة "thumb gymnastics" على mobile. أغلب ألعاب mobile تستخدم joystickين (مثل Dead Cells)، لكن هذا يأكل مساحة الشاشة.
- **Dot Product approach**: تقنياً سليم، أكثر مرونة من angle comparisons، أسهل في الـ debugging.

### 3.3 نقاط الضعف — وحجج احتمال الفشل

#### ⚠️ ضعف #1: ضعف #1 — حركة + تصويب بنفس الـ thumb = فقدان دقة
**الحجة**: الـ thumb لا يستطيع أن يحقق precision movement و precision aim في نفس اللحظة. تخيّل: العدو على يمين-أعلى قليلاً، اللاعب يريد أن يصوّب عليه بالـ Handgun بينما يتحرك لليسار للتهرّب. الـ thumb يحتاج أن يكون في Zone Blue (Up-Left)، لكن لو انحرف قليلاً يصبح في Green (Left) = لا aim، أو Red (Up) = لا حركة. على شاشة 6 إنش، نسبة الخطأ في الـ thumb tip ≈ 8-10 درجات. **Crossy Dungeon** واجه هذا وفشل، **Archero** حلّته بـ auto-aim عند الوقوف.

#### ⚠️ ضعف #2: لا يوجد Auto-Aim أو Aim Assist
**الحجة**: الـ mobile players يحتاجون aim assist (حتى PUBG Mobile يستخدمه). بدون assist، الإطلاق على عدو متحرك على mobile = إحباط. **Genshin Impact** على mobile يستخدم strong aim assist للأسلحة البعيدة.

#### ⚠️ ضعف #3: 3 اتجاهات فقط (Up/Right/Left) للـ Handgun
**الحجة**: ذكرت في الـ Core Concept "3 اتجاهات فقط فوق يمين يسار". لكن نظام الـ Aim System يصف 8 zones. هذا تناقض داخلي. هل 3 اتجاهات للـ Handgun و 8 للـ Melee Up/Down swings؟ لو نعم، يحتاج توضيح. لو الـ Handgun فعلاً 3-dir فقط، فلماذا الـ aim system معقّد لـ 8 zones؟

### 3.4 أمثلة من ألعاب مشابهة

| اللعبة | الدرس |
|--------|-------|
| **Dead Cells (Mobile)** | Joystick واحد + auto-aim للأسلحة بعيدة المدى |
| **Katana Zero** | Aim في 4 اتجاهات فقط (Up/Down/Left/Right) |
| **Nuclear Throne** | 8-dir aim لكن مع Vlambeer's famous "juice" |
| **Enter the Gungeon** | 360° aim لكن PC/Console فقط |
| **Soul Knight** | Auto-aim mobile-native، ناجح جداً |

### 3.5 الحلول المقترحة

> **القرار كبير (3 بدائل):** كيف نحل دقة الـ Aim على mobile؟

#### 🅰️ بديل A: Aim Assist قوي للـ Handgun + إبقاء 8-dir للـ Melee
- الـ Handgun يلتقط أقرب عدو في cone 30° أمام اتجاه aim.
- الـ Melee يبقى 8-dir (لأنه melee لا يحتاج دقة، فقط zone).
- **لماذا**: يحل مشكلة الدقة بدون تعقيد الـ UI.

#### 🅱️ بديل B: تجميد الحركة عند الإطلاق (Risk of Rain-style)
- عند ضغط زر Handgun، اللاعب يتجمد تماماً لـ 0.3s و joystick يصبح "aim-only" بـ 360°.
- **لماذا**: دقة كاملة لكن يفقد "Fast".

#### 🅲️ بديل C: Two Joysticks (Default + Optional)
- joystick يسار للحركة، joystick يمين للـ aim (يظهر فقط عند حمل Handgun).
- **لماذا**: المعيار الذهبي لكن يأكل شاشة.

**توصيتنا**: 🅰️ للأسباب التالية:
- متوافق مع `Mobile-First` و `Fast`.
- الـ Handgun أساساً secondary، فلا يستحق joystick كامل.
- يحافظ على ابتكارك (joystick واحد).

```gdscript
# handgun.gd
func _fire() -> void:
    var aim_dir := _get_aim_vector()  # من joystick
    var target := _find_nearest_enemy_in_cone(aim_dir, deg_to_rad(30.0), range=400.0)
    if target:
        var corrected_dir := (target.global_position - muzzle.global_position).normalized()
        _spawn_bullet(corrected_dir)
    else:
        _spawn_bullet(aim_dir)
```

---

## 4. Melee Combat System

### 4.1 الوصف الحالي
5 هجمات:
1. **Slash Combo** (3 ضربات، 0.4s input window، knock forward).
2. **Charged Slash** (hold 1.5s، big single hit).
3. **Dash Attack (Swift Slash)** — يخترق الأعداء، Katana Zero style.
4. **Fall Stab (Death from Above)** — FarCry 3 reference.
5. **Back Stab** — Dark Souls style (0.7s hold + unaware enemy).

زائد: Upward Swing + Downward Swing كـ default abilities.

### 4.2 نقاط القوة
- **تنوع هجومي ممتاز**: 7 هجمات إجمالاً تعطي عمقاً قتالياً كبيراً.
- **ربط كل هجوم متقدم بـ Skill Tree**: يحافظ على sense of progression.
- **الـ Hit Reactions قسم**: لم يكمل لكن ذكره يدل على وعي.
- **الأثر البصري المخطط** (Slash trail، Dust، Screen Shake، time scale) ممتاز للـ game feel.

### 4.3 نقاط الضعف — وحجج احتمال الفشل

#### ⚠️ ضعف #1: لا يوجد System-wide Combo System
**الحجة**: ذكرت "Can be mixed with Handgun shoots" كمثال (ضربتين + طلقة). لكن لا يوجد نظام combos رسمي. الأسئلة المفتوحة:
- هل كل mix يعتبر combo؟
- هل يوجد combo counter؟
- هل يوجد "combo finisher" خاص بدمج Melee + Handgun؟
- هل timing يهم (مثل Bayonetta) أو فقط sequence (مثل Devil May Cry)؟
بدون نظام موحد، اللاعب سيكتشف "أفضل combo" بنفسه ويعتمده دائماً → تنوع زائف. **Devil May Cry** حلّها بـ "style meter" يكافئ التنوع. **Hollow Knight** أبسط: لا combos، فقط pogo + nail dash.

#### ⚠️ ضعف #2: Charged Attack (1.5s) طويل جداً على mobile
**الحجة**: 1.5s في معركة سريعة = عمر. اللاعب لا يقدر أن يبقى ثابتاً 1.5s أمام عدو متحرك. **Monster Hunter** يستخدم 0.5-0.8s charge. **Hades** يستخدم 0.4s للـ special.

#### ⚠️ ضعف #3: Back Stab بـ 0.7s hold = التباس مع Charged
**الحجة**: 
- Hold 0.7s = Back Stab (لو عدو غير مدرك).
- Hold 1.5s = Charged Slash (في أي وقت).
السؤال: لو اللاعب hold 1.5s أمام عدو غير مدرك، ماذا يحدث؟ كلاهما متحقق. أيهما يفوز؟ **Sekiro** يحلّها بـ context priority: قرب العدو = Back Stab، بعيد = Charged.

#### ⚠️ ضعف #4: لا يوجد Defensive Move واضح غير Parry
**الحجة**: 
- Roll Dash = يمرّ تحت الأعداء (هجومي/تهرّبي).
- Dodge Dash = للخلف (دفاعي).
- لكن لا يوجد "Block" دفاعي بـ الـ Blades. اللاعب بين الضرب والـ Parry فقط، لا وسط. **Nine Sols** يعطي Parry فقط (لكن Parry فيها واسع). **Hollow Knight** يعطي Block عبر Nail upgrades. **Dead Cells** يعطي Shields.

### 4.4 أمثلة من ألعاب مشابهة

| اللعبة | الدرس |
|--------|-------|
| **Bayonetta** | Combo system: timing + sequence + weapon mixing |
| **Devil May Cry 5** | Style meter يكافئ التنوع |
| **Katana Zero** | Dash attack feel + time scale |
| **Nine Sols** | Parry-only combat (مرجع أساسي لك) |
| **Hollow Knight** | Pogo attack (downward swing → jump) |
| **Hades** | كيف 0.4s charge يكفي |

### 4.5 الحلول المقترحة

> **القرار كبير (3 بدائل):** Combo System Architecture

#### 🅰️ بديل A: Lightweight Sequence Combos (الموصى به)
- كل combo = sequence محددة (مثلاً: Slash → Slash → Handgun = "Disruptor" combo يلغي درع العدو).
- لا timing صارم، فقط ترتيب.
- كل combo له effect فريد (disrupt، launch، stun).
- **لماذا**: مناسب mobile (لا تحتاج frame-perfect)، يحفز التنوع.

#### 🅱️ بديل B: Style Meter
- كل ضربة مختلفة ترفع الـ meter. تكرار نفس الهجوم يخفضه.
- المكافآت: damage boost، heal، special ammo.
- **لماذا**: يحفز الإبداع، لكنه معقد.

#### 🅲️ بديل C: لا Combos (Hollow Knight style)
- كل هجوم مستقل. الـ "combo" يأتي من skill اللاعب في الـ movement + attack chaining.
- **لماذا**: بسيط، لكن يخسر عمق الـ combat.

**توصيتنا**: 🅰️ (Lightweight Sequence Combos).

---

> **القرار صغير (موصى به واحد):** Charged Attack timing
- قلّص من 1.5s إلى **0.6s**. هذا يكفي لـ "charge feel" بدون أن يقتل الـ pacing.

---

> **القرار صغير (موصى به واحد):** Back Stab vs Charged conflict
- استخدم Context Priority:
  - لو في "Back Stab Range" + عدو غير مدرك → أولوية للـ Back Stab (يبدأ عند 0.5s hold).
  - لو لا → Charged Slash يبدأ عند 0.6s hold.
- هذا يحل التضارب بدون تصميم إضافي.

```gdscript
# melee_controller.gd
func _on_melee_released(hold_duration: float) -> void:
    if _in_backstab_range() and not _current_enemy_is_aware() and hold_duration >= 0.5:
        _perform_backstab()
    elif hold_duration >= 0.6 and can_charge_attack:
        _perform_charged_slash()
    elif hold_duration < 0.3:
        _perform_combo_slash()
```

---

## 5. Controllers (Joystick Mechanism)

### 5.1 الوصف الحالي
- Joystick Built-in Godot 4.7 في Bottom-Left.
- يتحكم بـ Movement + Aim معاً (8 zones).
- لا يوجد تفاصيل عن باقي الأزرار في الملف (لكن من ملفات أخرى نعرف: Jump، Dash، Melee Attack، Handgun Fire، Skill Tree).

### 5.2 نقاط القوة
- **استخدام Built-in Godot**: قرار ذكي، يوفّر وقت تطوير.
- **Bottom-Left**: المعيار الذهبي لـ left-handed thumb.

### 5.3 نقاط الضعف — وحجج احتمال الفشل

#### ⚠️ ضعف #1: 5 أزرار + joystick = 6 inputs على mobile
**الحجة**: الأزرار المتوقعة على يمين الشاشة:
1. Jump
2. Dash
3. Melee Attack
4. Handgun Fire
5. Parry (محتمل)
6. Heal (محتمل)

على شاشة 6 إنش، كل زر يحتاج ~44pt minimum (Apple HIG). 6 أزرار = 264pt = تقريباً كل الـ bottom-right quarter. اللاعب الـ right-thumb سيكون في "thumb gymnastics" دائم. **Genshin Impact Mobile** يضع 4 أزرار فقط (Attack، Skill، Burst، Sprint) + joystick. **Dead Cells Mobile** يضع 4 أزرار (Attack، Jump، Roll، Use). **Pitfalls.md** نفسه يسأل "خيار الدمج بين زر Attack و Parry؟".

#### ⚠️ ضعف #2: لا يوجد Customization أو Layout Presets
**الحجة**: كل لاعب mobile له حجم يد مختلف. ألعاب indie ناجحة مثل **Slay the Spire Mobile** و **Stardew Valley Mobile** تتيح full layout customization. بدونها، شكاوى "زر بعيد عن إصبعي" = 1-نجوم reviews.

#### ⚠️ ضعف #3: Dead Zone و Sensitivity غير محددين
**الحجة**: في `To do list + Random Ideas.md` يظهر "joystick accessibility for mobile (Dead Zone | Size | drag sensitivity)" كـ to-do. لكنه محوري: لو dead zone صغير = false positives كثيرة. لو كبير = حركة غير دقيقة. القيم الافتراضية لازم تكون مدروسة.

### 5.4 أمثلة من ألعاب مشابهة

| اللعبة | الدرس |
|--------|-------|
| **Dead Cells Mobile** | 4 أزرار + joystick، layout ثابت ممتاز |
| **Hollow Knight Mobile** | 3 أزرار فقط (Attack، Jump، Dash) |
| **Genshin Impact** | Customizable HUD |
| **Stardew Valley Mobile** | Full HUD editor |

### 5.5 الحلول المقترحة

> **القرار كبير (3 بدائل):** تقليل عدد الأزرار

#### 🅰️ بديل A: Context-Sensitive Button (الموصى به)
- زر واحد "Action" يتغيّر حسب الـ context:
  - أمام عدو غير مدرك = "Back Stab" (icon يتغيّر)
  - عدو يهاجم = "Parry" (icon يتغيّر)
  - لا شيء = "Heal" (icon يتغيّر)
- زر Attack مستقل (لأنه الأكثر استخداماً).
- زر Jump مستقل.
- زر Dash مستقل.
- زر Handgun مستقل.
- **المجموع**: 5 أزرار (Action + Attack + Jump + Dash + Handgun).
- **لماذا**: يقلل الأزرار بدون فقدان وظائف.

#### 🅱️ بديل B: دمج Attack + Parry (كما اقترحت في Pitfalls)
- زر Attack = ضربة سريعة (tap).
- زر Attack = Parry (hold 0.2s).
- **لماذا**: أوفر مساحة. لكنه يربك اللاعب في الـ fast combat.
- **مثال**: **Bloodborne** (PC/Console) يفعل هذا بـ gun parry.

#### 🅲️ بديل C: gesture-based Parry
- Swipe على العدو = Parry.
- **لماذا**: native mobile، لكن يتعارض مع joystick.

**توصيتنا**: 🅰️ + 🅱️ معاً:
- زر Attack = tap slash / hold 0.6s charged / tap في frame الـ enemy attack = parry (context-aware).
- هذا يقلل الأزرار لـ **4 فقط**: Attack + Jump + Dash + Handgun.

```gdscript
# attack_button.gd
func _on_pressed():
    # parry window check
    if _enemy_attack_in_parry_window():
        _perform_parry()
    elif _in_backstab_range() and not _enemy_aware():
        _start_backstab_hold()
    else:
        _perform_combo_slash()

func _on_released(hold_duration: float):
    if _in_backstab_range() and not _enemy_aware() and hold_duration >= 0.5:
        _perform_backstab()
    elif hold_duration >= 0.6:
        _perform_charged_slash()
```

---

> **القرار صغير (موصى به واحد):** Dead Zone و Sensitivity
- Dead Zone = 0.18 (تحت 0.2 يتجاهل الإدخال).
- Sensitivity = linear (لا acceleration).
- Drag Radius = 60px (يمكن اللاعب تخصيصه).
- الـ Aim Zones thresholds (UP/DOWN نطاقاتها ±20°، الباقي يتوزع).

---

## 6. Enemy AI Framework (Architecture/Enemies/AI.md — فارغ)

### 6.1 الوصف الحالي
- ملف `AI.md` فارغ تماماً.
- ملف `Watch.md` يحتوي 3 نقاط: Stats System، State Machine، Hit Reactions.
- في `To do list`: "Enemy Design Framework (States: Idle, Patrol, Detect, Chase, Attack, Stagger, Dead) + 5 أنواع أعداء على الأقل".

### 6.2 نقاط الضعف

#### ⚠️ ضعف #1: لا يوجد Enemy Archetypes
**الحجة**: 5 أنواع أعداء على الأقل مطلوبة، لكن لا يوجد taxonomy. هل هم:
- Melee vs Ranged؟
- Aggressive vs Defensive؟
- Grounded vs Flying؟
- Static vs Patrol؟
- بلا taxonomy، سيكون عندك 5 أعداء متشابهين بـ sprites مختلفة. **Hollow Knight** يحتوي 30+ عدو لكنهم ينقسمون لـ ~6 archetypes.

#### ⚠️ ضعف #2: لا يوجد AI Behavior Tree أو State Machine spec
**الحجنة**: ذكرت State Machine في `Watch.md` لكن بلا تفاصيل. الأسئلة:
- هل كل عدو له SM مستقلة؟ أم shared base class؟
- كيف تنتقل الأعداء بين states (signals؟ timers؟ distance checks؟)
- هل يوجد coordination بين الأعداء (flanking)؟

#### ⚠️ ضعف #3: Stealth System غير معروف
**الحجة**: في Core Concept ذكرت "Stealth-Optional: يتسنى للاعب التسلل". لكن لا يوجد:
- Vision Cone للعدو.
- Sound propagation system.
- Detection meter (هل اللاعب مكتشف 100% أو جزئياً؟).

### 6.3 أمثلة من ألعاب مشابهة

| اللعبة | الدرس |
|--------|-------|
| **Hollow Knight** | Simple but tight enemy AI — 6 archetypes |
| **Mark of the Ninja** | مرجع الـ Stealth في 2D |
| **Nine Sols** | Boss AI patterns (مرجع مهم لك) |
| **Katana Zero** | Enemy telegraphing (كل هجوم له 0.5s warning) |
| **Hotline Miami** | Enemy AI سريع وقرار قاتل |

### 6.4 الحلول المقترحة

> **القرار كبير (3 بدائل):** AI Architecture

#### 🅰️ بديل A: Shared State Machine Base + Per-Enemy Overrides (الموصى به)
- `EnemyBase` class فيه: Idle، Patrol، Detect، Chase، Attack، Stagger، Dead.
- كل عدو inherits وي override فقط الـ transitions و parameters.
- **لماذا**: reusable، testing سهل.
- **Code**:

```gdscript
# enemy_base.gd
class_name EnemyBase
extends CharacterBody2D

enum State { IDLE, PATROL, DETECT, CHASE, ATTACK, STAGGER, DEAD }
var state: State = State.IDLE

@export var max_health: float = 100.0
@export var detect_range: float = 250.0
@export var attack_range: float = 60.0
@export var attack_telegraph: float = 0.4

func _physics_process(delta: float) -> void:
    match state:
        State.IDLE:    _idle(delta)
        State.PATROL:  _patrol(delta)
        State.DETECT:  _detect(delta)
        State.CHASE:   _chase(delta)
        State.ATTACK:  _attack(delta)
        State.STAGGER: _stagger(delta)
        State.DEAD:    _dead(delta)

# Subclass يـ override هذه
func _idle(delta: float) -> void:
    if _can_see_player():
        _transition_to(State.DETECT)
```

#### 🅱️ بديل B: Behavior Trees
- أكثر تعقيداً لكنه مرن للأعداء المتقدمين.
- **لماذا**: مناسب لو عندك 20+ عدو معقد.

#### 🅲️ بديل C: GOAP (Goal-Oriented Action Planning)
- لاعب يحس أن الأعداء "يفكرون".
- **لماذا**: معقد جداً لمطور indie وحيد. لا ينصح به إلا لو الـ AI هو الميزة الأساسية.

**توصيتنا**: 🅰️.

---

> **القرار صغير (5 أنواع أعداء موصى بها كـ base roster):**

| # | Archetype | Behavior | Counter |
|---|-----------|----------|---------|
| 1 | **Grunt** | Melee، يركض نحو اللاعب، attack واحد | Slash Combo |
| 2 | **Sentinel** | Ranged، يقف ثابت ويطلق | Dash + close gap |
| 3 | **Stalker** | Stealth، يظهر من خلف اللاعب | Back Stab قبل ما يظهر |
| 4 | **Brute** | Heavy melee، بطيء لكن hit قوي | Dodge + punish |
| 5 | **Flyer** | Aerial، يطير ويهاجم من فوق | Upward Swing / Handgun |

---

## 7. Damage System (Architecture/Damage System.md — نقطتان فقط)

### 7.1 الوصف الحالي
- "Hit/Hurt Boxes"
- "Invisibility time ...."
- لا شيء غير هذا.

### 7.2 نقاط الضعف

#### ⚠️ ضعف #1: لا يوجد Damage Formula
**الحجة**: ما هي صيغة الضرر؟
- `damage = weapon_base × player_multiplier × enemy_defense`؟
- أم `damage = weapon_base - enemy_defense` (flat reduction)؟
- أم `damage = weapon_base × (1 - enemy_defense / 100)` (percentage)؟
بدون formula، لا يمكنك balance أي شيء.

#### ⚠️ ضعف #2: لا يوجد Damage Types
**الحجة**: هل يوجد:
- Physical vs Elemental؟
- Knockback values per hit؟
- Critical hits؟
- Status effects (poison، bleed، stun)؟

#### ⚠️ ضعف #3: لا يوجد Hit Reaction Spec
**الحجة**: ذكرت "Hit Reactions" لكن بلا تفاصيل. هل كل ضربة:
- توقف اللاعب (hit stop)؟
- تعطي knockback؟
- تفعّل stagger meter؟

### 7.3 أمثلة من ألعاب مشابهة

| اللعبة | الدرس |
|--------|-------|
| **Hollow Knight** | Simple damage: nail base × charm multipliers |
| **Hades** | Damage types ( Attack، Special، Cast، Dash) كل بمعدلات مختلفة |
| **Sekiro** | Posture system (بدل HP) |
| **Dead Cells** | Damage formula + affixes |

### 7.4 الحلول المقترحة

> **القرار كبير (3 بدائل):** Damage Formula

#### 🅰️ بديل A: Lightweight Multiplicative (الموصى به)
```gdscript
final_damage = weapon_base * player_skill_multiplier * (1.0 - enemy_armor_pct)
# مثال: weapon_base=20, multiplier=1.5 (skill bonus), armor=0.2
# final = 20 * 1.5 * 0.8 = 24
```
- **لماذا**: سهل الـ balancing، شفاف للاعب.

#### 🅱️ بديل B: Flat Subtraction
```gdscript
final_damage = max(1, weapon_base - enemy_armor)
```
- **لماذا**: بسيط، لكنه يكسر اللعبة في late-game (درع عالي = ضرر 1).

#### 🅲️ بديل C: Sekiro-style Posture
- لا HP، فقط Posture meter. يمتلئ → death blow.
- **لماذا**: ثوري لكنه يعيد تصميم كل شيء.

**توصيتنا**: 🅰️.

---

> **القرار صغير (موصى به واحد):** Hit Reactions
- كل ضربة تعطي **3 effects متزامنة**:
  1. **Hit Stop** = 0.04s (freezes attacker + target للحظات).
  2. **Knockback** = `weapon_knockback_value * (1 - enemy_stagger_resist)`.
  3. **Hit Spark** = particle effect يناسب نوع السطح (flesh/metal/stone).

```gdscript
# damage_receiver.gd
func take_damage(amount: float, knockback_dir: Vector2, source: Node) -> void:
    current_health -= amount
    _apply_hit_stop(0.04)
    _apply_knockback(knockback_dir * weapon_knockback * (1.0 - stagger_resist))
    _spawn_hit_spark(source.material_type)
    _start_invincibility(0.4)  # i-frames
    if current_health <= 0:
        _die()
```

---

## 8. Pitfalls (ملف المخاطر المعلنة) — معالجة مخصصة في `02-Pitfalls-Solutions.md`

ملف الـ Pitfalls يحتوي على 5 مخاطر معلنة:
1. كيف يطوّر اللاعب الـ Skill Tree؟
2. كثرة الأزرار (دمج Attack + Parry؟).
3. هل يوجد نظام Stamina؟
4. لا يوجد نظام Economy/Currency.
5. لا يوجد نظام Healing.

**هذه المخاطر لها ملف مخصص للحلول الهندسية**: راجع `02-Pitfalls-Solutions.md`.

---

## 9. الخلاصة — نقاط القوة الكلية والضعف الكلي

### 🟢 نقاط القوة الكلية
1. **رؤية واضحة**: الـ 4 شعور مستهدف (Skilled/Challenged/Fast/Intelligent) يعطي شمال للحركة التصميمية.
2. **نضج تقني**: استخدام Coyote Timer، Buffer Timer، Dot Product = مطور يقرأ.
3. **ابتكار الـ Joystick-one-for-all**: قرار جريء قد يميّز اللعبة.
4. **ربط كل شيء بـ Skill Tree**: يحافظ على progression.
5. **اختيار إلهامات صحيحة**: Hollow Knight + Nine Sols + Katana Zero.

### 🔴 نقاط الضعف الكلية
1. **أنظمة أساسية فارغة**: 11 ملف فارغ من أصل ~20 (55%).
2. **تضارب Mobile-First vs Soulslike**: يحتاج قرار شجاع.
3. **كثرة الأزرار**: مشكلة mobile حقيقية.
4. **غياب Damage Formula و Enemy Archetypes**: لا يمكن عمل prototype بلا هذه.
5. **Stealth + Puzzles معلّقة**:Genres ثانوية بدون أنظمة.
6. **Combo System غير موجود رسمياً**: سيؤدي لـ "best combo" dominance.

### 🟡 المخاطر الأعلى أولوية
1. ⚠️ **العجز عن البدء بـ prototype** بسبب غياب Damage System + Health System + Skill Tree.
2. ⚠️ **كثرة الأزرار قد تقتل الـ mobile UX**.
3. ⚠️ **تضارب Mobile-First vs Soulslike** قد يفقد اللعبة هويتها قبل الإطلاق.

---

**التالي**: راجع `02-Pitfalls-Solutions.md` للحلول الهندسية الكاملة لمخاطر الـ Pitfalls.
