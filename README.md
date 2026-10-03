# Noita Lua API — behaviour reference

Everything the game's own one-line API signatures do not tell you, recovered by
disassembling `noita.exe`.

Noita ships no modding documentation beyond a usage string per function, embedded in the
binary and used to build error messages. Those strings are accurate about shape and silent
about behaviour. This is the behaviour: what the API does when arguments are missing or the
wrong type, how many arguments it really requires, what state it shares with other mods, and
where the traps are.

It is aimed at people writing mods, and at anyone reimplementing the API.

## Start here

| page | what is in it |
|------|---------------|
| [docs/how-the-api-works.md](docs/how-the-api-works.md) | The rules that apply to all 375 functions. **Read this first** — it changes how you write mod code. |
| [docs/api-reference.md](docs/api-reference.md) | All 375 functions: enforced arity, per-argument coercion, substituted defaults, return arity, runtime messages, shared globals. |
| [docs/family-notes.md](docs/family-notes.md) | Behaviour that holds across a family — `Entity*`, `Component*`, `Gui*`, `Physics*`, `Mod*` and the rest. |
| [docs/gui-options.md](docs/gui-options.md) | **What every `GUI_OPTION` value does**, as bit positions and effects — and which ones are no-ops. |
| [docs/gui-defaults.md](docs/gui-defaults.md) | The hardcoded values: which sprites, font and sounds the GUI uses, every default argument, and the numeric constants. |
| [docs/gui-cookbook.md](docs/gui-cookbook.md) | Reimplementing the GUI: the coordinate maths, the per-frame sequence, and the formulas, worked through. |
| [docs/gui-internals.md](docs/gui-internals.md) | The GUI object model, the widget-id system and the id-keyed state map — the data structures behind the behaviour. |
| [docs/gui-layout.md](docs/gui-layout.md) | The layout engine: the layout stack, the cursor algorithm, layers, scroll containers, auto-boxes and tooltips. |
| [docs/gui-bugs.md](docs/gui-bugs.md) | The bugs and sharp edges, with evidence strength marked — wrong values, memory-safety holes, silent no-ops, leaks. |

### The game's own interface

The pages above are about the GUI *API* — what a mod can draw with. These are about the interface
the game draws for itself, recovered from the executable.

| page | what is in it |
|------|---------------|
| [docs/game-ui.md](docs/game-ui.md) | **Where the game's UI lives.** The four address bands, the four row widgets every menu is made of, how each screen was identified without symbols, and the 40 UI tunables in `magic_numbers.xml`. |
| [docs/ui-screens.md](docs/ui-screens.md) | **Screen by screen.** The HUD bars and their formulas, the wand inventory, the perk row, the main menu, world select, progress, game over and the replay editor — plus the bugs that are in them. |
| [docs/ui-settings.md](docs/ui-settings.md) | **The options screen and the config file.** All ~60 options with their label, config key, struct offset and range; the `config.xml` format; the 30 key bindings and their defaults. |
| [docs/ui-modding.md](docs/ui-modding.md) | **What you can actually do to it.** What is open, what is closed, and what mods in the wild manage — with an approach table and a robustness rating. |

| [docs/enums.md](docs/enums.md) | Enum and constant tables, and why their values are not in the executable. |
| [docs/undocumented-functions.md](docs/undocumented-functions.md) | The eight functions the game never documents, what they actually return, and the hardcoded date tables inside one of them. |
| [docs/api-registration.md](docs/api-registration.md) | How the API is installed, the thirteen other registrars, and the documentation generator the game ships. |
| [docs/gun-actions.md](docs/gun-actions.md) | **`RegisterGunAction` and the wand-action record**: what each of its 65 arguments does, the defaults, the registry's overwrite and miss behaviour, and the traps. |

### World generation

How a Noita world is built, recovered from the executable. The parameters live in ~130 XML
attributes spread over five element types, not in a settings struct.

