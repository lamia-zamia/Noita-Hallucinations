# GUI bugs and sharp edges

Every item here is from the disassembly of one Steam build. Each is labelled with how solid the
evidence is:

- **verified** — read directly out of the decompilation while writing this page.
- **reported** — found by a systematic sweep of the 207-function GUI call graph; specific, with
  a named function, but not individually re-checked.

Addresses are build-specific.

If you are reimplementing the GUI rather than modding it, this page is the list of behaviours
you must either copy or fix deliberately. Several of them are load-bearing: the game itself
depends on the quirks.

## Wrong values

### `GuiImage` hit box and layout box ignore `scale_y`
**verified** — the builder (`0x00820bf0`) sizes the widget as `sprite_width * scale` by
`sprite_height * scale`: both axes use the `scale` argument. `scale_y` is stored on the image
object and used only when drawing (it falls back to `scale` when it is 0). The same box is what
gets hit-tested, written into the previous-widget record and fed to the layout cursor.

So with `scale = 1, scale_y = 3` the sprite is drawn three times as tall, but it is hovered and
clicked (and the cursor advances) as if it were one times as tall. Whenever `scale_y` differs
from `scale`, the hit box is the wrong shape for what is drawn. Rotation is not part of the box
either.

**Workaround:** pass the same value for `scale` and `scale_y`, or hit-test the area yourself.

### `GuiLayoutAddHorizontalSpacing` ignores its amount
**verified** — the most consequential one, because it is a silent wrong answer rather than a
crash.

```c
if (1 < iVar1) { lua_tonumber(param_1,2); }                        /* discarded */
iVar1 = *(int *)(state + 0x1a4);
iVar6 = *(int *)(iVar1 + -0xc);
if (*(int *)(iVar1 + -0x10) != iVar6) {
    *(float *)(iVar6 - 0x24) = *(float *)(iVar6 - 0x1c) + *(float *)(iVar6 - 0x24);
}
```

The amount is fetched from Lua and thrown away; the x cursor always advances by the layout's
`margin_x`. `GuiLayoutAddVerticalSpacing` uses its amount correctly, including the documented
margin fallback. **Workaround:** set the layout's `margin_x` to the spacing you want, or use a
zero-size widget.

### Percentage layout positions are truncated
**verified** — the wrappers compute `... * (float)(int)x * 0.01f`, so `50.7` is treated as `50`.

### `mirrorize_over_x_axis` is dead
**verified** — the Lua wrapper calls `lua_toboolean(param_1,5);` and discards the result; no
other read of that argument exists. Only `x_axis` mirrors anything.

### `GuiGetScreenDimensions` returns 0, 0 on every failure
**verified** — wrong argument type, stale handle, or a null `+0x88` all leave the two locals at
their initialised `0.0`, and both zeros are pushed. No error, no log.

```c
local_w = 0.0; local_h = 0.0;
if (lua_type(L,1) == 2) { h = validate(topointer(L,1));
  if (h != 0) { size = window_size();
                 scale = state->ui_scale;                 /* state + 0x2b0 */
                 local_w = size[0] / scale; local_h = size[1] / scale; } }
```

## Memory safety

### Layout and scroll End functions underflow unchecked
**verified** for the layout case, **reported** for the scroll case.

Layout frames live in a per-layer vector (see [gui-layout.md](gui-layout.md)). `GuiLayoutEnd`
computes the frame count from the top layer's vector, then decrements that vector's end pointer
unconditionally:

```c
iVar1 = state->layer_end;
iVar2 = *(int *)(iVar1 - 0x10);                    /* top layer: frames begin */
count = (*(int *)(iVar1 - 0xc) - iVar2) / 0x30;    /* frames end - begin */
if (1 < count) { /* fold the child's extents into the parent */ }
*(int *)(iVar1 - 0xc) -= 0x30;                     /* always */
```

One `GuiLayoutEnd` without a matching Begin leaves the layer's `end < begin`. Every later "is a
layout active" test compares `begin != end`, which is still true, so the layout code then reads
memory it does not own. `GuiLayoutEndLayer` has the same shape: it frees the top layer's frame
storage and decrements the layer vector with no emptiness test.

`GuiEndScrollContainer` decrements three end pointers with no emptiness test.

**Workaround:** keep your own Begin/End depth counter. Never rely on the engine to catch it.

### `GuiCreate` can hand back a null state pointer
**reported** — on the failure path the engine sets `+0x88` to null and the Lua wrapper still
pushes the handle. Any later call dereferences null plus an offset.

## Silent no-ops

### Id pushes past 1024 are dropped
**verified** — `GuiIdPush`'s bounds test has no `else` branch. Once the stack holds 1024
entries, further pushes are discarded and every subsequent id resolution in that frame returns
the wrong value, with no diagnostic.

### Nothing unwinds an unbalanced Begin or Push
**verified** — `GuiStartFrame` resets the option, colour, z and previous-widget fields. It does
**not** touch the layout stack, the id stack, the layer stack or the scroll stacks. `GuiDestroy`
does not unwind them either. A `GuiIdPush` you forget to pop, or a layout you forget to end,
persists into the next frame and the next frame after that.

### A stale handle does nothing, silently
**verified** — 40 of 41 `Gui*` functions run the handle through the validator and, on `0`, skip
the entire body. No log line, no return value, no error. This is the single most common reason
"my GUI stopped working".

