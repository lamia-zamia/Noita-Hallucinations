# Patching `noita.exe`

Three complaints come up often enough that they got their own investigation: the save
directory grows without bound, the game stalls during explosions and heavy material
updates, and there is no way to have a genuinely separate second world. This page is
what the executable actually does, and what can and cannot be changed about it.

Everything here is from one Steam build: `noita.exe`, x86 32-bit, imagebase
`0x00400000`, 15 460 864 bytes on disk. Addresses move when the game updates, so every
patch below carries the bytes it was derived from. If they do not match your build, the
patch is wrong for your build — re-derive it, do not force it.

---

## 1. What the save is made of, and why it never shrinks

A save directory holds a handful of XML files at the top level and then a `world/`
subdirectory whose contents are **one file per thing that has ever been streamed**:

```
save00/
  player.xml                  player state
  world_state.xml             per-run world state (the Lua WorldState API's backing store)
  session_numbers.xml
  magic_numbers.xml           the live copy of data/magic_numbers.xml
  persistent/                 cross-run unlocks (orbs, bones, flags)
  world/
    world_<X>_<Y>.png_petri   one per chunk          <- the bulk
    area_<N>.bin              one per non-entity object graph
    entities_<N>.bin          one per entity group
    world_tree.bin            the world tree
    world_pixel_scenes.bin    baked pixel scenes
    world_sim.bin             liquid/gas simulation state
    .stream_info              streaming bookkeeping
    .autosave*                crash-recovery markers
```

The two numbers in `world_<X>_<Y>` are **joined by an underscore, not a path separator**,
and they are pixel coordinates, not chunk indices. The `.png_petri` extension is a lie
in the other direction too: despite the name, the file is not a PNG. It is the engine's
own LZ stream with two `uint32` length fields in front. (There is a string in the binary
saying *"loads the chunk from this bitmap, the bitmap should be 512x512"* — that is a
red herring from a different, unused path.)

A live save looks like this — note the negative indices and the 2000 stride:

```
save00\world\area_-1.bin     save00\world\area_-1998.bin
save00\world\area_-1999.bin  save00\world\area_-2000.bin
```

`area_<N>` and `entities_<N>` use a *different* key from `world_<X>_<Y>`: `major * 2000 +
minor`, sign-extended to 64 bits. They are stream ids, not spatial indices.

### Nothing ever prunes it

This is the actual answer to "why does it grow when I travel to PW": **there is no
pruning mechanism at all.** No distance test, no age test, no file-count cap, no dirty
flag, no deletion pass.

- Deleting is rare and always a one-shot event. The one function that removes `world_*`,
  `area_*` and `entities_*` files is `0x006b1170`, which lists `??SAV/world` and deletes
  **every file in it except `magic_numbers.xml`**. Its callers are all one-shot: new-game start-up, the
  delete-save path (it logs `Deleting save`), the "discard the crashed run's autosave" prompt,
  front-end menu actions and start-up handling. None is in the streaming code.
- The other `DeleteFileW` callers remove fixed names: `player.xml`, `world_state.xml`,
  `session_numbers.xml` and their `.salakieli` backups, the `bones_new` and flag directories, the
  stats and streak files, `mod_config.xml` / `mod_settings.*`, and the `.autosave*` markers.
- The file-listing calls (`FindFirstFileW`, `FindNextFileW`) live in the file-device layer; no
  streaming or eviction code uses them.

So file count == number of distinct chunks you have ever visited, and every one of them
leaves three permanent files. A long Parallel Worlds run walks a straight line through
chunk space, so it accumulates linearly and without limit.

There is a second, independent growth source: on every stats-repair event the game
*renames* `_stats.xml` to `_stats_<uuid>.xml` instead of overwriting it, permanently
adding two files each time.

### The trade-off you cannot patch away

The game's design is: keep roughly `3 * STREAMING_CHUNK_TARGET` chunks resident in RAM,
and evict the rest to disk. **Save size and RAM usage are two ends of the same trade-off.**
You can have bounded RAM or a bounded save, not both.

The obvious "fix" — stop writing chunks when they are streamed out — is a two-byte edit,
and it is the wrong patch. Setting the per-pass eviction count to `0` (`0x00743b4e`,
`c7 45 e4 01 00 00 00` → `c7 45 e4 00 00 00 00`) makes save size near-constant, but then
chunks are never evicted at all and resident chunks grow without bound. It does not
delete anything, so nothing is lost while the process lives; it just moves the growth
from disk into the heap. For a long run — the exact case it would be applied to — that
ends in an out-of-memory crash instead of a large save directory.

Practical things that do help, none of which need a patch:

- **Exclude the Noita directory from antivirus real-time scanning.** Thousands of small
  file creates is exactly the pattern that makes a filesystem filter expensive, and it is
  the most likely cause of save-time stutter even when the save itself is fast.
