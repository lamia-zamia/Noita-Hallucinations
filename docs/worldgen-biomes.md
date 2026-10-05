# Biomes: the data model

A biome is one XML file, `data/biome/NAME.xml`, with exactly two children: a `<Topology>`
(how the shape is generated) and a `<Materials>` (what is in it). This page covers the
structure, every field with its compiled-in default, and how a biome gets chosen for a pixel.

For the shape algorithms see [worldgen-terrain.md](worldgen-terrain.md); for the tile system see
[worldgen-wang.md](worldgen-wang.md). The raw offset tables are collected in
[worldgen-reference.md](worldgen-reference.md).

## The file

```xml
<Biome>
  <Topology name="$biome_coalmine" type="BIOME_WANG_TILE"
            background_image="..." wang_template_file="data/wang_tiles/coalmine.png"
            lua_script="data/scripts/biomes/coalmine.lua"
            wang_map_width="256" wang_map_height="256" ...>
    <RandomMaterials>
      <RandomColor input_color="FF00BBEE" output_colors="FF000000,FFFFFFFF"/>
    </RandomMaterials>
    <BitmapCaves size_x="512" size_y="256" cave_count_min="2" cave_count_max="2" ...>
      <structures>
        <CaveStructure image_file="data/biome_impl/coalmine/dangerroom.png"
                       aabb_min_x="5" aabb_max_x="507" aabb_min_y="0" aabb_max_y="230"
                       count_min="2" count_max="4"
                       strength_min="1.45" strength_max="1.55"/>
      </structures>
    </BitmapCaves>
  </Topology>

  <Materials name="coalmine">
    <MaterialComponent material_name="coal" material_index="10"
                       material_min="0.51" material_max="0.55"
                       is_rare="1" rare_use_polka="1" rare_use_perlin="1" .../>
    <VegetationComponent tree_image_file="data/vegetation/vine_growth_1.xml"
                         tree_material="ceiling_plant_material" is_ceiling_plant="1" .../>
    <FossilComponent .../>
  </Materials>
</Biome>
```

Five element types matter:

| element | lives in | what it controls | page |
|---|---|---|---|
| `<Topology>` | `Biome + 0x08` … `+0x2F0` | the shape: which generator, backgrounds, audio, wang template, the Lua hook | this page + terrain |
| `<BitmapCaves>` | its own object, reached through the pointer at `Biome + 0xC0` | procedural cave/mountain/blob carving | terrain |
| `<MaterialComponent>` | flat array at `BiomeMaterials + 0x08`, 0x84 bytes each | **which material a pixel with value *v* becomes** | terrain |
| `<VegetationComponent>` | flat array, 0xE0 bytes each | trees, grass, ceiling plants | pixelscenes |
| `<FossilComponent>` | flat array, 0x48 bytes each | fossil/ore blobs stamped as images | terrain |

There are 150 shipped biome XML files in `data/biome/`, plus `data/biome/orbrooms/` and
`data/biome/tower/` for the pixel-scene-only rooms, and `data/biome_impl/<biome>/` for the
per-biome images, wang extras and Lua.

## How a pixel picks its biome

1. `x >> 9`, `y >> 9` gives the chunk (512 px each).
2. The chunk indexes the biome map at `map[chunks_wide * chunk_y + chunk_x]`, a `uint32`
   colour. The chunk index is `coord >> 9`; its **X is wrapped by `% chunks_wide`** and its
   **Y is clamped** to the last row. There is no 512-chunk torus on this array.
3. The colour is looked up in the table from `_biomes_all.xml`, giving a `Biome*`.
4. Before step 1 the world position is shifted: x gets half the map width in pixels (35 x 512) and
   y gets `biome_offset_y` x 512 (14 x 512 = 7168 in the shipped file), so the map covers
   x in [-17920, +17920) and y in [-7168, +17408).

The lookup itself never fails for position alone: X wraps modulo the map width and Y is clamped to
the first or last row, so a position outside the 70 x 48 window reuses the nearest edge row (or the
wrapped column). A chunk is left without a biome only when its map pixel's colour is not in the
table; that case logs `Biome - couldn't find a biome at: <x>, <y>`.

## The `Biome` record

Offsets and defaults below are **not** inferred. Two functions in the executable give them
directly: one is a binder that passes the field name together with `this + <byte offset>` for
every registered field, and the other is the constructor that writes every field's compiled-in
default.

Six names are **nested child elements with no slot of their own** - they are containers, not
fields:

