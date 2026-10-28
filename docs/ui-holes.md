# Holes in the pause menu: what is left after the options dead end

Companion to [ui-modding-2.md](ui-modding-2.md). That page says the pause menu cannot be
extended. This page covers the gaps around it — and starts with the one that looked like a hole,
was written up as one, and **turned out not to be**. The negative result is the most useful thing
here, because it closes off a whole class of idea.

---

## `GuiOptionsAdd` does not reach vanilla widgets

In-game, `GuiOptionsAdd(gui, 2)` inside
`ModSettingsGui` affects **only the mod settings widgets drawn after that call**, and nothing
outside. The reason is a one-line fact that is easy to miss:

**The frame-wide option word on the `gui` object is only ever read by the Lua API wrappers.**

The widget builders do not read the option word off the `gui` at all. They take it as an
*argument*:

```c
/* 0x008245d0:68-69 — the option word is param_5/param_6 */
local_58 = param_5;
local_5c = param_6;
```

Something has to merge `gui+0x08` and `gui+0x10` into that argument. That something is the Lua
callable layer, and only that layer:

```
low  = (gui + 0x10) | (gui + 0x08)      # next-widget | frame-wide
high = (gui + 0x14) | (gui + 0x0c)
```

All **20** merge sites in the binary are in the `0x007d`–`0x007e` band — `007dac30`, `007dce60`,
`007dd400`, `007dd900`, `007ddfb0`, `007de5e0`, `007dec70`, `007df250`, `007df8d0`, `007e0db0` and
the rest. **Zero** are in the game's own UI code. The game passes literals:

```c
/* 0x006e3410 — every pause-menu button */
FUN_008245d0(..., 0x8010200, ...)      /* new game   */
FUN_008245d0(..., 0x8010000, ...)      /* options    */
FUN_008245d0(..., 0x8010000, ...)      /* and four more */
```

So `gui+0x08` is a **Lua-only channel**. Setting it changes the options of the *next widget drawn
through the Lua API* — which, inside `ModSettingsGui`, is your own settings list. It cannot reach a
widget the game drew with a constant. The game is not bypassing a check; it is on the other side
of an API boundary, and the option word never crosses it.