| page | what is in it |
|------|---------------|
| [docs/worldgen.md](docs/worldgen.md) | **The pipeline.** The world is 70 x 48 chunks of 512 x 512 cells; the biome map is a Lua-driven PNG; the two phases and the 24 ordered generation steps. |
| [docs/worldgen-biomes.md](docs/worldgen-biomes.md) | **The biome data model.** All 87 `Biome` fields with compiled-in defaults, `BiomeModifiers`, `RandomColor`, and the three topology types. |
| [docs/worldgen-terrain.md](docs/worldgen-terrain.md) | **How a pixel becomes a material.** The three generators, the cave noise, and the ore-placement algorithm in full pseudocode - including that `material_index` is a sort key and the XML order is irrelevant. |
| [docs/worldgen-wang.md](docs/worldgen-wang.md) | **Wang tiles.** The template PNG format, the herringbone run-length rule, the four planes, and the colour-to-Lua-function table. |
| [docs/worldgen-pixelscenes.md](docs/worldgen-pixelscenes.md) | **Pixel scenes, vegetation, backgrounds, decor.** The authored-room format, the three tree-placement variants, and the chunk-boundary containment test. |
| [docs/worldgen-reference.md](docs/worldgen-reference.md) | Addresses, struct offsets, enums, the data-file inventory, and a reimplementation checklist. |

## The short version

- **Arguments are never type-checked.** A string where a number belongs becomes `0`; there is
  no `luaL_check*` or `lua_error` anywhere in the API, so `pcall` will not save you.
- **Too few arguments and the function does nothing.** It logs an error and returns no values,
  so the mistake surfaces one frame away as `nil`.
- **A missing string argument is replaced by a hardcoded string** — often the function's own
  signature text — after logging a type complaint.
- **A repeated error is logged only once**, so a per-frame bug looks like a single event.
- **Ids and `gui` state are global, not per-mod.** A widget's id is its literal value when the
  id stack is empty and a hash chain when it is not; widget hover, animation and scroll state
  live in a per-`Gui` map keyed by that id and never pruned. Two mods sharing an id share state,
  and only one widget per frame can be hovered.
- **The GUI never unwinds itself.** An unbalanced `GuiLayoutBegin*` or `GuiIdPush` survives into
  the next frame. Several `*End` functions will read out of bounds rather than complain.
- **There is no hidden API.** Thirteen other functions call `lua_setfield`, but all of them
  install onto engine objects rather than the global environment.
- **The game's own UI is the same API you have.** It is C++ calling the same widgets with
  pre-baked positions, not a privileged renderer. It is also **entirely closed** to mods: there
  is no way to read a widget's rectangle, add a settings tab, or move anything vanilla.
  [docs/ui-modding.md](docs/ui-modding.md) has the open/closed breakdown.
- **World generation is data, not settings.** There is no worldgen parameter struct and no
  worldgen screen. The biome map is a PNG generated by `data/scripts/biome_map.lua`, the cave
  shapes are wang tile images whose *colours are instructions*, and what a pixel becomes is
  decided by an ordered list of value ranges in `<MaterialComponent>` - where `material_index`
  is the sort key, so the order you write them in does not matter.

## Scope and caveats

- Derived from one Steam build of the 32-bit `noita.exe` by disassembling all 375 API
  functions. **Addresses are specific to that build and will move on update.** Function names,
  arity and behaviour are the durable parts; treat any address as a pointer into one build.
- The enum tables (`GUI_OPTION`, `GUI_RECT_ANIMATION_PLAYBACK`, key codes, cell materials) are
  game data, not code, and cannot be recovered statically. [docs/enums.md](docs/enums.md)
  explains how to get their real values.
- 8 of the 375 functions are undocumented even by the binary; they are listed explicitly in
  the reference, and [docs/undocumented-functions.md](docs/undocumented-functions.md) says what
  they actually do — three of the eight are dead or misleading, and one contains hardcoded
  date tables that stop matching in 2040.
- The UI pages are recovered from **string references and sprite paths**, because the screens
  have no symbols. Every screen's attribution rests on at least two independent signals, but the
  function that draws the wand stat panel timed out the decompiler and its layout arithmetic is
  not recovered. Individual claims are marked with their evidence strength where it matters;
  [docs/ui-screens.md](docs/ui-screens.md) ends with what is *not* known.
- The world-generation pages are recovered from the **config-system field registrations** (the
  executable carries every field name, type and in-source description as string literals) plus
  the constructors that write the compiled-in defaults, cross-checked against the shipped XML.
  Where a field could not be paired to an offset with confidence the page says `?` rather than
  guessing. The enum integer values are game data and are not recovered;
  [docs/worldgen-reference.md](docs/worldgen-reference.md) lists what is missing.
- The signatures in this reference are quoted from the game, so any typos in them are the
  game's. Where the enforced arity disagrees with the signature, both are shown.

## Not included

Editor autocomplete stubs (EmmyLua/LSP) are not here; the community maintains those
separately, and this reference is about behaviour they cannot express.
