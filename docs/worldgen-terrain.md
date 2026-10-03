# Terrain: how a pixel becomes a material

The per-pixel pass. One function, `ProceduralTerrain::GetMaterialAt` at `0x0087d0e0`, is called
once for each of a chunk's 262144 cells; it returns a `CellMaterial*`, and the caller writes it
or allocates a heap cell for it. This page is that function, all three of its branches, the
cave generator that feeds it, and the ore-placement algorithm in full.

For the biome record these values come from see [worldgen-biomes.md](worldgen-biomes.md); for the
tile system see [worldgen-wang.md](worldgen-wang.md).

## The dispatcher

```c
// 0x0087d0e0.c
biomeMap = FUN_0087d9a0(this);          // resolve the biome at this pixel
if (!biomeMap) return 0;
switch (biomeMap->type) {                // biomeMap + 0x04
  case 0:  /* PROCEDURAL */
      FUN_0087e110(this, biomeMap);      // shape the cave noise
      return FUN_0086d2a0(biomeMap->materials /* +0x2a4 */, x, y);
  case 1:  /* BITMAP */
      bmp = biomeMap->bitmap;            // +0x1b0
      u = ((x % bmp->w) + bmp->w) % bmp->w;
      v = ((y % bmp->h) + bmp->h) % bmp->h;
      value = bmp->data[bmp->w * v + u]; // packed RGBA, no interpolation
      if ((value & 0xffffff) == 0) return 0;          // black = air
      index = FUN_004ad000(factory->sorted /* +0xa0 */, value);
      break;
  case 2:  /* WANG_TILE */
      map = biomeMap->wangmap;           // +0x1d4
      tile = FUN_008712d0(map);
      if (tile < 1) goto procedural_fallback;
      material = FUN_004ad080(factory->indexed /* +0x28 */, tile);
      if (!material) goto procedural_fallback;
      threshold = material->min_y;       // CellMaterial + 0x238
      if (y < threshold) return 0;       // above the tile's floor: air
      // +0x23c picks one of three connectivity bookkeeping routines
      return material;
}
return FUN_004ad080(factory->indexed, index);
```

Three details worth knowing:

- **`BIOME_BITMAP` tiles.** `bitmap_filename` is read with `% width` and `% height` in both
  axes, so a bitmap smaller than the world repeats. There is no interpolation and no scaling.
- **`BIOME_WANG_TILE` falls back to procedural.** If the wang lookup yields no tile, or the tile
  index is not in the material list, control drops into the procedural path. A wang biome with
  a sparse template therefore generates procedurally in the gaps rather than failing.
- **A per-material x threshold decides air.** `CellMaterial + 0x238` is a `float`, compiled-in
  default `0.5`, compared against the pixel's **x** coordinate *expressed in wang tiles*
  (`x/10 + 0.05`). Below the threshold the pixel is air. Two neighbouring `CellMaterial` fields
  are used here and are **not** XML-registered, so they have no names: `+0x234` (default `1.0`)
  is passed to the same tile helper as a noise-blend amount, and `+0x23c` (default `0`) selects
  one of three tile-shape helpers. This is how a wang set draws a vertical boundary — a cave
  wall on one side, open space on the other — without any geometry.

`FUN_004ad000` is a linear search with a one-entry cache over the factory's material list,
matching `entry & 0xffffff` against `value & 0xffffff`. `FUN_004ad080` converts an index to a
pointer with `index * 0x290 + factory[0x18]` - i.e. **`CellMaterial* = index * 0x290 + base`**,
which is the formula a reimplementation needs.

## Type 0: procedural, and the material selector

Two stages. `0x0087e110` shapes the cave noise; `0x0086d2a0` turns the resulting value into a
material. The second is the interesting one and it is reproduced here in full, because it is
what actually decides what a pixel is made of.

