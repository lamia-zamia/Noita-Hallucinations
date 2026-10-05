# Wang tiles

The system that generates most of Noita's underground. A wang tile set is one PNG per biome,
and the PNG is fed through a herringbone wang-tile image generator (the `stb_herringbone_wang`
design) that expands it into the biome's `wang_map_width` x `wang_map_height` image. The **pixel
values** of that image are a density field plus a colour-indexed material lookup.

This page was rewritten after an audit found the first version of it wrong in its central claim.
What follows is the version that survives cross-checking against the shipped PNGs.

For what a wang tile *becomes* (a material, a spawn request) see
[worldgen-terrain.md](worldgen-terrain.md).

## The template images

`data/wang_tiles/*.png`, one per biome, named by the `wang_template_file` attribute of a
`BIOME_WANG_TILE` biome's `<Topology>`:

```
clouds 344x440        crypt 282x342          fungicave 144x235     liquidcave 344x440
coalmine 348x448      endgame 86x113         fungiforest 144x235   meat 440x560
coalmine_alt 232x300  excavationsite 344x440  pyramid 150x248      rainforest 172x360
coalmine_petri_experiment 232x332              robobase 344x440     rainforest_dark 172x360
snowcastle 232x332    rainforest_open 172x360  sandcave 140x220    snowcave 440x560
snowchasm 140x220     the_end 156x364          the_sky 102x316     town_under 60x100
tutorial 144x214      vault 344x440            vault_frozen 344x440  wand 264x340
water 110x230         winter_caves 1206x278    winter_caves_2 1206x278  wizardcave 282x342
```

They are large, and their pixel statistics are the key to understanding them:

| template | opaque greys | chromatic px | greyscale share |
|---|---|---|---|
| `coalmine.png` | 218 distinct | 12 897 | 91.7 % |
| `crypt.png` | 4 distinct | 14 675 | 84.8 % |
| `winter_caves_2.png` | **165 distinct** | 4 671 | 98.6 % |
| `wizardcave.png` | 4 distinct | 15 534 | 83.9 % |

So the dominant content of every template is a **greyscale ramp**, not coloured tiles.
`winter_caves_2.png` ships a 165-step ramp from 0 to 255 and is essentially monochrome.

## What one template pixel means

The template is first expanded by `0x00867500` (which calls the herringbone generator at
`0x00866b90`) into a 32-bit-per-pixel image; pixels copied from the template get alpha forced
to `0xFF`. The builder is `0x008704c0`, run once per biome region at load time over that image. For
each pixel, read as a 32-bit word:

### 1. Nothing at all

```c
if (((uVar2 != 0) && ((uVar2 & 0xffffff) != 0)) && ((uVar2 & 0xff000000) != 0))
```

A pixel is skipped entirely if it is transparent black, if its RGB is zero, or if its alpha is
zero. Nothing is written to any plane. In the shipped templates **every pixel is fully opaque**,
so this branch only fires for zero-RGB pixels.

### 2. Alpha `0xFE` — the mask

A separate earlier branch sets a byte in plane D to 1, and a later pass dilates that mask by one
pixel in each direction. **No shipped template contains an alpha-254 pixel** (checked across all
32 files), and pixels that come through the herringbone expansion are forced to alpha `0xFF`, so
this branch can only be reached by an image that is blitted directly.

### 3. Greyscale and white — the **density/weight plane**

A greyscale pixel writes **only** the float plane:

```c
// 0x008704c0
fVar13 = (float)(double)(uVar2 >> 16 & 0xff) * (1.0f/255.0f);        // bits 16-23: the PNG's blue channel
weight = ((float)(double)(uVar2 >>  8 & 0xff) * (1.0f/255.0f)       // G
          + fVar13 + fVar13) / 3.0f;
```

so

```
weight = (G + 2*B) / 765
```

which means:

- **white (`FFFFFF`, weight 1.0) is the maximum of the density field**;
- **any grey `(v,v,v)` gives exactly `v/255`** — a linear density ramp;
- it is *not* a run terminator and it does not terminate or start anything. It is a value.

That is why the templates ship greyscale ramps, and why `winter_caves_2.png` is 98.6 % grey with
165 distinct levels: it is an authored density map.

### 4. `0xFFC0FFEE` — the reserved colour

There is exactly one colour with a documented special meaning, and the game refuses to let you
use it in `materials.xml`:

> `materials.xml error - please don't use the color 0xFFC0FFEE, because it has a special
> meaning in the wang map. Thank you. Was used in: `

It is present in the shipped data:

| template | `0xFFC0FFEE` pixels |
|---|---|
| `coalmine_petri_experiment.png` | 1 956 |
| `coalmine.png` | 411 |
| `coalmine_alt.png` | 310 |
| `excavationsite.png` | 124 |

Total 2 801. **What it does is not resolved** — the diagnostic lives in the `materials.xml`
validator, not in the wang builder, and no test for this constant appears in the builder. Its
weight would be `(0xFF + 2*0xC0)/765 = 0.835`, which is *not* a sentinel value, so it is not
simply reaching the greyscale branch. Treat it as a known-reserved marker of unknown effect.

### 5. Every other colour — a material or a Lua spawn function

A chromatic, non-reserved colour is looked up in two places, in order:

**(a) The global `wang_color` table.** `materials.xml` carries a `wang_color` attribute on
**469 `CellData` / `CellDataChild` entries (467 distinct colours)**, and the engine has two dedicated diagnostics for it
(`CellFactory - wang_color collision in `, `ERROR in materials.xml!!! wang_color for `). A hit
gives **tile id = material id + 1** and weight `+1.0`; material 0 (air) is stored as tile id 1
with weight `−1.0`, i.e. "explicitly nothing".

**(b) The biome's own Lua script**, via `RegisterSpawnFunction`:

```lua
RegisterSpawnFunction( 0xff0000ff, "spawn_nest" )
RegisterSpawnFunction( 0xffB40000, "spawn_fungi" )
RegisterSpawnFunction( 0xff969678, "load_structures" )
```

A miss in (a) calls the biome's Lua dispatcher, which returns an index; that index is stored
1-based in a **byte plane** and pushed onto a 12-byte `{x, y, index}` record list.

This is the live path, and it is verifiable: the integer given to `RegisterSpawnFunction` is
`0xAARRGGBB`, so `0xff0000ff` is pure blue, and `coalmine.png` contains that colour (0, 0, 255)
102 times. Across all biome scripts, 103 of the 225 registered colours occur in the shipped
templates under this reading, and only 3 under the reversed one.

## The uint16 plane is a material id, not a run length

The tile plane is
**`material_id + 1`**, and there is no counter in the function that could be a run length:

- the value written at the "tile" site comes from a **pure map lookup** (`local_64 = fn(colour)`
  or `local_34 = map[colour]`), and it is recomputed **only when the colour changes** — so every
  pixel of one colour gets the *same* value, not an incrementing one;
- three consumers index a `CellFactory` array of stride `0x290` with `tile_id - 1`, which is the
  same formula as `GetMaterialAt`'s `CellFactory + 0x28` lookup;
- tile id `1` with weight `−1.0` is not "a run of length 1"; it is **material 0**, i.e. air.

So: **no run-length encoding, no herringbone resolve on this path.**

## Hole filling is a majority vote

Where the map has no value for a pixel, the four-neighbour helper `0x0092a260` is consulted, and
it is not a wang table. It tallies the four neighbour ids into an `unordered_map` and returns
the **mode**:

- **0** means *all four were 0* — a genuine hole, and the pixel is then marked as one;
- otherwise the most common neighbour value wins.

That is why a template needs broad, coherent colour regions to come out clean: the fill is a
plurality vote, so a tile surrounded by four different colours gets whichever of them occurs
twice, or stays a hole if all four differ.

## Two different herringbone paths

The executable contains the STB herringbone wang-tile generator (the string
`increase STB_HBWANG_MAX_X/Y` is in `0x00866b90`, which refuses images more than 106 tiles wide or
high). It **is** on the load path: `0x00867de0` calls `0x00867500`, which loads the template as RGB,
builds the tileset (`0x00867250`) and generates the biome's map image with `0x00866b90`. That is
why the template can be much smaller than `wang_map_width` x `wang_map_height`.

A second, separate function named `CreateHerringboneWang(` (`0x00879bf0`) is **not** on that path:

- `0x00879bf0` is called from exactly one place, `0x009a6300`, which is in the same address band
  as the other bake/dev tools (`0x009a4f50`, `0x009a6220`, `0x009a6510` all write `temptemp/`
  debug output). It also hardcodes `data/wang_tiles/coalmine.png` and
  `data/wang_tiles/excavationsite.png` as its inputs.
