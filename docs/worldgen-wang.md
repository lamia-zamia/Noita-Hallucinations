# Wang tiles

Most of Noita's underground is built from wang tiles. Each `BIOME_WANG_TILE` biome names one
template PNG; at load the game expands it with a herringbone wang-tile generator (the
`stb_herringbone_wang` design) into a `wang_map_width` x `wang_map_height` map. Each map pixel then
carries a density value, a material, and optionally a Lua spawn function.

What a map pixel becomes in the world (a material or a spawn call) is in
[worldgen-terrain.md](worldgen-terrain.md).

## The template images

`data/wang_tiles/*.png`, named by the biome's `wang_template_file`. The 31 shipped templates:

| size | templates |
|---|---|
| 1206x278 | `winter_caves`, `winter_caves_2` |
| 440x560 | `meat`, `snowcave` |
| 348x448 | `coalmine` |
| 344x440 | `clouds`, `excavationsite`, `liquidcave`, `robobase`, `vault`, `vault_frozen` |
| 282x342 | `crypt`, `wizardcave` |
| 264x340 | `wand` |
| 232x332 | `coalmine_petri_experiment`, `snowcastle` |
| 232x300 | `coalmine_alt` |
| 172x360 | `rainforest`, `rainforest_dark`, `rainforest_open` |
| 156x364 | `the_end` |
| 150x248 | `pyramid` |
| 144x235 | `fungicave`, `fungiforest` |
| 144x214 | `tutorial` |
| 140x220 | `sandcave`, `snowchasm` |
| 110x230 | `water` |
| 102x316 | `the_sky` |
| 86x113 | `endgame` |
| 60x100 | `town_under` |

The template is a herringbone tile set, so it can be much smaller than the generated map. The
generator (`0x00866b90`) refuses a map more than 106 tiles wide or high
(`increase STB_HBWANG_MAX_X/Y`).

Most of each template is greyscale - a density map - with coloured pixels for materials and
spawns:

| template | distinct greys | coloured pixels | greyscale share |
|---|---|---|---|
| `coalmine.png` | 218 | 12 897 | 91.7 % |
| `crypt.png` | 4 | 14 675 | 84.8 % |
| `winter_caves_2.png` | 165 | 4 671 | 98.6 % |
| `wizardcave.png` | 4 | 15 534 | 83.9 % |

## What a template pixel means

After expansion, the map builder (`0x008704c0`) classifies each pixel:

| pixel | effect |
|---|---|
| RGB `000000` or alpha 0 | nothing; the pixel is left empty |
| alpha `0xFE` | sets the mask plane (dilated by one pixel afterwards) |
| grey or white | density only |
| `0xFFC0FFEE` | reserved (see below) |
| any other colour | a material, or else a Lua spawn function |

Pixels that come through the herringbone expansion are forced to alpha `0xFF`, and no shipped
template contains alpha `0xFE`, so the mask path is only reachable by images stamped in directly.

### Grey: density

A grey pixel writes only the density plane:

```
density = (G + 2*B) / 765
```

so a grey `(v,v,v)` is `v/255` and white is `1.0`, the maximum. A grey pixel does not start, end
or separate anything; it is a value. That is why `winter_caves_2.png` is 98.6 % grey with 165
levels - it is an authored density map.

In the pixel pass, density above `0.5` lets the tile decide the pixel; density below `0` marks the
pixel in a separate flag plane.

### Colour: material

A coloured pixel is first looked up in the global `wang_color` table. `materials.xml` gives
`wang_color` to 469 `CellData` / `CellDataChild` entries (467 distinct colours). A hit stores
**material id + 1** in the tile plane, with density `+1.0`. Material 0 (air) is stored as tile `1`
with density `-1.0` - explicitly empty.

The game checks this table at load: `CellFactory - wang_color collision in ...` and
`ERROR in materials.xml!!! wang_color for ...`.

### Colour: spawn function

A colour that is not a material is passed to the biome's spawn dispatcher. If it is registered,
its index (1-based) goes into the spawn plane and an `{x, y, index}` record is added to the map.
Colours are registered by `data/scripts/wang_scripts.csv` (below) and by the biome script:

