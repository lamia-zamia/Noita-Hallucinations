# World generation

How Noita builds a world: the biome map, the two generation phases, and which data files
control what.

World generation is not one algorithm. It is a colour-indexed biome map, a per-biome generator
chosen by a three-valued type tag, a material-selection pass that runs once per pixel, and a set
of Lua hooks. There is no single "worldgen settings" struct: the parameters are ~150 XML
attributes spread over five element types, plus a few `magic_numbers.xml` entries.

| page | covers |
|---|---|
| [worldgen-biomes.md](worldgen-biomes.md) | the biome file, every `Biome` field with its default, `BiomeModifiers` |
| [worldgen-terrain.md](worldgen-terrain.md) | how one pixel becomes one material: the three generators, cave carving, ore placement, spawn requests |
| [worldgen-wang.md](worldgen-wang.md) | wang tile templates, the four map planes, `wang_scripts.csv` |
| [worldgen-pixelscenes.md](worldgen-pixelscenes.md) | pixel scenes, backgrounds, vegetation, decor, the biome Lua hook |
| [worldgen-reference.md](worldgen-reference.md) | function addresses, struct layouts, enums, file inventory, reimplementation checklist |

Addresses are for one Steam build of the 32-bit `noita.exe` (imagebase `0x00400000`) and move
on update; the behaviour they describe does not.

## The shape of the world

| quantity | value | source |
|---|---|---|
| chunk | 512 x 512 cells, 1 cell = 1 pixel | hardcoded `0x200` at `0x0073c220` |
| world | 70 x 48 chunks | `BiomeMapSetSize( 70, 48 )` in `data/scripts/biome_map.lua` |
| world extent | x in [-17920, +17920), y in [-7168, +17408) | x origin = 35 x 512; y origin = `biome_offset_y` x 512 = 14 x 512 |
| cell storage | two 1 MiB arrays per chunk: cell pointers and flat `uint32` materials | `0x0073c220`, `0x0073b040` |
| seed | one `int32` at `0x01205004`, set by `SetWorldSeed` | `0x007c3d90` |

Biome lookup and chunk storage use different index schemes:

- **Biome lookup:** chunk index is `coord >> 9`. X wraps modulo the map width, Y is clamped to the
  first/last row. A position outside the 70 x 48 window therefore always resolves to some map pixel.
- **Chunk storage:** `((coord >> 9) - 0x100) & 0x1FF` - a 512-chunk torus biased by -256. It is
  about seven times wider than the biome map, so it only matters for very distant teleports.

## The biome map

The map is a 70 x 48 image, `data/biome_impl/biome_map.png`; **one pixel is one chunk**. The
engine does not load it directly. `magic_numbers.xml` sets
`BIOME_MAP="data/scripts/biome_map.lua"`, and that script runs:

```lua
BiomeMapSetSize( 70, 48 )
BiomeMapLoadImage( 0, 0, "data/biome_impl/biome_map.png" )
```

So mods can replace or edit the map from Lua with `BiomeMapSetSize`, `BiomeMapLoadImage`,
`BiomeMapLoadImageCropped`, `BiomeMapSetPixel`, `BiomeMapGetPixel`, `BiomeMapLoad_KeepPlayer`
and the deprecated `BiomeMapLoad`.

Each map pixel is an RGBA colour. `data/biome/_biomes_all.xml` maps colours to biome files:

```xml
<BiomesToLoad biome_image_map="data/biome_impl/biome_map.png" biome_offset_y="14">
  <!-- index is between 0 and 10 -->
  <Biome biome_filename="data/biome/hills.xml" height_index="0" color="ff36d517"/>
  <Biome biome_filename="data/biome/hills.xml" height_index="0" color="ff3d3e40"/>
  ...
```

- `biome_offset_y` is the map row that contains world y = 0 (the loader stores
  `biome_offset_y * 512` as the Y origin).
- The biome is identified by `color`. `height_index` is a separate integer (0-10 per the shipped
  comment); it is not the biome id.
- Several colours may point at the same file.
- A colour missing from the table leaves that chunk without a biome and logs
  `Biome load error - couldn't find biome for color: ...`.

