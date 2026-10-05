# Lua execution flow

How a line of Lua in a mod actually gets run: which interpreter state it lands in, in what
order scripts are loaded, how the game calls back into your code every frame, and what
happens to your code when it throws.

From one Steam build of the 32-bit `noita.exe`, imagebase `0x00400000`. Addresses move on
update; behaviour does not. Read [vfs.md](vfs.md) for the file half of the story and
[api-reference.md](api-reference.md) for the 375 registered functions.

## The short version

| question | answer |
|---|---|
| Which Lua? | **Lua 5.1**, dynamically linked from `LUA51.DLL` - and **LuaJIT** |
| How many interpreters? | **one `lua_State` per mod**, plus one for the game itself |
| Which libraries open? | six: base, `table`, `string`, `math`, `bit`, `jit` |
| Can a mod reach the FFI? | **only with `request_no_api_restrictions="1"` in its `mod.xml`** - the whole FFI is imported, but an ordinary mod's state never opens `package`, so `require` is `nil` |
| Which base globals are removed? | `load`, `loadfile`, `loadstring`, `gcinfo`, `collectgarbage` |
| How are callbacks found? | in the **globals table** - not a per-mod table |
| Can a mod error break another mod? | **no** - every call is individually `pcall`-protected |
| Can C++ start a Lua coroutine? | **no** - the imports are not even present |
| What order do scripts load in? | mods' `init.lua` in mod order, then three separate passes |

## The interpreter is not just Lua 5.1

`noita.exe` imports 171 symbols from `lua51.dll`. Among the libraries the game opens is
**`jit`** - LuaJIT, not stock Lua 5.1. That is worth knowing before you rely on anything
subtle: LuaJIT is 5.1-compatible with extensions and a different performance profile, and
some pure-Lua 5.1 idioms that a stock interpreter would accept are compiled differently.
The library table is at `0x00ffd5d0`, six `(name, import thunk)` pairs followed by a null:

| name | thunk for |
|---|---|
| `""` (empty - installs into `_G`) | `luaopen_base` `0x00dfcc0a` |
| `table` | `luaopen_table` `0x00dfcc10` |
| `string` | `luaopen_string` `0x00dfcc04` |
| `math` | `luaopen_math` `0x00dfcbfe` |
| `bit` | `luaopen_bit` `0x00dfcc16` |
| `jit` | `luaopen_jit` `0x00dfcbf8` |

`bit` and `jit` are the give-away: stock Lua 5.1 has neither. (The thunk addresses are not
4-byte aligned because they are the MSVC import stubs themselves - a 6-byte `jmp [IAT]` each -
not the functions, and nothing ever calls them by address; every opener call goes through the
import address table.)

### The FFI is in the process, but not in your state

`noita.exe` imports the **entire LuaJIT FFI** from `lua51.dll` - `luaopen_ffi`, every
`lj_cf_ffi_*` and every `ffi_*` entry point, 48 symbols in all. The code is loaded and
relocated before your mod runs.

Whether you can *reach* it depends on `require`, and `require` comes from `package`, which is
**not** among the six libraries. The state builder branches on a flag at `+0x4E` of its context
object. For a mod state that flag is the byte at `+0x40` of the mod record, which is set from the
`request_no_api_restrictions` attribute of the mod's `mod.xml`. For an ordinary mod it is 0, so
the six-library setup runs, `package` is never opened, and `require` is `nil` - there is no route
to the FFI, and no workaround through `loadstring`, which is nil'd too.

A mod that sets `request_no_api_restrictions="1"` gets the other branch: `luaL_openlibs`, which
opens the full standard set (`package`, `io`, `os`, `debug` and the FFI included) and keeps
`load`, `loadstring`, `gcinfo` and `collectgarbage`. `loadfile` is still replaced by the game's
VFS loader afterwards. The mod loader only accepts such a mod when a game-settings flag (at
`+0xE80` of the settings object) is clear - the options screen has a mod sandbox toggle for this -
so it is opt-in for the player as well as the modder. Check what your state got:

```lua
print("require:", require, "package:", package, "jit:", jit and jit.version)
```

