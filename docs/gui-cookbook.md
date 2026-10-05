# Reimplementing the GUI: behaviour and formulas

This page is for anyone rebuilding Noita's `Gui*` API — for a GUI-rewrite mod, or for a
standalone GUI host that runs GUI mods without the game. It is written as **behaviour and
arithmetic**: what the API does, in what order, and with what formula. No struct offsets.

The two pages that go with it:

- [gui-defaults.md](gui-defaults.md) — the hardcoded values: which sprites, font, sounds and
  constants, and every default argument. You cannot reimplement the GUI without these.
- [gui-options.md](gui-options.md) — what each `GUI_OPTION` does, and which are no-ops.

The data structures behind the behaviour are in [gui-internals.md](gui-internals.md) and the
per-container detail is in [gui-layout.md](gui-layout.md).

Addresses are for one Steam build and will move on update; the behaviour is the stable part.

## The shape of it

A GUI object is a handle. The API is **immediate mode**: there is no retained widget tree. Each
call computes everything from scratch, except for a per-widget state cache keyed by an id.

The whole system is four things:

1. **A cursor.** Where the next widget goes. A layout pushes and pops cursor frames.
2. **An id stack.** Widget identity is derived from it.
3. **A per-widget state cache.** Hover, click, animation and scroll live here, keyed by id.
4. **A pending-options/colour/z slot.** Set for the next widget, consumed by it.

Everything else is detail.

## The frame

```
GuiStartFrame(gui)
  reset: frame options -> 1          (bit 0 on: default alignment)
         pending options -> 1        (same)
         next-widget colour -> opaque white
         frame z, pending z -> 0
         previous-widget record -> default (all zero, scale word 1.0)
  NOT reset: layout stack, id stack, layer stack, scroll stacks, widget-state cache
  read the mouse

  ... your widgets ...

  render: walk the draw-command list, then reset the list
```

**The asymmetry is the thing to get right.** A GUI implementation that resets the layout stack
each frame will differ from the game in one very visible way: an unbalanced `GuiLayoutBegin` or
`GuiIdPush` persists in the game, so a mod that forgets an `End` keeps drawing wrong forever
rather than recovering next frame. If you are reimplementing for compatibility, reproduce the
non-reset. If you are reimplementing for sanity, reset them, and you will have fixed a bug your
mods were quietly relying on.

There is no end-frame call. `GuiDestroy` is the only teardown, and it does not unwind the
stacks either.

## Coordinates

### Screen space and UI scale

The window has a pixel size. The GUI works in **UI units**, related to pixels by a scale
factor:

```
ui_scale = 1.0 by default
screen_size_in_ui_units = window_pixel_size / ui_scale
```

`GuiGetScreenDimensions` returns `window_pixel_size / ui_scale` — UI units, not pixels. Two
things follow:

- If your host has no scale, `ui_scale` is 1 and the two coincide.
- `InputGetMousePosOnScreen` returns **raw pixels**, not UI units. Do not use it to place
  widgets. The GUI tracks its own mouse, already divided by the scale.

### Percentages

Every widget takes x and y, and what they mean depends on one boolean:

```
position_in_ui_scale == false  (default):
    x = screen_w / ui_scale * x * 0.01
    y = screen_h / ui_scale * y * 0.01

position_in_ui_scale == true:
    x, y used verbatim in UI units
```

So by default x and y are **percentages of the screen**, not pixels. `x = 50` is the middle of
the screen, not 50 pixels. This is the single most common source of "my GUI is in the wrong
place".

Two sharp edges in the percentage path:

- The percentage is **truncated to an integer** before scaling, so `50.7` is treated as `50`.
- The flag is consumed by the Lua wrapper and never reaches the engine — it only changes how
  the two numbers are interpreted.

### Inside a layout

With an active layout, positions become **relative to the enclosing layout's cursor**, and the
layout's own origin is added. The first layout in a frame gets an implicit base.