Shipped map variants: `biome_map_demo`, `_niilo`, `_boss`, `_crypt`, `_lab`, `_metagame`,
`_trailer`, `_the_end` (`.png`), and `biome_map_newgame_plus.lua` / `.png`.

## Phase 1: load time, once per world

Runs inside `GameWorld::Load_Threaded` before the first frame, in `ProceduralTerrain::Init`
(`0x0087a900`). Its log lines bracket it:

```
ProceduralTerrain::Init(  ...  ProceduralTerrain - worldseed: <n>
Generating world...  ...  World generation took:  ...  ProceduralTerrain::Init took:  ...
```

1. **Read the biome list** from `_biomes_all.xml`: colour table, map path, `biome_offset_y`.
2. **Run the map script.** If `BIOME_MAP` names a `.lua` file it is executed in a
   `BiomeMapLuaManager`, which also receives the `OnBiomeConfigLoaded` callback and the
   `biome_modifiers_inject_spawns` hook - the supported way for a mod to change biome definitions
   before they are used.
3. **Load every biome file** (`0x0086b9f0`). A file without both `<Topology>` and `<Materials>` is
   rejected: `Error - Couldn't load biome: <file> - it lacked Topology or Materials (or the file doesn't exist)`.
4. **Load each wang template and biome script** (`0x00867de0`). The `wang_template_file` is
   expanded into a `wang_map_width` x `wang_map_height` image and the biome's `lua_script` is
   attached as its spawn dispatcher. On failure: `wang: <name> - couldn't load scripts`.
