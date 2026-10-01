# Changing Noita's UI: what actually works

This is the practical page. Everything else in this reference describes what the UI *is*; this
describes what a mod can *do* about it, based on what the API exposes and on what mods in the
wild actually achieve.

## The one-paragraph answer

The game's own UI is **not reachable through the modding API**. It is C++, it is closed, and
there is no getter for any of its state — no way to read the health bar's rectangle, query the
selected inventory slot, add a settings tab, or move a widget. What *is* reachable is
everything around it: you can draw your own UI with the same `Gui*` API the game uses, you can
own the mod-settings page, you can rewrite the game's text and fonts and layout tunables as data
files, you can patch vanilla Lua scripts, and with FFI you can byte-patch the executable.
Realistically you **layer over** the vanilla UI or **replace its content**, never its code.

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
function ModSettingsGui(gui, im_id, in_main_menu)
    GuiLayoutBeginHorizontal(gui, im_id)
    GuiText(gui, 0, 0, "hello")
    GuiLayoutEnd(gui)
end
```

This is the sanctioned UI surface and the only one where the game passes you a real `gui`. Note
the third argument: `in_main_menu` tells you whether you are being drawn in the front-end menu
or the in-game pause menu, because those are separate screens and the game uses the same hook
for both.

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
and call it what it is. Same trick for blocking input — an invisible 50×50 scroll container
under the cursor with the always-clickable option.

### Rewrite the game's text, fonts and tunables as data

All three are plain data files and all three are writable.

- **`data/translations/common.csv`** — every string in the game. Later duplicate keys win, so
  **appending** your own rows overrides the game's. This is how you relabel menus, items,
  status effects and death screens wholesale. A mod that wants to silence one message appends
  one empty row; that is the most surgical UI change in existence.
- **`data/translations/<lang>.xml`** — the only discovered layout knob for the native wand
  panel: `ui_action_info_offset2`, `ui_wand_info_offset1`, `ui_wand_info_offset2` are pixel
  offsets, and they are **per language file**, so a fix for one language does not apply to
  others.
- **`data/fonts/font_pixel.xml`** — parse it, rewrite the colour channels, and write a new font
  into your mod's vfs. You cannot restyle the vanilla font in place usefully, but you can ship
  recoloured copies.
- **`magic_numbers.xml`** — the UI's pixel constants, via `ModMagicNumbersFileAdd`. The stat
  bar origins, spacing, health-bar pitch, low-HP threshold, inventory icon size, game-over panel
  percentages and the main-menu background scroll are all here. This is the closest thing to a
  supported way to re-lay-out the game's own UI.

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

## What is closed, and what people do instead

### No vanilla UI state is readable

There is no API to get the health bar's rect, the wand row's rect, the hovered slot, or the HUD
layout. Every mod that needs to sit next to vanilla UI **hardcodes pixels**, and every one of
them has fudge constants with apologetic comments:

```lua
local offset = 2 -- why, nolla
```

This is the root difficulty and it is worth being blunt about: you cannot anchor to the vanilla
HUD. You can only guess where it is and hope.

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

The hardest single number to reproduce is the health bar's Y anchor, which comes from a compiled
tunable whose value has not been recovered. If you are matching the HUD exactly, that is the
thing to chase.