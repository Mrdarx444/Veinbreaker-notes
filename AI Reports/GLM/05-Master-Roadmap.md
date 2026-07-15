# خارطة الطريق الشاملة — Veinbreaker

> **الهدف**: تجميع كل التوصيات في ملف واحد قابل للتنفيذ. هذا الملف هو "north star" لتطوير المشروع.
> **المنهجية**: 
> - أولويات واضحة (P0 / P1 / P2 / P3).
> - كل مهمة = deliverable + acceptance criteria + dependencies.
> - تقدير زمني تقريبي (للمطور الوحيد).
> - مؤشرات نجاح (KPIs).

---

## 1. ملخص تنفيذي

### 1.1 أين أنت الآن؟
- ✅ **الرؤية واضحة**: Core Concept، 4 شعور مستهدف، Genres محددة.
- ✅ **أنظمة أساسية مصممة**: Movement (مفصّل)، Aim (تقني)، Melee (5 هجمات).
- ✅ **الإلهامات محددة**: Hollow Knight، Nine Sols، Katana Zero.
- ⚠️ **5 مخاطر معلنة** بدون حلول (Pitfalls).
- ⚠️ **11 نظام فارغ** من أصل ~20 ملف.
- ❌ **لا prototype بعد**.

### 1.2 أين تحتاج أن تكون؟
- MVP prototype قابل للّعب خلال **3-4 أشهر** (إذا عملت 20h/أسبوع).
- Vertical slice جاهز خلال **6-8 أشهر**.
- الإطلاق المتوقع (لـ mobile + PC): **18-24 شهراً**.

### 1.3 أهم 3 توصيات استراتيجية
1. **اقفل قرار الـ "Souls-Lite vs Souls-Authentic"** قبل أي كود. (راجع `01-Systems-Analysis.md` §1.5).
2. **اقفل تصميم الـ 5 أزرار** (Context-Sensitive Multi-Function). هذا يحدد كل UI.
3. **اقفل القصة** (Dark Fantasy أسهل بداية). هذا يحدد كل art + lore.

---

## 2. الأولويات الشاملة

### 🚨 P0 — قبل أي كود (1-3 أسابيع)
| #   | المهمة                                                    | Deliverable                                       | Time |
| --- | --------------------------------------------------------- | ------------------------------------------------- | ---- |
| 1   | كتابة `Skill Tree System.md` كامل                         | 20 skill موزعة على 4 branches                     | 4h   |
| 2   | كتابة `Health System.md` + `Damage System.md`             | الكود في `03-Empty-Systems-Foundations.md` §6, §9 | 3h   |
| 3   | كتابة `Enemy AI.md` + 5 archetypes                        | الكود في `03-Empty-Systems-Foundations.md` §2     | 4h   |
| 4   | كتابة `Camera System.md`                                  | الكود في `03-Empty-Systems-Foundations.md` §5     | 2h   |
| 5   | تحديث `Pitfalls.md` بالحلول                               | من `02-Pitfalls-Solutions.md`                     | 2h   |
| 7   | تحديث `Melee.md` (إضافة Combo System + تقليل Charge time) | تعديل القسم 4                                     | 2h   |
| 8   | تحديث `Core Concept.md` (تحديد 3-aim vs 8-aim)            | سطر توضيحي                                        | 0.5h |
| 9   | كتابة `Narrative Bible` (1 صفحة)                          | من `04-Story-Concepts.md` §1 أو §4                | 2h   |
| 10  | **اتخاذ قرار**: Souls-Lite؟ (موصى به 🅰️)                 | yes/no                                            | 0.5h |

**إجمالي P0**: ~21 ساعة = 1-3 أسابيع (لو 10h/أسبوع).

---