5. **Build each biome's map** - the expensive part of loading, done once per biome region, not per
   chunk. A wang biome gets a herringbone wang map; a procedural biome gets one carved from
   `<BitmapCaves>`, or its `bitmap_noise_file`, or a shared 1 x 1 dummy if it has neither. See
   [worldgen-terrain.md](worldgen-terrain.md#the-cave-generator-bitmapcaves).
6. **Load the pixel-scene tables** (`0x0087edd0`): `data/biome/_pixel_scenes.xml` and the files it
   lists.
7. Optional debug dumps to `temptemp/_biomes_all_wang.png` and `temptemp/_biomes_all_wang2.png`.

## Phase 2: per chunk, on demand

A chunk is generated the first time anything needs it. The streaming system (`0x007422c0`, owner
of `??SAV/world/.stream_info` and the `world_*.bin` files) hands jobs to `0x007420f0`, which runs
the dispatcher `0x0073d6a0` until the queue is empty and then saves each new chunk.

Generation is two functions:

| function | role |
|---|---|
| `0x0073b040` | **per-pixel pass**, on a worker thread. Calls `GetMaterialAt` for all 262144 cells, writes each material, and collects the decor, background, pixel-sprite and spawn lists. It is the only caller of `GetMaterialAt`. |
| `0x0073c440` | **per-object pass** (`IMPL_CreateNewChunk_PART2`). Physics bodies, decor, trees, backgrounds, sprites, the biome's Lua `init`, and one Lua spawn call per spawn record. |

### Ordered steps

| # | step | function |
|---|---|---|
| 1 | skip everything if the chunk has a `world_<x>_<y>.png_petri` save | inline |
| 2 | allocate and zero the 1 MiB cell table | `0x0073c220` |
| 3 | `Allocate64x64` - create the 64 chunk blocks | `0x00729470` |
| 4 | weather/rain initialiser | `0x006f31e0` |
| 5 | queue the per-pixel worker | lambda in `0x0073c220` |
| 6 | **per pixel:** resolve biome, compute material, write it | `0x0073b040` -> `0x0087d0e0` -> `0x007048c0` |
| 7 | air pixels inherit a neighbouring background/vegetation item | `0x00704a30` |
| 8 | append trees/decor to the chunk's object list | `0x008a8040` |
| 9 | append backgrounds | inline |
| 10 | collect spawn requests on a 10 px grid | `0x0087d380` |
| 11 | hand objects to the right one of the 3 x 3 neighbouring chunks | `0x00721870` |
| 12 | build the 128 x 128 coarse grid | `0x0073af60` |
| 13 | mark the chunk finished (under a lock) | inline |
| 14 | insert the coarse grid into the coarse map | `0x00861da0` |
| 15 | Box2D bodies | `0x0073eed0` (`PART1`) |
| 16 | decor | `0x0087d7e0` |
| 17 | trees (`BiomeMaterials::InsertTree`) | `0x0087d670`, `0x0089e010`, `0x0086ef30` |
| 18 | backgrounds | `0x0086f030` |
| 19 | pixel sprites | `0x0087d730`, `0x0086f3c0` |
| 20 | 3 x 3 neighbour object hand-off again | `0x00721da0` |
| 21 | call the biome's Lua `init(x0, y0, 512, 512, 0)` | `0x0078cfb0`, `0x0078c270` |
| 22 | one Lua call per spawn record | `0x0065dd70`, `0x0078ce40` |
| 23 | finalise | `0x006342e0`, `0x00739880` |
| 24 | once per batch: retry deferred trees, place pixel scenes | `0x0087ec40` -> `0x00882dd0` -> `0x00880fb0` |

Steps 6-13 are `0x0073b040`; steps 15-23 are `0x0073c440`. Steps 21 and 22 are skipped for a
chunk loaded from a save. The per-object steps are described in
[worldgen-pixelscenes.md](worldgen-pixelscenes.md).

Each pixel is stored in one of two ways, depending on the material's cell type: terrain-like
type 3 materials are a flat `uint32` value with no per-cell object; types 1, 2 and 4 allocate a
heap cell. Details in [worldgen-terrain.md](worldgen-terrain.md#writing-the-cell).

## What is not a parameter

- **World size.** 70 x 48 comes from `data/scripts/biome_map.lua`, which is replaceable.
- **Cave density.** Caves are per biome: `<BitmapCaves>` in the biome XML, or the wang template image.
- **Ore frequency.** Ore density is the set of `<MaterialComponent>` value ranges and their
  perlin/polka gates.
- **Additional seeds.** The world seed is a single `int32`; the cave RNG, wang offset and per-chunk
  RNG are all derived from it.

## `magic_numbers.xml` entries

"Shipped" is the value in the retail file; an empty cell means the file leaves the key at its
compiled-in default.

| key | shipped | effect |
|---|---|---|
| `BIOME_MAP` | `data/scripts/biome_map.lua` | the map script (or image) to load |
| `BIOME_PATH_FIND_HEIGHT_LIMIT` | `205` | vertical limit for biome pathfinding |
| `BIOME_PATH_FIND_WORLD_POS_MIN_X` / `_MAX_X` | (default `159` / `223`) | horizontal limits for biome pathfinding |
| `BIOME_APPARITION_CHANCE` | `30` | chance of a biome apparition |
| `BIOME_RANDOM_BLOCK_CHANCE` | | chance of a random blocked-off biome region |
| `BIOME_USE_BIG_WANG` | | use the large wang map variant |
| `NUM_ORBS_TOTAL` | `12` | orbs in the world |
| `WORLD_SEED` | | seed, if not set through Lua |
| `STREAMING_ENABLED`, `STREAMING_FREQUENCY`, `STREAMING_CHUNK_TARGET` (`12`), `STREAMING_PERSISTENT_WORLD`, `STREAMING_AUTOSAVE_PERIOD_SECONDS` | | how eagerly chunks are generated and saved |

## Troubleshooting edited data

| symptom | cause |
|---|---|
| `Biome load error - couldn't find biome for color: <rrggbb>` | a map pixel whose colour is not in `_biomes_all.xml` |
| `Biome - couldn't find a biome at: <x>, <y>` | the same problem, reported at chunk generation; the coordinates are the chunk centre |
| `Warning - biome: <name> - doesn't have lua file...` | the biome has no `lua_script`, so its spawn records are dropped |
| a wang biome that is mostly air | the template is mostly white/grey (density only, no material), or `wang_map_width`/`height` is too small for it |
| ore ignores its position in the XML | expected: `material_index` is the sort key, lower wins |
| nothing spawns on a coloured template pixel | the colour is not registered - neither in `data/scripts/wang_scripts.csv` nor by the biome's `RegisterSpawnFunction`. A biome cannot re-register a CSV colour; see [worldgen-wang.md](worldgen-wang.md#wang_scriptscsv) |
| an XML attribute has no effect | it is not a field the engine reads; see [worldgen-biomes.md](worldgen-biomes.md#what-the-loader-ignores) |
