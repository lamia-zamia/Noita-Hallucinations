# World generation: reference

Addresses, struct layouts, enums and the data-file inventory. The semantics live in
[worldgen.md](worldgen.md), [worldgen-biomes.md](worldgen-biomes.md),
[worldgen-terrain.md](worldgen-terrain.md), [worldgen-wang.md](worldgen-wang.md) and
[worldgen-pixelscenes.md](worldgen-pixelscenes.md); this page is the thing you look things up
in.

Addresses are for one Steam build of the 32-bit `noita.exe`, imagebase `0x00400000`. Function
behaviour is stable across updates; addresses are not.

## Function map

### Load time, once per world

| address | what it does |
|---|---|
| `0x0087a900` | `ProceduralTerrain::Init` - biome list, map script, wang maps, pixel-scene tables. Logs `ProceduralTerrain::Init( `, `ProceduralTerrain - worldseed: `, `Generating world...`, `World generation took: `, `ProceduralTerrain::Init took: ` |
| `0x0086b9f0` | `Biome::LoadFromFile` - parses one `data/biome/NAME.xml`. Asserts `Topology` / `Materials`, sets the runtime type at `+0x04` |
| `0x0086f4f0` | `BiomeMaterials::LoadFromFile` - resolves `material_name` to a handle, sorts by `material_index` |
| `0x0086c240` | the sort comparator: literally `rec->material_index < rec->material_index` |
| `0x00867de0` | wang map loader: expands the `wang_template_file` (via `0x00867500` and the herringbone generator `0x00866b90`) and attaches the biome's `lua_script`. It does not read `data/scripts/wang_scripts.csv` (that is read by `0x006e9f30` and `0x006afaa0`) |
| `0x0086b830` | builds the map for a `<BitmapCaves>` biome: a `size_x` x `size_y` map filled with weight 1.0, cached by the `BitmapCaves` name, then carved by `0x00868f70` |
| `0x00867c90` | map builder for a procedural biome's `bitmap_noise_file` |
| `0x00870de0` / `0x008704c0` | the wang-map builder (template image -> the four planes) |
| `0x0092a260` | the four-neighbour wang tile resolve |
| `0x00868f70` | the cave carver - reads the `BitmapCaves` record, inline Lehmer RNG |
| `0x00879090` | biome-map image loader |
| `0x0087edd0` | pixel-scene table loader (`data/biome/_pixel_scenes.xml`) |
| `0x0087f3d0` | pixel-scene prefab path |
| `0x008402f0` | `source/misc_utils/pixel_scene_splicer.cpp` |

### Per chunk

| address | what it does |
|---|---|
| `0x007422c0` | streaming system; owns `.stream_info`, `world_sim.bin`, `world_tree.bin`, `world_pixel_scenes.bin` |
| `0x007420f0` | drives `0x0073d6a0` while jobs remain, then saves each generated chunk |
| `0x0073d6a0` | threaded chunk dispatcher (walks the job tree) |
| `0x0073dab0` | creates a chunk record |
| `0x0073c220` | `IMPL_CreateNewChunk` - allocate the 1 MiB cell table, `Allocate64x64`, weather init, queue the worker |
| `0x0073b040` | **the per-pixel pass** - the only caller of `GetMaterialAt` |
| `0x0073c440` | `IMPL_CreateNewChunk_PART2` - the per-object pass |
| `0x0073eed0` | `IMPL_CreateNewChunk_PART1` - Box2D bodies |
| `0x00729470` | `GridWorldThreadImpl::Allocate64x64` |
| `0x007048c0` | `GridWorld::CreateCell` - no case for cell type 3 |
| `0x00704a30` | air pixels inherit a neighbouring background/vegetation item |
| `0x008a8040` | append a found tree/decor to the chunk's `+0x80` list |
| `0x0073af60` | builds the 128 x 128 coarse grid from the pointer grid |
| `0x00721870` | 3x3-neighbour object re-assignment, from the per-pixel worker |
| `0x00721da0` | 3x3-neighbour object re-assignment, from `PART2` (also used by pixel-scene placement and two entity paths) |

