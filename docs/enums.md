# Enum and constant tables

## They are not in the executable

Several tables the API refers to by name are defined in game data, not in `noita.exe`:

| table | where it is defined | used by |
|-------|---------------------|---------|
| `GUI_OPTION` | `data/scripts/lib/utilities.lua` | `GuiOptionsAdd`, `GuiOptionsAddForNextWidget`, `GuiOptionsRemove` |
| `GUI_RECT_ANIMATION_PLAYBACK` | `data/scripts/lib/utilities.lua` | `GuiImage` |
| key codes | `data/scripts/debug/keycodes.lua` | every `Input*` function |
| cell materials | generated from the material XMLs | `CellFactory_*`, anything taking `material_type` |
| damage types, `DamageModelComponent` bit fields | component definitions | `EntityInflictDamage` |

Reading names and integers out of a string pool does not work for any of them, because the
game loads these tables from Lua at startup. What the executable *does* show is how the
integers are consumed, which is enough to constrain them — see below.

## GUI_OPTION values are bit positions, not bit masks

`GuiOptionsAdd` and friends reduce their argument before storing it. `GuiOptionsAddForNextWidget`
shows the whole scheme:

```c
uVar8 = 1 << (v & 0x1f);
uVar9 = 0;
if (0x1f < v) { uVar9 = uVar8; }     /* v >= 32: move the bit to the high half */
uVar8 = uVar8 ^ uVar9;              /* v >= 32: clear the low half          */
if (0x3f < v) { uVar9 = uVar8; }    /* v >= 64: the high half becomes 0     */
pending_lo |= uVar8;   /* gui + 0x10 */
pending_hi |= uVar9;   /* gui + 0x14 */
```

What follows:

- **The values are bit indices.** The engine does the shifting, so there is no way to pass a
  pre-combined mask. You combine options by calling the function once per option — which is what
  "values from consecutive calls will be combined" means.
- **The option set is 64 bits wide, stored as two 32-bit halves.** `gui+0x10` holds options
  0–31 and `gui+0x14` holds options 32–63. Widget functions receive the two halves as separate
  parameters and test them independently, which is why a single `GUI_OPTION` value can never
  set one bit in each half.
- **A value of 64 or more is silently discarded** — both halves end up zero, with no error and
  no log line. A negative value falls into the `& 0x1f` path and sets an arbitrary low bit.
- Note this is **not** modulo-32 aliasing: `GuiOptionsAdd(gui, 32)` is a *different* option from
  `GuiOptionsAdd(gui, 0)`, it just happens to land in the other half.

The game defines far fewer than 32 options, so in practice only the "no pre-combined mask" rule
and the dropped-out-of-range case matter.

## There are two option fields, and only one is cleared

`GuiOptionsAdd` and `GuiOptionsRemove` operate on the frame-wide field at `gui+0x0c`;
`GuiOptionsAddForNextWidget` operates on the pending field at `gui+0x10`/`+0x14`.
`GuiOptionsClear` zeroes `+0x0c` and **not** the pending field. No Lua-callable function writes
the pending field other than `GuiOptionsAddForNextWidget` — it is consumed and cleared by the
per-widget commit, which also resets it to `1` (option bit 0, the default alignment) rather
than `0`.

The practical consequence: an abandoned "next widget" option can still apply to a later widget,
because nothing else clears it.

| function | operation | field |
|----------|-----------|-------|
| `GuiOptionsAdd` | `field \|= 1 << (v & 31)` | `+0x0c` frame-wide |
| `GuiOptionsRemove` | `field &= ~(1 << (v & 31))` | `+0x0c` frame-wide |
| `GuiOptionsAddForNextWidget` | `field \|= 1 << (v & 31)` | `+0x10` + `+0x14` pending |
| `GuiOptionsClear` | `field = 0` | `+0x0c` only |

## Getting the actual values

The game contains a generator that walks the live Lua state and writes these tables out. Run
the game once with `out_json` set on the `GlobalLuaManager` entry point and it produces
`tools_modding/lua_api_documentation.json`, which includes the enum tables that static analysis
of the executable cannot reach. See [api-registration.md](api-registration.md#the-game-ships-an-api-documentation-generator).

Cross-check that dump against the 375 names and enforced arities in
[api-reference.md](api-reference.md): it is the only way to get an authoritative, current
answer for both halves at once.

## Cell materials have no numeric literals in the API

No function takes a material by name; they all take a numeric `material_type`. The only
conversion is `CellFactory_GetType`, and the only way to enumerate the set is the
`CellFactory_GetAll*` family, which takes no arguments:

```
CellFactory_GetType        name -> id            CellFactory_GetAllSolids
CellFactory_GetName        id -> name            CellFactory_GetAllLiquids
CellFactory_GetTags        id -> tags            CellFactory_GetAllSands
CellFactory_GetUIName      id -> display name     CellFactory_GetAllFires
CellFactory_HasTag                            CellFactory_GetAllGases
```

So a mod that hardcodes a material id is relying on a number that depends on load order in the
game's material files, not on a constant in the binary. Resolve names at runtime.
