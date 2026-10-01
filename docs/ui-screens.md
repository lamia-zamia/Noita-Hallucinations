# The game's own UI, screen by screen

Every screen, its function, and the arithmetic that positions things. Read this with
[game-ui.md](game-ui.md), which explains how the screens were found and where they live in the
executable, and [gui-cookbook.md](gui-cookbook.md), which is the Lua-facing widget API they are
built on.

The single most important fact: **the game's own UI is not a separate renderer.** It calls the
same GUI engine the `Gui*` Lua API exposes, from C++, with pre-baked positions. There is no
privileged UI path. Everything below is ordinary widgets.

Addresses are from one Steam build and will move on update.

## Two things that catch everyone out

**Positions are in UI-scale units, not pixels.** Every screen divides the window dimensions by
a display scale at `*(float *)(*(int *)(gui + 0x88) + 0x2b0)` before positioning anything. A
mod that hardcodes pixels without dividing lands in the wrong place at any non-native
resolution.

**The menus and the in-game HUD are separate projects.** The menus (`0x006c`–`0x006e`) are
hand-laid-out with literal offsets; the HUD (`0x00b7`) is tunable-driven and reads
`magic_numbers.xml`. They are ~0x4a0000 bytes apart and share no code. Do not expect a menu
constant to mean anything in the HUD.

## The in-game HUD

One frame is driven by `0x00b7d8d0`, the top of the player-UI tree. It animates the inventory,
draws its background and the wand row, then calls three drawers.

### Stat bars — `0x00b750b0`

Draws one player's stat column and the wand/potion row. Two containers:

- left column at `UI_BARS_POS_X`, right column at `window_width/ui_scale + UI_BARS2_OFFSET_X`
- both start at `UI_BARS_POS_Y + playerIndex * pitch`, where the player index is a **linear scan**
  of the active-player array

Each bar is one call to the stat-bar widget `0x008212f0`, whose contract is:

```
(gui, rect, id, value, max, ghostValue, ghostAlpha, textScale, textExtra,
 fill, damage, damageFlash, more, background, ...)
```

**Five sprite slots, in that fixed order, and two of the five are the same file.** Passing four
gets you garbage. The order was confirmed from two independent call sites.

**The HP text size is logarithmic:**

```
textScale = max(40.0, log10(maxHp) * 40.0)
```

capped at a constant above a maximum-HP threshold. The HP readout grows with the *logarithm* of
max health and is floored at 40 — which is why it never overflows. A fixed size looks wrong the
moment the player takes a max-health container.

**The damage ghost.** On damage the game stores the HP at that moment and draws the difference
as a separate chunk, fading out over a second:

```
ghost      = min(0, hp − max(storedDamageHp, 0))
ghostAlpha = clamp01(1 − (now − lastDamageFrame) / 60)
```

**The low-HP flash and the ghost use opposite parities of the same 30-frame counter.** On the
even half-second the *background* switches to the low-HP sprite; on the odd half-second the
*fill* switches to the damage sprite. They alternate against each other, not together. The
threshold is compared against **raw HP**, not a ratio — which may be a latent bug; the tunable
default `UI_LOW_HP_THRESHOLD = 1.3333` is a ratio, and if that is what the compiled constant is,
the condition is essentially never true.

**Mana and max mana are truncated to integers** while HP keeps its decimals. Inconsistent, and
visible in game.

**Gold jitters.** One frame in three the displayed gold is off by up to ±3, and the jittered
value is written back into player state, so anything reading that field sees the noise.

**The jetpack readout always says `1`.** The percentage is computed as `(fuel * 100) / (fuel * 100)`.
The bar graphic is right; only the number is wrong.

Sprites, per bar: `colors_bar_bg.png` (background), `colors_flying_bar.png` (air/jetpack fill),
`colors_mana_bar.png` (mana/potion fill), `colors_reload_bar.png` (reload fill),
`colors_reload_bar_bg_flash.png`, `colors_health_bar.png`, `colors_health_bar_damage.png`,
`colors_health_bar_more.png`, `colors_health_bar_bg_low_hp.png`. Icons: `health.png`, `mana.png`,
`reload.png`, `fire_rate_wait.png`, `jetpack.png`, `potion.png`, `money.png`, `orbs.png`. Labels:
`$hud_health`, `$hud_air`, `$hud_jetpack`, `$hud_wand_mana`, `$hud_wand_reload`, `$hud_gold`,
`$hud_orbs`, `$hud_title_wands`, `$hud_title_throwables`, `$infinity_symbol`.

