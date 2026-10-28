# Special and engine-recognized tags

Noita tags are free-form labels. Most tags only categorize entities or components, while some are read by a particular engine subsystem or by vanilla Lua. This page audits every entry on the [wiki’s Special tags list](https://noita.wiki.gg/wiki/Modding%3A_Special_tags) against the executable and shipped game data, and adds tags with concrete behaviors that the list omits.

The descriptions below state the observed effect. An address in parentheses points to a native code path. “Not established” means this audit found no direct behavior for that tag; it does not mean a mod cannot use the tag as a selector or that a dynamic tag lookup is impossible. For the general tag system and its limits, see [Tags System](https://noita.wiki.gg/wiki/Modding%3A_Tags_System).

One wiki classification error: `tablet` is an **entity tag** in vanilla data, not just a component `_tags` entry.

## Entity tags on the wiki’s Special tags page

| Tag | What it does |
|---|---|
| `boss_centipede` | → Starts the boss-arena battle music when a tagged entity is nearby. (`0x00564580`) |
| `firemage` | → AnimalAI uses this tag in target/relationship checks. The exact relation and conditions are not fully resolved here; the wiki’s held-Brimstone neutrality claim is not independently established by this trace. (`0x00A28E10`) |
| `fish_attractor` | → Makes the tagged entity a target for entities using `AdvancedFishAIComponent`. (`0x009F82E0`) |
| `gold_nugget` | → Makes the entity eligible for Telekinesis attraction; the Attract Items perk also searches for this tag in Lua. (`0x00C2DFC0`) |
| `miniboss` | → Starts the miniboss music track when a tagged entity is nearby. (`0x00564580`) |
| `necromancer_shop` | → Starts the necromancer-shop music track when a tagged entity is nearby. (`0x00564580`) |
| `player_unit` | → Makes `IsPlayer()` return true for the entity. The wiki’s extra-stains claim was not separately traced in this audit. (`0x006BFCB0`) |
| `polymorphable_NOT` | → Excludes the entity from the polymorph conversion path. (`0x00B041E0`) |
| `teleportable_NOT` | → Acts as an exclusion marker in teleport/status-effect paths; a replacement path removes it when it adds `teleportable`. This does not establish that every teleport system honors the marker. (`0x00B041E0`, `0x006B6D60`) |
| `thundermage` | → AnimalAI uses this tag in target/relationship checks. The exact relation and conditions are not fully resolved here; the wiki’s held-Thunderstone neutrality claim is not independently established by this trace. (`0x00A28E10`) |
| `touchmagic_immunity` | → Prevents the material-conversion effect handled by `MagicConvertMaterialComponent` from affecting the entity. (`0x00BAB8C0`) |
| `ui_use_raw_name` | → Skips translation lookup for the entity name so the raw name is shown. (`0x00B196C0`) |
| `wand` | → The item-name path can substitute `$item_wand`; vanilla Lua also uses this tag to find and manipulate wands. The native string cross-reference does not conclusively identify the comparison literal, so the first link is consistent with the wiki claim but is not independently closed here. (`0x00B7FE90`) |
| `workshop` | → Makes a hitbox on the tagged entity define an area in which the player can edit wands. (`0x00B7D8D0`) |
| `workshop_aabb` | → Starts the temple-entry music cue when the player enters the tagged area. (`0x00564580`) |
| `workshop_untouched` | → Opens the inventory on the first overlap with the workshop marker, then clears this tag so the automatic opening does not repeat. (`0x00B7D8D0`) |

## Other entity tags listed by the wiki

| Tag | What it does |
|---|---|
| `drone` | → No tag-specific engine effect was established. The native string references checked here are in a path comparison and a ragdoll/category table, not an entity-tag predicate. |
| `enemy` | → Qualifies an entity for the engine’s damage hit-effect path. (`0x0063EA50`, called from `0x00AB9A90`) |
| `prey` | → Serves as a selector where a component or projectile asks for entities with a particular target tag; it has no distinct hardcoded behavior established here. |
| `effectable_prop` | → Qualifies an entity for the engine’s damage hit-effect path. (`0x0063EA50`, called from `0x00AB9A90`) |
| `hiteffect_enabled` | → Explicitly opts an entity into the damage hit-effect path when it was not accepted by the `enemy`, `prop`, or `effectable_prop` checks. (`0x0063EA50`) |
| `HAX_polymorph_death` | → Marks the polymorph-death path; the player-destroy handler checks it and writes a diagnostic log message. It is not a general gameplay toggle. (`0x00B041E0`, `0x006B7620`) |
| `prop` | → Qualifies an entity for the engine’s damage hit-effect path. (`0x0063EA50`, called from `0x00AB9A90`) |
| `polymorphed` | → No distinct native tag check or tag-specific effect was established in this audit. |
| `hittable` | → Adds the entity’s hitbox to grid-particle collision handling; the player also includes it in the nearby kick-target query. (`0x00712E40`, `0x00BF2040`) |
| `player_ragdoll` | → Lets camera/respawn handling find player ragdolls; the respawn path consumes nearby tagged ragdolls and clears their tag. (`0x006B3FF0`, `0x00ABB7D0`) |
| `player_projectile` | → Marks a player projectile that nearby AnimalAI can detect and react to after a collision-overlap check. This is not proof that every player projectile is tagged. (`0x00A28E10`, `0x00BDBAB0`) |
| `music_energy_100_near` | → Contributes to the strongest nearby music-energy tier, using the smaller “near” search region. (`0x005673A0`) |
| `music_energy_100` | → Contributes to the strongest music-energy tier using the wider search region. (`0x005673A0`) |
| `music_energy_050_near` | → Contributes to the middle music-energy tier using the smaller “near” search region. (`0x005673A0`) |
| `music_energy_050` | → Contributes to the middle music-energy tier using the wider search region. (`0x005673A0`) |
| `music_energy_000_near` | → Contributes to the lowest music-energy tier using the smaller “near” search region. (`0x005673A0`) |
| `music_energy_000` | → Contributes to the lowest music-energy tier using the wider search region. (`0x005673A0`) |
| `ending_sampo_spot_underground` | → Marks a nearby underground Sampo interaction spot; the ending interaction uses it to select the underground prompt/action. (`0x00ADAE80`) |
| `ending_sampo_spot_mountain` | → Marks a nearby mountain Sampo interaction spot; the ending interaction uses it to select the mountain prompt/action. (`0x00ADAE80`) |
| `coop_respawn` | → In the co-op respawn path, a nearby marker is removed and the player’s spawn countdown is armed. (`0x00BF2040`) |
| `mortal` | → Places an entity in broad target searches used by AnimalAI, worm/crawler attacks, and music-energy grouping. The tag alone does not mean the entity has health or can die. (`0x00A28E10`, `0x00A44730`, `0x00A98F20`, `0x005673A0`) |

## Component tags on the wiki’s Special tags page

| Tag | What it does |
|---|---|
| `enabled_in_hand` | → Enables the tagged component when its owning item is in the held state. (`0x00B53150`) |
| `enabled_in_world` | → Enables the tagged component when its owning item is in the world. (`0x00B53150`) |
| `enabled_in_inventory` | → Enables the tagged component when its owning item is in the inventory but not in the held or world state. (`0x00B53150`) |
| `not_enabled_in_wand` | → Is recognized as an exclusion marker in the item-component state evaluator when its third context flag is set. The saved function body does not expose a clear final result for this case, so the exact disable condition remains unverified. (`0x00B53150`) |
| `shop_cost` | → Selects the item-price display component for price synchronization. The wiki’s claim that it is removed on pickup was not proven by the native bodies checked here. (`0x00B8F9F0`, `0x00B8FD70`) |
| `fire` | → Vanilla altar-torch logic enables components tagged `fire` when it lights the torch. This confirms a Lua-side convention; automatic `TorchComponent` lit-state handling was not independently verified in the native trace. |
| `item_identified` | → Enables or displays the tagged component only for the identified-item state. (`0x00B53150`, `0x00B8C200`) |
| `item_unidentified` | → Enables or displays the tagged component for the unidentified-item state. (`0x00B53150`, `0x00B8C200`) |
| `item_unlocked` | → Enables or displays the tagged component when the item’s unlock check succeeds. (`0x00B53150`, `0x00B8C200`) |
| `item_locked` | → Enables or displays the tagged component when the item’s unlock check fails. (`0x00B53150`, `0x00B8C200`) |
| `tablet` | → **Entity tag:** when `DESIGN_TELEKINESIS_GLITCH_FOR_TABLETS` is enabled, allows tagged roots through the telekinesis target filter; a separate item-cost path overwrites the computed price field with `-1` for tagged entities. The meaning of that sentinel is unresolved. (`0x00B4FFA0`, `0x00C069D0`) |
| `charge_indicator` | → No engine consumer was established. In vanilla data it labels a disabled sprite-particle emitter beside an electric-charge component. |
| `turret_rotate_sound` | → Selects the turret’s rotation AudioLoopComponent, whose loop is updated by turret rotation handling. (`0x00A272D0`) |
| `sound_jetpack` | → Selects the player jetpack sound loop, which is updated with jetpack activity. (`0x00A811B0`) |
| `sound_underwater` | → Selects the underwater breathing sound loop. (`0x00AB81B0`) |
| `sound_suffocating` | → Selects the suffocation sound loop. (`0x00AB81B0`) |
| `sound_damage_curse` | → Selects the curse-damage sound loop/effect when curse damage is applied. (`0x00AB9A90`) |
| `sound_pick_gold_sand` | → Selects the sound component played when gold sand is picked up. (`0x00BBC330`) |
| `sound_spray` | → Selects the sound loop used while the player is spraying. (`0x00BFDD10`) |
| `sound_air_whoosh` | → Selects the movement whoosh sound used by the player movement path. (`0x00BF2040`) |
| `sound_telekinesis_move` | → Selects the sound loop used while a telekinetically held object is moving. (`0x00C2DFC0`) |
| `sound_telekinesis_hold` | → Selects the sound loop used while a telekinetic object is held stationary. (`0x00C2DFC0`) |
| `jetpack` | → Selects the player’s jetpack particle emitters; the engine enables or updates them according to flight/airborne state. (`0x00A7EA90`) |
| `ingestion` | → Vanilla data labels a MaterialInventoryComponent as the stomach inventory for IngestionComponent. No direct native tag lookup was found, so the exact engine association is not independently verified here. |
| `digger` | → Selects a component whose numeric field is passed into the material-suction update path. The meaning of that field is not resolved. (`0x00C0A930`) |
| `no_hitbox` | → Excludes the tagged component from a hitbox registration/update path. (`0x00A53610`) |
| `shield_ring` | → Selects shield-ring components whose state/geometry is updated during shield handling. (`0x00ADEFD0`) |
| `shield_hit` | → Selects shield-hit effect components and dispatches their hit response when the shield catches an attack. (`0x00ADEFD0`) |
| `shoot_pos` | → Supplies the HotspotComponent position used as a projectile/wand shot origin. (`0x00B50260`, `0x00B51670`, `0x00B813A0`) |
| `crouch_sensor` | → Supplies the HotspotComponent position used by the player crouch/ground sensor. (`0x00A7EA90`) |
| `hand` | → Vanilla uses it to label hand HotspotComponents, but no direct native tag consumer was established in this audit. |
| `kick_pos` | → Supplies the HotspotComponent position used to place the player’s kick trace. (`0x00BFA850`) |
| `spray_pos` | → Supplies the HotspotComponent position used as the spray origin. (`0x00BFDD10`) |
| `item_bg` | → Selects the inventory-card background sprite component. (`0x00B8C200`) |
| `fart` | → No tag-specific engine behavior was established; in vanilla data it labels particle emitters on fart effects. |
| `smoke` | → Selects smoke ParticleEmitterComponents for periodic randomized state updates. The exact updated field’s meaning is unresolved. (`0x00B015D0`) |
| `aiming_reticle` | → Selects the player’s reticle component; the aiming update moves it and controls its enabled/visible state. (`0x00BF2040`, `0x006F4D10`) |
| `character` | → The engine adds it to some secondary entities created by character contact/damage paths. No distinct downstream effect is established; treat it as a classification/propagation marker. (`0x00ABB580`, `0x00B52CB0`) |
| `laser_sight` | → Supplies an aiming offset and identifies the laser-sight component whose enabled state is reset by the aiming path. (`0x00A22540`, `0x00A272D0`) |
| `gun` | → No native tag-specific effect was established. In the scanned vanilla data it labels gun/tank sprite components. |
| `health_bar` | → Selects the foreground sprite used by an entity health bar. (`0x00A58820`) |
| `item` | → No dedicated native handler was established. Vanilla Lua uses it to find item/wand sprite components for sprite edits. |

## Verified omissions and candidates not promoted

These tags have concrete engine behavior but were absent from the Special tags page:

| Tag | What it does |
|---|---|
| `teleportable` | → Positive counterpart to `teleportable_NOT`: it is used by a region query and is added to replacement entities as the negative marker is removed. Individual teleport paths still need to be checked before treating it as a universal allow rule. (`0x00C30450`, `0x006B6D60`) |
| `kickable` | → Adds a nearby entity to the player’s kick-target list; collected targets receive the `$damage_kick` effect and the kick physics response. (`0x00BF2040`) |
| `polymorphed_cessation` | → Changes the tracked-entity bounds/camera path: when present, the ordinary bounds interpolation and camera recenter update are skipped. The reason for the vanilla Nibbana use is not known. (`0x006B3FF0`) |
| `homing_target` | → Default `HomingComponent` target tag. Entities carrying it are eligible targets for components/projectiles that use the default target tag. (`0x00A0B1D0`) |
| `workshop_show_hint` | → Gates the `$hint_editwands` tooltip when the player overlaps its marker. (`0x00B7D8D0`) |
| `this_is_sampo` | → Special-cases the tagged Sampo carrier in entity relationship/targeting logic; whether it is considered friendly or hostile depends on the caller. (`0x00B5DCB0`, `0x00B5EE40`, `0x00B724C0`) |
| `item_physics` | → When `DESIGN_TELEKINESIS_GLITCH_FOR_ITEM_PHYSICS` is enabled, allows a tagged root through telekinesis target validation. The setting is off by default in the inspected build. (`0x00B4FFA0`) |
| `workshop_spell_visualizer` | → Marks the anchor entity whose queued spell actions are rendered as visualizer tiles. (`0x00B2A310`) |
| `workshop_spell_visualizer_tile` | → Marks transient action-icon tiles; the visualizer path removes old tiles and creates/positions the current set. (`0x00B2A310`) |
| `health_bar_back` | → Selects the background sprite drawn behind the entity health-bar fill. (`0x00A58820`) |
| `fast_movement_particles` | → Tags player particle emitters that the engine toggles when movement crosses its fast-movement threshold. (`0x00BF2040`) |

The following component labels were also checked because they appear in the full vanilla tag inventory or in native string references, but they are not confirmed hardcoded special tags:

| Tag | What is established |
|---|---|
| `activate` | → Vanilla runestone logic enables components carrying this tag while the runestone is active and disables them otherwise. This is a Lua/data convention using the generic component-toggle API, not a separate native tag feature. |
| `sound_digger` | → Vanilla wands attach this to the digger AudioLoopComponent, and wand actions select it through their `sound_loop_tag` field. The state gating is handled by the generic item component-state rules; a separate native behavior for this label was not found. |
| `sound_prebattle_tinkering` | → Vanilla centipede data assigns it to an AudioLoopComponent with the pre-battle tinkering event. The tag’s specific trigger is unresolved. |
| `laser` | → The matching native strings are in LaserEmitterComponent property/serialization handling; no entity-tag predicate or tag-specific effect was established, so it is not counted as a verified omission. |
| `projectile` | → The native references are damage-type field names in component metadata, not entity-tag checks; no additional behavior for the root entity tag was established by those references. |
| `curse` | → The native references are damage-type field names in component metadata, not entity-tag checks; no additional behavior for the root entity tag was established by those references. |
| `deactivate` | → The shield routine passes this name into its effect path when the shield deactivates. The scanned vanilla XML has no matching tag assignments, so a general tag behavior is not established. |
| `block` | → The shield routine passes this name into its effect path when the shield blocks an attack. The scanned vanilla XML has no matching tag assignments, so a general tag behavior is not established. |
| `water` | → This is a material-registry tag used in liquid/material classification, not an entity or component `_tags` entry. |
| `item_pickup` | → This is a Lua callback field name, not an entity tag. |

## Scope and confidence

The audit used static decompilation and shipped XML/Lua data; it was not tested in a live game. Exact string references, tag predicates, and matching data assignments are strong evidence for the listed effects. A missing string xref is only a bounded negative: a dynamic lookup or Lua-driven use can still exist. Build offsets may change after a game update.
