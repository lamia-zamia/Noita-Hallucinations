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
  no log line. A negative value is compared as an unsigned number, so it is discarded the same way.
- Note this is **not** modulo-32 aliasing: `GuiOptionsAdd(gui, 32)` is a *different* option from
  `GuiOptionsAdd(gui, 0)`, it just happens to land in the other half.

`GUI_OPTION` defines values 1-29 in the low half and 47-51, 62 and 63 in the high half, so both
halves are in use and the "one value, one half" rule matters in practice.

**What each option value actually does is in
[gui-options.md](gui-options.md)** — the effects are in the executable even though the names
are not.

## There are two option fields, and only one is cleared

The option set is 64 bits wide and lives in **four** 32-bit fields on the `gui` object — two for
the frame-wide set and two for the pending "next widget" set:

| function | operation | low half | high half |
|----------|-----------|-----------|-----------|
| `GuiOptionsAdd` | `field \|= 1 << (v & 31)` | `+0x08` | `+0x0c` |
| `GuiOptionsRemove` | `field &= ~(1 << (v & 31))` | `+0x08` | `+0x0c` |
| `GuiOptionsAddForNextWidget` | `field \|= 1 << (v & 31)` | `+0x10` | `+0x14` |
| `GuiOptionsClear` | reset to the default set: low half `= 1`, high half `= 0` | `+0x08` | `+0x0c` |

A widget receives the two halves as two separate arguments, and the Lua wrapper merges the
pending and frame-wide sets before passing them:

```
low  = (gui + 0x10) | (gui + 0x08)
high = (gui + 0x14) | (gui + 0x0c)
```

Two consequences:

- **A single `GUI_OPTION` value can never set one bit in each half.** You combine options by
  calling the function once per value.
- **`GuiOptionsClear` resets the frame-wide pair** (`+0x08` to `1`, `+0x0c` to `0`) - the same
  default the pending pair is reset to - rather than to zero. Nothing Lua-callable writes the
  pending pair except `GuiOptionsAddForNextWidget` — it is consumed and reset by the per-widget
  commit, which resets it to `1` (option bit 0, the default alignment) rather than `0`.

The practical consequence of the pending half: an abandoned "next widget" option can still
apply to a later widget, because nothing else clears it.

## Getting the actual values

The tables are plain Lua in the data tree - read them from `data/scripts/lib/utilities.lua`
(`GUI_OPTION`, `GUI_RECT_ANIMATION_PLAYBACK`) and `data/scripts/debug/keycodes.lua`. The game's
documentation generator does not emit them: it only formats the function signature strings (see
[api-registration.md](api-registration.md#the-game-ships-an-api-documentation-generator)).

Cross-check those files against the arities in [api-reference.md](api-reference.md) to get an
authoritative, current answer for both halves at once.

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
