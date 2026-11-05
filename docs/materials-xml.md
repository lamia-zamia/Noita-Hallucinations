# Material XML

`data/materials.xml` - and any file a mod adds with `ModMaterialsFileAdd` - is a `<Materials>` root
holding `<CellData>`, `<CellDataChild>`, `<Reaction>` and `<ReqReaction>` elements. This page is the
reference for `<CellData>`, `<Reaction>`, and their graphics, status-effect, particle and explosion
configuration children. Types and defaults below are statically verified from the executable;
they have not been tested in a running game.

## Rules

- **Every attribute is optional.** It is read only if present; otherwise the constructor value
  below is kept.
- **The type is what the game parses, not what looks natural.** Several are surprising:
  - `lifetime` and `liquid_viscosity` are **integers** - `lifetime="4.0"` and `liquid_viscosity="0.9"`
    are not valid.
  - `liquid_sprite_stain_ignited_drop_chance` and `solid_on_collision_splash_power` are **floats**.
  - `gas_speed`, `gas_upwards_speed`, `gas_horizontal_speed`, `gas_downwards_speed` are **bytes**
    (0-255).
  - `cold_freezes_chance_rev` and `warmth_melts_chance_rev` are 16-bit; `liquid_sprite_stains_check_offset`
    is a signed byte.
  - `hp`, `fire_hp`, `durability` and the other whole-number fields are **signed** 32-bit ints, so
    `fire_hp="-1"` is legal and `fire_hp="10000000000"` overflows.
- **`wang_color` is hex ARGB.** It is read as a string and converted to a number, so a 6-digit value
  such as `720000` silently gets alpha `00` and will not match an opaque `FF720000` map pixel.
- Booleans are `0`/`1`.

## `<CellData>` attributes

| attribute | type | default |
|---|---|---|
| `always_ignites_damagemodel` | bool | `0` |
| `audio_is_soft` | bool | `0` |
| `audio_materialaudio_type` | enum | `NONE` |
| `audio_materialbreakaudio_type` | enum | `NONE` |
| `audio_physics_material_event` | string | `""` |
| `audio_physics_material_solid` | string | `""` |
| `audio_physics_material_wall` | string | `""` |
| `audio_size_multiplier` | float | `1` |
| `autoignition_temperature` | int | `100` |
| `burnable` | bool | `0` |
| `cell_holes_in_texture` | bool | `0` |
| `cell_type` | enum CELL_TYPE | `liquid` |
| `cold_freezes_chance_rev` | int16 | `100` |
| `cold_freezes_to_dont_do_reverse_reaction` | bool | `0` |
| `cold_freezes_to_material` | string | `""` |
| `convert_to_box2d_material` | string | `""` |
| `crackability` | int | `0` |
| `danger_fire` | bool | `0` |
| `danger_poison` | bool | `0` |
| `danger_radioactive` | bool | `0` |
| `danger_water` | bool | `0` |
| `density` | float | `1` |
| `durability` | int | `0` |
| `electrical_conductivity` | bool | `1` |
| `fire_hp` | int | `0` |
| `gas_downwards_speed` | uint8 (0-255) | `90` |
| `gas_horizontal_speed` | uint8 (0-255) | `100` |
| `gas_speed` | uint8 (0-255) | `50` |
| `gas_upwards_speed` | uint8 (0-255) | `100` |
| `generates_flames` | int | `30` |
| `generates_smoke` | int | `0` |
| `gfx_glow` | int | `0` |
| `gfx_glow_color` | string | `0` |
| `hp` | int | `100` |
| `ignore_self_reaction_warning` | bool | `0` |
| `is_just_particle_fx` | bool | `0` |
| `lifetime` | int | `0` |
| `liquid_damping` | float | `0.8` |
| `liquid_flow_speed` | float | `0.9` |
| `liquid_gravity` | float | `0.5` |
| `liquid_sand` | bool | `0` |
| `liquid_sand_never_box2d` | bool | `0` |
| `liquid_slime` | bool | `0` |
| `liquid_sprite_stain_ignited_drop_chance` | float | `10` |
| `liquid_sprite_stain_shaken_drop_chance` | float | `1` |
| `liquid_sprite_stains_check_offset` | int8 (-128..127) | `0` |
| `liquid_sprite_stains_status_threshold` | float | `0.01` |
| `liquid_stains` | int | `0` |
| `liquid_stains_custom_color` | string | `0` |
| `liquid_stains_self` | bool | `0` |
| `liquid_static` | bool | `0` |
| `liquid_sticks_to_ceiling` | int | `0` |
| `liquid_viscosity` | int | `50` |
| `name` | string | `""` |
| `on_fire` | bool | `0` |
| `on_fire_convert_to_material` | string | `""` |
| `on_fire_flame_material` | string | `fire` |
| `on_fire_smoke_material` | string | `smoke` |
| `platform_type` | int | `-1` |
| `requires_oxygen` | bool | `1` |
| `show_in_creative_mode` | bool | `0` |
| `slippery` | bool | `0` |
| `solid_break_to_type` | string | `""` |
| `solid_collide_with_self` | bool | `1` |
| `solid_friction` | float | `0.3` |
| `solid_go_through_sand` | bool | `0` |
| `solid_gravity_scale` | float | `1` |
| `solid_on_break_explode` | bool | `0` |
| `solid_on_collision_convert` | bool | `0` |
| `solid_on_collision_explode` | bool | `0` |
| `solid_on_collision_material` | string | `""` |
| `solid_on_collision_splash_power` | float | `1` |
| `solid_on_sleep_convert` | bool | `0` |
| `solid_restitution` | float | `0.2` |
| `solid_static_type` | int | `0` |
| `stainable` | bool | `1` |
| `status_effects` | string | `""` |
| `stickyness` | float | `0` |
| `tags` | string | `""` |
| `temperature_of_fire` | int | `10` |
| `ui_name` | string | `""` |
| `vegetation_full_lifetime_growth` | int | `10000` |
| `vegetation_random_flip_x_scale` | bool | `0` |
| `vegetation_sprite` | string | `""` |
| `wang_color` | hex colour | `0` |
| `wang_curvature` | float | `0.5` |
| `wang_noise_percent` | float | `1` |
| `wang_noise_type` | int | `0` |
| `warmth_melts_chance_rev` | int16 | `100` |
| `warmth_melts_to_material` | string | `""` |