### ⚡ P1 — أساس prototype (4-8 أسابيع)
| # | المهمة | Deliverable | Time |
|---|--------|------------|------|
| 11 | تنفيذ `InputSystem.gd` Autoload | من `03-Empty-Systems-Foundations.md` §1 | 4h |
| 12 | تنفيذ `SanguisManager.gd` + `CindersManager.gd` | من `02-Pitfalls-Solutions.md` §1, §4 | 3h |
| 13 | تنفيذ `Player.gd` + `MovementController` | من `03-Empty-Systems-Foundations.md` §3 + `Movement System.md` | 8h |
| 14 | تنفيذ `EnemyBase.gd` + 3 enemies (Grunt، Sentinel، Flyer) | من `03-Empty-Systems-Foundations.md` §2 | 12h |
| 15 | تنفيذ `DamageSystem.gd` Autoload | من `03-Empty-Systems-Foundations.md` §9 | 4h |
| 16 | تنفيذ `HealthController.gd` + Vein Flasks | من `02-Pitfalls-Solutions.md` §5 | 4h |
| 17 | تنفيذ `CameraController.gd` | من `03-Empty-Systems-Foundations.md` §5 | 4h |
| 18 | تنفيذ `MeleeController.gd` (5 هجمات + context-aware button) | من `02-Pitfalls-Solutions.md` §2 | 8h |
| 19 | تنفيذ `HandgunController.gd` + Bullet scene | من `03-Empty-Systems-Foundations.md` §10 | 6h |
| 20 | تنفيذ `ParryManager.gd` | من `03-Empty-Systems-Foundations.md` §11 | 6h |
| 21 | تنفيذ `SkillTree.gd` Autoload | من `03-Empty-Systems-Foundations.md` §8 | 4h |
| 22 | بناء Test Room (1 room قابلة للّعب) | Room A1 with 3 enemies + altar | 8h |
| 23 | تنفيذ UI: HUD (HP، Flasks، Ammo، Sanguis، Cinders) | — | 6h |
| 24 | تنفيذ Save/Load | من `03-Empty-Systems-Foundations.md` §8.5 | 3h |

**إجمالي P1**: ~84 ساعة = 4-8 أسابيع.

#### ✅ Acceptance Criteria للـ Prototype
- [ ] اللاعب يقدر يتحرك (walk, jump, dash) بسلاسة.
- [ ] اللاعب يقدر يضرب Grunt ويموت الـ Grunt.
- [ ] اللاعب يقدر يطلق Handgun على Sentinel.
- [ ] اللاعب يقدر يـ parry هجوم Grunt.
- [ ] اللاعب يقدر يفتح Skill Tree عند Vein Altar.
- [ ] اللاعب يقدر يشتري healing vial من merchant.
- [ ] اللاعب يقدر save/load.
- [ ] الموت يرجع اللاعب لآخر altar.
- [ ] كل الميكانيكيات تعمل بـ 60fps على device متوسط.

---

### 🎨 P2 — polish + content (8-16 أسبوع)
| # | المهمة | Notes |
|---|--------|-------|
| 25 | Game Feel: hit stop، screen shake، particles، SFX | راجع `01-Systems-Analysis.md` §4.2 |
| 26 | بناء 5 rooms إضافية (Region A كامل) | راجع `03-Empty-Systems-Foundations.md` §4 |
| 27 | تنفيذ 5 enemy types كاملة | راجع `03-Empty-Systems-Foundations.md` §2.5 |
| 28 | بناء 1 boss كامل (Boss Design Template) | ضروري قبل المضي |
| 29 | تنفيذ Map UI | راجع `03-Empty-Systems-Foundations.md` §7 |
| 30 | تنفيذ Tutorial Level | راجع `01-Systems-Analysis.md` §6.4 |
| 31 | Lore System (Memory Crystals / Lore Snippets) | راجع `04-Story-Concepts.md` §8.1 |
| 32 | Customizable HUD (layout presets) | راجع `01-Systems-Analysis.md` §5.4 |
| 33 | Audio System (SFX + Music + dynamic layers) | — |
| 34 | Settings Menu (audio، controls، accessibility) | — |

---

### 🚀 P3 — pre-launch (16+ أسبوع)
| # | المهمة |
|---|--------|
| 35 | Regions B-F (5 مناطق إضافية) |
| 36 | Bosses إضافية (3-5 bosses إجمالاً) |
| 37 | Multiple endings (3 endings) |
| 38 | Localization (عربي/إنجليزي على الأقل) |
| 39 | Cloud Save |
| 40 | Achievements |
| 41 | Performance optimization (mobile target 60fps) |
| 42 | QA + playtesting (10+ testers) |
| 43 | Marketing: trailer، store page، social media |
| 44 | Soft launch (TestFlight / Play Store beta) |
| 45 | Launch (iOS + Android + Steam) |

---

## 3. التوصيات الكلية (Master Recommendations)

