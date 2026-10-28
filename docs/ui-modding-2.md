# Getting into the pause menu: what actually works

Companion to [ui-modding.md](ui-modding.md). That page covers the sanctioned surfaces. This one
covers the things people hack around — *detecting* which screen you are on, *reading* the game's
own widget rectangles, *injecting* rows into a vanilla menu, and *stopping clicks from reaching a
menu you are drawing over* — and states for each whether it is possible, and by what route.

## The short version

| goal | possible? | route |
|---|---|---|
| know if the game is paused | **yes** | `OnPausedChanged(is_paused, is_inventory_pause)` |
| know if you are in the front-end or the in-game pause menu | **yes** | second arg of `ModSettingsGui` |
| know if the inventory panel is open | **yes** | `GameIsInventoryOpen()` |
| know if Options or Progress is open | **yes**, as callback side effects | `OnPausedChanged` on Options entry/exit, `OnCountSecrets` per Progress frame — see below |
| know if New Game, Mods, Release Notes or Credits is open | only by reconstruction | no callback; replay the menu and track input |
| read a vanilla widget's rectangle | **no** | the previous-widget record is only written by Lua wrappers | `gui` — see below |
| add a row to the options screen | **yes** | `ModSettingsGui`, already sanctioned |
| add a row to the pause menu or main menu | **no** | see "What stays closed" |
| add a *new tab* to the options screen | **no** | the tab list is C++ and hardcoded |
| add a row to the HUD | **yes** | `UIIconComponent` |
| move anything vanilla | only via `magic_numbers.xml` | no pixel control from Lua |
| block clicks to *your own* widgets | **yes** | claim the mouse on your own `Gui` first — see below |
| block clicks to a *vanilla* menu under your overlay | **no** | the claim is per-`Gui`; see below |
| swallow Escape | **no** | handled in C++ above every `Gui*` call |

The one genuinely new thing here is that **the `gui` object the game hands you is the game's own,
and it is the same object every menu screen draws into.** That makes it usable as a drawing surface
inside `ModSettingsGui` — you can put a full-screen opaque panel on the options screen's own `gui`
and it will be composited correctly with the rest of the screen.

It does **not** make vanilla widget geometry readable; see the next section.

## Screen detection: you are not as blind as advertised

Three separate signals exist, and they answer different questions.

**Paused at all, and paused *how*** — `init.lua:12` documents the real signature:

```lua
function OnPausedChanged( is_paused, is_inventory_pause )
```

The second argument is what people are actually after: it separates the pause menu from the
inventory / wand panel, which are both "paused" but are different screens with different widgets.

**Front-end vs in-game pause menu** — the second argument to `ModSettingsGui`:

```lua
-- the game passes exactly two arguments: gui, then this boolean
function ModSettingsGui(gui, in_main_menu)
```

`in_main_menu` is true in the main menu and false in the in-game pause menu. It is a direct read of
the engine's own front-end flag (`DAT_0120761b`, set once per frame at the top of the menu runner
and pushed as the boolean at the call site — see [ui-settings.md](ui-settings.md) for the address).

Note the arity: it is **two** parameters, not three. `im_id` is a mod-framework convention that the
game does not supply.

**Inventory panel specifically** — `GameIsInventoryOpen()`. This is the only `Game*` predicate in
the API that reads UI state; it returns the engine's own "inventory is open" flag rather than
anything derived.

There is no predicate for "which menu screen am I on" beyond these three. Options and Progress
can be followed through callback side effects (next section); there is nothing for the
world-select screen or the game-over screen. For those, see "Detecting an arbitrary screen" below.

### Is the base pause menu visible?

`OnPausePreUpdate()` is **not** a base-pause-menu visibility callback. It runs on paused/menu update
frames and takes no arguments. The game uses a C++ menu stack: when the stack is empty it draws the
base pause menu; when it is non-empty it draws the top submenu instead. Opening Options, Controls,
or another stacked screen does not stop `OnPausePreUpdate`, and returning to the base menu does not
produce a dedicated Lua event.

Two callbacks do carry screen information, as side effects:

- **`OnPausedChanged` fires on every write of the pause-flag word, not only on pause toggles.**
  The game keeps one flag word (bit `0x4` paused, `0x2` inventory, `0x8` Options open, `0x10`
  replay editor). Every writer calls the same notifier (`0x006f4d10`), which dispatches
  `OnPausedChanged(flags > 0, flags & 2)` with no comparison against the previous value. The
  Options row sets `0x8` (`0x006e3410`), and both ways out of Options clear it: Apply & Return or
  Escape with nothing changed (`0x006cc450`), and "Yes" on the display-mode confirmation
  (`0x006cc9b0`). So within an open pause menu, `OnPausedChanged(true, false)` means **Options
  opened** when you were on the base menu and **Options closed** when you were in Options. A
  display-mode change does not close Options directly: it pushes the "are you sure?" countdown
  dialog, and "No", Escape and the timeout all return to Options without a callback. Options
  opened from Mods or New Game (their "mod settings" route) does not set the flag, but its exit
  still clears it, so you get one callback, on the way back to that screen.
- **`OnCountSecrets` runs on every frame the Progress screen is drawn.** Its only call site is
  the Progress screen function (`0x006dfe40`), which is called each frame while it is the top
  screen, and nothing else calls that function's dispatcher. It is the same per-mod Lua state as
  your `init.lua`; return two integers (`0, 0` if you only want the signal). Since it fires
  inside the menu draw, which runs after `OnPausePreUpdate`, a paused update sees the previous
  frame's calls. Progress is reachable only from the base menu, so when the calls stop you are
  back on the base menu.

The pause itself is set only by the back action — an Escape key press or a gamepad Start press —
on an empty menu stack. The replay editor (default key F11) raises the same `(true, false)`
callback from gameplay without either key, which is how the two can be told apart. The callback
still names no screen; New Game, Mods, Release Notes and Credits produce no signal at all.

### Reconstructing Options → Apply & Return

The native button is drawn at a formula-based position. In the analyzed build, the Options screen
starts from `(screen_width / 2 - 169, screen_height × 0.14)` in UI coordinates. The footer adds an
X inset of `-10` and the options panel's effective height (`281` UI units) to Y. The panel builder
starts with the requested height (`265`) and adds `8` units of padding on both sides, then stores
the resulting height in its widget record at offset `0x30`; the footer reads that field when
building its position. These are the same scaled viewport units returned by
`GuiGetScreenDimensions`, and the button bypasses layout positioning. The reconstructed position is
`(screen_width / 2 - 179, screen_height × 0.14 + 281)`.

Lua can recreate this position and measure a duplicate localized text widget for its own hover
state. Use `GuiButton` for the replica: `GuiText` is non-interactive. This is a build-specific
reconstruction from the native draw calls, not a supported screen or widget query. You do not need
it to follow Options: the `OnPausedChanged` callback on exit is exact, and it also covers the
display-mode confirmation dialog, which this button opens instead of closing Options.

So the supported signals tell you that gameplay is paused, whether the inventory panel is open, and
(as side effects) when Options and Progress open and close; they cannot distinguish the base pause
menu from New Game, Mods, Release Notes or Credits. `ModSettingsGui`
is a positive signal for a mod settings panel when that callback runs, not a general screen-change
notification. Vanilla widget rectangles and IDs do not provide a supported substitute.

There is a direct but build-specific route if your mod can use LuaJIT FFI. The pause menu draws its
base screen when the C++ menu stack is empty; submenus push a screen, and returning to the base menu
pops it. On the analyzed executable, this stack condition is statically verified but has not been
runtime-tested. Read the stack begin/end globals and compare them:

```lua
local ffi = require("ffi")
local function u32(address) return ffi.cast("uint32_t*", address)[0] end

local MENU_STACK_BEGIN = 0x012076bc
local MENU_STACK_END   = 0x012076c0

function OnPausePreUpdate()
    if GameIsInventoryOpen() then return end

    local on_base_pause_menu =
        u32(MENU_STACK_BEGIN) == u32(MENU_STACK_END)
    if on_base_pause_menu then
        draw_my_overlay()
    end
end
```

