# Hardcoded GUI values: assets, defaults, and constants

The GUI's behaviour is not just formulas — it is formulas over specific assets and specific
numbers, most of which a mod never passes and cannot see. This page lists them.

Everything here is a hardcoded value baked into the executable. If you are reimplementing the
GUI, these are the parts you cannot derive from the API signatures.

For the formulas these feed, read [gui-cookbook.md](gui-cookbook.md). For option values, read
[gui-options.md](gui-options.md).

Addresses are for one Steam build and will move on update.

## Assets

These paths are compiled into the binary. A mod never supplies them, which means a
reimplementation has to have the same files or visibly different pixels.

### The default nine-piece

```
data/ui_gfx/decorations/9piece0_gray.png
```

This is the default for **`GuiEndAutoBoxNinePiece`** *and* **`GuiImageNinePiece`**, and it is
what **`GuiTooltip`** uses for its box. Both arguments default to the same file — the normal and
the highlight sprite are identical by default, so an unstyled auto-box does not visibly react
to hover.

`GuiTooltip` passes it twice, once for each sprite slot, so a tooltip's border is this grey
nine-piece unless a mod overrides the sprite arguments.

`GuiBeginScrollContainer` also references it directly, so a scroll container's frame is the
same grey nine-piece.

### Fonts

```
data/fonts/font_pixel_noshadow.xml
```

The default font, and the only one the GUI loads on its own. Reached by **`GuiText`**,
**`GuiTextCentered`** and **`GuiButton`** via a shared loader, and by
**`GuiGetTextDimensions`**.

Note the name: *no shadow*. The pixel font renders without a drop shadow, so a reimplementation
that adds one will look subtly wrong against every existing mod.

### Cursors

Drawn by the frame code, not by a `Gui*` call:

| asset | what it is |
|-------|-----------|
| `data/ui_gfx/mouse_cursor.png` | the mouse cursor |
| `data/ui_gfx/keyboard_cursor.png` | the text-input caret, left of the cursor |
| `data/ui_gfx/keyboard_cursor_right.png` | the text-input caret, right of the cursor |

The text input draws **two** carets, one either side of the insertion point, from two different
files. A reimplementation with a single caret will not match a mod that relies on the caret
width for text layout.

### Inventory sprites

| asset | used by |
|-------|---------|
| `data/ui_gfx/inventory/inventory_colors.png` | the image widget and the nine-piece widget, as the default sprite when no filename is given |
| `data/ui_gfx/inventory/highlight.xml` | the image widget's hover highlight |

The image widget picks `inventory/highlight.xml` when the mouse is over the image. That is the
only hover feedback an unstyled `GuiImage` gets.

### Sound

| asset | used by |
|-------|---------|
| `ui/button_click` | on a widget being clicked |
| `ui/button_select` | on a widget gaining hover focus |

Sound is not specific to `GuiButton`. **Button, slider, text input, image-nine-piece and the
scroll container all play these** — every interactive widget does. The pattern is the same
eachwhere: `ui/button_select` when the widget becomes hovered, `ui/button_click` when it is
clicked.

Both are **gated on option `15` being clear**:

```c
if ((options & 0x8000) == 0) {
    if (hovered && no_sound_flag) FUN_00828be0();   /* ui/button_select */
    if (clicked)               FUN_00828ad0();    /* ui/button_click  */
}
```

So `GuiOptionsAdd(gui, 15)` — the same option that takes a widget out of the layout — **also
silences it**. That is almost certainly unintended coupling, and it is a useful thing to know:
a mod that sets option 15 to get absolute positioning also loses its click sound.

The image-nine-piece additionally suppresses the select sound while the pointer is already
inside the widget, so hovering into a nine-piece does not double up with the highlight change.

## Documented defaults

