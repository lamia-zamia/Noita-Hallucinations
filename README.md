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
| [docs/gui-internals.md](docs/gui-internals.md) | The GUI object model, frame lifecycle, the widget-id system and the id-keyed state map, and a map of all 41 `Gui*` functions to what they call. |
| [docs/gui-layout.md](docs/gui-layout.md) | The layout engine: the layout stack, the cursor algorithm, layers, scroll containers, auto-boxes and tooltips. |
| [docs/gui-bugs.md](docs/gui-bugs.md) | The bugs and sharp edges, with evidence strength marked — wrong values, memory-safety holes, silent no-ops, uninitialised data, leaks. |
| [docs/enums.md](docs/enums.md) | Enum and constant tables, and why their values are not in the executable. |
| [docs/undocumented-functions.md](docs/undocumented-functions.md) | The eight functions the game never documents, what they actually return, and the hardcoded date tables inside one of them. |
| [docs/api-registration.md](docs/api-registration.md) | How the API is installed, the thirteen other registrars, and the documentation generator the game ships. |

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
- The signatures in this reference are quoted from the game, so any typos in them are the
  game's. Where the enforced arity disagrees with the signature, both are shown.

## Not included

Editor autocomplete stubs (EmmyLua/LSP) are not here; the community maintains those
separately, and this reference is about behaviour they cannot express.