The general lesson, and it is the useful part: **the per-widget state the wrapper merges is
Lua-private.** The frame-wide options, the pending options, the z, the previous-widget record.
The game reads none of them, because the game does not go through the wrapper. (The mouse-claim
byte is per-object and the game's builders do read it; the layout stack is read by every builder
— see [the layout-stack hole](#hole-the-layout-stack-is-not-lua-private-and-vanilla-widgets-obey-it).)
That cuts both ways — it is why the options route fails, and it is also why the previous-widget
record is a clean getter for *your own* widgets and nothing else.

**One thing that did survive the correction:** the option *polarity* is still worth having, because
`GUI_OPTION.NonInteractive` is bit 2 and it is the opposite of what the documentation said. The hit
test is `(bit2_clear || bit3_set)`, and the frame seeds the word with `1` precisely so widgets are
interactive by default. See [gui-options.md](gui-options.md). And bit 9, previously listed as a
dead gap, is live in the widget-state lookup and is set on every pause-menu button — a real
defect in the gap list, though not a hole you can walk through.

## The `gui` handle is a storable capability — now proven

Disassembled, so this is settled rather than inferred. `FUN_006c6290` is 0x8e bytes of
lazy-global accessor:

```asm
006c62bf  MOV ESI, dword ptr [0x012075d0]   ; the singleton
006c62c5  TEST ESI,ESI
006c62c7  JNZ 0x006c6350                    ; built already -> return it
006c62cd  PUSH 0x90
006c62d2  CALL [operator_new]
006c62e7  PUSH 0x100fa84                    ; "Menu gui"
006c6309  CALL 0x0081dbb0                   ; the one Gui constructor
006c6314  MOV dword ptr [0x012075d0],ESI     ; keep it
006c6350  ...
006c6354  CALL 0x0081dd50                   ; FUN_0081dd50(gui, 3) -- EVERY call
006c6359  MOV EAX,ESI
```

**Allocated once, reused forever, and the string it is built with is literally `"Menu gui"`.**

Three things follow, and they are the most useful facts on this page:

1. **The `gui` you get in `ModSettingsGui` is the base menu's own object** — the same one the pause
   menu, the main menu and every submenu draw with, because all of them are invoked from the same
   place with this same pointer.
2. **The handle is stable across frames.** Stashing it in a Lua global is sound. This was the open
   question that gated the whole capture plan in [ui-modding-2.md](ui-modding-2.md).
3. **`FUN_0081dd50(gui, 3)` — a frame start — runs on every invocation, from inside the menu
   runner, before any Lua callback.** The game opens its own frame on that object. A mod drawing
   on it later in the frame is drawing into a frame already in progress, which is exactly why
   `ModSettingsGui` works. The corollary: **a mod must not call `GuiStartFrame` on that handle**,
   or it will reset the option and claim state the game is midway through.

The handle validator cannot tell a live `Gui` from a destroyed one — no vtable probe, no generation
counter, no type tag (`0x0078c030`, nineteen lines) — but with the singleton proven, that
stopped being the interesting risk. The object outlives the frame.

One caveat: `DAT_01207619`, the front-end teardown flag, zeroes this global along with the game-over
one (`006c6018`, `006c6022`). Both are rebuilt on demand afterwards. Nothing in the decompiled
corpus writes that flag, so when it fires is not visible — if you stash the handle across a
front-end session, expect to re-acquire it.

## The app state machine — one state, and it is not a free screen

The other open question, also disassembled. `FUN_006c5ea0` returns `ESI`, initialised to `2` and
reloaded once from `DAT_012076a0`. The whole mechanism is one instruction:

```asm
006c5fe1  MOV EAX,[0x012076bc]      ; menu stack BEGIN
006c5ff3  MOV [0x012076c0],EAX      ; stack END := BEGIN  ->  the stack is EMPTIED
```

**State 1 is the frame in which the previous screen asked to be popped**, and the stack is collapsed
to make that happen. It is a one-frame transient, not an idle mode with no interface — and the base
menu draws on exactly that frame, because the runner then sees an empty stack.

`DAT_012076a0` is written only by C++ screen-transition code (`FUN_006c5e40`, `006cc030`,
`006ccc10`, `006d0da0`, with the values 0, 1 and 2), so nothing reachable from Lua is shown to
influence the returned state.

## The wand menu: a real trigger, but it does not pause the game

The idea was to force the wand-edit panel open and inherit its pause. The panel's trigger is fully
recovered, and it is not what you would guess — **it is the alternate-fire modifier over a wand
slot, not "grabbing an invalid wand"**:

```c
/* 0x00b7ee50:219-227 — the whole gate */
iVar5 = (**(code **)(DAT_01221bc0 + 0x30))();          /* current input state */
if ((((iVar5 + 0x18) < 0xe2 || (((iVar5 + 0xc)+0x1c >> 1) & 1) == 0)) &&
     ((iVar5 + 0x18) < 0xe6 || (((iVar5 + 0xc)+0x1c >> 5) & 1) == 0))) {
    if (DAT_012224fc != 0) {
        if (FUN_009e2c00(player) < FUN_00b5d940(wand)) {   /* cards held < deck size */
```

`0xe2` and `0xe6` are **226 and 230 — `Key_LALT` and `Key_RALT`**, confirmed against
`data/scripts/debug/keycodes.lua:217,221`. So the panel opens when you hold Alt over a wand slot
whose deck needs more cards than you are holding. It then appends a card, logging `"Added "`.

**Why it does not get you a paused screen:** the panel sets `DAT_0122250a`, which gates the wand
row and the stats card inside the *player UI* — `0x00b7d8d0:657` calls
`FUN_00b788e0` with it, and `:733` calls `FUN_00b5ed80`. This is the HUD's own path, on the HUD's
own `Gui`, alongside the health bars. It is a paused *state* in the sense that you stop shooting,
but it is not the menu stack, nothing is pushed onto it, and the base menu never stops drawing.
`GameIsInventoryOpen()` reads the neighbouring flag and is the supported way to see it.

The genuine limitation: **it is the same per-`Gui` island problem.** The wand panel is C++ on the
HUD's object, so you cannot block *it* either — you can only add to what you draw yourself. Still
worth knowing that the trigger is a plain key test on a plain component count, because both of
those are things a mod can move: `FUN_00b5d940` reads the deck size, `FUN_009e2c00` the held cards.

## Hole: the options screen draws a per-mod row *above* your callback — and that row is the only live one

There is a seam in the options screen that the previous sections missed, and it is the closest thing
to an input block that exists.

### What the screen actually does, in draw order

`0x006d5620` is 2472 lines and draws **one tab at a time**, but the Mods tab has an
internal structure that matters:

| lines | what |
|---|---|
| 235–460 | the tab strip (`0x35a7` and friends); a click writes the active tab to `DAT_012076ac` |
| 460–1206 | section content, gated on the active tab |
| **1208** | `if (DAT_012076ac != 5) goto end` — **everything below is the Mods tab only** |
| 1214 | a **hidden zero-size button**, id `0x3615` |
| 1247 | widget `0x3617` |
| 1253–1356 | the per-mod loop; **`ModSettingsGui` is called at 1341** |
| 1357–1379 | end of the mods section |

So when your callback runs, the tab strip is already drawn and the rest of the screen is still to
come. That is the ordering that matters.

### The hidden button is the interesting part

```c
/* 0x006d5620:1214 */
cVar3 = FUN_008277f0(param_1, 0x3615, 0);
if (cVar3 != '\0') {
    piVar5 = FUN_008365e0(...);      /* load every mod's settings.lua */
    ...iterate all active mods, FUN_00838900(..., 3, mod) ...
}
```

and the button itself (`0x008277f0`) is a widget with **no rectangle at all** — both
position values are `0x800000`, and it is built with option `4`, which is `NonInteractive`:

```c
local_14 = 0x800000;  local_10 = 0x800000;
FUN_0081b090(..., param_1, param_2, &local_14, &local_14, 0, 4, 0, &local_5);
```

**That is the game using a mechanism the Lua API does not expose.** A widget deliberately
constructed to be non-interactive and undrawable, used as a one-shot latch. It proves the engine
has an internal idiom for "invisible, inert, clickable-never" — and it is reachable only from C++,
because the `0x800000` sentinel and the option word are compile-time constants in that call.

### Why this still does not hand you the screen

Be clear-eyed about what it buys:

- **It does not block the pause menu.** The pause menu is drawn on the *same* `Gui` object (the
  menu stack is dispatched from `006e3410` with the singleton `this`), but it is only drawn when
  the options screen is not, so an input claim made here never coexists with a pause-menu button.
  Layout state is different — see the next-but-one section.
- **It does not give you the tab strip.** Lines 235–460 run before you, on the same `gui`, and they
  are drawn with literal option words, so nothing you set afterwards reaches them.
- **It does not survive the state crossing by value.** The handle cannot be copied from
  `settings.lua`'s state into `init.lua`'s — but `init.lua` can make the settings state *run*,
  on any gameplay frame, and the handle can be used there. See
  [using the handle outside the Mods tab](#using-the-handle-outside-the-mods-tab).

What it *is*: proof that the "everything on a `Gui` is Lua-private" rule has an exception on the C++
side, and therefore that a C++-side input block is a thing this codebase does. Which is an argument
for the DLL route below rather than against it.

### Using the handle outside the Mods tab

**Partly tested.** The per-frame trigger below is confirmed in game (a `ModSettingSet` with a changing value from `init.lua` makes `ModSettingsUpdate` run every frame; setting a value from inside `ModSettingsUpdate` does not re-trigger it). Using the menu handle from there is not yet tested. The handle cannot travel between states, but
code in the settings state can be made to run on demand during gameplay.

**The handle cannot be copied.** It is a light userdata. The whole executable has two
`lua_pushlightuserdata` sites (`0x00f07798` is referenced from `GuiCreate`, `0x007d9920`, and
the options screen, `006d5620`, and nowhere else), every `Gui*` function rejects any other type
(`lua_type(L,1) == 2`), and the cross-state channels (`GlobalsSetValue`, `ModSettingSet`) carry
strings and numbers. `load`/`loadstring` are removed, so there is no bytecode route. A stash in a
`settings.lua` global stays in that state.

**But the settings state can be entered on demand during gameplay.** It is the state `ModSettingsUpdate` runs
in, and `ModSettingsUpdate` has four dispatch sites (`0x00838900` ← `00834620` mod init,
`006d5620` Mods-tab first draw, `00838d40` unpause, `00835c90`). The last one is a poll:

```c
/* 0x00835c90, called from 0x006b59d0 at the tail of 0x006b26f0 */
if (settings_dirty && mods_exist) {
    for each mod: ModSettingsUpdate(2);          /* MOD_SETTING_SCOPE_RUNTIME */
    settings_dirty = 0;
    save "??S00/mod_settings.bin";               /* 0x00837f10 */
}
```

`settings_dirty` (`0x01207efc`) is set by `ModSettingSet`, `ModSettingSetNextValue` and
`ModSettingRemove` whenever the stored value actually changes (`007e7df0.c:191`). Those are
callable from `init.lua`. So:

1. Open Options → Mods once; `ModSettingsGui` stores the menu handle in a `settings.lua` global.
   There is no way round this step: the options screen is the only code that hands out the
   handle, it does so only on the Mods tab (`006d5620:1208`), only for a mod whose
   `ModSettingsGuiCount()` is above 0, and no Lua API pushes a screen onto the menu stack. Whether
   one visit from the main menu carries into a run depends on the settings state and the menu
   singleton both surviving the transition — untested.
2. In `init.lua`, call `ModSettingSet("yourmod.tick", n)` with a changing `n` on any gameplay frame you
   want the settings state to run.
3. At the end of that same frame, your `ModSettingsUpdate(2)` runs with the stored handle.

Costs: every trigger also runs every other mod's `ModSettingsUpdate(2)` (applying their pending
runtime settings early) and rewrites `mod_settings.bin`. Trigger only on the frames you need.

**When the poll runs.** At the end of every frame, paused or not. A paused frame runs
`OnPausePreUpdate` for every mod, then the pause menu (`0x006e4fe0` → `0x006e3410`), then the
poll. So `init.lua` can set the flag from `OnPausePreUpdate` and your `ModSettingsUpdate(2)` runs
at the end of the same frame, after the menu has drawn — what it pushes affects the next frame's
menu. That works on every pause-menu frame, Options included, at the cost of one settings-file
write per frame you trigger. A `ModSettingsUpdate` cannot re-arm itself: the flag is cleared
after the dispatch loop.

**A stable offset for the pause menu.** Outside the options screen nothing pops on your behalf,
so a frame you push stays until you pop it. A Lua layout frame drifts if left alone: the pause
menu opens its own vertical layout relative to your cursor (`006e3410:177`) and closing it folds
the menu's size into a horizontal or vertical parent (`0081e710`). Popping and re-pushing it on
every paused frame (triggered from `OnPausePreUpdate`) resets the cursor and lets you move the
menu live, one settings write per frame. For a menu that holds still without per-frame work, use a **mode-8** parent: it is skipped by that fold — and the only Lua call that
pushes one is `GuiBeginScrollContainer`. Left open on the menu handle, it gives:

- every menu widget offset by the container's content origin, with no drift;
- the container's hit rectangle as the only place menu buttons can be hovered (the button
  builder's scroll test, `008245d0.c:106-121`), and probably its clip rectangle — so make it
  screen-sized;
- a scroll offset frozen at 0, because the offset is only measured at End.

Push it during gameplay; pop it with `GuiEndScrollContainer` (a matched pair, so safe) and push
again to move it. The natural trigger is `OnPausedChanged(false)` in `init.lua` → `ModSettingSet`
→ the poll at the end of that unpause frame, which positions the menu for the *next* pause at the
cost of one settings-file write per unpause.

Before popping, check that your container is still the top one: right after pushing, draw
`GuiText(gui, 0, 0, " ")` and remember its `x, y`; before popping, draw it again and compare. A
mode-8 frame never advances, so the two match. If the menu singleton was rebuilt, the same
address may now be a fresh object with no layout at all, the probe reports a different position,
and you must push without popping.

### The one ordering fact that would still matter

If you ever do get a `Gui` handle into a state that runs before the options screen's layout, then
everything from line 1208 onward is drawable-after-you: the hidden button, `0x3617`, and the entire
per-mod loop. Only the tab strip and the earlier sections would be untouchable. That is the shape of
a partial input block, and it is the closest this API gets.

## The suppression pattern: a non-empty menu stack makes the game's own screens bail

This is the closest the game comes to an officially-supported "make my own screen instead", and it
is the mechanism behind the pattern people reach for. It works — but the trigger is C++-only.

### The rule, from the game-over screen

`0x006e51e0` is the game-over screen. It creates its own `Gui` (`"Game over gui"`,
`DAT_01207678`, line 127), starts its frame (141), and then at **line 145**:

```c
DAT_0120761b = 0;
if (DAT_012076bc != DAT_012076c0) {            /* stack not empty */
    (**(code **)(DAT_012076c0 + -4))(puVar7,uVar6);   /* draw whoever is on top */
    goto LAB_006e69e1;                          /* and stop */
}
```

**If any screen is on the menu stack, the game-over screen draws nothing.** The same guard is in the
base pause menu at `006e3410.c:91`. So "push a screen" is genuinely a screen-suppression mechanism,
not just navigation — and it is the only one in the binary.

### The stack, fully mapped

`DAT_012076bc` is the stack base, `DAT_012076c0` the top; pushes go through `FUN_008a5dd0`, pops are
`DAT_012076c0 = DAT_012076c0 - 4`. Complete writer list, all of it C++:

**Pushers (12 distinct screens):**

| screen | function | pushed from |
|---|---|---|
| new game | `FUN_006ceb00` | `006e3410:333`, `006e51e0:713`, `006dc130:1230` |
| options | `FUN_006d5620` | `006e3410:370`, `006ceb00:791,818`, `006dc130:1298` |
| mods list | `FUN_006dc130` | `006e3410:411` |
| save slots | `FUN_006d15f0` | `006ceb00:436` |
| game over | `FUN_006e51e0` | via `006cc450:52` → `FUN_006cc5f0` |
| credits / stats / etc. | `FUN_006ce940`, `FUN_006cdf30`, `FUN_006ce460`, `FUN_006cda00`, `FUN_006cd850`, `FUN_006ccdc0`, `FUN_006e33d0`, `FUN_006e33f0`, `FUN_006cc5f0` | `006ceb00`, `006dc130`, `006e3410` |

**Pops:** `006caab0:422`, `006cc450:61`, `006cc5f0:149`, `006d33f0:338`, `006d4720:349`.

**Dispatch:** `006e3410.c:779` and `006e51e0.c:146` call `(*(DAT_012076c0 - 4))(...)` — a raw
function pointer from the stack, invoked with `(this, unused)`.

### Why it is closed anyway

`FUN_008a5dd0` is called from **exactly 8 functions, and none of them is a Lua wrapper.** Searching
the whole `0x007d`–`0x007e` Lua band and the `0x0082` GUI subsystem: finds zero hits. Every push installs
a hardcoded C++ function pointer from the table above, and the dispatch at `006e3410:779` calls it
as raw code. There is no API that pushes, and no way to make the stack non-empty from Lua.

## The translation table and the `$` prefix

Every widget label goes through one transform, and it is the whole localisation contract.

### The table

`DAT_01207c38` is the table of languages: one 180-byte (`0xb4`) record per language column of
`data/translations/common.csv` (plus `common_dev.csv`), each holding that language's strings in
24-byte (`0x18`) `std::string` slots at the pointer stored at `+0xa8`. A separate map
(`DAT_01207c44`, a `std::map<std::string, int>`) takes a key to a string index. `FUN_0084ab80`
selects the active language by comparing each record's name against a caller-supplied string and
stores the matching record index in `DAT_01207c58`; `FUN_0084acb0` / `FUN_0084ace0` then look the
key up in the map and return `base[index]` from the active language, falling back to the first
language's string when the active one is empty. The population path is `FUN_0084a700`
(`FUN_008496d0` parses a file); `FUN_0084a510` is `Text_RegisterLanguage` (error text
`"Text_RegisterLanguage() error - Missing translation file: "`). A key that already exists in the
map keeps its index and has its strings overwritten, so a later row replaces an earlier one.

The `index * 0xb4 + base` pattern appears at 19 sites, including the pause menu (`006c3ec0`,
`006e6a00`, `006c7100`, `006c7c30`), the HUD (`00b7d8d0`, `00b788e0`) and the Lua wrappers.

### The `$` rule

`FUN_0084b3a0` is the transform every widget label goes through:

```
if label starts with '$':  return FUN_0084acb0(label)   // translation lookup, key = text after '$'
else:                      return label                  // copied through unchanged
```

**A label that starts with `$` is looked up in the translation table; a label that does not is
used as written.** That is why the game passes `"$menu_newgame"`-style literals everywhere, and
why `GuiText(gui, 0, 0, "Play")` from Lua shows `Play`. When the key is missing from the map, the
lookup returns the original `$key` text if the global flag at `DAT_01204baa` is set and an empty
string otherwise.

A mod's `data/translations/common.csv` can therefore add and override keys. It is a text lever
only: it cannot express a position, a size, or an input flag. Mods in the wild use plain
identifiers (such as `action_test_spell`) in their CSVs, because those are looked up by card or
action name rather than through the `$` label path.

### The arbitration rule

Every interactive builder gates on the same claim byte, and this part is verified — **9 readers,
3 writers, 1 reset**, all per-`Gui`:

- read: `008245d0.c:150,155` (button), `00823580.c:259` (image button), `00825cb0.c:210,215`
  (text input), `00820bf0.c:109`, `008205a0.c:89`, `008221d0.c:82`, `00824e70.c:164`,
  `00826b50.c:101`, `00820930.c:41`
- write: `008245d0.c:189`, `00823580.c:288`, `00825cb0.c:251`
- reset: `008187d0.c:1175` (NewFrame only)

Input ownership on any single `Gui` is **first widget drawn under the cursor wins, for the whole
frame**. No priority, no z-order tiebreak, no early release. Two `Gui` objects never interact, which
is the wall — but the rule is a draw-order race, and draw order is what a mod controls.

## Why the game never consults Lua-drawn input state
Following the dead end one step further, because it explains the shape of the whole problem. The
mouse-claim byte is at `*(gui+0x88) + 0x1fd` — on the object, and cleared only by that object's own
`NewFrame`. A Lua `GuiButton` on your own object sets it, and every later widget on *your* object is
suppressed. But the pause menu's widgets live on the menu's object, with its own claim byte, and
the game reads that one. Two objects, two flags, no interference in either direction.

This is the structural reason the "overlay and steal the mouse" approach cannot work, and it is not
a missing feature — it is the API boundary again. Everything that governs input is per-object Lua
state, and the game is not on the Lua side of it.

## Pausing the game yourself: not reachable

Not through the modding API. Two things look like they would do it; neither does.

There is no pause function. Sweeping all 375 API names for `pause|freeze|stop|halt|sim` finds
nothing relevant (the only substring hit is `GamePosToPhysicsPos`). The pause is owned by the menu stack.

- **`mPauseSimulation` and `mGuiDisabled`** are fields of the cheat-style game-effect component,
  beside `mPlayerNeverDies`, `mFreezeAI` and `mFogOfWarOpenEverywhere` — exactly the input block
  the API lacks. Both are dead: **zero** references from outside their own generic field-name table.
  Each of the twelve functions that mentions them pulls in 32–36 field names *including*
  `mPauseSimulation`, which is the signature of a schema accessor, not a consumer. A schema with
  nothing reading it.
- **The `GAME_EFFECT` enum is fully recovered** (`0x0100c6ec`–`0x0100ccd4`: 85 names, from
  `ELECTROCUTION` to the `_LAST` terminator) and contains
  no pause member. Closest relatives are `NO_WAND_EDITING`, `NO_HEAL`, `NO_DAMAGE_FLASH` and the
  `PROTECTION_*` family — all gameplay effects, applied via `GetGameEffectLoadTo`.

The nearest reachable thing is freezing the *player* with
`EntitySetComponentsWithTagEnabled`, which stops the player and not the world. The world keeps
simulating behind your menu. There is no supported way to stop it.

---

## Hole: the layout stack is *not* Lua-private, and vanilla widgets obey it

**Static analysis only — not yet tested in game.** The "anything stored on the `gui` object is
Lua-private" rule above holds for the option word, z and the previous-widget record. It does
**not** hold for the layout/layer stack.

- The game's own builders place every widget through the same function as Lua widgets,
  `0x0081e0d0` (called from the button builder at `008245d0.c:89`). It applies the top layout
  frame unless option `0x8000` is set. The pause-menu buttons pass `0x8010000` / `0x8010200`
  (`006e3410.c:257-549`) — bit `0x8000` is clear, so they are laid out by whatever frame is on top.
- The game's frame start (`0x0081dd50` → `0x008187d0`) never touches `state+0x1a0`/`+0x1a4`. A frame
  left open survives into every later frame on that object.
- The options screen and the pause menu share the menu singleton, so a frame left open while
  `ModSettingsGui` runs is still there when the pause menu draws.

### Why the obvious leak does nothing, and the one that should work

`ModSettingsGui` runs nested inside the game's own structure, and the game pops a **fixed count**
afterwards. On the Mods tab (`006d5620`):

| line | call | effect |
|---|---|---|
| 471 | `0081e480(…, 2, …)` | outer vertical layout **O** in the base layer |
| 1247 | `0081f870` (id `0x3617`) | mods-list scroll container: new layer **L**, mode-8 frame **F8** at the scrolled content origin, two clip/hit records |
| 1251 | `0081e480(…, 2, …)` | vertical layout **V** in **L** |
| 1270 / 1328 | `0081e480(…, 1)` / `0081e710` | per-mod row, balanced, closed *before* the callback |
| 1341 | `ModSettingsGui(gui, in_main_menu)` | your callback; stack is `B:[…, O]`, `L:[F8, V]` |
| after it | `00827540`, `0081eb50`, `0081e6e0` | id/scope pops and a vertical-spacing nudge — no layout pops |
| 1377 | `0081e710` | pop one frame from the top layer |
| 1378 | `00820060` | pop one frame (`0081e710`), pop the layer and one layer id (`0081e8b0`), pop both scroll records |
| 2419 | `0081e710` | pop one frame — meant for **O** |

Consequences:

- **A leaked `GuiLayoutBegin*` vanishes.** It sits in **L**, and `00820060` frees **L** with every
  frame in it. Harmless; the only visible effect is the rest of the Mods tab for that one frame.
- **A leaked layer or scroll container underflows.** `00820060` pops a frame *before* it pops a
  layer (`00820060.c:14-15`). With your layer on top and only one frame in it, the second pop runs
  on an empty vector. Do not do this.
- **The balanced leak** — once, guarded by a flag — is: *do not* close **V**; call
  `GuiLayoutBeginLayer(gui)` then `GuiLayoutBeginVertical` twice. The game then pops your two
  frames, your layer, its own scroll records and your layer id. **L** survives holding `[F8, V]`,
  and line 2419 pops **V** instead of **O**. What remains on top is **L** with **F8**: a mode-8
  frame, which offsets every later widget by the mods-list content origin and **never advances
  its cursor**. Every pop count matches, so nothing underflows; **O** leaks harmlessly below.

Predicted effect: from the next frame, everything on the menu `Gui` — the options screen and,
after you close it, the pause menu — is drawn shifted by the mods list's scrolled content origin,
hit boxes included, until the singleton is rebuilt. Scrolling the mods list before triggering
changes the offset. Applying it twice stacks a second shift.

**Undoing it (derived, not tested).** After *k* applications, the stack at callback time is
`B:[O]`, then *k* leaked layers each holding `[F8ᵢ, Oᵢ₊₁]`, then the live `L:[F8, V]`. To return
to a clean `B:[]` once the game's pops have run:

1. `GuiLayoutEndLayer` — drops the live **L** (it always exists here).
2. `GuiLayoutEnd` — the top layer is now either a leaked one (`[F8, O]` → `[F8]`) or `B` (`[O]` → `[]`).
3. Probe: draw `GuiText(gui, 0, 0, " ")` and read its `x, y` from `GuiGetPreviousWidgetInfo`.
   An empty `B` skips placement and reports exactly `0, 0`; a leaked `F8` reports its non-zero
   origin. If non-zero, `GuiLayoutEndLayer` and go back to step 2.
4. Rebuild what the game will pop: `GuiLayoutBeginVertical` (absorbs line 2419), then
   `GuiLayoutBeginLayer` and `GuiLayoutBeginVertical` twice (absorb 1377/1378).

Every pop lands on a non-empty vector: step 2 always has a frame to take, and step 3 only drops a
layer the probe has identified as leaked. Layer ids balance too (*k*+1 popped by you, one pushed,
one popped by the game). With *k* = 0 the sequence is a no-op.

Not established: whether the leaked layer id (`state+0x234`) also associates the menu with the
mods list's clip rectangle in the renderer. Test with a restart available; `Esc` still leaves
the pause menu if it ends up off-screen.

## What is still unproven

Much less than there was. The two functions that gated this page are disassembled and both are
settled: the menu `Gui` is a proven singleton, and the app state's third value is a proven
one-frame transient.

What remains:

- **`DAT_01207619`**, the front-end teardown flag that zeroes the menu `Gui` global. It has zero
  writers in the corpus, so *when* the singleton is dropped is not visible. It is rebuilt on
  demand, so a stashed handle is only at risk across a front-end session boundary.
- **`FUN_00b5d940` and `FUN_009e2c00`**, the deck-size and held-card readers in the wand-edit
  gate. Neither is decompiled, so the exact condition that makes a slot eligible is inferred from
  the comparison at `00b7ee50.c:227` rather than read.

## Where this leaves the original goal

Better than it was, and the improvement is concrete rather than speculative.

**What is now established:** you hold a handle to the pause menu's own `Gui` object, it is a
stable singleton, it survives across frames, and the game has already started a frame on it by
the time any Lua callback runs. That is the only game UI object the modding API ever receives,
and it is yours for the duration of a callback and — as far as the evidence goes — afterwards.

**What still does not work:** making that object's widgets non-interactive. The option word is
Lua-private and the claim byte is per-object; the game passes literals and keeps its own flag.
That wall is now understood rather than merely observed, which is worth something even though
it is still a wall.

**So the honest menu design** is: take the handle, draw on it during `ModSettingsGui` (or, after
it has been stored, from a `ModSettingsUpdate` you trigger — see
[using the handle outside the Mods tab](#using-the-handle-outside-the-mods-tab)), and accept that the vanilla rows remain live
underneath. Shape the panel so their click targets are covered by your own, and the leak stops
mattering. Everything else — pausing the game, adding a row, blocking a screen you did not draw —
is closed, and now closed with a reason.