`cell_type` is stored as an integer (1 `liquid`, 2 `gas`, 3 `solid`, 4 `fire`); the constructor value
1 is `liquid`. `<CellDataChild>` takes the same attributes plus `_parent` and `_inherit_reactions`.

These attributes appear in real material files but do not occur anywhere in the executable, so
they do nothing: `blob_radius3`, `liquid_solid`, `solid_break_on_explosion_rate`,
`supports_collapsible_structures`.

## `<Reaction>` attributes

The reader takes these nineteen attributes. The first two input and output material IDs start
at `-1`, represented as `unknown` when the reader prepares its fallback strings. Supply real
materials for a useful reaction. The optional third pair and cosmetic material use empty strings
instead. These are loader fallbacks, not four constructor-owned strings.

| attribute | type | default / fallback |
|---|---|---|
| `fast_reaction` | bool | `0` |
| `probability` | float | `0` |
| `input_cell1`, `input_cell2` | material string | `unknown` |
| `output_cell1`, `output_cell2` | material string | `unknown` |
| `input_cell3`, `output_cell3` | material string | `""` |
| `cosmetic_particle` | material string | `""` |
| `req_lifetime` | int | `0` |
| `blob_radius1`, `blob_radius2` | int, stored in a byte | `0` |
| `blob_restrict_to_input_material1`, `blob_restrict_to_input_material2` | bool | `1` |
| `destroy_horizontally_lonely_pixels` | bool | `0` |
| `direction` | enum: `none`, `top`, `bottom`, `left`, `right` | `none` |
| `convert_all` | bool | `0` |
| `entity` | entity path string | `""` |
| `audio_fx_volume_1` | float | `0` |

`blob_radius1` and `blob_radius2` are parsed as signed integers but only their low eight bits are
stored: `256` becomes `0`, and `-1` becomes `255`. Keep them in `0..255` to avoid wrapping.
`<Reaction>` also accepts an `<ExplosionConfig>` child.

## `<Graphics>`

```xml
<Graphics color="ff83bdff" texture_file="data/materials_gfx/water.png">
  <Edge>
    <EdgeGraphics type="NORMAL_BASED" z="1" percent="1">
      <Images>
        <Image filename="data/example_edge.png" min_angle="-22.5" max_angle="22.5" />
      </Images>
    </EdgeGraphics>
  </Edge>
</Graphics>
```

| attribute | type | constructor default |
|---|---|---|
| `texture_file` | string | `""` |
| `color` | hex ARGB | `0` |
| `fire_colors_index` | uint32 | `0` |
| `randomize_colors`, `is_grass`, `is_grass_hashed`, `normal_mapped` | bool | `0` |

The ten additional hex colour attributes are `pixel_lonely`, `pixel_top_left`, `pixel_top`,
`pixel_top_right`, `pixel_right`, `pixel_bottom_right`, `pixel_bottom`, `pixel_bottom_left`,
`pixel_left` and `pixel_all_around`. They are separate values, not child elements.
After loading graphics, a main `color` value of zero falls back to the material's `wang_color`.

`<Graphics>` accepts multiple `<Edge>` elements, each containing a list of `<EdgeGraphics>`
elements. An edge rule may omit `<Images>`.

