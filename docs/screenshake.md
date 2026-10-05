# Screen shake: what it is, and how to read it

Noita's screen shake is a single **trauma** value with a per-frame random offset derived from
it, decayed by a fixed fraction every frame. It is three floats in one struct. The game never
exposes a getter — `GameScreenshake` writes it and nothing in the 375-function API reads it —
but the shake is *already baked into* the camera rectangle that `GameGetCameraPos` and
`GameGetCameraBounds` return, so it is observable either way.

Addresses are for one Steam build (32-bit `noita.exe`, image base `0x00400000`, **no ASLR** —
`DYNAMICBASE` is clear, so absolute addresses are stable for a given build) and will move on
update.

## The short version

| question | answer |
|---|---|
| Is there a Lua getter? | **No.** `GameScreenshake` is write-only and nothing reads it back |
| What state exists? | three floats: `intensity`, `offset_x`, `offset_y` |
| Where | the world object at `*(Game**)0x0122374C`, at `+0x58`, `+0x50`, `+0x54` |
| Does shaking accumulate? | **No** — it is a `max()`, so shakes replace rather than add |
| Can you cancel a shake? | **No** — not even with strength 0; the value only ever decays |
| How long does one last | `ceil(ln(0.01/strength) / ln(0.9))` frames — 59..92 frames for the strengths the game uses (5 to 150) |
| Is it framerate-independent? | **No.** The decay is per *frame*, not per second |
| What does the shake look like | white noise, not oscillation: a fresh uniform random offset every frame |
| Does `screenshake_intensity = 0` stop it being readable | **no** — the setting is only a pixel multiplier, applied last. It *does* make the camera perfectly steady, so camera-based reading returns zero |
| Can you see it without any of this | **yes**, at non-zero settings — it is already in the camera rect |

## Where the state lives

Every camera-moving path in the game goes through one function, which is also the only place the
shake is consumed. The chain to reach it from Lua-adjacent code is:

```
*(Game**) 0x0122374C        the game object, 0x1A0 bytes, created on first use
  +0x0C                    the world object: owns the camera rect, the grid world,
                           the render view, and the screen-shake state
    +0x00 .. +0x0C          camera rectangle as floats: x, y, w, h   (x,y = top-left)
    +0x18                  render view / transform object
    +0x44                  grid world
    +0x50                  shake_x   — this frame's x offset, in shake units
    +0x54                  shake_y   — this frame's y offset, in shake units
    +0x58                  intensity — the trauma value
    +0x5C, +0x60            NOT shake: a separate decaying scalar and its rate (see below)
```

`0x0122374C` is read by a one-line accessor that also lazily constructs the game object if it
does not exist yet, so the pointer is valid as soon as anything has asked for it.

The `+0x5C` / `+0x60` pair decays toward zero at a rate of `+0x60 * dt` and is driven by a
separate three-mode setter that takes a flags word and an amount. Whatever it does to the view,
it is **not** the screenshake — do not confuse the two when reading the struct.

## `GameScreenshake(strength, x, y)`

```
GameScreenshake( strength:number, x:number = camera_x, y:number = camera_y )
```

The implementation is a `max`, not an accumulate, and it silently ignores anything that would not
raise the value:

```c
if (camera->intensity > strength) return;       /* already louder: do nothing at all */

/* distance falloff, only if (x,y) is outside the camera rectangle */
if (x < cam.x || cam.x + cam.w <= x || y < cam.y || cam.y + cam.h <= y) {
    float r  = cam.w * 0.5f;
    float dx = (cam.x + cam.w * 0.5f) - x;
    float dy = (cam.y + cam.h * 0.5f) - y;
    float d2 = dx*dx + dy*dy - r*r;
    if (d2 < 0.0f) d2 = 0.0f;
    float falloff = d2 / ((cam.w * 2.0f) * (cam.w * 2.0f));
    if (falloff > 1.0f) falloff = 1.0f;
    strength -= falloff * falloff * strength;
}

if (camera->intensity < strength) {
    camera->intensity = strength;
    /* also queues a Message_CameraShake {x, y, strength} */
}
```

