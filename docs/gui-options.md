# GUI_OPTION: what each option actually does

`GUI_OPTION` is a **bit position, not a mask** — the engine does the shifting, so you pass one
value per option and never a pre-combined mask. See
[enums.md](enums.md#gui_option-values-are-bit-positions-not-bit-masks) for why.

The option *values* are game data (`data/scripts/lib/utilities.lua`) and are not in the
executable. The option *effects* are: each one is a distinct bit tested by specific code, and
this page is what those tests do. So this is the page that tells you what a given number does,
even without the name the game gives it.

Read this with [gui-layout.md](gui-layout.md), which has the coordinate maths these options
feed into.

Addresses are for one Steam build and will move on update.

## Most of the option space is dead

**Read this before the table.** The option set is 64 bits wide, but the bit positions the engine
actually tests are a short, non-contiguous list. Most values in `0`–`63` are **no-ops**: setting
them changes nothing at all, on any widget, in any context. There is no error and no log.

The reason is that an option only does something if some code tests that exact bit against the
option word. The engine passes the option set into the widget functions as two plain 32-bit
values and tests a specific handful of masks. Every other bit is carried along, never read, and
silently discarded.

Worse, the bits that *are* tested are split across the two halves, and **the game's own option
names do not obviously line up with them** — the names live in `data/scripts/lib/utilities.lua`
and are not in the executable, so a value the game defines can be a no-op while an undocumented
value does something. Treat the table below as "these bit positions have this effect", not as
"these are the game's constants".

### The tested bits, and the gaps

Bits with a confirmed behavioural effect — **0, 2, 3, 5, 6, 7, 8, 10, 11, 12, 13, 14, 15, 16, 17,
19, 21, 22, 23, 24, 25, 29**, plus 30 and 31 which are sentinels rather than features.

The gaps — **4, 18, 20, 26, 27, 28**, and everything from 32 upward: no test of the option
word was found anywhere in the GUI call graph. On the evidence in the executable these are
no-ops.

**Bit 9 was on this list and should not have been.** It is read in the widget-state lookup
(`out/decomp-gui/0081b090.c:227`), where it gates a latch of the pending offset into the state
entry when the frame counter has advanced — see the input table below. The game's own pause-menu
buttons all set it. The census behind this list missed a consumer outside the widget builders;
re-run it before trusting any of the remaining gaps.

Bit 1 is the interesting one. `0x2` is tested in five places, but **none of them is a test of
the option word**: they are frame bookkeeping, a mask read in the text function, and two tests
of a *different* variable that merely gets OR-ed with the option set nearby. A modder passing
option `1` gets nothing.

Bit 4 is the same story in miniature: `0x10` appears twice, once in a mode-word comparison and
once in a size clamp. Neither reads the option set.

So the dead space is not a contiguous tail, it is scattered through the low bits — which is why
it survives contact with the documentation. A modder picking a "reasonable" low value like `1`,
`4` or `9` lands in it.

### A hardcoded bit you cannot set or clear

One wrapper passes a constant into the low option half:

```
low = (gui + 0x10) | (gui + 0x08) | 0x10000
```

So bit `16` is **always set** for that widget, whatever you do. `GuiOptionsRemove` cannot clear
it and `GuiOptionsAdd` is redundant. Which widget it is depends on the wrapper — at least one
text path hardcodes it.

This means "the option set a widget sees" is not purely a function of what you passed. If you
are reimplementing, decide deliberately whether to keep that bit hardcoded; matching mod output
requires it, and it is very likely an engine bug rather than a design choice.

## How this was derived, and how far to trust it

Every entry below comes from a `TEST`/`AND` of a single bit against the option word inside the
GUI code. That is solid for *where* a bit is read, and it is what makes this page possible at
all when the option *names* are unavailable.

The procedure: a widget receives the option set as two arguments, built by the Lua wrapper as
`(pending_low | frame_low, pending_high | frame_high)`. So a mask appearing in a widget function
is an option test only if it is applied to that value — which is why a bit can appear in the
function and still not be a real option, and why bit `1` is a no-op despite five hits. Every
claim on this page was checked against which word the test applies to.

The **effects** are named by reading the code at each test site, and some are inferences from
the surrounding code rather than things the binary states. The bit numbers, the gaps, and the
options that gate large obvious branches are solid. A few of the finer ones — the low text bits,
the sentinel bits — are described as "what the test does", which is accurate but less useful than
a name would be.

To re-derive or extend: `OptionBits.java` reports every bit test in the GUI call graph, and
`scripts/option_context.py` reprints each one with the register it applies to. When a game's
update adds options, the ones that show up here as new tests are the ones that started working;
the rest are still decoration.

## How to read the bit numbers

An option value `v` sets bit `v & 31` of a 64-bit set stored as two 32-bit halves. So the bit
this page talks about is the bit position, and the hex in the "bit" column is that position as a
mask.

| value range | which half |
|-------------|-----------|
| `0`–`31` | the low half |
| `32`–`63` | the high half |
| `>= 64` | **neither** — silently dropped |

`v >= 64` sets nothing at all: no error, no log. A negative value goes through `& 31` and sets
an arbitrary low bit.

## The options, by effect

Behavioural options, grouped by what they actually do. Names are descriptive; the game calls
them whatever `utilities.lua` calls them.

### Positioning and alignment

These are the ones you will use. They are all handled by one function, which every widget goes
through on its way to a final position.

| value | effect |
|-------|--------|
| `0` | **default alignment.** The reset value, restored after every widget commits, so it is set on every widget that does not choose an alignment. Bit 0 is the reason a widget is never left unaligned. |
| `10` (`0x400`) | Align **right**: shift x left by the widget's own width, plus the layout's `margin_x`. |
| `11` (`0x800`) | Align **left**: shift x right by the widget's width plus `margin_x`. |
| `12` (`0x1000`) | Align **bottom**: shift y up by the widget's height, plus `margin_y`. |
| `14` (`0x4000`) | Suppress the **y cursor advance** in a vertical layout. The widget is placed but the layout does not move down for it. |
| `13` (`0x2000`) | Force the origin-relative placement branch, even when a layer is on top of the layout stack. |
| `15` (`0x8000`) | **Ignore the layout entirely.** Skips cursor maths, placement and bounding-box accumulation; the widget keeps the position computed from its own arguments. The only other option that does this much. |
| `16` (`0x10000`) | Shift x left by **half** the widget's width. |
| `17` (`0x20000`) | Shift x left by the **full** widget's width. |
| `19` (`0x80000`) | Treat a click on this widget as a **drag**, not a click: records the mouse-down position into the widget-state entry instead of consuming the press. |

Options `16` and `17` are how a widget is centred or right-aligned **relative to a fixed
anchor** rather than to the cursor — combine them with `15` when you want a widget at an exact
screen position and no layout participation at all.

Note that `10`, `11`, `12` each replace the normal cursor advance, they do not stack with it.
Setting two of them takes the first branch in the chain, so pick one.

### Animation

| value | effect |
|-------|--------|
| `19` (`0x80000`) | **Drag instead of click.** See the input section. |
| `23` (`0x800000`) | **Animate.** Advances the widget's animation phase by a fixed step each frame, clamped at 1.0, and resets the phase to 0 when the animation ends. This is the option the scroll container always passes, which is why scroll smoothing is always on and cannot be switched off from Lua. |
| `25` (`0x2000000`) | Advance the phase **without** the "is it still animating" test — used by the begin/end animation pair to keep a phase ticking across a `GuiAnimateBegin` / `GuiAnimateEnd`. |
| `24` (`0x1000000`) | **Lerp toward full size** on hover. See the rendering section. |

### Low and high bits that are sentinels, not features

Two bit positions are tested as **equality against the whole mask** rather than for truth. That
means they only mean something when set alone, and setting them alongside any other option
silently disables them. Neither is a feature you want to reach for.

### Bits that are not features

| value | how it is tested | effect |
|-------|------------------|--------|
| `31` (`0x80000000`) | `== 0x80000000`, not for truth | A sentinel on the button and text-input path: it forces a flag to 0 when it is the *only* bit set, and silently does nothing alongside any other option. Not a feature — do not reach for it. |
| `30` (`0x40000000`) | `== 0`, always alongside bit `16` | Gates a frame-level path together with bit `16`. It is never tested on its own, so it is a *companion* to bit `16` rather than an option of its own. |

Bit `0` is the exception to all of this: it is the **default alignment** marker, and it is the
value the per-widget commit resets to, which is why it is on after every widget.

### Rendering and appearance

| value | effect |
|-------|--------|
| `22` (`0x400000`) | Draw the widget at an **animated offset** derived from the frame counter and a per-widget constant, so it moves as the frame count changes. |
| `21` (`0x200000`) | **Force the full-size scale.** Without it, a widget uses the 1.15-scale variant in some paths and skips a trailing pass. |
| `24` (`0x1000000`) | **Lerp toward full size** on hover: interpolates the widget's scale toward 1.0 using a stored value, and resets that value to 0 when hovered. |

### Input

These are the bits that decide whether a widget reacts to the mouse. They are read together, so
read them as a group.

| value | effect |
|-------|--------|
| `2` (`0x4`) | **Non-interactive.** Setting this bit takes the widget *out* of hit-testing: it is drawn but cannot be hovered or clicked. **The polarity is the opposite of what it looks like** — a *clear* bit 2 is what makes a widget interactive, which is why `GuiStartFrame` seeds the word with `1` and `GuiOptionsClear` restores `1`. The community enum mirror in `data/scripts/lib/utilities.lua:1011` names it `NonInteractive`, and that name is correct. |
| `3` (`0x8`) | **Full hit-test.** Overrides bit 2: with `2 \| 3` set the widget is hit-tested anyway, and the test becomes strict rectangle containment. |
| `8` (`0x100`) | **Click while held.** Lets the widget report a click when the mouse button is still down, rather than requiring a fresh press inside it. |
| `19` (`0x80000`) | **Drag instead of click.** When the mouse goes down inside the widget, this records the press position into the widget-state entry instead of consuming it as a click, which is the first half of drag support. |
| `6` (`0x40`) | **Override the computed position** with the coordinates passed to the widget, bypassing the animated/interpolated position the widget would otherwise use. |
| `9` (`0x200`) | **Latch the state's offset field.** Not a no-op, despite appearing in the dead-gap list. Read in the widget-state lookup (`out/decomp-gui/0081b090.c:227`): when set, it copies the pending offset into the state entry if the frame counter has moved. The game's own pause-menu buttons all set it. |

The hit test, exactly as compiled (`out/decomp-gui/008245d0.c:150-151`):

```
hovered = state_entry_is_new
      && !claimed_this_frame
      && (bit2_clear || bit3_set)          <- note the polarity
      && inside_rectangle
```

Note the ordering: the last term is the "one widget owns the mouse" rule — the first widget to
claim it sets a frame flag and later widgets are not considered hovered, regardless of their
options. That flag lives at `*(gui+0x88) + 0x1fd`, **per `gui` object**, and is cleared only by that
object's own `GuiStartFrame`. Two `Gui` objects never compete; see [ui-modding-2.md](ui-modding-2.md).

**And because the option words are just `gui+0x08`/`gui+0x0c` with no ownership check**, you can set
them on a `gui` the *game* gave you — but only for widgets drawn **through the Lua API**. The
builders take the option word as an argument (`008245d0.c:68-69`), and the only code that merges
`gui+0x08`/`gui+0x10` into that argument is the Lua wrapper layer: all 20 merge sites are in the
`0x007d`–`0x007e` band. The game passes literals (`006e3410.c` uses `0x8010000` on every pause-menu
button), so anything on the `gui` object is Lua-private state. See [ui-holes.md](ui-holes.md).

### Multi-line and text

| value | effect |
|-------|--------|
| `29` (`0x20000000`) | **Per-line rendering**: a text widget draws its lines individually rather than as one block. Only read on the multi-line path; the single-line path ignores it. |
| `5` (`0x20`) | Bypass one of the text widget's position refinements — when unset, a text widget consults the shared string-metrics store and can be repositioned by it. |
| `7` (`0x80`) | Suppress that same refinement in the nine-piece image widget under a specific argument combination. |

## What is *not* here

Two things a modder might expect and will not find:

- **No option controls clipping.** Clipping comes from the scroll container's clip records, not
  from a widget option. There is no "clip children" option.
- **No option makes a widget ignore the id stack.** Id hashing is unconditional; use a
  different id, not an option.

## Practical notes

- **Option `15` is the one to reach for** when a widget must land at an exact position. It is
  the only option that takes the widget out of the layout flow entirely, and combined with
  `16`/`17` it gives you centre and right anchoring against a fixed point.
- **Options are per-frame or per-next-widget, and they are cleared asymmetrically.** The
  frame-wide set (`GuiOptionsAdd` / `GuiOptionsRemove` / `GuiOptionsClear`) is reset to 0 at
  `GuiStartFrame`; the pending set (`GuiOptionsAddForNextWidget`) is consumed and reset to
  **1**, not 0, by the per-widget commit. `GuiOptionsClear` only zeroes the **high half** of the
  frame-wide set and does not touch the pending set at all, so an abandoned next-widget option
  can still apply to a later widget, and clearing does not clear everything. Full detail in
  [enums.md](enums.md#there-are-two-option-fields-and-only-one-is-cleared).
- **Since values are bit positions, combining options means calling the function repeatedly.**
  There is no way to pass `ALIGN_RIGHT | SOMETHING_ELSE`; you call `GuiOptionsAdd` once per
  option.
- **Check the value is real before you build a mod on it.** Given the dead space above, the
  cheap test is: set the option, draw one widget, and see whether anything moves. An option
  that does nothing produces no diagnostic, so a mod built on a no-op option looks like a mod
  with a layout bug.