### 🎯 توصيات تصميمية
| # | التوصية | الملف | الأولوية |
|---|---------|------|---------|
| R1 | اقرر **Souls-Lite** كنموذج (بديل A في §1.5 من `01`) | Core Concept | P0 |
| R2 | اعتمد **Context-Sensitive Multi-Function Button** (5 أزرار) | Controllers | P0 |
| R3 | **لا Stamina**، استخدم **Cooldowns** | Movement System | P0 |
| R4 | **Sanguis** (skill points) + **Cinders** (currency) | Economy | P0 |
| R5 | **Vein Flasks** (Estus-style) للـ healing | Health System | P0 |
| R6 | **Aim Assist** للـ Handgun (cone 30°) | Aim System | P1 |
| R7 | **Lightweight Sequence Combos** (Bayonetta-lite) | Melee | P1 |
| R8 | تقليل **Charged Attack** من 1.5s إلى 0.6s | Melee | P1 |
| R9 | **Air Control** full (مثل Hollow Knight) | Movement | P1 |
| R10 | **Tutorial integrated with story** (Ultrakill style) | Tutorial | P2 |

### 🎨 توصيات فنية
| # | التوصية | السبب |
|---|---------|------|
| T1 | استخدم Godot 4.7 (مثل ما اخترت) | مناسب للمشروع |
| T2 | اعتمد Autoloads لكل manager (SanguisManager، etc.) | singleton pattern |
| T3 | استخدم State Machines (لا Behavior Trees) | بساطة + كفاءة |
| T4 | كل enemy = inherits EnemyBase | reuse |
| T5 | حفظ بـ JSON في user:// | بسيط + mobile-friendly |
| T6 | استخدم FastNoiseLite للـ camera shake | performance |
| T7 | كل UI = Control nodes (لا raw draw) | Godot native |
| T8 | استخدم TileMapLayer للـ levels | Godot 4.x best practice |

### 📱 توصيات mobile
| # | التوصية |
|---|---------|
| M1 | اختبر الـ 5 أزرار على شاشة فعلية قبل البناء |
| M2 | **Dead Zone = 0.18**، Drag Radius = 60px |
| M3 | ابدأ بـ iPhone SE كـ minimum target |
| M4 | اختر 60fps (لا 120fps — استهلاك بطارية) |
| M5 | اعرض زاوية واحدة للـ portrait + زاويتين للـ landscape |
| M6 | ادعم iCloud / Google Drive للـ cloud save |
| M7 | ادرس **PUBG Mobile** و **Genshin Mobile** للـ control schemes |

### 📚 توصيات بحثية
- شاهد فيديوهات Naavik عن dead cells mobile design.
- اقرأ GDC talk "Designing Hollow Knight" (Team Cherry).
- العب Nine Sols (مرجعك الأساسي للـ Parry).
- العب Tunic (مرجعك للـ Souls-Lite على mobile).
- شاهد GMTK video عن Coyote Time و Buffer Time (للتعميق).

---

## 4. جدول زمني مقترح (12 شهراً)

```
شهر 1: P0 (القرارات + كتابة الـ .md)
   أسبوع 1: اقرارات التصميم + كتابة Skill Tree + Pitfalls solutions
   أسبوع 2: كتابة Enemy AI + Damage + Health + Camera
   أسبوع 3: تحديث الموجود + Narrative Bible
   أسبوع 4: راجع + prototype الـ controls على شاشة

شهر 2-3: P1 (Core prototype)
   أسبوع 5-6: Player + Movement + Camera
   أسبوع 7-8: Enemy AI + Damage System
   أسبوع 9-10: Combat (Melee + Handgun + Parry)
   أسبوع 11-12: Skill Tree + Economy + Save
   أسبوع 13: Test Room playable
   أسبوع 14: Internal playtest #1

شهر 4-5: P2 (Polish prototype)
   أسبوع 15-16: Game Feel (hit stop, shake, particles)
   أسبوع 17-18: HUD + UI polish
   أسبوع 19-20: 5 Rooms + 5 Enemies
   أسبوع 21: First Boss
   أسبوع 22: Playtest #2 + iterate

شهر 6-9: Vertical Slice
   أسبوع 23-30: Region A كامل (8 rooms + 1 boss)
   أسبوع 31-35: Tutorial + Lore + Audio
   أسبوع 36: Vertical slice complete
   أسبوع 37: External playtest (10 testers)

شهر 10-12: Content Production
   أسبوع 38-44: Regions B-D
   أسبوع 45-48: Bosses + Endings
   أسبوع 49-50: Localization + QA
   أسبوع 51-52: Soft launch + iterate
```