Consequences worth knowing before you call it:

- **`GameScreenshake(0)` does nothing**, and neither does a negative strength. There is no way
  to stop or shorten a shake from Lua.
- Two shakes never add. A second `GameScreenshake(5)` during a `GameScreenshake(100)` is a no-op.
- The default `x, y` is the **camera centre**, which is inside the rectangle, so the common
  one-argument form has no falloff at all.
- Falloff uses **only the camera width**, never the height, and the dead zone is a disc of
  radius `w/2` rather than the rectangle itself. A source point just outside the left edge of a
  640-wide camera is already at the edge of the dead zone; the same distance below the bottom
  edge is not. Effective strength reaches zero at about `2.06 * cam.w` from the centre.
- `GameScreenshake` is the only way *any* of this happens from Lua, but it is not the only
  source: explosions (`camera_shake`), projectiles (the `screenshake` field on
  `ProjectileComponent`), wand actions (`camera_shake_when_shot`), `LooseChunk` and the
  telekinesis/teleport component updaters all push trauma through the same function. A `max()`
  on a shared value means these sources also cannot stack.

Magnitudes the game itself uses, from `GameScreenshake` calls in `data/scripts`:

| strength | used by |
|---|---|
| 1 | boss limb / centipede hit ticks |
| 5 | light rain events, gold rain |
| 10-20 | weather events, worm rain, fireworks, sun spots |
| 30 | buildings collapsing, utility boxes, mimic rain |
| 80-100 | funroom, sun collision, destruction, mass polymorph |
| 150 | temple leaks |

## The per-frame update

Once per frame, in the world update:

```c
float I = camera->intensity;
if (I <= 0.01f) {
    camera->shake_x = camera->shake_y = camera->intensity = 0.0f;
} else {
    float lo = -I;                                  /* the bit pattern is XORed with 0x80000000 */
    unsigned r1 = lcg_next();                       /* seed lives in a single global, 0x343FD/0x269EC3 */
    unsigned r2 = lcg_next();
    camera->shake_y = (r1 >> 16 & 0x7FFF) / 32767.0f * (I - lo) + lo;
    camera->shake_x = (r2 >> 16 & 0x7FFF) / 32767.0f * (I - lo) + lo;
    camera->intensity = I * 0.9f;
}
```

- The offset is **uniform in `[-intensity, +intensity]`** on each axis, drawn fresh every frame.
  There is no smoothing and no oscillation — it is jitter, and it decays as a shrinking random
  walk, not as a sine.
- Both axes come from the **same global LCG** (a standard MSVC `rand`, seed in one global). The
  values are predictable if you know the seed, which is not reset per shake.
- **The decay is per frame, not per second.** At 144 Hz a shake lasts half as long in wall-clock
  time as at 72 Hz. Nothing in this path is `dt`-scaled.
- The cutoff is `<= 0.01`, so the tail is truncated, not faded. Frames from a shake of strength
  `S` to zero is `ceil(ln(0.01/S) / ln(0.9))`: 44 frames for `S = 1`, 73 for `S = 20`, 88 for
  `S = 100`, 92 for `S = 150` — about **1.0 to 1.5 seconds at 60 fps** for anything the game
  actually triggers at `S >= 5`.

## Turning shake units into pixels

The offset is applied to the camera target at the last moment, in the single function every
camera move goes through:

```c
float scale = (window_width_px / VIRTUAL_RESOLUTION_X)     /* both runtime globals */
            * 0.21629999577999115f
            * 0.5f
            * config.screenshake_intensity;               /* config field at +0xB8 */

target.x -= (camera->shake_x + DEBUG_CAMERA_SHAKE_OFFSET) * scale;
target.y -= (camera->shake_y + DEBUG_CAMERA_SHAKE_OFFSET) * scale;
```

| symbol | where | notes |
|---|---|---|
| window width in pixels | float global `0x01221BCC` | ~70 GUI layout sites use it as the horizontal extent |
| `VIRTUAL_RESOLUTION_X` | int global `0x01153128` | **427** by default — the `n_X` in `magic_numbers.xml` |
| `0.21629999577999115f` | float constant `0x0105354C` | fixed |
| `0.5f` | `0x0105361C` | fixed |
| `screenshake_intensity` | config field `+0xB8`, key `screenshake_intensity` | **0.7** by default |
| `DEBUG_CAMERA_SHAKE_OFFSET` | float global `0x012051C8` | debug config key, 0 unless you set it |

