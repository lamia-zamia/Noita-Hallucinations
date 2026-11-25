# Zoom: how big the world view is, and changing it while the game runs

Noita has no zoom. The size of the world view is the magic number pair
`VIRTUAL_RESOLUTION_X` / `VIRTUAL_RESOLUTION_Y` (427 x 242). Several objects copy it when the
world is created, and nothing recomputes those copies later. This page lists every place that
value goes, separates what is read every frame from what is fixed at world creation, and gives
the set of writes that gets the engine back into a consistent state at a different size. That
set is a runtime zoom with no rendering hack: the game draws exactly as it would have if it had
started at that resolution.

Addresses are for one Steam build (`Noita-Build-Jan 25 2025-15:55:41`, 32-bit `noita.exe`,
image base `0x00400000`, no ASLR) and will move on update. The `noita_dev.exe` shipped beside it
is a separate compile with different addresses; see [the end of the page](#noita_devexe).

**Evidence level.** Everything under "How the view is sized" is statically verified (read off
instructions and call paths). Two things are also confirmed in the game, on `noita_dev.exe`.
First, a mod's `VIRTUAL_RESOLUTION_*` magic numbers do set the view size of the new world.
Second, the recipe under "Changing the size at runtime" zooms in and out mid-run, across 12
steps from 0.5x to 2.67x. Lighting alignment, aim and simulation at the edges have not been
reported on yet.

## The short version

| question | answer |
|---|---|
| Is there a zoom in the engine? | **No.** The only "zoom" in the executable is the shader uniform `camera_inv_zoom_ratio`, which is fixed when rendering starts (see below) |
| What decides how much world you see? | `VIRTUAL_RESOLUTION_X` world pixels across. The height follows the window's aspect ratio |
| Screen pixels per world pixel | `window_width / VIRTUAL_RESOLUTION_X`, the same factor horizontally and vertically. 1920 px wide gives 4.496 |
| Can a mod change it at startup? | **Yes.** `ModMagicNumbersFileAdd` runs before the world is created, and that is when every copy is taken |
| Can it change mid-run? | Not by itself. Writing the global alone only moves the parts that read it every frame. With the extra writes below it can, up to the size the world was created with |
| Can it grow past the startup size? | **No.** The world's pixel buffers and textures are allocated once from the startup size |
| Does it change the simulated area? | **Probably.** The grid's update rectangle is derived from the camera rectangle (inferred, see below), so zooming in would shrink the simulated area and zooming out would grow it, as a different startup resolution does |

## How the view is sized

### The globals

| what | address | type | default |
|---|---|---|---|
| `VIRTUAL_RESOLUTION_X` | `0x01153128` | int | 427 |
| `VIRTUAL_RESOLUTION_Y` | `0x01152558` | int | 242 |
| `GRID_RENDER_BORDER` | `0x011532FC` | int | 2 |
| `VIRTUAL_RESOLUTION_OFFSET_X` / `_Y` | `0x01152710` / `0x01152658` | float | -1 / -1 |
| window width / height, pixels | `0x01221BCC` / `0x01221BD0` | float | runtime |

Below, `VRX` and `VRY` are the two virtual-resolution values and `border` is `GRID_RENDER_BORDER`.

### What is copied when the world is created

These are computed by the world loader. The options screen's resolution change does not
recompute any of them; it only rebuilds render targets (below). Only four places in the
executable compute `window_width / VRX`: the world loader, the background init, the camera's
screen-shake scale and the item card. A later copy elsewhere (one function copies a view's scale
to another view) is not excluded.

| copy | where | value | written by |
|---|---|---|---|
| grid world pixel buffers | three `w * h * 4` byte buffers, sizes at grid `+0x464` / `+0x468` | `w = VRX + 2*border`, `h = VRY` | grid constructor `0x00717C50` -> `0x007184D0` |
| camera size | world object `+0x08` / `+0x0C` (floats) | `VRX + 2*border`, `VRY` | world init `0x006F1230`, copied from the grid |
| world view scale | both views of the world object (`+0x18`, `+0x20`), each at `+0x78` / `+0x7C` | `window_width / VRX`, the same value in both | world loader `0x006AFAA0` |
| grid sprite textures | two textures the size of the grid buffers | `w x h` | `0x006F63D0` |
| background renderer | `Game+0x4C`: `+0x60` scale, `+0x64` / `+0x68` virtual width, `+0x6C` / `+0x70` window width (all floats) | `window_width / VRX`, `VRX`, `window_width` | `0x00888C80` |
| `camera_inv_zoom_ratio` | renderer `+0x208` | `VRX / 427` | render init `0x0088A9C0` |
| fog-of-war texture side | renderer `+0x20C` / `+0x210` | `64 * camera_inv_zoom_ratio` | `0x0088B2E0` |
| a quarter-resolution texture | owned by the grid | `VRX/4 x VRY/4` | `0x008613A0` |

The world object is `*(Game + 0x0C)` and `Game` is `*(Game**)0x0122374C`. The grid world is
`*(world + 0x44)`. The renderer is `*(*(Game + 0x50) + 0x10)`.

`camera_inv_zoom_ratio` is the reason the vanilla `post_final.vert` scales its fog texture
lookups. It is set from `VRX` once, when rendering is initialised. It lets the fog of war cover a
larger view when the game *starts* at a larger virtual resolution. It is not a zoom control.

### What is read every frame

| reader | what it does with the value |
|---|---|
| camera update `0x00636580` | rebuilds the camera rectangle as `centre +/- size/2` from the camera size above, positions both world views at the rectangle's origin, writes it to the grid (`+0x498..+0x4A4`, which is what `GameGetCameraBounds` returns), and uses `window_width / VRX` for screen shake |
| post-process uniforms `0x0088CA30` | `world_viewport_size = (VRX, VRX * window_height / window_width)`; `camera_pos` from the camera rectangle |
| grid tile renderer (`grid::GridWorld::RenderTiles`, `0x00723650`, built by `0x00722EF0`) | walks the **camera rectangle** in 32 x 32 tiles, clipped to the buffer size, and writes cells into the buffer with the buffer width as stride |
| grid sprite draw `0x006F66A0` | moves the sprite to the camera rectangle's origin and uploads **both buffers whole** to the two textures |
| world item card `0x00B90110`, `0x005673A0`, `0x00C42D30` | screen-space maths and distance tests with the current `VRX` |

So the cost of drawing cells follows the camera rectangle, while the texture upload follows the
size the buffers were allocated with.

### Render targets

The renderer's target rebuild, `0x0088BAB0`, is what the options screen calls after a
resolution or window-mode change. It also runs during world initialisation. It packs its inputs
(the low-resolution flag, `border`, `VRX`, `VRY`, window and back-buffer sizes) into a key,
compares it with the cached key at renderer `+0x1D0`, and returns at once if nothing changed.
Otherwise it destroys and recreates the targets:

- normal mode: the main targets are back-buffer sized;
- `rendering_low_resolution`: they are `(VRX + OFFSET_X) x (VRY + OFFSET_Y - 1)`, which is
  426 x 240 by default;
- three further targets are `(VRX + 2*border) x VRY` in either mode.

It takes the renderer in `ECX` and nothing on the stack (`__fastcall`, one argument, plain
`ret`).

### The update rectangle

When the camera rectangle changes, the camera's bounds setter (`0x006F2D20`) passes it to the
grid (`0x00720130`). The grid derives a second rectangle at grid `+0x4A8..+0x4B4`: the camera
rectangle's origin snapped to 64 pixels and moved 64 up and left, and its size plus 128. It is
handed to the grid's chunk/update bookkeeping. That makes it, by every sign, the simulated and
streamed area. This step is **inferred**: the consumer was not traced.

### How pixels look

At any virtual width, one world pixel is one texel of the grid texture, drawn with
`window_width / VRX` screen pixels per texel. When that factor is not a whole number (vanilla at
1920 px: 4.496), world pixels come out alternately 4 and 5 screen pixels wide. The graphics
option "pixel art anti-aliasing" (`rendering_pixel_art_antialiasing`) softens those edges. It
selects the shader variant that `data/shaders/post_cell_res_blit.frag` implements with a
bilinear trick, and the render init only enables it when low-resolution rendering is off. Whole
numbers come from `VRX = window_width / n`. At 1920 px those are 640 (3 px), 480 (4 px),
384 (5 px), 320 (6 px) and 240 (8 px).

## Changing the size at runtime

*Derived from the facts above; not yet run in the game.*

The buffers cannot grow after the world is created, so allocate them for the furthest zoom-out
first, then pick a smaller active size at runtime:

1. **At mod init** add a magic-numbers file with `VIRTUAL_RESOLUTION_X` / `_Y` set to the
   furthest zoom-out you want (for example 854 x 484 for 2x). The world is created at that
   size.
2. **On the first world update**, check that the copies still hold the startup values: grid
   buffer size `= (VRX + 2*border, VRY)`, camera size equal to that, view scale
   `= window_width / VRX`, background `+0x64 = VRX`. If anything differs, the build or another
   mod has changed the layout. Stop.
3. **To set an active size `x` by `y`** (with `x <= startup VRX` and `y <= startup VRY`):
   - `VIRTUAL_RESOLUTION_X = x`, `VIRTUAL_RESOLUTION_Y = y`;
   - camera size `= (x + 2*border, y)`;
   - both views: `+0x78 = +0x7C = window_width / x`;
   - background renderer: `+0x60 = window_width / x`, `+0x64 = +0x68 = x`;
   - call the render-target rebuild `0x0088BAB0(renderer)`.

   Leave the grid buffers, the grid textures and `camera_inv_zoom_ratio` alone. They stay at
   the startup size, which is larger than or equal to everything they now have to cover.

Why this should hold together:

- The next camera update centres a rectangle of the new size on the same point. That moves the
  views, the grid's camera rectangle, the update rectangle and the `camera_pos` uniform in
  step.
- Cells are rendered only inside the camera rectangle, into the top-left of the larger buffer.
  The sprite is placed at the rectangle's origin, so the unused right and bottom parts of the
  texture fall outside the view.
- Every per-frame reader of `VRX` (post-process, screen shake, item cards) now sees the active
  size.
- The fog texture keeps its startup size and ratio. The shader centres it with
  `world_viewport_size`, so a smaller view simply uses less of it.

Limits and costs:

- **Allocation is the zoom-out ceiling.** The texture upload is the startup size every frame, at
  every zoom: two RGBA textures of `(VRX + 4) x VRY`, which is 3.3 MB per frame at 2x, against
  0.8 MB in vanilla.
- **Change the size in steps, not continuously.** Each change recreates the render targets, the
  same work as changing resolution in the options.
- **Glow is drawn from the grid textures, and vanilla assumes they match the glow targets.** The
  grid's second texture (cell glow plus the liquid mask, bound as `tex_glow_source` and
  `tex_glow_unfiltered`) keeps the allocated size. The three glow render targets are rebuilt at
  `(VRX + 2*border) x VRY` and so follow the zoom. Vanilla treats the two as the same size in
  three places:
  - `post_glow2.frag` reads the source with the glow target's own `0..1` coordinates;
  - the final pass's glow coordinate centres a `VRX`-wide window using the source texture's width;
  - `post_final.frag` samples both textures with that one coordinate.

  At any size below the allocation, glow and liquid distortion come out shifted and stretched.
  Three shader edits fix it, all mod-editable text:
  - `post_glow2`: divide the source coordinate by `one_per_glow_texture_size * allocated size`;
  - `post_final.vert`: move the glow u to `(border + t * VRX) / glow_width` plus the engine's
    subpixel term;
  - `post_final.frag`: scale the `tex_glow_unfiltered` lookups by `glow size / allocated size`.

  The last two need per-zoom values, which `GameSetPostFxParameter` can supply.