This reads internal process memory, so it is not a supported or portable API. The addresses are for
the analyzed executable and must be rechecked after a game update. `require("ffi")` is only
available to mods granted unrestricted APIs by the game setting, or when the executable has been
patched to expose it; see [Reading screen shake](screenshake.md#reading-it-exactly-the-luajit-ffi)
for that access setup. The comparison tells you whether the base menu or a stacked submenu is
active; keep `GameIsInventoryOpen()` (and any other screen exclusions your mod needs) as a separate
filter.

Without FFI/native memory access, the workable fallback is an input-driven overlay: show it after
the pause key opens the pause screen, then hide it on the first observed input. The logo-splash
pattern used by `noita.hardmod` does this broadly across mouse, keyboard, gamepad buttons, sticks and
triggers. Because it scans every key in its list, it also treats F11 as menu input; F11 is the
default `key_replayedit_open` binding. A no-FFI implementation should whitelist actual menu
navigation inputs instead of scanning every key. Its coordinates use
`UI_PAUSE_MENU_LAYOUT_TOP_EDGE_PERCENTAGE` to place the splash near the vanilla logo; they do not
detect the menu screen.

This gets the overlay out of the way on the first detected input, but it cannot
tell whether that input opened a submenu, selected an item, or returned to the base menu. The
logo-splash pattern does not restore the overlay on return. You can special-case a known back input
(Escape, for example) and re-show it, but that is only reliable for a menu flow where that input
always returns directly to the base pause screen. The input API reports raw input, not which vanilla
widget consumed it, so coordinates cannot turn this into a general screen detector.
You can reconstruct a no-FFI tracker for a defined set of vanilla routes:

1. Start in `base_pause` when `OnPausedChanged(true, false)` fires from a closed state with Escape
   or Start pressed; clear the tracker when the game unpauses or enters the inventory pause.
2. Follow Options and Progress by their callbacks (see "Is the base pause menu visible?"): a
   `(true, false)` callback on the base menu enters Options and one in Options returns to the base
   menu; Progress is open while `OnCountSecrets` keeps firing.
3. For the other screens, in `OnPausePreUpdate`, match clicks against hitboxes replayed in your own
   `Gui` (below). Push the destination screen when a button opens a submenu.
4. Pop on the back action or a click on the submenu's Return. The back action is an **Escape key
   press or a gamepad Start press** and nothing else: the keyboard handler (`0x006b5e20`) reacts
   to key `0x29` and the gamepad handler (`0x006b83c0`) to button `15`; the shared back routine
   (`0x006e9810`) has no other callers apart from two Return buttons. Gamepad Back and B do not go
   back. The routine pops exactly one screen. Draw the overlay only while the tracked screen is
   `base_pause`.

New Game and Mods can open further screens on top of themselves (confirmation dialogs, world
select, mod settings), and nothing reports when those close. After a click inside either screen,
a tracker cannot know whether the next back action returns to the base menu or to that screen.
Treat the depth as unknown until a resync: unpausing, or Progress (reachable only from the base
menu) drawing and closing.

Two base-menu labels can change width without Lua being able to predict it. Progress reads
`Progress ($0 new)` (`$menu_progress_new`) when any `new_perk_picked_<id>`, `new_action_<id>` or
`new_kill_<name>` run flag is set; the kill part iterates the files in
`data/ui_gfx/animal_icons/`, a directory listing Lua cannot enumerate. Mods reads `Mods (!)`
(`$menu_mods_incompatibilities`) after a mod-version comparison that reads files Lua is not allowed
to open. Both labels are centered, so a hitbox the width of the wider variant covers both.

New Game and Credits call the same native Return helper. It starts a fresh GUI layer and places the
button at `(0.04 × width, (1 − 0.07) × height)`, using the active GUI scale just as
`GuiGetScreenDimensions` does. Mods also anchors its footer layout at `(0.04 × width,
(1 − 0.07) × height)`; its Return button is at local `(0, 0)` inside that layout. Progress
computes the same y anchor, then replaces x with the aggregate count returned by loaded mods'
`OnCountSecrets` callbacks. Its native Return is at
`(aggregate secret count, (1 − 0.07) × height)`, so its x can change with the installed mods and
game state.

Release Notes draws its body in a 280-unit scroll container inside a vertical layout that begins at
10% of the screen height. The scroll container adds 2 units of padding at both ends, the parent
layout has a −2 vertical margin, and the native code advances the parent cursor by another 8 units
before placing Return at local `(0, 0)`. Its widget y is therefore `0.1 × height + 290`; it is not
at the screen origin. Its x depends on half the measured width of the live release-notes text, so
Lua cannot recover the exact horizontal coordinate from the exposed UI API.

A no-FFI tracker can reproduce Return's hitbox with a `GuiButton` at the known engine coordinates.
New Game, Credits and Mods have recoverable positions. Progress instead uses the sum of every mod's
`OnCountSecrets` result for x, and Release Notes centers Return from the measured live
`_release_notes.txt` width; neither dynamic value is exposed to another mod's Lua state. Native code
opens the notes file at the game install root, while the supported `ModTextFileGetContent` paths are
under `data/` or `mods/`, so that API cannot supply the live text to recompute its width. Progress
does not need a hitbox at all, since `OnCountSecrets` stops when it closes. For Release Notes, the
only widget below the text box is Return, and its left edge (centered text) is never right of the
screen center, so a probe covering that row from x = 0 to the center plus the label width catches
every Return click; its only false positives are clicks on empty space in that row. Escape and
gamepad Start always pop it. The GUI engine
handles conversion from raw mouse pixels to UI units, and the Y positions are independently
reconstructible. These positions are specific to the analyzed build; other submenu routes need their
own mapping.

This is a real alternative to FFI, but it is a reconstruction. For mouse selection, you can replay
the relevant localized text rows in your own `Gui` using the same font and layout, including the
current `UI_PAUSE_MENU_LAYOUT_TOP_EDGE_PERCENTAGE` magic number and the native display factor and
integer truncation used for its vertical origin, then read `GuiGetPreviousWidgetInfo(gui)`'s
`hovered` result immediately after each text widget. The engine computes hover in that GUI's
coordinate system, so this avoids converting raw mouse coordinates. A transparent GUI widget can
also model a footer hitbox, but widening it beyond the label trades fewer missed hits for false
positives elsewhere on that row.
The native factor is 0.25 when the game treats a gamepad as the active device and 1.0 otherwise;
the same test picks the gamepad or keyboard help image. The active device (`0x00a8ccc0`) is the
gamepad selected in the `gamepad_mode` option when that pad is connected, none when the option is
off, and in auto-detect the pad with the most recent activity timestamp if it is newer than the
keyboard's and the mouse's (one of the two mouse timestamps is taken minus 90). Which mouse event
each timestamp records was not traced.
The returned hover belongs to your text replica; it does not reveal whether a vanilla widget was
hovered or activated. The replica must match the vanilla layout, and its text bounds may differ
from a `GuiButton` hit area. Include the help image in the vertical layout before the heading: the
native helper draws it with option `0x10000`, which still advances the vertical cursor. Omitting it
leaves the replayed labels near the top while the native labels follow the tall help image. Include
every nested route and
back action you need to support; controller navigation needs its own transition mapping. An
unmodeled path can desynchronize the tracker. Footer hitboxes vary between screens, so do not use
one shared bottom-strip region as a universal screen-change event.

## Reading a vanilla widget's rectangle: not possible

It looks as though `GuiGetPreviousWidgetInfo(gui)`, called with the options screen's own `gui` inside
`ModSettingsGui`, returns the rectangle of a widget the *game* drew. It does not, and the mechanism looks like it should.

### What the function actually does

`GuiGetPreviousWidgetInfo(gui)` returns eleven values — `clicked`, `right_clicked`, `hovered`, then
`x, y, width, height` and `draw_x, draw_y, draw_width, draw_height`. The record it reads is
**per-`gui`, at `gui + 0x38`** (`0x007e3b40:129`, with a fallback to the static
`DAT_01225fb0` only when the handle fails validation at `:134`).

That part is correct, and it is the part that makes the claim look reasonable: the record is a field
of the object, so of course a widget drawn on that object should land in it.

### Where the reasoning breaks

The record is filled by exactly one commit function, `FUN_007d97d0(gui, record)`. It has **11
callers, and every one of them is a Lua wrapper** — `0x007dce60`, `0x007dd400`, `0x007dd900`,
`0x007ddfb0`, `0x007de5e0`, `0x007dec70`, `0x007df250`, `0x007df8d0`, `0x007dff60`, `0x007e02a0`,
`0x007e0db0`. That is the whole `0x007d`–`0x007e` wrapper band and nothing else.

The game's own widgets are built by a different set of functions entirely: `0x008245d0` (button),
`0x00823580` (image button), `0x00825cb0` (text input), `0x0081b090`, `0x00821630` and so on. **None
of them calls `FUN_007d97d0`.** They compute their rectangle into a stack-local record, use it, and
discard it.

So the options screen's widgets never write `gui + 0x38`. What you read back is the last widget
**you** drew, or one drawn by an earlier mod's `ModSettingsGui` in the same frame — nothing the game
produced. In practice that means `GuiGetPreviousWidgetInfo` inside `ModSettingsGui` returns your own
previous widget, which you already knew the position of.

There is a separate global, `DAT_01154b98`, that the ten Lua wrappers build into before committing.
It is genuine scratch state and it is genuinely global — but it is only ever written by Lua wrappers
too, so it does not help. It is read by `GuiTooltip`.

### What this leaves you

- **A drawing surface, not a geometry oracle.** Handing you the game's `gui` is still useful: you can
  draw a full-screen panel on it and get correct compositing and z-ordering. That is a real,
  narrower win, and it is what the rest of this page is about.
- **No hover detection on vanilla widgets.** You cannot ask "is the cursor over the Options tab?"
  and you cannot measure the tab strip. Any mod doing that is hardcoding pixels.
- **The one thing that does work** is detecting clicks on your own rows, which you would get from
  `GuiButton` directly.

### The draw-order fact that does survive, and is worth more

Because the game *does* pass its own `gui`, and because input on any one `gui` is awarded by draw
order, you can influence which vanilla widget gets the cursor. The options screen draws like this:

| lines of `0x006d5620` | what |
|---|---|
| 235–460 | the tab strip |
| **1208** | `if (DAT_012076ac != 5) goto end` — everything below is the Mods tab only |
| 1214 | a hidden zero-size button, id `0x3615` |
| 1253–1356 | the per-mod loop; `ModSettingsGui` is called at **1341** |

Your callback runs *after* the tab strip and *before* the end of the Mods tab. So a full-screen
interactive widget drawn at the top of your `ModSettingsGui` takes the cursor away from everything
the screen still has left to draw — which, on the Mods tab, is `0x3617` and the trailing text. It
cannot take it from the tab strip, which is already committed to the frame before you are called.

That is a partial input block, and it is as far as this goes. See [ui-holes.md](ui-holes.md) for the
full picture and the things that stay closed.

## Adding rows: `ModSettingsGui` is a real settings page

Already covered in [ui-modding.md](ui-modding.md), but the mechanism is worth stating precisely,
because it explains what you can and cannot add.

Your settings are described as data and rendered by the engine's own generic settings renderer.
`ModSettingsUpdate(id, name)` declares one; the renderer picks a widget from the *Lua type of the
default value*:

| `value_default` type | widget |
|---|---|
| `boolean` | a text button reading on/off; left-click cycles, right-click resets |
| `number` | a slider, needs `value_min` / `value_max` / `value_default` |
| `string` + a `values` table | a button that cycles through the options |
| `string` | a text input |
| none, plus `not_setting` | a heading, `ui_name` only |

Nesting works: a setting with a `category_id` becomes a collapsible section with a fold arrow, and
`foldable = false` makes it a plain indented group. So an arbitrarily deep settings tree is
available — the constraint is that the *page* is one page, inside the mods section, and its rows
are laid out by the same code for everyone.

`ui_fn` replaces the widget entirely for one setting, which is the escape hatch: pass your own
function and you can draw whatever you want in that row's slot, still inside the vanilla layout.

### The undocumented fourth option: shadow the renderer

The generic renderer above is not in the executable. It is `data/scripts/lib/mod_settings.lua`, a
**shipped vanilla Lua file** — 302 lines, and the only Lua in the game that draws real UI. Every mod
that wants a settings page `dofile`s it (e.g. `Apotheosis/settings.lua:1`) and calls
`mod_settings_gui(mod_id, settings, gui, in_main_menu)`, which the engine then calls back.

That makes it a data file, and data files are writable. Three consequences:

1. **`ModTextFileSetContent("data/scripts/lib/mod_settings.lua", ...)` replaces it for every mod.**
   Later writes win, and you have the last word. You can change how *all* mod settings are drawn —
   add a column, change every slider, restyle the fold arrows.
2. **`ModLuaFileAppend` is the survivable form.** Append a wrapper that calls the original and then
   draws your own rows, rather than rewriting 300 lines you have to keep in sync with updates.
3. **You can extend the type table.** Add a case for a new `value_default` type, or a new
   `ui_name`-bearing pseudo-setting, and every mod that `dofile`s the file gets it.

This is the closest thing to a supported extension point for the pause menu that exists. It is also
a shared global resource — coordinate, or you will break other people's settings pages.

## Building your own menu over the pause menu

This is the part people actually want, and the honest answer has two halves: **the mouse half is
solved, the keyboard half is not, and the clean solution is to stop overlaying.**

### You cannot steal the mouse from another `Gui` object

The engine lets exactly one widget per `Gui` object be hovered per frame. The first widget to
claim the mouse sets a byte — `state+0x1fd`, where `state` is the 712-byte sub-object at `gui+0x88`
— and every later widget on that same object is unhoverable until `GuiStartFrame` clears it again.

**That byte is per-`gui`.** All three writers and all eleven read expressions (across nine functions) reach it as
`*(gui+0x88) + 0x1fd`; it is never a fixed global. So your `GuiCreate` object and the pause menu's
object each have their own claim, and **they do not compete**. Draw a full-screen invisible button
over the pause menu and you will block your own widgets perfectly while the pause menu carries on
happily underneath, still hovered, still clickable.

No `GUI_OPTION` bit changes *that*. But there is a different way to neutralise a screen you did not
draw — see [ui-holes.md](ui-holes.md).

### You cannot consume input either

All twelve `Input*` functions are pure queries — `lua_tointeger`, one virtual call on the global
input service, `lua_pushboolean`. There is no `InputSet*`, no consume, no block, no capture, no
inject anywhere in the 375-function API. The GUI's own input state (`state+0x1f6` mouse-down,
`+0x1f7` clicked, `+0x1fd` claimed) is **derived state on its own object**, cleared by its own
`GuiStartFrame`, and the three setters are reachable only from `GuiButton`, `GuiImageButton` and
`GuiTextInput`.