```c
// 0x0086d2a0.c, restructured; offsets are into the 0x84-byte MaterialComponent.
// in_XMM3_Da is the cave noise value, computed by 0x0087e110 and passed in XMM3 by
// the caller chain (XMM3 is NOT set at the 0x0073b79a call site, so it arrives from
// inside GetMaterialAt).
if (!(aggregate_min <= v && v <= aggregate_max)) return 0;   // union over ALL components

for (rec = materials->components; rec < end; rec += 0x84) {   // already sorted by material_index
    if (rec->limit_y && !(rec->limit_min_y <= y && y < rec->limit_max_y)) continue;

    if (rec->is_polygon) {                                    // crossing-number test
        px = hash(x, 0x200) / 512.0;                          // -> [0,1]
        py = hash(y, 0x200) / 512.0;
        if (!FUN_00ddc9a0(&{px, py}, rec->polygon, rec->polygon_count)) continue;
    }
    if (rec->resolved_material /* +0x04 */ == 0) continue;

    t = v;
    if (rec->add_perlin /* +0x2C */)
        t = FUN_00872300((double)(x * rec->add_perlin_scale_x), 0.0) + v;   // x only

    if (!(rec->material_min <= t && t < rec->material_max)) continue;
    if (!rec->is_rare /* +0x38 */) return rec->resolved_material;           // short-circuit

    rx = rec->rare_offset_x + x * rec->rare_scale_x;
    ry = rec->rare_offset_y + y * rec->rare_scale_y;
    if (rec->rare_offset_by_seed) { rx += world_seed * K1; ry += world_seed * K2; }

    if (rec->rare_use_perlin) {
        g = rec->rare_use_fbm_perlin ? FUN_009296d0(rx, ry) : FUN_00872300(rx, ry);
        if (g <= rec->rare_required_min) continue;
        if (rec->rare_required_max < g) continue;              // note: max is INCLUSIVE
    }
    if (!rec->rare_use_polka) return rec->resolved_material;
    p = FUN_00872140(rec->polka_params, rec->rare_polka_probability)(rx, ry);   // (1-d^2)^3
    if (rec->rare_required_min < p && p <= rec->rare_required_max)
        return rec->resolved_material;
}
return 0;   // air
```

What that means in practice:

- **Two different quantities are banded, and the names are misleading.** `limit_min_y` /
  `limit_max_y` band the **integer pixel y**. `material_min` / `material_max` band the **cave
  noise value** (a float), after an optional perlin perturbation. They are unrelated gates.
- **Evaluation order is fixed**: global band, `limit_y`, polygon, material resolved, `add_perlin`,
  `material_min`/`max`, `is_rare` short-circuit, rare coordinate, perlin gate, polka gate. A
  component that fails any gate does **not** claim the pixel; the next one gets a chance. So the
  list is a priority list, evaluated in `material_index` order.
- **`material_index` is the sort key and nothing else.** The array is sorted once at load time
  by a comparator that is literally `rec->material_index < rec->material_index`. **Lower index
  wins, and the order of the `<MaterialComponent>` elements in the XML is irrelevant.** This is
  the single most useful thing to know when writing an ore list, and it is the opposite of the
  intuition.
- **`is_rare` defaults to `false`, and a component with `is_rare="0"` returns its material before
  any of the rare machinery runs.** That, and not the window, is why the compiled-in defaults
  make the flag a no-op.
- **The default window is *not* permissive.** `rare_required_min`/`max` default to `0.2`/`1.0`,
  but the perlin they would be compared against is scaled by **70.0** inside the noise function,
  so it is roughly `[-70, +70]` and only a sliver falls inside `(0.2, 1.0]`. The polka function
  returns `(1 - d²)³`, which is 0 almost everywhere, so `p > 0.2` selects only the cores of the
  dots. A component that sets `is_rare="1"` and leaves the rest at their defaults is therefore
  **highly selective**, not a no-op — which is why shipped ore files set `rare_required_min` to
  values like `0.371429`.
- **Both rare gates use the same window and both are `min < v <= max`** — inclusive at the top,
  exclusive at the bottom. The name "rare" means *rare within the band*, not rare globally.