- The load path is `0x0087a900` → `0x0086b9f0` → `0x00867de0` → `0x00870de0` → `0x008704c0`, and
  it never calls `0x00879bf0`.

So `CreateHerringboneWang` is the red herring, not the herringbone generator.

## Map geometry

| quantity | value | evidence |
|---|---|---|
| per-biome wang map | `wang_map_width` × `wang_map_height`, default **256 × 256**, range 1..512 | `Biome + 0x1cc` / `+0x1d0` |
| one map cell | **10 × 10 world pixels** | the spawn scan strides 10 px and converts back with `tile * 10 - 5` |
| the world-sized wang map | `(70*512)/10 × (48*512)/10` = **3584 × 2457** | `0x0087a900` sizes the per-biome maps into it |
| a scale scalar | `+0x94 = (int)(width * 0.5 / 0.1)` = `width * 5` | `0x0086b830` |
| tile lookup | round to nearest, wrap, **minus one** | `0x008712d0` |

The `3584 × 2457` figure is the *world* map the per-biome 256×256 maps are composited into; it is
not the size of any one biome's map.

### The tile lookup is a round, not a noise function

`0x008712d0` looks like it does smoothstep interpolation. It does not:

```c
// 0x008712d0.c:26-33
if (0.5f <= (3.0f - fx * 2.0f) * fx * fx) iVar2 = iVar2 + 1;
if (0.5f <= (3.0f - fy * 2.0f) * fy * fy) iVar4 = iVar4 + 1;
return (FUN_0092a1d0(map->tilePlane, iVar4, iVar2) & 0xffff) - 1;
```

`f` is the fractional part of the coordinate, and for `f` in `[0,1]`,
`0.5 <= f²(3 − 2f)` holds exactly when `f >= 0.5`. So this is **`round()` written as
arithmetic**:

```
tileIndex = tilePlane_uint16[ (round(y) mod H) * W + (round(x) mod W) ] - 1
```

No interpolation, no noise. `0x00871380` is the same routine for the byte (colour) plane and
additionally writes the snapped coordinates back through two out-parameters.

## The four planes

The built map at `Biome + 0x1d4` (registered as `mBitmapNoise`) is four 0x20-byte plane
sub-structs at `+0x00`, `+0x20`, `+0x40`, `+0x60`, plus a record vector at `+0x80`. Each
sub-struct is:

| field | meaning |
|---|---|
| `+0x00` / `+0x04` | width / height |
| `+0x08` | element **count** |
| `+0x0C` | element **capacity** |
| `+0x10` | empty-container self-sentinel |
| `+0x14` | 2 bytes |
| `+0x18` | **data pointer** |
| `+0x1C` | capacity / "has data" flag — what the accessors test |

| plane | data at | element | holds |
|---|---|---|---|
| A | `+0x18` | `float` | the density weight `(G+2B)/765` |
| B | `+0x38` | `uint16` | **material id + 1** |
| C | `+0x58` | `uint8` | Lua spawn-function index, 1-based |
| D | `+0x78` | `uint8` | mask / flags |
| — | `+0x80` (end `+0x84`) | 12 bytes | `{x, y, index}` records |

Three typed accessors take a **plane sub-struct**, not the map base:

| accessor | element | used for |
|---|---|---|
| `0x0092a1d0(plane, x, y)` | `uint16` | the material id → which material |
| `0x0092a3a0(plane, x, y)` | `uint8` | the spawn index |
| `0x0092a430(plane, x, y)` | `uint8` | the mask |

All three wrap negatives and then take `% W` / `% H`, and all three return garbage unless the
plane's `+0x1C` flag is non-zero.

**What the weight plane does at generation time** is only partly resolved. Its one identified
gate is in the pixel-chunk worker: `> +0.5` lets the tile decide the pixel, `< 0` sets a separate
flag plane. Which function copies the plane into `PixelChunk + 0xb4` is not identified.

## `wang_scripts.csv`

`data/scripts/wang_scripts.csv` maps 30 colours to Lua function names:

```csv
#COLOR,#FUNCTION_NAME,#IN_PATH_MIN,#IN_PATH_MAX,#OUTSIDE_PATH_MIN,#OUTSIDE_PATH_MAX
ffff0000,spawn_small_enemies,-1,-1,-1,-1
ff00ff00,spawn_items,-1,-1,-1,-1
ffff0aff,load_pixel_scene,-1,-1,-1,-1
ff50A0F0,spawn_wands,-3,-3,-4,-4
```

