## Attack Animations Rule:
![[Animations Action Phases.png]]

> [!TODO] **Effects:**
> - [ ] Slash trail Effect.
> - [ ] Knock forward In Every Slash attack when [[Player]] `is_on_floor()`
> - [ ] Dust Particles.
> - [ ] Screen Shake Effect.
> - [ ] time scale for heavy blows.
> - [ ] Sound Effects from Elevenlabs + Randomize the Pitch.
> - [ ] Identify the type of the object that was struck and make a sound that matches it.

---
## Attacks:

> [!TODO] Input Buffering
> - Add Input Buffering for the attacks



### 1) Simple Slash's Combo:
- **Description:** Three consecutive Regular attacks:
	1. First swing
	2. right swing
	3. Middle swing

> [!NOTE] Note
>- The Animations Will be handled later after the gameplay establishment (**This is just place holder**).

> [!TODO] Attack State Descriptions
> - The Attack state animations and movement will be different in each case:
> 	- [[Player]] `is_on_floor()`:
> 		- If `Joystick` **Not Dragged** the [[Player]] will stay in the place while swinging with little bit of knock-forward (movement/dragging) with each attack in the facing direction.
> 		- If `Joystick` **Dragged** same thing but with bigger (forward movement dragging) in the dragging direction 
> 	- [[Player]] `!is_on_floor()`:
> 		- In the fall/jump state: the [[Player]] will swing freely while moving on the x axis like nine sols

- **Input:** Press `Melee Attack`
- **Conditions:**
	- `can_attack`
	- `!combo_delay_timer.is_stopped()`
- **Properties:**
	- Input window `0.4s`.
	- The **combo delay time** and **damage** can be reduced from [[Skill Tree System]].
	- The player will be knocked forward for small distance with each attack. 
	- Can be mixed with [[Handguns]] shoots.

> [!INFO] **Upward Swing:**
>  - There is also ***Upward Swing*** and it is a default ability 
>  - **Input**: Press `Melee Attack` + `Joystick UP` simultaneously

> [!INFO] **Dow-ward Swing:**
>  - There is also ***Dow-ward Swing*** and it is a default ability 
>  - **Input**: Press `Melee Attack` + `Joystick Down` simultaneously
>  - Add A Upward Velocity for the player like hollowknight

### 2) Charged Attack (`Charged Slash` Skill)
- **Description:** Big Single Heavy Attack
- **Input:** Hold  `Melee Attack` for more than `1.5s` and release
- **Conditions:**
	- `can_charge_attack`
	- `charged_attack_cooldown_timer.is_stopped()`
	- Unlocked from [[Skill Tree System]].
- **Properties:**
	- while the [[Player]] still holding `Melee Attack` Button/key the [[Player]] still had the charged attack until it release it to preform the attack.
	- The player will be knocked forward for medium distance with each attack.

### 3) Dash Attack (`Swift Slash` skill):
- **Description:** Player Will Preform Slash while dashing
- **Input:** Press `Melee Attack` while Dashing
- **Conditions:**
	- `can_dash_attack`
	- [[Player]] is in dash state.
	- Unlocked from [[Skill Tree System]].
- **Properties:** 
	- It allows him to penetrate enemies without taking collision or attack damage.
	- Animations & Effects like Katana Zero Attack (Like)
### 4) Fall Stab (`Death From Above` skill): ==**(Contextual Attack)**==
- **Description:** Like **Death from Above Attack** From  **Farcry 3** (but in 2D) it Depend on the perfect timing then push him.
- **Input:** Press `Melee Attack` + `joystick Down` simultaneously In the Fall State
- **Conditions:**
	- `can_attack_on_fall`
	- [[Player]] is in the Fall State.
	- Special `RayCast2D` **is colliding** with Regular Enemy.
	- Unlocked from [[Skill Tree System]].
	- This attack has second phase, when the [[Player]] must strike a second time after a successful ***Fall Stab*** to finish the attack and inflict maximum damage. If the player doesn't attack a second time, the enemy will throw them off their back (causing the player to fall, become temporarily paralyzed, and leave them vulnerable). There are two ways to finish an attack: 
		- **Melee Finish**: The [[Player]] presses Melee Attack again to spin the sword.
		- **[[Handguns]] Finish**: The [[Player]] presses Handgun Attack to fire headshots.
- **Properties:**
	- Just on Regular Enemies
	- Knock Back little bit after Pushing the Enemy.

### 5) Back stab (`Back Stab` skill): ==**(Contextual Attack)**==
- **Description:** Like The **Back Stab** from **Dark souls** Stabs The Enemy then throw it.
- **Input:** Hold `Melee Attack` for `0.7s` while in the Stabbing Range
- **Conditions:**
	- `can_back_stab`
	- The enemy should not be aware of the [[Player]] (The [[Player]] not detected).
	- Unlocked from [[Skill Tree System]].
- **Properties:**
	- Just on Regular Enemies
	- Knock Back little bit after throwing the Enemy.

---
## Enemy Hit Reactions:
...