Keys are worse. Keyboard is read straight off the process-global input service by vtable slot, so
your `InputIsKeyJustDown` and the game's own key handling are looking at the same unconsumed state.
The pause menu's own widgets are mouse-only — the GUI band contains no key reading at all outside
`GuiTextInput` — so **arrow keys do not actually leak into it**. The key that does is Escape, which
the menu-stack code handles in C++ before any `Gui*` call, and there is no hook above it.

### So: stop overlaying. Add a row to the menu instead.

The structural reason overlaying is hard is that the pause menu is a **stack**, and a submenu
*replaces* the base menu rather than drawing on top of it. The menu runner draws the base menu only
when the stack is empty, and otherwise calls the top of the stack and nothing else:

```
if (menu_stack is empty)  draw_base_pause_menu()      -- includes the "$menu_options" row
else                      top_of_stack(gui)           -- e.g. the options screen
```

That is why the options screen has no click leakage into the pause menu: the pause menu is not
drawn at all that frame. It is also the shape you want for your own menu — **a screen, not an
overlay** — and there is exactly one such screen you can draw inside: the options screen, via
`ModSettingsGui`.

Inside that callback you share the options screen's `Gui`, so you share its claim byte. Draw your
full-screen blocker *first* and every vanilla widget the options screen draws after you that frame
is unhoverable. The ones drawn before you have already had their turn. In practice the options
screen draws its tab list and headings before it reaches the mods section, so you cleanly own
everything from your own row downward — which is where a settings panel wants to be anyway.