- **`add_perlin` is one-dimensional**: `perlin(x * add_perlin_scale_x, 0)` — the second argument
  is a literal zero. `add_perlin_scale_y` at `+0x34` is parsed from XML and never read by this
  function.
- **`rare_scale_x` / `rare_scale_y` / `rare_offset_x` / `rare_offset_y`** build the coordinate fed
  to the rare noise, so ore blotches can be stretched and shifted per axis independently of the
  cave noise. A `rare_scale` of `0.0214286` makes blobs about 47 px across.
- **`rare_offset_by_seed`** adds the world seed times a per-axis constant, so the same biome has
  its ore in a different place each run.
- **`rare_polka_radius_low` / `rare_polka_radius_high` / `rare_polka_is_boxed`** shape the
  "polka dot" function: a periodic blob with a radius varying between the two bounds,
  optionally boxed to a square. `rare_polka_probability` scales it.

### The full `MaterialComponent` field table

0x84 bytes. Defaults are the compiled-in ones.

| offset | field | default | meaning |
|---|---|---|---|
| +0x04 | `celldata` | `NULL` | resolved from `material_name` at load |
| +0x08 | `material_name` | `""` | the material to place |
| +0x20 | `material_index` | `10` | **sort key, ascending** |
| +0x24 | `material_min` | `0.1` | inclusive lower bound on the (possibly perlin-adjusted) **cave noise value** |
| +0x28 | `material_max` | `0.1` | exclusive upper bound on the same value |
| +0x2C | `add_perlin` | `false` | add `perlin(x * add_perlin_scale_x, 0)` to the value before the range test |
| +0x30 | `add_perlin_scale_x` | `1.0` | scale for that perlin |
| +0x34 | `add_perlin_scale_y` | `1.0` | parsed, never read — the perlin's second argument is a literal `0` |
| +0x38 | `is_rare` | `false` | enable the rare gates; **false short-circuits them entirely** |
| +0x39 | `limit_y` | `false` | gate on the pixel-y band below |
| +0x3C | `limit_min_y` | `100.0` | inclusive lower bound on **integer pixel y** |
| +0x40 | `limit_max_y` | `2048.0` | exclusive upper bound on **integer pixel y** |
| +0x44 | `rare_use_perlin` | `false` | |
| +0x45 | `rare_use_fbm_perlin` | `false` | use fbm instead of plain perlin |
| +0x46 | `rare_use_polka` | **`true`** | |
| +0x48 | `rare_scale_x` | `0.05` | |
| +0x4C | `rare_scale_y` | `0.05` | |
| +0x50 | `rare_offset_by_seed` | `false` | |
| +0x54 | `rare_offset_x` | `0.0` | |
| +0x58 | `rare_offset_y` | `0.0` | |
| +0x5C | `rare_polka_radius_low` | `0.2` | |
| +0x60 | `rare_polka_radius_high` | `0.65` | |
| +0x64 | `rare_polka_is_boxed` | **`true`** | |
| +0x68 | `rare_polka_probability` | `0.2` | |
| +0x6C | `rare_required_min` | `0.2` | exclusive lower bound for both rare gates |
| +0x70 | `rare_required_max` | `1.0` | inclusive upper bound for both rare gates |
| +0x74 | `is_polygon` | `false` | "if true, uses points inside polygon to determine if this material is placed" |
| +0x78 | `polygon` | empty | "points are within [0-1],[0-1]" |

Two defaults are worth calling out because they make a freshly written `<MaterialComponent>`
behave differently from what the attribute names suggest: **`rare_use_polka` defaults to true**
and **`rare_polka_is_boxed` defaults to true**, while `is_rare` defaults to **false**, which
means a component that omits `is_rare` never consults any of the rare machinery. The shipped XML
sets every one of them explicitly.

### `is_polygon`

With `is_polygon` set, the component claims a pixel by **point-in-polygon** instead of by the
cave value at all. The pixel's x and y are each hashed with a `0x200` grid and scaled by `1/512`
into `[0,1]²`, then tested against the polygon with a crossing-number algorithm. This is a real
alternative selection mechanism, not part of the value path — and no shipped biome uses it, so it
is a modding hook.

