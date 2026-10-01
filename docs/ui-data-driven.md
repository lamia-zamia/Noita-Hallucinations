# Data-driven UI: changing the vanilla interface without touching its code

Companion to [ui-modding.md](ui-modding.md) and [ui-modding-2.md](ui-modding-2.md). Those pages
cover drawing your own UI and getting into the pause menu. This one covers a different question:
**which parts of the vanilla interface are themselves driven by data a mod can supply?**

The short answer is that the separation people assume — "the vanilla UI is code, my UI is data" —
is not true. Several vanilla screens read components, component fields, XML files and directory
listings at runtime, and a mod can write all of them.

## The map

| area | data-driven? | what you supply |
|---|---|---|
| **Item / wand tooltips** | **yes** | `ItemComponent.ui_sprite`, `ui_description`, `ui_name` |
| **HUD status & perk icons** | **yes** | `UIIconComponent` on a child entity |
| **Grid containers** (arbitrary grid anywhere on screen) | **yes** | `InventoryComponent` — see below |
| **Progress-menu enemy grid** | **yes** | files in `data/ui_gfx/animal_icons/` |
| **Perk "spent" state on the effects panel** | **yes** | sprite path + `$perk_*` strings |
| **Wand stats card values** | **yes** | the wand's own components |
| **Wand stats card *rows*** | no | 27 hardcoded label+icon triples |
| **Wand inventory slots** | no | the slot list is a C++ vector |
| **Every sprite and font the UI draws** | **yes** | replace the file in your mod's VFS |
| **Pause menu rows, options tabs** | no | hardcoded C++ sequences |

---

## The best find: `InventoryComponent`

Noita has a generic **grid container** component whose entire layout is four data fields, and the
game's own container UI draws it. It appears in **zero** vanilla entity XML files — it is a live
component with no shipped user — which is exactly why nobody knows about it.

```lua
local e = EntityCreateNew("")
EntityAddComponent( e, "InventoryComponent", {
    ui_container_type    = 0,                      -- UI_CONTAINER_TYPES enum
    ui_container_size    = { 4, 3 },                -- "how many items x*y we can fit in"
    ui_element_size      = { 32, 32 },              -- pixels per cell
    ui_position_on_screen = { 100, 200 },           -- "where do we load this on screen"
    ui_element_sprite    = "mods/mymod/files/gfx/my_box.png",  -- "ui back sprite"
} )
```

Those five field names and their documentation strings are compiled into the executable, read by
the ten functions at `0x0060cda0`–`0x006101xx`, and the drawing code holds the default sprite
`data/ui_gfx/inventory/inventory_box.png`. `ui_position_on_screen` is the key one: **it is an
absolute screen position**, so this is not a slot in someone else's layout — it is a grid you place
yourself, at a coordinate you choose, with a background sprite you choose.

The field types are confirmed independently: `Vec2` for the three vector fields in a mod's own
reflection table, and `InventoryComponent` appears in the community `component-explorer` mod's
component list with its fields enumerated.

**Caveat, and it is not small:** this component is *not* the player's wand inventory. The wand row
iterates a C++-owned vector (`out/decomp/00b788e0.c:111-118`) reached through a singleton, and each
element goes through typed getters, not a component scan. So `InventoryComponent` gives you a *new*
grid; it does not let you add a wand slot. It is also unproven that the container UI runs for an
arbitrary entity rather than only for the specific owners the game creates it on — the decompilation
of the reader functions is missing, so treat "it draws for any entity" as untested.

## HUD icons: `UIIconComponent`

The best-documented lever in the API, and worth restating precisely, because the boundary is
sharper than "you can add an icon".

Six fields, all writable at runtime with `ComponentSetValue2` / `EntityAddComponent2`:

```
icon_sprite_file   name   description   display_in_hud   display_above_head   is_perk
```

**Controllable per icon:** the picture, the name, the description, and visibility on each of the two
surfaces.

**Not controllable:** size, scale, alpha, position, ordering. The HUD lays the row out as
`cellWidth / count`, so a shorter row makes every icon bigger, and the hover scale and hover alpha
are two compiled constants, not fields.