For a menu that must own the *whole* screen, the honest options are:

- **Be the options screen's mod section** and accept the tab strip above you.
- **Own the pause yourself.** Stop using the pause menu. Detect your own key in
  `OnWorldPostUpdate`, draw a full-screen opaque panel on your own `Gui` — where you have total
  input isolation, because nothing else is drawing — and freeze the simulation yourself. There is
  no `GamePause`, so "paused" has to be simulated: `EntitySetComponentsWithTagEnabled` on the
  player's input-bearing components is the blunt instrument, and `GlobalsSetValue` is available as
  a private flag store for your own state. This is more work and it is the only route that gives a
  genuinely exclusive screen.
- **Accept the leak and pick your keybindings to avoid it.** A vanilla button only fires on a
  *click*, so if your panel's own buttons are the only things under the cursor in the regions you
  care about, the leak has no consequence. Arrow keys already do not reach the pause menu.

### What the mouse-claim flag is good for

It is a reliable **within-your-own-`Gui` input blocker**, which is more than most people realise.
Draw one invisible `GuiButton` covering the screen before your real widgets and every widget you own
is protected from stray hover for the rest of the frame:

```lua
-- own gui: claim the mouse first, then draw the real panel
GuiZSetForNextWidget( gui, 0 )
if GuiButton( gui, BLOCKER_ID, 0, 0, "" ) then end   -- never true; we only want the claim
-- everything below now has the mouse to itself
```