### `GuiStartFrame` wipes shared state even with a bad handle
**verified** — the reset of the shared scratch widget record happens outside the handle check, so
a call with an invalid handle skips the real frame reset but still clears the record
`GuiTooltip` reads, wiping the tooltip state of whatever mod was drawing.

### Passing something that is not a `gui` handle
**verified pattern** across the wrappers: they test `lua_type(L,1) == 2` and then
`if (handle != 0)`, with no `else`. A string in the first position does nothing, silently.

### `GuiOptionsAddForNextWidget` drops values of 64 or more
**verified** — the option index is split across two 32-bit halves:

```c
uVar8 = 1 << (v & 0x1f);
uVar9 = 0;
if (0x1f < v) { uVar9 = uVar8; }     /* v >= 32: move to the high half */
uVar8 = uVar8 ^ uVar9;              /* v >= 32: clear the low half  */
if (0x3f < v) { uVar9 = uVar8; }    /* v >= 64: high half becomes 0 */
pending_lo |= uVar8;  pending_hi |= uVar9;
```

`0 ≤ v < 32` sets a low-half bit, `32 ≤ v < 64` sets the corresponding high-half bit, and
**`v ≥ 64` sets neither** — no error, no log. A negative value reaches the `& 0x1f` path and
sets an arbitrary low bit. The game defines far fewer than 32 options, so this only bites on a
mistake.

Note this is *not* modulo-32 aliasing: `GuiOptionsAdd(gui, 32)` is a different option from
`GuiOptionsAdd(gui, 0)`.

## Uninitialised data

### The committed widget record carries stack garbage
**verified** — the record constructor writes through byte 75; the commit copies 80:

```c
*(undefined4 *)(gui + 0x84) = param_2[0x13];   /* never initialised */
```

Bytes 5–7 and 18–23 are also left untouched by the constructor. Thirteen bytes of the
"previous widget" record — which `GuiGetPreviousWidgetInfo` reads — are stack garbage. In
practice the fields modders read are the ones that do get written, which is why this has never
been noticed.

### Multi-line text ORs in an unassigned local
**reported** — `GuiText` declares a byte, never assigns it, and ORs it into the record's
mouse-over flag in the multi-line path. So hover and focus reporting for text containing a line
break includes a garbage bit. The single-line path is unaffected.

## Leaks and unbounded growth

### The widget-state map is never pruned
**verified** — the map at `state+0x64` is a red-black tree keyed by the 64-bit widget id. The
only tree operation anywhere in the GUI code is get-or-insert, whose "key already present" path
destroys the incoming payload and returns the existing node; **no erase path exists**.

A node is at least 0x200 bytes. A mod that varies its widget id per frame adds one node per id,
permanently, for the lifetime of that `Gui` object.

**Workaround:** keep your id set small and stable. A common mod pattern of
`GuiIdPush(entity_id)` or a frame counter as the id grows this without bound.

### Layout and layer stacks are unbounded
**verified** — there is no depth cap on the layout stack. A loop that begins layouts without
ending them grows the vector until memory runs out.

## Behaviour you may be relying on by accident

### Widget ids depend on the whole id stack
**verified** — a widget's effective id is its own id **verbatim** when the id stack is empty,
and an FNV-1a-64 hash chain seeded by the stack top when it is not. The same literal id
therefore identifies different widgets depending on whether anything is pushed. Anything that
"works" only inside a particular `GuiIdPush` scope will break when the scope changes.

### Two mods with the same id share state, not just pixels
**verified** — hover, focus, click latch, animation phase and scroll offset all live in the
id-keyed map. Sharing an id at the same stack depth means writing to each other's widget state.

### Only one widget can be hovered per frame
**verified** — the first widget to claim the mouse sets a flag that suppresses the rest. With
two mods drawing overlapping widgets, the second one cannot be hovered at all, and which one
wins is draw order.

**The claim is per-`gui`, not process-global.** The flag is the byte at `*(gui+0x88) + 0x1fd` —
reached through a runtime pointer at all three write sites (`008245d0.c:189`, `00823580.c:288`,
`00825cb0.c:251`) and all eleven read expressions (across nine functions), never as a fixed global. It is cleared only by that
object's own `NewFrame` (`008187d0.c:1175`, the sole clear site). So two mods with their own
`GuiCreate` objects never compete; sharing only happens when two mods are handed the *same*
handle, or when a mod draws through the game's own `Gui`. See
[ui-modding-2.md](ui-modding-2.md) for what this means for drawing a menu over a vanilla one.

No `GUI_OPTION` bit bypasses it: the claim test is a top-level conjunct of the hit test in every
widget builder, never in a disjunction with an option test. The one real bypass is not an option —
`GuiButton` and `GuiTextInput` skip the claim test on the frame their widget-state entry is
created, so a widget is briefly unhoverable the moment its entry first appears.

### Clicking requires the pointer to still be inside
**reported** — a click is the mouse-down flag and a current hover test. There is no
press-then-drag-out case, so a button cannot be clicked by pressing it and sliding off.

### The commit resets pending state, which is why it is called after every widget
**verified** — `FUN_007d97d0` writes the record to `gui+0x38` and then resets the pending
option, colour and z fields, setting the option set to `1` (option bit 0, the default
alignment) rather than `0`. Layout and scope functions do not commit, so a frame that only
pushes layout or opens a scope leaves the previous frame's "previous widget" record in place.
`GuiGetPreviousWidgetInfo` can then report a widget that was never drawn.