These are the values the API falls back to when an argument is omitted. They come from the
embedded usage strings, so they are the game's own statements — but note the section on
[hardcoded fallback strings](how-the-api-works.md#a-missing-string-argument-comes-back-as-a-string-often-the-signature-itself)
below, because passing the wrong *type* also lands you on the default.

| function | defaults |
|----------|----------|
| `GuiText` | `scale = 1`, `font = ""`, `font_is_pixel_font = true` |
| `GuiTextCentered` | same as `GuiText` |
| `GuiButton` | `scale = 1`, `font = ""`, `font_is_pixel_font = true` |
| `GuiGetTextDimensions` | `scale = 1`, **`line_spacing = 2`**, `font = ""`, `font_is_pixel_font = true` |
| `GuiImage` | `alpha = 1`, `scale = 1`, **`scale_y = 0`**, `rotation = 0` |
| `GuiImageNinePiece` | `alpha = 1`, both sprites = `9piece0_gray.png` |
| `GuiEndAutoBoxNinePiece` | **`margin = 5`**, `size_min_x = 0`, `size_min_y = 0`, `mirrorize_over_x_axis = false`, `x_axis = 0`, both sprites = `9piece0_gray.png` |
| `GuiLayoutBeginHorizontal` | **`margin_x = 2`, `margin_y = 2`** |
| `GuiLayoutBeginVertical` | **`margin_x = 0`, `margin_y = 0`** |
| `GuiBeginScrollContainer` | `scrollbar_gamepad_focusable = true`, plus its own margin defaults |
| `GuiTextInput` | `allowed_characters = ""` — **empty means every character is allowed** |
| `GuiZSet` / `GuiZSetForNextWidget` | no defaults; the z is a plain float |

Four of these are load-bearing and easy to miss:

- **`margin = 5` on the auto-box** is the border around tooltips and grouped widgets. It is the
  single most visible hardcoded number in the GUI, and it is why `GuiEndAutoBoxNinePiece` with
  no arguments produces a padded box rather than a tight one.
- **`line_spacing = 2`** in text measurement. Multi-line text is spaced by this, so text height
  computed by the game and text height computed naively disagree by 2 units per line.
- **Horizontal and vertical layout margins differ** (`2,2` vs `0,0`). Copying one into the other
  shifts every widget in the layout.
- **`scale_y = 0` on `GuiImage`** is not a sane default. It is documented as the default and it
  means "unset" — the engine substitutes the scale from `scale`. Passing `0` explicitly is
  indistinguishable from omitting it, so you cannot ask for a zero-height image through that
  argument.

## Numeric constants

Float constants the GUI code uses directly. These are not arguments and cannot be changed.

| value | what it is for |
|-------|----------------|
| `0.5` | half-width maths: `x * 0.5` for the half-width alignment option, and the `* 0.01` percentage path's companion |
| `1.0` | the default scale, the UI-scale divisor when it is unset, and the "is it still animating" bound |
| `0.1` | **added to normalised draw coordinates** — the draw-command builders finish each computed coordinate with `+ 0.1` |
| `0.05` | the **animation step** — a `GuiAnimate*` advances the phase by 0.05 per frame, so a full animation is 20 frames |
| `1.15` | one rung of the **scale ladder** — see below |
| `1.2` | the other rung of the scale ladder |
| `2.0` | used as a size and spacing constant in several places |

### The scale ladder

Some widget paths pick a scale from a three-step ladder rather than from the `scale` argument:

```
1.0    default
1.15   when a per-Gui flag at state+0x79 is clear
1.2    when that flag is set
```

An option (`0x200000`) short-circuits this and forces `1.0`. So the scale a widget ends up at is
a function of shared per-`Gui` state, not only of the `scale` argument — two mods drawing in the
same frame with the same `scale` argument can get different sizes.

The two constants that change visible output most:

- **`0.05` animation step**, i.e. **20 frames** for a complete animation. At 60 fps that is a
  third of a second. Any reimplementation that picks a different step will have different
  scroll-smoothing and fade timings, and mods that tuned against the original will feel wrong.
- **`0.1` on draw coordinates.** Every draw command the GUI emits adds `0.1` to each computed
  coordinate — in the text draw builder, the image builders and the nine-piece builder alike.
  It is small, but it is applied per widget rather than per layout, so it does not cancel out
  and it is a systematic offset between the layout's idea of a widget's position and where its
  pixels land. Reproduce it or every drawing sits a tenth of a unit off.

## Behaviour with no API surface

A few things the GUI does that you cannot reach or change from Lua, and that a reimplementation
has to decide about explicitly:

- **`GuiStartFrame` clears a process-global scratch widget record** that `GuiTooltip` reads. Nothing
  about it is configurable.
- **The frame draws the mouse and text cursors itself**, from the three cursor assets above.
  There is no API call for them.
- **Scroll smoothing is always on.** The scroll container always passes the animation option, so
  the 0.05 step applies to it whether or not you want it.
- **The UI scale is recomputed every frame** from the window size, so it is not a value you can
  set once. It is 1.0 unless something else changes it.

## What to check when reimplementing

The things that are cheap to get wrong and obvious once you see them:

- [ ] The **default font** is `font_pixel_noshadow.xml`, and it renders **without a shadow**.
- [ ] The **default nine-piece** is `9piece0_gray.png`, and it is used for the *highlight*
      sprite too, so an unstyled box does not change on hover.
- [ ] **`GuiEndAutoBoxNinePiece` defaults `margin` to 5**, not 0.
- [ ] **Text line spacing is 2**, not 0.
- [ ] **Horizontal layout margins are 2, vertical are 0.**
- [ ] **The text input draws two carets** from two different images.
- [ ] **A full animation is 20 frames** (0.05 per frame).
- [ ] **Draw coordinates get `+ 0.1`** before they are emitted.
- [ ] **`GuiTooltip` hardcodes the grey nine-piece twice** and passes no `x_axis`, so a tooltip
      is always that border and never mirrors.
- [ ] **Interactive widgets are not silent** — button, slider, text input, image-nine-piece and
      the scroll container all play `ui/button_select` and `ui/button_click`, and option `15`
      silences both.
- [ ] **`allowed_characters = ""` means "all characters allowed"**, not "none".

The first four are the ones that change how every existing mod looks. The rest are smaller but
equally invisible when missing.
