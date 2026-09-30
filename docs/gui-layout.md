# The GUI layout engine

How Noita turns `GuiLayoutBeginVertical` and a pile of widget calls into coordinates. If you
have ever had a layout behave in a way you did not expect, the reason is here.

For the behaviour in general — the frame sequence, coordinates, widget identity — read
[gui-cookbook.md](gui-cookbook.md) first. This page is the detailed engine reference behind
it, including the exact field offsets of a layout frame.

Addresses are for one Steam build and will move on update.

## One vector, two record sizes

Layout frames and "layers" are pushed onto the **same** `std::vector` at `state+0x1a0`
(begin `+0x1a0`, end `+0x1a4`, capacity `+0x1a8`), and the two record types have different
sizes:

| what | pushed by | record size | popped by |
|------|-----------|-------------|-----------|
| layout frame | `0x0081e480` via `FUN_008a9880` | **48 bytes** (12 dwords) | `GuiLayoutEnd` |
| layer | `0x0081e7e0` via `FUN_008a9d90` | **16 bytes** (4 dwords) | `GuiLayoutEndLayer` |

The two push helpers are otherwise identical — same vector layout, same realloc-on-full logic —
and differ only in the stride they advance the end pointer by (`+0x30` versus `+0x10`) and how
many dwords they copy in.

A second `std::vector<int>` at `state+0x230`/`+0x234` is pushed in lockstep with layers
(`-1` on begin, `-4` on end) and supplies the current layer id to the draw commands.

Nothing enforces that the two kinds are interleaved correctly. The engine's own code always uses
the order *layer, layout, …, layout end, layer end*, and every consumer computes the current
record with a hardcoded stride. If you begin a layout and then begin a layer, or end them out
of order, the readers will interpret the wrong bytes. This is a sharp edge, not a supported
mode — see [gui-bugs.md](gui-bugs.md).

## The layout frame

`GuiLayoutBeginHorizontal` and `GuiLayoutBeginVertical` call the **same** engine function
(`0x0081e480`) with a different first argument: `1` for horizontal, `2` for vertical.

| offset | meaning |
|--------|---------|
| `+0x00` | packed word: low nibble is the mode (`1` horizontal, `2` vertical, `4` wrap, `8` no-cursor-advance), bits `0x10` and `0x20` are extra flags (origin-relative placement; right/bottom align) |
| `+0x04` | frame origin x |
| `+0x08` | frame origin y |
| `+0x0c` | **x cursor** |
| `+0x10` | **y cursor** |
| `+0x14` | margin_x |
| `+0x18` | margin_y |
| `+0x1c` | accumulated content width |
| `+0x20` | accumulated content height |
| `+0x24`–`+0x2f` | zeroed, no reader found |

Lua can only ever produce mode `1` or `2`; `4` and `8` are used internally by the scroll
container and the wrap logic.

## Beginning a layout

```c
if (layout_begin == layout_end) FUN_0081e7e0(this, 1);   /* auto-root: push a layer */
begin_of_top = *(int *)(end - 0x0c);
if (*(int *)(end - 0x10) != begin_of_top) {             /* the top record is a layout frame */
    x += *(float *)(begin_of_top - 0x24);
    y  = *(float *)(begin_of_top - 0x20) + y;
}
```

So x and y are **relative to the enclosing layout's cursor**, not absolute. The first layout in
a frame gets an auto-pushed layer as its base — and that path reads one dword past the 16-byte
record it just pushed, which is a genuine uninitialised read (see
[gui-bugs.md](gui-bugs.md)).

Margins are **stored, not applied**. They are never added to x or y at begin time. They are
read later, as per-widget padding and as the spacing fallback.

Default margins differ by direction: horizontal defaults to `2.0` for both, vertical defaults
to `0.0` for both.

### `position_in_ui_scale`

This flag is not passed to the engine at all. The Lua wrapper consumes it and only changes how
x and y are interpreted:

- `false` (the default): x and y are **percentages of the screen**. The wrapper computes
  `x = screen_width / ui_scale * x * 0.01`.
- `true`: x and y are used verbatim in UI coordinates.

Two sharp edges in the percentage path:

- The percentage is **truncated to a whole number** before scaling — there is a narrowing
  `(int)` cast — so `50.7` becomes `50`.
- The same `ui_scale` division used here is what makes `GuiGetScreenDimensions` return
  `window_size / ui_scale`, so percentages and the reported screen size agree.

## Placing a widget

Every widget goes through `0x0081e0d0`, which turns the widget's own coordinates into final
ones using the current frame's cursor.

With `F[]` indexing the top 48-byte record:

1. **Half/full-width options** (unless option `0x8000` is set) shift x left by half or a full
   widget width.
2. **Placement**, chosen by bit `0x10` of the mode word:
   - cursor-relative: `x += F[3]`, `y = F[4] + y`; if bit `0x20`, x becomes
     `(x + F[3]) - width` (right-aligned).
   - origin-relative: `x += F[1]`, `y = (F[2] + y) - height - F[6]` — this is the branch that
     produces horizontal row flow.
3. **Cursor advance**, skipped when the caller passes a "do not advance" flag, by `mode & 0xf`:
   - `1` horizontal: `F[3] = F[5] + x + width`
   - `2` vertical: `F[4] = F[6] + y + height`
   - `4` wrap: bump an item counter, and on overflow reset the x cursor and drop the y cursor
   - mode `8`: no branch matches, so the cursor never advances — but the accumulated extents
     still update. This is how the scroll container measures its content.
4. **Always**: `F[7] = max(F[7], F[5] + width)`, `F[8] = max(F[8], F[6] + height)` — the
   accumulated content size.
