# Noita magic numbers: what the settings change

This guide explains the 404 magic-number keys accepted by the inspected Noita executable, plus seven similarly named entries in the local vanilla XML that the engine does not accept. Each entry describes the change a modder can expect in the game, based on the engine code.

## Before changing values

For keys present in the local vanilla XML, the listed value is that file's shipped value. For engine keys absent from that file, it is the executable's initial value before a mod overrides it. The engine type comes from the registration handler. Boolean settings use `0` or `1`. There is no shared min/max range: an entry calls out limits and invalid values where the code establishes them; otherwise, change values gradually from the listed starting point.

The engine recognizes many keys that are not present in the vanilla XML, so a mod can add them as overrides. Those engine-only keys have their own section below. Some XML entries have no vanilla effect because the key is never read by a game system or because its exact spelling is not registered; those are marked plainly. Mods can still read registered values with `MagicNumbersGetValue(key)` and give them their own behavior.

## Gameplay, world, and physics

| Setting | What it changes, based on the engine | Shipped default | Engine type |
|---|---|---:|---|
| `DESIGN_AGGRO_INDICATOR` | **No vanilla effect.** The engine loads this value but never reads it, so changing it does not turn an enemy aggro indicator on or off. | `0` | boolean/byte |
| `DESIGN_CARD_SYMBOL_UNLOCKS` | When enabled, card creation stops using the discovered-card list to mark a card as identified. This changes whether cards appear identified in inventory; it does not unlock every symbol. | `0` | boolean/byte |
| `DESIGN_ITEMS_CAN_BE_EATEN` | Enables the munching interaction for eligible items: the game plays the munching effect, consumes some of the item’s resource, and produces the drinking or empty-potion feedback. | `0` | boolean/byte |
| `DESIGN_ITEMCHEST_DROPS_ACTIONS` | Allows item chests to create spell or wand-action drops. With `0`, that drop path is skipped; with `1`, it can run when the chest is opened. | `0` | boolean/byte |
| `DESIGN_WAND_SLOTS_ARE_CONSUMED` | Shows the wand slot’s stored count in the inventory display (`?` when the count is negative). This code only displays the count; it does not consume the slot. | `0` | boolean/byte |
| `DESIGN_ENEMY_HEALTH_DROPS` | Enables the enemy health-drop behavior. With `0`, the engine skips the health-drop branch; with `1`, eligible enemies can produce its health pickup. | `0` | boolean/byte |
| `DESIGN_CARDS_MUST_BE_IDENTIFIED` | Controls whether a card counts as identified: at `0`, every card is treated as identified; at `1`, only cards whose saved flag says “identified” count. This affects whether cards are hidden or shown by inventory filters that require identification. | `0` | boolean/byte |
| `DESIGN_DAILY_RANDOM_STARTING_ITEMS` | Enables loading the randomized starting-items entity during daily runs. | `1` | boolean/byte |
| `DESIGN_RANDOM_STARTING_ITEMS` | Enables the randomized starting-items entity for ordinary runs; the daily-run flag can enable the same path too. | `0` | boolean/byte |
| `DESIGN_RANDOMIZE_TEMPLE_CONTENTS` | **No vanilla effect.** This registered value is never read by the engine. | `0` | boolean/byte |
| `DESIGN_MATERIAL_INGESTION_STATUS_FX` | **No vanilla effect.** This registered value is never read by the engine. | `1` | boolean/byte |
| `DESIGN_DAY_CYCLE_SPEED` | Sets how quickly the day/night cycle advances. Higher values advance it faster; the game multiplies this value into each cycle update. | `0.0015` | float |
| `DESIGN_SPELL_VISUALIZER` | When enabled, the game records eligible spell identifiers and creates the matching visualizer tiles in the spell workshop. Turning it off skips both steps. | `1` | boolean/byte |
| `DESIGN_PLAYER_START_POS_X` | Horizontal world position used when placing the player at the start of a run. Random-start modes can replace it. | `227` | float |
| `DESIGN_PLAYER_START_POS_Y` | Vertical world position used when placing the player at the start of a run. Random-start modes can replace it. | `-85` | float |
| `DROP_LEVEL_1` | Sets the first endpoint of the score used to decide how many actions a chest drops. The engine eases from this value toward `DROP_LEVEL_9` using `(1 - clamp(chest progress / 13,000, 0, 1))⁴`, then multiplies by half the chest’s action-count factor. It converts that score to lower and upper integer bounds, rounds the bounds with a random choice, and picks the final count between them. Larger endpoint values move chests toward more action drops; this is not a percentage. | `3.78` | float |
| `DROP_LEVEL_2` | **No vanilla effect.** The engine loads this float but never reads it; it does not set a level-2 drop chance. | `2.0` | float |
| `DROP_LEVEL_3` | **No vanilla effect.** The engine loads this float but never reads it; it does not set a level-3 drop chance. | `0.9` | float |
| `DROP_LEVEL_4` | **No vanilla effect.** The engine loads this float but never reads it; it does not set a level-4 drop chance. | `0.6` | float |
| `DROP_LEVEL_5` | **No vanilla effect.** The engine loads this float but never reads it; it does not set a level-5 drop chance. | `0.4` | float |
| `DROP_LEVEL_6` | **No vanilla effect.** The engine loads this float but never reads it; it does not set a level-6 drop chance. | `0.3` | float |
| `DROP_LEVEL_7` | **No vanilla effect.** The engine loads this float but never reads it; it does not set a level-7 drop chance. | `0.27` | float |
| `DROP_LEVEL_8` | **No vanilla effect.** The engine loads this float but never reads it; it does not set a level-8 drop chance. | `0.26` | float |
| `DROP_LEVEL_9` | Sets the other endpoint of the score used to decide how many actions a chest drops. The engine eases from `DROP_LEVEL_1` toward this value using `(1 - clamp(chest progress / 13,000, 0, 1))⁴`, then multiplies by half the chest’s action-count factor. It converts that score to lower and upper integer bounds, rounds the bounds with a random choice, and picks the final count between them. Larger endpoint values move chests toward more action drops; this is not a percentage. | `0.25` | float |
| `ANIMALAI_FLYING_OFFSET_Y` | **No vanilla effect.** The engine loads this float but never reads it, so it does not move flying animals up or down. | `0.0` | float |
| `BIOME_APPARITION_CHANCE` | Controls how often a player ghost can appear when the bone requirement below is met: the engine draws from `1` through this value and spawns only on `1`. At `30`, that is a 1-in-30 roll. In deeper areas, the game divides the denominator by a random 2 or 3, making ghosts more common there. | `30` | integer |
| `APPARITION_MIN_BONES_REQUIRED` | Minimum number of stored bone records needed before the game can spawn a player ghost. At the default, the spawn roll is skipped until at least 35 records are available. | `35` | integer |
| `BIOME_MAP` | Chooses the Lua file that supplies the world’s biome map. Pointing it at a custom biome-map script changes the map the game uses to decide where biomes go. | `data/scripts/biome_map.lua` | string |
| `BIOME_PATH_FIND_HEIGHT_LIMIT` | Sets the greatest vertical height the biome-path search accepts. Raising it lets the world-generation search consider higher candidate routes. | `205` | integer |
| `GAMEEFFECT_GLOBAL_GORE_GORE_MULTIPLIER` | In the gore game-effect branch, scales the number of blood particles and the gore material used when a ragdoll dies. The default branch multiplies the gore state by 5. | `5` | integer |
| `GAMEEFFECT_CRITICAL_HIT_BOOST_CRIT_EXTRA_CHANCE` | Adds 10 critical-hit chance points for each active critical-hit-boost effect. Chance can exceed 100; above 100, the excess multiplies the critical-hit damage multiplier. | `10` | integer |
| `GAMEEFFECT_EXTRA_MONEY_TRICK_KILL_MULTIPLIER` | Chooses the trick-kill money drop script: `2` selects 2× money, with other recognized values selecting 4×, 8×, 16×, or 32×. Values outside those cases use the 64× script. | `2` | integer |
| `GAMEEFFECT_MOVEMENT_FASTER_SPEED_MULTIPLIER` | Multiplies the player’s movement speed while the faster-movement effect is active. At the default, movement is 1.5 times the base value. | `1.5` | float |
| `GAMEEFFECT_FIRE_MOVEMENT_SPEED_MULTIPLIER` | Multiplies movement speed fields while the fire-movement effect is active. At the default, those fields are 1.15 times their base values. | `1.15` | float |
| `GAMEEFFECT_MANA_REGENERATION_SPEED_MULTIPLIER` | Multiplies how much mana the player regains on each regeneration update, before the engine stops at maximum mana. | `3` | float |
| `DAMAGE_BLOOD_AMOUNT_MAX` | Upper end of the random amount used to create blood effects after damage. The chosen amount is then scaled by the other blood settings. | `40` | integer |
| `DAMAGE_BLOOD_AMOUNT_MIN` | Lower end of the random amount used to create blood effects after damage. The chosen amount is then scaled by the other blood settings. | `20` | integer |
| `RAGDOLL_BLOOD_MULTIPLIER` | Multiplies the blood amount calculated for a dying or damaged ragdoll before the engine creates its blood effects. | `2` | float |
| `EXPLOSION_FACTORY_FALLING_DIRT_FX_PROBABILITY` | Sets the chance for the explosion factory’s falling-dirt branch. The engine rolls 0–100 and runs the branch when the roll is below this value: `0` disables it and `100` succeeds on 100 of 101 rolls. | `0` | integer |
| `EXPLOSION_FACTORY_GORE_FX_BLOOD_STAIN_COLOR` | **No vanilla effect.** The game stores this 32-bit value but never reads it to color an explosion stain. | `4286055685` | 32-bit value |
| `EXPLOSION_FACTORY_GORE_FX_HOLE_PROBABILITY` | Sets the chance that an eligible explosion cell is cleared to make a hole. The engine rolls 0–100 for the cell and removes its existing grid object when the roll is below this value; at `30`, that removal happens on 30 of 101 rolls. `0` disables this removal roll; `100` succeeds on 100 of 101 rolls. | `30` | integer |
| `EXPLOSION_FACTORY_GORE_FX_PROBABLITY` | Sets how often the explosion factory creates its gore effects. The engine rolls 0–100 and runs that effect when the roll is below this value: `60` succeeds on 60 of 101 rolls. | `60` | integer |
| `EXPLOSION_FACTORY_STAIN_PERCENTAGE` | Expands the explosion’s secondary stain scan radius: with the default `0.4`, the scan can extend to 1.4 times the explosion’s base size. An explicit outer radius on the explosion overrides this value. | `0.4` | float |
| `GAMEPLAY_LIVES_ENABLED` | **No vanilla effect.** The engine loads this boolean but never reads it to enable or disable lives. | `1` | boolean/byte |
| `PHYSICS_FLOATER_FORCE_COEFF` | **Ignored by this vanilla build.** This exact key is not registered, so changing it cannot tune floater force. | `15.0` | not registered |
| `PHYSICS_FLOATER_FORCE_VEC_MAX` | **Ignored by this vanilla build.** This exact key is not registered, so changing it cannot cap floater force. | `5` | not registered |
| `PHYSICS_FLOATER_TORQUE_COEFF` | **Ignored by this vanilla build.** This exact key is not registered, so changing it cannot tune floater torque. | `50.0` | not registered |
| `PHYSICS_FLOATER_FORCE_BALANCING_COEFF` | **Ignored by this vanilla build.** This exact key is not registered, so changing it cannot tune floater force balancing. | `2.3` | not registered |
| `PHYSICS_FLOATER_TORQUE_BALANCING_COEFF` | **Ignored by this vanilla build.** This exact key is not registered, so changing it cannot tune floater torque balancing. | `0.2` | not registered |
| `PHYSICS_RAGDOLL_VERY_STIFF_JOINT_STIFFNESS` | Sets the maximum torque of the enabled motor on a special ragdoll hinge joint. At `2`, the motor can apply up to 2 engine torque units to resist rotation; higher values let it apply more turning force. | `2` | float |
| `NUM_ORBS_TOTAL` | Sets the total orb count reported by the engine’s orb-count query. This changes the count mods receive from that query; the engine does not use it to spawn or remove orbs. | `12` | integer |
| `PHYSICS_JOINT_MAX_FORCE_MULTIPLIER` | Sets the force threshold for breaking ordinary physics joints. The engine multiplies the two bodies’ combined mass by this value (and the joint’s own scale); if the measured reaction force exceeds the result, it destroys the joint and may create break debris. Raising it makes those joints harder to break. | `160` | float |
| `PLAYER_KICK_FORCE` | **No vanilla effect.** The engine loads this float but never reads it; kick force comes from the separate Verlet radius/force settings. | `50` | float |
| `PLAYER_KICK_VERLET_FORCE` | **No vanilla effect.** The kick code puts this force (or half of it in one kick path) into a Verlet request, but the physics system clears its request queue without applying it. | `4` | float |
| `PLAYER_KICK_VERLET_RADIUS` | **No vanilla effect.** This sets the radius the kick code searches when preparing Verlet requests, but the physics system clears those requests before they affect nearby cells. | `3` | float |
| `PLAYER_USE_NEW_JETPACK` | **No vanilla effect.** The engine loads this boolean but never reads it to select a jetpack implementation. | `0` | boolean/byte |
| `RAGDOLL_RAGDOLL_FX_EXPLOSION_ROTATION` | **Ignored by this vanilla build.** This exact XML key is not registered; a similarly named key does not make this spelling work. | `0.5` | not registered |
| `RAGDOLL_FIRE_DEATH_IGNITE_EVERY_N_PIXEL` | The fire-death path ignites a candidate pixel when its index is divisible by this value. Zero is invalid because the code uses it as a divisor. | `5` | integer |
| `RAGDOLL_OWN_VELOCITY_IMPULSE_MULTIPLIER` | Scales ragdoll X/Y velocity into the impulse applied to it. | `3` | float |
| `RANDOMIZE_LARGE_EXPLOSION_RAYS` | Adds a sine-shaped offset to the rays used to build large explosions. Turning it off removes that variation, making the ray layout more regular; turning it on changes the explosion’s sampled outline. | `1` | boolean/byte |
| `THROW_UI_TIMESTEP_COEFF` | Changes the spacing of the collision checks used to draw the throw-aim preview. Larger values move each preview check farther along the predicted path, making the preview coarser; it does not change the thrown object’s physics. | `4` | float |
| `VERLET_STAIN_DROP_CHANCE_DIV` | Sets the divisor in the random gate for creating a Verlet stain. Increasing it makes that gate pass less often; decreasing it makes it pass more often. | `200` | integer |
| `ITEM_SWITCH_ITEM_TWEEN_TIME_FRAMES` | Time divisor for an item-switch interpolation: a larger value spreads the same transition over more frames; a smaller value shortens it. | `4` | float |
| `PLAYER_FAST_MOVEMENT_PARTICLES_ALPHA` | Sets the baseline opacity used by the player’s fast-movement particles. Raising it makes the trail more visible; the animation still changes opacity over time. | `0.09` | float |
| `PLAYER_FAST_MOVEMENT_PARTICLES_ALPHA_CHANGE_SPD` | Sets how quickly the fast-movement particle trail changes opacity. Raising it makes the fade respond faster. | `0.2` | float |
| `PLAYER_FAST_MOVEMENT_EXPLOSION_TIME_MOVED_FAST_MIN` | Minimum fast-movement counter value before the game checks whether to create the fast-movement explosion effect. Raising it delays when the effect can start. | `60` | integer |
| `PLAYER_FAST_MOVEMENT_EXPLOSION_TIME_MOVED_FAST_MAX` | Elapsed-counter threshold selecting the large rather than normal fast-movement explosion effect. | `150` | integer |
| `PLAYER_FAST_MOVEMENT_EXPLOSION_COOLDOWN` | Sets how long the fast-movement explosion effect waits before it can trigger again. Raising it spaces repeated effects farther apart. | `90` | integer |
| `PLAYER_FAST_MOVEMENT_EXPLOSION_GROUND_PENETRATE_LENGTH` | Sets the length of the second downward ground probe used to place the fast-movement explosion effect. Raising it lets that probe reach farther into the ground. | `12` | float |
| `PLAYER_FAST_MOVEMENT_EXPLOSION_GROUND_RAY_LENGTH` | Sets the length of the first ground probe used by the fast-movement effect. Raising it lets the effect search farther below the player. | `30` | float |
| `PATHFINDING_DISTANCE_FIELD_ENABLED` | Enables distance-field guidance used by navigation. With `0`, the engine skips building and updating that field, so creatures that rely on it lose this route guidance. | `1` | boolean/byte |

