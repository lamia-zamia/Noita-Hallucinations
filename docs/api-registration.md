# How the API is registered, and the documentation generator hidden in it

## One registrar, 375 functions

Every global Lua function is installed by a single function, which walks its table as adjacent
pairs:

```c
lua_pushcclosure(L, FUN_0078e060, 0);
lua_setfield(L, K, "EntityLoad");
```

`FUN_0078e060` is the implementation; `"EntityLoad"` is the name mods see. One registrar,
375 pairs, one name per address.

## The other thirteen registrars

The binary contains thirteen other functions that call `lua_setfield`. None of them installs
anything a mod can call, because they set the field on an engine object rather than on the
global environment.

| address | what it registers | on what |
|---------|-------------------|---------|
| `0x00433040` | `RegisterStreamingEvent` | `StreamingIntegrationLuaManager` |
| `0x0065dbe0` | `RegisterPerk` | `Perks` |
| `0x007a0390` | MetaObject / component-type reflection table | the reflection registry |
| `0x007a0770` | MetaObject reflection with validation | the reflection registry |
| `0x007ec750` | `___main`, `in_function_signatures` on `GlobalLuaManager` | see below |
| `0x007ef130` | `do_mod_appends`, `loadfile`, `print`, `print_error` | globals (the mod-append bootstrap) |
| `0x00838c70` | clears `OnMagicNumbersAndWorldSeedInitialized` to `nil` | globals |
| `0x009d65f0` | `_ConfigGunActionInfo_ReadToGame` | gun internals |
| `0x00b294a0` | gun-system callbacks: `RegisterGunAction`, `OnActionPlayed`, `BeginProjectile`, `StartReload`, … | `GunSystem` |
| `0x00b2bcc0` | `Reflection_RegisterProjectile`, `GunSystem::GetGunActionInfos` | `GunSystem` |
| `0x00ba6b50` | `____cached_func` cache | Lua component internals |
| `0x00ba6e70` | `____cached_func` cache (`LuaSystem::Execute - reuse per component`) | Lua component internals |
| `0x00c23e00` | `GameRegisterStatusEffect` and the status-effect names (`ALCOHOLIC`, `FOOD_POISONING`, `INVISIBILITY`, `OILED`, `RADIOACTIVE`, `SLIMY`) | `StatusEffectSystem` |

Two of these are worth knowing about as a mod author, because they are how you register the
things that cannot be expressed in an entity XML file:

- `GameRegisterStatusEffect` — the only way to add a status effect. It is reached through the
  `StatusEffectSystem` reflection object rather than as a plain global, which is why it is
  absent from the list of 375.
- The MetaObject tables behind `0x007a0390` / `0x007a0770` — this is the machinery that makes
  `ComponentObjectGetValue2`, `EntityAddComponent2` and every typed `Component*Value*` call
  work. When one of those reports `isn't a MetaObject or doesn't exist`, this is what it means.

These thirteen do not use the `pushcclosure` → `setfield` adjacency because they either
register a value that is already on the stack, build a table, or compute the field name at run
time. That is why they are easy to miss when auditing the API surface.

## The game ships an API documentation generator

`0x007ec750` is `GlobalLuaManager`'s `___main`, and it is not an API registrar. It is a
documentation generator, and it is the most useful thing in this build.

It runs `data/scripts/debug/generate_lua_documentation.lua` and writes:

```
tools_modding/lua_api_documentation.txt          always
tools_modding/luacheck_config.lua                always
tools_modding/lua_api_documentation.html         when out_html is set
tools_modding/lua_api_documentation.json         when out_json is set
```

It also emits a `Current modding API version: ` line, a `read_globals = { … }` block, and
annotations naming the engine source files the API is defined in (`lua\lua_api.cpp`,
`misc_utils\modding.cpp`, `component_updators\gun_system.cpp`).

### Why this matters

The executable can tell you how the API behaves — the enforced arity, the argument coercion,
the substituted defaults, the shared global state. All of that is in
[how-the-api-works.md](how-the-api-works.md) and [api-reference.md](api-reference.md).

It cannot tell you the *values* of the enum tables, because those live in game data rather
than in code. The generator walks the live Lua state, so it emits them. See
[enums.md](enums.md).