If an ordinary mod prints `nil nil`, [screenshake.md](screenshake.md#the-two-byte-patch-that-enables-it)
has a two-byte patch to the state builder and what it gives up: `loadfile` stays pointed at the
game's VFS loader, but `load` and `loadstring` come back, and `loadstring` does not go through
the VFS.

## One state per mod

There is exactly **one call to `luaL_newstate`** in the whole executable, at `0x007ef148`,
inside the state builder `0x007ef130`. Its result is stored at offset `+0x38` of a
0x68-byte context object. The game creates one such object per mod, so **each mod gets its
own `lua_State`**. The game's own scripts get one too, from a separate accessor.

The builder does the same thing for every state, in this order:

1. `luaL_newstate()`, store at `this+0x38`.
2. Unless a flag at `this+0x4e` is set: open the six libraries above, then **nil five
   globals** - `load`, `loadfile`, `loadstring`, `gcinfo`, `collectgarbage` (list at
   `0x011a2aa0`). If the flag *is* set, call `luaL_openlibs` instead and keep everything.
   The flag arrives as the state builder's third constructor argument. Every site that
   constructs a state passes a literal `0` except the mod-initialisation loop, which passes the
   mod record's `request_no_api_restrictions` byte (see the mod table below). So the
   five-global removal is what every mod gets unless its `mod.xml` asks for unrestricted API.
3. Register four functions:
   | name | C function |
   |---|---|
   | `print` | `0x007eea10` |
   | `print_error` | `0x007eec70` |
   | `loadfile` | `0x007ee3c0` |
   | `do_mod_appends` | `0x007ee6c0` |

   Note that `loadfile` is *replaced* after being nil'd in step 2 - the game wants you to go
   through its own file system, so you cannot reach `loadstring`/`load` to bypass it. There
   is exactly one `loadfile` implementation, and it is not a stub: it counts its arguments
   (`lua_gettop`) and takes the normal path when given one.
4. Compile two chunks of **pure Lua** and run them. These are the definitions of `dofile` and
   `dofile_once`, and they are the most useful thing in this page, because they are the
   game's actual script-loading semantics, written as the game itself wrote them:

```lua
__loadonce = {}
dofile_once = function(filename)
    local result = nil
    local cached = __loadonce[filename]
    if cached ~= nil then
        result = cached[1]
    else
        local f, err = loadfile(filename)
        if f == nil then return f, err end
        result = f()
        __loadonce[filename] = {result}
        do_mod_appends(filename)
    end
    return result
end
```

```lua
__loaded = {}
dofile = function(filename)
    local f = __loaded[filename]
    if f == nil then
        f, err = loadfile(filename)
        if f == nil then return f, err end
        __loaded[filename] = f
    end
    local result = f()
    do_mod_appends(filename)
    return result
end
```

Read those carefully, because the difference is the whole design:

- `dofile_once` caches the **first return value** and returns it forever after.
- `dofile` caches the **chunk** and re-runs it every time. Caching the chunk is what lets a
  file be re-loaded after you have appended to it.
- **Both call `do_mod_appends(filename)` after every actual execution.** That is how a mod's
  `ModLuaFileAppend` content gets injected into a vanilla file: the appends are applied
  around execution, not baked into the file on disk.
- A failed load returns `nil, err` rather than raising.
- Both `__loaded` and `__loadonce` are **globals in the mod's own state**, so they are
  per-mod and invisible to other mods.

5. Register the 375-function modding API (`0x007ea410`).
6. Register nine extra functions that exist **only during mod initialisation**. They are not
   part of the 375: they live in their own nine-entry table at `0x01222340`, filled by a
   global initialiser (`0x00406dc0`), and are walked and pushed by the state builder itself -
   see [After magic numbers, nine globals disappear](#after-magic-numbers-nine-globals-disappear).

## The mod table

Loaded mods live in one flat array, a `std::vector` of 0x60-byte records:

| | |
|---|---|
| begin | `0x01207e9c` |
| end | `0x01207ea0` |
| capacity | `0x01207ea4` |
| stride | **0x60** |

The stride is worth stating carefully because the decompilation of the mod-initialisation
function disagrees with itself about it - some loops there render as `+0x18` and one as
`/ 0x60`. The instruction stream is unambiguous: `add esi, 0x60` followed by
`mov [0x1207ea0], esi`, a `push_back`. **Trust 0x60.** The `0x18` renderings are an
artefact of the decompiler losing track of a badly-typed pointer in a very large function.

Record fields, from their use sites:

| offset | contents |
|---|---|
| `+0x00` | the mod id string |
| `+0x18` | an 8-byte value, printed during init |
| `+0x20` | `"/mods/"` + the mod id - the directory the mod's files live under (`"/init.lua"` is appended to this) |
| `+0x38` | a **pointer** to the mod's 0x68-byte Lua manager (named `Mod init.lua`), whose own `+0x38` is the `lua_State` |
| `+0x3c` | a pointer to the mod's 0x54-byte file device - the one `init.lua` existence is probed through |
| `+0x40` | the mod's `request_no_api_restrictions` flag, passed as the manager's third constructor argument, where it becomes `+0x4e` and selects `luaL_openlibs` (set) vs. the six-library setup (clear) |
| `+0x44` | the `directory` attribute from the mod's `mod.xml` |

Note that `+0x20` and `+0x44` are two different paths and are easy to confuse: `+0x20` is
derived from the id, `+0x44` from the manifest.

## Load order

Mod initialisation is one function, and it logs its progress. In order:

1. **Once per process**, install the `mods/` directory change watcher.
2. Read the **enabled** mod list. Mod discovery is not a scan of `mods/`: it starts from the
   enabled set and probes each candidate's `mod.xml` through the device list.
3. For each mod, build its 0x60-byte record and push it.
4. Sweep every mod to dispatch `ModSettingsUpdate`.
5. Read `??S00/mod_config.xml`, rebuild the device list, install it, and **write** the mod
   list back to the same path. (It is loaded earlier, during step 2.)
6. **For each mod in order: run its `init.lua`, immediately, one at a time.** The path is
   `record+0x20` + `"/init.lua"`; existence is checked through the mod device. A mod without
   an `init.lua` is skipped silently - and because this is also where `record+0x38` is filled
   in, **a mod with no `init.lua` never gets a Lua state and therefore never
   receives `OnModPreInit` / `OnModInit` / `OnModPostInit` at all.**
7. **Then three separate passes over all mods**, in this order:
   `OnModPreInit` -> `OnModInit` -> `OnModPostInit`.
8. Log `Mods init done`.
9. Apply the magic-number appends, then the material appends. These run *after* the log line
   and *after* the init-only guard has already been lowered (see below), so "before the magic
   numbers are applied" is not the deadline for calling the append APIs - "before the end of
   step 7" is.

Two things follow, and both matter:

- **`init.lua` for every mod runs before any `OnModInit`.** So a mod's `init.lua` cannot
  rely on another mod having finished initialising. Anything shared has to be published as a
  global and picked up in `OnModInit` - and remember globals are per-state here, so a global
  you set is invisible to other mods. Cross-mod communication is files, or
  `ModSetting*`, or the appends mechanism.
- **Step 6 and step 7 are separate passes.** `OnModPreInit` does not run immediately after
  your `init.lua`; every mod's `init.lua` completes first.

The vanilla game's own entry point is `data/scripts/init.lua` in the data tree, which opens
with `dofile_once("data/scripts/lib/utilities.lua")`. **The ordered list of the game's own
boot scripts is not in the executable** - no string in the binary enumerates them. What *is*
in the binary is a single bare `"data/scripts"` literal (one referrer) plus eleven functions
that name individual `data/scripts/...` files; the ordering lives in the data tree, not the
binary.

## Callbacks

The game calls your code by name. The lookup is:

```c
lua_checkstack(L, 2);
lua_getfield(L, LUA_GLOBALSINDEX, "OnPlayerDied");   // 0xFFFFD8EE == -10002
if (lua_type(L, -1) == LUA_TFUNCTION) {             // 6
    lua_pushinteger(L, player_entity);
    int h = lua_gettop(L) - 1;
    lua_pushcclosure(L, error_handler, 0);           // the traceback handler
    lua_insert(L, h);                                // ...placed *under* the function
    lua_pcall(L, 1, 0, h);                           // 1 arg, 0 results
    lua_remove(L, h);
} else {
    lua_settop(L, -2);                               // not a function: pop it, carry on
}
```

`0xFFFFD8EE` is `LUA_GLOBALSINDEX` from Lua 5.1 - the pseudo-index of the globals table. So
**callbacks are plain globals.** Defining `function OnPlayerDied() end` anywhere at the top
level of your mod installs the callback. There is no registration call, no table, and no
per-mod namespace.

That has two consequences worth stating plainly. Because the callback name is resolved in the
globals table and the state is per mod, a mod can only ever be called through **its own**
state - and its globals are its own. But because the lookup is a fresh `lua_getfield` on
**every** call, a mod *can* swap implementations at runtime: keep two functions in locals and
reassign the global between frames. What it cannot do is register several implementations under
distinct keys - there is no registry, so the mod has to do the reassignment itself. And
because the slot is an ordinary global, any library your mod loads can accidentally overwrite
a callback.

### The callback set

| callback | arguments |
|---|---|
| `OnModPreInit`, `OnModInit`, `OnModPostInit` | none |
| `OnMagicNumbersAndWorldSeedInitialized` | none |
| `OnBiomeConfigLoaded` | none |
| `biome_modifiers_inject_spawns` | biome modifiers table |
| `OnWorldInitialized` | none |
| `OnPlayerSpawned` | player entity |
| `OnPlayerDied` | player entity |
| `OnCountSecrets` | none, **returns 2 integers, both summed across mods** |
| `OnPausedChanged` | is_paused, is_inventory_pause |
| `OnModSettingsChanged` | none |
| `OnWorldPreUpdate` | none |
| `OnWorldPostUpdate` | none |
| `OnPausePreUpdate` | none |
| `ModSettingsUpdate` | mod settings table |
| `ModSettingsGuiCount` | none, **returns 1 integer, which is tested for `> 0`** |

All of them are dispatched through the same mechanism - the name is fetched from the globals
table and called under a `pcall` with the same error handler. `biome_modifiers_inject_spawns` has
no `On` prefix, which is why a name-prefix search misses it. The two `ModSettings*` entries are
the modding side of the settings system and are called in a separate Lua state, described under
`GlobalLuaManager` below.

Two of them use their return value. `OnCountSecrets` requests two results from every mod and
adds both into running totals in C++, so each mod contributes a pair. `ModSettingsGuiCount`
requests one result and tests it against zero. Every other callback in the set discards
whatever it returns.

### When each callback fires, and in what order

Every callback is dispatched by looping over the mod list **in mod order**, so for any one
callback mod A is called before mod B; the loop for one callback finishes before the next
callback starts. The relative order of *different* callbacks follows from where the game
dispatches them:

| callback | dispatched from |
|---|---|
| `OnModPreInit`, `OnModInit`, `OnModPostInit` | mod initialisation (see [Load order](#load-order)), once per run start and again whenever the mod list is rebuilt |
| `OnMagicNumbersAndWorldSeedInitialized` | right after mod initialisation, on both a new game and a continue |
| `OnBiomeConfigLoaded`, then `biome_modifiers_inject_spawns` | the biome-config load (`_biomes_all.xml`), after the magic-numbers callback; also again whenever `BiomeMapLoad` is called |
| `OnWorldInitialized` | the loading screen's per-frame update, in the tick where it sees that loading has finished (see below) |
| `OnWorldPreUpdate` | every gameplay frame, **before** the engine's own per-frame update |
| `OnWorldPostUpdate` | every gameplay frame, **after** that update |
| `OnPlayerSpawned` | inside the player-spawn routine, once the player entity exists and is registered |
| `OnPlayerDied` | the "player entity destroyed" event handler (polymorph deaths included) |
| `OnPausePreUpdate` | instead of the two above, on frames where a menu screen is up |
| `OnPausedChanged` | whenever the pause state toggles |
| `OnModSettingsChanged` | right after the pause is released, if a setting changed while paused |
| `OnCountSecrets` | when the progress / secrets screen is built |

What this means in practice:

- **A normal frame is `OnWorldPreUpdate` -> engine update -> `OnWorldPostUpdate`.** In a frame
  where the game is paused (a menu is open) none of those fire; `OnPausePreUpdate` fires instead.
  The per-frame callbacks sit in the gameplay branch of the update function, which is skipped while
  the game is still loading.
- **`OnPlayerSpawned` fires about ten gameplay frames after the world is ready, not on the first
  one.** The world-load routine (`0x006afaa0`) and a second routine (`0x006b8460`) both write **10** into a
  spawn countdown on the game object (`+0xA8`). The per-frame update decrements it once per
  gameplay frame, *after* it has dispatched `OnWorldPostUpdate`, and when it reaches 0 it calls the
  spawn routine, which creates the player (`player.xml`) and dispatches `OnPlayerSpawned` as its
  last step. So `OnWorldPreUpdate` and `OnWorldPostUpdate` run for roughly ten frames with no
  player entity, and a mod cannot assume one exists inside them. `BiomeMapLoad` is the exception:
  it spawns the player itself, straight after reloading the biome config.
- **`OnPausedChanged` runs for every mod before the settings are applied.** On unpausing, if a
  setting was changed, the game first calls `ModSettingsUpdate` for each mod and only then
  `OnModSettingsChanged`.
- The magic-numbers callback and the biome-config load both happen during run set-up, before the
  per-frame callbacks start.

`OnWorldInitialized` is dispatched by the loading screen, not by the game object. The loading
screen is the active application while the world is built on a worker thread; on each tick it
checks whether loading has finished (the game object's state word at `+0x38` is 0). The tick in
which it is, the screen registers the game's input listeners, makes the game the active
application, calls the game's own per-frame update (`0x006b26f0`) **once**, and only then
dispatches `OnWorldInitialized`. What that single hand-over update dispatches depends on the pending-start flag word
`DAT_012059e0`, which the update tests first: while any bit is set it only runs the front-end
screens (one bit each: main menu `1`, autosave prompt `2`, and so on) and none of the per-frame
callbacks; when it is 0 it takes the gameplay branch. The bits are set once at start-up, and the
update clears a bit only when that screen's handler returns something other than 1.

- **Continue / load path:** the update clears the main-menu bit *before* it starts loading (the
  loader `0x006af160` is called right after), and every lower-numbered bit has already been cleared
  by then. The flag word is therefore 0 at the hand-over, the hand-over update takes the gameplay
  branch, and `OnWorldPreUpdate` / `OnWorldPostUpdate` are dispatched **before**
  `OnWorldInitialized`.
- **New-game path:** the new-game request (`0x009a2430`, which sets `game+0x7d` and the front-end
  teardown flag) leads to `0x006af010` while the main-menu bit is still set, and the code that
  clears it afterwards was not found, so no order is claimed for a new game.

## `GlobalLuaManager`

A mod's own state lives in a 0x68-byte Lua manager named `Mod init.lua` (the object the mod
record points at). `GlobalLuaManager` is a different 0x68-byte object of the same class: a
get-or-create accessor builds it on first request, only for a descriptor whose field at `+0xd4` is
non-null, and keeps it in a table keyed by an id string at offset `+0xc4` of that descriptor, so
asking twice returns the same object. It is the state the mod-settings dispatcher uses:
`ModSettingsUpdate` and `ModSettingsGuiCount` are looked up and called in it, not in the state
that ran the mod's `init.lua`. The documentation generator (below) creates one more object with
the same name.

Two globals support dispatch. A pointer to the mod currently being initialised is set before
and cleared after each call during init; the API functions read it to know who is calling.
Only one of them complains when it is unset - `ModRegisterMusicBank`, which logs
`ModRegisterMusicBank - current_initialized_mod is NULL??!` - while `ModImageMakeEditable` and
`ModTextFileSetContent` read the same global silently, `ModTextFileSetContent` deriving a mod
*index* from it. The guard is a disjunction of two flags, not one.

The second is a byte that is set for **the whole of mod initialisation** - across
`init.lua` and all three `OnModXxx` passes - and again for the magic-numbers dispatch. It is
cleared before the appends are physically applied, which is why "the magic-numbers pass" and
"the window in which appends are applied" are not the same window.

### After magic numbers, nine globals disappear

Immediately after `OnMagicNumbersAndWorldSeedInitialized` has been dispatched to every mod,
the game walks a table and nils nine named globals in every mod's state. The table is
populated by a static initialiser (`0x00406dc0`) as nine `(name, function)` pairs:

| global | function |
|---|---|
| `ModLuaFileAppend` | `0x0082d1d0` |
| `ModLuaFileSetAppends` | `0x0082d990` |
| `ModTextFileSetContent` | `0x0082e350` |
| `ModImageMakeEditable` | `0x0082edf0` |
| `ModMagicNumbersFileAdd` | `0x008306b0` |
| `ModMaterialsFileAdd` | `0x00830a30` |
| `ModRegisterAudioEventMappings` | `0x00830db0` |
| `ModRegisterMusicBank` | `0x008313f0` |
| `ModDevGenerateSpriteUVsForDirectory` | `0x008317d0` |

These are the **init-only** modding APIs, and they are *not* in the 375-function list - they
come from the separate table at `0x01222340` that step 6 of the state builder walks. They are
installed into every state when it is built, they are the only way to declare appends and
content overrides, and once the magic numbers are resolved the game revokes them by walking
every mod's state and setting all nine to nil.

Six of the nine additionally carry a runtime guard that logs
`init.lua API called outside init - '<name>' doesn't do anything at this point` and returns
without doing anything: `ModImageMakeEditable`, `ModMagicNumbersFileAdd`, `ModMaterialsFileAdd`,
`ModRegisterAudioEventMappings`, `ModRegisterMusicBank` and `ModDevGenerateSpriteUVsForDirectory`.
The other three - `ModLuaFileAppend`, `ModLuaFileSetAppends` and `ModTextFileSetContent` - have
no such guard. Either way the global is nil by then, so a call afterwards reaches nothing.

So: **everything that patches a data file has to happen during initialisation.** That is not a
style recommendation, it is enforced by nil-ing the globals.

The precise deadline is worth spelling out, because it is earlier than "the magic numbers are
resolved". The guard is still up during `init.lua` and during all three `OnModXxx` passes, and
goes down immediately after them. So `ModMagicNumbersFileAdd` and `ModMaterialsFileAdd` must
be called from `init.lua`, `OnModPreInit`, `OnModInit` or `OnModPostInit`; the calls are queued
and physically applied after `Mods init done`. A mod that calls one from `OnWorldInitialized`
gets the "doesn't do anything at this point" line.

## Errors

### The error handler

Every protected call installs the same C closure, `0x007ede90`, as `lua_pcall`'s error
function. It does three things:

1. `lua_tolstring(L, 1, NULL)` - the error object as a string.
2. Hands it to the logging function (`0x007efbc0`), which builds the full text and logs it.
3. **Returns 1.**

The full text is the message followed by a stack traceback in the usual Lua layout:

```
<message>
  Stack traceback:
	  <source>:<line>: in function '<name>'
	  ...
```

Frames are described as `function '<name>'`, `function <source:line>` or `main chunk`; a stack
deeper than 22 levels is abbreviated, with `...` between its first and last ten or so frames.
The text is built on top of the Lua stack, so returning 1 hands it to `pcall`: **the error value
your `pcall` receives is the message plus the traceback, not the original message.** If the error
object is not a string (so `lua_tolstring` returns null), the logger substitutes the string
`Unspecified error` first.

### Error dedup

The logger (`0x007efbc0`) is where the interesting behaviour is. It keeps **one global
string** - the text (including the traceback) of the last error it logged - and one flag:

- If the new error is **byte-identical to the last one logged**, it prints
  `Lua error - (skipping logging of recurring lua errors)` **once**, and then stays silent
  until a different error arrives.
- Any different error is logged in full as `Lua error - <text>` and becomes the new
  reference text.

The effect is that a mod which throws the same error every frame - an off-by-one in a
per-frame `OnWorldPreUpdate`, a nil deref in a hot path - produces **one entry in the log,
not sixty per second**. The comparison is against the single most recent message only, so
two errors alternating will both log every time.

There is also a global flag (initially set, so dedup is on) that, when cleared, logs everything.

This is the single most useful thing to know when debugging a mod: **if your error is not
appearing in the log, it is probably because the identical error appeared just before it.**

### A mod cannot break another mod

The dispatch loop iterates the mod table and, for each mod, calls `lua_pcall` and then
unconditionally `lua_remove`s the handler. **The status is never examined** and there is no
`break`. Consequences:

- An error in one mod's callback unwinds only that mod's chunk. The C++ frame continues.
- Every later mod in the same pass still gets called.
- The dedup state is global, not per mod, so a mod spamming errors can cause *another*
  mod's first error to be reported as "recurring" if the texts happen to match.
- An error inside a mod's `init.lua` does not stop other mods' `init.lua` from running.

What an error *cannot* do is skip your own remaining work: a `pcall` at the C boundary
unwinds the whole Lua chunk, so everything after the failing line in that function is
skipped. There is no per-statement isolation.

## Coroutines

**The C++ side cannot start a Lua coroutine.** `lua_newthread`, `lua_resume` and
`lua_yield` are not in the import table at all - not called once, not present. The game
never resumes or yields a Lua thread.

The strings `enable_coroutines` and the `LuaComponent` note are a *Noita* VM concept -
entity components that run their own bytecode - not Lua 5.1 coroutines. Do not conflate
them.

If your mod uses `coroutine.create`, it is doing so entirely inside Lua: nothing on the C++
side can pick the thread back up. The practical consequence is that a coroutine is bounded by
the callback it started in - if the callback returns, a still-suspended coroutine is simply
never resumed. Whether a yield that happens *inside* such a chunk raises
`attempt to yield across C-call boundary` or unwinds silently is LuaJIT-internal and cannot be
settled from the executable; the game's own note about component coroutines says only that it
"never gets to resume" them. If you need to suspend across callbacks, you need your own
scheduler on top of plain function calls, not `coroutine`.

## Arity and type handling

The API functions do **no type checking in the usual sense** and never raise across the C
boundary. Instead:

- **The arity guard.** Each function knows its minimum arity from its own signature string.
  Too few arguments produces a constructed message
  (`requires N parameters, only M given`) that goes through the same logger, and the
  function **returns no values**. It does not raise, and it does not return a sensible
  default.

- **Defaults come from the signature strings.** Noita's functions carry their own
  documentation as a string literal, including defaults (`margin:number = 5`). When an
  argument slot is absent, the documented default is substituted: the function
  pre-initialises every slot from a compiled-in constant and only overwrites it when the slot
  is actually present.

  A slot holding the *wrong type* is a different matter, and the distinction matters when you
  are debugging. For numbers and booleans there is no type check at all - the function calls a
  bare `lua_tonumber` / `lua_toboolean` and takes the coercion silently. For strings there is
  a branch, but it does **not** substitute anything: the slot keeps the compiled-in default
  and a warning is logged (`<signature> param 7 wasn't a string, string was expected`). So the
  outcome of a mistyped string argument is the same as an absent one, plus a log line.

  The per-function facts - arity floor, which `lua_to*` accessor each argument uses, the
  resolved default values, the return arity - are tabulated in
  [api-reference.md](api-reference.md).

- **String fallbacks.** Where a non-string is passed, the game's own conversion path is
  used rather than a `lua_tostring` error.

The combined effect: a mod that passes the wrong types gets defaults and no warning, and a
mod that passes too few arguments gets nothing and a log line. Neither is a hard failure,
which is why type errors in Noita mods tend to surface as "the value is just always the
default" rather than as an exception.

## The documentation generator

The generator is the function at `0x007ec750`. It is reached from the start-up function's
exit-immediately developer branch, on the command-line flag
`-build_lua_api_documentation_n_exit`. It runs `data/scripts/debug/generate_lua_documentation.lua`.
It is *not* a method of anything: the name `___main` is a **Lua global** that `0x007ec750`
itself writes after loading the chunk, and the C++ entry point has no recoverable name.

It works like this:

1. Open `tools_modding/lua_api_documentation.txt`.
2. Write the current modding API version as the line `Current modding API version: ` followed by
   the number (the shipped `lua_api_documentation.txt` reads `12`).
3. Write `tools_modding/luacheck_config.lua` - a **luacheck global whitelist**: the literal
   `read_globals = {`, then every global name the generator collected, then `}`. This is the
   file community tooling reads; it is not a stub set of function signatures.
4. **Create a brand-new, isolated `GlobalLuaManager` with its own fresh `lua_State`.** The
   documentation run does not touch any mod's state - the object is a stack local and is never
   entered into the get-or-create map.
5. Load `data/scripts/debug/generate_lua_documentation.lua` into it and set the chunk as the
   global `___main`; likewise set a global `in_function_signatures`.
6. Call `___main` through the standard dispatch - `pcall` with the same error handler.
7. If the Lua code set a global `out_html` to a string, write
   `tools_modding/lua_api_documentation.html`. If it set `out_json`, write the `.json`. The
   shipped script sets `out_html` unconditionally (it renders the signature list to HTML) and has
   its `out_json` line commented out, so the `.json` is never written unless the script is
   replaced.
8. Close the state.

The three source filenames the generator carries internally (`lua\lua_api.cpp`,
`misc_utils\modding.cpp`, `component_updators\gun_system.cpp`) are passed as lookup keys to a
resolver, not written into the output - so they do not appear in the `.txt`.

The generator's input is the list of function signatures the game carries as strings, so its
output is a signature list and a global whitelist. It does **not** walk the live state and does
not emit the enum tables (`GUI_OPTION`, keycodes, cell materials); those are plain Lua in the data
tree (see [enums.md](enums.md)). A reimplementation or editor integration can use the `.txt` and
`luacheck_config.lua` as an authoritative list of the 375 names plus `dofile`, `dofile_once` and
a few `SetValue*`/`GetValue*` entries.

## For a reimplementation

The minimum that reproduces observed behaviour:

1. One `lua_State` per mod. Never share a state between mods - per-state globals are the
   isolation boundary, and the callback lookup depends on it.
2. Open base + `table` + `string` + `math` + `bit` + `jit`; then nil `load`, `loadfile`,
   `loadstring`, `gcinfo`, `collectgarbage`; then re-add your own `loadfile` that goes
   through the access gate. (For a mod with `request_no_api_restrictions`, open every standard
   library instead and skip the nil-ing.)
3. Inject `dofile` and `dofile_once` as the two Lua chunks above, including the
   `do_mod_appends` call after each execution. This is where mod appends get applied.
4. Register the API, then the nine init-only APIs.
5. Load every mod's `init.lua` in order, then sweep `OnModPreInit` / `OnModInit` /
   `OnModPostInit` as three separate passes.
6. Dispatch callbacks out of the **globals** table, with a per-call `pcall` and an error
   handler, and never abort the loop on error.
7. Log the first occurrence of each distinct error and suppress exact repeats, comparing
   against the single most recent message.
8. After the magic-numbers pass, nil the nine init-only globals in every state.

## Undetermined

- **The ordered list of the game's own boot scripts.** Not in the executable - nothing in it
  enumerates boot scripts, and the ordering lives in the data tree.
- The full field map of the 0x60-byte mod record. The offsets above are the ones with
  confirmed use sites; `+0x08`-`+0x17`, `+0x28`-`+0x37` and `+0x48`-`+0x5b` are unlabelled.
- Whether `coroutine` survives inside the globals table, and whether a yield inside a
  `pcall`-entered chunk raises or unwinds. Both are properties of the shipped `LUA51.DLL`, not
  of `noita.exe`; only a runtime test can settle them.