## Camera and controls

| Setting | What it changes, based on the engine | Shipped default | Engine type |
|---|---|---:|---|
| `CAMERA_NO_MOVE_BUFFER_NEAR_PLAYER` | Sets the inner edge of the mouse camera’s tracking ramp. Offsets inside it contribute no camera shift; offsets beyond it begin to pull the view toward the cursor. | `0` | float |
| `CAMERA_NO_MOVE_BUFFER_NEAR_VIEWPORT_EDGE` | Sets the outer edge of the mouse camera’s tracking ramp. The cursor offset is scaled from zero at the near-player buffer up to full influence at this edge. | `0.3` | float |
| `CAMERA_MOUSE_INTERPOLATION_SPEED` | Sets the smoothing rate for camera movement toward the mouse-directed target. Higher values make the camera close the gap more quickly. | `0.25` | float |
| `CAMERA_POSITION_INTERPOLATION_SPEED` | Scales the per-update camera position error; larger values make the traced update close the gap faster. | `3.52` | float |
| `CAMERA_DISTANCE_INTERPOLATION_SPEED` | Controls how quickly the camera reaches its target distance from the player. Higher values close the zoom/distance gap faster. | `1.16` | float |
| `CAMERA_GAMEPAD_INTERPOLATION_SPEED` | Controls how quickly the gamepad-directed camera reaches its target offset. Higher values move the view toward the stick-directed target faster. | `1.75` | float |
| `CAMERA_GAMEPAD_MAX_DISTANCE` | Sets how far the gamepad can push the camera target from the player. The engine multiplies the stick direction by this distance, then eases the camera toward that target. Larger values let the view move farther off-center. | `55` | float |
| `CAMERA_RECOIL_AMOUNT` | Scales the recoil offset applied to camera coordinates. | `1` | float |
| `CAMERA_RECOIL_ATTACK_SPEED` | Controls the per-update approach toward the camera recoil target. | `0.15` | float |
| `CAMERA_RECOIL_RELEASE_SPEED` | Controls how quickly the stored camera recoil decays after the attack. | `15` | float |
| `CAMERA_ERROR_CORRECTION_AMOUNT_X` | Scales the final horizontal camera correction. Raising it shifts the view farther to compensate for horizontal error. | `1.1` | float |
| `CAMERA_ERROR_CORRECTION_AMOUNT_Y` | Scales the final vertical camera correction. Raising it shifts the view farther to compensate for vertical error. | `1` | float |
| `CAMERA_ERROR_CORRECTION_SPD_UP_X` | Sets the per-update correction factor for horizontal camera error in the up-direction branch. Larger factors move the view toward the corrected position faster. | `0.935` | float |
| `CAMERA_ERROR_CORRECTION_SPD_UP_Y` | Sets the per-update correction factor for vertical camera error in the up-direction branch. Larger factors move the view toward the corrected position faster. | `0.935` | float |
| `CAMERA_ERROR_CORRECTION_SPD_DOWN_X` | Sets the per-update correction factor for horizontal camera error in the down-direction branch. Larger factors move the view toward the corrected position faster. | `0.935` | float |
| `CAMERA_ERROR_CORRECTION_SPD_DOWN_Y` | Sets the per-update correction factor for vertical camera error in the down-direction branch. Larger factors move the view toward the corrected position faster. | `0.935` | float |
| `CAMERA_ERROR_SMOOTHING_THRESHOLD_X` | Sets the horizontal camera-error threshold that selects the smoothing/correction branch. Raising it lets a larger error pass before that branch changes. | `10` | float |
| `CAMERA_ERROR_SMOOTHING_THRESHOLD_Y` | **Ignored by this vanilla build.** This exact key is not registered. The engine has separate Y maximum and minimum thresholds instead. | `10` | not registered |
| `CAMERA_FAST_SWITCH_SPEED` | Controls how quickly the camera catches up during a fast camera switch. Higher values make the view move to the new target faster. | `1` | float |
| `MULTIPLAYER_CAMERA_SMOOTHING` | Sets the fraction of the remaining multiplayer camera offset corrected each update, while the offset is inside the distance cutoff. At `0.25`, the camera closes one quarter of the remaining gap per update. | `0.25` | float |
| `MULTIPLAYER_CAMERA_MAX_SMOOTH_DISTANCE` | Sets the farthest player offset that uses the multiplayer camera’s smoothing calculation. Beyond this distance, that smoothing rule no longer applies. | `512` | float |
| `DOUBLE_CLICK_MAX_SPAN_SECONDS` | Sets the maximum time between clicks for the double-click action. The engine also enforces a calculated minimum window; raising this value lets slower clicks count as a double-click. | `0.35` | float |
| `GAMEPAD_SHOT_VIBRATION_FORCE_COEFF` | Controls how strongly hit force increases controller rumble. The resulting rumble is at least the minimum below and never exceeds full strength (`1`). | `0.015` | float |
| `GAMEPAD_SHOT_VIBRATION_FORCE_MIN` | Sets the minimum controller-rumble strength for shot feedback. Stronger hits can raise it, up to full strength (`1`). | `0.2` | float |
| `GAMEPAD_SHOT_VIBRATION_TIME_COEFF` | Scales hit force when calculating how long the controller rumbles after a shot. | `0.04` | float |

