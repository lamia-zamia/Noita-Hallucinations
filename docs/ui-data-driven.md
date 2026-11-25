# Data-driven UI: changing the vanilla interface without touching its code

Companion to [ui-modding.md](ui-modding.md) and [ui-modding-2.md](ui-modding-2.md). Those pages
cover drawing your own UI and getting into the pause menu. This one covers a different question:
**which parts of the vanilla interface are themselves driven by data a mod can supply?**

The short answer is that the vanilla UI is not purely code: several vanilla screens read components, component fields, XML files and directory
listings at runtime, and a mod can write all of them.

## The map

| area | data-driven? | what you supply |
|---|---|---|
| **Item / wand tooltips** | **yes** | `ItemComponent.ui_sprite`, `ui_description`, `ui_name` |
| **HUD status & perk icons** | **yes** | `UIIconComponent` on a child entity |
| **Grid containers** (arbitrary grid anywhere on screen) | **unverified** | `InventoryComponent` has UI-related fields, but its rendering path is not established — see below |
| **Progress-menu enemy grid** | **add and reorder** | icon files and `_list.txt` in `data/ui_gfx/animal_icons/` |
| **Perk "spent" state on the effects panel** | **yes** | sprite path + `$perk_*` strings |
| **Wand stats card values** | **yes** | the wand's own components |
| **Wand stats card *rows*** | no | 27 hardcoded label+icon triples |
| **Wand quick-bar order** | **limited / unresolved** | the renderer traverses `inventory_quick` children and also reads `ItemComponent.inventory_slot` in a slot-layout path; the effective order source is unresolved |
| **Every sprite and font the UI draws** | **yes** | replace the file in your mod's VFS |
| **Pause menu rows, options tabs** | no | hardcoded C++ sequences |

---

## An unverified lead: `InventoryComponent`

Noita has an `InventoryComponent` with five UI-related fields that describe a possible grid
container. It appears in **zero** vanilla entity XML files. Whether the game actually renders this
component as a grid, and which entities can use it, remains unverified.

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

The constructor and field-access methods at `0x0060cda0`–`0x00610c40` establish the component's
defaults and expose these fields through its component interface. The constructor's default sprite
is `data/ui_gfx/inventory/inventory_box.png`. This code does not establish how a renderer consumes
the fields or whether it draws the component for an arbitrary entity, so treat the example as an
experiment rather than a confirmed way to add a grid.

The field types are confirmed independently: `Vec2` for the three vector fields in a mod's own
reflection table, and `InventoryComponent` appears in the community `component-explorer` mod's
component list with its fields enumerated.

**Caveat, and it is not small:** this component is separate from the player's wand inventory. The wand row
iterates a C++-owned vector (`0x00b788e0:111-118`) reached through a singleton, and each
element goes through typed getters, not a component scan. `InventoryComponent` cannot add a wand
slot. At `0x0060cda0`–`0x00610c40`, the executable defines the component and its property and
serialization methods; those methods do not establish a UI draw/update path or which entities it
would display. Treat "it draws for any entity" as untested.

## HUD icons: `UIIconComponent`

The best-documented lever in the API, and worth restating precisely, because the boundary is
sharper than "you can add an icon".

Six fields, all writable at runtime with `ComponentSetValue2` / `EntityAddComponent2`:

```
icon_sprite_file   name   description   display_in_hud   display_above_head   is_perk
```

**Consumed directly per icon:** the picture, the name and the description. Visibility fields are
writable too, but the native HUD consumers have the caveat below.

