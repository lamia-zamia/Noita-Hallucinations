# Changing Noita's UI: what actually works

This is the practical page. Everything else in this reference describes what the UI *is*; this
describes what a mod can *do* about it, based on what the API exposes and on what mods in the
wild actually achieve.

## The one-paragraph answer

The game's UI is C++ and its widget state is not exposed: Lua has no getter for the health-bar
rectangle or selected wand slot, and no API to insert or reorder native widgets. That does not make
the UI untouchable. Data tunables can move or scale several HUD groups and menu panels, component
fields control dynamic HUD icons, and files control many sprites and labels. A mod can hide much of
the native HUD, draw a replacement with the same `Gui*` API, and keep the game's state underneath.
The remaining gaps are native interaction behavior and UI structure that has no data control.

## What is genuinely open

### Change game state and let the vanilla UI render it

The cheapest and most robust technique, and it is what most mods do. You touch entities and
components; the HUD shows the result.

```lua
-- max health, and the vanilla bar follows
local damage_model = EntityGetFirstComponent(player, "DamageModelComponent")
ComponentSetValue2(damage_model, "max_hp", 40)
```

You can also create a **native HUD icon from Lua alone**, with no UI code at all, by adding a
`UIIconComponent` to an entity:

```lua
EntityAddComponent2(effect_entity, "UIIconComponent", {
    icon_sprite_file = "mods/mymod/files/gfx/icons/tinker.png",
    name = "$perk_edit_wands_everywhere",
    description = "Tinker",
    is_perk = false,
})
```

That appears as a real entry in the vanilla HUD status-icon column, with hover text. It is the
most under-used capability in the API.

### Own the mod-settings page

Every mod gets a settings page, and a custom UI function gets the game's own `gui` object:

```lua
ModSettingsUpdate("my_mod.category", "my_mod.setting_name", "My setting")
ModSettingsGuiCount(1)
function ModSettingsGui(gui, in_main_menu)
    GuiLayoutBeginHorizontal(gui, im_id)
    GuiText(gui, 0, 0, "hello")
    GuiLayoutEnd(gui)
end
```

This is the sanctioned UI surface and the only one where the game passes you a real `gui`. The game
passes exactly **two** arguments: the `gui` object and `in_main_menu`. Note that `in_main_menu`
tells you whether you are being drawn in the front-end menu or the in-game pause menu, because
those are separate screens and the game uses the same hook for both.

Because that `gui` is the *options screen's own object*, you can draw a full-screen panel
directly onto it from inside `ModSettingsGui` and get correct compositing with the rest of the
screen. You still cannot read any vanilla widget's geometry from it - the previous-widget record
is only written by Lua wrappers - and you cannot move or re-layout the screen's own rows. The
generic settings renderer that draws your rows is a shipped Lua file,
`data/scripts/lib/mod_settings.lua`, which a mod can append to or replace. Both are covered in
[ui-modding-2.md](ui-modding-2.md).

### Draw your own overlay

```lua
function OnWorldPostUpdate(game_globals_frame_num, is_paused_screen)
    gui:start_frame()
    -- draw anything
    gui:end_frame()
end
```

Your own `gui` object, on your own id space, at whatever z you choose. This is how every
non-trivial UI mod works: an experience bar, a level-up menu, a player list, a shared item
bank, a debug overlay. You are limited to what the `Gui*` API offers, which is a lot more than
people assume — text, images, nine-pieces, buttons, image buttons, sliders, text inputs,
tooltips, scroll containers, auto-boxes, layers, animations.

**But there is no scissor rectangle**, so clipping is a hack: open a transparent
`GuiBeginScrollContainer` and the engine will clip subsequent widgets to it. Real mods do this
and call it what it is.

**Blocking input on your own `Gui` works, and the mechanism is specific.** Draw a `GuiButton`
covering the full screen as the *first* thing you draw in the frame; it claims the mouse and
every later widget on that object sees the claim and refuses hover. Three details matter:

- The claim must come from an **enabled** button. The button builder returns a disabled/invisible
  flag, and a widget with that flag set skips claiming the mouse entirely
  (`0x008245d0:160`) even though it is still hit-tested.
- A `GuiBeginScrollContainer` does **not** claim the mouse. It never touches the claim byte at
  all (`0x0081f870`, zero references) - it is a clip and scroll region, nothing more. The
  "invisible scroll container blocks input" recipe does not work as such; use a button.
- The claim is cleared only by that object's own `GuiStartFrame`, so it lasts the whole frame and
  you cannot release it early.

**All of this is confined to your own `Gui` object.** The claim byte lives on the object, so it
cannot protect you from a vanilla menu drawn on a different one, and nothing in the API consumes
a click outright. Drawing a menu over the pause menu therefore leaks clicks to whatever is
underneath; the fix is to draw a screen rather than an overlay.
[ui-modding-2.md](ui-modding-2.md) has the details and the three workable designs.

