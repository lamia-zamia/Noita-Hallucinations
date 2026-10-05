# The game's own UI: where it is and what is in it

Everything on this page is about the interface **Noita draws itself** — the main menu, the
options screen, the HUD, the wand inventory, the progress screen. None of it is the Lua
`Gui*` API, which is documented separately in [gui-cookbook.md](gui-cookbook.md).

Addresses are from one Steam build of the 32-bit `noita.exe` and will move on update.

## Where the UI lives

The game's UI is not one subsystem. It is spread across the executable in four address
bands, and knowing which band a function is in tells you what screen you are looking at:

| band | what lives there | example |
|---|---|---|
| `0x006c0000`–`0x006f0000` | **menus** — main menu, options, mods list, rebinding, world select, game over, credits, streaming integration | `0x006e3410` main menu |
| `0x00b50000`–`0x00b80000` | **in-game HUD and inventory** — stat bars, wand row, item backgrounds, stat readout, status indicators | `0x00b750b0` HUD bars |
| `0x0081d000`–`0x0082c000` | **the widget layer** — the generic primitives every screen is built from | `0x008212f0` the stat-bar widget |
| `0x0041d000`–`0x00444000` | **text, fonts, sprites, sound** | `0x0041dd60` sprite load |

The widget layer is the important boundary. A reimplementation does not need to copy any
screen code; it needs `0x008212f0`-equivalent primitives (stat bar, 9-slice box, tab button,
text row, checkbox, slider, scroll list) and then each screen becomes a short function.

## The four row widgets every menu is made of

The options screen is 2,471 lines of decompiled C, and that is only because it is a long
list of four row types. Nothing else is in it.

| row kind | function | behaviour |
|---|---|---|
| section heading | `0x006db930` | one text row, flag `0x4000000`; foldable headings also nudge a fold animation by +8 px when the state changes |
| section heading (variant) | `0x006db820` | same plus a float (−5.2 at every call site) passed to the text widget |
| toggle / checkbox | `0x006c87c0` | draws a button, and on click does `*value = (*value == 0)` — **writes straight into the config struct field** |
| slider | `0x006c6470` | forwards to the real slider `0x00824e70`, which computes `(*value − min) / (max − min)` for the fill |
| plain button | `0x008245d0` | everything that is not a toggle or a slider |

Because the checkbox writes through its own argument pointer, the options screen is a
literal transcription of the config struct. [ui-settings.md](ui-settings.md) is that
transcription.

## How each screen was identified

There are no symbols, so three independent signals were used, and each screen's attribution
is backed by at least two:

1. **Sprite paths.** Every UI element loads its art by literal path. The function holding the
   literal *is* the draw routine. 120 distinct asset paths resolve to 79 functions. The
   `data/ui_gfx/` subfolder the asset sits in is the screen: `hud/` → HUD,
   `inventory/` → wand editing, `save_slot/` → world select, and so on.
2. **Translation keys.** The screens are built from literal `$menu…`, `$menuoptions…`,
   `$hud_…`, `$inventory…` keys. The function that references a key builds that row. The
   options screen alone references 98.
3. **RTTI class names.** The executable is built with RTTI, so 2,591 C++ class names are
   recoverable — including `InventoryGuiComponent`, `InventoryGuiSystem`, `LoadingScreen`,
   `SimpleButton`, `SimpleSlider`, `SimpleTextbox`, `ConfigSliders`, `DebugUI`,
   `CreativeUITool`, `Screenshotter`, `CFont`, `TextImpl`, `TextSprite`. The inventory is a
   component-and-system pair in the ECS; the menu widgets are the older `Simple*` classes.
   Classes with no `dynamic_cast`/`typeid` sites have no code attributed, so RTTI names the
   shape of the UI but not always its functions.

## What the HUD is made of

One frame is driven by `0x00b7d8d0`, which is the top of the player-UI tree. It animates
the inventory, draws its background, draws the wand row, and calls three drawers:

| function | draws |
|---|---|
| `0x00b750b0` | one player's stat column and the wand/potion row: health, air, jetpack, potion contents, wand mana, fire-rate wait, reload countdown, gold, orbs |
| `0x00b795f0` | satiation (7-step ladder), on-fire, and the perk row with its "N more" popup |
| `0x00b7cbf0` | a per-player health-bar stack — **dead code in this build**, see below |

There is exactly **one** health bar: the one in `0x00b750b0`. A second function,
`0x00b7cbf0`, contains a complete per-player health-bar stack and is called from the same
frame driver — but its data source is never populated, so it cannot draw. See
[ui-screens.md](ui-screens.md), section "Dead code", for the chain.

Coordinates are in **UI-scale units**, not pixels. Every screen divides the window dimensions
by a display scale at `*(float *)(*(int *)(gui + 0x88) + 0x2b0)` before positioning anything,
which is why a mod that hardcodes pixels must also divide (see [ui-modding.md](ui-modding.md)).

Full formulas, per-screen layout and the bugs: [ui-screens.md](ui-screens.md).

## Tunables

The UI's pixel constants are game data, in `data/magic_numbers.xml`, and are readable and
editable. 40 UI-flavoured tunables are set there, of which
the important ones are:

| tunable | default | what it moves |
|---|---|---|
| `UI_BARS_POS_X` / `UI_BARS_POS_Y` | 20 / 20 | the stat column's top-left corner |
| `UI_BARS2_OFFSET_X` | −40 | the right-hand column's right margin |
| `UI_STAT_BAR_EXTRA_SPACING` | 2 | gap between stat rows |
| `UI_STAT_BAR_ICON_OFFSET_Y` | −1 | bar icon nudge |
| `UI_STAT_BAR_TEXT_OFFSET_X` / `_Y` | 10 / 0 | where the value text sits relative to the bar |
| `UI_HEALTHBAR_Y_SPACING` | 24 | vertical pitch when stacking bars |
| `UI_PLAYER_FULL_STATS_POS_X` / `_Y` | 530 / 70 | the expanded stats panel's origin |
| `UI_LOW_HP_THRESHOLD` | 1.3333 | when the low-HP flash starts |
| `UI_LOW_HP_WARNING_FLASH_FREQUENCY` | 8 | — |
| `INVENTORY_ICON_SIZE` | 20 | inventory slot icon size |
| `UI_ITEM_STAND_OVER_INFO_BOX_OFFSET_X` / `_Y` | 50 / −190 | the hover info box offset from the cursor |
| `UI_GAMEOVER_SCREEN_BOX_FROM_TOP_PERCENT` | 40 | game-over panel, inset from the top |
| `UI_GAMEOVER_SCREEN_BOX_FROM_SIDE_PERCENT` | 32 | game-over panel, inset from the side |
| `UI_GAMEOVER_SCREEN_BOX_FROM_BOTTOM_PERCENT` | 12 | game-over panel, inset from the bottom |
| `UI_PAUSE_MENU_LAYOUT_TOP_EDGE_PERCENTAGE` | 10 | pause menu top margin |
| `MAIN_MENU_BG_OFFSET_X` / `_Y` / `_Y_END` | 9250 / 2250 / 2320 | the animated main-menu background scroll |
| `MAIN_MENU_BG_TWEEN_SPEED` | 0.12 | how fast it moves |
| `UI_FULL_INVENTORY_OFFSET_X` | 170 | where the expanded inventory opens, relative to the wand row |
| `UI_IMPORTANT_MESSAGE_TITLE_SCALE` | 1 | the scale of an "important message" title |
| `UI_DAMAGE_INDICATOR_RANDOM_OFFSET` | 0 | randomises damage-number placement |
| `UI_WOBBLE_SPEED` / `UI_WOBBLE_AMOUNT_DEGREES` | 10 / 3 | the idle wobble on icons |
| `UI_SCALE_IN_SPEED` | 0.2 | how fast widgets scale in |
| `UI_LOCALIZE_RECORD_TEXT` | 1 | whether record values go through localisation |
| `UI_PLAYER_FULL_STATS_COLUMN2_OFFSET_X` / `UI_PLAYER_FULL_STATS_COLUMN3_OFFSET_X` | 10 / 45 | column offsets in the expanded stats panel |
| `UI_GAME_OVER_MENU_LAYOUT_TOP_EDGE_PERCENTAGE` | 19 | game-over menu top margin |
| `INVENTORY_STASH_X` / `INVENTORY_STASH_Y` | 370 / 80 | the stash pane's position |
| `INVENTORY_DEBUG_X` / `INVENTORY_DEBUG_Y` | 165 / 45 | the debug inventory overlay |
| `CREDITS_SCROLL_SPEED` | 25 | credits roll rate |
| `CREDITS_SCROLL_END_OFFSET_EXTRA` | 85 | extra offset past the end of the credits |
| `CREDITS_SCROLL_SKIP_SPEED_MULTIPLIER` | 15 | speed-up while the skip input is held |

Forty UI-flavoured tunables are set in vanilla `magic_numbers.xml`. The executable registers more
UI names that the vanilla file does not set (`UI_MAX_PERKS_VISIBLE`, `UI_QUICKBAR_OFFSET_X` / `_Y`,
`SETTINGS_MIN_RESOLUTION_X` / `_Y`, `UI_BARS_SCALE` and others), which keep their compiled-in defaults
unless a mod's magic-numbers file sets them. The rest of the file is audio, spawning and physics.

## What is not in the executable

- **Enum values** for `GUI_OPTION`, `GUI_RECT_ANIMATION_PLAYBACK`, key codes and cell
  materials — see [enums.md](enums.md).
- **The real on-disk path of the user data folder.** The config file is `??USR/config.xml`
  and the logical name is `Config`, both read straight out of the quit path, but `??USR` is
  a virtual-filesystem token resolved at runtime from `-always_store_userdata_in_appdata` /
  `-always_store_userdata_in_workdir`. The AppData folder name is `Nolla_Games_Noita`.
- **The names of modders' own assets**, naturally.

## Bugs worth knowing before you copy the behaviour

Each of these is read straight off the decompilation of the function that computes it, which is
named at the top of its bullet. Per-screen detail is in [ui-screens.md](ui-screens.md).

- **The jetpack readout always says `1`.** The percentage is computed as
  `(fuel * 100) / (fuel * 100)`. The bar graphic is right; only the number is wrong.
- **Gold jitters.** One frame in three the displayed gold is off by up to ±3, and the
  jittered value is written back into player state — so anything reading that field sees
  the noisy value.
- **The HP number's font size is logarithmic**, capped at 80 above 1000 max HP (40 at 100 HP).
  A fixed size looks wrong the moment max HP changes.
- **Mana values are truncated to integers** while HP keeps its decimals. Inconsistent, and
  visible in game.
- **Sprite paths are rebuilt and hand-length-scanned every frame, per bar, per player.** A
  reimplementation that interns them is strictly faster and behaves the same.
- **The low-HP flash and the damage-ghost bar use opposite parities of the same 30-frame
  counter**, so the fill and its background alternate against each other, not together.
- **The fold arrow on options section headings was not recovered.** `button_fold_*.png` is
  per-mod only; the general headings use some other unnamed asset.