**HUD visibility caveat:** the native status visitor (`0x00b7c8f0`) and perk collector (`0x00b7cac0`)
do not test `display_in_hud`; neither do their component-buffer lookup helpers. Do not rely on
setting this field false to suppress these HUD rows. Visibility on other surfaces is a separate
consumer question. Moving the native right column avoids changing component state; see
[the hiding recipe](ui-modding.md#share-one-gui-library-across-mods).

Both visitors walk the player and its descendants in root-first depth-first order, retaining
child order (`0x0056dda0`). Each entity lookup supplies one `UIIconComponent`. Status icons are
handled individually; perks are grouped by exact `name`, keeping the first component's artwork
and description and counting later matches for the duplicate label.

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

- **`is_perk` is used both ways in the wild.** Apotheosis, Hydroxide and copis_things set it to
  `1` for perk-style icons, while new_enemies sets `is_perk="0"` on its plain status badges.
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

### Adding and revealing an enemy tile

Progress → Enemies builds its grid from two sources in `0x006dfe40`.

1. `data/ui_gfx/animal_icons/_list.txt` seeds the grid in file order. Each line is converted to
   `data/ui_gfx/animal_icons/<id>.png`. A list entry can create a tile even if its image is missing.
2. The game lists the base icon directory and the enabled mods' icon directories. It sorts those
   paths, skips `_list.txt` itself, and appends paths absent from the grid so far. The comparison
   uses the full path. The grid takes the filename without its last extension as the ID and
   skips entries whose ID is empty or all whitespace. Use PNGs for usable icons.

For example, ship `mods/mymod/data/ui_gfx/animal_icons/my_enemy.png` for an entity loaded from
`mods/mymod/files/creatures/my_enemy.xml`. The ID `my_enemy` comes from the filename in both cases.
Add the bare ID to `_list.txt` only if you want to place it at a specific position. The entity's
directory, `name` attribute and tags play no role in creating a tile. The revealed tooltip tries
`$animal_my_enemy` in the translations, then the `name` from `data/entities/animals/my_enemy.xml`
if that file exists. An entity loaded from a mod-specific directory can use the translation key
to supply the tooltip name.

In normal play the tile shows `???` until the game has recorded at least one player kill under
that ID. If the entity's `GameStatsComponent` sets `dont_do_logplayerkill="1"`, call
`StatsLogPlayerKill(entity_id)` from your own death logic to record the kill. The global run flag
`no_progress_flags_animal` suppresses enemy kill-counter updates and automatic new markers while
set. `GameAddFlagRun("new_kill_my_enemy")` sets the new marker alone.

The draw loop skips `turret_left`. The `turret_right` tooltip adds the left turret's counts once
the right tile is revealed.

### Removing a vanilla enemy tile

Remove both sources for the ID. For example, remove `boss_dragon` from the effective `_list.txt`
and remove `boss_dragon.png` from every icon directory the game enumerates. If the icon remains,
the tile reappears among the appended entries.

Ordinary Lua and asset overrides can replace the list or the image. The base directory listing
still contains the vanilla PNG because it comes from the mounted `data/data.wak`. For a local
loose-data copy, run `noita.exe -wizard_unpak` or `noita_dev.exe -wizard_unpak` from the copied
game directory. Then move `data/data.wak` aside and remove the line and the loose PNG. With a
packed install, the archive's icon directory must omit the PNG. An enabled mod can add the tile
back by providing either source.

### Hiding or reusing a tile in a mod

To hide the revealed portrait, ship a valid transparent PNG at the same mod-relative path, for
example `mods/mymod/data/ui_gfx/animal_icons/boss_dragon.png`. The native grid still draws the
frame and keeps the slot, tooltip and counts. An undiscovered entry uses the shared
`data/ui_gfx/progress_menu/icon_unknown.png`. A corrupt or zero-byte PNG cannot remove the slot
either. The enemy list is already built when the renderer tries to load the image, and the grid
advances using a fixed column count and cell spacing. `ModImageMakeEditable` and
`ModImageSetPixel` affect image content through the same separation from the directory listing.

If a mod replaces a vanilla creature, it can reuse the vanilla tile. For a replacement for
`boss_dragon`, load the new creature from an XML named `boss_dragon.xml`, override
`data/ui_gfx/animal_icons/boss_dragon.png`, and update the `animal_boss_dragon` translation.
The replacement inherits kill history under that ID. Use a distinct ID when the old and new
creatures need separate stats.

### New-entry markers

To check whether a particular entry is marked new, use the matching run flag:

```lua
local action_is_new = GameHasFlagRun("new_action_" .. action_id)
local perk_is_new = GameHasFlagRun("new_perk_picked_" .. perk_id)
local enemy_is_new = GameHasFlagRun("new_kill_" .. enemy_id)
```

The pause-menu Progress row shows `$menu_progress_new` with a positive count and `$menu_progress`
when the count is zero. It checks perk and action flags during the draw and caches the enemy count
from a separate directory scan when its widget is created. The Progress screen also reads the
flags for its per-entry markers. Opening Progress leaves those flags set. Clear them with
`GameRemoveFlagRun`, or check the IDs you care about with `GameHasFlagRun`.

The screen checks `PERK_PICKED_<id>` for discovery and the matching `new_perk_picked_<id>` run flag
for the new marker, so `GameAddFlagRun` can mark an *existing* vanilla perk or spell as newly
acquired. You cannot add a new perk *cell* that way — that list is a compiled vector.

## The wand stats card: values yes, rows no

The card is a fixed list of stat rows, and each row is baked into one function: a component field
name, a `$inventory_*` label, for some rows a `$inventory_*_tooltip` (the function references nine),
and an `icon_<field>.png` path built from a hardcoded prefix. The same renderer is used by the
expanded wand inventory and the new-wand chooser, so the row set follows the wand across both screens.
The item-pickup overlay also calls this renderer for the current world-item candidate. The pickup
function receives separate picker and item entities: it reads `InventoryGuiComponent` from the picker,
then positions the card at the item's projected screen location. The picker component must have a
nonzero `imgui` field and a clear `mActive` field. The chooser also uses `imgui` as its GUI object and
starts a GUI frame on it before drawing. The component's reflected C++ type for `imgui` is
`ImGuiContext*`; `mActive` is a `bool`.
The game's `InventoryGuiSystem` updates these components by iterating its component buffer and
calling the inventory renderer. The renderer starts a GUI frame using that `ImGuiContext*` and
`mActive`; when active, it draws the inventory background and the rest of the inventory UI. Its
open/close path changes `mActive`, which is also why the pickup detail card is suppressed while the
inventory is open.
The panel background sizes itself to the content: it starts at 96 UI units wide, grows to fit
measured text, and adds one measured text-line step for each applicable row. Rows can disappear when
their component values do not call for them, but the game has no data-driven way to add a new row.
The always-cast spell icons are hoverable too; hovering one can open a nested detail card for that
action.

- **You can change any value** — it reads the wand's real components (`gunaction_config`,
  `AbilityComponent`, `gun_config`). Alter a wand's `damage_projectile` and the card shows it.
