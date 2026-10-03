# Pixel scenes, vegetation and decor

Everything that is not a material: the authored rooms, the parallax backgrounds, the trees, the
decor objects and the pixel sprites. All of it is placed in the *second* half of chunk
generation, after the pixels exist.

The step list is in [worldgen.md](worldgen.md); this page is what each of those steps does and
what the data files look like.

## Pixel scenes

A pixel scene is an **authored, fixed-position room**: a material image, an optional visual
(colours) image and an optional background image, stamped into the world at a literal
coordinate. This is how Noita's temples, vaults, the boss arena, the orb rooms and the sky are
made - they are not generated, they are placed.

### The format

`data/biome/_pixel_scenes.xml` is the master list:

```xml
<PixelScenes>
  <PixelSceneFiles>
    <File>data/biome_impl/spliced/lavalake2.xml</File>
    <File>data/biome_impl/spliced/boss_arena.xml</File>
    <File>data/biome_impl/spliced/tree.xml</File>
    ...
  </PixelSceneFiles>
</PixelScenes>
```

Each referenced file is itself a `<PixelScenes>` of `<PixelScene>` entries with **absolute world
coordinates**:

```xml
<PixelScene background_filename="data/biome_impl/spliced/tree/0_background.png"
            colors_filename="data/biome_impl/spliced/tree/0_visual.plz"
            material_filename="data/biome_impl/spliced/tree/0.plz"
            pos_x="-2016" pos_y="-1324"
            skip_biome_checks="1">
</PixelScene>
```

`pos_x`/`pos_y` are in world pixels, and they are negative because the world is centred on
`(0,0)`. `data/biome_impl/spliced/` holds about 20 such files, each with a subdirectory of the
actual images (`.plz` = material and colour layers, `.png` = background and visual).

There are three shipped variants of the master list: `_pixel_scenes.xml`,
`_pixel_scenes_laboratory.xml` and `_pixel_scenes_newgame_plus.xml`.

### The `PixelScene` fields

These come from the XML reader at `0x00873fe0`, which is the only place the offsets appear
explicitly - the reader stores the parsed value and names the attribute in the same basic block.
There are no compiled-in defaults, because nothing constructs a `PixelScene` without XML.

| offset | attribute | type | meaning |
|---|---|---|---|
| +0x04 | `pos_x` | int | absolute world x |
| +0x08 | `pos_y` | int | absolute world y |
| +0x0C | `material_filename` | string | the cell-material image; **required** - an empty one is rejected with `"material_filename is empty, we require a material_filename"` |
| +0x24 | `colors_filename` | string | the visual/colour layer |
| +0x3C | `background_filename` | string | drawn behind everything |
| +0x54 | `background_z_index` | int | |
| +0x58 | `skip_biome_checks` | bool | place even if the biome does not match |
| +0x59 | `skip_edge_textures` | bool | also sets the biome's `skip_edge_textures` |
| +0x74 | `clean_area_before` | bool | erase what is there first |
| +0x5C | `just_load_an_entity` | string | place an entity instead of blitting |

`clean_area_before` is the destructive one: it carves a rectangle before stamping, which is how
the game punches the entrance shafts into otherwise solid rock.

### Splicing

The directory is called `spliced` because of `source/misc_utils/pixel_scene_splicer.cpp`
(`0x008402f0`). The splicer takes several images and merges them into one `.plz` pair, which is
how a room authored from many separate PNGs becomes a single placeable scene. The tool's output
convention is visible in the file names: `0.plz` (materials), `0_visual.plz` (colours),
`0_background.png`.

### Placement

Two things place pixel scenes, and they are different:

1. **Streaming placement**, at the end of every generation batch: `0x0087ec40` prunes the
   terrain's tree list and then calls `0x00882dd0`, whose assert string is
   `"Streaming - loading shit to get shit into the screen"`. It creates the chunks the scenes
   need (`0x0073dab0` -> `0x0073c220`) and then calls `0x00880fb0`, the actual placement, which
   logs `"PIXEL SCENE SUCCESS! "`, `" - at pos( "`, `" at pos ("` and `") - at ( "`, and reports
   `"Error - Couldn't find biome at "` and `"Biome lua error - Biome ("` on failure.
2. **`LoadPixelScene`**, reachable from Lua as
   `LoadPixelScene( materials_filename, colors_filename, x, y, ... )`. This is the supported
   way for a mod to stamp a room at a chosen coordinate, and the same code path the game uses.

The biome flag `pixel_scene` ("for 'pixel scene' biomes, this forces the loading before hand")
makes a biome's scenes load eagerly rather than on demand, so that a room exists before the
player can see the gap.

Pixel scenes are also **saved**: `??SAV/world/world_pixel_scenes.bin` and `.xml`, plus a
`.autosave_world_pixel_scenes`. A streamed-in scene that is not in the save manifest is dropped
and regenerated on load, which is why pixel-scene rooms re-materialise after a reload.