This is the mechanism *perks themselves* use, which is why it is reliable — vanilla `perk.lua` adds a
child entity carrying `UIIconComponent` and parents it to the player, and so does the essence
pickup, fungal shift, greed curse and the streaming integration. **A mod does exactly the same
thing.** `is_perk` chooses which of the two rows the icon lands in.

Runtime mutation works too: at least one mod rewrites `icon_sprite_file` on a live component and
reads it back, so these are not spawn-time-only values.

Two facts worth knowing that are not in any guide:

- **No mod in the corpus sets `is_perk="1"`.** Every one that wants a plain status badge explicitly
  sets `is_perk="0"`. The field demonstrably changes behaviour, and the community has concluded it
  does not want the perk treatment.
- **The overflow popup is a different index into the same vector.** Past a certain count, icons are
  replaced by a single "and N more" entry that opens a popup listing them. Mods that add many icons
  hit this without noticing.

## Item tooltips: `ItemComponent`

The "stand over an item and see what it is" box reads the hovered entity's component handle,
branches on an identified flag, and uses a name string off the component. Two fields do the work,
and the executable documents both:

| field | the executable's own description |
|---|---|
| `ui_sprite` | *"sprite displayed for the item in various UIs. If not empty overrides sprites declared by Ability and ItemAction"* |
| `ui_description` | *"item description displayed in various UIs"* |

So an item's tooltip content is **per-entity data**, not a translation key. Vanilla relies on this:
every perk pickup entity gets `ItemComponent{item_name=…, ui_description=…}` plus a
`UIInfoComponent{name=…}` and a `SpriteComponent`, and the game's own
`change_entity_ingame_name` helper rewrites exactly these four fields at runtime. A mod can rename
and re-describe any item, and can override its icon.

**The exception:** for wands and potions the label is generated in C++ from the item's kind
(`$item_wand`, `$item_potion_fullness`, `$item_potion_empty`), so the translation table is the only
lever there.

There is also a config option, `ui_show_world_hover_info_next_to_mouse`, which switches the label
between a fixed stand-over anchor and following the mouse. It is a real user setting.

## The progress menu's enemy grid is filesystem-driven

This is the cleanest "add UI content by dropping in a file" lever in the game. The progress screen
**lists a VFS directory** to build its enemy grid:

```
data/ui_gfx/animal_icons/     <- enumerated at runtime
data/ui_gfx/animal_icons/_list.txt   <- subtracted, so pre-listed entries are not "new"
```

It takes the difference between what is on disk and what `_list.txt` names, and what is left over
becomes new cells. The label for each is resolved in three steps: try `$animal_<name>`; else look
for `data/entities/animals/<name>.xml` and take its name field; else use the raw file name.

So a mod can add a progress cell by shipping:

```
mods/mymod/data/ui_gfx/animal_icons/my_enemy.png
mods/mymod/data/entities/animals/my_enemy.xml
```

and marking it discovered with `GlobalsSetValue("new_kill_my_enemy", "1")`. Vanilla has 185 of
these icons. **This is the mechanism to reach for if you want to extend a vanilla screen with real
new content** — the one place where the extension point is a directory listing rather than a fixed
table.

Note the same screen reads `PERK_PICKED_<id>` and `new_perk_picked_<id>` from the globals store, so
you can also mark an *existing* vanilla perk or spell as newly acquired. You cannot add a new perk
*cell* that way — that list is a compiled vector.

## The wand stats card: values yes, rows no

The card is 27 stat rows, and each row is a fixed triple baked into one function: a component field
name, a `$inventory_*` label, a `$inventory_*_tooltip`, and an `icon_<field>.png` path built from a
hardcoded prefix.

- **You can change any value** — it reads the wand's real components (`gunaction_config`,
  `AbilityComponent`, `gun_config`). Alter a wand's `damage_projectile` and the card shows it.
- **You cannot add a row.** There is no iteration over the component's field map; the row set is
  the literal list. A stat the game does not know about is invisible on the card.

One trap if you go here: the card reads *runtime* field names while the XML exposes *different*
attribute names — `gun_capacity` on the card is `deck_capacity` in the XML, `damage_slice` is
`damage_slice_add`. Both spellings are in the binary.

