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

### 4) Fall Attack (`Death From Above` skill):
- **Description:** Like **Death from Above Attack** From  **Farcry 3** (but in 2D) it Depend on the perfect timing then push him.
- **Input:** Press `Melee Attack` + `joystick Down` simultaneously In the Fall State
- **Conditions:**
	- `can_attack_on_fall`
	- [[Player]] is in the Fall State.
	- Special `RayCast2D` **is colliding** with Regular Enemy.
	- Unlocked from [[Skill Tree System]].
- **Properties:**
	- Just on Regular Enemies
	- Knock Back little bit after Pushing the Enemy.

### 5) Back stab (`Back Stab` skill):
- **Description:** Like The **Back Stab** from **Dark souls** Stabs The Enemy then throw it.
- **Input:** Hold `Melee Attack` for `0.7s` while in the Stabbing Range
- **Conditions:**
	- `can_back_stab`
	- The enemy should not be aware of the [[Player]] (The [[Player]] not detected).
	- Unlocked from [[Skill Tree System]].
- **Properties:**
	- Just on Regular Enemies
	- Knock Back little bit after throwing the Enemy.

## Enemy Hit Reactions
...