The air bar's sprite scale is **1.43**, not 1.0 — the only non-unit bar scale in the file.

### Status indicators and perks — `0x00b795f0`

Three things in one column: the satiation indicator, on-fire, and the perk row with its
overflow popup.

**Satiation** is a 7-step sprite ladder selected by a **linear scan from the top threshold
down**, first threshold ≤ value wins, defaulting to `satiation_00`. Two of the seven
thresholds are literal (0.25, 0.90, 1.40) and the rest come from tunables. The fill fraction is
clamped with the game's usual unsigned-64-bit compare idiom.

**On fire** uses the same component as the health bar: a flag at `+0x204`, a timer and timer-max
at `+0x220`/`+0x224`.

**The perk row** is a horizontal strip of N equally spaced slots near the bottom of the screen,
with mouse clamping that produces a "does it fit on screen" flag — and when it doesn't fit, the
draw count drops by one. That is the whole overflow mechanism; there is no scroll.

A **dt-scaled lerp** smooths the hover position: `v += (target − v) * dt * k`. Get this wrong
with a per-frame constant and the strip behaves differently at 144 fps than at 60.

### Dead code: the multi-player health-bar stack — `0x00b7cbf0`

A complete per-player health-bar stack that **cannot draw anything in this build**. It is
called every frame from the same driver as the HUD, it has a full implementation — the loop,
the sprite selection, the player-name label — and it is unreachable.

The reason is its data source. It iterates a global vector of 72-byte per-player HUD records
between two pointers, and bails unless the range is non-empty. Tracing who fills it:

```
FUN_00b7d510   constructs a snapshot struct, publishes it, destroys it
  └─ FUN_00b7d570   copies the snapshot field-by-field into the HUD globals
       └─ FUN_00c86980  assigns the snapshot's record vector to DAT_0122252c/30
```

`FUN_00b7d570` has exactly one caller, and the snapshot that caller publishes comes from a
constructor that **zeroes all eighteen fields** before publication. So the two pointers are
always equal, the loop never runs, and the function returns having drawn nothing.

The only other writer, `FUN_00b7d890`, destroys the range and then sets the end pointer back to
the beginning — it *empties* the vector, and has no callers.

Which makes this a fossil rather than a bug: the game shipped local co-op UI code and never
populated it. Noita Together exists as a mod precisely because the game has no multiplayer. If
you are reimplementing, this is the one HUD function you can skip entirely — and it is also a
trap, because it is a faithful, working, well-commented-looking health bar sitting in the same
address band as the real one.

Had this not been checked, the obvious reading — two code paths that both draw a health bar, in
the same frame, with the same sprites — is "there are two health bars on screen". There aren't.

### The frame driver — `0x00b7d8d0`

The whole open/close animation, and it is the interesting part:

```c
v += (target − v) * dt * 10.0
```

stored on the player and mirrored into the GUI. That scalar is then **pushed as a colour into the
player's own sprite**, which is how Noita dims the player while the inventory is open — target
0.5, so half brightness. A one-line trick any mod can copy.

The text row height is *measured*, not hardcoded: `GuiGetTextDimensions` on a 3-byte string at
scale 2, minus 2.

The wand row origin is `UI_BARS_POS_X + <slot>`, `UI_BARS_POS_Y`, advancing by the icon pitch.
Each inactive slot gets a greyed-out overlay.

The inventory background is drawn at a **z of 1000** so it sits behind everything, and the
open/close transition plays `ui/inventory_open` / `ui/inventory_close`.

**The inventory toggles itself** from raw input bits including gamepad D-pad buttons, by flipping
a flag on the player record. A mod cannot suppress it without patching that code.

### Wand inventory — `0x00b788e0`

The wand-editing box, the inactive overlay, and the wand-info panel hookup. Per-slot widget ids
in a contiguous band. The wand-info panel call passes the hovered slot and two immediate floats
(8.0 and 32.0, plausibly text scales), and the box offset depends on **whether the wand's stack
size is under 22** — that is the wide-versus-narrow box switch.

