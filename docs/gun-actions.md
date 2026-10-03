# `RegisterGunAction` and the wand-action record

Every wand card in Noita — the fireball, the light bullet, the "unlimited" wand's bonus cards — is
one record of 65 fields. `RegisterGunAction` is the function that writes one. It is not one of the
375 documented API functions: it is installed by one of the secondary registrars (`0x00b294a0`),
which sets `RegisterGunAction` as a plain global alongside the `GunSystem` table.

This page is the reference for what those 65 arguments do, what they default to, and where the
traps are.

## What the function actually does

```c
FUN_00b27ff0(L)                      // the Lua cfunction
  FUN_009d4bb0(&info, L)             // read all 65 Lua arguments into a ConfigGunActionInfo
  append(&DAT_012093b4, &info)       // per-shot scratch list
  if (DAT_012093b0) insert_or_overwrite(map_at_0x012093cc, info.action_id, info)
```

`ConfigGunActionInfo` is 572 bytes (`0x23c`) and every field is registered with the engine's
reflection system under the same 65 names. Three consequences:

- **It is a global registry, not per-wand state.** Registering is permanent for the process.
- **Lookup is by `action_id` in a `std::map`** (`0x00b2bf30`, a red-black-tree `lower_bound`, not a
  hash). Fifteen call sites use it, all passing the wand component's id string.
- **A duplicate `action_id` silently overwrites.** The existing map node's value is overwritten in
  place, so the *last* registration of an id wins and the earlier one is gone. No error, no log.
  (The per-shot vector is appended unconditionally instead, so duplicates coexist there; only
  `SetProjectileConfigs` reads that vector, and it reads the *last* element.)

Argument handling is the permissive kind described in
[how-the-api-works.md](how-the-api-works.md), with three specifics:

- **No arity check.** Each argument is `lua_type(L, i)`; absent or `nil` means "use the default".
  Extra arguments past 65 are ignored. Unlike some other registrars in the binary, this one does
  not log "expected N parameters".
- **Strings are read with `lua_tolstring`**, so a number is accepted and stringified. Anything that
  is not a string or number (a table, a boolean) becomes `""`, not an error.
- **A lookup miss is silent.** If nothing was ever registered under an id, the lookup returns a
  **default-constructed record** — all defaults, empty id — and the shot proceeds with it.

### The three globals, and what the second registrar is for

| global | what it is |
|--------|-----------|
| `0x012093b4` | `vector<ConfigGunActionInfo>`, 0x23c stride. Cleared at the start of every draw/shot pass, so it is scratch space, not the registry. |
| `0x012093b0` | one byte: "collect metadata" flag. |
| `0x012093cc` | `std::map<action_id, ConfigGunActionInfo>` — the real registry. Node layout: `+0x10` key string, `+0x28` value. |

The other of the two gun registrars (`0x00b2bcc0`) installs `RegisterGunAction` again, together
with `Reflection_RegisterProjectile` (`0x00b27f70`, which just stores a string), and then runs
`data/scripts/gun/gun_collect_metadata.lua`. That script is a dozen lines: it loads `gun.lua`, sets
`reflecting = true`, walks every entry of the action table calling `create_shot()`,
`set_current_action()`, the action's own closure and `register_action()` for each one, then clears
the flag. Because the flag is on for that pass, every registration also lands in the map. So the
registry is *built* by replaying every action once at startup, and `RegisterGunAction` is how you
add to it.

## The 65 arguments

`off` is the byte offset inside `ConfigGunActionInfo`. The defaults are the constructor's
(`0x004c7b00`) and match `ConfigGunActionInfo_Init` in `data/scripts/gun/gunaction_generated.lua`
line for line — that generated file is the same struct, written out as Lua.