### Rewrite the game's text, fonts and tunables as data

All three are plain data files and all three are writable.

- **`data/translations/common.csv`** - the game's string table, loaded at startup from this
  exact path (`FUN_0084a700`, which also reads `common_dev.csv`). Rows are held in a map keyed by
  the text after the `$`, and a row whose key already exists **overwrites** the earlier one, so on
  duplicate keys the *later* row wins. Relabelling is therefore appending rows after the vanilla
  ones (`ModTextFileGetContent` / `ModTextFileSetContent` on the vanilla path), and a mod wanting
  to silence one message sets that row's value to an empty string. This is the most surgical UI
  change in existence.

  Note the key format. GUI labels go through `FUN_0084b3a0`, which checks for a leading `$`
  (`0x24`): a `$`-prefixed label is looked up in this table, and **any other label is used
  verbatim**. So `GuiText(gui, 0, 0, "Play")` shows `Play`; only `$`-prefixed text is translated.
  `GuiOptionsAdd`-style option names are not the same namespace as card names (`action_*`), which
  are looked up elsewhere. Details in [ui-holes.md](ui-holes.md).

Those are the *tunable* levers. The larger set — which components and files the UI reads at
runtime, and therefore what a mod can replace outright — is in
[ui-data-driven.md](ui-data-driven.md). In short the rule is: the vanilla UI is data-driven for
its **content** (sprites, names, descriptions, values, item lists) and hardcoded for its
**structure** (which rows exist, in what order, where).

### Patch vanilla Lua

`ModLuaFileAppend` on any `data/scripts/...` file. Appends are robust; string-manipulation
patches that insert a line before a matched line are not — they fail silently when the upstream
text changes, and there is no verification step. In-place replacement of a vanilla global
function is the standard idiom and is as robust as appends.

### Generate sprites at runtime

`ModImageMakeEditable` + `ModImageSetPixel` + `ModTextFileSetContent` for the companion `.xml`.
Widely used to build icon atlases.

**`SpriteSet*` is used by essentially no mod.** Sprite animation is reachable only indirectly,
through `SpriteComponent` fields or an XML `EntityLoad`.

### Load a native UI framework

Dear ImGui can be injected into the game process and exposed to Lua, with the game's own event
pump and GL swap handed to it. Requires `request_no_api_restrictions="1"` and a real DLL.
Useful for developer tooling, not for replacing game screens.

### Share one GUI library across mods

Noita gives each mod its own Lua state. A callback, table, GUI handle or hook closure registered
by one mod is not shared with another. Copying a renderer into every mod and letting each copy
draw would therefore create several independent renderers, not one shared instance. See the
[Lua execution model](lua-execution.md) and [virtual file system](vfs.md).

A shared library can instead be appended to `data/scripts/init.lua` from each participating mod's
early `ModSettingsUpdate` callback. The append runs as part of the target script, so library copies
and hook-registration files appended there can use one game-script Lua state. The settings
callback's own state remains separate; use it to register the append, not to hold GUI handles or
hook closures. The `biome_modifiers_patch` pattern demonstrates registering this append before the
game's init script loads.

The test library implements this pattern by inspecting active mods' version headers, choosing the
highest `major.minor.patch`, and loading only that implementation into the shared state. Equal
versions need identical contents; a stable mod-ID tie-break makes selection deterministic. Other
mods can append hook registrations after the library and add behavior through its shared hook table
without a separate host mod or settings page. The version election and callback/draw ordering in
this implementation have not yet been runtime-tested.

This is a composability path for Lua code, not access to the game's native widget tree. For
recreation, supported tunables can move the stat bars, quickbar, and expanded inventory out of the
way; `UIIconComponent.display_in_hud` can suppress the dynamic status/perk row. The renderer then
draws replacement content through the GUI API. It still cannot read native widget rectangles,
vanilla tooltip text, or the selected wand slot, and moving a panel does not replace its native
input behavior. See [the screen-by-screen limits](ui-screens.md),
[data-driven UI](ui-data-driven.md), and [the documented modding surfaces](ui-modding-2.md).

For example, a mod can move the HUD cluster off-screen through its existing magic-number file:

```xml
<MagicNumbers
  UI_BARS_POS_Y="10000"
  UI_BARS2_OFFSET_X="10000"
  UI_FULL_INVENTORY_OFFSET_X="10000"
  UI_QUICKBAR_OFFSET_Y="10000"
/>
```

Register that XML during the mod's normal initialization with `ModMagicNumbersFileAdd`. These are
group offsets, not a blanket hide flag; the exact result depends on which screen is open and should
be checked in-game. For the dynamic icon row, set `display_in_hud` false on the relevant
`UIIconComponent`s while the replacement is active, then restore their saved values when handing
control back.

## What is closed, and what people do instead

### No vanilla UI state is readable