Six rows — `damage_melee`, `damage_slice`, `damage_drill`, `damage_curse`, `damage_holy`,
`damage_healing` — **never appear as XML attributes anywhere in the vanilla tree.** They are fed by
a synthesised aggregate, so setting them directly may do nothing.

## Perks are three different systems

Worth separating, because they behave differently and the names are confusing:

1. **The HUD perk/status row** — `UIIconComponent` again. Fully mod-extensible, described above.
2. **The progress screen's perk and spell cells** — a C++-owned vector. Cells are fixed, but their
   "newly acquired" shine is driven by globals you can write.
3. **The perk-effects panel** (`00b015d0`) — a *hardcoded* panel that identifies a perk by
   **comparing its sprite path and its `$perk_*` name string against literals**. So a mod can make
   its own perk render there by shipping a `perk_icons/*.png` and matching translation keys. Only
   the respawn row has a "spent" variant in the shipped game.

Note that `perk_list.lua` is never read by C++. Appending to it gives you a pickup and an effect,
but the C++ only ever sees the three `ui_*` strings, because Lua copies them into a
`UIIconComponent`. That is the whole trick, and it is worth understanding: **the perk system is
data-driven all the way down, but through a component, not through the table.**

## Replacing the pictures themselves

Every asset path the UI draws is a string literal in the identified UI functions, and the mod VFS
shadows `data/`. So replacing the **file** — not the path — changes what the game draws. This is
neither text nor a number, and it is completely underexploited.

- **HUD**: `data/ui_gfx/hud/{health,mana,jetpack,potion,reload,fire_rate_wait,money,orbs}.png` and
  the ten `colors_*.png` bar variants.
- **Inventory**: `data/ui_gfx/inventory/{background,inventory_box,inventory_colors,icon_info,
  icon_danger,icon_warning,hover_info_empty_slot}.png`, nine `item_bg_*.png` action-type
  backgrounds, and 27 `icon_<stat>.png`.
- **Pause menu**: `data/ui_gfx/pause_menu/{noita_logo,help_keyboardmouse,help_gamepad360}.png`.
- **Status column**: `data/ui_gfx/status_indicators/satiation_00..06.png`, `bg_ingestion.png`.
- **Fonts** — and here it is content, not just path: a mod can add *glyphs* to the pixel font, which
  is the only way to get a character the game does not have.
- **Cursors**: `mouse_cursor{,_big}.png`, `keyboard_cursor{,_right}.png`.

A mod can also repaint the health bar by replacing `colors_health_bar.png` with a different image, or
give the wand card a different look by replacing all 27 `icon_*.png`. Nobody does this.

## What stays hardcoded

Being straight about the ceiling, because the list above invites the opposite conclusion:

- **Wand inventory slots.** The slot list is a C++ vector behind a singleton, iterated with typed
  getters. No component, no tag, no child-entity relationship feeds it. You cannot add a wand slot.
- **Wand stats card rows.** 27 compile-time triples.
- **Options-screen tabs and the main menu.** Fixed literal widget-id sequences.
- **Pause menu rows.** C++, pre-baked positions.
- **HUD icon layout.** Proportional to count, with two compiled constants for hover scale and alpha.

The pattern across all of them is the same: **the vanilla UI is data-driven for its _content_ and
hardcoded for its _structure_.** Content means sprites, labels, values, names, descriptions and
item lists — and every one of those is yours to change. Structure means which rows exist, in what
order, at what position — and none of it is.

## How to verify any of this yourself

Everything above is checkable in ten seconds without the decompiler:

- Drop a replacement PNG at a `data/ui_gfx/hud/*.png` path in your mod and see whether the HUD
  changes. This works; it is the cheapest confirmation in the list.
- Add a `UIIconComponent` to a child of the player and watch the icon column.
- Set `ItemComponent.ui_description` on a spawned item and hover it.
- Put a file in `data/ui_gfx/animal_icons/` and open the progress menu.
- Try `InventoryComponent` with an explicit `ui_position_on_screen` and see whether a grid appears.
  **This one is untested** and is the most interesting thing to try.