- **The rebuild also reallocates the replay recorder, and that is the freeze.** The rebuild's
  last call (`0x0088A3F0`; `0x00919490` in `noita_dev.exe`, called with
  `this = renderer + 0x50`) frees every texture of the replay/GIF recorder. It then allocates
  `floor(replay_recorder_max_budget_mb * 2^20 / (w * h * 4))` new ones, logging
  `RenderTextureRecorder - allocating N textures`. The size `w x h` is the main render
  target's size capped by `replay_recorder_max_resolution_x/_y`. A 1920 x 1080 back buffer with
  a 640 x 360 cap special-cases to 480 x 270, so a 500 MB budget is 1011 textures. Outside
  low-resolution mode that size depends only on the back buffer, so a zoom change does not need
  the reallocation. The function is `thiscall` with six stack arguments and one caller, so
  writing `ret 0x18` over its first three bytes (`55 8B EC`) for the length of the rebuild call
  skips it. Turning the replay recorder off in the options also makes it nearly free.
- **Keep each step under 512 pixels of width change.** The camera update has a jump detector.
  If the rectangle origin moves more than 256 pixels (`0x01053EC8`) in one update, it sets
  `Game+0x3C`, which makes later unforced camera updates return early. The origin moves by half
  the size change. What clears the flag was not traced.