5. Options `0x400`, `0x800`, `0x1000` nudge the position relative to the cursor for explicit
   alignment.
6. The out-parameter is the offset **from the layout origin**, not the final position.

There is **no clipping in this function**. Clipping comes from the 24-byte clip/transform
records that the scroll container pushes.

With no active layout, the whole block is skipped and the widget keeps whatever position the
caller computed — so widgets drawn outside any layout are positioned by their own arguments
alone.

### Content measurement

If the bounding-box accumulator stack is non-empty, the same function unions
`(x, y, x+w, y+h)` into the top accumulator. This is how `GuiBeginAutoBox` and
`GuiBeginScrollContainer` learn how big their contents are: they do not measure anything
themselves, they just push an accumulator and read it at End time.

## Ending a layout

`GuiLayoutEnd` pops the 48-byte record and **folds the child's extents into the parent's
cursors**, with the direction deciding how:

- vertical: the parent's y cursor advances by the child's height,
- horizontal: the parent's x cursor advances by the child's width,
- with the align flag, by a signed amount.

If the stack holds only the implicit base record, it just pops.

There is **no underflow check** — it reads the record below the end pointer before testing
whether one exists, and it decrements the end pointer unconditionally. One `GuiLayoutEnd` too
many leaves `end < begin`, after which the "is a layout active" test keeps passing over memory
the layout system does not own.

## Layers

`GuiLayoutBeginLayer` pushes a 16-byte record: a pointer, two zero dwords, and a byte that is
the argument. That byte is the clip flag, and it is what distinguishes a layer record from a
layout frame when some code reads "the byte at end-4".

`GuiLayoutEndLayer` frees the record's pointer member, zeroes it, then decrements the layout
vector by `0x10` and the layer vector by `4`. Neither operation is guarded.

The layer's practical effect is on the draw pipeline: the current layer id comes from
`state+0x230`'s top, and the draw-command builder records it. So a layer is a z/draw-group
boundary as well as a layout scope.

## Spacing

Both spacing functions are thin shims: arity guard, handle validation, then a direct poke at
the top record. No widget, no id hashing, no layout pass. Both are silent no-ops when no layout
is active.

Vertical spacing works as documented — with the fallback to the layout's `margin_y` when the
amount is omitted:

```c
amount = (argc < 2) ? record.margin_y : lua_tonumber(L, 2);
record.y_cursor += amount;
```

**Horizontal spacing ignores its `amount` argument.** It reads the number and throws it away,
then always advances the x cursor by the layout's `margin_x`:

```c
if (argc > 1) { lua_tonumber(L, 2); }          /* result discarded */
record.x_cursor = record.margin_x + record.x_cursor;
```

So `GuiLayoutAddHorizontalSpacing(gui, 100)` adds `margin_x`, not 100. The vertical twin does
not have this problem, so it is an asymmetry rather than a convention. See
[gui-bugs.md](gui-bugs.md).

## Scroll containers

`GuiBeginScrollContainer` pushes two 40-byte records: a container frame at `state+0x1b8` and a
child bounding-box accumulator at `state+0x1ac`. It then pushes a layer and a layout frame with
mode `8` at the scrolled content origin, so that every child is positioned relative to the
scrolled origin and measured into the accumulator without advancing the cursor.

The scroll offset is kept in three places:

| where | meaning |
|-------|---------|
| persistent entry `+0x78` | the target offset |
| persistent entry `+0x7c` | the animated value, lerped toward the target |
| the container frame `+0x10` | the value published for this frame |

It is clamped to `[0, 1.0]` — in GUI units relative to the content extent, not a pixel or
percentage value you set directly. The animation is always on for the Lua binding, which always
passes the option that enables it.

`GuiEndScrollContainer` pops the layer, the layout frame, the layer id and both 40-byte records,
and writes the measured content size back into the persistent entry. Nesting is not checked: no
depth counter, no id match against the frame being closed, no emptiness test.

## Auto box

`GuiBeginAutoBox` is a stub — it pushes an empty bounding-box accumulator and nothing else. No
rectangle, no id, no widget.

`GuiEndAutoBoxNinePiece` is where all the work happens. It reads the accumulated child bounding
box and computes:

```
x = min_x - margin
y = min_y - margin
w = max(max_x - min_x, size_min_x)
h = max(max_y - min_y, size_min_y)
```

and then adds `2 * margin` to the extents, hit-tests the resulting box, picks the highlight
sprite when hovered, and hands the whole 80-byte widget record to the nine-piece image widget.

Two documented arguments behave unexpectedly:

- **`mirrorize_over_x_axis` does nothing.** Its value is read and discarded. Only `x_axis`
  mirrors, and mirroring is disabled when `x_axis` equals a specific sentinel constant rather
  than when the boolean is false.
- Mirroring itself is `w = max(|x + w - x_axis|, |x - x_axis|) * 2; x = x_axis - w;` — a
  reflection about `x_axis`, and it is unconditional apart from the sentinel check.

## Tooltip

`GuiTooltip` is the only function that reads the process-global previous-widget record rather
than anything on the `gui` object. Its sequence is:

1. If the previous widget was not hovered, or its record has already been consumed, draw nothing
   and return.
2. Mark the record consumed.
3. Push a layer and a vertical layout at the mouse position.
4. `GuiBeginAutoBox`, draw the title, draw the description if non-empty,
   `GuiEndAutoBoxNinePiece`.
5. Pop the layout and the layer.

Because it reads global state, a tooltip describes whatever widget committed last — which, in
a frame where two mods both draw, is whichever of them ran most recently. It also runs in the
same Lua call as the widget it describes, not deferred to the end of the frame.