`BitmapCaves`, `RandomMaterials`, `modifiers`, `bitmap_data`, `mBitmapNoise`,
`mBiomeMaterials`

`modifiers` is a `BiomeModifiers` sub-object at `+0x170`; the other five are reached through a
pointer rather than living inline.

### Fields

| offset | field | binder index | compiled-in default |
|---|---|---|---|
| `+0x020` | `background_image` | 2 | "" |
| `+0x038` | `background_edge_left` | 3 | "" |
| `+0x050` | `background_edge_right` | 4 | "" |
| `+0x068` | `background_edge_top` | 5 | "" |
| `+0x080` | `background_edge_bottom` | 6 | "" |
| `+0x098` | `background_edge_priority` | 7 | `0` |
| `+0x09c` | `background_use_neighbor` | 8 | `false` |
| `+0x0a0` | `background_image_height` | 9 | `225.0` |
| `+0x0a4` | `limit_background_image` | 10 | `true` |
| `+0x0a8` | `bitmap_noise_file` | 11 | "" |
| `+0x0c4` | `noise_biome_edges` | 13 | `true` _(shares one store with `big_noise_biome_edges`, `fat_biome_edges`, `skip_edge_textures`)_ |
| `+0x0c5` | `big_noise_biome_edges` | 14 | `true` |
| `+0x0c6` | `fat_biome_edges` | 15 | `false` |
| `+0x0c7` | `skip_edge_textures` | 16 | `false` |
| `+0x0c8` | `audio_music_enter` | 17 | "" |
| `+0x0e0` | `audio_music_2` | 18 | "" |
| `+0x0f8` | `audio_music_energy_coeff` | 19 | `1.0` |
| `+0x0fc` | `audio_music_no_forced_quietness` | 20 | `false` |
| `+0x100` | `audio_music_forced_quietness_duration_seconds` | 21 | `0.0` |
| `+0x104` | `audio_music_trigger_without_danger` | 22 | `false` |
| `+0x108` | `audio_music_trigger_min_y` | 23 | `0.0` |
| `+0x10c` | `audio_music_trigger_max_y` | 24 | `0.0` |
| `+0x110` | `audio_biome_id_for_music` | 25 | "" |
| `+0x128` | `audio_ambience` | 26 | `"cave"` |
| `+0x140` | `audio_ambience_surface` | 27 | "hills" |
| `+0x158` | `color_grading_r` | 28 | `1.0` |
| `+0x15c` | `color_grading_g` | 29 | `1.0` |
| `+0x160` | `color_grading_b` | 30 | `1.0` |
| `+0x164` | `color_grading_grayscale` | 31 | `0.0` |
| `+0x168` | `has_rain` | 32 | `true` |
| `+0x188` | `gradient_sky_alpha_target_mod` | 35 | `0.0` |
| `+0x18c` | `rain_target_mod` | 36 | `0.0` |
| `+0x190` | `fog_target_mod` | 37 | `0.0` |
| `+0x194` | `sunset_alpha_target` | 38 | `1.0` |
| `+0x198` | `bitmap_filename` | 39 | "" |
| `+0x1b4` | `wang_template_file` | 41 | "" |
| `+0x1cc` | `wang_map_width` | 42 | `256` |
| `+0x1d0` | `wang_map_height` | 43 | `256` |
| `+0x1d8` | `lua_script` | 45 | "" |
| `+0x1f0` | `pixel_scene` | 46 | `false` |
| `+0x1f8` | `static_tile` | 48 | `false` |
| `+0x1fc` | `static_tile_bg_mask` | 49 | "" |
| `+0x214` | `static_tile_bg_mask_threshold` | 50 | `0.5` |
| `+0x218` | `game_enemy_hp_scale` | 51 | `1.0` |
| `+0x21c` | `game_enemy_attack_speed` | 52 | `1.0` |
| `+0x228` | `mInsideNoiseFBM` | 55 | `true` |
| `+0x230` | `mInsidePerlinScaleX` | 56 | `1.0` |
| `+0x238` | `mInsidePerlinScaleY` | 57 | `1.0` |
| `+0x240` | `mInsidePerlinSquared` | 58 | `false` _(shares one store with `mInsidePerlinClamped`)_ |
| `+0x241` | `mInsidePerlinClamped` | 59 | `false` |
| `+0x242` | `mInsidePerlinScaled` | 60 | `false` |
| `+0x244` | `mInsidePerlinScaleMin` | 61 | `0.0` |
| `+0x248` | `mInsidePerlinScaleMax` | 62 | `0.0` |
| `+0x24c` | `mInsidePerlinForceInside` | 63 | `false` _(shares one store with `mInsidePerlinOffsetBySeed`)_ |
| `+0x24d` | `mInsidePerlinOffsetBySeed` | 64 | `false` |
| `+0x250` | `mInsidePerlinOffsetX` | 65 | `0.0` |
| `+0x258` | `mInsidePerlinOffsetY` | 66 | `0.0` |
| `+0x260` | `mInsideAddValue` | 67 | `0.0` |
| `+0x264` | `mMultiplierGradient` | 68 | `1.0` |
| `+0x268` | `mMultiplierPerlin` | 69 | `1.0` |
| `+0x26c` | `mMultiplierExtraPerlin` | 70 | `1.0` |
| `+0x270` | `mGradientStartY` | 71 | `-512.0` |
| `+0x274` | `mGradientEndY` | 72 | `512.0` |
| `+0x278` | `mGradientSlopeStartX` | 73 | `0.0` |
| `+0x280` | `mGradientSlopeDelta` | 74 | `1.0` |
| `+0x288` | `mGradientAddNoise` | 75 | `1` |
| `+0x290` | `mGradientNoiseScale` | 76 | `0.01` |
| `+0x298` | `mGradientLowNoise` | 77 | `-100.0` |
| `+0x29c` | `mGradientHighNoise` | 78 | `100.0` |
| `+0x2a0` | `coarse_map_force_terrain` | 79 | `false` _(shares one store with `coarse_map_not_terrain`, `coarse_map_cell_count_always_zero`, `coarse_map_inject_light`)_ |
| `+0x2a1` | `coarse_map_not_terrain` | 80 | `false` |
| `+0x2a2` | `coarse_map_cell_count_always_zero` | 81 | `false` |
| `+0x2a3` | `coarse_map_inject_light` | 82 | `false` |
| `+0x2a8` | `mModifierUIDescription` | 84 | "" |
| `+0x2c0` | `mModifierUIDecorationFile` | 85 | "" |
| `+0x2d8` | `mDebugFilename` | 86 | "" |