## HUD, menus, and visual feedback

| Setting | What it changes, based on the engine | Shipped default | Engine type |
|---|---|---:|---|
| `TELEPORT_FLASH_COLOR_R` | Sets the red channel of the additive screen overlay during teleport flashes. | `0` | float |
| `TELEPORT_FLASH_COLOR_G` | Sets the green channel of the additive screen overlay during teleport flashes. | `0` | float |
| `TELEPORT_FLASH_COLOR_B` | Sets the blue channel of the additive screen overlay during teleport flashes. | `0` | float |
| `CREDITS_SCROLL_END_OFFSET_EXTRA` | Adds extra distance to the measured end position of the credits scroll. | `85` | float |
| `CREDITS_SCROLL_SPEED` | Scales the upward movement of the credits during normal scrolling. Higher values make the credits move faster. | `25` | float |
| `CREDITS_SCROLL_SKIP_SPEED_MULTIPLIER` | Replaces the normal scroll multiplier while the credits are being skipped. | `15` | float |
| `REPORT_DAMAGE_FONT` | **No vanilla effect.** The game loads and later frees this path, but no damage-report code uses it to choose a font. | `data/fonts/font_damage.xml` | string |
| `REPORT_DAMAGE_SCALE` | Sets the damage-report text scale. The engine divides it by the font’s line height before drawing, so doubling this value doubles the requested text size. | `2` | float |
| `REPORT_DAMAGE_TYPE` | **No vanilla effect.** The engine loads this boolean but never reads it to change the damage report. | `0` | boolean/byte |
| `INVENTORY_ICON_SIZE` | Sets the inventory item icon’s base size in UI pixels, which the UI also uses for its layout space. Higher values make icons larger and spread the inventory layout. | `20` | integer |
| `INVENTORY_STASH_X` | **No vanilla effect.** The engine loads this float but never reads it to position the inventory stash. | `370` | float |
| `INVENTORY_STASH_Y` | **No vanilla effect.** The engine loads this float but never reads it to position the inventory stash. | `80` | float |
| `INVENTORY_DEBUG_X` | Horizontal position of the debug inventory window. | `165` | float |
| `INVENTORY_DEBUG_Y` | Vertical position of the debug inventory window. | `45` | float |
| `DEBUG_UI_ALL_THE_CARDS_HEIGHT` | Sets the height of the debug “all cards” grid. | `20` | integer |
| `DEBUG_UI_ALL_THE_CARDS_WIDTH` | Sets the width of the debug “all cards” grid. | `24` | integer |
| `UI_BARS_POS_X` | Horizontal origin of the HUD stat bars. | `20` | float |
| `UI_BARS_POS_Y` | Vertical origin of the HUD stat bars; it also shifts for different active-player slots. | `20` | float |
| `UI_BARS2_OFFSET_X` | Horizontal offset for the second HUD stat-bar group. | `-40` | float |
| `UI_FULL_INVENTORY_OFFSET_X` | Horizontal offset used to position the full inventory/action-storage display. | `170` | float |
| `UI_PAUSE_MENU_LAYOUT_TOP_EDGE_PERCENTAGE` | Sets the pause-menu vertical placement from a screen-height calculation. One state uses a fixed 20-pixel placement instead. | `10` | integer |
| `UI_GAME_OVER_MENU_LAYOUT_TOP_EDGE_PERCENTAGE` | Sets the game-over menu vertical placement from a screen-height calculation. | `19` | integer |
| `UI_GAMEOVER_SCREEN_BOX_FROM_BOTTOM_PERCENT` | **No vanilla effect.** This value is loaded but never read by the game-over screen. | `12` | float |
| `UI_GAMEOVER_SCREEN_BOX_FROM_SIDE_PERCENT` | **No vanilla effect.** This value is loaded but never read by the game-over screen. | `32` | float |
| `UI_GAMEOVER_SCREEN_BOX_FROM_TOP_PERCENT` | **No vanilla effect.** This value is loaded but never read by the game-over screen. | `40` | float |
| `UI_STAT_BAR_EXTRA_SPACING` | Adds spacing between HUD stat-bar rows/elements. | `2` | float |
| `UI_STAT_BAR_ICON_OFFSET_Y` | Vertical placement offset for HUD stat-bar icons. | `-1` | float |
| `UI_STAT_BAR_TEXT_OFFSET_X` | Horizontal placement offset for HUD stat-bar text. | `10` | float |
| `UI_STAT_BAR_TEXT_OFFSET_Y` | Vertical placement offset for HUD stat-bar text. | `0` | float |
| `UI_PLAYER_FULL_STATS_POS_X` | **No vanilla effect.** This value is loaded but never read to position the full-stats panel horizontally. | `530` | float |
| `UI_PLAYER_FULL_STATS_POS_Y` | Vertical origin of the full player-stat panel. | `70` | float |
| `UI_PLAYER_FULL_STATS_COLUMN2_OFFSET_X` | **No vanilla effect.** This value is loaded but never read to position the second stats column. | `10` | float |
| `UI_PLAYER_FULL_STATS_COLUMN3_OFFSET_X` | **No vanilla effect.** This value is loaded but never read to position the third stats column. | `45` | float |
| `UI_IMPORTANT_MESSAGE_TITLE_SCALE` | Text scale passed to the important-message title draw path. | `1` | float |
| `UI_WOBBLE_AMOUNT_DEGREES` | Scales the amplitude of the UI wobble after converting the result to radians. | `3` | float |
| `UI_WOBBLE_SPEED` | Scales elapsed time before the sine used for UI wobble. | `10` | float |
| `UI_SCALE_IN_SPEED` | Scales the remaining size-to-one error in a UI scale-in transition. | `0.2` | float |
| `UI_LOW_HP_THRESHOLD` | Sets the health amount below which the low-health warning begins. It is measured in HP, not as a percentage. | `1.3333` | float |
| `UI_LOW_HP_WARNING_FLASH_FREQUENCY` | Sets how quickly the low-health warning pulses. Higher values make it flash faster. | `8` | float |
| `UI_DAMAGE_INDICATOR_RANDOM_OFFSET` | Sets the size of the random position offset applied to damage indicators. At the default `0`, the engine adds no random displacement; larger values spread indicators farther from their base position. | `0` | float |
| `UI_ITEM_STAND_OVER_INFO_BOX_OFFSET_X` | Horizontal offset added to the item hover-info box position. | `50` | float |
| `UI_ITEM_STAND_OVER_INFO_BOX_OFFSET_Y` | Vertical offset added to the item hover-info box position. | `-190` | float |
| `UI_LOCALIZE_RECORD_TEXT` | Shows the record image on the game-over screen when enabled. Despite its name, this flag does not localize text. | `1` | boolean/byte |
| `UI_HEALTHBAR_Y_SPACING` | Vertical spacing subtracted as the engine places successive player health bars. | `24` | float |
| `MAIN_MENU_BG_OFFSET_X` | Horizontal placement of the main-menu background. | `9250` | float |
| `MAIN_MENU_BG_OFFSET_Y` | Starting vertical position of the main-menu background tween. | `2250` | float |
| `MAIN_MENU_BG_OFFSET_Y_END` | Ending vertical position of the main-menu background tween. | `2320` | float |
| `MAIN_MENU_BG_TWEEN_SPEED` | Approach factor for the main-menu background tween. | `0.12` | float |