### Item backgrounds — `0x00b53a00`

Not a draw function: a `switch` on the item-kind enum at entity offset `+0x7c` that writes the
nine-piece background path into a `std::string`.

| kind | sprite |
|---|---|
| 0 | `item_bg_projectile.png` |
| 1 | `item_bg_static_projectile.png` |
| 2 | `item_bg_modifier.png` |
| 3 | `item_bg_draw_many.png` |
| 4 | `item_bg_material.png` |
| 5 | `item_bg_other.png` |
| 6 | `item_bg_utility.png` |
| 7 | `item_bg_passive.png` |
| default | `item_bg_projectile.png` |

**An unknown item kind renders with the projectile background.** Also: it is an *assigning*
constructor that never frees the old buffer, so reusing one string for many items leaks one
allocation per item.

### Wand stat readout — `0x00b65fb0`

The wand info panel: damage, spread, speed, reload, mana, cast delay, recharge, capacity,
knockback, bounces, explosion radius, critical chance, plus the localised description. It is the
largest single UI string consumer in the game — 94 literals — because every stat has three
strings: a value label, a tooltip, and the stat key it reads.

Note the naming mismatch in the game data: the stat key is `damage_electricity` while the label
is `$inventory_mod_damage_electric`. Reproduce it verbatim if you key off the stat name.

`$inventory_mod_*` versus `$inventory_dmg_*` is a real distinction — there is a parallel mod
label for each damage type plus knockback, speed, spread, bounces, explosion radius, recharge,
cast delay and crit chance, and the panel picks between them.

This function **timed out the decompiler**, so its layout arithmetic is not recovered here.
Only its strings and its call shape are known.

## The front-end menus

### Main menu / pause menu — `0x006e3410`

One function serves both. The discriminator is **not** the menu state — it is the argument: an
empty string means front-end, non-empty means in-game pause. The item list, ids and layout code
are literally identical between the two.

The state check has three branches: draw the menu; close and resume; or **delegate to the pushed
sub-screen callback**. Options, the two rebinding screens and everything else are reached
through that delegation, not by this function knowing about them.

The items, in code order:

| # | label | id | shown when |
|---|---|---|---|
| — | logo | `0x3663` | front-end only |
| — | `$menu_paused` + help image | `0x3665` | pause only; the help image is `help_keyboardmouse.png`, or `help_gamepad360.png` with a gamepad |
| 1 | `$menu_continue` | `0x3667` | only if a save exists; greyed out in the front-end when there is none |
| 2 | `$menu_newgame` | `0x3669` | always |
| — | progress notifier | — | pause only |
| 3 | `$menu_options` | `0x366b` | always |
| 4 | `$menu_mods` | `0x366d` | always; becomes `$menu_mods_incompatibilities` when incompatible mods are loaded |
| 5 | `$menu_releasenotes` | `0x366f` | always |
| 6 | `$menu_credits` | `0x3671` | always |
| 7 | `$menu_saveandquit` / `$menu_quit` | `0x3673` | unless demo mode |

Then, in pause only, an info block: `$menupause_worldseed`, `$menupause_gamemode`, a run of blank
lines, `$menupause_location` (suppressed when the location is the empty sentinel — no world
loaded), `$menupause_modsused`. Then the build string, with `" - DEMO MODE"` appended in demo
mode.

The front-end background scrolls with a per-frame accumulator scaled by a constant, resetting to
zero when it would cross — and that accumulator is **frame-rate dependent**, unlike the HUD's
dt-scaled lerps.

### Options — `0x006d5620`

Six tabs, a pointer-table tab strip, ~60 options. Full detail in
[ui-settings.md](ui-settings.md).

### World select — `0x006d15f0`

A centred row of up to **7 save slots**, each a banner with the slot name, world name, play
time, save date and a delete button. The whole list is centred:

```
x = (screen_w/ui_scale − slot) * 0.5
y = (screen_h/ui_scale − slot) * 0.5
```

**14 widget ids, stride 112 bytes: 7 slots × (banner + delete button)**, ids `0x3525` through
`0x3531` in steps of 2. A slot with no save gets a greyed-out banner and a dark tint
(`0x7f7f7f7f`).