Rows marked *see note* or `*` are covered by the notes below; the constructor writes them
with a wider store than the field size.

**`+0xC4..+0xC7` are four bools written by one dword.** The store is `0x101` at `+0xC4`, which
little-endian is `C4=1, C5=1, C6=0, C7=0`:

| offset | field | default |
|---|---|---|
| `+0xC4` | `noise_biome_edges` | **`true`** |
| `+0xC5` | `big_noise_biome_edges` | **`true`** |
| `+0xC6` | `fat_biome_edges` | `false` |
| `+0xC7` | `skip_edge_textures` | `false` |

**`+0x230`, `+0x238`, `+0x250`, `+0x258`, `+0x278`, `+0x280` and `+0x290` are 8-byte `double`
fields**, not merged pairs of two float fields. Each is written by one `mov qword`, and the
constants settle it: both `_DAT_010545e0` and `_DAT_010545e8` are `{0x00000000, 0x3ff00000}`, i.e.
the double **1.0**, while `DAT_01053790` is the double **0.01**. The other four default to `0.0`.

**`+0x220` and `+0x224` have no name in the binder.** They are enums, registered through a
different path, and are stored as the integers `0` and `5`. `+0x220` is `noise_type`
(0 `IQ2_SIMPLEX1234`, 1 `IQ_SIMPLEX`, 2 `SIN_CAPPED_EVERYTHING`, 3 `SIN_CAPPED_SIMPLEX`; default 0)
and `+0x224` is `mInsideNoiseType`, a `GENERAL_NOISE` enum (0 `IQNoise`, 1 `DirtyPeeNoise`,
2 `QemNoise`, 3 `WhiteNoise`, 4 `MixNoise`, 5 `SimplexNoise`, 6 `STB_Perlin`, 7 `FastBlockNoise`,
8 `SimplexNoise1234`; default 5).

### Slots with no registered name