## Backgrounds

Each biome names a parallax background and up to four edge images:

| attribute | offset | default |
|---|---|---|
| `background_image` | `Biome + 0x20` | `""` |
| `background_edge_left` | `+0x38` | `""` |
| `background_edge_right` | `+0x50` | `""` |
| `background_edge_top` | `+0x68` | `""` |
| `background_edge_bottom` | `+0x80` | `""` |
| `background_edge_priority` | `+0x98` | `0` |
| `background_use_neighbor` | `+0x9C` | `false` |
| `background_image_height` | `+0xA0` | `225.0` |
| `limit_background_image` | `+0xA4` | `true` |

At a biome boundary the two adjacent biomes each propose a background, and the edge images
decide the seam. The rule, from the in-source description: "if both biomes have edges defined,
will use the one with higher priority (if priority is same, will compare `(>)`
background_images)". `background_use_neighbor` makes a biome adopt a neighbour's image instead of
its own - which is how a strip of `mountain_hall_*` biomes shares one continuous background.

Placement is step 18 of the chunk pass, `0x0086f030`, which creates a liquid cell at `(x, y)`
from the component's material, loads the component's XML fragment as an entity and adds it to
the sprite container literally named **`"background"`**. If the world has no sprite container it
reports `"Error GetGameWorld() ... and GetGameWorld()->GetSpriteContainer(): "`. The height
limiter uses images from `data/weather_gfx/limit_y/`, suffixed `_left.` and `_right.`.

## Vegetation

`<VegetationComponent>` entries, 0xE0 bytes each, sorted and evaluated per pixel like
`<MaterialComponent>`. Trees are not "spawned at points" - the pass scans the generated pixels
for surfaces and puts a tree on the ones that qualify.

| offset | field | default | meaning |
|---|---|---|---|
| +0x04 | `is_visual` | `false` | "if 0 will insert the pixels into the world as cells. if 1 will add a single cell" |
| +0x05 | `is_real_pixels` | `false` | "if true will create the actual pixels into the world of the material given" |
| +0x06 | `is_grass` | `false` | "if set will treat this as grass and lay it on surfaces defined in `material_on_top_of` (if not set will put it on top of everything)" |
| +0x07 | `is_ceiling_plant` | `false` | |
| +0x08 | `check_safety` | `false` | "if set, will check the 4 corners for safety" |
| +0x0C | `rand_seed` | `1234.0` | "uses this to randomize the positions" |
| +0x10 | `tree_width` | `120.0` | "partitions the space into tree_width lengths; each partition might have a tree" |
| +0x14 | `tree_radius_low` | `0.3` | "within the tree width what's the min distance it can spawn in (0) all the way to the left" |
| +0x18 | `tree_radius_high` | `0.7` | "within the tree width what's the max distance (1) all the way to the right" |
| +0x1C | `tree_probability` | `0.7` | "what's the probability of a tree being spawned in a tree_width section" |
| +0x20 | `is_rare` | `false` | "if true, the tree is spawned only 1/3 of cases" |
| +0x24 | `tree_extra_y` | `2` | "if is_visual is true, then this will move it down by this much" |
| +0x28 | `tree_image_file` | `""` | "filename of the png that will get loaded as the tree. The filename can have the `$[1-4]` notation in it" |
| +0x40 | `tree_image_visual` | `""` | |
| +0x58 | `tree_material` | 4-byte literal | the cell material the tree is made of |
| +0x70 | `load_this_xml_instead` | `""` | |
| +0x88 | `visual_offset_x` | `0.0` | |
| +0x8C | `visual_offset_y` | `0.0` | |
| +0x90 | `visual_color` | `"0xFFFFFF"` | |
| +0xA8 | `grass_requires_neighbors` | `false` | |
| +0xAC | `material_on_top_of` | `""` | |
| +0xC4 | `height_check` | `false` | |
| +0xC8 | `max_y` | `99999` | |
| +0xCC | `random_flip_x_scale` | `false` | |
| +0xD0 | `mVisualColor` | `0` | runtime |
| +0xD4 | `tree_material_id` | `-1` | runtime |
| +0xD8 | `material_id_on_top_of` | `-1` | runtime |
| +0xDC | `mTreeData` | `NULL` | runtime |

`tree_width` is the key parameter and it is not what it sounds like: the world is partitioned
into `tree_width`-long vertical strips, and each strip independently rolls
`tree_probability` and then a position in `[tree_radius_low, tree_radius_high]`. That is why
trees form visible columns in a cave - the strips are in world space, not per chunk.

`rand_seed` is a per-component float, not a global; two components with the same geometry but
different seeds give different trees. The shipped files use values like `8376.86`, `1248`,
`507812`.

The `$[1-4]` and `$[1-9]` notation in image paths is a game-wide convention, not worldgen
specific: `data/vegetation/aerial_root_$[1-9].png` means "pick one of the nine".