---

## 5. مخاطر المشروع و Risk Mitigation

### ⚠️ المخاطر التقنية
| الخطر | الاحتمال | التأثير | Mitigation |
|------|--------|--------|-----------|
| Godot 4.7 joystick bugs | متوسط | متوسط | استخدم virtual joystick plugin محترم |
| Mobile performance (60fps) | عالي | عالي | Optimize من اليوم الأول (draw calls < 100/room) |
| Save file corruption | متوسط | عالي | JSON validation + backup slots |
| Touch input lag | عالي | عالي | Godot 4.7 قلّل هذا، اختر 60fps |

### ⚠️ المخاطر التصميمية
| الخطر | الاحتمال | التأثير | Mitigation |
|------|--------|--------|-----------|
| كثرة الأزرار تقتل UX | عالي | قاتل | 5 أزرار صارمة من اليوم الأول |
| Souls-Authentic يطرد mobile players | عالي | قاتل | Souls-Lite (بديل A) |
| Combo System سيئ → meta dominance | متوسط | متوسط | Lightweight Combos + style meter |
| Lore ممل | متوسط | متوسط | Lore snippets قصيرة (3 جمل max) |
| Boss صعب جداً | متوسط | عالي | Boss Design Template صارم |

### ⚠️ المخاطر الإنتاجية
| الخطر | الاحتمال | التأثير | Mitigation |
|------|--------|--------|-----------|
| Scope creep | عالي | قاتل | ابدأ بـ Region A فقط، لا تتجاوز |
| Burnout (مطور وحيد) | عالي | قاتل | جدول واقعي + take breaks |
| Art bottleneck | عالي | متوسط | استخدم asset packs للبداية |
| Music/SFX bottleneck | متوسط | متوسط | استخدم ElevenLabs (كما في plans) |

---

## 6. مؤشرات النجاح (KPIs)

### 📊 KPIs للـ Prototype
- [ ] 5 testers خارجيين يلعبون 15 دقيقة بدون ما يسألوا "how do I...?"
- [ ] Frame rate ≥ 55fps على iPhone SE.
- [ ] معدل mortality للـ Grunt = 4-6 ضربات (suitable for tutorial enemy).
- [ ] معدل mortality للاعب = 3-5 hits من Grunt (suitable challenge).
- [ ] Save/Load time < 0.5s.
- [ ] وقت تحميل الـ game < 3s.

### 📊 KPIs للـ Vertical Slice
- [ ] Testers يكملون Region A خلال 45-90 دقيقة.
- [ ] معدل الإحباط (self-reported) < 30%.
- [ ] 80% من testers يفهمون الـ Skill Tree بدون tutorial.
- [ ] 70% من testers يستخدمون Parry بشكل صحيح بعد boss.
- [ ] 100% من testers يفهمون الـ plot الأساسي.

### 📊 KPIs للـ Launch
- [ ] Day-1 retention ≥ 35%.
- [ ] 7-day retention ≥ 15%.
- [ ] Average session time ≥ 15 min.
- [ ] Rating ≥ 4.0 on stores.
- [ ] Crash rate < 1%.

---

## 7. قائمة مرجعية للقرارات المعلّقة (Open Decisions)

هذه القرارات تحتاج إجابة منك قبل المضي:

| # | القرار | الخيارات الموصى بها | الموعد |
|---|--------|-------------------|-------|
| D1 | Souls-Lite vs Souls-Authentic? | 🅰️ Souls-Lite | قبل أي كود |
| D2 | 5 أزرار (Context-Sensitive)؟ | 🅰️ نعم | قبل أي كود |
| D3 | Stamina؟ | 🅰️ لا، Cooldowns | قبل أي كود |
| D4 | Aim: 3-dir vs 8-dir؟ | 3 للـ Handgun، 8 للـ Melee | قبل أي كود |
| D5 | القصة؟ | Dark Fantasy (دم الإله السابع) أو Mythological (دم أنزو) | قبل الـ art |
| D6 | Joystick واحد أم اثنان؟ | واحد (كما خططت) | قبل الـ UI |
| D7 | Portrait أم Landscape؟ | Landscape (موصى به للـ action) | قبل الـ UI |
| D8 | iOS + Android + Steam؟ | ابدأ Android أسهل | قبل الإطلاق |
| D9 | Monetization؟ | Premium (one-time purchase) | قبل الإطلاق |
| D10 | localization strategy؟ | عربي + إنجليزي للبداية | قبل الإطلاق |

