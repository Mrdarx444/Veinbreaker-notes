### Joystick Mechanism:
- The **`JoyStick`** Controls The **Basic Movement** of the **[[Player]]** and The [[Aim system]] simultaneously.
- The **`JoyStick`** is in Bottom left Corner of the screen
#### Diagram:
![[JoyStick Chart.png]]

---
### Simple Controllers:
> **Note**: *The* **`JoyStick`** *is Built-in godot 4.7 feature which acts like Control button but trigger actions from Input Map*

| Input                           | Actions       |
| ------------------------------- | ------------- |
| Joystick left                   | Move Left     |
| Joystick right                  | Move right    |
| Joystick up                     | Aim Up        |
| Joystick Down                   | Aim Down      |
| Joystick (Up-left/Up-right)     | Aim Up & Move |
| Joystick (Down-left/Down-right) | Aim Down      |
#### HUD:
![[HUD Controllers V2.png]]

---
### Jump Mechanisms

| Sub-Mechanism    | Conditions                                                                                  | Reset                                                       | access                  | Explanation                                                                                   |
| ---------------- | ------------------------------------------------------------------------------------------- | ----------------------------------------------------------- | ----------------------- | --------------------------------------------------------------------------------------------- |
| **Coyote Timer** | Was `is_on_floor()` &<br>in Move State &<br>`!timer.is_stopped()`                           | `is_on_floor()` +<br>`coyote_time = 0.12`                   | Default                 | The [[Player]] Can jump after leaving the edge for short amount of time                       |
| **Buffer Timer** | Was `!is_on_floor()`<br>&<br>in Fall State &<br>`!timer.is_stopped()`                       | `Jump Input` and `!is_on_floor()` +<br>`buffer_time = 0.16` | Default                 | the jump input will be stored in the air and trigger the next jump if the timer still running |
| **Jump Cut**     | `!is_on_floor()` &<br>`velocity.y < 0` &<br>**`Jump`** Released<br>                         | `is_on_floor()`                                             | Default                 | The [[Player]] Can control how high can he jump                                               |
| **Wall Jump**    | `is_on_wall()` &<br>`can_wall_jump` &<br>**`Jump`** Clicked &<br>Is on Wall Slide State<br> | `is_on_floor()` \|\| <br>`is_on_wall()`                     | [[Ability Tree System]] | The [[Player]] Can Jump off the walls to overcome obstacles or to solve puzzles               |
| **Double Jump**  | `!is_on_floor()` &<br>`velocity.y > 0` &<br>`can_double_jump`                               | `is_on_floor()` \|\|<br>`is_on_wall()`                      | [[Ability Tree System]] | The player can do another jump in fall state only one time                                    |


---
### Dash Mechanism

> **Note**: *Use `Input Buffer` to differentiate between types of dash*

| Dash Type       | Motion input                                     | Conditions                                                           | Activating States               | access                  | Effects                                            |
| --------------- | ------------------------------------------------ | -------------------------------------------------------------------- | ------------------------------- | ----------------------- | -------------------------------------------------- |
| **Normal Dash** | `Dash`                                           | `can_dash` & <br>`stamina > x` &<br>`special_cooldown_timer`         | Idle,<br>Move,<br>Jump,<br>Fall | Default                 | حركة سريعة مستقيمة<br>شبه انعدام الجاذبية فالسماء  |
| **Slide Dash**  | `Dash` + <br>`Aim Down`+ <br>`Move (L/R)`        | `can_slide_dash` & <br>`stamina > x` &<br>`special_cooldown_timer`   | Idle<br>Move                    | [[Ability Tree System]] | يمر تحت الأعداء يلغي ضرر الاصطدام بالاعداء         |
| **Wall Dash**   | `Dash` + <br>`Move (L/R)`                        | `can_wall_dash` & <br>`stamina > x` &<br>`special_cooldown_timer`    | WallSlide                       | [[Ability Tree System]] | يندفع من على الجدار<br>شبه انعدام الجاذبية فالسماء |
| **Dodge Dash**  | `Dash`**×2** + <br>`Move (L/R)` <br>(In Reverse) | `can_dodge_dash` &<br>`stamina > x` &<br>`special_cooldown_timer`    | Idle,<br>Move                   | [[Ability Tree System]] | يتفادى الضربات                                     |
| **Shadow Dash** | `Dash`**×2** + `Move (L/R)`                      | `can_shadow_dash` & <br>`stamina > x`  &<br>`special_cooldown_timer` | Idle,<br>Move,<br>Jump,<br>Fall | [[Ability Tree System]] | يخترق الأعداء ويؤذيهم دون تلقي دمج الاصطدام        |

### Wall Slide Mechanism (from [[Ability Tree System]])
- #### Conditions:
	- `can_wall_slide`
	- `is_on_wall()`
	- `L_raycast.is_colliding() || R_raycast.is_colliding()`
	- `!Bottom_raycast.is_colliding()`
	- `!is_on_floor()`
- Movement Effect:
	- Gravity Decreasing ratio