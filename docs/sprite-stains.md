# Sprite stains

Noita's sprite-stain system is separate from the material-stain settings used by liquids. An entity receives visible sprite stains through a `SpriteStainsComponent` attached to a sprite the engine can resolve and map.

## Invisibility liquid and enemy targeting

The visible sprite overlay and a liquid's status effect are separate systems. The `magic_liquid_invisibility` material is configured with `liquid_stains="1"` and `status_effects="INVISIBILITY"`; the status entry loads `effect_invisibility.xml`, whose `GameEffectComponent` specifies `frames="7200"`. The status system keeps a matching GameEffect entity while the status is active; it does not reset that entity's duration every update. The 7200-frame counter therefore describes the effect entity's lifetime, while liquid contact controls whether the source status remains active.

Enemy detection follows the active invisibility effect, not the visible stain texture. `IsInvisible(entity_id)` reports the active `INVISIBILITY` game effect. The native AnimalAI candidate scan performs its visibility/obstacle check and then excludes a candidate when that same effect is active (`0x00A28E10`, helper `0x00B06640`). Each AI schedules these scans by its configured detection interval, so a scan can consider the player only if the invisibility bit is off when that scan runs.

The system manager sorts systems by update priority and then name. `AnimalAISystem`, `GameEffectSystem`, `StatusEffectDataSystem`, and `SpriteStainsSystem` all have priority 100, so they run in that order. At the normal 7200-frame GameEffect expiry, the GameEffect system removes the old entity and the later status pass recreates it synchronously if the status is still active. The replacement's add callback turns invisibility back on before that pass ends. AnimalAI already ran earlier in the same update, so it cannot see this short interval between removal and recreation.

The scan runs before that AI's behavior callback in the same update (`0x00A22540`). When the source status becomes inactive, the status processor marks its linked effect to expire by setting `frames` to `1` (or leaves it at `0`); the GameEffect system later removes it. If a stain sample first restores an active amount after the status pass that sees the old effect already gone, AI runs before the next status pass recreates it and can scan once while the effect is absent. The exact liquid-contact timing that would produce this sequence has not been traced. Same-update shooting after a newly successful scan is also unresolved; the enemy's state and attack timing matter.

### How status processing recreates the effect

The registered invisibility status is evaluated by the `StatusEffectDataSystem` once per system update and when its component is added. A status row is active when its stain amount meets the active threshold or its separate ingestion-effect timer remains positive after that update's decrement and row-specific threshold checks. If it is active and no matching effect entity is tracked for it, the status processor clones the configured entity and marks its GameEffect component with the status index (`0x00C67F00` → `0x00C24670`). If a matching effect is still present, the processor skips creation even when its `frames` counter is zero. It does not reset or refresh a matching effect every update.
The ingestion timer is separate from the GameEffect counter. When material is consumed, native ingestion adds the material's configured `StatusEffect amount × consumed quantity` to that status's timer in seconds; stinky meat adds 2 seconds per consumed unit for `PROTECTION_POLYMORPH`. The status system subtracts 1/60 second each update. A positive timer can therefore keep the row active after its GameEffect entity reaches its fixed 7200-frame expiry, which is when the status system recreates it.

After the status becomes inactive, the processor still handles the transition while the previous stain amount was active or an ingestion timer was running. It sets a linked GameEffect's `frames` to `1` when its counter is above 1 or negative, and leaves a counter of `0` unchanged; the GameEffect updater performs final removal on a later `0 → -1` update. Once both current and previous stain amounts are below threshold and the timer has expired, the status processor skips that row until a source becomes active again. Skipping an already inactive row does not itself remove or reload an effect. At system level, an absent or empty status-component buffer means no periodic status pass; otherwise the processor runs for each enabled StatusEffectDataComponent. Its add callback skips a disabled component or an owner already entering removal. During a periodic pass, a component on an owner entering removal can still reach the processor, but the effect loader rejects a new effect for a target already being removed. A missing or unloadable effect template also prevents replacement. The manager-wide dispatch gate skips all systems together; its native write path was not found, so it is not established as an ordinary gameplay trigger.

The GameEffect update decrements a nonnegative `frames` counter once per invocation. A transition from one to zero still runs the effect-specific logic; the following invocation changes zero to negative, emits `game_effect_end`, and removes the effect entity (`0x00B015D0`). Adding an enabled GameEffect component to a live entity calls its updater immediately through the component-added callback (`0x00D8BC70`), which sets the invisibility bit and consumes one frame synchronously. GameEffectSystem also calls the updater once for each enabled component in its buffer on its normal pass; a component attached before that pass can therefore consume a second frame in the same world update. Status-driven replacements happen after the pass, so they receive only the synchronous update on that update. The system skips its loop when its component buffer is absent or empty and skips disabled components; it has no per-effect refresh cadence.