| `0x0087ec40` | end of batch: prune the tree list, then pixel scenes |
| `0x00882dd0` | pixel-scene streaming (`Streaming - loading shit to get shit into the screen`) |
| `0x00880fb0` | pixel-scene placement (`PIXEL SCENE SUCCESS!`) |

### The pixel path

| address | what it does |
|---|---|
| `0x0087d0e0` | `GetMaterialAt` - the three-way type switch |
| `0x0087d9a0` | resolve the biome for a world position |
| `0x0087e110` | procedural cave-noise shaping (gradient, blob caves, shape type at `+0x220`) |
| `0x0086d2a0` | `MaterialComponent` selection - the ore algorithm |
| `0x008712d0` / `0x00871380` | wang tile lookup (`uint16` plane) / colour lookup (`uint8` plane), round-to-nearest + wrap |
| `0x00870e60` / `0x00870f80` / `0x00871160` | connectivity bookkeeping, chosen by `CellMaterial + 0x23C` |
| `0x004ad000` | material-value -> index, linear search with a one-entry cache over `factory + 0xA0` |
| `0x004ad080` | index -> `CellMaterial*`: `index * 0x290 + factory[0x18]` |
| `0x0092a1d0` / `0x0092a3a0` / `0x0092a430` | the three typed wang-plane accessors |

### The object path

| address | what it does |
|---|---|
| `0x0087d670` | chunk-boundary containment test (three points) |
| `0x0087d730` | the same test for pixel sprites |
| `0x0087d7e0` | decor: place now or queue |
| `0x0089e010` | `push_back` onto the pending-object vector (20-byte records) |
| `0x0086ef30` | `BiomeMaterials::InsertTree` |
| `0x0086da50` / `0x0086dec0` / `0x0086e6a0` | the three tree-placement variants |
| `0x0086f030` | background images |
| `0x0086f3c0` | pixel sprites -> pixel scenes |
| `0x0087d380` | collect ore/spawn requests on the 10 px grid |
| `0x0092adf0` | the deduplicating appender |
| `0x0065dd70` | the spatial rect query used before a decorated spawn |
| `0x0078cfb0` / `0x0078c710` / `0x0078c270` / `0x0078ce40` | the biome Lua lookup and call wrappers |

## Struct sizes and offsets

| struct | size | where | default table |
|---|---|---|---|
| `Biome` | 0x2F0 | one per biome | [worldgen-biomes.md](worldgen-biomes.md) |
| `BiomeModifiers` | 0x18 | `Biome + 0x170` | [worldgen-biomes.md](worldgen-biomes.md) |
| `BitmapCaves` (`CavesSetup`) | at least 0xB8 | pointer at `Biome + 0xC0` | [worldgen-terrain.md](worldgen-terrain.md) |
| `CaveStructure` | 0x3C | inside `BitmapCaves + 0x7C` | [worldgen-terrain.md](worldgen-terrain.md) |
| `MaterialComponent` | 0x84 | flat array at `BiomeMaterials + 0x08` | [worldgen-terrain.md](worldgen-terrain.md) |
| `FossilComponent` | 0x48 | flat array | [worldgen-terrain.md](worldgen-terrain.md) |
| `VegetationComponent` | 0xE0 | flat array | [worldgen-pixelscenes.md](worldgen-pixelscenes.md) |
| `CellMaterial` | **0x290** | `factory[0x18] + index * 0x290` | see below |
| `WangTileMap` | - | `Biome + 0x1D4` | four planes, [worldgen-wang.md](worldgen-wang.md) |

### The runtime biome record

`+0x00` is the vftable, `+0x08` is the `name`
string, `+0xC4..+0xC7` are the four `noise_*_edges` bools (defaulting true, true, false, false),
`+0x218` is `game_enemy_hp_scale`, `+0x220` is `noise_type` and `+0x268` is `mMultiplierPerlin`. What the generation code
actually uses:

| offset | meaning |
|---|---|
| +0x00 | `Biome::vftable` |
| +0x04 | **biome type**: 0 procedural, 1 bitmap, 2 wang tile |
| +0x08 | the `name` std::string |
| +0xC0 | unregistered slot, zero until set: pointer to the biome's `BitmapCaves` record (its `+0x1C`/`+0x20` are `size_x`/`size_y`, its `+0x88` the Lua script) |
| +0xC4 / +0xC5 / +0xC6 | `noise_biome_edges` / `big_noise_biome_edges` / `fat_biome_edges` |
| +0xC7 | `skip_edge_textures` |
| +0x1B0 | the loaded `bitmap_data` bitmap (its path is at +0x198) |
| +0x1B4 | `wang_template_file`, a path string |
| +0x1CC / +0x1D0 | `wang_map_width` / `wang_map_height` |
| +0x1D4 | `mBitmapNoise` - the built wang/cave map |
| +0x1D8 | `lua_script` |
| +0x1F0 | `pixel_scene` |
| +0x220 | `noise_type` (shape switch in the noise shaping: 1, 2, 3) |
| +0x224 | `mInsideNoiseType` |
| +0x228 | `mInsideNoiseFBM` |
| +0x268 | `mMultiplierPerlin`, tested `!= 0.0` by the noise shaping |
| +0x2A4 | the materials pointer (unregistered, but zero-initialised) |

`BitmapCaves` and `mBiomeMaterials` are nested child elements with **no slot of their own**;
they are reached through pointers.

### The chunk record

Audited against `0x0073c220`, `0x0073b040`, `0x0073c440`, `0x0073e950` and `0x0073af60`.

| offset | meaning |
|---|---|
| +0x20 / +0x24 | world pixel x / y of the top-left corner (`chunk << 9`) |
| +0x28 / +0x2C | width / height, always `0x200` |
| +0x30 | pointer to a 1 MiB array of **4-byte cell pointers** |
| +0x34 / +0x38 | the flat material array's width / height, set to `0x200` |
| +0x3C / +0x50 | `0x40000` dword count |
| +0x4C | a **second** 1 MiB buffer: the flat `uint32` material array, allocated lazily |
| +0x54 | last material written |
| +0x58 / +0x5C / +0x60 | chunk x, chunk y, load slot |
| +0x6C | weather/rain system |
| +0x70 | the world-position key (the biome map descriptor) |
| +0x74 | the cell factory |
| +0x78 | **overwrite existing cells** - when zero, a pixel whose cell slot is already non-null is skipped |
| +0x79 | **contents came from a save file** - set by the save loader; `PART2` runs generation passes only when it is zero |
| +0x7C | a 512x512 byte map followed by a **128x128** table of 16-byte records (the coarse grid, 4x4 step) |
| +0x80 / +0x84 | vector of 12-byte `{x, y, ptr}` records; filled from cave-structure edge hits |
| +0x8C / +0x90 | a second 12-byte-record vector |
| +0x98 / +0x9C | a **de-duplicating set** of 32-byte wang-tile match records |
| +0xBC / +0xC0 | a vector of 4-byte pointers |
| +0xC8 | finished flag, set under a lock |

`+0x78` is *overwrite* (not *skip*), and `+0x79` is a skip-generation flag set by the save loader.

**A chunk holds two separate 1 MiB buffers**, both `512*512*4`: one array of cell *pointers* and
one flat `uint32` *material* array. The sub-block levels are 64x64 (`Allocate64x64`) and 4x4 (the
coarse grid).

### The 0x20-byte ore/spawn record