So peak displacement in pixels is `intensity * scale`, and because the window width is in the
numerator **the same shake is bigger in pixels on a bigger window** — it is tied to the zoom
level, not to the pixel grid.

The `screenshake_intensity` default was **1.0** before config format version 5 and **0.7**
after; the migration rewrites a stale 1.0 to 0.7. Your `save_shared/config.xml` is the
authoritative value for your install.

Worked example, `1920x1080`, `screenshake_intensity = 0.200893`, `VIRTUAL_RESOLUTION_X = 427`:

```
scale = (1920 / 427) * 0.2163 * 0.5 * 0.200893 = 0.0977  px per unit
```

| `GameScreenshake(n)` | peak offset, x and y independently |
|---|---|
| 5 | ±0.5 px |
| 20 | ±2.0 px |
| 100 | ±9.8 px |
| 150 | ±14.6 px |

At the default intensity of 0.7 on a 1280-wide window the scale is 0.227, so `GameScreenshake(20)`
peaks at ±4.5 px. This is why Noita's shake reads as subtle at low values: the unit is not a
pixel.

## What the setting gates, and what it does not

`screenshake_intensity` is an **accessibility slider**, range **0.0 to 1.0** on the options
screen, default 0.7. A great many players run it at 0. The important thing about it is *where* it
is applied, because that decides whether the values are still readable:

**It is read in exactly one place — the camera application, at the last step, as a multiplier on
the way to pixels.** Nothing upstream of that knows it exists:

| | affected by `screenshake_intensity = 0`? |
|---|---|
| `GameScreenshake` accepting the call, applying distance falloff, raising `intensity` | **no** |
| `intensity` accumulating and decaying by 0.9 per frame | **no** |
| `shake_x` / `shake_y` being drawn as fresh uniform random offsets | **no** |
| the `Message_CameraShake` being queued | **no** |
| the camera rectangle actually moving | **yes — it stops moving dead** |

The trauma write does not read the config at all. The whole shake pipeline runs at full strength
with the slider at zero; only the final `target -= shake * scale` multiplies by zero.

Two consequences:

1. **Reading `intensity` works perfectly at any setting, including 0.** The number you get is
   the game's *intent* — what the shake would have been at the default 0.7. This is the value
   to use if you want "how big a shake was that", independent of how the player has configured
   their game.
2. **Reading it through the camera does not work at all when it is 0.** `GameGetCameraPos` and
   `GameGetCameraBounds` return a perfectly steady rectangle, so the high-pass estimator from
   the next section returns zero. That route is only usable at non-zero settings, which is a
   shame, because it is the only route that needs no memory access.

The slider is also capped at 1.0, so it can only ever reduce shake — you cannot turn it up past
vanilla from the options screen. If you want the vanilla look on a 0-slider install you compute
the pixels yourself:

```lua
local VANILLA_INTENSITY = 0.7     -- what the config migration writes
local scale = (window_w / VIRT_RES_X) * 0.21629999577999115 * 0.5 * VANILLA_INTENSITY
local ox, oy = -shake_x * scale, -shake_y * scale
```

`screenshake_intensity` itself lives at offset `0xB8` of the application/config object, which
the game reaches through a virtual call rather than a fixed global, so the cheap way to read it
is `save_shared/config.xml`. It is only needed if you want to honour the player's setting rather
than substitute your own.

## What Lua can see today, with no patch