### How the stain supplies the source status

The stain system samples the current stain map on a cadence, after the status system has already processed that update. For an entity with the `player_unit` tag, the sample runs when the global frame is divisible by three or when `((frame ^ entity_id) & 0xF) == 0`. Other entities use only the second, entity-and-frame phase check. These skipped sample ticks do not clear the previous status amounts; the old values stay in place until a sample actually runs.

On a sample, the game clears the entity's stain-backed status amounts, counts each material's stained pixels among the opaque sprite pixels it visits, and divides by the opaque-pixel count. A material contributes that fraction to each of its configured status rows only when the fraction is strictly greater than the material's `liquid_sprite_stains_status_threshold`. A positive cooldown suppresses the contribution for that sample and is decremented instead.

The invisibility material does not set an explicit material threshold, so it uses the `CellData` default of `0.01`. That lets a small stain contribute a positive amount, but the status system activates the effect only when the amount reaches `0.15` or the separate ingestion-effect timer remains positive. With invisibility liquid as the only source, a sampled coverage above 1% but below 15% therefore leaves a positive stain amount that is still too small to keep invisibility active. The sprite overlay can be visible while the status effect is inactive.

Because stain sampling runs after status processing, the sample cadence can leave the status amount inactive through the status pass that observes the expired effect. If a later sample restores an active amount after that pass, AnimalAI gets one update with the effect absent; the next status pass recreates it and the add callback turns invisibility back on. A skipped sample alone does not clear the old status amount, though it can delay the next refresh after an earlier sample cleared it. This is a statically traced route to a one-update exposure window; no runtime capture establishes how often uninterrupted contact with vanilla invisibility liquid crosses the threshold, or whether an enemy fires during that window.

There is also a direct loading route separate from liquid status processing. `GetGameEffectLoadTo` returns an existing matching effect component without resetting its frame counter by default, including when that counter is `0`; a call before the GameEffect system removes it can therefore hand back the expiring component instead of loading a replacement. If the caller does not retry or reset its frames, this route can fail to renew the effect. The shipped streaming-event BERSERK action is a vanilla example that passes `false` and does not reset the returned counter; the other shipped `false` callers set frames after a successful return. The invisibility-machine script passes `true`, then sets the new component's `frames` to `-1`. `LoadGameEffectEntityTo` loads directly from a supplied entity path. The liquid-status route instead follows its status index and the configured 7200-frame effect entity.