| offset | meaning |
|---|---|
| +0x00 | x in pixels |
| +0x04 | y in pixels |
| +0x08 | tile width in tiles (1 for wang records, 0 for procedural); the Lua call receives it times 10 |
| +0x0C | tile height in tiles (same values); the Lua call receives it times 10 |
| +0x10 | a byte read from wang plane D |
| +0x14 | **wang script id** (colour - 1) |
| +0x18 | set for procedural records: a spatial rect query must succeed before the Lua call |
| +0x1C | owning biome |

### `CellMaterial` fields that matter here

| offset | meaning |
|---|---|
| +0x04 | the resolved handle `MaterialComponent` returns |
| +0x30 | the packed 4-byte material value written for a type-3 cell |
| +0x38 | **cell type**: 1, 2, 3 or 4. Only 1, 2 and 4 allocate a heap cell |
| +0x158 | copied into every new cell |
| +0x238 | float threshold (default `0.5`) on the pixel's x **in wang tiles**; below it the pixel is air |
| +0x234 | default `1.0`, passed to the tile helper as a noise-blend amount |
| +0x23C | default `0`; selects one of three tile-shape helpers |

## Enums

| enum | values | where |
|---|---|---|
| `BIOME_TYPE` (`type`) | `BIOME_PROCEDURAL` = 0, `BIOME_BITMAP` = 1, `BIOME_WANG_TILE` = 2 | `Biome + 0x04` |
| `NOISE_TYPE` (`noise_type`) | `IQ2_SIMPLEX1234` = 0, `IQ_SIMPLEX` = 1, `SIN_CAPPED_EVERYTHING` = 2, `SIN_CAPPED_SIMPLEX` = 3 | `Biome + 0x220` (default 0) |
| `GENERAL_NOISE` (`mInsideNoiseType`) | `IQNoise` = 0, `DirtyPeeNoise` = 1, `QemNoise` = 2, `WhiteNoise` = 3, `MixNoise` = 4, `SimplexNoise` = 5, `STB_Perlin` = 6, `FastBlockNoise` = 7, `SimplexNoise1234` = 8 | `Biome + 0x224` (default 5) |
| `FOG_OF_WAR_TYPE` (`fog_of_war_type`) | `DEFAULT` = 0, `HEAVY_CLEAR_AT_PLAYER` = 1, `HEAVY_CLEAR_WITH_MAGIC` = 2, `HEAVY_NO_CLEAR` = 3 | `Biome + 0x16C` (default 0) |

The values are the cases of the enum-to-name functions the config system uses to write each
enum, so they are exact. (Unlike `GUI_OPTION`, which is a Lua-side table - see
[enums.md](enums.md).)

## Data-file inventory

| path | what it is |
|---|---|
| `data/biome/_biomes_all.xml` | **the colour -> biome table**. `BiomesToLoad` with `biome_image_map`, `biome_offset_y`, and one `<Biome biome_filename= height_index= color=>` per entry |
| `data/biome/*.xml` | 150 biome definitions, `<Topology>` + `<Materials>` |
| `data/biome/orbrooms/`, `data/biome/tower/` | pixel-scene-only "biomes" |
| `data/biome/_pixel_scenes.xml` | the master pixel-scene list |
| `data/biome/_pixel_scenes_laboratory.xml`, `_pixel_scenes_newgame_plus.xml` | variants |
| `data/biome_impl/biome_map*.png` / `.lua` | the map, 70 x 48, 1 pixel per chunk |
| `data/biome_impl/<biome>/` | per-biome `CaveStructure` images, wang extra layers, scenes, Lua |
| `data/biome_impl/spliced/` | authored rooms, merged by the pixel-scene splicer |
| `data/biome_impl/static_tile/` | `static_tile` biome definitions and their Lua |
| `data/biome_impl/biome_modifiers/` | `BiomeModifiers` presets |
| `data/biome_impl/*.png` | 155 images: the 11 `biome_map*.png` maps and the standalone scene images |
| `data/wang_tiles/*.png` | wang templates |
| `data/wang_tiles/extra_layers/` | wang overrides, composited on top of the template |
| `data/scripts/wang_scripts.csv` | **colour -> Lua function**, 30 entries |
| `data/scripts/wang_scripts_removed.txt` | three deliberately disabled entries |
| `data/scripts/biome_map.lua` | the map script `BIOME_MAP` points at |
| `data/scripts/biome_modifiers.lua` | the full modifier table, 41 KB |
| `data/scripts/biome_scripts.lua` | shared helpers for biome Lua |
| `data/scripts/biomes/*.lua` | per-biome `lua_script` |
| `data/weather_gfx/background_*.png`, `edges/` | biome backgrounds and edge seams |
| `data/weather_gfx/limit_y/*_left.*`, `*_right.*` | background height limiters |
| `data/vegetation/*.png`, `*.xml` | tree and grass images (`$[1-9]` notation) |