```lua
RegisterSpawnFunction( 0xff0000ff, "spawn_nest" )
RegisterSpawnFunction( 0xffB40000, "spawn_fungi" )
RegisterSpawnFunction( 0xff969678, "load_structures" )
```

The integer is `0xAARRGGBB`: `0xff0000ff` is pure blue, which occurs 102 times in `coalmine.png`.

### `0xFFC0FFEE`: reserved

The game refuses this colour in `materials.xml`:

> `materials.xml error - please don't use the color 0xFFC0FFEE, because it has a special
> meaning in the wang map.`

Four shipped templates use it (2 801 pixels): `coalmine_petri_experiment` (1 956), `coalmine`
(411), `coalmine_alt` (310), `excavationsite` (124). Its exact effect is not documented here; do
not use it for anything else.

### Filling holes

Where the map has no tile, the four neighbours vote (`0x0092a260`): the most common neighbour
value wins, and only if all four are empty does the pixel stay a hole. Templates therefore need
broad, coherent colour regions to come out clean.

## Map geometry

| quantity | value |
|---|---|
| per-biome wang map | `wang_map_width` x `wang_map_height`, default 256 x 256, range 1-512 |
| one map cell | **10 x 10 world pixels** |
| whole-world wang map | (70 x 512)/10 x (48 x 512)/10 = 3584 x 2457; per-biome maps are composited into it |
| tile lookup | round to nearest, wrap, minus one |

The tile lookup (`0x008712d0`) looks like smoothstep interpolation but is a rounding:
`f²(3 - 2f) >= 0.5` exactly when `f >= 0.5`. So

```
tile = tilePlane[(round(y) mod H) * W + (round(x) mod W)] - 1
```

with no interpolation. `0x00871380` does the same for the spawn plane.

## The four planes

The built map (`mBitmapNoise`, `Biome + 0x1D4`) holds four planes and a record list:

| plane | element | holds |
|---|---|---|
| A | `float` | density `(G+2B)/765` |
| B | `uint16` | material id + 1 |
| C | `uint8` | spawn-function index, 1-based |
| D | `uint8` | mask |
| records | 12 bytes | `{x, y, index}` spawn points |