## Sound and music

| Setting | What it changes, based on the engine | Shipped default | Engine type |
|---|---|---:|---|
| `AUDIO_GAMEEFFECT_FIRE_WEIGHT` | **No vanilla effect.** The engine loads this float but never reads it to mix fire sounds. | `0.1` | float |
| `AUDIO_FIRE_WEIGHT` | Sets the weight of fire when the engine mixes material sounds. A larger value reduces fire’s share of that mix, so fire has less influence on the sound selected for a material interaction. | `1500` | float |
| `AUDIO_MAGICAL_MATERIAL_WEIGHT` | **No vanilla audio effect.** The game stores this value in the magical-material descriptor, but no audio event or mixer reads that weight. | `1` | float |
| `AUDIO_GAME_START_FADE_FRAME` | **No vanilla effect.** The engine loads this float but never reads it to time the game-start fade. | `10` | float |
| `AUDIO_GAME_START_FADE_SPEED` | Controls how quickly the relevant audio gain moves toward full strength during the game-start/state transition. Higher values reach full gain faster. | `1.2` | float |
| `AUDIO_MUSIC_VOLUME_DEFAULT` | Default supplied to the music-volume setting and used when migrating an old config. The settings slider shown in engine code runs from 0 to 1; this value is a default, not a forced runtime volume. | `0.8` | float |
| `AUDIO_MUSIC_QUIET_FADE_DOWN_SPEED` | Controls how quickly music fades toward silence during the quieting branch. Higher values make that fade faster. | `0.25` | float |
| `AUDIO_MUSIC_NORMAL_FADE_DOWN_SPEED` | Step rate used when the normal music mix fades down. | `0.04` | float |
| `AUDIO_MUSIC_ACTION_FADE_DOWN_SPEED` | Step rate used when the action music mix fades down. | `0.12` | float |
| `AUDIO_MUSIC_NORMAL_FADE_UP_SPEED` | Step rate used when the normal music mix fades up. | `0.075` | float |
| `AUDIO_MUSIC_ACTION_FADE_UP_SPEED` | Step rate used when the action music mix fades up. | `0.1` | float |
| `AUDIO_MUSIC_LOW_ENERGY_TRIGGER_COOLDOWN_SECONDS` | Adds a cooldown in game-time seconds before another low-energy music trigger is eligible. | `20` | 64-bit number |
| `AUDIO_MUSIC_FORCED_QUIETNESS_DURATION_SECONDS` | Duration of the forced-quietness interval; the engine converts the seconds value to frames by multiplying by 60. | `45` | integer |
| `AUDIO_MUSIC_FORCED_QUIETNESS_TRIGGERS_AFTER_SECONDS` | Delay before forced quietness can trigger; the engine converts the seconds value to frames by multiplying by 60. | `150` | integer |
| `AUDIO_COLLISION_SIZE_MULTIPLIER` | Sets how much object size contributes to the intensity sent with collision sounds. Larger impacts from bigger objects send a stronger intensity value to the sound event. | `0.15` | float |
| `AUDIO_COLLISION_SPEED_MULTIPLIER` | Sets how much impact speed contributes to the intensity sent with collision sounds. Faster impacts send a stronger intensity value to the sound event. | `0.01` | float |
| `AUDIO_COLLISION_COOLDOWN_SECONDS` | Minimum delay before one collision source can play another collision sound. Lower values allow repeated impacts to make sounds more often. | `0.20` | float |
| `AUDIO_COLLISION_STATIC_WALL_INTENSITY` | Sets the intensity input for the static-wall collision-sound branch. The input is normalized by the size multiplier. | `0.2` | float |
| `AUDIO_COLLISION_STATIC_WALL_SPEED_MULTIPLIER` | Scales the speed input for the static-wall collision-sound branch. | `0.5` | float |
| `AUDIO_PHYSICS_BREAK_MASS_MULTIPLIER` | Changes the intensity sent to wood-break and chain-break sounds. Some collision paths square the value; another uses it directly, and very low intensity is suppressed. | `5` | float |
| `AUDIO_CHARACTER_LIQUID_SPLASH_INTENSITY_MIN` | Minimum input to the liquid-splash sound-intensity formula. | `0.00` | float |
| `AUDIO_CHARACTER_LIQUID_SPLASH_INTENSITY_MAX` | Adds the upper-end intensity term in the character splash calculation. The engine combines it with speed and the minimum below; this value is not a hard ceiling on the final sound-event intensity. | `0.63` | float |
| `AUDIO_CHARACTER_LIQUID_SPLASH_INTENSITY_SPEED_DIV` | Divisor that scales squared character speed in the liquid-splash sound-intensity formula. Zero would divide by zero. | `250` | float |
| `AUDIO_EXPLOSION_NO_SOUND_BELOW_RADIUS` | Explosions smaller than this radius skip the traced explosion-sound selection path. | `2.05` | float |
| `AUDIO_EXPLOSION_SMALL_SOUND_MAX_RADIUS` | After the no-sound threshold, radii up to this boundary select the small explosion sound; larger radii select the regular explosion sound. | `7` | float |
| `AUDIO_AMBIENCE_ALTITUDE_SCALE` | Divides altitude before it is normalized and sent to the ambience event. Zero is invalid because the code divides by it. | `1000` | float |
| `AUDIO_CREDITS_TRACK_NAME` | Selects the music-bank event used for the credits track. | `music/credits/03` | string |

## Rendering and engine behavior

| Setting | What it changes, based on the engine | Shipped default | Engine type |
|---|---|---:|---|
| `INTRO_WEATHER_FOG` | Sets the fog amount that the intro weather transition blends toward. Raising it makes the intro scene foggier. | `0.0` | float |
| `INTRO_WEATHER_RAIN` | Sets the rain amount that the intro weather transition blends toward. Raising it makes the intro scene rainier. | `0.0` | float |
| `DEBUG_LOG_INSTANT_FLUSH` | Makes the logger flush each message to its log file immediately instead of waiting for a later buffered write. | `0` | boolean/byte |
| `DRAW_PARALLAX_BACKGROUND` | Turns the parallax-background drawing branch on or off. | `1` | boolean/byte |
| `DRAW_PARALLAX_BACKGROUND_BEFORE_DEPTH` | Depth cutoff compared by the parallax-background draw path. Changing it changes which scene depths take that drawing branch. | `512` | float |
| `GRID_RENDER_BORDER` | Adds a two-cell margin on every side of the grid area drawn around the camera view. Increase it to include more off-screen tiles at the edge. | `2` | integer |
| `GRID_RENDER_TILE_SIZE` | **No vanilla effect.** The engine loads this integer but never reads it; tile-size calculations use fixed values in the executable. | `32` | integer |
| `RENDER_FIRE_GRAVITY` | Sets the vertical acceleration applied to fire particles. Increasing its magnitude makes particles curve more strongly during their motion; changing its sign reverses the acceleration direction. | `-160` | float |
| `RENDER_FIRE_LIFETIME_MAX` | Upper bound, in particle-update ticks, for the randomly chosen lifetime of fire particles. | `30` | integer |
| `RENDER_FIRE_LIFETIME_MIN` | Lower bound, in particle-update ticks, for the randomly chosen lifetime of fire particles. | `12` | integer |
| `RENDER_FIRE_HI_FORCE` | Scales the motion of the detailed fire particles. Higher values move them farther during each update. | `1.5` | float |
| `RENDER_FIRE_HI_SCALE` | Scales the detailed fire-particle animation’s spatial offsets. | `0.15` | float |
| `RENDER_FIRE_HI_TIME` | Sets the time scale used by the detailed fire-particle animation. Higher values advance its angle/phase faster. | `0.25` | float |
| `RENDER_FIRE_LO_FORCE` | **No vanilla effect.** The engine loads this float but never reads it; it does not tune the low-detail fire particles. | `0` | float |
| `RENDER_FIRE_LO_SCALE` | **No vanilla effect.** The engine loads this float but never reads it; it does not scale the low-detail fire particles. | `0.2` | float |
| `RENDER_FIRE_LO_TIME` | **No vanilla effect.** The engine loads this float but never reads it; it does not change low-detail fire animation timing. | `0.2` | float |
| `RENDER_FIRE_GLOW_ALPHA` | Sets the opacity of the soft glow layer drawn around fire particles. Higher values make the glow more visible. | `0.04` | float |
| `RENDER_FIRE_SHARP_ALPHA` | Sets the opacity of the sharp, bright fire-particle layer. Higher values make that layer more visible. | `1.0` | float |
| `RENDER_POTION_PARTICLE_MAX_COLOR_COMPONENT` | Caps the brightest channel returned by `GameGetPotionColorUint`; when a potion is brighter, all three color channels are scaled down together to fit this cap. | `0.65` | float |
| `RENDER_PARALLAX_BACKGROUND_SHADER_GRADIENT` | Selects whether the parallax-background shader receives the gradient value from the weather state. Turning it off uses the non-gradient shader value. | `1` | boolean/byte |
| `USE_CUSTOM_THREADPOOL` | Selects how grid entities are processed. With `0`, the engine handles them one at a time on the current thread; with `1`, it submits work to the worker scheduler, which can change frame-time behavior on large updates. | `1` | boolean/byte |
| `RENDER_SKYLIGHT_MAX_REDUCTION_AMOUNT` | **No vanilla effect.** The engine loads this float but never reads it to change skylight. | `96.0` | float |
| `RENDER_SKYLIGHT_ABOVE_WEIGHT` | **No vanilla effect.** The engine loads this float but never reads it to weight skylight from above. | `1.0` | float |
| `RENDER_SKYLIGHT_SIDES_WEIGHT` | **No vanilla effect.** The engine loads this float but never reads it to weight skylight from the sides. | `0.75` | float |
| `RENDER_SKYLIGHT_TOTAL_WEIGHT` | **No vanilla effect.** The engine loads this float but never reads it to set total skylight. | `0.9` | float |
| `STREAMING_CHUNK_TARGET` | Sets the target amount of chunk-streaming work per update. The engine also uses two and three times this value as backlog thresholds, so raising it allows more streaming work before those thresholds are reached. | `12` | integer |
| `VIRTUAL_RESOLUTION_OFFSET_X` | Added to transformed horizontal coordinates in the virtual-resolution mapping. | `-1` | float |
| `VIRTUAL_RESOLUTION_OFFSET_Y` | Added to transformed vertical coordinates in the virtual-resolution mapping. | `-1` | float |
| `VIRTUAL_RESOLUTION_X` | Logical virtual-canvas width used in coordinate transforms and grid/texture dimensions. | `427` | integer |
| `VIRTUAL_RESOLUTION_Y` | Logical virtual-canvas height used in coordinate transforms and grid/texture dimensions. | `242` | integer |