- **The update rectangle follows the camera** (inferred above). Zooming in shrinks the area that
  is simulated and streamed. Zooming out grows it, and costs what that larger area costs.
- **The skylight texture covers about 2048 world pixels** (32 x 32 texels of 64 pixels, fixed
  in `post_final.vert`). The default fog texture covers the same. Views wider than that lose
  skylight at the edges.

## noita_dev.exe

The Steam install also ships `noita_dev.exe` from the same release. It is a different compile:
it has inline asserts and splits up functions that the retail build inlines, so retail byte
patterns do not carry over. The struct offsets used on this page are the same at every site
checked: `Game +0x0C/+0x4C/+0x50`, `+0x10` from there to the renderer, world
`+0x08/+0x0C/+0x18/+0x20/+0x44`, view `+0x78/+0x7C`, grid `+0x464/+0x468/+0x498..+0x4A4`,
background `+0x60..+0x70`, renderer `+0x208`. The globals and functions are at:

| what | `noita.exe` | `noita_dev.exe` |
|---|---|---|
| `VIRTUAL_RESOLUTION_X` | `0x01153128` | `0x012EC180` |
| `VIRTUAL_RESOLUTION_Y` | `0x01152558` | `0x012EB6A0` |
| `GRID_RENDER_BORDER` | `0x011532FC` | `0x012EB198` |
| window width (float) | `0x01221BCC` | `0x01344B9C` |
| `Game*` | `0x0122374C` | `0x0134632C` |
| render-target rebuild (`__fastcall`, renderer in `ECX`) | `0x0088BAB0` | `0x0091B9E0` |
| world loader's view-scale store | `0x006AFF3B` | `0x006E248B` |
| background renderer init | `0x00888C80` | `0x00917200` |

You can tell the two apart from `logger.txt`. Only `noita_dev.exe` prints
`NOTE: Running the game with data.wak` and the `Gridworld::CreateNewWorld - P1: ...` lines.