## The cave generator: `<BitmapCaves>`

`BitmapCaves` is a 0x2F0-byte object at `Biome + 0x00`, and it is what a procedural biome's
shape actually comes from. The parameters describe **how many caves to draw and how hard**; the
shape is perlin.

| offset | field | default | meaning |
|---|---|---|---|
| +0x04 | `name` | `""` | |
| +0x1C | `size_x` | `512` | "How big the image is that we generate these from" |
| +0x20 | `size_y` | `256` | |
| +0x24 | `spawn_percent` | `0.05` | "0-1 - with what chance do we add a spawn point to a cave" |
| +0x28 | `cave_count_min` | `50` | "generates caves n, where n = random(cave_count_min, cave_count_max)" |
| +0x2C | `cave_count_max` | `100` | |
| +0x30 | `do_beginning_paths` | `false` | "if set, will do the paths that go down near the center of the image, used for setting up the caves to coal mines near where player starts" |
| +0x31 | `do_beginning_down` | `false` | "if true, will do a hole straight down" |
| +0x34 | `surface_caves_count_min` | `7` | "generates caves n from the surface, where n = random(surface_caves_count_min, surface_caves_count_max)" |
| +0x38 | `surface_caves_count_max` | `12` | |
| +0x3C | `cave_strength_min` | `0.2` | "how strongly do we carve the cave" |
| +0x40 | `cave_strength_max` | `1.0` | |
| +0x44 | `cave_childs_min` | `0` | "how many child trails a cave can have" |
| +0x48 | `cave_childs_max` | `2` | |
| +0x4C | `surface_cave_childs_min` | `2` | |
| +0x50 | `surface_cave_childs_max` | `7` | |
| +0x54 | `mountain_count_min` | `0` | |
| +0x58 | `mountain_count_max` | `15` | |
| +0x5C | `mountain_size_min` | `1.0` | |
| +0x60 | `mountain_size_max` | `10.0` | |
| +0x64 | `blob_caves_count_min` | `20` | |
| +0x68 | `blob_caves_count_max` | `55` | |
| +0x6C | `blob_caves_strength_min` | `1.5` | |
| +0x70 | `blob_caves_strength_max` | `3.0` | |
| +0x74 | `blob_caves_radius_min` | `1.0` | |
| +0x78 | `blob_caves_radius_max` | `10.0` | |
| +0x7C | `structures` | empty | vector of `CaveStructure` |
| +0x88 | `mLuaScript` | `""` | "we share these based on the names, so make them unique" |
| +0xA0 | `DEBUG_output_image` | `""` | "if set, will save a png of the height map to the file specified here" |

Note that the defaults are for a *large* cave system: 50-100 caves, 20-55 blobs, 15 mountains,
on a 512x256 field. `coalmine.xml` overrides almost all of it down to 2 caves and 0 blobs,
because a wang-tile biome's caves come from its template image instead and only the
`CaveStructure` blits remain.

**The output is not an image.** This is the part that surprises people: for a procedural biome
the caves are written into the **wang tile map** at `Biome + 0x1D4` (registered as
`mBitmapNoise`), as four parallel byte planes (see [worldgen-wang.md](worldgen-wang.md)). The
per-pixel path for a procedural biome then reads that map, not a separate bitmap. A procedural
biome with no `CaveStructures` and no wang template gets a shared 1×1 dummy map, and therefore no
caves at all - just gradient and perlin.

### The carver and its seed

The carver is `0x00868f70` (it reads the `CavesSetup` it is handed - `cave_count_min`/`max` at
`+0x28`/`+0x2c`, `surface_caves_count_min`/`max` at `+0x34`/`+0x38`,
`cave_strength_min`/`max` at `+0x3c`/`+0x40`). It carries an inline Lehmer generator:

```c
seed = (int)seed * 0x41A7 - ((int)seed / 0x1F31D) * 0x7FFFFFFF;
if (seed < 1) seed += 0x7FFFFFFF;      // state stays in [1, 0x7FFFFFFF]
```