Note this only works because the blocker is on the *same* object as the panel. It is exactly the
mechanism that does not work across objects, which is the whole reason a vanilla overlay leaks.

## Detecting the cursor over a vanilla widget

**Static analysis; the trigger is confirmed in game, the probe is not yet tested.** Needs the
menu `Gui` handle (one visit to Options → Mods per session) and the per-frame
`ModSettingsUpdate` trigger — see
[ui-holes.md](ui-holes.md#using-the-handle-outside-the-mods-tab).

The menu `Gui` has one "mouse claimed" byte per frame. The vanilla button, image-button and
text-input builders set it when they are hovered; the frame start clears it. The game also
records the hovered and clicked widget ids on the object (`+0xd0`, `+0x70`), but no Lua call
reads them back.

`GuiImage` reads the claim byte in its own hover test and never sets it
(`00820bf0:109`: hovered only if the byte is clear, the widget is interactive and the cursor is
inside). So a transparent `GuiImage` covering the screen, drawn on the menu `Gui` *after* the
menu has drawn, reports **not hovered exactly when a vanilla interactive widget has the
mouse**. On a paused frame the order is `OnPausePreUpdate` → menu draw → `ModSettingsUpdate(2)`,
so the probe belongs in `ModSettingsUpdate`:

```lua
-- settings.lua
local menu_gui
function ModSettingsGui(gui, in_main_menu)
  menu_gui = gui
  -- ... your settings ...
end

function ModSettingsUpdate(init_scope)
  if init_scope ~= 2 or menu_gui == nil then return end
  if GlobalsGetValue("yourmod.paused", "0") ~= "1" then return end
  local w, h = GuiGetScreenDimensions(menu_gui)
  local s = math.max(w, h)                  -- the hit box uses `scale` for both axes
  GuiImage(menu_gui, 0x7f00, 0, 0, "mods/yourmod/files/px.png", 0, s, s)   -- 1x1 PNG, alpha 0
  local _, _, hovered = GuiGetPreviousWidgetInfo(menu_gui)
  GlobalsSetValue("yourmod.vanilla_hover", hovered and "0" or "1")
end
```

```lua
-- init.lua
function OnPausedChanged(is_paused, is_inventory_pause)
  GlobalsSetValue("yourmod.paused", (is_paused and not is_inventory_pause) and "1" or "0")
end
function OnPausePreUpdate()
  ModSettingSet("yourmod.tick", GameGetFrameNum())   -- runs ModSettingsUpdate after the menu
end
```

`yourmod.vanilla_hover` then lags the menu by one frame, because `init.lua` reads it in the next
`OnPausePreUpdate`. A vanilla click is "hover was set" plus the mouse button going down on that
frame: a click registers on press, while the pointer is inside the widget.

What it does not tell you: *which* widget. Combine it with the mouse position and the screen you
are tracking (above) if you need that. Only paused frames are meaningful: without a menu draw
the byte keeps the last menu frame's value. Each trigger also rewrites `mod_settings.bin` and runs
every mod's `ModSettingsUpdate(2)`; to limit that, trigger only on frames where the mouse moved or
a button went down.

## Gameplay HUD state without any `gui` handle

The inventory HUD keeps its state on the player's `InventoryGuiComponent`, and the component
getter reads all of it except the `gui` pointer. Offsets from the getter (`0x00430af0`):

| field | offset | what the code shows |
|---|---|---|
| `mActive` | `+0x60` | the inventory is open. Set and toggled by the HUD updater (`00b7d8d0:315`, `:508`); gates the full inventory draw (`:546`, `:753`). **Statically confirmed.** |
| `mBackgroundOverlayAlpha` | `+0x68` | fades toward its target every frame in the same updater (`:767-769`) — a smooth "inventory opening/closing" value |
| `mLastFrameActionsVisible` | `+0x58` | compared with the frame counter before the spell row is drawn (`:649`); zeroed in `00b8c200` |
| `mLastFrameInteracted` | `+0x54` | writer not located; untested |
| `mAlpha` | `+0x64` | not traced |
| `imgui` | `+0x50` | the HUD's own `Gui` object. **Not readable**: the getter routes it to a helper that logs `field has unsupported type: imgui - returning 'nil'` (`0x0042cb20`) |

`ControlsComponent` carries both `mButtonDownLeftClick` (its in-source note: "Ignores gamepad,
if mouse is pressed this will be true") and `mButtonDownFire`. If the game withholds fire while
the mouse is used on the HUD, the two will disagree on those frames. That is untested.

## What stays closed

**Blocking input to a screen you did not draw.** This is the one hard no on the page: the claim is
per-`gui` and there is no consume API, so a vanilla screen underneath your overlay keeps working.

**Swallowing Escape.** The menu-stack key handling is in C++, above every `Gui*` call. Nothing in
the modding API sits above it.

**New tabs and new top-level menu entries.** The options screen's tab list is a fixed C++ sequence
of literals (`0x35a9`–`0x361b` for the options rows, `0x3667`–`0x3673` for the main menu). There is
no API to add one, and no data file that lists them. A mod that wanted its own settings *category
in the pause menu itself* has to draw its own screen.

**The pause menu's own rows.** Resume / Options / Mods / Quit are C++, drawn by the menu runner
with pre-baked positions. You cannot insert a row, and you cannot read their rectangles: the
previous-widget record is only ever written by Lua wrappers, so it never holds a game widget's
geometry in the first place.

**Pushing onto the menu stack.** The stack (`DAT_012076bc` .. `DAT_012076c0`) is a vector of C++
function pointers, pushed with `FUN_008a5dd0`. A mod cannot put a Lua screen on it - the push
helper has **no callers in the `0x007d`-`0x007e` Lua wrapper band at all**; all 8 call sites are
menu screens, installing 12 fixed C++ screens.

This matters more than it looks, because a non-empty stack is the game's own screen-suppression
mechanism: `0x006e51e0:145-148` and `0x006e3410:91` both bail out early if
the stack is not empty, so pushing a screen makes the game-over menu and base pause menu **draw
nothing**. It works, and it is unreachable from Lua. Full map in [ui-holes.md](ui-holes.md).

**The progress menu.** Not reachable and not detectable. This is the screen the "feed it an empty
string" trick works on, and there is no predicate for it — you can perturb it but not observe it.

**Screen detection in general.** The three signals above cover paused, front-end-vs-pause, and
inventory. Anything else needs a different approach.

### Detecting an arbitrary screen

Since there is no screen predicate, the general technique is to detect the screen's *side effects*
instead. A screen that draws widgets leaves traces you can read:

```lua
-- no screen predicate exists, but the engine's own input state is observable
function OnWorldPostUpdate(frame, is_paused_screen)
    if is_paused_screen then
        -- you know a menu is up; combine with GameIsInventoryOpen()
        -- and your own OnPausedChanged bookkeeping to narrow it down
    end
end
```

`OnWorldPostUpdate`'s second argument tells you a paused screen is up at all. `OnPausedChanged`'s
`is_inventory_pause` separates the inventory. What remains — options, controls, world select,
progress — you can only infer, and only by the fact that `ModSettingsGui` is running (options) or
not.

## The cursed option

Not recommended, listed for completeness: a mod with `request_no_api_restrictions="1"` can load a
DLL and call any exported function, including the ones that read the menu-stack globals directly.
That gives you the real screen id and the real widget rectangles. It is also how you get a
configuration that silently stops working on the next update, and it is why none of the above is
written that way — everything above is a supported or data-file route.

## Reimplementation notes

For a wrapper that reproduces the pause menu: the one thing worth stealing from this page is that
the engine keeps the previous widget's geometry **on the `gui` object**, not in a global. That is
why a wrapper must too — a global makes two `Gui` objects in one frame clobber each other, which
is the single most common source of cross-mod interference in the real game. See
[gui-cookbook.md](gui-cookbook.md).