| `<EdgeGraphics>` attribute | type | default |
|---|---|---|
| `type` | enum: `COLOR_EDGE_PIXELS`, `EVERYWHERE`, `CARDINAL_DIRECTIONS`, `NORMAL_BASED` | `COLOR_EDGE_PIXELS` |
| `z` | float | `1` |
| `percent` | float | `1` |
| `require_same_material` | bool | `1` |
| `require_same_material_type` | bool | `0` |
| `overwrite` | bool | `0` |
| `color` | hex ARGB | `ff233112` |

The meaning of the unusual default edge colour has not been traced into rendering.
`<Images>` contains a list of `<Image>` entries:

| `<Image>` attribute | type | default |
|---|---|---|
| `filename` | string | `""` |
| `min_angle`, `max_angle` | float, degrees | `0` |
| `do_only_horizontal_stripe`, `do_only_vertical_stripe`, `allow_random_rotation` | bool | `0` |

Angles may be fractional. They are converted from degrees to radians when read.

## `<StatusEffects>`

```xml
<StatusEffects>
  <Stains><StatusEffect type="WET" amount="0.5" /></Stains>
  <Ingestion><StatusEffect type="POISON" amount="1" /></Ingestion>
</StatusEffects>
```

Both `<Stains>` and `<Ingestion>` contain a list of `<StatusEffect>` entries and use the same
reader. Each entry accepts `type` (a status-effect name) and `amount` (float, default `1`).
`amount` is accepted for stains too. Its effect on each kind of status has not been traced here.
Empty containers are allowed. The initial type ID is `0`; this page does not assign it a name.

## `<ParticleEffect>`

This child uses the game's `CosmeticParticleConfig` record, with seventeen fields expanded into
twenty-four XML attributes. Vector and range members use dotted attribute names:

| attribute | type | constructor default |
|---|---|---|
| `m_material_id` | int | `1` |
| `vel.x`, `vel.y` | float | `0`, `0` |
| `vel_random.min_x`, `vel_random.min_y`, `vel_random.max_x`, `vel_random.max_y` | float | all `0` |
| `color` | hex ARGB | `0` |
| `lifetime.min`, `lifetime.max` | float | `0.5`, `1` |
| `gravity.x`, `gravity.y` | float | `0`, `100` |
| `cosmetic_force_create`, `render_back`, `render_on_grid`, `draw_as_long` | bool | `0` |
| `airflow_force`, `airflow_scale`, `friction` | float | `0` |
| `probability` | float | `1` |
| `count.min`, `count.max` | int | `1`, `1` |
| `particle_single_width`, `fade_based_on_lifetime` | bool | `1` |

Although `color` is stored as a `uint32`, its XML reader parses a hexadecimal string, not a decimal
integer. For example, `color="fffa8f2c"` is valid. The parser swaps red and blue bytes for internal
storage; write the XML colour in ARGB order. Zero keeps the material's normal colour rather than
requesting an override.

## `<ExplosionConfig>`

Materials and reactions use the shared `ConfigExplosion` configuration, including its nested
`damage_critical` record. One easily missed attribute is `not_scaled_by_gamefx` (bool, default
`0`). Setting it to `1` excludes the explosion from `DAMAGE_MULTIPLIER` and related game-effect
scaling, according to its embedded field description. The rest of that record's defaults are not
enumerated on this page.

## Evidence

| claim | address |
|---|---|
| `<CellData>` reader: 87 attributes via the generic read path, plus `liquid_stains_custom_color` (hex) and the two audio enums via helpers | `0x006fa4e0` |
| `<CellData>` constructor: every default above, read from the disassembly (several byte and short fields share one `dword` store, e.g. the four gas speeds are `0x5A646432`) | `0x004ac7e0` |
| byte/short types: RTTI checks `.E` (unsigned char), `.F` (short), `.C` (signed char) | `0x00902980`, `0x0095b190`, `0x009028f0` |
| `cell_type` names | `0x004acce0` |
| shared hex colour parser: hex input, then red/blue byte swap | `0x0044ed00` |
| `<Reaction>` reader and constructor; material-name fallback | `0x00701d60`, `0x004ac6f0`, `0x00704e70` |
| `<Materials>` reader: `CellData`, `CellDataChild`, `Reaction`, `ReqReaction`; `_parent`/`_inherit_reactions` | `0x00707c20`, `0x007074f0` |
| `<Graphics>` reader and constructor; colour attribute helper | `0x00700fb0`, `0x004ace00`, `0x00701c20` |
| `<Edge>` list wrapper; `<EdgeGraphics>` reader, initializer and enum names | `0x00700ea0`, `0x007001f0`, `0x008a6348`, `0x004ac580` |
| `<Image>` reader and list-entry initializer | `0x006ff950`, `0x0099fb9e` |
| `<StatusEffects>` reader, shared entry reader and initializer | `0x006f9c60`, `0x006f9930`, `0x0099f876` |
| `CosmeticParticleConfig` constructor and field registration; XML serializer; colour callback | `0x005b0210`, `0x005b1670`, `0x00815df0` |
| `ConfigExplosion` constructor and field registration, including `not_scaled_by_gamefx` | `0x004b2780` |