The state is carried between calls as a **`double`**, and is seeded by `FUN_0044d070` (halving the
input if it is `>= INT_MAX`).

**The seed is the world seed**, not a clock:

```asm
0086ba51  mov   eax, dword ptr [0x1205004]   ; the world-seed global
0086ba60  cvtdq2pd xmm1, xmm0
0086ba70  addsd xmm1, [eax*8 + 0x1054630]     ; unsigned fixup
0086ba75  call 0x44d070                        ; seed the Lehmer state
```

Three draws are taken from that world-seeded stream and kept; the **third** is what seeds the
carver (`0x0086bbb5` -> passed to `0x0086b830` at `0x0086b9f0`), and the same draw is reused for
the wang-template path (`0x0086bf5c` -> `0x0086b9f0`). So cave shapes are **deterministic per
world seed**.

### `CaveStructure`

Inside `<structures>`, each entry stamps an authored image into the cave map.

| offset | field | default |
|---|---|---|
| +0x04 | `image_file` | `""` |
| +0x1C | `count_min` | `50` |
| +0x20 | `count_max` | `100` |
| +0x24 | `aabb_min_x` | `50` |
| +0x28 | `aabb_min_y` | `100` |
| +0x2C | `aabb_max_x` | `50` |
| +0x30 | `aabb_max_y` | `100` |
| +0x34 | `strength_min` | `1.5` |
| +0x38 | `strength_max` | `1.5` |

The blit mechanics, at `0x00870250`: lazily allocate and zero each of the four planes, blit the
planes from the template (only those whose source element count is non-zero), copy the
template's 12-byte record list, **rebase every record by `(-dst_x, -dst_y)`**, and drop any
record whose new x or y is negative. That record list is what the ore/spawn pass walks, so a
`CaveStructure` blit is how a cave instance writes *both* its shape and its spawn points.

`coalmine.xml` ships one: `data/biome_impl/coalmine/dangerroom.png`, 2-4 copies, strength
1.45-1.55, inside `aabb 5..507 x 0..230`. The `data/biome_impl/<biome>/` directories are where
these images live.

### `FossilComponent`

A third material mechanism, image-based rather than noise-based.

| offset | field | default |
|---|---|---|
| +0x04 | `count_min` | `0` |
| +0x08 | `count_max` | `0` |
| +0x0C | `seed` | `1` |
| +0x10 | `material` | `"bone"` |
| +0x28 | `image_file` | `""` |
| +0x40 | `material_id` | `-1` |
| +0x44 | `mImageData` | `NULL` (runtime) |

`seed` and the `bone` default were recovered by reading the two 4-byte literals straight out of
`noita.exe`; an earlier pass had written them off as unrecoverable because the surrounding
`.rdata` padding is not covered by any string table.

## Writing the cell

After `GetMaterialAt` returns a `CellMaterial*`:

```c
if (material->cell_type /* +0x38 */ == 3) {
    table[(y - y0) * 512 + (x - x0)] = material->packed_value /* +0x30 */;
} else {
    cell = FUN_007048c0(factory, x, y, material, 0);      // GridWorld::CreateCell
    pointer_grid[(row << 9) | col] = cell;
}
```

`GridWorld::CreateCell` has **no case for type 3** and returns 0 - which is the whole reason for
the special case. Types 1, 2 and 4 allocate 0x40-, 0x3C- and 0x28-byte cells. The type-3 path is
therefore a flat material with no per-cell object, which is why terrain is cheap and liquids and
fire are not.

Two guards sit in front of the loop: the whole thing is skipped when the "world position key"
is null, and a pixel whose pointer-grid slot is already non-zero is skipped unless the chunk's
`+0x78` flag says otherwise (i.e. unless it is being force-regenerated).

There is also a **save-cache escape hatch**: if the chunk has a cache entry the worker builds
`world_<x>_<y>.png_petri` and, if that file exists, skips generation entirely.

## Ore and spawn requests