---

## 8. مراجع مفيدة

### 📚 كتب
- "Designing Games" — Tynan Sylvester (game feel)
- "The Art of Game Design" — Jesse Schell (lenses)
- "Level Up!" — Scott Rogers (practical design)
- "Blood, Sweat, and Pixels" — Jason Schreier (indie stories)

### 🎮 ألعاب يجب لعبها (تحليلياً)
- **Hollow Knight** (مرجع Metroidvania + Soulslike)
- **Nine Sols** (مرجع Parry)
- **Katana Zero** (مرجع Dash + time scale)
- **Dead Cells Mobile** (مرجع mobile port)
- **Tunic** (مرجع Souls-Lite)
- **Blasphemous** (مرجع Dark Fantasy)
- **Sekiro** (مرجع Posture system)

### 📺 قنوات يوتيوب
- **GMTK** (Game Maker's Toolkit): design analysis
- **Game Wisdom**: deep dives
- **Noclip**: documentaries
- **Sebastian Lague**: technical

### 🛠️ أدوات
- **Godot 4.7** (engine)
- **Aseprite** (pixel art)
- **LDtk** (level design)
- **ElevenLabs** (SFX، كما في plans)
- **FMOD** (audio middleware، اختياري)
- **Notion / Obsidian** (project management، كما تستخدم)

---

## 9. الخلاصة النهائية

### ما الذي يميّز مشروعك؟
1. **رؤية واضحة**: 4 شعور مستهدف (Skilled/Challenged/Fast/Intelligent) — نادر في indie.
2. **ابتكار الـ Joystick-one**: لو نجح، يصبح USP (Unique Selling Proposition) للعبة.
3. **اختيار إلهامات صحيحة**: Hollow Knight + Nine Sols = مراجع من الطراز الأول.
4. **استخدام Godot**: قرار تقني سليم (مجاني + open source + mobile-ready).
5. **Mobile-First**: سوق ينمو، فرصة كبيرة.

### ما الذي يجب الحذر منه؟
1. **Scope creep**: التزم بـ Region A للبداية.
2. **كثرة الأنظمة قبل prototype**: لا تكتب المزيد من الـ .md قبل ما تلعب.
3. **Burnout**: جدول واقعي (10-20h/أسبوع مستدام أفضل من 40h ثم استسلام).
4. **Perfectionism**: prototype قبيح يلعب = أفضل من design doc جميل لا يلعب.
5. **Mobile UX**: اختبر الـ controls على device حقيقي أسبوعياً.

### الكلمة الأخيرة
> **"The best game design document is the one that gets you to a playable prototype in the shortest time."**  
> — Anonymous indie dev

لديك **أساس ممتاز** (رؤية + إلهامات + بعض الأنظمة المفصّلة). تحتاج **اتخاذ قرارات شجاعة** (5 خيارات في D1-D10) ثم **بناء prototype** سريعاً.

الملفات الأربعة في هذا الـ Review تعطيك:
- `01-Systems-Analysis.md` → فهم عميق للأنظمة الموجودة.
- `02-Pitfalls-Solutions.md` → حلول هندسية للمخاطر المعلنة.
- `03-Empty-Systems-Foundations.md` → كود جاهز للـ 11 نظام فارغ.
- `04-Story-Concepts.md` → 5 قصص كاملة + توصية.
- `05-Master-Roadmap.md` (هذا الملف) → خطة تنفيذ.

**ابدأ بـ D1-D10، ثم انتقل لـ P0، ثم P1. لا تتوقف، لا تنحرف.**

---

> **ملاحظة أخيرة**: كل ما في هذه الملفات **قابل للتغيير**. خذ ما ينفعك، اترك ما لا ينفع. أنت الـ Game Director. أنا مستشار.

**حظاً موفقاً في Veinbreaker. 🩸**
