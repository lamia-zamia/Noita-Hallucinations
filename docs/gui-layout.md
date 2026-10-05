# The GUI layout engine

How Noita turns `GuiLayoutBeginVertical` and a pile of widget calls into coordinates. If you
have ever had a layout behave in a way you did not expect, the reason is here.

For the behaviour in general — the frame sequence, coordinates, widget identity — read
[gui-cookbook.md](gui-cookbook.md) first. This page is the detailed engine reference behind
it, including the exact field offsets of a layout frame.

Addresses are for one Steam build and will move on update.

## Layers own layout frames

The layout state is a two-level structure rooted at `state+0x1a0` (begin `+0x1a0`, end `+0x1a4`,
capacity `+0x1a8`). The outer `std::vector` holds **layers**, 16 bytes each:

| layer field | meaning |
|-------------|---------|
| `+0x00`, `+0x04`, `+0x08` | begin / end / capacity pointers of the layer's **own** `std::vector` of layout frames |
| `+0x0c` (byte) | the layer's flag; `1` for every layer the engine pushes with `GuiLayoutBeginLayer` or implicitly |

Each layout frame is **48 bytes** (12 dwords) and is pushed onto the *top layer's* inner vector
(`0x0081e480` via `FUN_008a9880`). A layer is pushed by `0x0081e7e0` via `FUN_008a9d90` and popped
by `GuiLayoutEndLayer`; a frame is popped by `GuiLayoutEnd`. So the frame stack is per layer: a
layer is a scope that starts with an empty frame stack, which is what lets a tooltip or scroll
container lay out independently of whatever layout it was opened inside.

A second `std::vector<int>` at `state+0x230`/`+0x234` is pushed in lockstep with layers
(`-1` on begin, `-4` on end) and supplies the current layer id to the draw commands.

Nothing checks that Begin and End calls match. The failure mode is underflow, not mis-stride:
`GuiLayoutEnd` with no frame in the top layer, or `GuiLayoutEndLayer` with no layer, moves an end
pointer below its begin pointer — see [gui-bugs.md](gui-bugs.md).

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
| `+0x24` | wrap limit: item count per row (mode `4` only; `0` otherwise) |
| `+0x28` | zeroed, no reader found |
| `+0x2c` | wrap item counter (mode `4` only) |

Lua can only ever produce mode `1` or `2`; `4` and `8` are used internally by the scroll
container and the wrap logic.

## Beginning a layout

```c
if (layer_begin == layer_end) FUN_0081e7e0(this, 1);    /* auto-root: push a layer */
top = top_layer.frames_end;
if (top_layer.frames_begin != top) {                    /* the layer already holds a frame */
    x += *(float *)(top - 0x24);                        /* parent's x cursor */
    y  = *(float *)(top - 0x20) + y;                    /* parent's y cursor */
}
```

So x and y are **relative to the enclosing layout's cursor**, not absolute — but only the
enclosing layout *in the same layer*. The first layout in a frame gets an auto-pushed layer; its
frame stack is empty, so its x and y are used as given.

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

With `F[]` indexing the top 48-byte frame of the top layer:

1. **Half/full-width options** (unless option `0x8000` is set) shift x left by half or a full
   widget width.
2. **Placement**, chosen by bit `0x10` of the mode word:
   - cursor-relative: `x += F[3]`, `y = F[4] + y`; if bit `0x20`, x becomes
     `(x + F[3]) - width` (right-aligned).
   - origin-relative: `x += F[1]`, `y = (F[2] + y) - height - F[6]` — this is the branch that
     produces horizontal row flow.
3. **Alignment options** `0x400`, `0x800`, `0x1000` nudge the position relative to the cursor
   (right, left, bottom). When one is set, steps 4 and 5 are skipped entirely.
4. **Cursor advance**, only when bit `0x10` of the mode is clear and the caller has not passed a
   "do not advance" flag, by `mode & 0xf`:
   - `1` horizontal: `F[3] = F[5] + x + width`
   - `2` vertical: `F[4] = F[6] + y + height` (skipped under option `0x4000`)
   - `4` wrap: bump an item counter, and on overflow reset the x cursor and drop the y cursor
   - mode `8`: no branch matches, so the cursor never advances — but the accumulated extents
     still update. This is how the scroll container measures its content.
5. **Extents**, on the same path as step 4: `F[7] = max(F[7], F[5] + width)`,
   `F[8] = max(F[8], F[6] + height)` — the accumulated content size.
6. The out-parameter is the offset **from the layout origin**, not the final position.

There is **no clipping in this function**. Clipping comes from the 24-byte clip/transform
records that the scroll container pushes.

With no frame in the top layer, or when that layer's flag byte is `0` and option `13` is not set,
the whole block is skipped and the widget keeps whatever position the
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

If the closing frame is the only one in its layer, it just pops.

There is **no underflow check** — it computes the frame count from the top layer's inner vector
and decrements that vector's end pointer unconditionally. One `GuiLayoutEnd` too many leaves the
inner `end < begin`, after which the "is a layout active" test (`begin != end`) keeps passing over
memory the layout system does not own.

## Layers

`GuiLayoutBeginLayer` pushes a 16-byte layer record: an empty frame vector (three null
pointers) and a flag byte, set to `1`. Placement (`0x0081e0d0`) only applies the layout maths when
the top layer's flag byte is non-zero or option `13` (`0x2000`) is set; the scroll container
pushes a flag-`0` layer only while it draws its scrollbar.

`GuiLayoutEndLayer` frees the layer's frame-vector storage, zeroes the three pointers, then
decrements the layer vector by `0x10` and the layer-id vector by `4`. Neither operation is
guarded.

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
percentage value you set directly. The animated value moves a quarter of the remaining distance
to the target each frame. The animation is always on for the Lua binding, which ORs bit `0x40000`
of the **high** option half (option `50`) into every call.

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

`GuiTooltip` is the only function that reads a **process-global** scratch widget record rather than
the per-`gui` record at `gui+0x38`. Its sequence is:

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
