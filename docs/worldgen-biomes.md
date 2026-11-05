# Biomes: the data model

A biome is one XML file, `data/biome/NAME.xml`, with exactly two children: `<Topology>` (how the
shape is generated) and `<Materials>` (what it is made of). This page covers the file structure,
every field with its compiled-in default, and `BiomeModifiers`.

Shape algorithms: [worldgen-terrain.md](worldgen-terrain.md). Tile system:
[worldgen-wang.md](worldgen-wang.md). Runtime layout: [worldgen-reference.md](worldgen-reference.md).

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

| element | controls | documented in |
|---|---|---|
| `<Topology>` | generator type, backgrounds, audio, wang template, Lua hook, noise | this page |
| `<BitmapCaves>` (inside `<Topology>`) | procedural cave carving | [terrain](worldgen-terrain.md#the-cave-generator-bitmapcaves) |
| `<RandomMaterials>` (inside `<Topology>`) | wang colour remap | [wang](worldgen-wang.md#randomcolor) |
| `<MaterialComponent>` | which material a pixel becomes | [terrain](worldgen-terrain.md#type-0-procedural-and-the-material-selector) |
| `<VegetationComponent>` | trees, grass, ceiling plants | [pixel scenes](worldgen-pixelscenes.md#vegetation) |
| `<FossilComponent>` | image-stamped blobs of one material | [terrain](worldgen-terrain.md#fossilcomponent) |

The game ships 146 biome files in `data/biome/` (plus `_biomes_all.xml` and three `_pixel_scenes*.xml` tables), and pixel-scene-only rooms in
`data/biome/orbrooms/` and `data/biome/tower/`. Per-biome images, wang extras and Lua live in
`data/biome_impl/<biome>/`.

## The three topology types

`type` selects the per-pixel algorithm:

| value | name | per-pixel algorithm | shipped by |
|---|---|---|---|
| 0 | `BIOME_PROCEDURAL` (default) | cave noise, then `<MaterialComponent>` selection | 60 of the 146 (`hills`, `desert`, `empty`, `water`, `gold`, ...); most omit `type` |
| 1 | `BIOME_BITMAP` | `bitmap_filename` tiled over the world; each pixel's RGB *is* the material colour | none |
| 2 | `BIOME_WANG_TILE` | look up the wang tile at the pixel | 86 of the 146 (`coalmine`, `crypt`, `vault`, `rainforest`, `wizardcave`, ...) |

The field's in-source description mentions only two values ("0 = standard, 1 = bitmap level
loading"), but the generator handles all three. Algorithms in
[worldgen-terrain.md](worldgen-terrain.md).

## `<Topology>` fields

The XML attribute names are the C++ field names. Defaults are the values the constructor writes;
offsets are into the `Biome` record.

**Identity and generator**

| offset | field | default | notes |
|---|---|---|---|
| `+0x04` | `type` | `BIOME_PROCEDURAL` | see above |
| `+0x08` | `name` | "" | display name, usually a `$biome_*` translation key |
| `+0x198` | `bitmap_filename` | "" | `BIOME_BITMAP` source image |
| `+0x0a8` | `bitmap_noise_file` | "" | procedural biome: use this image as the cave map |
| `+0x1b4` | `wang_template_file` | "" | `BIOME_WANG_TILE` template |
| `+0x1cc` | `wang_map_width` | `256` | range 1-512 |
| `+0x1d0` | `wang_map_height` | `256` | range 1-512 |
| `+0x1f8` | `static_tile` | `false` | blit the wang template directly, no randomisation ([wang](worldgen-wang.md#static_tile)) |
| `+0x1fc` | `static_tile_bg_mask` | "" | mask image path |
| `+0x214` | `static_tile_bg_mask_threshold` | `0.5` | |
| `+0x1d8` | `lua_script` | "" | biome Lua: `init` and spawn functions |
| `+0x1f0` | `pixel_scene` | `false` | load this biome's pixel scenes eagerly |

**Backgrounds and edges**

| offset | field | default |
|---|---|---|
| `+0x020` | `background_image` | "" |
| `+0x038` | `background_edge_left` | "" |
| `+0x050` | `background_edge_right` | "" |
| `+0x068` | `background_edge_top` | "" |
| `+0x080` | `background_edge_bottom` | "" |
| `+0x098` | `background_edge_priority` | `0` |
| `+0x09c` | `background_use_neighbor` | `false` |
| `+0x0a0` | `background_image_height` | `225.0` |
| `+0x0a4` | `limit_background_image` | `true` |
| `+0x0c4` | `noise_biome_edges` | `true` |
| `+0x0c5` | `big_noise_biome_edges` | `true` |
| `+0x0c6` | `fat_biome_edges` | `false` |
| `+0x0c7` | `skip_edge_textures` | `false` |

**Audio**

| offset | field | default |
|---|---|---|
| `+0x0c8` | `audio_music_enter` | "" |
| `+0x0e0` | `audio_music_2` | "" |
| `+0x0f8` | `audio_music_energy_coeff` | `1.0` |
| `+0x0fc` | `audio_music_no_forced_quietness` | `false` |
| `+0x100` | `audio_music_forced_quietness_duration_seconds` | `0` (int) |
| `+0x104` | `audio_music_trigger_without_danger` | `false` |
| `+0x108` | `audio_music_trigger_min_y` | `0` (int) |
| `+0x10c` | `audio_music_trigger_max_y` | `0` (int) |
| `+0x110` | `audio_biome_id_for_music` | "" |
| `+0x128` | `audio_ambience` | "cave" |
| `+0x140` | `audio_ambience_surface` | "hills" |

**Look, weather and gameplay**

| offset | field | default |
|---|---|---|
| `+0x158` | `color_grading_r` | `1.0` |
| `+0x15c` | `color_grading_g` | `1.0` |
| `+0x160` | `color_grading_b` | `1.0` |
| `+0x164` | `color_grading_grayscale` | `0.0` |
| `+0x168` | `has_rain` | `true` |
| `+0x16c` | `fog_of_war_type` | `DEFAULT` |
| `+0x188` | `gradient_sky_alpha_target_mod` | `0.0` |
| `+0x18c` | `rain_target_mod` | `0.0` |
| `+0x190` | `fog_target_mod` | `0.0` |
| `+0x194` | `sunset_alpha_target` | `1.0` |
| `+0x218` | `game_enemy_hp_scale` | `1.0` |
| `+0x21c` | `game_enemy_attack_speed` | `1.0` |
| `+0x2a0` | `coarse_map_force_terrain` | `false` |
| `+0x2a1` | `coarse_map_not_terrain` | `false` |
| `+0x2a2` | `coarse_map_cell_count_always_zero` | `false` |
| `+0x2a3` | `coarse_map_inject_light` | `false` |
| `+0x2a8` | `mModifierUIDescription` | "" |
| `+0x2c0` | `mModifierUIDecorationFile` | "" |
| `+0x2d8` | `mDebugFilename` | "" |

`fog_of_war_type` values: `DEFAULT`, `HEAVY_CLEAR_AT_PLAYER`, `HEAVY_CLEAR_WITH_MAGIC`,
`HEAVY_NO_CLEAR`.

**Procedural noise** (used by `BIOME_PROCEDURAL`; see
[terrain](worldgen-terrain.md#procedural-noise-shaping))

| offset | field | default |
|---|---|---|
| `+0x220` | `noise_type` | `IQ2_SIMPLEX1234` |
| `+0x224` | `mInsideNoiseType` | `SimplexNoise` |
| `+0x228` | `mInsideNoiseFBM` | `true` |
| `+0x230` | `mInsidePerlinScaleX` | `1.0` (double) |
| `+0x238` | `mInsidePerlinScaleY` | `1.0` (double) |
| `+0x240` | `mInsidePerlinSquared` | `false` |
| `+0x241` | `mInsidePerlinClamped` | `false` |
| `+0x242` | `mInsidePerlinScaled` | `false` |
| `+0x244` | `mInsidePerlinScaleMin` | `0.0` |
| `+0x248` | `mInsidePerlinScaleMax` | `0.0` |
| `+0x24c` | `mInsidePerlinForceInside` | `false` |
| `+0x24d` | `mInsidePerlinOffsetBySeed` | `false` |
| `+0x250` | `mInsidePerlinOffsetX` | `0.0` (double) |
| `+0x258` | `mInsidePerlinOffsetY` | `0.0` (double) |
| `+0x260` | `mInsideAddValue` | `0.0` |
| `+0x264` | `mMultiplierGradient` | `1.0` |
| `+0x268` | `mMultiplierPerlin` | `1.0` |
| `+0x26c` | `mMultiplierExtraPerlin` | `1.0` |
| `+0x270` | `mGradientStartY` | `-512.0` |
| `+0x274` | `mGradientEndY` | `512.0` |
| `+0x278` | `mGradientSlopeStartX` | `0.0` (double) |
| `+0x280` | `mGradientSlopeDelta` | `1.0` (double) |
| `+0x288` | `mGradientAddNoise` | `1` (int) |
| `+0x290` | `mGradientNoiseScale` | `0.01` (double) |
| `+0x298` | `mGradientLowNoise` | `-100.0` |
| `+0x29c` | `mGradientHighNoise` | `100.0` |

`noise_type` values: `IQ2_SIMPLEX1234`, `IQ_SIMPLEX`, `SIN_CAPPED_EVERYTHING`,
`SIN_CAPPED_SIMPLEX`. `mInsideNoiseType` values: `IQNoise`, `DirtyPeeNoise`, `QemNoise`,
`WhiteNoise`, `MixNoise`, `SimplexNoise`, `STB_Perlin`, `FastBlockNoise`, `SimplexNoise1234`.
Integer values are in [worldgen-reference.md](worldgen-reference.md#enums).

Booleans are written `0`/`1`. Fields marked int must be written as integers.

## `BiomeModifiers`

`<modifiers>` inside `<Topology>`: ten gameplay fields at `Biome + 0x170`. The descriptions are
the game's own.

| field | default | range | meaning |
|---|---|---|---|
| `dust_amount` | `0.0` | float | amount of dust rendered |
| `projectile_drag_coeff` | `1.0` | float | projectile velocity is multiplied by this every frame |
| `entity_gravity_y_multiplier` | `1.0` | float | multiplies `VelocityComponent` `gravity_y` |
| `fog_of_war_delta` | `0` | -10..10 | fog of war change per frame |
| `fire_extinguish_chance` | `0` | 0..100 | chance of fire being extinguished per cell update |
| `reaction_freeze_chance` | `0` | 0..100 | chance of spontaneous freezing per cell reaction update |
| `reaction_unfreeze_chance` | `0` | 0..100 | chance of frozen material melting per cell reaction update |
| `random_water_stains_chance` | `0` | 0..100 | chance of water stains being added to characters |
| `random_water_stains_amount` | `0` | 0..100 | water cells added per stain |
| `everything_is_conductive` | `0` | bool | every static material conducts electricity |

The shipped presets are in `data/biome_impl/biome_modifiers/*.xml`, and
`data/scripts/biome_modifiers.lua` decides which biomes get which modifier. Its
`biome_modifiers_inject_spawns` callback is how a mod adds spawns to a modified biome.

### The biome-entry message

Entering a biome for the first time submits two messages to the same handler `GamePrintImportant`
uses:

1. The `$log_entered` title - only if the biome's `name` is non-empty and not `_EMPTY_`, and the
   handler has not seen that name before.
2. If `mModifierUIDescription` is non-empty: a second message with that text as the **title** and
   `mModifierUIDecorationFile` as its decoration.

`biome_modifiers.lua` sets those two fields; it does not call `GamePrintImportant` itself.

Consequences for mods:

- The empty-name / `_EMPTY_` check gates the whole entry handler, not just the popup. A biome that
  fails it also skips `audio_music_enter`, the `event_cues/new_biome/create` sound and the
  `Message_VisitedNewBiome` event. `GameStatsComponent` uses that event for
  `biomes_visited_with_wands` (no-wand-run detection) - which is why Hills does not count.
- Leaving `mModifierUIDescription` empty removes the modifier message; the title still appears.
- There is one decoration image per message, no per-modifier icon slot, and no way to merge the
  modifier text into the title.
- Lua cannot inspect, cancel or reorder pending important messages, so a custom `Gui` overlay
  cannot be kept in sync with the native toast.
- Overriding `log_entered` with empty values in `data/translations/common.csv` (via
  `ModTextFileSetContent`, see [ui-modding.md](ui-modding.md#rewrite-the-games-text-fonts-and-tunables-as-data))
  blanks the title text, but an empty message is still shown. Override the row rather than
  delete it: a missing key can display as the literal `$log_entered`.
- No special string cancels a message. Using `_EMPTY_` as a message title has no effect; the check
  applies to the biome name only.

## Writing biome files

### What the loader reads

The reader (`0x0086b9f0`) ignores the root element's name and attributes and reads only:

- `<Topology>` - the fields above, plus its children `<BitmapCaves>`, `<RandomMaterials>` and
  `<modifiers>`;
- `<Materials>` - one attribute, `name`, and children `MaterialComponent`, `VegetationComponent`,
  `FossilComponent`.

A `<RandomMaterials>` placed directly under the root (as a couple of mods do) is ignored.

### What the loader ignores

Many shipped and modded biome files carry `<Topology>` attributes from an older format. **None of
these names exists in the executable**, so they do nothing:

`mBiomeMaterialsFile`, `mWaterLevel`, `mGradientType`, `mGradientMinY`, `mGradientMaxY`,
`mGradientSinMultipleMin`, `mGradientSinMultipleMax`, `mPerlinScale_x`, `mPerlinScale_y`,
`mExtraPerlinScaleX`, `mExtraPerlinScaleY`, `mCaveType`, `mCaveOpenness`, `mCaveZoom`,
`mCaveStartValue`, `mCavePerlinNoise`, `mCavePerlinNoise2`, `mCavePerlinDisplacement`,
`mInsidePerlinFunc`, `color_correction_r`, `color_correction_g`, `color_correction_b`,
`color_correction_lerp`.

`sky_sunset_alpha_target` exists in the executable but is not a biome field; use
`sunset_alpha_target`.

### Which `<MaterialComponent>` fields are used in practice

Of the 746 shipped `<MaterialComponent>` entries, about 680 set `rare_use_perlin`,
`rare_scale_x/y`, `rare_polka_*` and `rare_required_*`; 88 use `add_perlin`. None sets
`rare_offset_x/y`, `rare_offset_by_seed` or `rare_use_fbm_perlin`.

### Malformed XML in the shipped files

83 of the 170 shipped biome files are not well-formed XML: their section comments are written
`<!------------ MATERIALS -------------------->`, and `--` is illegal inside an XML comment. The
game's parser accepts them; standard XML tools reject the whole file.