## Save-game interaction

| file | what it holds |
|---|---|
| `??SAV/world/world_<x>_<y>.png_petri` | the chunk-local pixel raster; if present, generation is skipped entirely |
| `??SAV/world/world_sim.bin` | one blob, deserialised to `world + 0x70` |
| `??SAV/world/world_tree.bin` | one blob, deserialised to `world + 0x6C` |
| `??SAV/world/world_pixel_scenes.bin` / `.xml` | which pixel scenes exist |
| `??SAV/world/.autosave_world_pixel_scenes` | autosave marker |
| `??SAV/world/.stream_info` | the stream registry |
| `??SAV/world_state.xml` | run-wide flags (`ENDING_HAPPINESS`, `day_count`, fog) |

A pixel scene that is not in the save manifest is dropped and regenerated on load, which is why
streamed-in rooms re-materialise after a reload.

## Reimplementation checklist

If you are writing a world generator that produces Noita-compatible worlds, the minimum is:

1. A **colour-indexed biome map**, one pixel per 512 x 512 chunk, with the colours resolved
   through a table rather than an index - because that is what the shipped game does and what
   every biome-map Lua hook assumes.
2. A **biome type tag** with three values, and a per-pixel branch on it.
3. For procedural biomes: a **scalar cave value** per pixel, plus an ordered list of
   (value-range, material, y-limit, optional rare-gate) records, evaluated in list order, first
   hit wins.
4. For wang biomes: a **tile id per pixel** obtained by rounding to the nearest tile and
   wrapping, plus a per-material float threshold on x (in wang tiles) that turns the tile into air below it.
5. A **10-pixel spawn grid** carrying a script id per cell, and a script table mapping ids to
   calls. Without the 10 px grid, spawn density is wrong by two orders of magnitude.
6. Chunk-boundary **containment tests on the object's extent, not its origin**, plus a deferred
   queue for objects whose chunk is not loaded - otherwise objects pop in as chunks stream.
7. A **coarse 128 x 128 grid per chunk** for skylight and the camera-bounded entity set. It is
   built from the cell *pointer* grid, so a chunk with no heap cells has no coarse grid entries.
8. Exactly **two cell representations**, flat material value and heap object, split on the cell
   type - because terrain, liquids, fire and gas have completely different costs and the game's
   performance depends on terrain taking the cheap path.

## What is not recovered

Stated once here rather than repeated on each page.

- The cell type names for 1, 2 and 4. Type 3 is established as "not an object"; the others are
  only established as "allocates 0x40 / 0x3C / 0x28 bytes".
- What `0xFFC0FFEE` does. It is provably reserved - the game refuses it in `materials.xml` "because
  it has a special meaning in the wang map" - and provably present in four shipped templates, 2801
  pixels in total. Its effect is not identified.
- How a template colour is mapped to a `wang_scripts.csv` row, and how the CSV and the per-biome
  `RegisterSpawnFunction` registrations are combined. Fourteen of the 30 CSV colours occur in
  shipped templates; the reader of the CSV is outside the recovered band.
- Whether `add_perlin_scale_y` is dead or read on a path not recovered.