## Engine settings accepted by the executable but absent from the local XML

These keys are registered by the inspected game build even though they are not in the local `magic_numbers.xml`. Mods can add them to their own magic-number files. Their defaults below are the executable's initial values before a mod override. `empty (null)` means the engine starts with no string value.

### Additional gameplay, world, and physics settings

| Setting | What changes in the game or its tools | Engine initial value | Engine type |
|---|---|---:|---|
| `BIOME_PATH_FIND_WORLD_POS_MAX_X` | Right edge of the horizontal world-position strip cleared before start-route pathfinding; min/max are world-position units and coordinates are converted to path-grid cells by dividing by 10 (008771a0). | `223` | integer |
| `BIOME_PATH_FIND_WORLD_POS_MIN_X` | Left edge of the horizontal world-position strip cleared before start-route pathfinding (008771a0). | `159` | integer |
| `BIOME_RANDOM_BLOCK_CHANCE` | During biome start-path preparation, sets the probability that an eligible special-color region in the temporary route map is rewritten to white instead of opaque black. The choice flood-fills the region’s connected cells; it does not directly change terrain or materials (00876ce0, 00876aa0, 008771a0). | `0.5` | float |
| `BIOME_USE_BIG_WANG` | Enables the Big-Wang world-generation block that builds the biome-region grid and assigns biome Wang/extra layers; disabling skips that generation path (0087a900). | `1` | boolean/byte |
| `BOX2D_FREEZE_STUCK_BODIES` | Enables the stuck-worker recovery watchdog: after its wait expires, it cancels the highest-priority outstanding grid item and marks it canceled (00718d50, 00717aa0). | `1` | boolean/byte |
| `BOX2D_THREAD_MAX_WAIT_IN_MS` | Timeout in milliseconds before the Box2D/grid stuck-worker recovery watchdog runs (00718d50). | `12` | float |
| `BUGFIX_NEVER_DEFAULT_SERIALIZE_PLAYER` | Clears the loaded player record’s default-serialization byte before insertion, preventing the player from being serialized as a default object (006b6200). | `1` | boolean/byte |
| `CELLFACTORY_CELLDATA_MAX_COUNT` | Maximum number of CellData/material slots retained; excess entries are removed and missing entries are allocated up to this capacity (0089c7d0). | `1024` | integer |
| `DAMAGE_BLOOD_SPRAY_CHANCE` | Chance threshold, compared against a 1–100 random roll, for taking the blood-spray death/effect branch (00abb7d0). | `20` | integer |
| `DAMAGE_CRITICAL_HIT_MULTIPLIER` | Multiplies damage on the critical-hit path (00c0d290). | `5` | float |
| `DAMAGE_FIRE_DAMAGE_MAX_HP_MIN_BOUND` | Lower bound on fire damage when max-HP-based fire damage is enabled (00ab7a70, 00b015d0). | `0.8` | float |
| `DESIGN_ALLOW_FULL_INVENTORY_SPELLS_DRAG` | Allows dragging a spell icon out of a full wand/inventory without taking the normal close/reset path (00b5ee40, 00b724c0, 00b7d8d0). | `1` | boolean/byte |
| `DESIGN_ALLOW_INVENTORY_CLOSING_AND_DRAGGING_GLITCH` | Allows the inventory to be opened and closed in the same frame while a drag is active (00b7d8d0); default off blocks that glitch. | `0` | boolean/byte |
| `DESIGN_BLOOD_RESTORES_HP` | Allows blood-material ingestion to restore HP by consumed amount times the material coefficient, capped at max HP; otherwise this needs status effect ID 0x2c (00b44970). | `0` | boolean/byte |
| `DESIGN_ENEMY_2X_MONEY_DROPS` | On eligible enemy deaths, selects a money-drop entity instead of the health-drop path; New Game Plus count chooses the 2x/4x/8x/16x/32x/64x variant (00abb7d0). | `1` | boolean/byte |
| `DESIGN_FIRE_DAMAGE_BASED_ON_MAX_HP` | Switches fire damage to a max-health-based calculation, with DAMAGE_FIRE_DAMAGE_MAX_HP_MIN_BOUND as its lower bound (00ab7a70). | `1` | boolean/byte |
| `DESIGN_NEW_GAME_PLUS_ATTACK_SPEED` | Scale applied to spawned enemies’ attack timer in New Game Plus (00a58bc0); 0.5 halves that timer. | `0.5` | float |
| `DESIGN_NEW_GAME_PLUS_HP_SCALE_MAX` | Upper clamp for the New Game Plus enemy health scale, interpolated by spawn Y/depth (00a58bc0). | `100` | float |
| `DESIGN_NEW_GAME_PLUS_HP_SCALE_MIN` | Lower clamp for the New Game Plus enemy health scale, interpolated by spawn Y/depth (00a58bc0). | `3` | float |
| `DESIGN_PLAYER_ALWAYS_TELEPORT_TO_GROUND` | Forces the fallback ground-search/teleport path when the initial player placement is blocked, even if the per-player request flag is absent (006b26f0). | `0` | boolean/byte |
| `DESIGN_PLAYER_PHYSICS_KILLS_DONT_TRICK_KILL` | Copies the physics hit’s killer attribution into the physics-damage event before applying damage, so physics kills credit the responsible attacker (00ab6d00). | `0` | boolean/byte |
| `DESIGN_PLAYER_PICKUP_ENABLED` | Enables the player pickup handler; when false it returns before processing pickup input (00be6100). | `0` | boolean/byte |
| `DESIGN_PLAYER_START_RAYCAST_COARSE_TO_GROUND` | Uses a coarse ground query to replace the configured player start Y with the ground position minus a small offset (006b6200). | `0` | boolean/byte |
| `DESIGN_PLAYER_START_TELEPORT_TO_GROUND` | Sets the player-start teleport-to-ground state flag during startup (006b6200); default on enables the ground placement step. | `1` | boolean/byte |
| `DESIGN_POLYMORPH_CONTROLS_ENABLED` | Keeps player control, stats, and callback ownership attached to the new body during polymorph; disabled path skips that transfer (006b6d60). | `1` | boolean/byte |
| `DESIGN_POLYMORPH_PLAYER_POLYMORPH_ENABLED` | When disabled, prevents player-targeting polymorph effects (IDs 7, 8, 0x59, 0x62) from applying to the player (00b041e0). | `1` | boolean/byte |
| `DESIGN_RELOAD_ALL_THE_TIME` | When enabled, the wand update skips decrementing its positive `mReloadFramesLeft` counter, so the reload countdown does not advance through that update path (00b4f8c0; field mapping 00604d40). | `1` | boolean/byte |
| `DESIGN_TELEKINESIS_GLITCH_FOR_ITEM_PHYSICS` | Allows telekinesis target validation to accept a root tagged item_physics (00b4ffa0). | `0` | boolean/byte |
| `DESIGN_TELEKINESIS_GLITCH_FOR_TABLETS` | Allows telekinesis target validation to accept a root tagged tablet (00b4ffa0). | `1` | boolean/byte |
| `DESIGN_TEMPLE_CHECK_FOR_LEAKS` | No-op in this executable: the setting is loaded but has no executable read references (bounded whole-image reference scan). | `1` | boolean/byte |
| `GAMEEFFECT_DAMAGE_MULTIPLIER_COEFF` | Sets the damage factor used by damage-multiplier status effect ID 0x2b; the default `5` multiplies its stored damage values by five. A separate status ID 5 uses its own tier-based coefficient (00b06870). | `5` | float |
| `GAMEEFFECT_ELECROCUTION_RESISTANCE_DURATION_FRAMES` | Adds this many frames to the expiry time for electrocution-resistance status effect ID 0x15 (00b015d0). | `60` | integer |
| `GAMEEFFECT_INVISIBILITY_SHOT_COOLDOWN_FRAMES` | Cooldown in frames used before invisibility can trigger again after a shot; normal timing is divided by five while the relevant high-priority state is active (00b066f0). | `300` | integer |
| `GAMEEFFECT_MOVEMENT_FASTER_2X_SPEED_MULTIPLIER` | Movement multiplier used by the 2x speed effect, status ID 0x48 (00b015d0). | `2.5` | float |
| `GAMEEFFECT_MOVEMENT_SLOWER_SPEED_MULTIPLIER_MAX` | Upper endpoint of the random movement-speed multiplier for the slow effect (00b015d0). | `0.85` | float |
| `GAMEEFFECT_MOVEMENT_SLOWER_SPEED_MULTIPLIER_MIN` | Lower endpoint of the random movement-speed multiplier for the slow effect (00b015d0). | `0.6` | float |
| `GAMEEFFECT_STAINS_DROP_FASTER_DROP_CHANCE_MULTIPLIER` | Scales the active stain-dropping effect’s chance factor; the default multiplies that factor by 10, making the effect produce stains more often (00b02779–00b02798; helper 00c4aa80). | `10` | integer |
| `GAMEEFFECT_TELEPORTITIS_COOLDOWN_FRAMES` | Cooldown in frames written after teleportitis triggers (00b07380). | `60` | integer |
| `GAMEEFFECT_TELEPORTITIS_DAMAGE_MULTIPLIER` | Damage multiplier for teleportitis status effect ID 0x33 (00b06d20). | `0.8` | float |
| `GAMEFFECT_WEAKNESS_DAMAGE_MULTIPLIER` | Multiplies the damage value of weakness status effect ID 0x5d by this factor (00b06d20:160–162). | `10` | float |
| `GAMEPLAY_CHARACTER_LIQUID_FORCES_ENABLED` | When enabled, adds averaged liquid-flow velocity times the character coefficient to the body velocity (00ab6010). | `0` | boolean/byte |
| `GLITCH_ALLOW_5TH_WAND_CARRY` | Bypasses the telekinesis parent-root whitelist, allowing carry targets such as the otherwise rejected fifth-wand case (00b4ffa0). | `0` | boolean/byte |
| `GLITCH_ALLOW_ALT_TAB_SILLINESS` | Preserves the focus-loss/Alt-Tab input glitch: the sampled input state is forced to pressed, and focus-loss handling does not set its normal recovery flag (008184c0, 008187d0). | `0` | boolean/byte |
| `GLITCH_ALLOW_STAIN_DUPLICATION_GLITCH` | Leaves the stain accumulation state intact on the affected cleanup branch, allowing stain duplication (00b01310). | `0` | boolean/byte |
| `GLITCH_ALLOW_VOMIT_BASED_STATUS_EFFECT_GLITCH` | Skips the vomit status-effect cleanup/reconciliation call after clearing its per-material arrays, allowing the related status effect to persist through vomit (00b015d0). | `0` | boolean/byte |
| `GRID_FLEXIBLE_MAX_UPDATES` | When enabled, lowers the adaptive grid-work budget by dividing it by 1.5 below 60 FPS, or raises it by multiplying by 1.4 at 60 FPS or above; the result is clamped to the minimum and maximum settings (00726340). | `0` | boolean/byte |
| `GRID_MAX_UPDATES_PER_FRAME` | Caps how many queued grid-update work items are selected each frame; leftover items wait for later ticks. It does not cap the cells processed by each item (00726340). | `128` | integer |
| `GRID_MIN_UPDATES_PER_FRAME` | Lower clamp for the flexible grid-update budget: when flexible mode is enabled, the engine schedules at least this many queued work items per frame (00726340). | `40` | integer |
| `GUI_HP_MULTIPLIER` | Scales health values used for displayed HP numbers and the Saving Grace HP reserve/threshold calculations (00b5f850, 00b65fb0, 00ab9a90, 00bbc330). | `25` | float |
| `INGESTION_AMOUNT_PER_CELL_MULTIPLIER` | Multiplies the material’s ingestion coefficient and requested amount before adding consumed material to the ingestion total (00b44e80). | `1` | float |
| `INGESTION_LIMIT_DAMAGE` | Fullness threshold as a fraction of maximum satiation that adds ingestion damage; 1.5 means 150% full (00b44c40). | `1.5` | float |
| `INGESTION_LIMIT_EXPLODING` | Fullness threshold as a fraction of maximum satiation that adds the exploding-overeating status; 1.75 means 175% full (00b44c40). | `1.75` | float |
| `INGESTION_LIMIT_EXPLOSION` | Fullness threshold as a fraction of maximum satiation that triggers the overeating explosion entity; 2.0 means 200% full (00b44e80). | `2` | float |
| `INGESTION_LIMIT_SLOW_MOVEMENT` | Fullness threshold as a fraction of maximum satiation that adds the movement-slowed ingestion status; 1.0 means 100% full (00b44c40). | `1` | float |
| `INGESTION_OVERINGESTION_MSG_PERIOD` | Frame interval before the next over-ingestion warning is allowed (00b44e80). | `120` | unsigned integer |
| `INGESTION_SATIATION_PER_CELL` | Satiation points awarded per consumed cell; consumed-cell count is rounded to nearest integer before multiplication and addition (00b44e80). | `6` | unsigned integer |
| `PARTICLE_EMITTER_MAX_PARTICLES` | Caps the requested particle count for particle-emitter creation when the limit is positive (00bc8bb0). | `100000` | integer |
| `PHYSICS_FIX_SHELF_BUG` | Enables an alternate grid-update scheduling path intended to fix the shelf-material bug. The setting changes live update handling; the historical bug fix itself was not runtime-tested (00726340). | `0` | boolean/byte |
| `PHYSICS_RAGDOLL_JOINT_MAX_FORCE_MULTIPLIER` | Sets the ragdoll joint break-force threshold as combined body mass times this multiplier (0071ed40). | `400` | float |
| `PHYSICS_RAGDOLL_JOINT_MIN_BREAK_FORCE` | Floors the ragdoll joint break-force threshold; mass-scaled thresholds below this are raised to this value (0071ed40). | `200` | float |
| `PHYSICS_RAGDOLL_JOINT_STIFFNESS` | Sets the torque limit for a ragdoll revolute joint: higher values let the solver apply stronger correction, so linked body parts resist bending or rotation more firmly (0071ed40, 009cbe50, 009cc510). | `0.05` | float |
| `PLAYER_KICK_FRAMES_IGNORE_COLLISION` | After a kick, the struck body ignores collision for this many game frames (0083d970; also 00bdbab0). | `35` | integer |
| `RAGDOLL_CRITICAL_HIT_FORCE_MULTIPLIER` | Multiplies the force applied to ragdolls for a critical hit (00ab9a90). | `3` | float |
| `RAGDOLL_FX_EXPLOSION_ROTATION` | Scales the rotation applied to ragdoll FX pieces during explosion (00abb7d0). | `0.1` | float |
| `RAGDOLL_IMPULSE_RANDOMNESS` | Sets the random perturbation applied to ragdoll impulse direction/magnitude (00be8240). | `0.04` | float |
| `TELEPORT_ATTACK_COOLDOWN_FRAMES` | Sets three attack cooldown deadlines to current frame plus this many frames after teleport attack (00a22c20). | `60` | integer |
| `WORLD_SEED` | Seed for the generated world: zero requests a fresh random seed; a nonzero value becomes the world-generation RNG seed (008835a0, 00883540, 008771a0, 0087a900). | `0` | unsigned integer |

