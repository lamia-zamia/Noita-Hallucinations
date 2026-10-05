# The options screen and the config file

Noita's settings are a literal transcription of a C++ struct. The checkbox widget writes
through its own argument pointer (`*value = (*value == 0)`), so every row on the options
screen *is* a struct field, and the config file is that struct serialised. This page is that
transcription: every option, its label, the struct offset it lives at, and its range.

Addresses are from one Steam build and will move on update.

## The screen

One function, `0x006d5620`, 2,471 lines of decompiled C. It is a **sub-menu pushed onto a
global menu stack** — the main menu's `$menu_options` calls `FUN_008a5dd0(&DAT_012076bc,
&FUN_006d5620)` and sets a "first frame" flag, so the screen knows to enumerate displays and
set up its banner countdown the first time it runs.

Six tabs, selected by the global `DAT_012076ac`:

| # | tab label |
|---|---|
| 0 | `$menuoptions_general` |
| 1 | `$menuoptions_graphics` |
| 2 | `$menuoptions_audio` |
| 3 | `$menuoptions_input` |
| 4 | `$menuoptions_streaming` |
| 5 | `$menu_mods_settings_short` |

Four of those six labels **have no code reference anywhere in the binary**, which is a trap
worth knowing about if you go looking for them: the tab strip is a **pointer table indexed
arithmetically**, at `0x01153430`, holding six pointers plus a null terminator. The loop is
`ppuStack_3c0 = &PTR_s__menuoptions_general_01153430 + i`. No `PUSH` or `LEA` of those string
addresses is ever emitted, so a string cross-reference pass cannot find them. Only a data
reference from the table exposes them. The same mechanism hides the sixth tab.

### Layout

Coordinates are divided by the GUI's display scale at `*(float *)(*(int *)(gui + 0x88) + 0x2b0)`
before use. In UI-scale units:

| element | position |
|---|---|
| screen box | `x = (screenW / scale) * 0.1355`, `y = (screenH / scale) * 0.5 − 200` |
| tab strip | advances 24 px per tab row, plus a fraction-of-width cursor nudge |
| body panel | a 338 × 265 nine-slice box, then a vertical container with a **4 px row gap** |
| tab button | three sprites: `tab_selected.png`, `tab_hovered.png`, `tab.png` |

Note what is **not** used: this screen does not read the `UI_*` tunables from
`magic_numbers.xml`. It hardcodes 24, 338, 265, 200, 4, 0.1355. The tunables move the in-game
HUD, not the menus.

### Widget ids

Every row carries a hand-assigned 16-bit id, and these are the ids the GUI's widget-state and
hover map is keyed on — the same id space the Lua `Gui*` API uses. They are in two contiguous
bands, which matters if you are reimplementing and want stable ids:

- `0x34c5`, `0x34c7` — list fallbacks
- `0x35a7` — gamepad-detected banner
- `0x35a9`–`0x35c9` — General
- `0x35cb`–`0x35df` — Graphics
- `0x35e1`–`0x35e3` — Audio
- `0x35e5`–`0x35f7` — Input
- `0x35f9`–`0x3613` — Streaming
- `0x3615`–`0x3619` — Mods
- `0x361b` — Apply
- `0x3535`–`0x3569` — keyboard rebinding screen
- `0x356b`–`0x3611` — gamepad rebinding screen

## The options, tab by tab

`off` is the byte offset in the config struct. `✓` means the offset was read out of the
serialiser and agrees with what the options screen dereferences.

### Tab 0 — General

| label | id | config key | off | type | default | range |
|---|---|---|---|---|---|---|
| `$menuoptions_windowmode` | `0x35a9` | `fullscreen` | `0x40` ✓ | enum, cycles 0/1/2 | from GraphicsDevice | `$windowmode_windowed` / `_fullscreen` / `_fullscreen_real` |
| `$menuoptions_resolution` | `0x35ab` | `window_w`, `window_h` | `0x38`, `0x3c` ✓ | enum, cycles the mode list | from GraphicsDevice | left-click next, right-click previous; below **640 px** wide shows `$menuoptions_resolution_illegible` |
| `$menuoptions_matchresolution` | `0x35ad` | — | — | button | — | sets the internal render size to match the window |
| `$menuoptions_display_number` | `0x35af` | `display_id` | `0x7c` ✓ | enum, cycles monitors | from GraphicsDevice | re-picks the closest resolution on the new monitor |
| `$menuoptions_vsync` | `0x35b1` | `vsync` | `0x78` ✓ | enum 0/1/2 | from GraphicsDevice | `$option_off` / `$option_on` / `$option_adaptive` |
| `$menuoptions_application_rendered_cursor` | `0x35b3` | `application_rendered_cursor` | `0xb4` ✓ | bool | — | re-creates the cursor on change |
| `$menuoptions_replayrecorder` | `0x35b5` | `replay_recorder_enabled` | `0xc0` ✓ | bool | — | |
| `$menuoptions_replaybudget` | `0x35b7` | `replay_recorder_max_budget_mb` | `0xc4` | int | 150 | 10 … 500 |
| `$menu_replayedit_opengifdir` | `0x35b9` | — | — | button | — | opens the recorded-GIF folder |
| `$menuoptions_online_features` | `0x35bb` | `online_features` | `0xe41` ✓ | bool | — | turning it off also clears `streaming_integration_autoconnect` |
| `$menuoptions_checkforupdates` | `0x35bd` | `check_for_updates` | `0xe8` | bool | — | only drawn if the build supports it |
| `$menuoptions_steamcloud` | `0x35bf` | — | — | bool | Steam API | value lives in the Steam object, not the config struct |
| `$menuoptions_steamcloud_warning_enabled` | `0x35c1` | — | `0xe48` | bool | — | **the one row with no matching config key** |
| `$menuoptions_steamcloud_warning_limit` | `0x35c3` | `steam_cloud_size_warning_limit_mb` | `0xe44` ✓ | int | 50 | 10 … 300 |
| `$menuoptions_privacypolicy` | `0x35c5` | — | — | button | — | opens the privacy policy URL |
| `$menuoptions_language` | `0x35c7` | `language` | `0xd0` ✓ | enum | current | reloads translations on change |
| `$menuoptions_resetsave` | `0x35c9` | — | — | button | — | confirm dialog |
| `$menu_applyandreturn` | `0x361b` | — | — | button | — | drawn on every tab, outside the tab switch |

Plus five section headings: `$menuoptions_heading_window` (not foldable),
`_compatibility`, `_replayrecorder`, `_online`, `_misc`.

### Tab 1 — Graphics

| label | id | config key | off | type | default | range |
|---|---|---|---|---|---|---|
| `$menuoptions_pixelart_aa` | `0x35cb` | `rendering_pixel_art_antialiasing` | `0x96` ✓ | bool | — | |
| `$menuoptions_lowres` | `0x35cd` | `rendering_low_resolution` | `0x95` ✓ | bool | — | |
| `$menuoptions_lowqualityrendering` | `0x35cf` | `rendering_low_quality` | `0x94` ✓ | bool | — | |
| `$menuoptions_dithering` | `0x35d1` | `rendering_filmgrain` | `0xe40` | bool | — | label and key disagree |
| `$menuoptions_cosmeticparticlecoeff` | `0x35d3` | `rendering_cosmetic_particle_count_coeff` | `0xa8` | float | 1.0 | 0.05 … 1.0, shown ×100 |
| `$menuoptions_brightness` | `0x35d5` | `rendering_brightness_delta` | `0x98` ✓ | float | 0.0 | −0.09 … 0.09 |
| `$menuoptions_contrast` | `0x35d7` | `rendering_contrast_delta` | `0x9c` ✓ | float | 0.0 | −0.12 … 0.2 |
| `$menuoptions_gamma` | `0x35d9` | `rendering_gamma_delta` | `0xa0` ✓ | float | 0.0 | −0.32 … 0.695 |
| `$menuoptions_damagenumbers` | `0x35db` | `ui_report_damage` | `0xbe` ✓ | bool | — | label and key disagree |
| `$menuoptions_ui_snappy_hover_boxes` | `0x35dd` | `ui_snappy_hover_boxes` | `0xe4a` ✓ | bool | — | |
| `$menuoptions_screenshake_intensity` | `0x35df` | `screenshake_intensity` | `0xb8` | float | 0.7 | 0.0 … 1.0 |

Headings: `$menuoptions_heading_rendering` (not foldable), `_userinterface_graphics`,
`_accessibility`.

### Tab 2 — Audio

| label | id | config key | off | type | default | range |
|---|---|---|---|---|---|---|
| `$menuoptions_musicvolume` | `0x35e1` | `audio_music_volume` | `0x8c` ✓ | float | 0.75 | 0.0 … 1.0 |
| `$menuoptions_soundsvolume` | `0x35e3` | `audio_effects_volume` | `0x90` ✓ | float | 1.0 | 0.0 … 1.0 |

### Tab 3 — Input

| label | id | config key | off | type | default | range |
|---|---|---|---|---|---|---|
| `$menuoptions_configurecontrols` | `0x35e5` | — | — | button | — | pushes the keyboard rebinding screen |
| `$menuoptions_ui_inventory_icons_always_clickable` | `0x35e7` | `ui_inventory_icons_always_clickable` | `0xbc` ✓ | bool | — | |
| `$menuoptions_ui_allow_shooting_while_inventory_open` | `0x35e9` | `ui_allow_shooting_while_inventory_open` | `0xbd` ✓ | bool | — | |
| `$menuoptions_showhoverinfonexttomouse` | `0x35eb` | `ui_show_world_hover_info_next_to_mouse` | `0xbf` ✓ | bool | — | |
| `$menuoptions_capturemouseinsidewindow` | `0x35ed` | `mouse_capture_inside_window` | `0xe49` ✓ | bool | — | re-inits the window on change |
| *(dynamic label)* | `0x35ef` | `gamepad_mode` | `0xe3c` ✓ | enum, cycles devices | −2 = none | −2 → `$option_off`, −1 → `$menuoptions_controls_autodetectgamepad`, ≥0 → device name |
| `$menuoptions_configuregamepad` | `0x35f1` | — | — | button | — | pushes the gamepad rebinding screen |
| `$menuoptions_gamepad_rumble` | `0x35f3` | `joystick_rumble_intensity` | `0x34` ✓ | float | 1.0 | 0.0 … 1.0 |
| `$menuoptions_gamepad_analog_flying` | `0x35f5` | `gamepad_analog_flying` | `0xe4c` ✓ | bool | — | |
| `$menuoptions_pause_the_game_when_unfocused` | `0x35f7` | `application_pause_when_unfocused` | `0xe4b` ✓ | bool | — | NoitaDearImGui documents this one as its own dependency |

Headings: `$menuoptions_heading_controls` (not foldable), `_userinterface_mouse`, `_gamepad`,
`_userinterface_input`.

### Tab 4 — Streaming

The game's built-in broadcast-overlay feature. Every row is a config field except the connect
buttons and the per-event rows.

| label | id | config key | off | type | default | range |
|---|---|---|---|---|---|---|
| `$menu_streaming_channelname` | `0x35fb` | `streaming_integration_channel_name` | `0xe84` | **text input** | — | 25 chars, charset `[A-Za-z0-9_]` |
| `$menu_streaming_connect` / `_disconnect` | `0x35ff` / `0x35fd` | — | — | buttons | — | |
| `$menu_streaming_timebetweenvotes` | `0x3601` | `streaming_integration_time_seconds_between_votings` | `0xea8` ✓ | int | 120 | 10 … 180 |
| `$menu_streaming_timevoting` | `0x3603` | `streaming_integration_time_seconds_voting` | `0xea4` ✓ | int | 60 | 10 … 120 |
| `$menu_streaming_playvotesound` | `0x3605` | `streaming_integration_play_new_vote_sound` | `0xeac` ✓ | bool | — | |
| `$menu_streaming_hidevotecounts` | `0x3607` | `streaming_integration_hide_votes_during_voting` | `0xeae` ✓ | bool | — | |
| `$menu_streaming_uiposleft` | `0x3609` | `streaming_integration_ui_pos_left` | `0xeaf` ✓ | bool | — | |
| `$menu_streaming_usernames_visible` | `0x360b` | `streaming_integration_viewernames_ghosts` | `0xead` ✓ | bool | — | |
| `$menu_streaming_installmods` | `0x360d` | — | — | button | — | |
| `$menu_streaming_eventliststartnewgame` | `0x3611` | — | — | button | — | front-end only |
| `$menuoptions_disableall` / `_enableall` | `0x3613` | — | — | button | — | label switches on whether any event is enabled |
| *(per-event rows)* | — | — | `+0x20` | toggles | — | one row per event, with an author/kind/distribution tooltip |

The **text input** widget is worth naming because it settles what the charset string is for.
`ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz_0123456789` is referenced exactly once in
the whole binary, as field `[10]` of a text-input descriptor, together with `128.0f` width and
`25` max length. That is the documented `GuiTextInput(..., allowed_characters)` argument, and
the descriptor's default for field `[10]` is `NULL` = unrestricted.

The connect action posts the user **`justinfan`** with the password **`passwordcabeanything`**
and a hardcoded account string. It is in the binary in plaintext, and it is the game's own
stock-integration credential, not a secret of yours — but if you are writing your own client,
do not assume it still works.

### Tab 5 — Mods

Not options but the same screen. A master checkbox (`0x3615`) enable/disables everything, then
one row per installed mod: a name, a toggle that flips `modPtr[0x22]`, and a fold button using
`button_fold_close.png` / `button_fold_open.png` — **the only place those two sprites are used**;
the general options headings use some other, unnamed fold asset that was not recovered.

When a mod is expanded, the C++ screen calls **back into that mod's Lua**. The contract:

> a mod defines a global function `ModSettingsGui(gui, is_front_end)`. The game fetches it,
> checks that it is a function, pushes the C++ GUI object as light userdata and a boolean
> saying whether the front-end menu (rather than the in-game pause menu) is what's open, and
> calls it with two arguments. If the value is not a function it unwinds cleanly.

The mod's settings come from its `settings.lua`, read after `mods/<id>/mod.xml` and
`mod_id.txt`.

## The config file

`??USR/config.xml`, logical name `Config`, written on the quit path. `??USR` is Noita's
virtual-filesystem token for the user-data root; the concrete directory is chosen at runtime
from the command-line switches `-always_store_userdata_in_appdata` and
`-always_store_userdata_in_workdir`, and the AppData folder name is `Nolla_Games_Noita`. The
binary does not contain the resolved path.

Format is XML: a single `<Config>` element with **every key as an attribute** (one per line,
two-space indent), and two child elements for the named sub-sections, which carry their keys
the same way:

- `[KeyboardControls]` at config offset `0x10c`
- `[GamepadControls]` at config offset `0x7a4`

**Everything is a string in the file.** There is no per-key type tag — the type lives only in
the compiled-in visitor. That is why `rendering_brightness_delta=0.05` works and
`rendering_brightness_delta=abc` silently becomes 0. It is also why some keys round-trip
through a float converter and some through an int one, so a value you hand-write in decimal
form may not survive a load/save cycle for the int-typed keys.

Writes are **write-if-absent**: the serialiser checks `map.find(key)` and only appends when the
key is not already present. But the save path builds a *fresh* document, so at the top level
the guard never fires and **the file is rewritten in full, with the attributes in alphabetical
order, on every save**. Your hand-added ordering and comments do not survive.

Defaults for absent keys come from the config struct's own initialiser, not from a table in
the file. For the two control sections the default is generated on demand and copied in
wholesale.

`config_format_version` (`0x104`) and `is_default_config` (`0x108`) let the game notice a stale
or foreign config. What it does about a mismatch was not traced.

### Config keys with no UI row

Worth knowing because they are the interesting ones for a reimplementer:

| key | off | what |
|---|---|---|
| `internal_size_w` / `internal_size_h` | `0x4` / `0x8` | the internal render resolution |
| `has_been_started_before`, `audio_fmod`, `last_started_game_version_hash` | `0x88`, `0x89`, `0xec` | first-run flag, audio-engine switch, build hash of the last launch |
| `framerate` | `0xc` | |
| `sounds`, `report_fps`, `joysticks_enabled` | `0x11`, `0x30`, `0x31` | engine-wide toggles with no menu row |
| `record_events`, `do_a_playback`, `playback_file`, `event_recorder_flush_every_frame` | `0x13`, `0x14`, `0x18`, `0x12` | the event recorder |
| `backbuffer_width` / `_height` | `0xac` / `0xb0` | written together with `window_w`/`_h` on apply |
| `replay_recorder_max_resolution_x` / `_y` | `0xc8` / `0xcc` | caps the GIF recorder |
| `rendering_teleport_flash_brightness` | `0xa4` | |
| `mods_active`, `mods_active_privileged` | `0xe50`, `0xe68` | the mod lists, as written by the Mods tab |
| `mods_sandbox_enabled`, `mods_sandbox_warning_done`, `mods_disclaimer_accepted` | `0xe80`, `0xe81`, `0xe82` | the main menu shows a disclaimer dialog when `0xe82` is 0 |
| `streaming_integration_autoconnect`, `streaming_integration_events_per_vote` | `0xe83`, `0xe9c` | |
| `single_threaded_loading` | `0xeb0` | set to 1 immediately before the config is saved on quit |
| `DEBUG_DONT_LOAD_OTHER_CONFIG` | `0xecc` | |
| `graphics_settings.textures_resize_to_power_of_two`, `graphics_settings.textures_fix_alpha_channel` | `0x74`, `0x75` | texture loading, no row |

## Key bindings

The controls serialiser is `0x00854690` — it is large enough to time out the decompiler, but its
structure is fully determined by three independent pieces of evidence: the 122 key strings it
references, its two callers (load and save), and the block-copy function `0x006d4220` which
copies `this + 0x38 * n` for n = 0..29.

**Each bindable action is a 56-byte record:**

```
int     primary          ; the engine key/button code tested at runtime
int     secondary        ; the alternate binding, 0 = none
string  primary_name     ; display name
string  secondary_name
```

So each action contributes **four** config keys — `<action>.primary`, `<action>.secondary`,
`<action>.primary_name`, `<action>.secondary_name` — which is what the 122 strings are: 30
actions × 4, plus two scalars per section.

Those two scalars are `gamepad_analog_sticks_threshold` (default **0.5**) and
`gamepad_analog_buttons_threshold` (default **0.25**).

### The 30 actions

`key_up`, `key_down`, `key_left`, `key_right`, `key_use_wand`, `key_spray_flask`, `key_throw`,
`key_kick`, `key_inventory`, `key_interact`, `key_drop_item`, `key_drink_potion`,
`key_item_next`, `key_item_prev`, `key_item_slot1` … `key_item_slot10`, `key_takescreenshot`,
`key_replayedit_open`, `aim_stick`, `key_ui_confirm`, `key_ui_drag`, `key_ui_quick_drag`.

### Keyboard defaults

The keycode namespace is the engine's own, not Win32: `4` left, `7` right, `22` down, `26` up,
`−1` mouse-left, `−2` mouse-right, `−4` mouse-middle, `−16` wheel down, `−8` wheel up, `44` space, `43` tab.

| action | primary | secondary | action | primary | secondary |
|---|---|---|---|---|---|
| `key_up` | 26 | 44 | `key_item_slot1` | 30 | |
| `key_down` | 22 | | `key_item_slot2` | 31 | |
| `key_left` | 4 | | `key_item_slot3` | 32 | |
| `key_right` | 7 | | `key_item_slot4` | 33 | |
| `key_use_wand` | −1 | | `key_item_slot5` | 34 | |
| `key_spray_flask` | −1 | | `key_item_slot6` | 35 | |
| `key_throw` | −2 | | `key_item_slot7` | 36 | |
| `key_kick` | 9 | | `key_item_slot8` | 37 | |
| `key_inventory` | 12 | 43 | `key_item_slot9` | 38 | |
| `key_interact` | 8 | | `key_item_slot10` | 39 | |
| `key_item_next` | −16 | | `key_ui_quick_drag` | 225 | 229 |
| `key_item_prev` | −8 | | `key_takescreenshot` | 59 | |
| `key_replayedit_open` | 68 | | | | |

`key_drop_item` and `key_drink_potion` have **no keyboard default** — they are never assigned
and the rebinding UI does not list them, which suggests they were bound to something that no
longer exists in the input layer. Not verified in game.

### Gamepad defaults

`key_up` 47, `key_down` 42, `key_left` 39, `key_right` 40, `key_use_wand` 48,
`key_spray_flask` 48, `key_throw` 26, `key_kick` 24 (+17), `key_inventory` 16,
`key_interact` 23, `key_drop_item` 25, `key_drink_potion` 26, `key_item_next` 20,
`key_item_prev` 19, `key_replayedit_open` 25, `aim_stick` 22, `key_ui_confirm` 23,
`key_ui_drag` 24. The item slots, screenshot and quick-drag are unbound.

### The rebinding screens

Two screens, `0x006d33f0` (keyboard) and `0x006d4720` (gamepad), bound to the two config
sections. The keyboard one lists 25 of the 30 actions and hides `key_item_slot9`,
`key_item_slot10`, `aim_stick`, `key_ui_confirm`, `key_ui_drag`; the gamepad one lists 26 and
hides `key_item_slot9`, `key_item_slot10`, `key_takescreenshot`, `key_ui_quick_drag`. Each has
a "reset all" row that regenerates the defaults and copies them over wholesale.

The displayed name is resolved **through the translation system**, and on capture the engine's
own localised name for the key just pressed is written straight into `primary_name`. So after a
rebind those fields hold a display name, not a translation key — the two roles are overloaded in
the same field, which is worth knowing before you parse the file yourself.

## What is modder-reachable

| mechanism | reachable? | how |
|---|---|---|
| the ~60 visible options | **No** | no Lua binding reads or writes the config struct |
| key bindings | **No** | you can *read* input (`InputIsKeyDown`, `InputIsJoystickButtonDown`, `InputGetJoystickAnalogStick`) but not rebind or enumerate the player's bindings |
| a mod's own settings | **Yes** | `ModSetting*`, persisted to `??S00/mod_settings.bin` (with any legacy `mod_settings.xml` deleted on load) — a different file from `config.xml` |
| magic numbers | **Yes** | `ModMagicNumbersFileAdd`, and `magic_numbers.xml` itself is data |
| translations | **Yes** | `ModTextFileGetContent`/`SetContent` on `data/translations/common.csv`; duplicate keys later in the file win, so appending is enough |
| the config file itself | **No** | `ModTextFile*` is gated to the `data/` and `mods/` prefixes, so `??USR/config.xml` is out of reach — the call logs and does nothing |

The practical consequence: the game settings are a closed C++ struct, but **mod settings,
magic numbers and translations are all open**, and those three cover most of what a modder
actually wants to change about the game's presentation.

More on the practical side: [ui-modding.md](ui-modding.md).