- **Prune externally.** Delete `world_<X>_<Y>.png_petri` outside a radius around where you
  are, keeping `world_state.xml`, `player.xml` and the recent chunks. Those chunks
  regenerate from the world seed if you walk back, so this trades revisitable terrain for
  disk. Never run it while the game is open.
- **Raise the residency budget instead**, so more chunks stay in RAM and fewer get
  written: `STREAMING_CHUNK_TARGET` in `data/magic_numbers.xml` (default `12`). This is
  the game's own documented tunable and needs no patch.

---

## 2. Why explosions and heavy materials stall the game

### The cell update loop is not budgeted

The simulation grid is `grid::IGridWorld`, implemented by `grid::GridWorldThreaded` /
`grid::GridWorldThreadImpl`. A chunk is **64×64 cells** (512×512 pixels — the binary
says so: `IMPL_CreateNewChunk_PART2() - Allocate64x64 returned false`). Per-cell state is
`grid::CellData`, updated through a virtual on `grid::ICell`.

The per-frame update at `0x0072a150` walks the merged dirty rectangle cell by cell with
**no cell counter, no time slice, and no ceiling**. The only loop bounds are the
rectangle's own width and height. Work per frame == width × height of whatever got dirtied,
and an explosion dirties a large area at once.

There is no fixed-timestep accumulator either — one render frame is one simulation step,
unconditionally — so this is not a spiral of death. The stall is pure per-frame cost, and
it is proportional to how much area one event dirties.

### The budget tunables are dead code

The binary contains the strings `GRID_MAX_UPDATES_PER_FRAME`,
`GRID_MIN_UPDATES_PER_FRAME` and `GRID_FLEXIBLE_MAX_UPDATES`. They look exactly like the
knob you want. They are referenced **only** by the two magic-number UI generators and by
nothing else — they are never read at runtime. The per-frame update budget does not exist
in this build.

That absence is also why the loop is the only place worth looking: there is no smaller
version of it to re-enable.

### The dirty-region cap is a trap

`GridWorldThreadImpl::IMPL_AddUpdateAABB()` takes dirty rectangles. At `0x00728aea` there
is `CMP ECX, 0x64` / `JLE` — a cap of 100 chunks on an incoming rectangle. This is the
most obvious-looking lever in the whole binary and it must not be touched:

> when a rectangle exceeds the cap, `IMPL_AddUpdateAABB` logs
> `trying to add a huge area. Might cause crashes` and **returns without queueing
> anything**.

Lowering 100 buys smooth frames by silently discarding simulation updates. The symptom
shows up much later as permanently stale, non-simulating material — correctness loss, not
a speed trade.

The rectangle size is instead governed by a streaming work queue whose capacity is
hard-capped at 300 items per pass (`world[0x5bc]`, see the patch table). Lowering that
is the safe way to smooth travel-time stutter.

---

## 3. A second, separate world

Short version: **the game already has the portal mechanic; it does not have a place to put
the second world.**

`TeleportComponent` takes an absolute world target (`target_x_is_absolute_position` /
`_y`, bound at `0x00785130`) and already waits for the destination's pixel scene to
generate before completing the jump (`safety_counter`, same function). Lua already gets
`portal_teleport_used(from_x, from_y, to_x, to_y)` and `teleported(...)`. So the portal
side is solved and is pure Lua.

What is missing is a second destination to point it at.

### Why "just teleport far away" has a small ceiling

Chunk coordinates are not free. In the per-frame update, immediately, disassembly for
every axis:

```
SAR  ECX, 0x9      ; chunk = pixel >> 9   (512 px per chunk)
SUB  ECX, 0x100    ; recentre on the origin
AND  ECX, 0x1FF    ; mask to 9 bits
SHL  ECX, 0x9      ; back to pixels
```

**The chunk coordinate is reduced modulo 512.** The world is a 512×512-chunk torus,
262 144 × 262 144 pixels. A true third dimension is not patchable: that 9-bit mask is baked
into many separate expressions across several functions, plus a second independent 18-bit
mask for the in-chunk raster.

### The real limit is smaller than the torus

The binding constraint is the biome map, a **single non-tiling 70×48-chunk window** — 35 840 × 24 576 pixels (`data/scripts/biome_map.lua`,
`BiomeMapGetSize`). That gives a maximum usable offset of roughly:

> **±17 920 px in x (35 chunks); y from −7 168 to +17 408 px (14 chunks above the origin, 34 below)** — about 1 490 player-widths in x.

The chunk index and float32 precision are nowhere near binding: they allow 262 144 px and
8 388 607 px respectively, i.e. 14.6× and 468× further out. If you teleport past the biome
window you get unpopulated or wrongly-biomed terrain, not a crash.