### Additional interface, input, audio, camera, and streaming settings

| Setting | What changes in the game or its tools | Engine initial value | Engine type |
|---|---|---:|---|
| `AUDIO_COLLISION_KICK_SIZE` | Scales the impact “kick” value sent with collision sound events. | `1` | float |
| `AUDIO_FADE_MUSIC_ON_DEATH` | Enables or suppresses the music fade triggered when the player dies. | `1` | boolean/byte |
| `AUDIO_PICK_GOLD_SAND_AMOUNT_ACCUMULATION_FRAMES` | Sets the frame window in which gold pickups are accumulated for one pickup sound; the accumulated amount resets after the window expires. | `20` | integer |
| `AUDIO_PICK_GOLD_SAND_MIN_AMOUNT_FOR_SOUND` | Sets how much collected gold must accumulate before the gold-pickup sound plays. | `10` | integer |
| `AUDIO_PLAY_GAME_OVER_SOUND` | Enables or suppresses the game-over sound event. | `1` | boolean/byte |
| `CAMERA_ERROR_SMOOTHING_THRESHOLD_Y_MAX` | Sets the upper vertical camera-error threshold used to choose the camera correction rate; it changes how quickly the camera eases toward its target. | `65` | float |
| `CAMERA_ERROR_SMOOTHING_THRESHOLD_Y_MIN` | Sets the lower vertical camera-error threshold used to choose the camera correction rate; it changes how quickly the camera eases toward its target. | `-40` | float |
| `CONTROLS_AIMING_VECTOR_FULL_LENGTH_PIXELS` | Sets the pixel length of the gamepad aiming vector when stick input reaches full extension. | `100` | float |
| `COOP_RESPAWN_TIMER_FRAMES` | Sets the co-op respawn countdown length in game frames. | `7200` | integer |
| `ESC_QUITS_GAME` | When enabled, Escape invokes the quit-game action in the engine input paths that check this setting. | `0` | boolean/byte |
| `GAMEPAD_AIMING_VECTOR_SMOOTHING_COEFF` | Controls how quickly gamepad aim catches up to the stick direction; a larger interpolation coefficient follows the stick faster. | `0.75` | float |
| `GAMEPAD_ANALOG_FLYING_HIGH` | Sets the upper stick-magnitude threshold for analog flying; at and above it, the flying ramp is at its maximum. | `0.95` | float |
| `GAMEPAD_ANALOG_FLYING_LOW` | Sets the lower stick-magnitude threshold for analog flying; below it, the flying ramp is at its minimum. | `0.1` | float |
| `GAME_OVER_DAMAGE_FLASH_FADE_SPEED_MULTIPLIER` | Scales how quickly the damage flash fades; higher values shorten the fade. | `5` | float |
| `INVENTORY_GUI_ALWAYS_VISIBLE` | With `0`, player status bars remain drawn during the inventory wand-row transition. With `1`, the game can hide them for that transition frame; they draw at other times. | `1` | boolean/byte |
| `RENDER_color_grading_LERP_SPEED` | Controls the per-update step used to ease color-grading channels (red, green, blue, and grayscale) toward their target values. | `0.015` | float |
| `REPORT_DAMAGE_BLOCK_MESSAGE_INTERVAL_FRAMES` | Sets the minimum game-frame interval before the game logs another blocked-attack message. | `30` | integer |
| `SETTINGS_MIN_RESOLUTION_X` | Sets the minimum accepted window width; smaller saved or pending widths are raised to this value, and narrower display modes are filtered out. | `640` | integer |
| `SETTINGS_MIN_RESOLUTION_Y` | Sets the minimum accepted window height; smaller saved or pending heights are raised to this value. | `360` | integer |
| `STEAM_CLOUD_SIZE_WARNING` | Enables or suppresses the Steam Cloud size warning in the settings UI. | `0` | boolean/byte |
| `STREAMING_AUTOSAVE_PERIOD_SECONDS` | Sets the autosave period in seconds for the streaming/save update. | `180` | integer |
| `STREAMING_ENABLED` | Enables the chunk-to-disk streaming update. When disabled, that update returns without processing chunks. | `1` | boolean/byte |
| `STREAMING_FREQUENCY` | Sets the base interval between chunk-streaming batches; smaller values schedule batches more often. | `1` | double |
| `STREAMING_PERSISTENT_WORLD` | Controls whether persistent-world entities are included when world data is saved and restored. | `1` | boolean/byte |
| `UI_BARS_SCALE` | Scales the size of the player status bars. | `1` | float |
| `UI_COOP_QUICK_INVENTORY_HEIGHT` | Sets the vertical spacing between co-op players’ quick-inventory rows. | `24` | float |
| `UI_COOP_STAT_BARS_HEIGHT` | Sets the vertical spacing between co-op players’ status-bar rows. | `48` | float |
| `UI_DISPLAY_NUMBERS_WITH_KS_AND_MS` | Formats large UI numbers with K and M shorthand when enabled. | `1` | boolean/byte |
| `UI_DISPLAY_POTION_CONTENT_PERCENTS` | When enabled, potion contents UI shows the container’s fill percentage. | `1` | boolean/byte |
| `UI_GAMEOVER_SCREEN_MUSIC_CUE_TIMER_FRAMES` | Sets how many game-over frames elapse before the game-over statistics music cue plays. | `300` | integer |
| `UI_IMPORTANT_MESSAGE_POS_Y` | Sets the vertical position of important messages as a percentage of screen height; `65` means 65% down the screen before UI scaling is applied. | `65` | integer |
| `UI_INVENTORY_BACKGROUND_POSITION_X` | Sets the horizontal position of the inventory background. | `0` | float |
| `UI_INVENTORY_BACKGROUND_POSITION_Y` | Sets the vertical position of the inventory background. | `0` | float |
| `UI_MAX_PERKS_VISIBLE` | Sets the maximum number of perk icons shown in the visible perk row. | `7` | integer |
| `UI_MOUSE_WORLD_HOVER_TEXT_OFFSET_X` | Offsets the horizontal position of world-hover text next to the mouse cursor. | `10` | float |
| `UI_MOUSE_WORLD_HOVER_TEXT_OFFSET_Y` | Offsets the vertical position of world-hover text next to the mouse cursor. | `10` | float |
| `UI_PIXEL_FONT_GAME_LOG` | Selects the pixel font for the in-game log when enabled. | `1` | boolean/byte |
| `UI_POTION_CONTENTS_COMBINE_SAME_NAME_MATERIALS` | Combines potion-content entries that have the same material name when enabled. | `1` | boolean/byte |
| `UI_QUICKBAR_OFFSET_X` | Sets the horizontal position offset of the quick inventory bar. | `0` | float |
| `UI_QUICKBAR_OFFSET_Y` | Sets the vertical position offset of the quick inventory bar. | `0` | float |
| `UI_SNAP_TO_NEAREST_INTEGER_SCALE` | Rounds scaled GUI pixel dimensions down to whole pixels, keeping the layout aligned to integer screen pixels. | `1` | boolean/byte |
| `UI_WAND_TAG_OVERWRITES_NAME` | For an entity tagged “wand,” replaces its displayed inventory/HUD item name with the localized generic wand label. | `1` | boolean/byte |