- **You cannot add a row.** There is no iteration over the component's field map; the row set is
  the literal list. A stat the game does not know about is invisible on the card.

One trap if you go here: the card reads *runtime* field names while the XML exposes *different*
attribute names — `gun_capacity` on the card is `deck_capacity` in the XML, `damage_melee` is
`damage_melee_add`. Both spellings are in the binary.

Five rows — `damage_slice`, `damage_drill`, `damage_curse`, `damage_holy`,
`damage_healing` — **never appear as XML attributes anywhere in the vanilla tree**, so there is no
shipped example of setting them.

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
  the nine `colors_*.png` bar variants.
- **Inventory**: `data/ui_gfx/inventory/{background,inventory_box,inventory_colors,icon_info,
  icon_danger,icon_warning,hover_info_empty_slot}.png`, the `item_bg_*.png` action-type
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

- **Wand quick-bar order.** `FUN_00b7d8d0` finds the player's inventory buckets. Capacity reads in
  `FUN_00b5d940` identify `DAT_012224f4` as the quick bucket (`quick_inventory_slots`) and
  `DAT_012224f8` as the full grid (`full_inventory_slots_x * full_inventory_slots_y`). The compact
  HUD path traverses the quick bucket's `+0x90` child vector, and `EntityGetAllChildren(quick)`
  returns that same sequence (`0x007935b0`). `FUN_00b724c0`, called by the HUD renderer, also reads
  the item's `ItemComponent.inventory_slot` field at `+0xd8` in a slot-layout path. The available
  evidence does not establish which state determines the final visible order. In reported runtime
  tests, reparenting changed the `EntityGetAllChildren` order but left the visible wand order
  unchanged. Detaching two wands, exchanging their `inventory_slot` coordinates, and reattaching
  them changed immediate component readback but did not change the HUD; later reads returned the
  original coordinates. Direct writes while carried also read old or reverted. A further test dropped
  two wands, exchanged their preferred coordinates, and called `GamePickUpInventoryItem`; immediate
  readback showed the exchange, but the HUD stayed unchanged and later invocations again began with
  the original coordinates. A follow-up changed the values after the engine update; the next
  `OnWorldPreUpdate` sample still showed the swapped coordinates and child order. A two-stage
  drop/write/pickup sequence then kept each requested wand-slot mapping for 120 updates in both
  directions, without setting `mForceRefresh`; the second invocation began from the first result and
  toggled the pair back. The child sequence can change independently while the target wand
  coordinates still match. The non-wand child with an `ItemComponent` is returned by
  `GameGetAllInventoryItems`, and its slot moves between `(0,0)` and `(1,0)` in the reported runs.
  These results establish that Lua can preserve the component slot mapping through repeated engine
  updates using the staged sequence. Changing inventory capacity at runtime is not established.
- **Wand stats card rows.** 27 compile-time triples.
- **Options-screen tabs and the main menu.** Fixed literal widget-id sequences.
- **Pause menu rows.** C++, pre-baked positions.
- **HUD icon layout.** Proportional to count, with two compiled constants for hover scale and alpha.

The pattern is that the vanilla UI is data-driven for much of its _content_ and hardcoded for most
of its _structure_. The quick wand row has a mixed source: it traverses `inventory_quick` children
and also passes item slot coordinates through a layout path. Which state controls final placement,
and which mutation sequence reliably updates it in game, remain unresolved.
Content means sprites, labels, values, names, descriptions and item lists — and many of those are
yours to change. Structure means which rows exist and where they appear; most of that is fixed.

## How to verify any of this yourself

The first four checks exercise established data-driven paths. The `InventoryComponent` check is an
unresolved experiment:

- Drop a replacement PNG at a `data/ui_gfx/hud/*.png` path in your mod and see whether the HUD
  changes. This works; it is the cheapest confirmation in the list.
- Add a `UIIconComponent` to a child of the player and watch the icon column.
- Set `ItemComponent.ui_description` on a spawned item and hover it.
- Put a file in `data/ui_gfx/animal_icons/` and open the progress menu.
- Try `InventoryComponent` with an explicit `ui_position_on_screen` and see whether a grid appears.
  No working result has been verified yet.