Worth knowing: Noita already has a world axis — `GetParallelWorldPosition` (`0x007a6800`)
computes `world = floor(chunk / biome-map chunks wide)` (70 in vanilla) on x, and likewise on y with
the chunks-high field. It simply never populates more than one tile. So a real second world means a
mod that *builds* tile ±1 out of pixel scenes, which is a large but ordinary mod project —
not a patch.

### Swapping the world pointer is not a patch either

The grid world hangs off one 4-byte field, `[[0x0122374c] + 0x0C] + 0x44`. In isolation
that looks like a one-byte patch. It is not: **2 058 cross-reference sites across 576
functions** reach the root global. And even if they were all rewritten, 11 further
singletons would not follow the swap — world seed (`0x01205004`), biome map, pixel-scene
tree, fog of war, streaming state — plus Box2D and the entity manager, and the streamer
evicts by camera distance with no way to pin a world. Keeping a second world alive while
the player is elsewhere is the part with no mechanism at all.

---

## 4. The patches

Two are enabled. Two are documented but disabled, because they are the first thing anyone
would try and they are wrong.

| Name | VA | Bytes | Issue | What it does | Risk |
|---|---|---|---|---|---|
| `autosave-marker-period` | `0x00743a24` | `c1 e6 04` → `c1 e6 06` | save | `.autosave*` marker rewrite period `DAT*60` frames → `DAT*252`. 4.2× less frequent. Cuts the part of the save that repeats on a timer when nothing changed. | Low. Wider crash-recovery window. |
| `stream-request-cap` | `0x0071fe8f` | `c7 83 bc 05 00 00 2c 01 00 00` → `c7 83 bc 05 00 00 96 01 00 00` | hitches | Caps the streaming work queue at 150 instead of 300 items per pass. Fewer long frames while travelling. Possible edge pop-in. | Low. Cannot corrupt a save. |
| `huge-aabb-cap` | `0x00728aea` | `83 f9 64` | hitches | **Disabled.** The dirty-rectangle cap; over-cap input is dropped, not clamped. | Correctness loss. |
| `no-evict-on-stream` | `0x00743b4e` | `c7 45 e4 01 00 00 00` → `c7 45 e4 00 00 00 00` | save | **Disabled.** Stops chunks being written when evicted. | Unbounded RAM; wrong direction for long runs. |

### Applying them

The rule is: **verify, then patch a copy, never in place.**

Map each virtual address to a file offset through the PE section table, and refuse to write a
single byte unless the bytes already in the file match the expected value exactly. That check is
what stops a patch from silently corrupting a build the addresses were not taken from. Check the
bytes first, write to a copy, and treat "bytes already patched" as a no-op so the procedure is
safe to run twice.

Addresses are absolute virtual addresses (imagebase included). `.text` starts at VA
`0x00401000` and maps to file offset `0x00000400`, so `file_offset = VA - 0x00401000 + 0x400`
for any address inside it (`.text` is RVA `0x1000..0xb06e1e`, file offsets `0x400..0xb06400`).

**Steam will not launch a modified `noita.exe`** — it verifies its own binaries. Either
launch the patched copy from something that does not go through Steam's launcher, or
replace `noita_dev.exe`, which Steam does not verify.

### Verifying a patch did what was predicted

- **`autosave-marker-period`** — count writes to `.autosave`, `.autosave_player`,
  `.autosave_world_state`, `.autosave_world_pixel_scenes` over a fixed stretch of play.
  They should drop by roughly 4×. Use Process Monitor filtered to
  `Path contains save00\world\.autosave`.
- **`stream-request-cap`** — walk in one direction at speed and watch frame time. Long
  frames get shorter; chunks arrive over more frames, so you may see slightly more edge
  pop-in. Note this changes travel stutter only. It does nothing for the explosion stall,
  which is the unbounded cell loop in section 2.

For profiling in general: the game ships a **built-in profiler that writes named counters**
to `profiler_game.txt` in the game directory — `grid_chunk_count`, `grid_active_count`,
`grid_particles_count`, `thread_pool_jobs`, `thread_pool_jobs_this_frame`,
`box2d_grid_world_bodies`, `world_tree_alloc_used` / `_reserved`, `entity_count`.
`grid_chunk_count` is the number to watch when testing anything about save growth, and
`grid_active_count` when testing anything about the cell loop.

---

## 5. Removed — do not reintroduce

- **`GRID_MAX_UPDATES_PER_FRAME` and friends are not tunables.** They exist as strings for
  the magic-number UI and are read nowhere. Do not write an article telling people to set
  them in `magic_numbers.xml`; it does nothing.
- **`world_<X>_<Y>.png_petri` is not a PNG.** The extension is historical; the payload is
  the engine's own LZ format. Tools that expect a valid PNG will fail on it.
- **`GameCutThroughWorldVertical` is not a teleport.** It is a vertical beam cut. Do not
  use it as the model for a dimension portal.
- **The chunk coordinate is not unbounded.** It is masked to 9 bits. Any plan that assumes
  an infinite world is wrong by a factor of 512.