This expiry path does not produce a full AnimalAI update without invisibility when the status remains active. Other systems ordered between GameEffectSystem and StatusEffectDataSystem could observe the short intra-update interval after removal; the polymorph guard in `0x00B041E0` tests for GameEffect id `0x55` while loading polymorph variants, except cessation. The status table processes `POLYMORPH` at row 12 before `PROTECTION_POLYMORPH` at row 48, so a co-active polymorph row can reach the guard before the expired protection row is recreated. This statically traced ordering matches the reported failure mechanism, though ordinary-gameplay frequency is not runtime-verified. The stain sampling path above provides a separate, statically traced way for the source row to become inactive, followed by a possible one-update AI exposure when a later sample restores it. The Lua API `EntityRemoveStainStatusEffect` can also explicitly clear a named stain status and set its cooldown; vanilla scripts use it for `PROTECTION_ALL`, not `INVISIBILITY`. The polymorph-immunity example in [NB-47](https://bugs.noita.app/bug/47) is configured as a status-backed GameEffect with a 7200-frame template. That does not establish that ordinary invisibility-liquid contact reaches an inactive status interval. No in-game capture yet establishes how often that happens or whether an enemy fires in the exposure window.

An inactive invisibility bit can also occur while the GameEffect entity remains present. The updater clears it if any other status is at least `0.15`, or if its runtime cooldown field remains positive. Although the shipped invisibility entity sets no initial cooldown, native listeners set one when an invisible entity receives `Message_Shot`, `Message_Kick`, `Message_MeleeOrDashAttack`, or qualifying incoming damage (`0x00B06870`, `0x00B06CA0`, `0x00B06CC0`, `0x00B06D20`). The `GAMEEFFECT_INVISIBILITY_SHOT_COOLDOWN_FRAMES` default is 300; the effect updater decrements the cooldown and keeps the invisibility bit off while it remains positive (`0x00B015D0`). This built-in reveal is separate from a stain-source gap. A matching effect at `frames=0` suppresses status-driven creation only while it remains attached when the status processor checks; in normal system order GameEffect removes it before the later status pass. An earlier status component callback can see the attached zero-frame component and skip loading, but that early return does not itself establish a full missed update if the normal later status pass still runs. Invalid effect paths/templates, a missing or disabled status component, and a target already being removed can prevent creation. The central manager's dispatch gate skips both systems together, so it does not independently create a countdown-only gap. These conditions are separate from the normal 7200-frame expiry path.

## Minimum entity setup

The engine needs:

- An enabled `SpriteStainsComponent`.
- A valid `SpriteComponent` selected by the stain component's `sprite_id`. This is a zero-based index into the entity's ordered sprite-component list; it defaults to `0`.
- A loaded sprite resource with frame data and a UV map that the stain system can load.

For one sprite, the component can be explicit:

```xml
<SpriteStainsComponent _enabled="1" sprite_id="0" />
```

`SpriteAnimatorComponent`, `HitboxComponent`, `CharacterDataComponent`, `StatusEffectDataComponent`, and the `mortal`/`hittable` tags are not prerequisites for building the sprite-stain map. The stain initializer resolves the selected `SpriteComponent` directly. It exits when that sprite resource or its frame data is unavailable, and stains cannot be added until the component's stain texture has initialized (`0x00c22080`, `0x00c23290`).

Each `SpriteStainsComponent` targets one sprite index. If an entity has two sprites and both should stain, give it two stain components, with `sprite_id="0"` and `sprite_id="1"`. Vanilla meatmaggot follows this pattern for its nine sprites (indices 0–8). The index refers to the final entity's ordered SpriteComponents, including inherited components; it is not an ID to guess. The engine returns no sprite for a negative or out-of-range index, or when the selected list entry is not active (`0x00c50680`).

The Lua function `EntityAddRandomStains(entity, material_type, amount)` is a useful direct check: it is a no-op when the entity has no `SpriteStainsComponent` (`0x007b52d0`). Use a valid material ID, such as `CellFactory_GetType("water")`, and a positive amount. The amount is a number of stain-cell placement attempts, not a pixel size. The selected sprite needs opaque pixels for the stain routine to land a cell (`0x00c23290`).

## UV-map location and filename

The game looks for generated sprite UV maps under:

```text
data/generated/sprite_uv_maps/
```

The map filename is based on the full virtual path of the sprite's underlying PNG: replace each `/` with `.` and retain the `.png` extension. For example:

```text
Sprite PNG:  mods/example/files/whatever.png
UV map:     data/generated/sprite_uv_maps/mods.example.files.whatever.png
```

Use the PNG path from the sprite definition's `filename`, not the entity XML path or the `SpriteComponent.image_file` path when that points to a sprite XML. The shipped player sprite is an example: its sprite definition names `data/enemies_gfx/player.png`, and the corresponding generated map is `data.enemies_gfx.player.png`.

The init-only `ModDevGenerateSpriteUVsForDirectory(directory_path, override_existing)` API generates maps for a directory. Its path must start with `mods/YOUR_MOD_HERE/` or `data/`; it is available during initialization in `noita_dev.exe`. Pass `true` for `override_existing` when an old map must be regenerated (`0x008317d0`).

## If the entity still receives no stains

Check these in order:

1. Confirm `_enabled` is not `0`.
2. Count the final entity's SpriteComponents, including those inherited through `<Base>`, and confirm `sprite_id` selects the intended entry. An extra decorative or emissive sprite can shift the index. Values `0` and `1` only select the first and second entries; to stain both, use a stain component for each index.
3. Confirm the selected `SpriteComponent` resolves to the sprite XML and that XML's `filename` points to the PNG whose UV map was generated.
4. Confirm the UV-map file exists at the derived path and matches the current PNG. Regenerate it with override enabled after changing the source image.
5. Test with `EntityAddRandomStains` and a known stain-capable material. If that works but contact with a particular liquid does not, the remaining issue is the liquid's stain configuration, not the entity's sprite setup.

The shader samples the UV map to find the reference-pose stain texture (`data/shaders/sprite_stains.frag`). The material XML field named `stainable` is a material property; it does not make an entity's sprite stainable.