### Additional developer and diagnostic settings

| Setting | What changes in the game or its tools | Engine initial value | Engine type |
|---|---|---:|---|
| `CAMERA_IS_FREE` | Stops the camera following the player and lets the free-camera controls move it; DEBUG_FREE_CAMERA_SPEED scales that movement. | `0` | boolean/byte |
| `CAMERA_IS_UI_OPEN` | **No vanilla effect.** The engine stores the flag but never checks it, so changing the value does not affect the camera or UI. | `0` | boolean/byte |
| `DEBUG_ALWAYS_COMPLETE_THE_GAME` | Bypasses the normal completion gates so the game treats the run as complete. | `0` | boolean/byte |
| `DEBUG_ALWAYS_GET_UNLOCKS` | Writes the unlock flags for completed-kill unlocks even when those flags were already set. | `0` | boolean/byte |
| `DEBUG_ALWAYS_RANDOM_SEED` | Seeds the random generator from the current system time before choosing the world seed, making the debug seed vary with startup time. | `0` | boolean/byte |
| `DEBUG_ALWAYS_RANDOM_START_POS` | Chooses a randomized starting position instead of the normal start position. | `0` | boolean/byte |
| `DEBUG_ATTRACT_MODE` | Enters the attract/demo mode path. | `0` | boolean/byte |
| `DEBUG_AUDIO_DEV_MODE` | Initializes FMOD in its alternate developer/debug mode; this changes the audio engine setup, not game rules. | `0` | boolean/byte |
| `DEBUG_AUDIO_MUSIC_MUTE` | Sets the active music event volume/intensity field to zero, muting the music. | `0` | boolean/byte |
| `DEBUG_AUDIO_MUTE` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_AUDIO_VOLUME` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `1` | float |
| `DEBUG_AUDIO_WRITE_TO_FILE` | Selects FMOD file output and increases the audio DSP buffer size to 800 samples. | `0` | boolean/byte |
| `DEBUG_CAMERABOUND_DISPLAY_ENTITIES` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_CAMERA_SHAKE_OFFSET` | Adds this offset to both horizontal and vertical camera shake. | `0` | float |
| `DEBUG_COLLISION_TRIGGERS` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_CRASH_IF_OLD_VERSION` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `1` | boolean/byte |
| `DEBUG_CTRL_O_USES_PREV_ENTITY_ALWAYS` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_DEMO_MODE` | Runs the demo/attract sequence and its timed reset behavior instead of leaving a normal run untouched. | `0` | boolean/byte |
| `DEBUG_DEMO_MODE_RESET_TIMEOUT_FRAMES` | Sets how many frames pass before the demo run resets. | `10800` | integer |
| `DEBUG_DEMO_MODE_RESET_WARNING_TIME_FRAMES` | Sets how many frames before reset the demo warning is shown. | `900` | integer |
| `DEBUG_DISABLE_MOUSE_SCROLL_WHEEL` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_DISABLE_PHYSICSBODY_OUT_OF_BOUNDS_WARNING` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_DISABLE_POSTFX_DITHERING` | Disables dithering in the post-processing effect. | `0` | boolean/byte |
| `DEBUG_DISPLAY_INTERNAL_ID_IN_PROGRESS_MENU` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `1` | boolean/byte |
| `DEBUG_DRAW_ANIMAL_AI_STATE` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_ENABLE_AUTOSAVE` | Enables the engine autosave paths. | `1` | boolean/byte |
| `DEBUG_EXTRA_SCREENSHOT_KEYS_ENABLED` | Enables the extra screenshot/debug keyboard shortcuts handled by the screenshot tool. | `1` | boolean/byte |
| `DEBUG_F12_OPEN_FOG_OF_WAR` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `1` | boolean/byte |
| `DEBUG_FREE_CAMERA_SPEED` | Scales free-camera movement speed. | `2` | float |
| `DEBUG_FULL_WANG_MAPS` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_GAME_LOG_SHOW_DRAWN_ACTIONS` | Adds the drawn wand-action name to the game log. | `0` | boolean/byte |
| `DEBUG_GENERATE_BIG_WANG_MAP` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_GIF_HEIGHT` | Sets the height of GIF/video captures. | `360` | integer |
| `DEBUG_GIF_RECORD_60FPS` | Makes GIF/video recording use 60 frames per second. | `0` | boolean/byte |
| `DEBUG_GIF_WIDTH` | Sets the width of GIF/video captures. | `640` | integer |
| `DEBUG_HOTLOAD_MATERIAL_EDGES` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_IMGUI_HOT_LOAD_ENABLED` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `1` | boolean/byte |
| `DEBUG_INTRO_PLAY_ALWAYS` | Forces the intro playback decision on each startup, even when saved intro state says it has played. | `0` | boolean/byte |
| `DEBUG_KEYS_ENABLED` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `1` | boolean/byte |
| `DEBUG_LOG_LEVEL` | Filters game-log entries by their level. | `-1` | integer |
| `DEBUG_LOG_LIFETIME_COMPONENT_DANGLING_PARENTS` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_LOG_NEVER_VISIBLE` | Suppresses drawing the game-log panel. | `0` | boolean/byte |
| `DEBUG_LOG_SOLID_BACKGROUND` | Changes the game-log panel background mode (transparent, solid, or alternate solid mode). | `2` | integer |
| `DEBUG_LOG_STD_COUT` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `1` | boolean/byte |
| `DEBUG_LOG_STREAMING_STATS` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `1` | boolean/byte |
| `DEBUG_LOG_TODO_ERRORS` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `1` | boolean/byte |
| `DEBUG_LUA` | Prints each Lua script path as the engine runs it. | `0` | boolean/byte |
| `DEBUG_LUA_DONT_REPEAT_ERRORS` | Suppresses duplicate Lua error reports. | `1` | boolean/byte |
| `DEBUG_LUA_LOG_BIOME_SPAWN_SCRIPTS` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_LUA_REPORT_BIOME_SPAWN_ERRORS` | Writes biome-spawn Lua script errors to the diagnostic output. | `1` | boolean/byte |
| `DEBUG_LUA_REPORT_PRINT_FILES` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_LUA_REPORT_SLOW_SCRIPTS` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_MATERIAL_AREA_CHECKER` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_NO_LOGO_SPLASHES` | Skips the startup logo/splash screens. | `0` | boolean/byte |
| `DEBUG_NO_PAUSE_ON_WINDOW_FOCUS_LOST` | Keeps the simulation running when the game window loses focus. | `0` | boolean/byte |
| `DEBUG_NO_SAVEGAME_CLEAR_ON_GAME_OVER` | Keeps the run save data when the player dies instead of clearing it. | `0` | boolean/byte |
| `DEBUG_OLLI_CONFIG` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_PAUSE_BOX2D` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_PAUSE_GRID_UPDATE` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_PAUSE_SIMULATION` | Pauses simulation updates while the game is active. | `0` | boolean/byte |
| `DEBUG_PERSISTENT_FLAGS_DISABLED` | Suppresses the warning/debug handler when a persistent-flag lookup misses; the traced consumer does not disable normal persistent flags globally. | `0` | boolean/byte |
| `DEBUG_PETRI_START` | Starts the Petri developer tool path. | `0` | boolean/byte |
| `DEBUG_PETRI_TAKE_RANDOM_SHADERSHOT` | Enables the Petri developer tool’s random shader screenshot capture. | `0` | boolean/byte |
| `DEBUG_PIXEL_SCENES` | Prints pixel-scene placement successes and failure diagnostics; it does not change placement. | `0` | boolean/byte |
| `DEBUG_PLAYER_NEVER_DIES` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_PREV_OPENED_ENTITY` | No vanilla effect in this executable: the key is registered and loaded, but its value has no gameplay or developer-tool consumer; only string cleanup references it. | `empty (null)` | string |
| `DEBUG_PRINT_COMPONENT_UPDATOR_ORDER` | Prints the component-updater execution order for diagnosing update sequencing. | `0` | boolean/byte |
| `DEBUG_PROFILER_CAPTURE_OLLI_STYLE` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `1` | boolean/byte |
| `DEBUG_PROFILER_CAPTURE_PETRI_STYLE` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_PROFILE_ALLOCATOR` | Enables allocator profiling instrumentation. | `0` | boolean/byte |
| `DEBUG_REPLAY_RECORDER_FPS` | Sets the replay recorder frame rate. | `30` | integer |
| `DEBUG_RESTART_GAME_IF_OLD_VERSION` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_SCREENSHOTS_VIDEO_PATH_PREFIX` | Sets the path prefix used by the screenshot/video capture output. | `empty (null)` | string |
| `DEBUG_SCREENSHOTTER_ENABLED` | Enables the screenshot/video capture tool. | `1` | boolean/byte |
| `DEBUG_SCREENSHOTTER_FFMPEG_PATH` | Sets the FFmpeg executable path used by video capture. | `empty (null)` | string |
| `DEBUG_SCREENSHOTTER_SAVE_PPNG` | Selects PPNG output for screenshots instead of the other screenshot format. | `1` | boolean/byte |
| `DEBUG_SHOW_MOUSE_MATERIAL` | Adds a diagnostic line for the mouse position to the game log, including the hovered entity/material label and entity health and durability when available. | `0` | boolean/byte |
| `DEBUG_SINGLE_THREADED_LOADING` | Forces asset loading onto the single-threaded loading path. | `0` | boolean/byte |
| `DEBUG_SI_TYPE` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | integer |
| `DEBUG_SKIP_ALL_START_MENUS` | Skips the startup menu sequence. | `0` | boolean/byte |
| `DEBUG_SKIP_MAIN_MENU` | Skips the main-menu step during startup. | `0` | boolean/byte |
| `DEBUG_SKIP_RELEASE_NOTES` | Skips the release-notes screen during startup. | `0` | boolean/byte |
| `DEBUG_SKYLIGHT_NO_SIMD` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_SPRITE_UV_GEN_REPORT_MISSING_FILES` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_STREAMING_DISABLE_SAVING` | Prevents streamed world chunks from being written to save storage. | `0` | boolean/byte |
| `DEBUG_STREAMING_INTEGRATION_DEV_MODE` | Uses the developer test identity for the streaming integration (random “justinfan…” name and test channel credentials) instead of the normal account path. | `0` | boolean/byte |
| `DEBUG_TELEPORT` | Replaces the normal 1.5-second teleport screen flash with a visible debug marker at the teleport location. | `0` | boolean/byte |
| `DEBUG_TEST_SAVE_SPAWN_X` | Uses this X coordinate for the test-save spawn location. | `190` | float |
| `DEBUG_TEST_SYMBOL_CLASSIFIER` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_TEXT_ENABLE_WORK_MODE` | Shows the missing translation’s key itself instead of the generic “why are you looking here” placeholder, so missing strings can be spotted while editing text. | `0` | boolean/byte |
| `DEBUG_TEXT_WRITE_MISSING_TRANSLATIONS` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `DEBUG_THREADED_WORLD_CREATION` | Uses threaded world creation rather than the single-threaded path. | `1` | boolean/byte |
| `DEBUG_TREES` | Prints diagnostic messages when tree placement fails; it does not change the placement decision. | `0` | boolean/byte |
| `DEBUG_UI_ALL_THE_CARDS_DISPLAY_EXTRA_INFO` | Shows extra internal information on all cards in the UI. | `1` | boolean/byte |
| `DEBUG_USE_DEBUG_PRELOAD` | Adds a broad recursive entity preload during startup, so more assets are loaded before play. | `0` | boolean/byte |
| `DEBUG_USE_PRELOAD` | Preloads common game assets during startup to reduce later loading stalls. | `1` | boolean/byte |
| `DEBUG_WANG` | Prints Wang terrain-generation diagnostics, including the selected offsets and biome regions, and enables the diagnostic map output. | `0` | boolean/byte |
| `DEBUG_WANG_PATH` | Highlights the recorded pathfinding route in bright magenta on the Wang diagnostic map; it changes the diagnostic image, not the generated terrain. | `0` | boolean/byte |
| `GAME_LOG_ENABLED` | Enables drawing the in-game diagnostic log panel. | `1` | boolean/byte |
| `_DEBUG_DONT_LOAD_OTHER_MAGIC_NUMBERS` | No vanilla effect in this executable: the key is registered and can be loaded, but no executable code reads its value. | `0` | boolean/byte |
| `_DEBUG_DONT_SAVE_MAGIC_NUMBERS` | Suppresses the MagicNumbers notification/dialog path for queued magic-number entries (including its special salakieli message); the traced consumer does not stop magic-number loading. | `0` | boolean/byte |