There is no API to get the health bar's rect, the wand row's rect, the hovered slot, or the HUD
layout. Every mod that needs to sit next to vanilla UI **hardcodes pixels**, and every one of
them has fudge constants with apologetic comments:

```lua
local offset = 2 -- why, nolla
```

There is no runtime geometry getter for anchoring to an individual vanilla widget. Group tunables
can move a supported cluster; precise per-widget layout still has to be reconstructed or measured.

The options screen is the one place you are handed a `gui` that is not yours, so it is the only
place you can draw outside your own widget tree. It does not help with geometry: the HUD, the
pause menu and the progress menu all draw widgets that never enter the previous-widget record. See
[ui-modding-2.md](ui-modding-2.md).

### No world-to-screen projection

World coordinates cannot be projected into GUI space. Mods hand-roll it from the virtual
resolution magic numbers and the camera position, and the best-documented attempt in the wild
admits:

> `-- Bunch of weird constants in here but it seems to improve the accuracy of the conversion.`

with `-2.8`, `-0.5` and a `virt_y * 0.99` fudge. A radar can only clamp direction arrows to the
screen edge, because there is no projection to do it properly with.

### No input enumeration

No API lists keycodes. Every mod scrapes `data/scripts/debug/keycodes.lua` with a regex and
renames the fields. Gamepad support is detectable only after the fact, from a component field
saying whether gamepad controls were used last frame. An on-screen keyboard is hardcoded to
ASCII because there is no way to ask what keys exist.

### No real scroll container, checkbox, slider or dropdown

These are all hand-built by modders. The vanilla scroll container is unstyleable and
unmeasurable, so it gets used as a scissor. Checkboxes are a nine-piece plus a coloured `V`/`X`.
And the game's own comment on the option system is worth reading as a warning:

> `-- These options are exposed to the modding API due to public demand but are completely unsupported.`
> `-- You just have to live with the fact that the gui library exists mainly to support the game, and we have limited time to work on it.`

### The main menu, pause menu, options screen, world select and game-over screen are untouchable in structure

Pure C++. Three things you can do: patch their strings in the translation files; draw an
unrelated overlay on top; or byte-patch the executable.

Note what mods do *not* do: a large mod that wanted a level-up screen and a meta-progression
screen **reinvented both from scratch** rather than trying to modify the vanilla ones. That is
the realistic ceiling.

### The HUD icon column cannot be reordered

Adding an icon works (see `UIIconComponent` above). Inserting one at a chosen index does not.
A mod that needs to control placement has to discover its own icons by scanning player children
for a marker component, sort them by a manually assigned priority value, and stack them with a
hardcoded pitch — and it cannot reserve a slot, only count what already exists.

### Inventory slots cannot be added at runtime

The inventory UI reads the entity tree, and there is no API to add or reorder a vanilla
inventory slot. The one mod in the corpus that adds a row does it by **rewriting another mod's
entire source file at load time**, matching literal Lua text with `gsub`. If the upstream mod
reformats one line, it breaks silently.

## Choosing an approach

| goal | approach | robustness |
|---|---|---|
| show a new number on the HUD | add a `UIIconComponent` to an entity | excellent |
| change a HUD value | set the component the HUD reads | excellent |
| add a settings tab for your mod | `ModSettingsGui` | excellent |
| relabel menus, items, statuses | append rows to `common.csv` | excellent |
| re-lay-out the game's own UI | `magic_numbers.xml` via `ModMagicNumbersFileAdd` | good |
| draw your own panel or menu | `Gui*` on your own `gui` in `OnWorldPostUpdate` | good, with the clipping and input caveats |
| recolour text | synthesise a font into your vfs | good |
| add a feature to vanilla behaviour | `ModLuaFileAppend` on a vanilla script | good |
| hover a vanilla widget | overlay an invisible hitbox at hardcoded pixels | fragile |
| insert a row into vanilla's own layout | patch the Lua source text | fragile, fails silently |
| change a game setting | byte-patch the exe at a hardcoded address | breaks on every update |

## If you are writing a wrapper instead of a mod

Two facts from this reference change the design:

1. **The game's UI is the same API you have.** There is no privileged path. Anything a wrapper
   needs to reproduce, it reproduces with the same primitives — and [gui-cookbook.md](gui-cookbook.md)
   has the coordinate maths and formulas.
2. **The widget-id system is global and unhashed-for-you.** The menus pass literal ids
   (`0x35a9`–`0x361b` for the options screen, `0x3525`–`0x3531` for the world-select slots,
   `0x3667`–`0x3673` for the main menu). A wrapper that does not manage ids will **share hover
   and focus state with the game's own buttons**. See [gui-internals.md](gui-internals.md).

The health bars start at `UI_BARS_POS_Y + activePlayerIndex * UI_HEALTHBAR_Y_SPACING`; the shipped
defaults are 20 and 24. The live HUD uses these data values directly. A separate, unreachable
co-op bar routine uses a compiled fraction for its own anchor; that routine does not affect the
visible bar.
