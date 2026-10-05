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

The general lesson, and it is the useful part: **anything stored on the `gui` object is Lua-private
state.** The frame-wide options, the pending options, the z, the previous-widget record, the
mouse-claim byte. The game reads none of them, because the game does not go through the wrapper.
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

- **It does not block the pause menu.** That is a different `Gui` object on a different code path.
- **It does not give you the tab strip.** Lines 235–460 run before you, on the same `gui`, and they
  are drawn with literal option words, so nothing you set afterwards reaches them.
- **It does not survive the state crossing.** The handle that would let you exploit any of this
  still cannot get from `settings.lua`'s state into `init.lua`'s.

What it *is*: proof that the "everything on a `Gui` is Lua-private" rule has an exception on the C++
side, and therefore that a C++-side input block is a thing this codebase does. Which is an argument
for the DLL route below rather than against it.

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
- Whether the ordering of `OnPausePreUpdate` against the menu layout lets a mod draw on the menu
  `Gui` in the same frame it is read. Now that the handle is known to be a stable singleton, this
  is the last question between "you hold the pause menu's gui" and "you can use it".

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

**So the honest menu design** is: take the handle, draw on it during `ModSettingsGui` (or
`OnPausePreUpdate` if the ordering cooperates), and accept that the vanilla rows remain live
underneath. Shape the panel so their click targets are covered by your own, and the leak stops
mattering. Everything else — pausing the game, adding a row, blocking a screen you did not draw —
is closed, and now closed with a reason.