The cursor algorithm, given a widget of size `(w, h)` at requested `(x, y)` in a frame with
origin `(ox, oy)` and cursor `(cx, cy)`:

```
1. Alignment options shift x or y first        (see gui-options.md)
2. Place:
     cursor-relative:   x = cx + x ,  y = oy + y
     origin-relative:   x = ox + x ,  y = oy + y - h - margin_y
3. Advance the cursor by direction (cursor-relative placement only, and not under the
   right/left/bottom alignment options):
     horizontal: cx = margin_x + x + w
     vertical:   cy = margin_y + y + h
     wrap:       count items; on overflow, reset cx to ox and drop cy
     none:       no branch matches, so the cursor does not advance
4. On the same path, grow the frame's content extent:
     content_w = max(content_w, margin_x + w)
     content_h = max(content_h, margin_y + h)
```

Two consequences that matter when reimplementing:

- **Extents are updated even when the cursor does not advance.** That is how the scroll
  container measures its content: it pushes a layout in "no advance" mode, draws children, and
  accumulates their bounding box for the end call.
- **With no active layout, step 2–4 are skipped entirely** and the widget keeps whatever
  position its own arguments produced. This is the "widgets outside a layout are positioned by
  their arguments alone" rule, and it is why option `15` works as an escape hatch.

### Margins

Margins are **stored, not applied**. They are not added to the cursor at layout-begin time.
They are used later, as per-widget padding, as the spacing fallback, and in the alignment
shifts. Defaults differ by direction: horizontal layouts default to `2.0` for both margins,
vertical to `0.0` for both.

## Layouts

```
GuiLayoutBeginHorizontal(gui)  /  GuiLayoutBeginVertical(gui)
   push a frame: mode, origin, cursor = origin, margins, content extents = 0
   if the stack was empty, first push an implicit base layer
   ... widgets ...
GuiLayoutEnd(gui)
   pop the frame
   fold the child's extents into the parent cursor, direction deciding which axis
```

The cursor advance for a child is what makes a row or a column: a horizontal layout advances x
by the child's **width**, a vertical layout advances y by the child's **height**.

### Modes

| mode | meaning |
|------|---------|
| `1` | horizontal — advance x |
| `2` | vertical — advance y |
| `4` | wrap — count items per row, on overflow reset x and drop y |
| `8` | no cursor advance, but still accumulate extents |

Lua can only produce modes `1` and `2`. The scroll container uses `8`, and the wrap logic uses
`4`. If you are reimplementing, implement all four: mods depend on the scroll container's
measurement behaviour, and anything that reaches for the others does so through containers
rather than directly.

### Spacing

- **Vertical spacing works as documented**: `cy += amount`, falling back to the layout's
  `margin_y` when the amount is omitted.
- **Horizontal spacing ignores its amount.** It reads the number and throws it away, then
  always advances x by the layout's `margin_x`. `GuiLayoutAddHorizontalSpacing(gui, 100)` adds
  `margin_x`, not 100. This is a real asymmetry in the game, not a convention — reproduce it if
  you are matching mod output.

### Layers

`GuiLayoutBeginLayer` / `GuiLayoutEndLayer` push a record that carries a **layer id**, which
becomes a draw-group / z-boundary, and that owns its own stack of layout frames. A layout begun
inside a layer is positioned relative to the enclosing layout *in that layer*, not relative to a
layout in an outer layer. A freshly begun layer has no frames, so widgets drawn in it before any
layout is begun are not laid out — which is what makes a layer the way to draw non-layouted
widgets inside a layout.

The scroll container and the tooltip both do *layer → layout → … → layout end → layer end*. If
you reimplement this, a stack of layers each holding a stack of frames reproduces it directly.

## Widget identity

The id system is the root of all cross-mod interference, and the rule is short enough to state
exactly:

```
effective_id(widget) =
    id                                  if the id stack is empty
    FNV1a64(bytes_of(id), top_of_stack)  otherwise
```

FNV-1a-64, standard constants: offset basis `0xcbf29ce484222325`, prime `0x100000001b3`, one
byte at a time, `hash = (byte XOR hash) * prime`.

```
GuiIdPush(gui, id)
    if depth < 1024:
        seed = offset_basis   if the stack is empty
               top_of_stack   otherwise
        push(FNV1a64_over_8_bytes(id, seed))
    # no else: past 1024 the push is silently dropped
```

Three consequences:

- **With an empty id stack a widget's id is the raw integer.** Push anything and the same
  literal becomes a hash chain. So the same `id` can be two different widgets depending on
  whether a mod happened to be inside a `GuiIdPush`.
- **The hash is cumulative over the whole stack.** A widget's identity depends on everything
  pushed, not just its own id.
- **Past 1024 pushes the push is dropped silently**, and every later id resolution in that
  frame is wrong with no diagnostic.

State is keyed by this id: hover, focus, click latch, animation phase, scroll offset. **Two
mods passing the same id at the same stack depth are writing each other's widget state**, not
just overdrawing each other.

If you are reimplementing for composability — which is the actual goal of a GUI-rewrite mod —
this is the thing to change. Namespacing ids per mod at the boundary makes two mods that
collide today stop colliding, at the cost of not matching the original byte-for-byte.

## The per-widget bracket

Every drawing function is the same three steps:

```
1. begin:  build a widget record on the stack (zeros, with a default scale of 1.0)
2. fill:   the widget's own function computes rect, colours, and the state lookup
3. commit: copy the record to "previous widget", and reset the pending
           option / colour / z
```

The commit is what resets the pending slot, and it resets the option set to **1**, not 0 —
bit 0 is the default alignment. So a widget that does not set an option explicitly still gets
default alignment.

**Consequence:** layout and scope functions do not commit, so a frame that only moves things
leaves the previous frame's "previous widget" record in place. `GuiGetPreviousWidgetInfo` can
then describe a widget that was never drawn. In a reimplementation, commit only on widgets and
accept the same behaviour.

A widget receives its option set as two values, merged by the Lua wrapper:

```
low  = (gui + 0x10) | (gui + 0x08)      # pending low  | frame-wide low
high = (gui + 0x14) | (gui + 0x0c)      # pending high | frame-wide high
```

Most of the 64-bit space does nothing — see [gui-options.md](gui-options.md) for which bits
actually work. If you are reimplementing, implement the tested ones and treat the rest as
reserved, rather than inventing behaviour for all 64.

## The previous-widget record

`GuiGetPreviousWidgetInfo` returns 11 values from a record **on the `gui` object, at `gui+0x38`**.
Two `Gui` objects do not interfere. If the `gui` argument fails handle validation the function
falls back to a static zeroed record, so it returns zeroes rather than another object's data.

Exactly ten functions write it:

```
GuiBeginScrollContainer   GuiButton              GuiEndAutoBoxNinePiece
GuiImage                  GuiImageButton         GuiImageNinePiece
GuiSlider                 GuiText                GuiTextCentered
GuiTextInput
```

If you are reimplementing, make it per-widget rather than one "last widget" slot per `gui`, and
`GuiTooltip` becomes safe. That is a deliberate divergence from the original, not a bug fix.

## Containers

### Scroll container

```
GuiBeginScrollContainer(gui, id, ...)
   push a container frame
   push a child bounding-box accumulator
   push a layer, then a layout frame in mode 8 at the scrolled origin
   ... children, measured but not flowed ...
GuiEndScrollContainer(gui)
   pop all of it
   write the measured content size back to the persistent entry
```

The scroll offset is kept in three places: a target, an animated value lerped toward it, and
the value published for the frame. It is clamped to `[0, 1.0]` **in GUI units relative to the
content extent**, not as a pixel or percentage value you set. Animation is always on — the
container always passes the animation option, so smooth scrolling cannot be switched off.