All plane reads wrap negative coordinates and take `% W` / `% H`. Plane layout and accessor
addresses are in [worldgen-reference.md](worldgen-reference.md#wang-map-planes).

## `wang_scripts.csv`

`data/scripts/wang_scripts.csv` registers 30 colours for every wang biome:

```csv
#COLOR,#FUNCTION_NAME,#IN_PATH_MIN,#IN_PATH_MAX,#OUTSIDE_PATH_MIN,#OUTSIDE_PATH_MAX
ffff0000,spawn_small_enemies,-1,-1,-1,-1
ff00ff00,spawn_items,-1,-1,-1,-1
ffff0aff,load_pixel_scene,-1,-1,-1,-1
ff50A0F0,spawn_wands,-3,-3,-4,-4
```

- Colours are hex `AARRGGBB`, the same reading as `RegisterSpawnFunction`.
- Rows starting with `#` are skipped; empty or non-numeric path columns read as `-1`.
- The table is loaded when a world starts or loads.
- When a biome's script manager is created, **every CSV row is registered first**, then the
  biome's own script runs.
- Registering a colour twice is refused with
  `Lua error - AddScriptColor( <colour>, <function> ) - already registered...`, and the first
  registration stays. **A biome script cannot override a CSV colour.**

The path columns constrain the spawn distance from the path ("in path" vs "outside path"). Only
`spawn_wands` sets them; `-1` means no constraint.

14 of the 30 colours occur in shipped templates. The most common: `ffff0000` `spawn_small_enemies`
(5 869 px), `ffffff00` `spawn_lamp` (2 607), `ffc88d1a` `spawn_props` (2 052), `ff800000`
`spawn_big_enemies` (1 185), `ff00ff00` `spawn_items` (1 136). None of the 30 is also a material
`wang_color`.

`data/scripts/wang_scripts_removed.txt` lists three disabled entries: `spawn_persistent_teleport`,
`spawn_chest`, `spawn_blood`.

## `RandomColor`

`<RandomMaterials>` inside `<Topology>` remaps template colours:

```xml
<RandomColor input_color="FF00BBEE" output_colors="FF000000,FFFFFFFF"/>
```

The in-source description is "for wang tiles allows color -> random color transform", but the
reader (`0x00865920`) contains no random number generator; it is a palette remap of `input_color`
to a member of `output_colors`. The only two shipped rules (`FF00BBEE`, `FF12BBEE`, both in
`coalmine.xml`) match no pixel in any shipped template, so they have no effect in vanilla.

## `static_tile`

`static_tile` (default `false`):

> if 1, directly blits `wang_template_file` without wang randomization, kind of like a pixel
> scene but at the wang tile resolution

The template is used as a literal tile image, one tile per pixel. `static_tile_bg_mask` (an image
path, "same size and resolution as wang_template_file") and `static_tile_bg_mask_threshold`
(default `0.5`) drive the `sprite_static_tile_bg.frag` shader, which decides which background
pixels show through. `data/biome_impl/static_tile/` holds the shipped static-tile biomes.

## Tile choice ignores tile content

The herringbone generator chooses tiles by their edge-colour constraints alone (`0x00866970`):
it counts the tiles that fit and draws one with the wang RNG. It never reads a tile's pixels. So
editing pixels *inside* tiles, while leaving the header pixels (the last three pixels of row 0)
and the tile grid alone, gives the same tile arrangement for the same seed.

That RNG is a Park-Miller stream (`0x00866120`). The game creates it lazily from the world seed
and does **not** reseed it per biome: the biome loader passes seed `0.0`, which means "continue"
(`0x00867de0` -> `0x00867500`). Each biome's layout therefore depends on how many draws every
earlier wang biome made. Those biomes are loaded in `_biomes_all.xml` order.

## `coalmine.png` is hard-coded by path

The start-path pass (`0x008771a0`) runs for each biome region and makes sure a route exists
through it. It compares the biome's `wang_template_file` with the literal string
`data/wang_tiles/coalmine.png`, and it takes different branches on the result:

| | template path is exactly `data/wang_tiles/coalmine.png` | any other path |
|---|---|---|
| start area | fixed: map column `0x8e`, width 12 | from `BIOME_PATH_FIND_WORLD_POS_MIN_X`/`MAX_X` (159/223, divided by 10); a 7-row strip there is cleared to black |
| `extra_layers/coalmine.png` | loaded (`0x0087a900`) and copied over the region map: `(0,0,66)` pixels are cleared to black, and other opaque non-black pixels are copied in | not loaded |
| after the start path is found | `(0,66,0)` extra-layer pixels (6 023 px, a band across the top 51 rows) are cleared to black | nothing |
| ramp pixels | `FEFEFE` in rows 0-14, columns `0x8c`-`0x9d` are cleared to black | kept |

Black (`000000`) means "no wang tile": the pixel gets procedural terrain or the neighbour vote
instead (see [Filling holes](#filling-holes)).

So a biome that uses a copy of `coalmine.png` under another name, even a pixel-identical copy,
gets a different Mines. The hand-authored top band is gone, the entrance moves, and the route
search starts from somewhere else. The vanilla readme in `data/wang_tiles/extra_layers/` says the
same thing ("only loads this for coalmine").

A mod that wants a re-skinned Mines therefore has to keep
`wang_template_file="data/wang_tiles/coalmine.png"`. It must change the image's content rather
than its path.

## Extra layers and debug output

`data/wang_tiles/extra_layers/` holds `coalmine.png` (and a `readme.txt`), loaded on top of the
template of the same name.

Debug images written at load when enabled:

```
temptemp/biome_wangs/                per-biome wang maps
temptemp/_biomes_all_wang.png        all biomes, one sheet
temptemp/_biomes_all_wang2.png       the same, second pass
```

`CreateHerringboneWang` (`0x00879bf0`) is a separate development tool, not part of world loading:
it is called from one place among other `temptemp/` writers and hardcodes `coalmine.png` and
`excavationsite.png` as inputs. The load path is `0x0087a900` -> `0x0086b9f0` -> `0x00867de0` ->
`0x00870de0` -> `0x008704c0`.