Materials are only half of it. The second thing the per-pixel pass collects is *where things
should be spawned*, and it does it on a completely different grid.

`0x0087d380` walks the chunk rect in **10-pixel strides**, resolves the biome at each sample,
and then:

- **`BIOME_WANG_TILE`**: reads the wang map's colour plane (`0x0092a3a0` over plane `+0x40`).
  `colour - 1` is the **wang script id**; `>= 0` means "spawn here", at
  `(tile_x * 10 - 5 - wang_offset_x, tile_y * 10 - 5 - wang_offset_y)`.
- **`BIOME_PROCEDURAL`**: reads the map's colour plane through `0x00871380` and keeps the raw
  sample pixel.

Accepted requests go into the chunk's `+0x98` vector as 0x20-byte records, **deduplicated** - the
appender scans for an existing record whose x, y, two flags, byte and script id all match and
returns without appending if it finds one.

| offset | meaning |
|---|---|
| +0x00 | x in pixels |
| +0x04 | y in pixels |
| +0x08 | **tile width, in tiles** — consumed as `(int) * 10` |
| +0x0C | **tile height, in tiles** — consumed as `(int) * 10`; `0/0` for procedural records |
| +0x10 | one byte from wang plane D, the mask |
| +0x14 | a spawn-script id at production time (colour − 1); **read as a pointer to a name** by the consumer, so the id is resolved somewhere in between — that step is not in the recovered band |
| +0x18 | "this record came from the procedural path" |
| +0x1C | the owning `Biome*`; the consumer reads `biome + 0x76`, the Lua file, from it |

Then `IMPL_CreateNewChunk_PART2` turns each record into a Lua call. The call is:

```c
FUN_0078ce40(lua, biome_lua_file_string, script_name_ptr,
             rec.x, rec.y,
             (int)rec.tile_width  * 10,
             (int)rec.tile_height * 10,
             (uint8)rec.mask);
```

so the rect is `(rec+0x00, rec+0x04)` with size `(rec+0x08 * 10, rec+0x0C * 10)`, and the mask byte
is passed separately as the eighth argument. If `rec+0x18` is set, a spatial rect query
(`0x0065dd70`, radius `64.0`) runs first and the record is skipped if it finds nothing.

This is where `spawn_items`, `spawn_wands`, `spawn_orb`, `spawn_perk`, `load_pixel_scene` and
the rest are invoked - once per coloured tile, with the rect, and with a 10-pixel grid. **A wang
image that is 348 x 448 with 20 script colours produces tens of thousands of spawn calls per
biome region**, which is why `spawn_percent` exists.

If the biome has no `lua_script`, `biome + 0x1D8` is empty and the game logs
`"Warning - biome: <name> - doesn't have lua file..."` and drops the record.

## The procedural noise shaping, as far as it is recovered

`0x0087e110` converts world pixels into wang-tile space using the map's `+0x94` scalar, blends
toward the surface, applies a blob cave when the biome's blob amplitude (`+0x268`) is non-zero:

```c
if (0.5 < v && biome->blob_amplitude != 0.0f) {
    w = (v - 0.5) * 2.0;      // remap 0.5..1 to -1..1
    FUN_0087e7a0(biome);
    ...
}
```

and then switches on the shape type at `Biome + 0x220` (values 1, 2, 3), where types 2 and 3
build `sin`/`cos` terms through libm. The gradient parameters
(`mGradientStartY`, `mGradientEndY`, `mGradientSlopeStartX` default `-512.0`,
`mGradientSlopeDelta` default `512.0`) feed that: with `mGradientAddNoise = 3` the slope term is
`(x - mGradientSlopeStartX) * mGradientSlopeDelta`, i.e. a linear ramp across the whole world
width by default.

**Not recovered**: the exact float constants behind two of the `mInside*`/`mGradient*` default
pairs. They are `.rdata` literals that no string table covers; they can be read out of the
executable, but they are build-specific, so a reimplementation should treat the parameter names
as the contract and read the constants from a build rather than trusting a table here.