**Nesting is not checked**: no depth counter, no id match against the frame being closed, no
emptiness test. Track depth yourself.

### Auto box

`GuiBeginAutoBox` is effectively a stub: it pushes an empty bounding-box accumulator and
nothing else. No rect, no id, no widget. All the work is in the End:

```
GuiEndAutoBoxNinePiece(gui, ...)
   read the accumulated child bounding box
   x = min_x - margin
   y = min_y - margin
   w = max(max_x - min_x, size_min_x)
   h = max(max_y - min_y, size_min_y)
   then add 2*margin to the extents
   hit-test the result, pick the highlight sprite if hovered, draw the nine-piece image
```

Two argument behaviours worth copying or fixing deliberately:

- **`mirrorize_over_x_axis` does nothing.** Its value is read and discarded; only `x_axis`
  mirrors anything. Mirroring is `w = max(|x + w - x_axis|, |x - x_axis|) * 2; x = x_axis - w`
  — a reflection about `x_axis`.
- Mirroring is disabled when `x_axis` equals a specific sentinel constant, not when a boolean
  is false. "No mirror axis" is a magic number.

### Tooltip

`GuiTooltip` reads the **process-global** previous-widget record, not the `gui` object:

```
1. if the previous widget was not hovered, or its record was already consumed: draw nothing
2. mark the record consumed
3. push a layer and a vertical layout at the mouse
4. begin auto box, draw title, draw description if non-empty, end auto box
5. pop layout, pop layer
```

Because it reads global state, in a frame where two mods both draw, the tooltip describes
whichever widget ran most recently. It also runs in the same Lua call as the widget it
describes, not deferred to end of frame.

## Text

Measurement is cached per string: the first draw of a string measures and stores, later draws
reuse it. A multi-line text (one containing a newline) is drawn **per line**, and that path
also takes the per-line rendering option. A single-line text does not.

Text colour comes from the pending colour slot, which is opaque white by default. A
`GuiColorSetForNextWidget` applies to exactly one widget, because the commit resets it.

## A checklist for a reimplementation

If you are building this, the behaviours that are easy to miss and change how mods look:

- [ ] x and y are **percentages** by default, and the percentage is truncated to an integer.
- [ ] `GuiGetScreenDimensions` returns **UI units**, not pixels.
- [ ] The empty-id-stack case uses the **raw id**, unhashed.
- [ ] The pending option set resets to **1**, not 0.
- [ ] A widget's option set is **two words** (pending | frame-wide, low and high), not one.
- [ ] Most option bit positions are **no-ops**; implement only the tested ones.
- [ ] Layout/id/layer stacks are **not** reset per frame.
- [ ] Extents accumulate even when the cursor does not advance.
- [ ] `GuiTooltip` reads a **process-global** scratch record, not the `gui`'s own.
- [ ] Horizontal spacing **ignores its argument**.
- [ ] Margins are stored, not applied at begin time.
- [ ] The scroll offset is a **0..1 fraction of content**, and always animated.

The first four are the ones that will make your mod look broken in a way that is hard to
diagnose from the outside. The rest are subtler and matter mostly for pixel-compatibility.

## What to copy and what to fix

The quirks above are described as behaviour because that is what they are. If your goal is a
GUI that behaves the same, copy all of them. If your goal is a GUI that is composable, the
three worth changing are the ones that make mods interfere:

| quirk | change to | why |
|-------|-----------|-----|
| ids are raw and global | namespace per mod | stops two mods writing each other's widget state |
| `GuiTooltip` reads a process-global record | read the `gui`'s own | stops two mods' tooltips describing each other's widgets |
| per-frame stacks are not unwound | reset them | turns a forgotten `End` into a one-frame glitch instead of a permanent one |

Everything else on this page is arithmetic, and arithmetic you can reimplement directly.