Insertion is `0x0086ef30` (`BiomeMaterials::InsertTree`), which dispatches on the component's
flags:

- the `+4` branch with a noise mask set is a **2-D noise mask keyed by the object's offset from
  its spawn point**, which is how vegetation density clumps rather than spreading evenly;
- `is_ceiling_plant` branches to a different placement routine;
- the default branch is the one whose asserts are `"tree failed - no world chunk"`,
  `"tree failed - found a pixel at "`, `"tree failed - no bottom"` and
  `"tree failed - IsSafe() is false"`, and which loads `data/entities/vegetation/tree_entity.xml`.

## Decor and the chunk boundary

Decor objects, trees, backgrounds and pixel sprites all go through the same containment test,
`0x0087d670`:

```c
// 0x0087d670.c
offset = object->pixel_offset;             // +0xDC
if (!chunk_ring_contains(offset_x >> 9, offset_y >> 9)) return 0;   // chunk must be loaded
if (!FUN_0089c1b0(this, offset_x + x, offset_y + y)) return 0;
if (!FUN_0089c1b0(this, x,             offset_y + y)) return 0;
if (!FUN_0089c1b0(this, offset_x + x,  y))            return 0;
return 1;
```

Three points, not one, because a sprite is bigger than a pixel. `((coord >> 9) - 0x100) & 0x1FF`
is the **512 x 512 chunk ring** centred on the player - chunks `-256 .. +255`, which is the same
torus the world grid uses. An object whose chunk is not loaded is not placed now; it is pushed
onto a pending vector (`0x0089e010`, 20-byte records at `ProceduralTerrain + 0x88`) and
retried at the end of the next batch, which is what `0x0087ec40`'s pruning pass drains.

`0x0087d7e0` combines the two outcomes for decor, `0x0087d730` is the same test for pixel
sprites (using an offset at `decor + 0x44`).

## Pixel sprites

Distinct from pixel scenes: a pixel sprite is a **mask**, and the pass walks the mask's alpha
and places a scene at the first pixel that lands on empty grid:

```c
// 0x0086f3c0.c
if ((pixel->rgba & 0xff000000) != 0 &&
    (coarse_slot == 0 || pixel_slot == 0))
   ... spawn here ...
```

So: **one pixel scene per sprite mask, at the first transparent-grid pixel of the mask**, and
only if the containing chunk is loaded. The alpha test is `& 0xff000000`, i.e. fully opaque
only - a partially transparent mask pixel is not a candidate.

## The biome Lua hook

`lua_script` on the `<Topology>` defaults to `data/scripts/biomes/NAME.lua`. It is a different
mechanism from the wang scripts, and it is called at two different times:

| when | function | arguments |
|---|---|---|
| once per biome-map chunk at load | `init` | `(x, y, 512, 512)` |
| once per generated chunk | `init` | `(x0, y0, 512, 512, 0)` |

The per-chunk call is `0x0073c440` looking the function up **by name** - the 4-byte string it
loads is literally `"init"` - and then calling it by resolved index. The load-time call is the
by-name variant. Both go through `0x0078cfb0` (fetch the biome's Lua table), `0x0078c710`
(resolve a field by name) and `0x0078c270` / `0x0078ce40` (call it), which are thin wrappers over
`lua_getfield` and `lua_pcall`.

Per-chunk `init` only runs when the chunk's `+0x79` flag is clear. That is why a biome's Lua
script is the right place for "put one thing here if this chunk is at least so deep", and the
wrong place for anything that must happen exactly once.

The shipped scripts are `data/scripts/biomes/*.lua` and `data/scripts/biomes/mountain/*.lua`,
and they use `spawn_items`, `spawn_small_enemies`, `spawn_wands`, `SpawnEntity`,
`LoadPixelScene` and friends. `data/scripts/biome_scripts.lua` (7 KB) is the shared library they
pull in, and `data/scripts/debug_biomes.lua` is the debug entry point that loads a single biome's
script and dumps a spawn list.

## The other `data/biome_impl/` directories

| directory | what is in it |
|---|---|
| `_examples/` | template files for authoring a biome |
| `biome_modifiers/` | the `BiomeModifiers` presets |
| `caves/`, `coalmine/`, `crypt/`, `excavationsite/`, `laboratory/`, `liquidcave/`, `mountain/`, `overworld/`, `pillars/`, `pyramid/`, `rainforest/`, `snowcastle/`, `snowcave/`, `the_end/`, `trailer/`, `vault/`, `wandcave/`, `wizardcave/` | per-biome `CaveStructure` images, extra wang layers, biome-specific pixel scenes and Lua |
| `hidden/`, `spliced/`, `static_tile/` | authored rooms and `static_tile` biome definitions |
| `*.png` at the top level | 155 standalone pixel-scene images (rooms, arenas, shrines, the orb room) |

The top-level PNGs are the individual scenes; the subdirectories are what a biome XML refers to
from its `<CaveStructure image_file=...>` and `<PixelScene>` entries.