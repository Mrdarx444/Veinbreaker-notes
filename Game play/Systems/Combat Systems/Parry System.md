
> [!Info] The Parry System Devised into 2 Parts:
> 1) The Block (The Actual Parry)
> 2) The Reward (After Parry)

## 1) The Parry:
- The [[Player]] can parry an attack from the Enemy by Pressing on The **Parry button** in the [[Game play/Systems/Controllers|Controllers]] HUD, This allows him to Completely negates the damage of an Enemy attack if executed in the right time window. paralyzing the enemy for a period, giving the player time to regain health or setting the stage to ***Counter Attack***.

> [!NOTE] Parry Stress Bar
> - To Prevent **Spamming** The parry should Add 2 mechanics:
> 	1)  **Parry Stress Bar:**
> 		- Bar that filled with stress points from each parried attack and slowly back to zero if there is no parried attack,  if  the stress bar is filled the player can't parry anymore until it resets (cooldown)
> 		- The Stress Points Can be converted to extra ammo for the [[Handguns]] so the player can reset the bar quickly but this will be unlocked from the [[Skill Tree System]]
> 		- if the stress bar filled the player can't parry for time and the stress bar will stay filled for a time and starts resetting by it's own if it didn't drained out with charged handgun attack.
> 	2) **Anti-spamming window:** 
> 		 - inspired by nine sols if the player clicked on the parry button multiple times in row the parry time window will decrease with each fake parry to prevent spamming

- Parry Conditions:
	- `can_parry`
	- [[Player]] `is_on_floor()`.
	- parry stress bar didn't fill up yet $Value < MaxValue$
	- `parry_cooldown_timer.is_stopped()`