The colour is read as hex `AARRGGBB` (the same reading as `RegisterSpawnFunction`'s integer),
so `ffff0000` is pure red. Checked against the 32 shipped templates, **14 of the 30 colours
occur**: `ffff0000` / `spawn_small_enemies` (5 869 pixels in total), `ffffff00` / `spawn_lamp`
(2 607), `ffc88d1a` / `spawn_props` (2 052), `ff800000` / `spawn_big_enemies` (1 185),
`ff00ff00` / `spawn_items` (1 136), and nine rarer ones; the other 16 occur in none. None of the
30 collides with a `wang_color` in `materials.xml`, so these pixels are not materials.

**How it is used.** Each row is parsed into a colour, a function name and four path integers
(columns that are empty or non-numeric read as `-1`); rows starting with `#` are skipped. The
table is loaded when a world is started or loaded (`0x0078c910`, from `0x006e9f30` and `0x006afaa0`).
Whenever a biome's script manager is created (`0x0078cfb0`, called from `0x00867de0` and
others), every CSV row is registered into it first, and only then is the biome's own Lua script
run. Both paths register through the same function (`0x0078c4b0`), which **refuses a colour that
is already registered**: it logs
`Lua error - AddScriptColor( <colour>, <function> ) - already registered...` and keeps the first
registration. So a CSV colour always wins, and a biome script that calls `RegisterSpawnFunction`
for one of the 30 CSV colours is rejected with that log line. At generation time the pixel's
packed value is looked up in the manager's colour map (`0x0078c6b0`), which returns the
registration index or `-1`.

The four path columns are a pathfinding request — "the spawn point must be between
`IN_PATH_MIN` and `IN_PATH_MAX` pixels from the tile origin when inside the path, or between
`OUTSIDE_PATH_MIN` and `OUTSIDE_PATH_MAX` when outside". Only `spawn_wands` uses them
(`-3,-3,-4,-4`); everything else passes `-1`, meaning no constraint.
`data/scripts/wang_scripts_removed.txt` lists three deliberately disabled entries:
`spawn_persistent_teleport`, `spawn_chest`, `spawn_blood`.

## `RandomColor` — also dead in the shipped data

`<RandomMaterials>` maps template colours to a palette:

```xml
<RandomColor input_color="FF00BBEE" output_colors="FF000000,FFFFFFFF"/>
```

`RandomColor::LoadFromFile` (`0x00865920`) reads `input_color` and `output_colors`; it is a
**palette remap, not a randomiser** — there is no RNG in it.

`FF00BBEE` and `FF12BBEE` (the only two rules the game ships, both in `coalmine.xml`) appear in
**zero** shipped templates, so both rules are dead. Where the 1→N substitution set is consumed is
not identified.

## `static_tile`: the shortcut

`static_tile` (default `0.5`, so off) does not randomise:

> if 1, directly blits `wang_template_file` without wang randomization, kind of like a pixel
> scene but at the wang tile resolution

With it set, the template is used as a literal tile image — no density field, no colour lookup,
one tile per texel. `static_tile_bg_mask` (a **string path**, not a float) and
`static_tile_bg_mask_threshold` then control a shader (`sprite_static_tile_bg.frag`) that decides
which background pixels are visible through the tile, using a mask image that must be "same size
and resolution as wang_template_file".

## `extra_layers` and debug output

`data/wang_tiles/extra_layers/` holds `coalmine.png` and a `readme.txt`. The load-time log prints
both the template and the extra layer:

```
data/wang_tiles/coalmine.png
data/wang_tiles/extra_layers/coalmine.png
```

Three debug images are written at load when the debug paths are taken:

```
temptemp/biome_wangs/                     per-biome wang maps
temptemp/_biomes_all_wang.png             all biomes, one sheet
temptemp/_biomes_all_wang2.png            the same, second pass
```

## Open questions

- What `0xFFC0FFEE` actually does. It is provably reserved and provably present in shipped
  content, and its effect is not identified.
- Where `RandomColor`'s substitution set is consumed, and what fills the map's `+0xb0` member.
- Which function copies the weight plane into the pixel chunk.
- Why the `+0x94` scale (= `width * 5`) offsets one axis by half the map width.
- The exact plumbing from a colour to the CSV row ordinal, and the precedence between the CSV and
  `RegisterSpawnFunction`. The id appears to be the row ordinal, but the reader is outside the
  recovered band.