These are constructor-initialised and read by the runtime, but no XML attribute maps to them:

  `+0x0c0` = 0  (at runtime: pointer to the biome's `BitmapCaves` record)
  `+0x1f4` = 0
  `+0x2a4` = 0  (at runtime: the materials pointer)

`+0x16c` (`fog_of_war_type`), `+0x220` (`noise_type`) and `+0x224` (`mInsideNoiseType`) are enums
registered through the enum path and do have XML names. `+0x254` and `+0x25c` are the upper halves
of the 8-byte fields at `+0x250` and `+0x258`.

`+0x04` is the biome **type** (0 procedural, 1 bitmap, 2 wang tile) and `+0x08` is the **`name`**
string; both are named in the source but registered differently from the XML fields. `+0x1B0`,
`+0x1D4` and `+0x2A4` are the runtime pointers to the loaded bitmap, the built wang/cave map and
the materials array. **`+0x2A4` *is* zero-initialised** by the constructor (`param_1[0xa9] = 0`),
and it is simply unregistered.

## `BiomeModifiers`

0x18 bytes at `Biome + 0x170`, ten fields, all of them gameplay rather than shape. These are
what `Message_VisitedNewBiome` and `Message_EnteredBiome` are about; a biome with non-default
modifiers shows a message when you enter it.

| offset | field | default | range | meaning |
|---|---|---|---|---|
| +0x04 | `dust_amount` | `0.0` | float | "amount of dust rendered" |
| +0x08 | `projectile_drag_coeff` | `1.0` | float | "projectile velocity is multiplied with this value every frame" |
| +0x0C | `entity_gravity_y_multiplier` | `1.0` | float | "affects the VelocityComponents gravity_y" |
| +0x10 | `fog_of_war_delta` | `0` | -10..10 | "fog of war change per frame" |
| +0x11 | `fire_extinguish_chance` | `0` | 0..100 | "probability of fire being extinguished per cell update" |
| +0x12 | `reaction_freeze_chance` | `0` | 0..100 | "probability of spontaneous freezing reactions per cell reaction update" |
| +0x13 | `reaction_unfreeze_chance` | `0` | 0..100 | "probability of spontaneous frozen material melting reactions per cell reaction update" |
| +0x14 | `random_water_stains_chance` | `0` | 0..100 | "probability of random water stains being added to characters" |
| +0x15 | `random_water_stains_amount` | `0` | 0..100 | "number of water cells added when adding random water stains" |
| +0x16 | `everything_is_conductive` | `0` | bool | "if 1, every static material in this place conduct electricity" |

The shipped per-biome modifier files are `data/biome_impl/biome_modifiers/*.xml`; the whole set
is 41 KB of `data/scripts/biome_modifiers.lua`. The manager created during load gets a
`biome_modifiers_inject_spawns` callback, which is how a mod injects spawns into a biome that
has modifiers.

## `RandomColor`

Inside `<RandomMaterials>`, and the only place colour is reinterpreted:

```xml
<RandomColor input_color="FF00BBEE" output_colors="FF000000,FFFFFFFF"/>
```

"for wang tiles allows color -> random color transform" - but the implementation contains no
random number generator. It is a **palette remap**: a template pixel of `input_color` becomes
a member of `output_colors`. Two rules ship, in `coalmine.xml`, for `FF00BBEE` and `FF12BBEE` -
and **neither colour occurs in any shipped wang template**, so both rules are dead in vanilla.
See [worldgen-wang.md](worldgen-wang.md).

## The three topology types

`Topology::type` is a three-valued enum, and it selects the per-pixel algorithm:

| value | name | per-pixel algorithm | shipped by |
|---|---|---|---|
| 0 | `BIOME_PROCEDURAL` | perlin/gradient cave noise, then `<MaterialComponent>` selection | `hills`, `desert`, `empty`, `null`, `water`, `gold` and the other biomes that omit `type` (the default) |
| 1 | `BIOME_BITMAP` | read `bitmap_filename` as an RGBA image with wrap-around tiling, RGB *is* the material value | none - no shipped biome sets it |
| 2 | `BIOME_WANG_TILE` | look up the wang tile at the pixel, then the material's x threshold | `coalmine`, `crypt`, `vault`, `rainforest`, `wizardcave` |

The type is read from the `type` attribute and stored at `+0x04`. The slot's in-source
description lists only two values ("0 = standard, 1 = bitmap level loading"), but the runtime
tests it against 0, 1 and 2 for the three topology types. [worldgen-terrain.md](worldgen-terrain.md) has all three algorithms.

## A note on XML field names

The C++ field names and the XML attribute names are the same strings - the config system
registers them for serialisation, which is why `data/biome/*.xml` can be written by hand. In the
shipped `<MaterialComponent>` entries (746 of them) `rare_use_perlin`, `rare_scale_x/y`,
`rare_polka_*` and `rare_required_*` appear on about 680, and `add_perlin`/`add_perlin_scale_x/y`
on 88; none sets `rare_offset_x/y`, `rare_offset_by_seed` or `rare_use_fbm_perlin`.