The *types*, *defaults* and *offsets* below are read out of the executable. The *effects* are not:
the code that applies the numbers was not located (see [what is not known](#what-is-not-known)), so
each effect is the field's name plus how the shipped action files use it.

| # | argument | off | read as / default | what it does |
|---|----------|-----|-------------------|--------------|
| 1 | `action_id` | +0x04 | string `""` | the registry key, and the id the rest of the gun API uses (`AddGunAction`, `OnActionPlayed`, …) |
| 2 | `action_name` | +0x1c | string `""` | card title; a `$`-prefixed string is a translation key (`$action_bomb`) |
| 3 | `action_description` | +0x34 | string `""` | tooltip; `$actiondesc_bomb` |
| 4 | `action_sprite_filename` | +0x4c | string `""` | card icon |
| 5 | `action_unidentified_sprite_filename` | +0x64 | string `data/ui_gfx/gun_actions/unidentified.png` | icon shown while the spell is unidentified — the only argument with a non-empty default |
| 6 | `action_type` | +0x7c | int `0` | see the enum below |
| 7 | `action_spawn_level` | +0x80 | string `""` | comma-separated wand levels `0,1,2,…` in which procedural generation may roll this card |
| 8 | `action_spawn_probability` | +0x98 | string `""` | comma-separated weight per level, same indexing |
| 9 | `action_spawn_requires_flag` | +0xb0 | string `""` | a stats flag that must be set first, e.g. `"card_unlocked_black_hole"` |
| 10 | `action_spawn_manual_unlock` | +0xc8 | bool `false` | keep it out of random rolls; only obtainable by explicit unlock |
| 11 | `action_max_uses` | +0xcc | int `-1` | charges; `-1` is unlimited |
| 12 | `custom_xml_file` | +0xd0 | string `""` | card entity XML, in practice `data/entities/misc/custom_cards/*.xml`, carrying the card's extra entity and component configuration |
| 13 | `action_mana_drain` | +0xe8 | number `10.0` | mana cost |
| 14 | `action_is_dangerous_blast` | +0xec | bool `false` | marks the shot as a blast; used by the game's own scripted and enemy wand sets |
| 15 | `action_draw_many_count` | +0xf0 | int `0` | for `ACTION_TYPE_DRAW_MANY`: how many extra cards to draw |
| 16 | `action_ai_never_uses` | +0xf4 | bool `false` | exclude from AI-generated wands |
| 17 | `action_never_unlimited` | +0xf5 | bool `false` | exclude from the "unlimited" wand action sets |
| 18 | `state_shuffled` | +0xf6 | bool `false` | engine wand state: the deck was shuffled this cycle |
| 19 | `state_cards_drawn` | +0xf8 | int `0` | engine wand state: cards drawn counter |
| 20 | `state_discarded_action` | +0xfc | bool `false` | engine wand state: an action was discarded |
| 21 | `state_destroyed_action` | +0xfd | bool `false` | engine wand state: an action was destroyed |
| 22 | `fire_rate_wait` | +0x100 | number → **int field** `0` | extra frames added to this action's re-fire lock; vanilla uses +2 … +20 |
| 23 | `speed_multiplier` | +0x104 | number `1.0` | projectile speed scale |
| 24 | `child_speed_multiplier` | +0x108 | number `1.0` | speed scale for projectiles spawned by this projectile's children |
| 25 | `dampening` | +0x10c | number `1.0` | projectile velocity damping scale |
| 26 | `explosion_radius` | +0x110 | number `0` | explosion radius applied when the projectile dies |
| 27 | `spread_degrees` | +0x114 | number `0` | random spread cone, in degrees |
| 28 | `pattern_degrees` | +0x118 | number `0` | angular step of the multi-projectile pattern |
| 29 | `screenshake` | +0x11c | number `0` | camera shake amount |
| 30 | `recoil` | +0x120 | number `0` | shooter kickback |
| 31 | `damage_melee_add` | +0x124 | **integer → float field** `0` | melee damage addend — see the trap below |
| 32 | `damage_projectile_add` | +0x128 | number `0` | projectile damage addend |
| 33 | `damage_electricity_add` | +0x12c | number `0` | electricity damage addend |
| 34 | `damage_fire_add` | +0x130 | number `0` | fire damage addend |
| 35 | `damage_explosion_add` | +0x134 | number `0` | explosion damage addend |
| 36 | `damage_ice_add` | +0x138 | number `0` | ice damage addend |
| 37 | `damage_slice_add` | +0x13c | number `0` | slice damage addend |
| 38 | `damage_healing_add` | +0x140 | number `0` | healing addend |
| 39 | `damage_curse_add` | +0x144 | number `0` | curse addend |
| 40 | `damage_drill_add` | +0x148 | number `0` | drill addend |
| 41 | `damage_null_all` | +0x14c | number `0` | null-damage addend |
| 42 | `damage_critical_chance` | +0x150 | int `0` | critical chance in percent; vanilla uses 5, 40 |
| 43 | `damage_critical_multiplier` | +0x154 | number `0` | critical damage scale |
| 44 | `explosion_damage_to_materials` | +0x158 | number `0` | explosion damage dealt to terrain pixels; vanilla uses `+300000` to excavate |
| 45 | `knockback_force` | +0x15c | number `0` | knockback applied on impact |
| 46 | `reload_time` | +0x160 | int `0` | per-action reload override in frames. The wand-level value lives in `<gun_config reload_time>` and reaches Lua through `StartReload` |
| 47 | `lightning_count` | +0x164 | int `0` | number of lightning arcs to spawn |
| 48 | `material` | +0x168 | string `""` | material name spawned by the action, e.g. `"fire"`, `"gunpowder_unstable"` |
| 49 | `material_amount` | +0x180 | int `0` | how much of it |
| 50 | `trail_material` | +0x184 | string `""` | trail material name |
| 51 | `trail_material_amount` | +0x19c | int `0` | trail material amount |
| 52 | `bounces` | +0x1a0 | int `0` | projectile bounce count |
| 53 | `gravity` | +0x1a4 | number `0` | gravity scale applied to the projectile |
| 54 | `light` | +0x1a8 | number `0` | light emitted |
| 55 | `blood_count_multiplier` | +0x1ac | number `1.0` | blood particle multiplier |
| 56 | `gore_particles` | +0x1b0 | int `0` | extra gore particles spawned on hit |
| 57 | `ragdoll_fx` | +0x1b4 | int `0` | ragdoll effect level; vanilla uses 2 and 3 |
| 58 | `friendly_fire` | +0x1b8 | bool `false` | let the projectile hit its own caster |
| 59 | `physics_impulse_coeff` | +0x1bc | number `0` | physical impulse coefficient on impact |
| 60 | `lifetime_add` | +0x1c0 | int `0` | added projectile lifetime in frames |
| 61 | `sprite` | +0x1c4 | string `""` | sprite override for the spawned projectile or card |
| 62 | `extra_entities` | +0x1dc | string `""` | comma-separated entity XML paths attached to the projectile |
| 63 | `game_effect_entities` | +0x1f4 | string `""` | comma-separated status-effect entity XMLs (`effect_frozen.xml`, `effect_blindness.xml`, …) |
| 64 | `sound_loop_tag` | +0x20c | string `""` | tag for the looping shot sound, e.g. `"sound_digger"` |
| 65 | `projectile_file` | +0x224 | string `""` | projectile entity XML this action belongs to |

### `action_type`

The values are game data, in `data/scripts/gun/gun_enums.lua`, which notes they mirror
`GunActionType` in `gun_component.h`:

| value | name | meaning |
|-------|------|---------|
| 0 | `ACTION_TYPE_PROJECTILE` | spawns a projectile |
| 1 | `ACTION_TYPE_STATIC_PROJECTILE` | spawns a projectile that does not move |
| 2 | `ACTION_TYPE_MODIFIER` | only changes the action record |
| 3 | `ACTION_TYPE_DRAW_MANY` | draws extra cards; see `action_draw_many_count` |
| 4 | `ACTION_TYPE_MATERIAL` | spawns material |
| 5 | `ACTION_TYPE_OTHER` | — |
| 6 | `ACTION_TYPE_UTILITY` | — |
| 7 | `ACTION_TYPE_PASSIVE` | — |

`gun.lua` treats 0, 1 and 4 as "this action produced projectiles" and runs the extra modifiers for
them.

## Writing an action: the short keys

Nobody calls `RegisterGunAction` with 65 positional arguments in practice, including the game. An
action is a table with short keys, and `data/scripts/gun/gun.lua` copies them onto the record:

```lua
{
    id          = "BOMB",
    name        = "$action_bomb",
    description = "$actiondesc_bomb",
    sprite      = "data/ui_gfx/gun_actions/bomb.png",
    type        = ACTION_TYPE_PROJECTILE,
    spawn_level       = "0,1,2,3,4,5,6",
    spawn_probability = "1,1,1,1,0.5,0.5,0.1",
    price = 200,
    mana  = 25,
    max_uses = 3,
    custom_xml_file = "data/entities/misc/custom_cards/bomb.xml",
    action = function()
        add_projectile("data/entities/projectiles/bomb.xml")
        c.fire_rate_wait = c.fire_rate_wait + 100
    end,
}
```

The keys that `set_current_action` forwards are `id`, `name`, `description`, `sprite`,
`sprite_unidentified`, `type`, `recursive`, `spawn_level`, `spawn_probability`,
`spawn_requires_flag`, `spawn_manual_unlock`, `max_uses`, `custom_xml_file`, `mana`,
`is_dangerous_blast`, `ai_never_uses`, `never_unlimited`, `sound_loop_tag`. Everything else is
written by the action's own closure onto the `c` table, whose field names are exactly the 65
argument names.

So the practical loop is: mutate `c.<field>`, and the whole table is re-registered once per shot by
`register_action`. Fields you do not touch keep the value the constructor gave them, which is why
`c.fire_rate_wait = c.fire_rate_wait + 100` accumulates across modifiers instead of replacing.

## The XML form

The same 65 names are the attributes of a `<gunaction_config>` element inside a wand's
`<AbilityComponent>`, so a wand XML can preset any of them:

```xml
<AbilityComponent use_gun_script="1" ...>
  <gun_config actions_per_round="1" deck_capacity="7" reload_time="27" shuffle_deck_when_empty="1">
  </gun_config>
  <gunaction_config action_mana_drain="10" fire_rate_wait="2" speed_multiplier="1.03398" ... >
  </gunaction_config>
</AbilityComponent>
```

102 shipped files contain this element. Note that all of them also set five attribute names the
engine does not know — `damage_melee`, `damage_projectile`, `damage_electricity`, `damage_fire`,
`damage_explosion` — which are the pre-rename spellings of the five `_add` fields. They are
ignored. If you are writing a wand XML by hand, use `damage_projectile_add`.

## Traps

1. **`damage_melee_add` cannot hold a fraction.** The argument is read with `lua_tointeger`, but
   the field is a float and the mirror function (`0x009d5c60`) pushes it back with
   `lua_pushnumber`. An integer bit pattern ends up in a float slot, so `2.7` becomes
   approximately zero. Whole numbers only.
2. **`fire_rate_wait` is an integer field** — the mirror function reads it with
   `lua_pushinteger` — but the argument is read with `lua_tonumber`. A fractional value stores
   float bits into an int slot and comes back as garbage. Keep it a whole number of frames.
3. **Duplicate ids do not warn.** Register the same `action_id` from two mods and the second one
   silently replaces the first.
4. **An unknown id is not an error.** You get a record of defaults and a wand that fires nothing.
5. **Registration is global and permanent.** There is no unregister, and nothing is scoped to a
   wand or a run.
6. **The damage field names all end in `_add`.** `c.damage_projectile` is not a field at all: it
   is `nil`, so `c.damage_projectile = c.damage_projectile + 0.4` raises a Lua error. The gun
   callbacks are invoked inside `lua_pcall` (with the engine's error handler), so the game survives
   — but the rest of that action's closure is skipped too, and the game carries on.

## Data-file traps found while doing this

`data/scripts/gun/` ships four action lists that **nothing loads**: `gun_actions_unlimited.lua`,
`gun_actions_limited.lua`, `gun_actions_petri.lua` and `_gun_actions_unlimited.lua`. `gun.lua`
loads only `gun_actions.lua` and `gun_extra_modifiers.lua`. The four dead files still use the
pre-rename `c.damage_projectile` / `c.damage_explosion` / `c.damage_electricity` names in 17 live
places each. `gun_actions.lua` itself is clean — its six occurrences of the old names are all inside
commented-out blocks.

If you are copying an action out of one of those files, the damage line will not work.

## What is not known

- **Where the numeric fields are finally interpreted.** After registration the record is copied
  whole — into a `ProjectileConfigEntry` (0x25c stride) by `SetProjectileConfigs`, then into a
  0x278-byte accumulator by the shot-bytecode walker `0x00b28ee0` (opcode 6), then consumed by the
  shot executor `0x00b2a310`. No direct `[reg+offset]` read of the individual fields was found in
  the gun and projectile code (`0x00b0xxxx`–`0x00b9xxxx`), and the only reference to
  `projectile_file` outside the reflection cluster is a debug logger. The effects in the table
  above therefore come from the field names, the generated config file and how the vanilla action
  files use them, not from reading the arithmetic that applies them. `speed_multiplier` being a
  multiplier rather than an absolute speed is the one inference that could still be wrong in detail.
- **`action_draw_many_count`, `physics_impulse_coeff` and `blood_count_multiplier`** are never set
  by any shipped action file, so their exact units are inferred from the names alone.
- The four `state_*` fields are engine-owned wand state (they appear in wand XMLs alongside the
  action fields and are mirrored by the globals of the same name at the top of `gun.lua`), but the
  code that writes them into the record was not located.