## Entries not recognized by the inspected engine

These seven names are present in the XML snapshot but do not match an exact key in the engine registration code:

- `CAMERA_ERROR_SMOOTHING_THRESHOLD_Y`
- `PHYSICS_FLOATER_FORCE_BALANCING_COEFF`
- `PHYSICS_FLOATER_FORCE_COEFF`
- `PHYSICS_FLOATER_FORCE_VEC_MAX`
- `PHYSICS_FLOATER_TORQUE_BALANCING_COEFF`
- `PHYSICS_FLOATER_TORQUE_COEFF`
- `RAGDOLL_RAGDOLL_FX_EXPLOSION_ROTATION`

The binary has nearby names in some of these families, but those are separate keys. Do not assume they make these XML spellings work.

## What the engine check establishes

The inspected game build accepts 404 exact keys. Of the 201 names in the local XML, 194 are registered and seven are ignored because the engine does not accept those exact spellings. Across all 404 registered settings, 77 have no vanilla gameplay, visual, audio, or diagnostic effect and are labeled in the tables. That count includes the 34 XML settings with no value reader, cleanup-only `REPORT_DAMAGE_FONT`, the two player-kick Verlet requests that are queued and then discarded, and 40 engine-only settings with no effective consumer. Mods can still read registered settings through `MagicNumbersGetValue(key)` and assign their own behavior.

The values in this guide describe the shipped defaults, engine types, and effects visible in the executable. They are not a promise that every possible replacement value is safe. Change values in small steps, and keep booleans at `0` or `1`.

Mods can also add magic-number XML during initialization with `ModMagicNumbersFileAdd`; see [Lua execution and file loading](lua-execution.md#after-magic-numbers-nine-globals-disappear).