The shake is added to the camera target *before* the rectangle is published, so both camera
getters already include it — **provided `screenshake_intensity` is not 0**. At 0 the rectangle is
perfectly steady and this whole section returns nothing. See
[What the setting gates](#what-the-setting-gates-and-what-it-does-not).

**`GameGetCameraPos()`** returns the **centre** of the rectangle, as floats:

```c
cam = *(float **)(game + 0x0C);
pushnumber(cam[2] * 0.5f + *cam);      /* x + w/2 */
pushnumber(cam[3] * 0.5f + cam[1]);    /* y + h/2 */
```

**`GameGetCameraBounds()`** returns the rectangle as four **integers**, read from the grid world
at `+0x498`. It is documented as "may not be 100% pixel perfect", and it is not — the two
getters disagree:

- the integer rectangle's `x` has a hardcoded **`- 2`**, the float one does not;
- the integer rectangle **truncates**, and the float rectangle is only rounded to whole pixels at
  all when the `rendering_low_resolution` option is on (in which case rounding is
  half-away-from-zero);
- if the camera target moves more than **256 px** in one frame, the camera object also raises a
  "jumped" flag, and while it is set every non-forced camera update is ignored. Both rectangles
  are written on the frame of the jump itself, so they agree with each other; the flag only
  matters for the updates after it.

Neither of those is the shake, but both will corrupt a naive differencing scheme.

**Why you cannot just subtract the player's position.** The follow target is not the player. Each
tick the game picks a new point from a box around the player, weighted by entities tagged
`player_unit`, and re-uses it until a frame-counter gate fires. It is piecewise constant with
occasional jumps — a low-pass on `GameGetCameraPos()` will track most of it, and the shake is
white noise on top, so a high-pass *is* a usable estimator. But it is an estimator: it measures
camera jitter, not trauma, and it cannot recover `intensity` (only its product with `scale`).

A second, independent mechanism also moves the camera and shows up in the same rectangle:
`jump_cam_shake` and `jump_cam_shake_distance`, two fields on the platforming component in
`player.xml`. They offset the camera *target* and never touch `intensity`, so a shake of
`GameScreenshake(0)` from a jump is indistinguishable from a real one through the camera alone.

## Reading it exactly: the LuaJIT FFI

The interpreter is LuaJIT, and LuaJIT has an FFI that can read arbitrary memory by address — no
symbol lookup needed, which is exactly what this requires:

```lua
local ffi = require("ffi")

local function u32(a) return ffi.cast("uint32_t*", a)[0] end
local function f32(a) return ffi.cast("float*",  a)[0] end

local GAME_PTR    = 0x0122374C   -- C++ global holding Game*
local CAMERA_OFF  = 0x0C         -- game -> world/camera object
local VIRT_RES_X  = 0x01153128   -- int
local WINDOW_W    = 0x01221BCC   -- float

local SHAKE_X, SHAKE_Y, INTENSITY = 0x50, 0x54, 0x58
local K = 0.21629999577999115 * 0.5  -- the two fixed constants, folded

local function camera() return u32(GAME_PTR) + CAMERA_OFF end

--- returns intensity, offset_x, offset_y  (all in GameScreenshake units,
--- and all independent of the screenshake_intensity setting)
function GetScreenshake()
    local c = camera()
    return f32(c + INTENSITY), f32(c + SHAKE_X), f32(c + SHAKE_Y)
end

--- the same values converted to pixels. Pass the player's own setting to
--- reproduce what they see, or 0.7 for the vanilla look.
function GetScreenshakePixels(intensity_setting)
    local c = camera()
    local s = (f32(WINDOW_W) / u32(VIRT_RES_X)) * K * intensity_setting
    local I = f32(c + INTENSITY)
    return I, -f32(c + SHAKE_X) * s, -f32(c + SHAKE_Y) * s
end
```

Sanity check: call `GameScreenshake(100)` and read `intensity` on the next `OnWorldPostUpdate`.
It must be `>= 100` (it is a `max`, so a pre-existing louder shake survives), and it must fall by
a factor of exactly 0.9 per frame afterwards. If you get a constant, you are reading the wrong
struct; if you get plausible-but-scaled numbers, your `screenshake_intensity` is wrong. **Set the
slider to 0 first** — that is the cleanest test, because it removes the pixel multiplier from the
picture entirely and leaves the raw units unambiguous.

### On `require`

An ordinary mod's `lua_State` gets six libraries (`base`, `table`, `string`, `math`, `bit`, `jit`)
and **not** `package`, so `require` is `nil` and `require("ffi")` fails. The state builder
branches on a flag at `+0x4E` of its context object; the mod loader fills it from the mod's
`request_no_api_restrictions` attribute in `mod.xml`, and only when the player's mod-sandbox
setting allows it ([lua-execution.md](lua-execution.md)). A mod that sets the attribute gets
`luaL_openlibs` instead, which opens `package`, `io`, `os` and `debug` as well, so `require("ffi")`
works.

- **if `require` already works for your mod, you do not need the patch below**; go straight to the
  recipe;
- if it does not, either set `request_no_api_restrictions="1"` in your `mod.xml` (the player has to
  allow it) or use the patch below, which makes the unrestricted path taken for every mod.

```lua
print("require:", require, "package:", package, "jit:", jit and jit.version)
```

### The two-byte patch that enables it

`noita.exe` imports the *entire* LuaJIT FFI from `lua51.dll` — `luaopen_ffi`, every
`lj_cf_ffi_*` and `ffi_*` entry point — so the code is already in the process. All that keeps it
out of reach is one conditional in the state builder:

```
VA 0x007EF14E   cmp byte ptr [esi + 0x4E], 0
VA 0x007EF15B   je  0x007EF169          <-- jumps to the sandboxed six-library path; change 74 0C to 90 90
VA 0x007EF15D   push eax
VA 0x007EF15E   call [luaL_openlibs]    <-- now always reached
```

`74 0C` -> `90 90` (file offset `0x3EE55B`, two bytes: the `je` is removed, so execution always falls through into `luaL_openlibs`). Every mod state then goes through
`luaL_openlibs`, which gives `package`, `io`, `os` and `debug` — and therefore
`require("ffi")`.

**What that costs you.** This branch is the sandbox. Taking it unconditionally means
`luaL_openlibs` no longer runs the follow-up step that nils five base globals, so `load`,
`loadstring`, `gcinfo` and `collectgarbage` come back. `loadfile` is still replaced by the
game's own VFS loader afterwards, but `loadstring` does not go through the VFS — a patched
install will let a mod read arbitrary files. That is fine for your own machine and wrong for
anything you distribute. The patch is also build-specific; re-derive the address by finding the
`cmp byte ptr [reg+0x4E], 0` / `je` pair in the state builder rather than trusting the number
above.

### If you would rather not patch

A second route, and a smaller one: add one binding instead of the whole FFI. A Lua C function
that pushes three floats read from `*(Game**)0x0122374C + 0x0C + {0x50,0x54,0x58}` and splice it
into the API registrar. The registrar is a table of name/function pairs; the function itself is
about forty bytes of x86 and needs no FFI, no new library, and no sandbox change — the trade is
that it is per-build and has to be redone on every update, which is the same cost as any other
byte patch.

## Reimplementation notes

If you are rebuilding this:

- the whole thing is **three floats and two branches**. The only non-obvious parts are that the
  combine is a `max` rather than a sum, that the decay is per frame rather than per second,
  that the axes are white noise rather than a decaying oscillation, and that the unit is scaled
  by the zoom level on the way to pixels.
- if you want shake to compose, you want a sum with a cap, not a `max`.
- if you want it to behave the same at 60 and 144 Hz, multiply by `pow(0.9, dt * 60)`.
- the falloff disc should be the rectangle, not a circle of radius `w/2`, and it should use both
  axes.

## Undetermined

- Whether `screenshake_intensity` is read anywhere *else*. The evidence is that the key string
  appears only in the config parse, the format migration and the options-menu label, and that the
  only float read of `+0xB8` on the application object found anywhere is the one in the camera
  function. A scan keyed on the application pointer rather than on the offset would be needed to
  exclude a second consumer.
- What the `+0x5C` / `+0x60` pair on the same struct actually does to the view. It is a separate
  decaying scalar with a three-mode setter and it is not the screenshake; its consumer was not
  traced.
- Whether the LCG seed global is reset on world load. It is read and written from several
  systems, so it is probably shared with non-shake random consumers, which would make the
  predicted values wrong — but the *distribution* is unaffected either way.
