### Movement [[Game play/Systems/Controllers|Controllers]] Concept:
> **Note** 1: *The* **`JoyStick`** *is Built-in godot 4.7 feature which acts like Control button but trigger actions from Input Map*
> **Note 2**: The **`JoyStick`** Has 8 **Zones**:
> 	1) Green Zones (Left/Right): Simple forward Movement
> 	2) Red Zones (Up/down): Idle while Aiming (Up/down)
> 	3) Blue Zones (Up Left/Up Right): Slower movement while Aiming Up (For penetrable platforms or top attack spatially for bosses)
> 	4) Purple Zones (Down Left/Down Right): Slower movement while Aiming Down (For penetrable platforms or air attack)

| Input                           | Actions         |
| ------------------------------- | --------------- |
| Joystick left                   | Move Left       |
| Joystick right                  | Move right      |
| Joystick up                     | Aim Up          |
| Joystick Down                   | Aim Down        |
| Joystick (Up-left/Up-right)     | Aim Up & Move   |
| Joystick (Down-left/Down-right) | Aim Down & Move |

---
### Jump Mechanisms

| Sub-Mechanism    | Conditions                                                                                | Reset                                                       | access                  | Explanation                                                                                   |
| ---------------- | ----------------------------------------------------------------------------------------- | ----------------------------------------------------------- | ----------------------- | --------------------------------------------------------------------------------------------- |
| **Coyote Timer** | Was `is_on_floor()` &<br>in Move State &<br>`!timer.is_stopped()`                         | `is_on_floor()` +<br>`coyote_time = 0.12`                   | Default                 | The [[Player]] Can jump after leaving the edge for short amount of time                       |
| **Buffer Timer** | Was `!is_on_floor()`<br>&<br>in Fall State &<br>`!timer.is_stopped()`                     | `Jump Input` and `!is_on_floor()` +<br>`buffer_time = 0.16` | Default                 | the jump input will be stored in the air and trigger the next jump if the timer still running |
| **Jump Cut**     | `!is_on_floor()` &<br>`velocity.y < 0` &<br>**`Jump`** Released<br>                       | Allways                                                     | Default                 | The [[Player]] Can control how high can he jump                                               |
| **Wall Jump**    | `is_on_wall()` &<br>`can_wall_jump` &<br>Press **`Jump`** &<br>Is on Wall Slide State<br> | `is_on_floor()` \|\| <br>`is_on_wall()`                     | [[Skill Tree System]] | The [[Player]] Can Jump off the walls to overcome obstacles or to solve puzzles               |
| **Double Jump**  | `!is_on_floor()` &<br>`velocity.y > 0` &<br>`can_double_jump`                             | `is_on_floor()` \|\|<br>`is_on_wall()`                      | [[Skill Tree System]] | The [[Player]] can do another jump in fall state only one time                                |
| **Forced Fall**  | Press `Jump`+<br>`Joystick Down` simultaneously while [[Player]] In the Air               | Allways                                                     | Default                 | The [[Player]] will Fall Fast In The Jump State"                                              |

---
### Dash Mechanism

> **Note**: *Use `Input Buffer` to differentiate between types of dash*

| Dash Type       | Motion input                  | Conditions                                     | Activating States               | access                | Effects                                            |
| --------------- | ----------------------------- | ---------------------------------------------- | ------------------------------- | --------------------- | -------------------------------------------------- |
| **Normal Dash** | `Dash`                        | `can_dash` & <br>`special_cooldown_timer`      | Idle,<br>Move,<br>Jump,<br>Fall | Default               | حركة سريعة مستقيمة<br>شبه انعدام الجاذبية فالسماء  |
| **Roll Dash**   | `Dash` + <br>`Joystick Down`+ | `can_roll_dash` & <br>`special_cooldown_timer` | Idle<br>Move                    | [[Skill Tree System]] | يمر تحت الأعداء يلغي ضرر الاصطدام بالاعداء         |
| **Wall Dash**   | `Dash` + <br>`Move (L/R)`     | `can_wall_dash` & <br>`special_cooldown_timer` | WallSlide                       | [[Skill Tree System]] | يندفع من على الجدار<br>شبه انعدام الجاذبية فالسماء |
| **Dodge Dash**  | `Dash`**×2**                  | `can_dodge_dash` &<br>`special_cooldown_timer` | Idle,<br>Move                   | [[Skill Tree System]] | يتفادى الضربات<br>خطوة سريعة الى الخلف             |


---
### Wall Slide Mechanism (from [[Skill Tree System]])
- #### Conditions:
	- `can_wall_slide`
	- `is_on_wall()`
	- `L_raycast.is_colliding() || R_raycast.is_colliding()`
	- `!Bottom_raycast.is_colliding()`
	- `!is_on_floor()`
- Movement Effect:
	- Gravity Decreasing ratio

### Fall Mechanism:
- Max Fall Velocity
- if player Reached max fall speed the fall will considered big fall
	- Screen shack effect.
	- Heavy Landing Animations.
	- Like Hollow knight