Banner sprites: `banner_background_continue.png` / `_hovered`, and the `newworld` pair for an
empty slot.

### Progress — `0x006dfe40`

Three stacked sections — Perks, Spells/actions, Enemies — each a grid of cells with a
`found/total` header and "N new" markers, plus the ending badge and a Return button.

The whole screen is a scroll container wrapping a horizontal container wrapping a vertical one.
It uses **literal** `4.0` offsets and margins and does not read the `UI_*` tunables at all.

Each grid cell is drawn by `0x006df000`: the box, an unknown variant, and the "new" shine, with
sprites `grid_box.png`, `grid_box_unknown.png` and `grid_highlight_new.png`. Perks additionally
get their `item_bg_*.png` background at scale 0.5 and alpha 0.09.

### Game over — `0x006e51e0`

The title is a **slide-in from the right** driven by a per-frame accumulator: scale is
`clamp01(ease(t))`, x is `slot − ease(t) * slot − 48`, with a different trailing offset for the
completed variant. Sprites: `game_over.png`, `game_completed.png`, `record.png`.

Then a run of stat rows through one helper, which computes the "is this a record" flag by
comparing the run's 64-bit value against the stored best and prefixes a record with the
infinity glyph. The stats: `$stat_modsenabled`, `$stat_gold`, `$stat_time`, `$stat_depth`,
`$stat_places_visited`, `$stat_enemies_slain`, `$stat_max_hp`, `$stat_items_found`, `$stat_orbs`,
`$stat_streaks`, `$stat_total_deaths`, `$stat_total_wins`, plus cause of death.

Buttons: `$menugameover_newgame`, `$menugameover_savereplay` (only when the recorder is active),
`$menugameover_quit`.

The accumulator here is **frame-rate dependent, not dt-scaled** — unlike the HUD. If you
reimplement it with a constant, the title animation runs at a different speed on a 144 Hz
display.

### Replay editor — `0x0062c990`

Twenty `$menu_replayedit_*` keys: clip start/end for gamepad and keyboard separately, frame
counter, image centre, output scale and size format, save-as-GIF, open GIF folder, and the
writing-GIF progress readout.

## How the UI reads game state

Worth knowing, because it tells you what a mod could imitate. Every player-status number comes
from one component, reached from three different draw functions:

| offset | meaning |
|---|---|
| `+0x48` / `+0x50` | health / max health, as `double` |
| `+0x2bc` | frame index of the last damage event |
| `+0x2c0` | the HP the damage ghost decays from |
| `+0xc8` / `+0xcc` | air / fluid current / max, as `float` |
| `+0x6c` / `+0x78` | satiation array and timers |
| `+0x204` | on-fire flag |
| `+0x220` / `+0x224` | fire timer / max |

Sibling components have their own getters: jetpack (`+0x110` fuel, `+0xa0` has-fuel flag), wand,
wand state (`+0x84` reload, `+0x88`/`+0x8c` mana max/current, `+0x3a8` reload end frame), gold,
orb count.

**What to copy:** the stat-bar contract (value, max, ghost, ghost alpha, text scale, five
sprites) is a complete reusable description of a Noita bar; the log-scaled numeric text; the
2 Hz square wave with opposite parities; the dt-scaled lerp; and the player tint driven from the
inventory-open animation.

**What not to copy:** the per-frame hand-rolled `strlen` over
sprite paths, the jetpack's `(f*100)/(f*100)`, the gold jitter written back into player state,
and the frame-rate-dependent title animation. All four are real, three are bugs, and all four
bite a reimplementation that copies the structure without reading the arithmetic. And do not be misled by `0x00b7cbf0` - a second, complete, unreachable health bar sitting in the same address band as the real one.

## Not recovered

- The fold-arrow sprite used by options section headings — `button_fold_*.png` is per-mod only,
  so the general headings use some other unnamed asset.
- The **real** path of the user data directory.
- The `config.xml` element value format — element text or attribute.
- The health-bar Y anchor fraction, which comes from a compiled tunable whose value was not
  decoded. This is the one number you would need to place a bar exactly where the game does.
- The arithmetic of `0x00b65fb0`, the wand stat panel — it timed out the decompiler.