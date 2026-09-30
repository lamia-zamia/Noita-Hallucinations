# Noita Lua API reference

The 375 functions that make up Noita's modding API, with what the game binary actually does
with each argument — as opposed to what its signature claims.

Most Lua API documentation for Noita is the game's own usage strings, copied verbatim. Those
are accurate about *shape* and silent about *behaviour*. This reference adds the behaviour:
the argument count each function really enforces, how it coerces each argument, what it
substitutes when one is missing, and what it does when you get it wrong.

For the reasoning behind all of this, and for the global rules that apply to the whole API,
read [how-the-api-works.md](how-the-api-works.md) first. It is short and it changes how you
write mod code.

## How to read an entry

```
#### `Name`  · address · instruction count · enforced arity · return count
```

| column | meaning |
|--------|---------|
| `enforces >= N args` | The real minimum. If you pass fewer, the call **does nothing** and returns no values. |
| `read as` | How each argument is read. `lua_tonumber` means a silently-coerced number; `lua_isstring` + `lua_tolstring` means a string with a hardcoded fallback; `lua_type` + `lua_topointer` means a lightuserdata handle. |
| `if absent` | The documented default, and underneath it the default the game actually substitutes — usually only visible when the argument is missing *or* the wrong type. |
| `Runtime messages` | The validation failures the function can report to the log. |
| `writes` / `reads` | Process-global state the function touches. This is where cross-mod interference comes from. |

Entries are grouped by family. Blockquoted lines under an entry are warnings derived
automatically from the disassembly.

**Addresses are specific to the build analysed and will move on update.** Treat the names as
the stable part.

## Coverage

<!-- BEGIN GENERATED overview -->
These numbers come from disassembling all **375** Lua-callable functions in one Steam build of `noita.exe`. Addresses are specific to that build and will move on update; the function names will not.

| Property | Count |
|----------|------:|
| Functions decompiled | 375 |
| Have a runtime arity check (`lua_gettop` guard) | 366 |
| Have **no** arity check | 9 |
| Signature string recovered from the binary | 367 |
| Single-jump aliases (thunks) | 0 |
| Return nothing | 178 |

**No aliases.** Not one of the 375 names is a jump to another implementation, so every name is a real function body. Where two names look interchangeable (`EntityGetHerdRelation` / `GetHerdRelation`, `EntityGetComponent` / `EntityGetFirstComponent`) they are genuinely separate code with separate validation, and can diverge between builds.

### The whole API touches only these C functions

Note what is *absent*: there is no `luaL_check*`, no `luaL_argerror`, no `luaL_error` and no `lua_error` anywhere. **Noita never type-checks a Lua argument** — see [how-the-api-works.md](how-the-api-works.md).

| import | functions using it |
|--------|--------------------:|
| `lua_gettop` | 366 |
| `lua_checkstack` | 175 |
| `lua_tointeger` | 163 |
| `lua_tolstring` | 147 |
| `lua_isstring` | 142 |
| `lua_tonumber` | 99 |
| `lua_pushinteger` | 62 |
| `lua_pushnumber` | 54 |
| `lua_pushboolean` | 47 |
| `lua_type` | 44 |
| `lua_toboolean` | 42 |
| `lua_topointer` | 36 |
| `lua_pushstring` | 34 |
| `lua_createtable` | 4 |
| `lua_pushlightuserdata` | 1 |
| `lua_pushvalue` | 1 |

### Shared internals worth knowing

The engine functions that a large share of the API routes through. Everything else it calls is C++ plumbing (`std::string`, the stack cookie) and is omitted.

| address | functions | role |
|---------|----------:|------|
| `0x007efbc0` | 367 | Error sink: formats the message, logs `"Lua error - <msg>"`, and **suppresses an immediate repeat of an identical message** (globals `DAT_01225c10` / `DAT_01223e88`). Does not raise a Lua error. |
| `0x007efe10` | 152 | String-arg fallback: logs `"<n> param wasn't a string, string was expected"` and returns the hardcoded default string supplied by the caller. |
| `0x00439bb0` | 105 | Lazy constructor for the **entity-manager singleton** stored at `DAT_01204b98`. |
| `0x0056eba0` | 59 | Entity lookup: linear scan of the manager's pointer vector comparing ids. |
| `0x0078c030` | 40 | Gui handle validator: returns 0 for a stale `gui` handle (call then no-ops). Caches the last handle in the global `DAT_01223fe0` and checks it against a registry sentinel `DAT_01224ac8`. |
| `0x008a8a00` | 33 | Component lookup: 29 of the 30 `Component*` functions resolve their `component_id` through this. |
| `0x0078da80` | 20 | reads an entity argument: `lua_tointeger` then an entity-manager lookup. |
| `0x0078dd30` | 11 | reads argument 1 as a number, with a hardcoded fallback. |
| `0x0078d9f0` | 10 | pushes a container to Lua as a table (`lua_createtable` + `lua_rawseti`). |

### Functions with no arity check

Nothing in these rejects a short argument list; a missing argument reads as zero / nil and the call proceeds.

- `Debug_SaveTestPlayer` (`0x007e4a60`) &mdash; -
- `EntitySave` (`0x0078f270`) &mdash; EntitySave( entity_id:int, filename:string ) [Note: works only in dev builds.]
- `GameGetDateAndTimeLocal` (`0x007c6a80`) &mdash; -
- `GameGetDateAndTimeUTC` (`0x007c6a00`) &mdash; -
- `InputGetMousePosOnScreen` (`0x007c0230`) &mdash; -
- `StreamingForceNewVoting` (`0x007ea0b0`) &mdash; -
- `StreamingGetConnectedChannelName` (`0x007e9ba0`) &mdash; -
- `StreamingGetRandomViewerName` (`0x007e9cd0`) &mdash; -
- `StreamingGetVotingCycleDurationFrames` (`0x007e9c80`) &mdash; -

### Functions with no usage string

The binary documents these not at all: no signature string is embedded, so there is nothing to cross-check the arity against. What they actually do is in [undocumented-functions.md](undocumented-functions.md) — three of the eight are dead or misleading in a release build.

| function | arity check | returns |
|----------|-------------|---------|
| `Debug_SaveTestPlayer` (`0x007e4a60`) | **no** | 0 |
| `GameGetDateAndTimeLocal` (`0x007c6a80`) | **no** | 8 |
| `GameGetDateAndTimeUTC` (`0x007c6a00`) | **no** | 6 |
| `InputGetMousePosOnScreen` (`0x007c0230`) | **no** | 2 |
| `StreamingForceNewVoting` (`0x007ea0b0`) | **no** | 0 |
| `StreamingGetConnectedChannelName` (`0x007e9ba0`) | **no** | 1 |
| `StreamingGetRandomViewerName` (`0x007e9cd0`) | **no** | 1 |
| `StreamingGetVotingCycleDurationFrames` (`0x007e9c80`) | **no** | 1 |

### Return arity distribution

| values returned | functions |
|----------------:|----------:|
| 0 | 189 |
| 1 | 140 |
| 2 | 29 |
| 3 | 4 |
| 4 | 5 |
| 5 | 2 |
| 6 | 3 |
| 7 | 1 |
| 8 | 1 |
| 11 | 1 |

### Most-written process globals

| global | functions writing it |
|--------|--------------------:|
| `0xDAT_01154b98` | 10 |
| `0xDAT_01154b9c` | 10 |
| `0xDAT_01154ba0` | 10 |
| `0xDAT_01154ba4` | 10 |
| `0xDAT_01154ba8` | 10 |
| `0xDAT_01152ff0` | 7 |
| `0xDAT_0122374c` | 6 |
| `0xDAT_01204704` | 3 |
| `0xDAT_01204b98` | 3 |
| `0xDAT_01207efc` | 3 |
| `0xDAT_0120741c` | 2 |
| `0xDAT_01221d18` | 2 |
| `0xDAT_012236e8` | 2 |
| `0xDAT_012236f4` | 2 |
| `0xDAT_012236f8` | 2 |

<!-- END GENERATED overview -->

## Signature versus enforcement

Every function that documents its own signature can be checked against the arity floor the
binary actually enforces. Doing that for all of them:

<!-- BEGIN GENERATED crosscheck -->
Checked 317 functions where both a signature and an arity floor exist. In **310** of them the enforced minimum equals the count of non-optional parameters, i.e. the embedded signature is accurate.

The rest disagree. **These are the ones to be careful with** — the signature is not the whole truth:

| function | enforced minimum | required per signature |
|----------|-----------------:|----------------------:|
| `GuiSlider` | 11 | 12 of 12 |
| `LoadPixelScene` | 4 | 5 of 10 |
| `ModSettingGetAtIndex` | 0 | 1 of 1 |
| `PhysicsAddJoint` | 3 | 6 of 6 |
| `PhysicsBodyIDSetTransform` | 3 | 7 of 7 |
| `PhysicsComponentSetTransform` | 3 | 7 of 7 |
| `PolymorphTableAddEntity` | 2 | 1 of 3 |

<!-- END GENERATED crosscheck -->

## The reference

<!-- BEGIN GENERATED reference -->
### Entity (54)

#### `EntityAddChild`  
`0x00793150` &middot; 1116 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `parent_id` | `int` | `lua_tointeger` | - |
| 2 | `child_id` | `int` | `lua_tointeger` | - |

Runtime messages:
- `child already has a parent`
- `child not found`
- `parent not found`

reads `0x01204b98`

#### `EntityAddComponent`  
`0x0078fc90` &middot; 2178 instructions &middot; enforces >= 2 args &middot; returns via a helper

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | - | - |
| 2 | `component_type_name` | `string` | - | -<br>hardcoded *this function's own usage string* |
| 3 | `table_of_component_values` | `{string}` | - | documented `nil` |

Returns `component_id:int`

writes `0x01152ff0` = 1 &middot; reads `0x00ff6660` = 44, `0x010027a4` = _tags, `0x01204b30`, `0x01204b98`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

> Returns through a push-helper rather than a `lua_push*` call, so the return value does not appear in the decompiled body.

#### `EntityAddComponent2`  
`0x0079e5e0` &middot; 2059 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 2 | (unnamed) | - | - | -<br>hardcoded *this function's own usage string* |

Runtime messages:
- `EntityAddComponent2`
- `_enabled`
- `could not create component of type :`

writes `0x01152ff0` = 1 &middot; reads `0x00ff6660` = 44, `0x010027a4` = _tags, `0x01204b30`, `0x01204b98`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityAddRandomStains`  
`0x007b52d0` &middot; 777 instructions &middot; enforces >= 3 args &middot; returns nothing

> Adds random visible stains of 'material_type' to entity. 'amount' controls the number of stain cells added. Does nothing if 'entity' doesn't have a SpriteStainsComponent. Use CellFactory_GetType() to convert a material name to material type.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity` | `int` | `helper:entity` | - |
| 2 | `material_type` | `number` | `lua_tointeger` | - |
| 3 | `amount` | `number` | `lua_tonumber` | - |



#### `EntityAddTag`  
`0x00796d50` &middot; 846 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `tag` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

reads `0x01204b98`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityApplyTransform`  
`0x00792a50` &middot; 886 instructions &middot; enforces >= 2 args &middot; returns nothing

> Sets the transform and tries to immediately refresh components that calculate values based on an entity's transform.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `x` | `number` | `lua_tonumber` | - |
| 3 | `y` | `number` | `lua_tonumber` | documented `0` |
| 4 | `rotation` | `number` | - | documented `0` |
| 5 | `scale_x` | `number` | - | documented `1` |
| 6 | `scale_y` | `number` | - | documented `1` |

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `EntityConvertToMaterial`  
`0x007e4e70` &middot; 1403 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `material` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `use_material_colors` | `bool` | `lua_toboolean` | documented `true` |
| 4 | (unnamed) | `replace_existing_cells` | `lua_toboolean` | documented `false` |

Runtime messages:
- `couldn't find entity with id:`
- `couldn't find material:`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityCreateNew`  
`0x0078f360` &middot; 856 instructions &middot; enforces >= 0 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `name` | `string` | `lua_isstring`, `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |

Returns `entity_id:int`

reads `0x01204b98`

> The arity guard is **vacuous** (`lua_gettop` is never negative), so even the 1 documented parameter(s) are not actually enforced.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityGetAllChildren`  
`0x007935b0` &middot; 1284 instructions &middot; enforces >= 1 arg &middot; returns 1

> If passed the optional 'tag' parameter, will return only child entities that have that tag (If 'tag' isn't a valid tag name, will return no entities). If no entities are returned, might return either an empty table or nil.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `tag` | `string` | `lua_isstring`, `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |

Returns `{entity_id:int}|nil`

reads `0x01204b98`, `0x01206fac`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityGetAllComponents`  
`0x00790840` &middot; 976 instructions &middot; enforces >= 1 arg &middot; returns 1

> Returns a table of component ids.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |

Returns `{int}`

writes `0x01152ff0` = 1 &middot; reads `0x01204b98`

#### `EntityGetClosest`  
`0x00796330` &middot; 773 instructions &middot; enforces >= 2 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `pos_x` | `number` | `lua_tonumber` | - |
| 2 | `pos_y` | `number` | `lua_tonumber` | - |

Returns `entity_id:int`

reads `0x01204b98`

#### `EntityGetClosestWithTag`  
`0x00796640` &middot; 911 instructions &middot; enforces >= 3 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `pos_x` | `number` | `lua_tonumber` | - |
| 2 | `pos_y` | `number` | `lua_tonumber` | - |
| 3 | `tag` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `entity_id:int`

reads `0x01204b98`

> A missing/invalid string argument (argument 3) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityGetClosestWormAttractor`  
`0x007bdb70` &middot; 993 instructions &middot; enforces >= 2 args &middot; returns 3

> NOTE: entity_id might be NULL, but pos_x and pos_y could still be valid.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `pos_x` | `number` | `lua_tonumber` | - |
| 2 | `pos_y` | `number` | `lua_tonumber` | - |

Returns `entity_id:int, pos_x:number, pos_y:number`

reads `0x01054028` = 2139095039, `0x0122197c`, `0x01221a0c`

#### `EntityGetClosestWormDetractor`  
`0x007bdf60` &middot; 1053 instructions &middot; enforces >= 2 args &middot; returns 4

> NOTE: entity_id might be NULL, but pos_x and pos_y could still be valid

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `pos_x` | `number` | `lua_tonumber` | - |
| 2 | `pos_y` | `number` | `lua_tonumber` | - |

Returns `entity_id:int, pos_x:number, pos_y:number, radius:number`

reads `0x01054028` = 2139095039, `0x0122170c`, `0x012219c0`

#### `EntityGetComponent`  
`0x00790c10` &middot; 1721 instructions &middot; enforces >= 2 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `component_type_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `tag` | `string` | `lua_isstring`, `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |

Returns `{component_id}|nil`

Runtime messages:
- `component type`
- `doesn't exist`

writes `0x01152ff0` = 1, `0x01225ae4`, `0x01225b2c`, `0x01225b30` &middot; reads `0x00000001`, `0x01204b98`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityGetComponentIncludingDisabled`  
`0x00791a10` &middot; 1746 instructions &middot; enforces >= 2 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `component_type_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `tag` | `string` | `lua_isstring`, `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |

Returns `{component_id}|nil`

Runtime messages:
- `component type`
- `doesn't exist`

writes `0x01152ff0` = 1, `0x01225b4c`, `0x01225ebc`, `0x01225b50` &middot; reads `0x00000001`, `0x01204b98`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityGetFilename`  
`0x00797760` &middot; 940 instructions &middot; enforces >= 1 arg &middot; returns 1

> Return value example: 'data/entities/items/flute.xml'. Incorrect value is returned if the entity has passed through the world streaming system.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |

Returns `full_path:string`

reads `0x00fe3c84`, `0x01204b98`

#### `EntityGetFirstComponent`  
`0x007912d0` &middot; 1847 instructions &middot; enforces >= 2 args &middot; returns 1 (of 2 push sites)

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `component_type_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `tag` | `string` | `lua_isstring`, `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |

Returns `component_id|nil`

Runtime messages:
- `component type`
- `doesn't exist`
- `invalid vector<T> subscript`

writes `0x01225eb4`, `0x01225ef0`, `0x01225eb8` &middot; reads `0x01204b98`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityGetFirstComponentIncludingDisabled`  
`0x007920f0` &middot; 1464 instructions &middot; enforces >= 2 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `component_type_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `tag` | `string` | `lua_isstring`, `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |

Returns `component_id|nil`

Runtime messages:
- `component type`
- `doesn't exist`

writes `0x01225ac4`, `0x01226fc0`, `0x01225ac8` &middot; reads `0x01204b98`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityGetFirstHitboxCenter`  
`0x007b9d70` &middot; 959 instructions &middot; enforces >= 1 arg &middot; returns 2

> Returns the centroid of first enabled HitboxComponent found in entity, the position of the entity if no hitbox is found, or nil if the entity does not exist. All returned positions are in world coordinates.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `helper:entity` | - |

Returns `(x:number,y:number)|nil`

reads `0x0105361c` = 1056964608 / 0.5, `0x01204b98`

#### `EntityGetHerdRelation`  
`0x007bcd40` &middot; 1745 instructions &middot; enforces >= 2 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_a` | `int` | `lua_tointeger` | - |
| 2 | `entity_b` | `int` | `lua_tointeger` | - |

Returns `number`

Runtime messages:
- `doesn't exist`
- `entity`
- `entity_a doesn't have an enabled GenomeComponent`
- `entity_b doesn't have an enabled GenomeComponent`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `EntityGetHerdRelationSafe`  
`0x007bd420` &middot; 993 instructions &middot; enforces >= 2 args &middot; returns 1

> does not spam errors, but returns 0 if anything fails

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_a` | `int` | `lua_tointeger` | - |
| 2 | `entity_b` | `int` | `lua_tointeger` | - |

Returns `number`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `EntityGetHotspot`  
`0x007b60c0` &middot; 1256 instructions &middot; enforces >= 3 args &middot; returns 2

> Returns the position of a hot spot defined by a HotspotComponent. If 'transformed' is true, will return the position in world coordinates, transformed using the entity's transform.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity` | `int` | `helper:entity` | - |
| 2 | `hotspot_tag` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `transformed` | `bool` | - | - |
| 4 | `include_disabled_components` | `bool` | - | documented `false` |

Returns `x:number,y:number`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityGetInRadius`  
`0x00795ab0` &middot; 960 instructions &middot; enforces >= 3 args &middot; returns 1

> Returns all entities in 'radius' distance from 'x','y'.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `pos_x` | `number` | `lua_tonumber` | - |
| 2 | `pos_y` | `number` | `lua_tonumber` | - |
| 3 | `radius` | `number` | `lua_tonumber` | - |

Returns `{entity_id:int}`

reads `0x01204b98`

#### `EntityGetInRadiusWithTag`  
`0x00795e70` &middot; 1203 instructions &middot; enforces >= 4 args &middot; returns 1

> Returns all entities in 'radius' distance from 'x','y'.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `pos_x` | `number` | `lua_tonumber` | - |
| 2 | `pos_y` | `number` | `lua_tonumber` | - |
| 3 | `radius` | `number` | `lua_tonumber` | - |
| 4 | `entity_tag` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `{entity_id:int}`

reads `0x01204b98`, `0x01206fac`

> A missing/invalid string argument (argument 4) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityGetIsAlive`  
`0x0078f9a0` &middot; 747 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |

Returns `bool`

reads `0x01204b98`

#### `EntityGetName`  
`0x00794c90` &middot; 832 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |

Returns `name:string`

reads `0x01204b98`

#### `EntityGetParent`  
`0x00793ac0` &middot; 756 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |

Returns `entity_id:int`

reads `0x01204b98`

#### `EntityGetRootEntity`  
`0x00793dc0` &middot; 768 instructions &middot; enforces >= 1 arg &middot; returns 1

> Returns the given entity if it has no parent, otherwise walks up the parent hierarchy to the topmost parent and returns it.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |

Returns `entity_id:int`

reads `0x01204b98`

#### `EntityGetTags`  
`0x00795320` &middot; 806 instructions &middot; enforces >= 1 arg &middot; returns 1

> Returns a string where the tags are comma-separated, or nil if 'entity_id' doesn't point to a valid entity.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |

Returns `string|nil`

reads `0x01204b98`

#### `EntityGetTransform`  
`0x00792dd0` &middot; 893 instructions &middot; enforces >= 1 arg &middot; returns 5

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |

Returns `x:number,y:number,rotation:number,scale_x:number,scale_y:number`

reads `0x01204b98`

#### `EntityGetWandCapacity`  
`0x007b5dd0` &middot; 742 instructions &middot; enforces >= 1 arg &middot; returns 1

> Returns the capacity of a wand entity, or 0 if 'entity' doesnt exist.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity` | `int` | `helper:entity` | - |

Returns `int`



#### `EntityGetWithName`  
`0x007969d0` &middot; 892 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `entity_id:int`

reads `0x01204b98`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityGetWithTag`  
`0x00795650` &middot; 1113 instructions &middot; enforces >= 1 arg &middot; returns 1

> Returns all entities with 'tag'.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `tag` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `{entity_id:int}`

Runtime messages:
- `invalid vector<T> subscript`

reads `0x01204b98`, `0x01206fac`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityHasTag`  
`0x007973f0` &middot; 866 instructions &middot; enforces >= 2 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `tag` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `bool`

reads `0x01204b98`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityInflictDamage`  
`0x007b4030` &middot; 1860 instructions &middot; enforces >= 7 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity` | `int` | `lua_tointeger` | - |
| 2 | `amount` | `number` | `lua_tonumber` | - |
| 3 | `damage_type` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 4 | `description` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 5 | `ragdoll_fx` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 6 | `impulse_x` | `number` | `lua_tonumber` | - |
| 7 | `impulse_y` | `number` | `lua_tonumber` | - |
| 8 | `entity_who_is_responsible` | `int` | `lua_tointeger` | documented `0` |
| 9 | `world_pos_x` | `number` | `lua_tonumber` | documented `entity_x` |
| 10 | `world_pos_y` | `number` | `lua_tonumber` | documented `entity_y` |
| 11 | `knockback_force` | `number` | - | documented `0` |

Runtime messages:
- `couldn't find entity with id:`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 3) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityIngestMaterial`  
`0x007b4780` &middot; 953 instructions &middot; enforces >= 3 args &middot; returns nothing

> Has the same effects that would occur if 'entity' eats 'amount' number of cells of 'material_type' from the game world. Use this instead of directly modifying IngestionComponent values, if possible. Might not work with non-player entities. Use CellFactory_GetType() to convert a material name to material type.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity` | `int` | `helper:entity` | - |
| 2 | `material_type` | `number` | `lua_tointeger` | - |
| 3 | `amount` | `number` | `lua_tonumber` | - |

Runtime messages:
- `no enabled IngestionComponent found in entity.`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `EntityKill`  
`0x0078f6c0` &middot; 729 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |

reads `0x01204b98`

#### `EntityLoad`  
`0x0078e060` &middot; 1501 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `pos_x` | `number` | `lua_tointeger` | documented `0` |
| 3 | `pos_y` | `number` | `lua_tointeger` | documented `0` |

Returns `entity_id:int`

Runtime messages:
- `- probably doesn't exist`
- `Error! Couldn't create a new entity, out of memory?`
- `couldn't load file:`

writes `0x01204b98` &middot; reads `0x0120866c`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityLoadCameraBound`  
`0x0078eba0` &middot; 895 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `pos_x` | `number` | `lua_tointeger` | documented `0` |
| 3 | `pos_y` | `number` | `lua_tointeger` | documented `0` |



> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityLoadEndGameItem`  
`0x0078e640` &middot; 1364 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `pos_x` | `number` | `lua_tointeger` | documented `0` |
| 3 | `pos_y` | `number` | `lua_tointeger` | documented `0` |

Returns `entity_id:int`

Runtime messages:
- `- probably doesn't exist`
- `Error! Couldn't create a new entity, out of memory?`
- `couldn't load file:`
- `data/entities/animals/boss_centipede/sampo.xml`
- `data/entities/items/orbs/orb_11.xml`

writes `0x01204b98` &middot; reads `0x0120866c`, `0x01221d18`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityLoadToEntity`  
`0x0078ef20` &middot; 843 instructions &middot; enforces >= 2 args &middot; returns nothing

> Loads components from 'filename' to 'entity'. Does not load tags and other stuff.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `entity` | `int` | `helper:entity` | - |

reads `0x01204b98`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityRefreshSprite`  
`0x007b59d0` &middot; 1024 instructions &middot; enforces >= 2 args &middot; returns nothing

> Immediately refreshes the given SpriteComponent. Might be useful with text sprites if you want them to update more often than once a second.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity` | `int` | `helper:entity` | - |
| 2 | `sprite_component` | `int` | `lua_tointeger` | - |

Runtime messages:
- `is not a SpriteComponent`

reads `0x01208018`

#### `EntityRemoveComponent`  
`0x00790520` &middot; 786 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `component_id` | `int` | `lua_tointeger` | - |

reads `0x01204b98`, `0x01208018`

#### `EntityRemoveFromParent`  
`0x007940c0` &middot; 762 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |

reads `0x01204b98`

#### `EntityRemoveIngestionStatusEffect`  
`0x007b4b40` &middot; 935 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity` | `int` | `helper:entity` | - |
| 2 | `status_type_id` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityRemoveStainStatusEffect`  
`0x007b4ef0` &middot; 992 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity` | `int` | `helper:entity` | - |
| 2 | `status_type_id` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `status_cooldown` | `int` | `lua_tointeger` | documented `0` |

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntityRemoveTag`  
`0x007970a0` &middot; 835 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `tag` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

reads `0x01204b98`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntitySave`  
`0x0078f270` &middot; 232 instructions &middot; **no arity check** &middot; returns nothing

> Note: works only in dev builds.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | - | - |
| 2 | `filename` | `string` | - | - |



> **No arity check.** Nothing stops you passing too few arguments; a missing one reads as zero/nil and the call proceeds.

#### `EntitySetComponentIsEnabled`  
`0x007947c0` &middot; 1230 instructions &middot; enforces >= 3 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `component_id` | `int` | `lua_tointeger` | - |
| 3 | `is_enabled` | `bool` | `lua_toboolean` | - |

Runtime messages:
- `couldn't find component with id:`
- `entity not found`

reads `0x01204b98`, `0x01208018`

#### `EntitySetComponentsWithTagEnabled`  
`0x007943c0` &middot; 1023 instructions &middot; enforces >= 3 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `tag` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `enabled` | `bool` | `lua_toboolean` | - |

Runtime messages:
- `entity not found`

reads `0x01204b98`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntitySetDamageFromMaterial`  
`0x007b55e0` &middot; 1005 instructions &middot; enforces >= 3 args &middot; returns nothing

> Modifies DamageModelComponents materials_that_damage and materials_how_much_damage variables (and their parsed out data structures)

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity` | `int` | `helper:entity` | - |
| 2 | `material_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `damage` | `number` | `lua_tonumber` | - |

Runtime messages:
- `no enabled DamageModelComponent found in entity`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntitySetName`  
`0x00794fd0` &middot; 847 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

reads `0x01204b98`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `EntitySetTransform`  
`0x007926b0` &middot; 926 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `x` | `number` | `lua_tonumber` | - |
| 3 | `y` | `number` | `lua_tonumber` | documented `0` |
| 4 | `rotation` | `number` | `lua_tonumber` | documented `0` |
| 5 | `scale_x` | `number` | `lua_tonumber` | documented `1` |
| 6 | `scale_y` | `number` | `lua_tonumber` | documented `1` |

reads `0x010546e0` = -2147483648 / -0, `0x01204b98`

### Component (30)

#### `ComponentAddTag`  
`0x0079c080` &middot; 1139 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |
| 2 | `tag` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Runtime messages:
- `couldn't find component with id:`

reads `0x01204b30`, `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentGetEntity`  
`0x007a0060` &middot; 810 instructions &middot; enforces >= 1 arg &middot; returns 1

> Returns the id of the entity that owns a component, or 0.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |

Returns `entity_id:int`

reads `0x01204b98`, `0x01208018`

#### `ComponentGetIsEnabled`  
`0x0079fc00` &middot; 1114 instructions &middot; enforces >= 1 arg &middot; returns 1

> Returns true if the given component exists and is enabled, else false.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |

Returns `bool`

Runtime messages:
- `couldn't find component with id:`

reads `0x01208018`

#### `ComponentGetMembers`  
`0x007a0390` &middot; 986 instructions &middot; enforces >= 1 arg &middot; returns 1

> Returns a string-indexed table of string.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |

Returns `{string-string}|nil`

reads `0x01208018`

#### `ComponentGetMetaCustom`  
`0x0079ac60` &middot; 2279 instructions &middot; enforces >= 2 args &middot; returns 1

> Deprecated, use ComponentGetValue2() instead.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |
| 2 | `variable_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `string|nil`

Runtime messages:
- `couldn't find component with id:`
- `doesn't have MetaCustom()`

reads `0x00fe4840` = 40, `0x00ff8670` = 8236, `0x01019b14` = ) - , `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentGetTags`  
`0x0079c970` &middot; 1103 instructions &middot; enforces >= 1 arg &middot; returns 1

> Returns a string where the tags are comma-separated, or nil if can't find 'component_id' component.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |

Returns `string|nil`

Runtime messages:
- `couldn't find component with id:`

reads `0x01208018`

#### `ComponentGetTypeName`  
`0x007a0d40` &middot; 775 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |

Returns `string`

reads `0x00fe3c84`, `0x01208018`

#### `ComponentGetValue`  
`0x00797e20` &middot; 974 instructions &middot; enforces >= 2 args &middot; returns 1

> Deprecated, use ComponentGetValue2() instead.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |
| 2 | `variable_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `string|nil`

reads `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentGetValue2`  
`0x0079d240` &middot; 1132 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | (unnamed) | - | `lua_tointeger` | - |
| 2 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Runtime messages:
- `ComponentGetValue2`
- `couldn't find component with id:`

reads `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentGetValueBool`  
`0x007981f0` &middot; 930 instructions &middot; enforces >= 2 args &middot; returns 1

> Deprecated, use ComponentGetValue2() instead.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |
| 2 | `variable_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `bool|nil`

reads `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentGetValueFloat`  
`0x00798940` &middot; 962 instructions &middot; enforces >= 2 args &middot; returns 1

> Deprecated, use ComponentGetValue2() instead.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |
| 2 | `variable_name` | `string` | `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `number|nil`

reads `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentGetValueInt`  
`0x007985a0` &middot; 927 instructions &middot; enforces >= 2 args &middot; returns 1

> Deprecated, use ComponentGetValue2() instead.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |
| 2 | `variable_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `int|nil`

reads `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentGetValueVector2`  
`0x00798d10` &middot; 946 instructions &middot; enforces >= 2 args &middot; returns 2

> Deprecated, use ComponentGetValue2() instead.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |
| 2 | `variable_name` | `string` | `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `x:number,y:number|nil`

reads `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentGetVector`  
`0x0079f740` &middot; 1215 instructions &middot; enforces >= 3 args &middot; returns nothing

> 'type_stored_in_vector' should be "int", "float" or "string".

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |
| 2 | `array_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `type_stored_in_vector` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `{int|number|string}|nil`

Runtime messages:
- `couldn't recognize type:`
- `float`
- `string`

reads `0x00fe4038` = int, `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentGetVectorSize`  
`0x0079edf0` &middot; 1179 instructions &middot; enforces >= 3 args &middot; returns 1

> 'type_stored_in_vector' should be "int", "float" or "string".

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |
| 2 | `array_member_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `type_stored_in_vector` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `int`

Runtime messages:
- `couldn't recognize type:`
- `float`
- `string`

reads `0x00fe4038` = int, `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentGetVectorValue`  
`0x0079f290` &middot; 1197 instructions &middot; enforces >= 4 args &middot; returns nothing

> 'type_stored_in_vector' should be "int", "float" or "string".

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |
| 2 | `array_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `type_stored_in_vector` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 4 | `index` | `int` | `lua_tointeger` | - |

Returns `int|number|string|nil`

Runtime messages:
- `couldn't recognize type:`
- `float`
- `string`

reads `0x00fe4038` = int, `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentHasTag`  
`0x0079cdc0` &middot; 1149 instructions &middot; enforces >= 2 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |
| 2 | `tag` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `bool`

Runtime messages:
- `couldn't find component with id:`

reads `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentObjectGetMembers`  
`0x007a0770` &middot; 1487 instructions &middot; enforces >= 2 args &middot; returns 1

> Returns a string-indexed table of string or nil.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |
| 2 | `object_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `{string-string}|nil`

Runtime messages:
- `isn't a MetaObject or doesn't exist`

reads `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentObjectGetValue`  
`0x0079b550` &middot; 1377 instructions &middot; enforces >= 3 args &middot; returns 1

> Deprecated, use ComponentObjectGetValue2() instead.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |
| 2 | `object_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `variable_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `string|nil`

Runtime messages:
- `isn't a MetaObject or doesn't exist`

reads `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentObjectGetValue2`  
`0x0079dbe0` &middot; 1266 instructions &middot; enforces >= 3 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | (unnamed) | - | `lua_tointeger` | - |
| 2 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Runtime messages:
- `ComponentObjectGetValue2`
- `isn't a MetaObject or doesn't exist`

reads `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentObjectSetValue`  
`0x0079bac0` &middot; 1457 instructions &middot; enforces >= 4 args &middot; returns nothing

> Deprecated, use ComponentObjectSetValue2() instead.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |
| 2 | `object_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `variable_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 4 | `value` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Runtime messages:
- `isn't a MetaObject or doesn't exist`

reads `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentObjectSetValue2`  
`0x0079e0e0` &middot; 1271 instructions &middot; enforces >= 4 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | (unnamed) | - | `lua_tointeger` | - |
| 2 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Runtime messages:
- `ComponentObjectSetValue2`
- `isn't a MetaObject or doesn't exist`

reads `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentRemoveTag`  
`0x0079c500` &middot; 1122 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |
| 2 | `tag` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Runtime messages:
- `couldn't find component with id:`

reads `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentSetMetaCustom`  
`0x0079a4b0` &middot; 1955 instructions &middot; enforces >= 3 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | (unnamed) | - | `lua_tointeger` | - |
| 2 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Runtime messages:
- `couldn't find component with id:`
- `doesn't have MetaCustom()`

reads `0x00ff8670` = 8236, `0x01019b14` = ) - , `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentSetValue`  
`0x007990d0` &middot; 1253 instructions &middot; enforces >= 3 args &middot; returns nothing

> Deprecated, use ComponentSetValue2() instead.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |
| 2 | `variable_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `value` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Runtime messages:
- `couldn't find component with id:`

reads `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentSetValue2`  
`0x0079d6b0` &middot; 1315 instructions &middot; enforces >= 3 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | (unnamed) | - | `lua_tointeger` | - |
| 2 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Runtime messages:
- `, ... ) - couldn't find component with id:`
- `ComponentSetValue2`

reads `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentSetValueValueRange`  
`0x00799ac0` &middot; 1267 instructions &middot; enforces >= 4 args &middot; returns nothing

> Deprecated, use ComponentSetValue2() instead.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |
| 2 | `variable_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `min` | `number` | `lua_tonumber` | - |
| 4 | `max` | `number` | `lua_tonumber` | - |

Runtime messages:
- `couldn't find component with id:`

reads `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentSetValueValueRangeInt`  
`0x00799fc0` &middot; 1253 instructions &middot; enforces >= 4 args &middot; returns nothing

> Deprecated, use ComponentSetValue2() instead.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |
| 2 | `variable_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `min` | `number` | `lua_tointeger` | - |
| 4 | `max` | `number` | `lua_tointeger` | - |

Runtime messages:
- `couldn't find component with id:`

reads `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ComponentSetValueVector2`  
`0x007995c0` &middot; 1267 instructions &middot; enforces >= 4 args &middot; returns nothing

> Deprecated, use ComponentSetValue2() instead.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger` | - |
| 2 | `variable_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `x` | `number` | `lua_tonumber` | - |
| 4 | `y` | `number` | `lua_tonumber` | - |

Runtime messages:
- `couldn't find component with id:`

reads `0x01208018`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GetUpdatedComponentID`  
`0x007a1320` &middot; 716 instructions &middot; enforces >= 0 args &middot; returns 1

Returns `component_id:int`

reads `0x01208020`

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

### Game (71)

#### `GameAddFlagRun`  
`0x007d5e70` &middot; 792 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `flag` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |



> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GameClearOrbsFoundThisRun`  
`0x007ab8b0` &middot; 690 instructions &middot; enforces >= 0 args &middot; returns nothing



> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GameCreateCosmeticParticle`  
`0x007b2a50` &middot; 1725 instructions &middot; enforces >= 6 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `material_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `x` | `number` | `lua_tonumber` | - |
| 3 | `y` | `number` | `lua_tonumber` | - |
| 4 | `how_many` | `int` | `lua_tointeger` | - |
| 5 | `xvel` | `number` | `lua_tonumber` | - |
| 6 | `yvel` | `number` | `lua_tonumber` | - |
| 7 | `color` | `uint32` | `lua_tointeger` | documented `0` |
| 8 | `lifetime_min` | `number` | `lua_tonumber` | documented `5.0` |
| 9 | `lifetime_max` | `number` | `lua_tonumber` | documented `10` |
| 10 | `force_create` | `bool` | `lua_toboolean` | documented `true` |
| 11 | `draw_front` | `bool` | - | documented `false` |
| 12 | `collide_with_grid` | `bool` | - | documented `true` |
| 13 | `randomize_velocity` | `bool` | - | documented `true` |
| 14 | `gravity_x` | `float` | - | documented `0` |
| 15 | `gravity_y` | `float` | - | documented `100.0` |

writes `0x0120741c`, `0x0122374c` &middot; reads `0x01053c18` = 1084227584 / 5, `0x01053ccc` = 1092616192 / 10, `0x01053e58` = 1120403456 / 100, `0x01053fa4` = 1191181824 / 32767, `0x010546e0` = -2147483648 / -0

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GameCreateParticle`  
`0x007b3120` &middot; 1436 instructions &middot; enforces >= 7 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `material_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `x` | `number` | `lua_tonumber` | - |
| 3 | `y` | `number` | `lua_tonumber` | - |
| 4 | `how_many` | `int` | `lua_tointeger` | - |
| 5 | `xvel` | `number` | `lua_tonumber` | - |
| 6 | `yvel` | `number` | `lua_tonumber` | - |
| 7 | `just_visual` | `bool` | `lua_toboolean` | - |
| 8 | `draw_as_long` | `bool` | `lua_toboolean` | documented `false` |
| 9 | `randomize_velocity` | `bool` | `lua_toboolean` | documented `true` |

writes `0x0120741c`, `0x0122374c` &middot; reads `0x01053fa4` = 1191181824 / 32767, `0x010546e0` = -2147483648 / -0

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GameCreateSpriteForXFrames`  
`0x007b36c0` &middot; 1388 instructions &middot; enforces >= 3 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `x` | `number` | `lua_tonumber` | - |
| 3 | `y` | `number` | `lua_tonumber` | - |
| 4 | `centered` | `bool` | `lua_toboolean` | documented `true` |
| 5 | `sprite_offset_x` | `number` | `lua_tonumber` | documented `0` |
| 6 | `sprite_offset_y` | `number` | `lua_tonumber` | documented `0` |
| 7 | `frames` | `int` | `lua_tointeger` | documented `1` |
| 8 | `emissive` | `bool` | `lua_toboolean` | documented `false` |

Runtime messages:
- `sprite creation failed for:`



> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GameCutThroughWorldVertical`  
`0x007c6f90` &middot; 790 instructions &middot; enforces >= 5 args &middot; returns nothing

> Each beam adds a little overhead to things like chunk creation, so please call this sparingly.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `int` | `lua_tointeger` | - |
| 2 | `y_min` | `int` | `lua_tointeger` | - |
| 3 | `y_max` | `int` | `lua_tointeger` | - |
| 4 | `radius` | `number` | `lua_tointeger` | - |
| 5 | `edge_darkening_width` | `number` | `lua_tointeger` | - |



#### `GameDestroyInventoryItems`  
`0x007b10e0` &middot; 787 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `helper:entity` | - |

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `GameDoEnding2`  
`0x007a63f0` &middot; 1027 instructions &middot; enforces >= 0 args &middot; returns nothing

Runtime messages:
- `data/entities/misc/effect_protection_all.xml`
- `data/entities/misc/effect_protection_polymorph.xml`

reads `0x01204b98`, `0x01205010`, `0x01054680`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GameDropAllItems`  
`0x007b09b0` &middot; 1065 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |

Runtime messages:
- `couldn't find entity with id:`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `GameDropPlayerInventoryItems`  
`0x007b0de0` &middot; 757 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger`, `helper:entity` | - |

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `GameEmitRainParticles`  
`0x007c6b90` &middot; 1010 instructions &middot; enforces >= 8 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `num_particles` | `int` | `lua_tointeger` | - |
| 2 | `width_outside_camera` | `number` | `lua_tonumber` | - |
| 3 | `material_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 4 | `velocity_min` | `number` | `lua_tonumber` | - |
| 5 | `velocity_max` | `number` | `lua_tonumber` | - |
| 6 | `gravity` | `number` | `lua_tonumber` | - |
| 7 | `droplets_bounce` | `bool` | `lua_toboolean` | - |
| 8 | `draw_as_long` | `bool` | `lua_toboolean` | - |

reads `0x01205010`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 3) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GameEntityPlaySound`  
`0x007d75b0` &middot; 1189 instructions &middot; enforces >= 2 args &middot; returns nothing

> Plays a sound through all AudioComponents with matching sound in 'entity_id'.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `event_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Runtime messages:
- `couldn't find entity with id:`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GameEntityPlaySoundLoop`  
`0x007d7a90` &middot; 1178 instructions &middot; enforces >= 3 args &middot; returns nothing

> Plays a sound loop through an AudioLoopComponent tagged with 'component_tag' in 'entity'. 'intensity' & 'intensity2' affect the intensity parameters passed to the audio event. Must be called every frame when the sound should play.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity` | `int` | `lua_tointeger` | - |
| 2 | `component_tag` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `intensity` | `number` | `lua_tonumber` | - |
| 4 | `intensity2` | `number` | `lua_tonumber` | documented `0` |

Runtime messages:
- `couldn't find entity with id:`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GameGetAllInventoryItems`  
`0x007b04e0` &middot; 1229 instructions &middot; enforces >= 1 arg &middot; returns nothing

> Returns all the inventory items that entity_id has.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | - | - |

Returns `{item_entity_id}|nil`

Runtime messages:
- `couldn't find entity with id:`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `GameGetCameraBounds`  
`0x007aea80` &middot; 885 instructions &middot; enforces >= 0 args &middot; returns 4

> Returns the camera rectangle. This may not be 100% pixel perfect with regards to what you see on the screen. 'x','y' = top left corner of the rectangle.

Returns `x:number,y:number,w:number,h:number`



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GameGetCameraPos`  
`0x007ae150` &middot; 836 instructions &middot; enforces >= 0 args &middot; returns 2

Returns `x:number,y:number`

reads `0x0105361c` = 1056964608 / 0.5

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GameGetDateAndTimeLocal`  
`0x007c6a80` &middot; 270 instructions &middot; **no arity check** &middot; returns 8

reads `0x00fe0c98` = 25, `0x011535c0` = 28, `0x011535c4` = 3

> **No arity check.** Nothing stops you passing too few arguments; a missing one reads as zero/nil and the call proceeds.

#### `GameGetDateAndTimeUTC`  
`0x007c6a00` &middot; 118 instructions &middot; **no arity check** &middot; returns 6

> **No arity check.** Nothing stops you passing too few arguments; a missing one reads as zero/nil and the call proceeds.

#### `GameGetFogOfWar`  
`0x007bb720` &middot; 777 instructions &middot; enforces >= 2 args &middot; returns 1

> Returns an integer between 0 and 255. Larger value means more coverage. Returns -1 if query is outside the bounds of the fog of war grid. For performance reasons consider using the components that manipulate fog of war.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `pos_x` | `number` | `lua_tonumber` | - |
| 2 | `pos_y` | `number` | `lua_tonumber` | - |

Returns `fog_of_war:int`



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `GameGetFogOfWarBilinear`  
`0x007bba30` &middot; 772 instructions &middot; enforces >= 2 args &middot; returns 1

> Returns an integer between 0 and 255. Larger value means more coverage. Returns -1 if query is outside the bounds of the fog of war grid. The value is bilinearly filtered using four samples around 'pos'. For performance reasons consider using the components that manipulate fog of war.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `pos_x` | `number` | `lua_tonumber` | - |
| 2 | `pos_y` | `number` | `lua_tonumber` | - |

Returns `fog_of_war:int`



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `GameGetFrameNum`  
`0x007bf2f0` &middot; 717 instructions &middot; enforces >= 0 args &middot; returns 1

Returns `int`



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GameGetGameEffect`  
`0x007b7060` &middot; 1279 instructions &middot; enforces >= 2 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `game_effect_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `component_id:int`

Runtime messages:
- `couldn't find entity with id:`

writes `0x01152ff0` = 1 &middot; reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GameGetGameEffectCount`  
`0x007b7560` &middot; 1220 instructions &middot; enforces >= 2 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `game_effect_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `int`

Runtime messages:
- `couldn't find entity with id:`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GameGetIsGamepadConnected`  
`0x007aa410` &middot; 728 instructions &middot; enforces >= 0 args &middot; returns 1

Returns `bool`



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GameGetIsTrailerModeEnabled`  
`0x007e4790` &middot; 719 instructions &middot; enforces >= 0 args &middot; returns 1

Returns `bool`

reads `0x01207e40`

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GameGetOrbCollectedAllTime`  
`0x007ab560` &middot; 835 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `orb_id_zero_based` | `int` | `lua_tointeger` | - |

Returns `bool`

reads `0x01207404`

#### `GameGetOrbCollectedThisRun`  
`0x007ab280` &middot; 735 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `orb_id_zero_based` | `int` | `lua_tointeger` | - |

Returns `bool`



#### `GameGetOrbCountAllTime`  
`0x007aacd0` &middot; 722 instructions &middot; enforces >= 0 args &middot; returns 1

Returns `int`

reads `0x01207404`

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GameGetOrbCountThisRun`  
`0x007aafb0` &middot; 715 instructions &middot; enforces >= 0 args &middot; returns 1

Returns `int`



> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GameGetOrbCountTotal`  
`0x007abb70` &middot; 720 instructions &middot; enforces >= 0 args &middot; returns 1

> Returns the number of orbs, picked or not.

Returns `int`

reads `0x01152544` = 10

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GameGetPlayerStatsEntity`  
`0x007aa9d0` &middot; 759 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | (unnamed) | - | `lua_tointeger` | - |

Returns `entity_id:int`



#### `GameGetPotionColorUint`  
`0x007b9800` &middot; 1377 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |

Returns `uint`

Runtime messages:
- `couldn't find entity with id:`

reads `0x01053ec4` = 1132396544 / 255, `0x01054630`, `0x01153250` = 333?<, `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `GameGetRealWorldTimeSinceStarted`  
`0x007bf5c0` &middot; 775 instructions &middot; enforces >= 0 args &middot; returns 1

Returns `number`

reads `0x01221bc0`

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GameGetSkyVisibility`  
`0x007bb3e0` &middot; 825 instructions &middot; enforces >= 2 args &middot; returns 1

> Returns the approximate sky visibility (sky ambient level) at a point as a number between 0 and 1. The value is not affected by weather or time of day. This value is used by the post fx shader after some temporal and spatial smoothing.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `pos_x` | `number` | `lua_tonumber` | - |
| 2 | `pos_y` | `number` | `lua_tonumber` | - |

Returns `sky:number`



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `GameGetVelocityCompVelocity`  
`0x007b6ba0` &middot; 1209 instructions &middot; enforces >= 1 arg &middot; returns 2

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |

Returns `x:number,y:number`

Runtime messages:
- `couldn't find entity with id:`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `GameGetWorldStateEntity`  
`0x007aa6f0` &middot; 730 instructions &middot; enforces >= 0 args &middot; returns 1

Returns `entity_id:int`

writes `0x01204bd0`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GameGiveAchievement`  
`0x007a6060` &middot; 745 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `id` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |



> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GameHasFlagRun`  
`0x007d64b0` &middot; 832 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `flag` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `bool`



> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GameIsBetaBuild`  
`0x007e3f30` &middot; 723 instructions &middot; enforces >= 0 args &middot; returns 1

Returns `bool`



> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GameIsDailyRun`  
`0x007c2990` &middot; 720 instructions &middot; enforces >= 0 args &middot; returns 1

Returns `bool`



> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GameIsDailyRunOrDailyPracticeRun`  
`0x007c2c60` &middot; 720 instructions &middot; enforces >= 0 args &middot; returns 1

Returns `bool`



> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GameIsIntroPlaying`  
`0x007aa130` &middot; 723 instructions &middot; enforces >= 0 args &middot; returns 1

Returns `bool`

reads `0x01154da4`

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GameIsInventoryOpen`  
`0x007b1400` &middot; 719 instructions &middot; enforces >= 0 args &middot; returns 1

Returns `bool`

reads `0x01222510`

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GameIsModeFullyDeterministic`  
`0x007c2f30` &middot; 720 instructions &middot; enforces >= 0 args &middot; returns 1

Returns `bool`



> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GameKillInventoryItem`  
`0x007afa90` &middot; 1370 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `inventory_owner_entity_id` | `int` | `lua_tointeger` | - |
| 2 | `item_entity_id` | `int` | `lua_tointeger` | - |

Runtime messages:
- `couldn't find entity with id:`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `GameOnCompleted`  
`0x007a5da0` &middot; 692 instructions &middot; enforces >= 0 args &middot; returns nothing



> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GamePickUpInventoryItem`  
`0x007afff0` &middot; 1249 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `who_picks_up_entity_id` | `int` | `lua_tointeger` | - |
| 2 | `item_entity_id` | `int` | `lua_tointeger` | - |
| 3 | `do_pick_up_effects` | `bool` | `lua_toboolean` | documented `true` |

Runtime messages:
- `couldn't find entity with id:`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `GamePlayAnimation`  
`0x007b65b0` &middot; 1509 instructions &middot; enforces >= 3 args &middot; returns nothing

> Plays animation. Follow up animation ('followup_name') is applied only if 'followup_priority' is given.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `priority` | `int` | `lua_tointeger` | - |
| 4 | `followup_name` | `string` | `lua_isstring`, `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |
| 5 | `followup_priority` | `int` | `lua_tointeger` | documented `0` |

Runtime messages:
- `couldn't find entity with id:`

reads `0x00fe3c84`, `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GamePlaySound`  
`0x007d7190` &middot; 1049 instructions &middot; enforces >= 4 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `bank_filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `event_path` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `x` | `number` | `lua_tonumber` | - |
| 4 | `y` | `number` | `lua_tonumber` | - |



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GamePosToPhysicsPos`  
`0x007d3d50` &middot; 880 instructions &middot; enforces >= 1 arg &middot; returns 2

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `number` | `lua_tonumber` | - |
| 2 | `y` | `number` | `lua_tonumber` | documented `0` |

Returns `x:number,y:number`

reads `0x01207b28`, `0x01207b30`, `0x01207b38`, `0x01207b40`

#### `GamePrint`  
`0x007be380` &middot; 816 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `log_line` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GamePrintImportant`  
`0x007be6b0` &middot; 1104 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `title` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `description` | `string` | `lua_isstring`, `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |
| 3 | `ui_custom_decoration_file` | `string` | `lua_isstring`, `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |

reads `0x00fe3c84`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GameRegenItemAction`  
`0x007aee00` &middot; 1065 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |

Runtime messages:
- `couldn't find entity with id:`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `GameRegenItemActionsInContainer`  
`0x007af230` &middot; 1065 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |

Runtime messages:
- `couldn't find entity with id:`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `GameRegenItemActionsInPlayer`  
`0x007af660` &middot; 1065 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |

Runtime messages:
- `couldn't find entity with id:`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `GameRemoveFlagRun`  
`0x007d6190` &middot; 792 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `flag` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |



> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GameScreenshake`  
`0x007a5a50` &middot; 834 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `strength` | `number` | `lua_tonumber` | - |
| 2 | `x` | `number` | `lua_tonumber` | documented `camera_x` |
| 3 | `y` | `number` | `lua_tonumber` | documented `camera_y` |

reads `0x0105361c` = 1056964608 / 0.5

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `GameSetCameraFree`  
`0x007ae790` &middot; 749 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `is_free` | `bool` | `lua_toboolean` | - |

reads `0x01221bc0`

#### `GameSetCameraPos`  
`0x007ae4a0` &middot; 737 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `number` | `lua_tonumber` | - |
| 2 | `y` | `number` | `lua_tonumber` | - |



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `GameSetFogOfWar`  
`0x007bbd40` &middot; 802 instructions &middot; enforces >= 3 args &middot; returns 1

> 'fog_of_war' should be between 0 and 255 (but will be clamped to the correct range with a int32->uint8 cast). Larger value means more coverage. Returns a boolean indicating whether or not the position was inside the bounds of the fog of war grid. For performance reasons consider using the components that manipulate fog of war.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `pos_x` | `number` | `lua_tonumber` | - |
| 2 | `pos_y` | `number` | `lua_tonumber` | - |
| 3 | `fog_of_war` | `int` | `lua_tointeger` | - |

Returns `pos_valid:bool`



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `GameSetPostFxParameter`  
`0x007d7f30` &middot; 867 instructions &middot; enforces >= 5 args &middot; returns nothing

> Can be used to pass custom parameters to the post_final shader, or override values set by the game code. The shader uniform called 'parameter_name' will be set to the latest given values on this and following frames.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `parameter_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `x` | `number` | `lua_tonumber` | - |
| 3 | `y` | `number` | `lua_tonumber` | - |
| 4 | `z` | `number` | `lua_tonumber` | - |
| 5 | `w` | `number` | `lua_tonumber` | - |



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GameSetPostFxTextureParameter`  
`0x007d85a0` &middot; 1762 instructions &middot; enforces >= 4 args &middot; returns nothing

> Can be used to pass 2D textures to the post_final shader. The shader uniform called 'parameter_name' will be set to the latest given value on this and following frames. 'texture_filename' can either point to a file, or a virtual file created using the ModImage API.
> If 'update_texture' is true, the texture will be re-uploaded to the GPU (could be useful with dynamic textures, but will incur a heavy performance hit with textures that are loaded from the disk).
> Accepted values for 'filtering_mode' and 'wrapping_mode' can be found in 'data/libs/utilities.lua'. Each call with a unique 'parameter_name' will create a separate texture while the parameter is in use, so this should be used with some care. While it's possible to change 'texture_filename' on the fly, if texture size changed, this causes destruction of the old texture and allocating a new one, which can be quite slow.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `parameter_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `texture_filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `filtering_mode` | `int` | `lua_tointeger` | - |
| 4 | `wrapping_mode` | `int` | `lua_tointeger` | - |
| 5 | `update_texture` | `bool` | `lua_toboolean` | documented `false` |

Runtime messages:
- `- should be 0-2`
- `- should be 0-4`
- `invalid filtering mode:`
- `invalid wrapping mode:`

reads `0x00000001`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GameShootProjectile`  
`0x007b3c30` &middot; 1009 instructions &middot; enforces >= 6 args &middot; returns nothing

> 'shooter_entity' can be 0. Warning: If 'projectile_entity' has PhysicsBodyComponent and ItemComponent, components without the "enabled_in_world" tag will be disabled, as if the entity was thrown by player.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `shooter_entity` | `int` | `lua_tointeger` | - |
| 2 | `x` | `number` | `lua_tonumber` | - |
| 3 | `y` | `number` | `lua_tonumber` | - |
| 4 | `target_x` | `number` | `lua_tonumber` | - |
| 5 | `target_y` | `number` | `lua_tonumber` | - |
| 6 | `projectile_entity` | `int` | `lua_tointeger` | - |
| 7 | `send_message` | `bool` | `lua_toboolean` | documented `true` |
| 8 | `verlet_parent_entity` | `int` | `lua_tointeger` | documented `0` |

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `GameTextGet`  
`0x007d9010` &middot; 1962 instructions &middot; enforces >= 1 arg &middot; returns 1 (of 5 push sites)

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `key` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `param0` | `string` | `lua_isstring`, `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |
| 3 | `param1` | `string` | `lua_isstring`, `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |
| 4 | `param2` | `string` | `lua_isstring`, `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |

Returns `string`

Runtime messages:
- `Invalid number of parameters`

reads `0x00fe3c84`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GameTextGetTranslatedOrNot`  
`0x007d8ca0` &middot; 876 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `text_or_key` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `string`



> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GameTriggerGameOver`  
`0x007b16d0` &middot; 700 instructions &middot; enforces >= 0 args &middot; returns nothing

reads `0x01204bc0`

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GameTriggerMusicCue`  
`0x007d6b70` &middot; 790 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GameTriggerMusicEvent`  
`0x007d67f0` &middot; 886 instructions &middot; enforces >= 4 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `event_path` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `can_be_faded` | `bool` | `lua_toboolean` | - |
| 3 | `x` | `number` | `lua_tonumber` | - |
| 4 | `y` | `number` | `lua_tonumber` | - |

reads `0x011549f0`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GameTriggerMusicFadeOutAndDequeueAll`  
`0x007d6e90` &middot; 758 instructions &middot; enforces >= 0 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `relative_fade_speed` | `number` | `lua_tonumber` | documented `1` |

reads `0x01053780` = 1065353216 / 1

> The arity guard is **vacuous** (`lua_gettop` is never negative), so even the 1 documented parameter(s) are not actually enforced.

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `GameUnsetPostFxParameter`  
`0x007d82a0` &middot; 758 instructions &middot; enforces >= 1 arg &middot; returns nothing

> Will remove a post_final shader parameter value binding set via game GameSetPostFxParameter().

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `parameter_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GameVecToPhysicsVec`  
`0x007d4430` &middot; 864 instructions &middot; enforces >= 1 arg &middot; returns 2

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `number` | `lua_tonumber` | - |
| 2 | `y` | `number` | `lua_tonumber` | documented `0` |

Returns `x:number,y:number`

reads `0x01207b38`

### Gui (41)

#### `GuiAnimateAlphaFadeIn`  
`0x007dc7d0` &middot; 846 instructions &middot; enforces >= 5 args &middot; returns nothing

> Does an alpha tween animation for all widgets inside a scope set using GuiAnimateBegin() and GuiAnimateEnd().

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `id` | `int` | `lua_tointeger` | - |
| 3 | `speed` | `number` | `lua_tonumber` | - |
| 4 | `step` | `number` | `lua_tonumber` | - |
| 5 | `reset` | `bool` | `lua_toboolean` | - |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiAnimateBegin`  
`0x007dc200` &middot; 736 instructions &middot; enforces >= 1 arg &middot; returns nothing

> Starts a scope where animations initiated using GuiAnimateAlphaFadeIn() etc. will be applied to all widgets.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiAnimateEnd`  
`0x007dc4e0` &middot; 742 instructions &middot; enforces >= 1 arg &middot; returns nothing

> Ends a scope where animations initiated using GuiAnimateAlphaFadeIn() etc. will be applied to all widgets.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiAnimateScaleIn`  
`0x007dcb20` &middot; 828 instructions &middot; enforces >= 4 args &middot; returns nothing

> Does a scale tween animation for all widgets inside a scope set using GuiAnimateBegin() and GuiAnimateEnd().

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `id` | `int` | `lua_tointeger` | - |
| 3 | `acceleration` | `number` | `lua_tonumber` | - |
| 4 | `reset` | `bool` | `lua_toboolean` | - |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiBeginAutoBox`  
`0x007dff60` &middot; 821 instructions &middot; enforces >= 1 arg &middot; returns nothing

> [Together with GuiEndAutoBoxNinePiece() this can be used to draw an auto-scaled background box for a bunch of widgets rendered between the calls.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiBeginScrollContainer`  
`0x007e0db0` &middot; 1319 instructions &middot; enforces >= 6 args &middot; returns nothing

> This can be used to create a container with a vertical scroll bar. Widgets between GuiBeginScrollContainer() and GuiEndScrollContainer() will be positioned relative to the container.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `id` | `int` | `lua_tointeger` | - |
| 3 | `x` | `number` | `lua_tonumber` | - |
| 4 | `y` | `number` | `lua_tonumber` | - |
| 5 | `width` | `number` | `lua_tonumber` | - |
| 6 | `height` | `number` | `lua_tonumber` | - |
| 7 | `scrollbar_gamepad_focusable` | `bool` | `lua_toboolean` | documented `true` |
| 8 | `margin_x` | `number` | `lua_tonumber` | documented `2` |
| 9 | `margin_y` | `number` | `lua_tonumber` | documented `2` |

writes `0x01154b98`, `0x01154b9c`, `0x01154ba0`, `0x01154ba4`, `0x01154ba8` &middot; reads `0x010539f8` = 1073741824 / 2

> Writes the process-global *previous widget* block (`0x01154b98`-`0x01154ba8`) that `GuiGetPreviousWidgetInfo` reads. Two `Gui` objects in one frame clobber each other here.

> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiButton`  
`0x007de5e0` &middot; 1677 instructions &middot; enforces >= 5 args &middot; returns 2

> The old parameter order where 'id' is the last parameter is still supported. The function dynamically picks the correct order based on the type of the 4th parameter.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `id` | `int` | `lua_tonumber`, `lua_tointeger` | - |
| 3 | `x` | `number` | `lua_tonumber` | - |
| 4 | `y` | `number` | `lua_type`, `lua_isstring`, `lua_tolstring`, `lua_tonumber` | -<br>hardcoded *this function's own usage string* |
| 5 | `text` | `string` | `lua_tointeger`, `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 6 | `scale` | `number` | `lua_tonumber` | documented `1` |
| 7 | `font` | `string` | `lua_isstring`, `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |
| 8 | `font_is_pixel_font` | `bool` | `lua_toboolean` | documented `true` |

Returns `clicked:bool,right_clicked:bool`

writes `0x01154b98`, `0x01154b9c`, `0x01154ba0`, `0x01154ba4`, `0x01154ba8` &middot; reads `0x01053780` = 1065353216 / 1

> Writes the process-global *previous widget* block (`0x01154b98`-`0x01154ba8`) that `GuiGetPreviousWidgetInfo` reads. Two `Gui` objects in one frame clobber each other here.

> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

> A missing/invalid string argument (argument 4) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GuiColorSetForNextWidget`  
`0x007daf40` &middot; 878 instructions &middot; enforces >= 5 args &middot; returns nothing

> Sets the color of the next widget during this frame. Color components should be in the 0-1 range.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `red` | `number` | `lua_tonumber` | - |
| 3 | `green` | `number` | `lua_tonumber` | - |
| 4 | `blue` | `number` | `lua_tonumber` | - |
| 5 | `alpha` | `number` | `lua_tonumber` | - |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiCreate`  
`0x007d9920` &middot; 866 instructions &middot; enforces >= 0 args &middot; returns 1

Returns `gui:obj`

Runtime messages:
- `LuaImGui`



> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GuiDestroy`  
`0x007d9c90` &middot; 737 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiEndAutoBoxNinePiece`  
`0x007e02a0` &middot; 1574 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `margin` | `number` | `lua_tonumber` | documented `5` |
| 3 | `size_min_x` | `number` | `lua_tonumber` | documented `0` |
| 4 | `size_min_y` | `number` | `lua_tonumber` | documented `0` |
| 5 | `mirrorize_over_x_axis` | `bool` | `lua_toboolean` | documented `false` |
| 6 | `x_axis` | `number` | `lua_tonumber` | documented `0` |
| 7 | `sprite_filename` | `string` | `lua_isstring`, `lua_tolstring` | documented `"data/ui_gfx/decorations/9piece0_gray.png"`<br>hardcoded *this function's own usage string* |
| 8 | `sprite_highlight_filename` | `string` | `lua_isstring`, `lua_tolstring` | documented `"data/ui_gfx/decorations/9piece0_gray.png"`<br>hardcoded *this function's own usage string* |

Runtime messages:
- `data/ui_gfx/decorations/9piece0_gray.png`

writes `0x01154b98`, `0x01154b9c`, `0x01154ba0`, `0x01154ba4`, `0x01154ba8` &middot; reads `0x01053c18` = 1084227584 / 5

> Writes the process-global *previous widget* block (`0x01154b98`-`0x01154ba8`) that `GuiGetPreviousWidgetInfo` reads. Two `Gui` objects in one frame clobber each other here.

> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

> A missing/invalid string argument (argument 7) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GuiEndScrollContainer`  
`0x007e12e0` &middot; 736 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiGetImageDimensions`  
`0x007e36a0` &middot; 1175 instructions &middot; enforces >= 2 args &middot; returns 2

> Returns size of the given image in the gui coordinate system.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type` | - |
| 2 | `image_filename` | `string` | `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `scale` | `number` | - | documented `1` |

Returns `width:number,height:number`

reads `0x01053780` = 1065353216 / 1

> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GuiGetPreviousWidgetInfo`  
`0x007e3b40` &middot; 1007 instructions &middot; enforces >= 1 arg &middot; returns 11

> Returns the final position, size etc calculated for a widget. Some values aren't supported by all widgets.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |

Returns `clicked:bool, right_clicked:bool, hovered:bool, x:number, y:number, width:number, height:number, draw_x:number, draw_y:number, draw_width:number, draw_height:number`

writes `0x01225a94` &middot; reads `0x01225fb0`

> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiGetScreenDimensions`  
`0x007e2da0` &middot; 958 instructions &middot; enforces >= 1 arg &middot; returns 2

> Returns dimensions of viewport in the gui coordinate system (which is equal to the coordinates of the screen bottom right corner in gui coordinates). The values returned may change depending on the game resolution because the UI is scaled for pixel-perfect text rendering.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |

Returns `width:number,height:number`

reads `0x01221bc0`

> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiGetTextDimensions`  
`0x007e3160` &middot; 1341 instructions &middot; enforces >= 2 args &middot; returns 2

> Returns size of the given text in the gui coordinate system.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type` | - |
| 2 | `text` | `string` | `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `scale` | `number` | - | documented `1` |
| 4 | `line_spacing` | `number` | - | documented `2` |
| 5 | `font` | `string` | `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |
| 6 | `font_is_pixel_font` | `bool` | - | documented `true` |

Returns `width:number,height:number`

reads `0x01053780` = 1065353216 / 1, `0x010539f8` = 1073741824 / 2

> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GuiIdPop`  
`0x007dbf00` &middot; 756 instructions &middot; enforces >= 1 arg &middot; returns nothing

> See GuiIdPush().

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiIdPush`  
`0x007db8b0` &middot; 757 instructions &middot; enforces >= 2 args &middot; returns nothing

> Can be used to solve ID conflicts. All ids given to Gui* functions will be hashed with the ids stacked (and hashed together) using GuiIdPush() and GuiIdPop(). The id stack has a max size of 1024, and calls to the function will do nothing if the size is exceeded.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `id` | `int` | `lua_tointeger` | - |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiIdPushString`  
`0x007dbbb0` &middot; 834 instructions &middot; enforces >= 2 args &middot; returns nothing

> Pushes the hash of 'str' as a gui id. See GuiIdPush().

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type` | - |
| 2 | `str` | `string` | `lua_tolstring` | -<br>hardcoded *this function's own usage string* |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GuiImage`  
`0x007dd900` &middot; 1701 instructions &middot; enforces >= 5 args &middot; returns nothing

> 'scale' will be used for 'scale_y' if 'scale_y' equals 0.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `id` | `int` | `lua_tointeger` | - |
| 3 | `x` | `number` | `lua_tonumber` | - |
| 4 | `y` | `number` | `lua_tonumber` | - |
| 5 | `sprite_filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 6 | `alpha` | `number` | `lua_tonumber` | documented `1` |
| 7 | `scale` | `number` | `lua_tonumber` | documented `1` |
| 8 | `scale_y` | `number` | `lua_tonumber` | documented `0` |
| 9 | `rotation` | `number` | `lua_tonumber` | documented `0` |
| 10 | `rect_animation_playback_type` | `int` | `lua_tointeger` | documented `GUI_RECT_ANIMATION_PLAYBACK.PlayToEndAndHide` |
| 11 | `rect_animation_name` | `string` | - | documented `""` |

writes `0x01154b98`, `0x01154b9c`, `0x01154ba0`, `0x01154ba4`, `0x01154ba8` &middot; reads `0x00fe3c84`

> Writes the process-global *previous widget* block (`0x01154b98`-`0x01154ba8`) that `GuiGetPreviousWidgetInfo` reads. Two `Gui` objects in one frame clobber each other here.

> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

> A missing/invalid string argument (argument 5) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GuiImageButton`  
`0x007dec70` &middot; 1493 instructions &middot; enforces >= 6 args &middot; returns 2

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `id` | `int` | `lua_tointeger` | - |
| 3 | `x` | `number` | `lua_tonumber` | - |
| 4 | `y` | `number` | `lua_tonumber` | - |
| 5 | `text` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 6 | `sprite_filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `clicked:bool,right_clicked:bool`

writes `0x01154b98`, `0x01154b9c`, `0x01154ba0`, `0x01154ba4`, `0x01154ba8`

> Writes the process-global *previous widget* block (`0x01154b98`-`0x01154ba8`) that `GuiGetPreviousWidgetInfo` reads. Two `Gui` objects in one frame clobber each other here.

> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

> A missing/invalid string argument (argument 5) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GuiImageNinePiece`  
`0x007ddfb0` &middot; 1571 instructions &middot; enforces >= 6 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `id` | `int` | `lua_tointeger` | - |
| 3 | `x` | `number` | `lua_tonumber` | - |
| 4 | `y` | `number` | `lua_tonumber` | - |
| 5 | `width` | `number` | `lua_tonumber` | - |
| 6 | `height` | `number` | `lua_tonumber` | - |
| 7 | `alpha` | `number` | `lua_tonumber` | documented `1` |
| 8 | `sprite_filename` | `string` | `lua_isstring`, `lua_tolstring` | documented `"data/ui_gfx/decorations/9piece0_gray.png"`<br>hardcoded *this function's own usage string* |
| 9 | `sprite_highlight_filename` | `string` | `lua_isstring`, `lua_tolstring` | documented `"data/ui_gfx/decorations/9piece0_gray.png"`<br>hardcoded *this function's own usage string* |

Runtime messages:
- `data/ui_gfx/decorations/9piece0_gray.png`

writes `0x01154b98`, `0x01154b9c`, `0x01154ba0`, `0x01154ba4`, `0x01154ba8` &middot; reads `0x01053780` = 1065353216 / 1

> Writes the process-global *previous widget* block (`0x01154b98`-`0x01154ba8`) that `GuiGetPreviousWidgetInfo` reads. Two `Gui` objects in one frame clobber each other here.

> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

> A missing/invalid string argument (argument 8) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GuiLayoutAddHorizontalSpacing`  
`0x007e1e90` &middot; 791 instructions &middot; enforces >= 1 arg &middot; returns nothing

> Will use the horizontal margin from current layout if amount is not set.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `amount` | `number` | `lua_tonumber` | documented `optional` |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiLayoutAddVerticalSpacing`  
`0x007e21b0` &middot; 825 instructions &middot; enforces >= 1 arg &middot; returns nothing

> Will use the vertical margin from current layout if amount is not set.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `amount` | `number` | `lua_tonumber` | documented `optional` |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiLayoutBeginHorizontal`  
`0x007e15c0` &middot; 1122 instructions &middot; enforces >= 3 args &middot; returns nothing

> If 'position_in_ui_scale' is 1, x and y will be in the same scale as other gui positions, otherwise x and y are given as a percentage (0-100) of the gui screen size.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `x` | `number` | `lua_tonumber` | - |
| 3 | `y` | `number` | `lua_tonumber` | - |
| 4 | `position_in_ui_scale` | `bool` | `lua_toboolean` | documented `false` |
| 5 | `margin_x` | `number` | `lua_tonumber` | documented `2` |
| 6 | `margin_y` | `number` | `lua_tonumber` | documented `2` |

reads `0x01053464` = 1008981770 / 0.01, `0x010539f8` = 1073741824 / 2, `0x01221bcc`, `0x01221bd0`

> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiLayoutBeginLayer`  
`0x007e27d0` &middot; 738 instructions &middot; enforces >= 1 arg &middot; returns nothing

> Puts following things to a new layout layer. Can be used to create non-layouted widgets inside a layout.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiLayoutBeginVertical`  
`0x007e1a30` &middot; 1117 instructions &middot; enforces >= 3 args &middot; returns nothing

> If 'position_in_ui_scale' is 1, x and y will be in the same scale as other gui positions, otherwise x and y are given as a percentage (0-100) of the gui screen size.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `x` | `number` | `lua_tonumber` | - |
| 3 | `y` | `number` | `lua_tonumber` | - |
| 4 | `position_in_ui_scale` | `bool` | `lua_toboolean` | documented `false` |
| 5 | `margin_x` | `number` | `lua_tonumber` | documented `0` |
| 6 | `margin_y` | `number` | `lua_tonumber` | documented `0` |

reads `0x01053464` = 1008981770 / 0.01, `0x01221bcc`, `0x01221bd0`

> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiLayoutEnd`  
`0x007e24f0` &middot; 736 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiLayoutEndLayer`  
`0x007e2ac0` &middot; 736 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiOptionsAdd`  
`0x007da320` &middot; 774 instructions &middot; enforces >= 2 args &middot; returns nothing

> Sets the options that apply to widgets during this frame. For 'option' use the values in the GUI_OPTION table in "data/scripts/lib/utilities.lua". Values from consecutive calls will be combined. For example calling this with the values GUI_OPTION.Align_Left and GUI_OPTION.GamepadDefaultWidget will set both options for the next widget. The options will be cleared on next call to GuiStartFrame().

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `option` | `int` | `lua_tointeger` | - |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiOptionsAddForNextWidget`  
`0x007dac30` &middot; 774 instructions &middot; enforces >= 2 args &middot; returns nothing

> [Sets the options that apply to the next widget during this frame. For 'option' use the values in the GUI_OPTION table in "data/scripts/lib/utilities.lua". Values from consecutive calls will be combined. For example calling this with the values GUI_OPTION.Align_Left and GUI_OPTION.GamepadDefaultWidget will set both options for the next widget.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `option` | `int` | `lua_tointeger` | - |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiOptionsClear`  
`0x007da940` &middot; 743 instructions &middot; enforces >= 1 arg &middot; returns nothing

> Clears the options that apply to widgets during this frame.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiOptionsRemove`  
`0x007da630` &middot; 778 instructions &middot; enforces >= 2 args &middot; returns nothing

> Sets the options that apply to widgets during this frame. For 'option' use the values in the GUI_OPTION table in "data/scripts/lib/utilities.lua". Values from consecutive calls will be combined. For example calling this with the values GUI_OPTION.Align_Left and GUI_OPTION.GamepadDefaultWidget will set both options for the next widget. The options will be cleared on next call to GuiStartFrame().

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `option` | `int` | `lua_tointeger` | - |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiSlider`  
`0x007df250` &middot; 1662 instructions &middot; enforces >= 11 args &middot; returns 1

> This is not intended to be outside mod settings menu, and might bug elsewhere.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type` | - |
| 2 | `id` | `int` | - | - |
| 3 | `x` | `number` | `lua_tonumber` | - |
| 4 | `y` | `number` | `lua_tonumber` | - |
| 5 | `text` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 6 | `value` | `number` | - | - |
| 7 | `value_min` | `number` | `lua_tonumber` | - |
| 8 | `value_max` | `number` | `lua_tonumber` | - |
| 9 | `value_default` | `number` | `lua_tonumber` | - |
| 10 | `value_display_multiplier` | `number` | `lua_tonumber` | - |
| 11 | `value_formatting` | `string` | - | - |
| 12 | `width` | `number` | - | - |

Returns `new_value:number`

Runtime messages:
- `data/fonts/font_pixel_noshadow.xml`

writes `0x01154b98`, `0x01154b9c`, `0x01154ba0`, `0x01154ba4`, `0x01154ba8` &middot; reads `0x01207c38`, `0x01207c58`

> **The binary enforces 11 arguments, fewer than the 12 the signature marks as required** (12 declared). Trust the enforced number.

> Writes the process-global *previous widget* block (`0x01154b98`-`0x01154ba8`) that `GuiGetPreviousWidgetInfo` reads. Two `Gui` objects in one frame clobber each other here.

> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

> A missing/invalid string argument (argument 5) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GuiStartFrame`  
`0x007d9f80` &middot; 756 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiText`  
`0x007dce60` &middot; 1440 instructions &middot; enforces >= 4 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `x` | `number` | `lua_tonumber` | - |
| 3 | `y` | `number` | `lua_tonumber` | - |
| 4 | `text` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 5 | `scale` | `number` | `lua_tonumber` | documented `1` |
| 6 | `font` | `string` | `lua_isstring`, `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |
| 7 | `font_is_pixel_font` | `bool` | `lua_toboolean` | documented `true` |

writes `0x01154b98`, `0x01154b9c`, `0x01154ba0`, `0x01154ba4`, `0x01154ba8` &middot; reads `0x01053780` = 1065353216 / 1

> Writes the process-global *previous widget* block (`0x01154b98`-`0x01154ba8`) that `GuiGetPreviousWidgetInfo` reads. Two `Gui` objects in one frame clobber each other here.

> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

> A missing/invalid string argument (argument 4) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GuiTextCentered`  
`0x007dd400` &middot; 1269 instructions &middot; enforces >= 4 args &middot; returns nothing

> Deprecated. Use GuiOptionsAdd() or GuiOptionsAddForNextWidget() with GUI_OPTION.Align_HorizontalCenter and GuiText() instead.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `x` | `number` | `lua_tonumber` | - |
| 3 | `y` | `number` | `lua_tonumber` | - |
| 4 | `text` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

writes `0x01154b98`, `0x01154b9c`, `0x01154ba0`, `0x01154ba4`, `0x01154ba8`

> Writes the process-global *previous widget* block (`0x01154b98`-`0x01154ba8`) that `GuiGetPreviousWidgetInfo` reads. Two `Gui` objects in one frame clobber each other here.

> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

> A missing/invalid string argument (argument 4) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GuiTextInput`  
`0x007df8d0` &middot; 1677 instructions &middot; enforces >= 7 args &middot; returns 1

> 'allowed_characters' should consist only of ASCII characters. This is not intended to be outside mod settings menu, and might bug elsewhere.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `id` | `int` | `lua_tointeger` | - |
| 3 | `x` | `number` | `lua_tonumber` | - |
| 4 | `y` | `number` | `lua_tonumber` | - |
| 5 | `text` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 6 | `width` | `number` | `lua_tonumber` | - |
| 7 | `max_length` | `int` | `lua_tointeger` | - |
| 8 | `allowed_characters` | `string` | `lua_isstring`, `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |

Returns `new_text`

Runtime messages:
- `data/fonts/font_pixel_noshadow.xml`

writes `0x01154b98`, `0x01154b9c`, `0x01154ba0`, `0x01154ba4`, `0x01154ba8` &middot; reads `0x01207c38`, `0x01207c58`

> Writes the process-global *previous widget* block (`0x01154b98`-`0x01154ba8`) that `GuiGetPreviousWidgetInfo` reads. Two `Gui` objects in one frame clobber each other here.

> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

> A missing/invalid string argument (argument 5) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GuiTooltip`  
`0x007e08d0` &middot; 1243 instructions &middot; enforces >= 3 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `text` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `description` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Runtime messages:
- `data/ui_gfx/decorations/9piece0_gray.png`

writes `0x01154bac` = 256 &middot; reads `0x01154b98`

> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GuiZSet`  
`0x007db2b0` &middot; 762 instructions &middot; enforces >= 2 args &middot; returns nothing

> Sets the rendering depth ('z') of the widgets following this call. Larger z = deeper. The z will be set to 0 on the next call to GuiStartFrame(). 

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `z` | `float` | `lua_tonumber` | - |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

#### `GuiZSetForNextWidget`  
`0x007db5b0` &middot; 766 instructions &middot; enforces >= 2 args &middot; returns nothing

> [Sets the rendering depth ('z') of the next widget following this call. Larger z = deeper.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `gui` | `obj` | `lua_type`, `lua_topointer` | - |
| 2 | `z` | `float` | `lua_tonumber` | - |



> Validates the `gui` handle; a stale handle makes the whole call a **silent no-op** with no log line.

### Physics (31)

#### `PhysicsAddBodyCreateBox`  
`0x007cc4d0` &middot; 1981 instructions &middot; enforces >= 6 args &middot; returns 1

> Does not work with PhysicsBody2Component. Returns the id of the created physics body.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | - | - |
| 2 | `material` | `string` | - | -<br>hardcoded *this function's own usage string* |
| 3 | `offset_x` | `number` | - | - |
| 4 | `offset_y` | `number` | - | - |
| 5 | `width` | `int` | - | - |
| 6 | `height` | `int` | - | - |
| 7 | `centered` | `bool` | - | documented `false` |

Returns `int|nil`

Runtime messages:
- `couldn't find entity with id:`
- `width and height <= 0:`

reads `0x00ff8670` = 8236, `0x010145f8` = wood, `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `PhysicsAddBodyImage`  
`0x007cbbf0` &middot; 2265 instructions &middot; enforces >= 2 args &middot; returns 1

> Does not work with PhysicsBody2Component. Returns the id of the created physics body.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | - | - |
| 2 | `image_file` | `string` | - | -<br>hardcoded *this function's own usage string* |
| 3 | `material` | `string` | - | documented `""`<br>hardcoded *this function's own usage string* |
| 4 | `offset_x` | `number` | - | documented `0` |
| 5 | `offset_y` | `number` | - | documented `0` |
| 6 | `centered` | `bool` | - | documented `false` |
| 7 | `is_circle` | `bool` | - | documented `false` |
| 8 | `material_image_file` | `string` | - | documented `""`<br>hardcoded *this function's own usage string* |
| 9 | `use_image_as_colors` | `bool` | - | documented `true` |

Returns `int_body_id`

Runtime messages:
- `couldn't find entity with id:`
- `couldn't load image file:`

reads `0x00fe3c84`, `0x010145f8` = wood, `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `PhysicsAddJoint`  
`0x007ccc90` &middot; 1343 instructions &middot; enforces >= 3 args &middot; returns 1

> Does not work with PhysicsBody2Component. Returns the id of the created joint.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `body_id0` | `int` | `lua_tointeger` | - |
| 3 | `body_id1` | `int` | `lua_tointeger` | - |
| 4 | `offset_x` | `number` | `lua_tonumber` | - |
| 5 | `offset_y` | `number` | `lua_tonumber` | - |
| 6 | `joint_type` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 7 | (unnamed) | - | `lua_toboolean` | - |

Returns `int|nil`

Runtime messages:
- `couldn't find entity with id:`

reads `0x01204b98`

> **The binary enforces 3 arguments, fewer than the 6 the signature marks as required** (6 declared). Trust the enforced number.

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 6) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `PhysicsApplyForce`  
`0x007cd1d0` &middot; 1045 instructions &middot; enforces >= 3 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `helper:entity` | - |
| 2 | `force_x` | `number` | `lua_tonumber` | - |
| 3 | `force_y` | `number` | `lua_tonumber` | - |

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `PhysicsApplyForceOnArea`  
`0x007cdfd0` &middot; 1191 instructions &middot; enforces >= 6 args &middot; returns 1

> [Applies a force calculated by 'calculate_force_for_body_fn' to all bodies in an area. 'calculate_force_for_body_fn' should be a lua function with the following signature: function( body_entity:int, body_mass:number, body_x:number, body_y:number, body_vel_x:number, body_vel_y:number, body_vel_angular:number ) -> force_world_pos_x:number,force_world_pos_y:number,force_x:number,force_y:number,force_angular:number

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `calculate_force_for_body_fn` | `function` | `lua_type` | - |
| 2 | `ignore_this_entity` | `int` | `lua_tointeger` | - |
| 3 | `area_min_x` | `number` | `lua_tonumber` | - |
| 4 | `area_min_y` | `number` | `lua_tonumber` | - |
| 5 | `area_max_x` | `number` | `lua_tonumber` | - |
| 6 | `area_max_y` | `number` | `lua_tonumber` | - |

Runtime messages:
- `PhysicsApplyForceOnArea - expected function for parameter 0, but got a`



#### `PhysicsApplyTorque`  
`0x007cd5f0` &middot; 991 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `helper:entity` | - |
| 2 | `torque` | `number` | `lua_tonumber` | - |

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `PhysicsApplyTorqueToComponent`  
`0x007cd9d0` &middot; 1535 instructions &middot; enforces >= 3 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `component_id` | `int` | `lua_tointeger` | - |
| 3 | `torque` | `number` | `lua_tonumber` | - |

Runtime messages:
- `component with id`
- `couldn't find component with id:`
- `couldn't find entity with id:`
- `is not a PhysicsBodyComponent or PhysicsBody2Component`

reads `0x01204b98`, `0x01208018`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `PhysicsBody2InitFromComponents`  
`0x007d3700` &middot; 715 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `helper:entity` | - |



#### `PhysicsBodyIDApplyForce`  
`0x007d13f0` &middot; 1405 instructions &middot; enforces >= 3 args &middot; returns nothing

> [NOTE! force is in box2d units. world_pos_ is game world coordinates. If world_pos is not given will use the objects center as the position of where the force will be applied.],

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `physics_body_id` | `int` | `helper:number` | - |
| 2 | `force_x` | `number` | `lua_tonumber` | - |
| 3 | `force_y` | `number` | `lua_tonumber` | - |
| 4 | `world_pos_x` | `number` | - | documented `nil` |
| 5 | `world_pos_y` | `number` | `lua_tonumber` | documented `nil` |

Runtime messages:
- `Locked at:`
- `d:\projects\ ollagames\fallingeverything\build\vc12\..\..\source\lua\lua_api.cpp`
- `d:\projects ollagames\fallingeverything\build\vc12\..\..\source\lua\lua_api.cpp`

writes `0x0122374c` &middot; reads `0x01053a48`, `0x01207b28`, `0x01207b30`, `0x01207b38`, `0x01207b40`, `0x0120866c`, `0x01221bc0`

#### `PhysicsBodyIDApplyLinearImpulse`  
`0x007d1970` &middot; 1322 instructions &middot; enforces >= 3 args &middot; returns nothing

> [NOTE! impulse is in box2d units. world_pos_ is game world coordinates. If world_pos is not given will use the objects center as the position of where the force will be applied.],

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `physics_body_id` | `int` | `helper:number` | - |
| 2 | `force_x` | `number` | `lua_tonumber` | - |
| 3 | `force_y` | `number` | `lua_tonumber` | - |
| 4 | `world_pos_x` | `number` | - | documented `nil` |
| 5 | `world_pos_y` | `number` | `lua_tonumber` | documented `nil` |

Runtime messages:
- `Locked at:`
- `d:\projects\ ollagames\fallingeverything\build\vc12\..\..\source\lua\lua_api.cpp`
- `d:\projects ollagames\fallingeverything\build\vc12\..\..\source\lua\lua_api.cpp`

writes `0x0122374c` &middot; reads `0x01053a48`, `0x01207b28`, `0x01207b30`, `0x01207b38`, `0x01207b40`, `0x0120866c`, `0x01221bc0`

#### `PhysicsBodyIDApplyTorque`  
`0x007d1ea0` &middot; 1134 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `physics_body_id` | `int` | `helper:number` | - |
| 2 | `torque` | `number` | `lua_tonumber` | - |

Runtime messages:
- `Locked at:`
- `d:\projects\ ollagames\fallingeverything\build\vc12\..\..\source\lua\lua_api.cpp`
- `d:\projects ollagames\fallingeverything\build\vc12\..\..\source\lua\lua_api.cpp`

writes `0x0122374c` &middot; reads `0x01053a48`, `0x0120866c`, `0x01221bc0`

#### `PhysicsBodyIDGetBodyAABB`  
`0x007d3380` &middot; 885 instructions &middot; enforces >= 1 arg &middot; returns 4

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `physics_body_id` | `int` | `helper:number` | - |

Returns `nil`



#### `PhysicsBodyIDGetDamping`  
`0x007d2630` &middot; 827 instructions &middot; enforces >= 1 arg &middot; returns 2

> NOTE! returns nil, if body was not found. Results are 0-1. 

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `physics_body_id` | `int` | `helper:number` | - |

Returns `linear_damping:number, angular_damping:number`



#### `PhysicsBodyIDGetFromEntity`  
`0x007d00b0` &middot; 1247 instructions &middot; enforces >= 1 arg &middot; returns 1

> NOTE! If component_id is given, will return all the bodies linked to that component. If component_id is not given, will return all the bodies linked to the entity (with joints or through components).

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `helper:entity` | - |
| 2 | `component_id` | `int` | `helper:component` | documented `0` |

Returns `{physics_body_id}`

writes `0x01225ccc`, `0x01225d28`, `0x01225cd0` &middot; reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `PhysicsBodyIDGetGravityScale`  
`0x007d2d80` &middot; 794 instructions &middot; enforces >= 1 arg &middot; returns 1

> NOTE! returns nil, if body was not found. 

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `physics_body_id` | `int` | `helper:number` | - |

Returns `gravity_scale:number`



#### `PhysicsBodyIDGetTransform`  
`0x007d0b20` &middot; 924 instructions &middot; enforces >= 1 arg &middot; returns 6

> NOTE! returns nil, if body was not found. Results are Box2D units. Velocities need to converted with PhysicsVecToGameVec.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `physics_body_id` | `int` | `helper:number` | - |

Returns `nil | x:number, y:number, angle:number, vel_x:number, vel_y:number, angular_vel:number`



#### `PhysicsBodyIDGetWorldCenter`  
`0x007d2310` &middot; 787 instructions &middot; enforces >= 1 arg &middot; returns 2

> NOTE! returns nil, if body was not found. Results are Box2D units. 

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `physics_body_id` | `int` | `helper:number` | - |

Returns `x:number, y:number`



#### `PhysicsBodyIDQueryBodies`  
`0x007d05a0` &middot; 1399 instructions &middot; enforces >= 4 args &middot; returns 1

> NOTE! returns an array of physics_body_id(s) of all the box2d bodies in the given area. The default coordinates are in game world space. If passing a sixth argument with true, we will assume the coordinates are in box2d units. 

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `world_pos_min_x` | `number` | `lua_tonumber` | - |
| 2 | `world_pos_min_y` | `number` | `lua_tonumber` | - |
| 3 | `world_pos_max_x` | `number` | `lua_tonumber` | - |
| 4 | `world_pos_max_y` | `number` | `lua_tonumber` | - |
| 5 | `include_static_bodies` | `boolean` | - | documented `false` |
| 6 | `are_these_box2d_units` | `boolean` | - | documented `false` |

Returns `{physics_body_id}`

writes `0x01225c2c`, `0x01225c34`, `0x01225c30` &middot; reads `0x01207b28`, `0x01207b30`, `0x01207b38`, `0x01207b40`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `PhysicsBodyIDSetDamping`  
`0x007d2970` &middot; 1037 instructions &middot; enforces >= 2 args &middot; returns nothing

> NOTE! if angular_damping is given will set it as well.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `physics_body_id` | `int` | `helper:number` | - |
| 2 | `linear_damping` | `number` | `lua_tonumber` | - |
| 3 | `angular_damping` | `number` | `lua_tonumber` | documented `nil` |

Runtime messages:
- `Locked at:`
- `d:\projects\ ollagames\fallingeverything\build\vc12\..\..\source\lua\lua_api.cpp`
- `d:\projects ollagames\fallingeverything\build\vc12\..\..\source\lua\lua_api.cpp`

reads `0x01053a48`, `0x0120866c`, `0x01221bc0`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `PhysicsBodyIDSetGravityScale`  
`0x007d30a0` &middot; 729 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `physics_body_id` | `int` | `helper:number` | - |
| 2 | `gravity_scale` | `number` | `lua_tonumber` | - |



#### `PhysicsBodyIDSetTransform`  
`0x007d0ec0` &middot; 1324 instructions &middot; enforces >= 3 args &middot; returns nothing

> Requires min 3 first parameters.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `physics_body_id` | `int` | `helper:number` | - |
| 2 | `x` | `number` | - | - |
| 3 | `y` | `number` | `lua_tonumber` | - |
| 4 | `angle` | `number` | - | - |
| 5 | `vel_x` | `number` | - | - |
| 6 | `vel_y` | `number` | `lua_tonumber` | - |
| 7 | `angular_vel` | `number` | - | - |

Runtime messages:
- `Locked at:`
- `d:\projects\ ollagames\fallingeverything\build\vc12\..\..\source\lua\lua_api.cpp`
- `d:\projects ollagames\fallingeverything\build\vc12\..\..\source\lua\lua_api.cpp`

reads `0x01053a48`, `0x0120866c`, `0x01221bc0`

> **The binary enforces 3 arguments, fewer than the 7 the signature marks as required** (7 declared). Trust the enforced number.

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `PhysicsComponentGetTransform`  
`0x007cf510` &middot; 1296 instructions &middot; enforces >= 1 arg &middot; returns 6

> NOTE! results are Box2D units. Velocities need to converted with PhysicsVecToGameVec.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger`, `helper:component` | - |

Returns `x:number, y:number, angle:number, vel_x:number, vel_y:number, angular_vel:number`

Runtime messages:
- `couldn't find a valid physics body for component`



#### `PhysicsComponentSetTransform`  
`0x007cfa20` &middot; 1676 instructions &middot; enforces >= 3 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `component_id` | `int` | `lua_tointeger`, `helper:component` | - |
| 2 | `x` | `number` | - | - |
| 3 | `y` | `number` | `lua_tonumber` | - |
| 4 | `angle` | `number` | - | - |
| 5 | `vel_x` | `number` | - | - |
| 6 | `vel_y` | `number` | `lua_tonumber` | - |
| 7 | `angular_vel` | `number` | - | - |

Runtime messages:
- `Locked at:`
- `couldn't find a valid physics body for component`
- `d:\projects\ ollagames\fallingeverything\build\vc12\..\..\source\lua\lua_api.cpp`
- `d:\projects ollagames\fallingeverything\build\vc12\..\..\source\lua\lua_api.cpp`

reads `0x01053a48`, `0x0120866c`, `0x01221bc0`

> **The binary enforces 3 arguments, fewer than the 7 the signature marks as required** (7 declared). Trust the enforced number.

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `PhysicsGetComponentAngularVelocity`  
`0x007cf190` &middot; 886 instructions &middot; enforces >= 2 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `helper:entity` | - |
| 2 | `component_id` | `int` | - | - |

Returns `vel:number`



#### `PhysicsGetComponentVelocity`  
`0x007cee10` &middot; 888 instructions &middot; enforces >= 2 args &middot; returns 2

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `helper:entity` | - |
| 2 | `component_id` | `int` | - | - |

Returns `vel_x:number,vel_y:number`



#### `PhysicsPosToGamePos`  
`0x007d39d0` &middot; 892 instructions &middot; enforces >= 1 arg &middot; returns 2

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `number` | `lua_tonumber` | - |
| 2 | `y` | `number` | `lua_tonumber` | documented `0` |

Returns `x:number,y:number`

reads `0x01207b28`, `0x01207b30`, `0x01207b38`, `0x01207b40`

#### `PhysicsRemoveJoints`  
`0x007ce480` &middot; 1332 instructions &middot; enforces >= 4 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `world_pos_min_x` | `number` | `lua_tonumber` | - |
| 2 | `world_pos_min_y` | `number` | `lua_tonumber` | - |
| 3 | `world_pos_max_x` | `number` | `lua_tonumber` | - |
| 4 | `world_pos_max_y` | `number` | `lua_tonumber` | - |

writes `0x0122374c` &middot; reads `0x01207b28`, `0x01207b30`, `0x01207b38`, `0x01207b40`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `PhysicsSetStatic`  
`0x007ce9c0` &middot; 1095 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `is_static` | `bool` | `lua_toboolean` | - |

Runtime messages:
- `couldn't find entity with id:`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `PhysicsVecToGameVec`  
`0x007d40c0` &middot; 868 instructions &middot; enforces >= 1 arg &middot; returns 2

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `number` | `lua_tonumber` | - |
| 2 | `y` | `number` | `lua_tonumber` | documented `0` |

Returns `x:number,y:number`

reads `0x01207b38`, `0x01207b40`

#### `VerletApplyCircularForce`  
`0x007d4d90` &middot; 882 instructions &middot; enforces >= 4 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `world_pos_x` | `number` | `lua_tonumber` | - |
| 2 | `world_pos_y` | `number` | `lua_tonumber` | - |
| 3 | `radius` | `number` | `lua_tonumber` | - |
| 4 | `force` | `number` | `lua_tonumber` | - |

writes `0x012236e8`, `0x012236f4`, `0x012236f8`, `0x01223708`, `0x012236e0`, `0x012236ec`, `0x012236f0`, `0x012236fc`

#### `VerletApplyDirectionalForce`  
`0x007d5110` &middot; 910 instructions &middot; enforces >= 5 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `world_pos_x` | `number` | `lua_tonumber` | - |
| 2 | `world_pos_y` | `number` | `lua_tonumber` | - |
| 3 | `radius` | `number` | `lua_tonumber` | - |
| 4 | `force_x` | `number` | `lua_tonumber` | - |
| 5 | `force_y` | `number` | `lua_tonumber` | - |

writes `0x012236e8`, `0x012236f4`, `0x012236f8`, `0x01223708`, `0x012236e0`, `0x012236ec`, `0x012236f0`, `0x012236fc`

### Materials & cells (16)

#### `AddMaterialInventoryMaterial`  
`0x007a4b10` &middot; 1237 instructions &middot; enforces >= 3 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `material_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `count` | `int` | `lua_tointeger` | - |

Runtime messages:
- `couldn't find entity with id:`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `CellFactory_GetAllFires`  
`0x007ad1e0` &middot; 855 instructions &middot; enforces >= 0 args &middot; returns via a helper

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `include_statics` | `bool` | `lua_toboolean` | documented `true` |
| 2 | `include_particle_fx_materials` | `bool` | `lua_toboolean` | documented `false` |

Returns `{string}`



> The arity guard is **vacuous** (`lua_gettop` is never negative), so even the 2 documented parameter(s) are not actually enforced.

> Returns through a push-helper rather than a `lua_push*` call, so the return value does not appear in the decompiled body.

#### `CellFactory_GetAllGases`  
`0x007ace80` &middot; 855 instructions &middot; enforces >= 0 args &middot; returns via a helper

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `include_statics` | `bool` | `lua_toboolean` | documented `true` |
| 2 | `include_particle_fx_materials` | `bool` | `lua_toboolean` | documented `false` |

Returns `{string}`



> The arity guard is **vacuous** (`lua_gettop` is never negative), so even the 2 documented parameter(s) are not actually enforced.

> Returns through a push-helper rather than a `lua_push*` call, so the return value does not appear in the decompiled body.

#### `CellFactory_GetAllLiquids`  
`0x007ac7c0` &middot; 855 instructions &middot; enforces >= 0 args &middot; returns via a helper

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `include_statics` | `bool` | `lua_toboolean` | documented `true` |
| 2 | `include_particle_fx_materials` | `bool` | `lua_toboolean` | documented `false` |

Returns `{string}`



> The arity guard is **vacuous** (`lua_gettop` is never negative), so even the 2 documented parameter(s) are not actually enforced.

> Returns through a push-helper rather than a `lua_push*` call, so the return value does not appear in the decompiled body.

#### `CellFactory_GetAllSands`  
`0x007acb20` &middot; 855 instructions &middot; enforces >= 0 args &middot; returns via a helper

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `include_statics` | `bool` | `lua_toboolean` | documented `true` |
| 2 | `include_particle_fx_materials` | `bool` | `lua_toboolean` | documented `false` |

Returns `{string}`



> The arity guard is **vacuous** (`lua_gettop` is never negative), so even the 2 documented parameter(s) are not actually enforced.

> Returns through a push-helper rather than a `lua_push*` call, so the return value does not appear in the decompiled body.

#### `CellFactory_GetAllSolids`  
`0x007ad540` &middot; 855 instructions &middot; enforces >= 0 args &middot; returns via a helper

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `include_statics` | `bool` | `lua_toboolean` | documented `true` |
| 2 | `include_particle_fx_materials` | `bool` | `lua_toboolean` | documented `false` |

Returns `{string}`



> The arity guard is **vacuous** (`lua_gettop` is never negative), so even the 2 documented parameter(s) are not actually enforced.

> Returns through a push-helper rather than a `lua_push*` call, so the return value does not appear in the decompiled body.

#### `CellFactory_GetName`  
`0x007abe40` &middot; 799 instructions &middot; enforces >= 1 arg &middot; returns 1

> Converts a numeric material id to the material's strings id.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `material_id` | `int` | `lua_tointeger` | - |

Returns `string`



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `CellFactory_GetTags`  
`0x007ad8a0` &middot; 1043 instructions &middot; enforces >= 1 arg &middot; returns via a helper

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `material_id` | `int` | `lua_tointeger` | - |

Returns `{string}`

Runtime messages:
- `couldn't find material with id:`



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> Returns through a push-helper rather than a `lua_push*` call, so the return value does not appear in the decompiled body.

#### `CellFactory_GetType`  
`0x007ac160` &middot; 839 instructions &middot; enforces >= 1 arg &middot; returns 1

> Returns the id of a material.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `material_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `int`



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `CellFactory_GetUIName`  
`0x007ac4b0` &middot; 772 instructions &middot; enforces >= 1 arg &middot; returns 1

> Returns the displayed name of a material, or an empty string if 'material_id' is not valid. Might return a text key.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `material_id` | `int` | `lua_tointeger` | - |

Returns `string`

reads `0x00fe3c84`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `CellFactory_HasTag`  
`0x007adcc0` &middot; 1165 instructions &middot; enforces >= 2 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `material_id` | `int` | `lua_tointeger` | - |
| 2 | `tag` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `{bool}`

Runtime messages:
- `couldn't find material with id:`



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ConvertMaterialEverywhere`  
`0x007e59d0` &middot; 893 instructions &middot; enforces >= 2 args &middot; returns nothing

> Converts 'material_from' to 'material_to' everwhere in the game world, replaces 'material_from_type' to 'material_to_type' in the material (CellData) global table, and marks 'material_from' as a "Transformed" material. Every call will add a new entry to WorldStateComponent which serializes these changes, so please call sparingly. The material conversion will be spread over multiple frames. 'material_from' will still retain the original name id and wang color. Use CellFactory_GetType() to convert a material name to material type.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `material_from_type` | `int` | `lua_tointeger` | - |
| 2 | `material_to_type` | `int` | `lua_tointeger` | - |

Runtime messages:
- `"air" is not a supported material`



#### `ConvertMaterialOnAreaInstantly`  
`0x007e5d50` &middot; 824 instructions &middot; enforces >= 8 args &middot; returns nothing

> Converts cells of 'material_from_type' to 'material_to_type' in the given area. If 'box2d_trim' is true, will attempt to trim the created cells where they might otherwise cause physics glitching. 'update_edge_graphics_dummy' is not yet supported.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `area_x` | `int` | `lua_tointeger` | - |
| 2 | `area_y` | `int` | `lua_tointeger` | - |
| 3 | `area_w` | `int` | `lua_tointeger` | - |
| 4 | `area_h` | `int` | `lua_tointeger` | - |
| 5 | `material_from_type` | `int` | `lua_tointeger` | - |
| 6 | `material_to_type` | `int` | `lua_tointeger` | - |
| 7 | `trim_box2d` | `bool` | `lua_toboolean` | - |
| 8 | `update_edge_graphics_dummy` | `bool` | - | - |



#### `GetMaterialInventoryMainMaterial`  
`0x007a5530` &middot; 1298 instructions &middot; enforces >= 1 arg &middot; returns 1

> Returns the id of the material taking the largest part of the first MaterialInventoryComponent in 'entity_id', or 0 if nothing is found.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `ignore_box2d_materials` | `bool` | `lua_toboolean` | documented `true` |

Returns `material_type:int`

Runtime messages:
- `couldn't find entity with id:`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `LooseChunk`  
`0x007d4790` &middot; 1516 instructions &middot; enforces >= 3 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `world_pos_x` | `number` | `lua_tonumber` | - |
| 2 | `world_pos_y` | `number` | `lua_tonumber` | - |
| 3 | `image_filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 4 | `max_durability` | `int` | `lua_tointeger` | documented `2147483647` |

Runtime messages:
- `image_aabb failed`

reads `0x0105361c` = 1056964608 / 0.5, `0x0120866c`

> A missing/invalid string argument (argument 3) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `RemoveMaterialInventoryMaterial`  
`0x007a4ff0` &middot; 1340 instructions &middot; enforces >= 1 arg &middot; returns nothing

> If material_name is empty, all materials will be removed.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `material_name` | `string` | `lua_isstring`, `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |

Runtime messages:
- `couldn't find entity with id:`

reads `0x00fe3c84`, `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

### World & loading (11)

#### `FindFreePositionForBody`  
`0x007bacb0` &middot; 892 instructions &middot; enforces >= 5 args &middot; returns 2

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `ideal_pos_x` | `number` | `lua_tonumber` | - |
| 2 | `idea_pos_y` | `number` | `lua_tonumber` | - |
| 3 | `velocity_x` | `number` | `lua_tonumber` | - |
| 4 | `velocity_y` | `number` | `lua_tonumber` | - |
| 5 | `body_radius` | `number` | `lua_tonumber` | - |

Returns `x:number,y:number`



#### `LoadBackgroundSprite`  
`0x007b2020` &middot; 950 instructions &middot; enforces >= 3 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `background_file` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `x` | `number` | `lua_tointeger` | - |
| 3 | `y` | `number` | `lua_tointeger` | - |
| 4 | `background_z_index` | `number` | `lua_tonumber` | documented `40.0` |
| 5 | `check_biome_corners` | `bool` | `lua_toboolean` | documented `false` |

reads `0x01053df0` = 1109393408 / 40

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `LoadEntityToStash`  
`0x007a4660` &middot; 1195 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_file` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `stash_entity_id` | `int` | `lua_tointeger` | - |

Runtime messages:
- `couldn't find entity with id:`

writes `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `LoadGameEffectEntityTo`  
`0x007b7a30` &middot; 1199 instructions &middot; enforces >= 2 args &middot; returns 1 (of 2 push sites)

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `game_effect_entity_file` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `effect_entity_id:int`

Runtime messages:
- `couldn't find entity with id:`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `LoadPixelScene`  
`0x007b1990` &middot; 1672 instructions &middot; enforces >= 4 args &middot; returns via a helper

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `materials_filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `colors_filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `x` | `number` | `lua_tointeger` | - |
| 4 | `y` | `number` | `lua_tointeger` | - |
| 5 | `background_file` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 6 | `skip_biome_checks` | `bool` | `lua_toboolean` | documented `false` |
| 7 | `skip_edge_textures` | `bool` | `lua_toboolean` | documented `false` |
| 8 | `color_to_material_table` | `{string-string}` | `lua_type` | documented `{}` |
| 9 | `background_z_index` | `int` | `lua_tointeger` | documented `50` |
| 10 | `load_even_if_duplicate` | `bool` | `lua_toboolean` | documented `false` |

Runtime messages:
- `material_filename is empty, we require a material_filename`



> **The binary enforces 4 arguments, fewer than the 5 the signature marks as required** (10 declared). Trust the enforced number.

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

> Returns through a push-helper rather than a `lua_push*` call, so the return value does not appear in the decompiled body.

#### `LoadRagdoll`  
`0x007e6090` &middot; 1295 instructions &middot; enforces >= 3 args &middot; returns nothing

> Loads a given .txt file as a ragdoll into the game, made of the material given in material.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `pos_x` | `float` | `lua_tonumber` | - |
| 3 | `pos_y` | `float` | `lua_tonumber` | - |
| 4 | `material` | `string` | `lua_isstring`, `lua_tolstring` | documented `"meat"`<br>hardcoded *this function's own usage string* |
| 5 | `scale_x` | `float` | `lua_tonumber` | documented `1` |
| 6 | `impulse_x` | `float` | `lua_tonumber` | documented `0` |
| 7 | `impulse_y` | `float` | `lua_tonumber` | documented `0` |

reads `0x00ff9e80` = meat, `0x01053780` = 1065353216 / 1

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `RemovePixelSceneBackgroundSprite`  
`0x007b23e0` &middot; 872 instructions &middot; enforces >= 3 args &middot; returns 1

> NOTE! Removes the pixel scene sprite if the name and position match. Will return true if manages the find and destroy the background sprite

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `background_file` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `x` | `number` | `lua_tointeger` | - |
| 3 | `y` | `number` | `lua_tointeger` | - |

Returns `bool -`



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `RemovePixelSceneBackgroundSprites`  
`0x007b2750` &middot; 768 instructions &middot; enforces >= 4 args &middot; returns nothing

> NOTE! Removes pixel scene background sprites inside the given area.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x_min` | `number` | `lua_tonumber` | - |
| 2 | `y_min` | `number` | `lua_tonumber` | - |
| 3 | `x_max` | `number` | `lua_tonumber` | - |
| 4 | `y_max` | `number` | `lua_tonumber` | - |



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `SpawnActionItem`  
`0x007a3570` &middot; 827 instructions &middot; enforces >= 3 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `number` | `lua_tonumber` | - |
| 2 | `y` | `number` | `lua_tonumber` | - |
| 3 | `level` | `int` | `lua_tointeger` | - |

reads `0x01205004`

#### `SpawnApparition`  
`0x007a4160` &middot; 1256 instructions &middot; enforces >= 3 args &middot; returns 2

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `number` | `lua_tonumber` | - |
| 2 | `y` | `number` | `lua_tonumber` | - |
| 3 | `level` | `int` | `lua_tointeger` | - |
| 4 | `spawn_now` | `bool` | `lua_toboolean` | documented `false` |

Returns `spawn_state_id:int,entity_id:int`

reads `0x010534fc` = 1036831949 / 0.1, `0x01152548` = 20, `0x0115325c` = 7, `0x01204b98`, `0x01205010`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `SpawnStash`  
`0x007a38b0` &middot; 1110 instructions &middot; enforces >= 4 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `number` | `lua_tonumber` | - |
| 2 | `y` | `number` | `lua_tonumber` | - |
| 3 | `level` | `int` | `lua_tointeger` | - |
| 4 | `action_count` | `int` | `lua_tointeger` | - |

Returns `entity_id:int`

Runtime messages:
- `data/entities/items/stash.xml`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

### Mod file & image overrides (9)

#### `ModImageDoesExist`  
`0x00830300` &middot; 934 instructions &middot; enforces >= 1 arg &middot; returns 1

> Returns true if a file or virtual image exists for the given filename. Unlike most Mod* functions, this one is available everywhere.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `bool`

reads `0x01221bc0`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ModImageGetPixel`  
`0x0082f8b0` &middot; 800 instructions &middot; enforces >= 3 args &middot; returns 1

> Returns the color of a pixel in ABGR format (0xABGR). 'x' and 'y' are zero-based. 
> Use ModImageMakeEditable to create an id that can be used with this function. 
>  While it's possible to edit images after mod init, it's not guaranteed that game systems will see the changes, as the system might already have loaded the image at that point. 
> The function will silently fail nad return 0 if 'id' isn't valid. 
> Unlike most Mod* functions, this one is available everywhere.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `id` | `int` | `lua_tointeger` | - |
| 2 | `x` | `int` | `lua_tointeger` | - |
| 3 | `y` | `int` | `lua_tointeger` | - |

Returns `uint`

reads `0x01207d1c`

#### `ModImageIdFromFilename`  
`0x0082f330` &middot; 1399 instructions &middot; enforces >= 1 arg &middot; returns 3

> Returns an id that can be used with ModImageGetPixel and ModImageSetPixel, and the dimensions of the image. 
>  If a previous successful call to ModImageMakeEditable hasn't been made with the given filename, 0 will be returned as 'id', 'w' and 'h'. 
> Unlike most Mod* functions, this one is available everywhere.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `id:int,w:int,h:int`

Runtime messages:
- `file extension needs to be '.png'`
- `no existing image created using ModImageMakeEditable was found found for filename:`

reads `0x0100e398` = png

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ModImageSetPixel`  
`0x0082fbd0` &middot; 786 instructions &middot; enforces >= 4 args &middot; returns nothing

> Sets the color of a pixel in ABGR format (0xABGR). 'x' and 'y' are zero-based. 
> Use ModImageMakeEditable to create an id that can be used with this function. 
>  The function will silently fail if 'id' isn't valid. 
> Unlike most Mod* functions, this one is available everywhere.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `id` | `int` | `lua_tointeger` | - |
| 2 | `x` | `int` | `lua_tointeger` | - |
| 3 | `y` | `int` | `lua_tointeger` | - |
| 4 | `color` | `uint` | `lua_tointeger` | - |

reads `0x01207d1c`

#### `ModImageWhoSetContent`  
`0x0082fef0` &middot; 1025 instructions &middot; enforces >= 1 arg &middot; returns 1

> Returns the id of the last mod that called ModImageMakeEditable with 'filename', or "". Unlike most Mod* functions, this one is available everywhere.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `string`

reads `0x00fe3c84`, `0x01207e9c`, `0x01207ea0`, `0x01221bc0`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ModLuaFileGetAppends`  
`0x0082d5f0` &middot; 922 instructions &middot; enforces >= 1 arg &middot; returns 1 (of 0 push sites)

> Returns the paths of files that have been appended to 'filename' using ModLuaFileAppend(). Unlike most Mod* functions, this one is available everywhere.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `{string}`

reads `0x01207ed0`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

> Returns through a push-helper rather than a `lua_push*` call, so the return value does not appear in the decompiled body.

#### `ModMaterialFilesGet`  
`0x007e76c0` &middot; 812 instructions &middot; enforces >= 0 args &middot; returns via a helper

> Returns a list of filenames from which materials were loaded.

Returns `{string}`



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> Returns through a push-helper rather than a `lua_push*` call, so the return value does not appear in the decompiled body.

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `ModTextFileGetContent`  
`0x0082de60` &middot; 1251 instructions &middot; enforces >= 1 arg &middot; returns 1 (of 2 push sites)

> Returns the current (modded or not) content of the data file 'filename'. Allows access only to data files and files from enabled mods. "mods/mod/data/file.xml" and "data/file.xml" point to the same file. Unlike most Mod* functions, this one is available everywhere.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `string`

Runtime messages:
- `file cannot be read:`

reads `0x00fe3c84`, `0x01221bc0`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ModTextFileWhoSetContent`  
`0x0082e970` &middot; 1142 instructions &middot; enforces >= 1 arg &middot; returns 1

> Returns the id of the last mod that called ModTextFileSetContent with 'filename', or "". Unlike most Mod* functions, this one is available everywhere.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `string`

reads `0x00fe3c84`, `0x01207e9c`, `0x01207ec0`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

### Mod settings (7)

#### `ModSettingGet`  
`0x007e79f0` &middot; 1014 instructions &middot; enforces >= 1 arg &middot; returns 3

> Returns the value of a mod setting. 'id' should normally be in the format 'mod_name.setting_id'. Cache the returned value in your lua context if possible.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `id` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `bool|number|string|nil`

reads `0x01207ef4`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ModSettingGetAtIndex`  
`0x007e9490` &middot; 1061 instructions &middot; enforces >= 0 args &middot; returns 3 (of 7 push sites)

> 'index' should be 0-based index. Returns nil if 'index' is invalid.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `index` | `int` | `lua_tointeger` | - |

Returns `(name:string, value:bool|number|string|nil, value_next:bool|number|string|nil) | nil`

reads `0x01207ef4`, `0x01207ef8`

> The arity guard is **vacuous** (`lua_gettop` is never negative), so even the 1 documented parameter(s) are not actually enforced.

#### `ModSettingGetCount`  
`0x007e91c0` &middot; 715 instructions &middot; enforces >= 0 args &middot; returns 1

> Returns the number of mod settings defined. Use ModSettingGetAtIndex to enumerate the settings.

Returns `int`

reads `0x01207ef8`

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `ModSettingGetNextValue`  
`0x007e8370` &middot; 1014 instructions &middot; enforces >= 1 arg &middot; returns 3

> Returns the latest value set by the user, which might not be equal to the value that is used in the game (depending on the 'scope' value selected for the setting).

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `id` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `bool|number|string|nil`

reads `0x01207ef4`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ModSettingRemove`  
`0x007e8e30` &middot; 899 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `id` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `was_removed:bool`

writes `0x01207efc` &middot; reads `0x01207ef4`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ModSettingSet`  
`0x007e7df0` &middot; 1395 instructions &middot; enforces >= 2 args &middot; returns nothing

> [Sets the value of a mod setting. 'id' should normally be in the format 'mod_name.setting_id'.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `id` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `value` | `bool|number|string` | `lua_type`, `lua_toboolean`, `lua_tonumber`, `lua_tolstring` | - |

Runtime messages:
- `value type must be boolean, number or string`

writes `0x01207efc` &middot; reads `0x01207ef4`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ModSettingSetNextValue`  
`0x007e8770` &middot; 1716 instructions &middot; enforces >= 3 args &middot; returns nothing

> Sets the latest value set by the user, which might not be equal to the value that is displayed to the game (depending on the 'scope' value selected for the setting).

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `id` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `value` | `bool|number|string` | `lua_type`, `lua_toboolean`, `lua_tonumber`, `lua_tolstring` | - |
| 3 | `is_default` | `bool` | `lua_toboolean` | - |

Runtime messages:
- `value type must be boolean, number or string`

writes `0x01207efc` &middot; reads `0x00000001`, `0x01207ef4`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

### Mod queries (4)

#### `ModDoesFileExist`  
`0x007e7310` &middot; 940 instructions &middot; enforces >= 1 arg &middot; returns 1

> Returns true if the file exists.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `boolean`

reads `0x01221bc0`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `ModGetAPIVersion`  
`0x007e7040` &middot; 711 instructions &middot; enforces >= 0 args &middot; returns 1

Returns `int`

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `ModGetActiveModIDs`  
`0x007e6d50` &middot; 748 instructions &middot; enforces >= 0 args &middot; returns via a helper

> Returns a table filled with the IDs of currently active mods.

Returns `{string}`



> Returns through a push-helper rather than a `lua_push*` call, so the return value does not appear in the decompiled body.

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `ModIsEnabled`  
`0x007e6a10` &middot; 822 instructions &middot; enforces >= 1 arg &middot; returns 1

> Returns true if a mod with the id 'mod_id' is currently active. For example mod_id = "nightmare". 

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `mod_id` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `bool`



> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

### Stats (5)

#### `StatsBiomeGetValue`  
`0x007c5550` &middot; 981 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `key` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `string`



> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `StatsBiomeReset`  
`0x007c5930` &middot; 731 instructions &middot; enforces >= 0 args &middot; returns nothing

reads `0x01208848`

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `StatsGetValue`  
`0x007c4d90` &middot; 981 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `key` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `string|nil`



> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `StatsGlobalGetValue`  
`0x007c5170` &middot; 992 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `key` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `string`



> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `StatsLogPlayerKill`  
`0x007c5c10` &middot; 761 instructions &middot; enforces >= 0 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `killed_entity_id` | `int` | `helper:entity` | documented `0` |

writes `0x012087f4`

> The arity guard is **vacuous** (`lua_gettop` is never negative), so even the 1 documented parameter(s) are not actually enforced.

### Flags & globals (5)

#### `AddFlagPersistent`  
`0x007d54a0` &middot; 840 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `key` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `bool_is_new`

reads `0x012073f4`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GlobalsGetValue`  
`0x007c35b0` &middot; 1083 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `key` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `default_value` | `string` | `lua_isstring`, `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |

reads `0x00fe3c84`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GlobalsSetValue`  
`0x007c3200` &middot; 940 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `key` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `value` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |



> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `HasFlagPersistent`  
`0x007d5b20` &middot; 841 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `key` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `bool`

reads `0x012073f4`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `RemoveFlagPersistent`  
`0x007d57f0` &middot; 808 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `key` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

reads `0x012073f4`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

### Persistent values (16)

#### `BiomeGetValue`  
`0x007a8080` &middot; 1150 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Runtime messages:
- `BiomeGetValue`
- `couldn't find biome with filename:`



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `BiomeMaterialGetValue`  
`0x007a9a60` &middot; 1729 instructions &middot; enforces >= 3 args &middot; returns nothing

> Can be used to read biome config MaterialComponents during initialization. Returns the given value in the first found MaterialComponent with matching material_name. See biome_modifiers.lua for an usage example.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `material_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `field_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `multiple types|nil`

Runtime messages:
- `, ... ) - couldn't find biome with filename:`
- `BiomeMaterialSetValue`

reads `0x00fe4840` = 40, `0x00ff8670` = 8236

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `BiomeMaterialSetValue`  
`0x007a93a0` &middot; 1724 instructions &middot; enforces >= 4 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Runtime messages:
- `, ... ) - couldn't find biome with filename:`
- `BiomeMaterialSetValue`

reads `0x00fe4840` = 40, `0x00ff8670` = 8236

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `BiomeObjectSetValue`  
`0x007a8500` &middot; 1968 instructions &middot; enforces >= 4 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Runtime messages:
- `, ... ) - couldn't find biome with filename:`
- `BiomeObjectSetValue`
- `isn't a MetaObject or doesn't exist`

reads `0x00fe4840` = 40, `0x00ff8670` = 8236

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `BiomeSetValue`  
`0x007a7af0` &middot; 1411 instructions &middot; enforces >= 3 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Runtime messages:
- `, ... ) - couldn't find biome with filename:`
- `BiomeSetValue`

reads `0x00ff8670` = 8236

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `BiomeVegetationSetValue`  
`0x007a8cb0` &middot; 1758 instructions &middot; enforces >= 4 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | (unnamed) | - | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Runtime messages:
- `, ... ) - couldn't find biome with filename:`
- `BiomeVegetationSetValue`

reads `0x00fe4840` = 40, `0x00ff8670` = 8236

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GetValueBool`  
`0x007a2900` &middot; 1025 instructions &middot; enforces >= 2 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | (unnamed) | `key` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | (unnamed) | `default_value` | `lua_toboolean` | - |

reads `0x01208024`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GetValueInteger`  
`0x007a2160` &middot; 1018 instructions &middot; enforces >= 2 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | (unnamed) | `key` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | (unnamed) | `default_value` | `lua_tointeger` | - |

reads `0x01208024`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GetValueNumber`  
`0x007a19a0` &middot; 1052 instructions &middot; enforces >= 2 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | (unnamed) | `key` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | (unnamed) | `default_value` | - | - |

reads `0x01208024`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `MagicNumbersGetValue`  
`0x007c39f0` &middot; 913 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `key` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `string`



> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `SessionNumbersGetValue`  
`0x007c4060` &middot; 913 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `key` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `string`



> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `SessionNumbersSave`  
`0x007c47d0` &middot; 762 instructions &middot; enforces >= 0 args &middot; returns nothing



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `SessionNumbersSetValue`  
`0x007c4400` &middot; 965 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `key` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `value` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |



> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `SetValueBool`  
`0x007a2560` &middot; 923 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | (unnamed) | `key` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | (unnamed) | `value` | `lua_toboolean` | - |

reads `0x01208024`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `SetValueInteger`  
`0x007a1dc0` &middot; 918 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | (unnamed) | `key` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | (unnamed) | `value` | `lua_tointeger` | - |

reads `0x01208024`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `SetValueNumber`  
`0x007a15f0` &middot; 937 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | (unnamed) | `key` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | (unnamed) | `value` | `lua_tonumber` | - |

reads `0x01208024`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

### Streaming (7)

#### `StreamingForceNewVoting`  
`0x007ea0b0` &middot; 134 instructions &middot; **no arity check** &middot; returns nothing

writes `0x01204704`

> **No arity check.** Nothing stops you passing too few arguments; a missing one reads as zero/nil and the call proceeds.

#### `StreamingGetConnectedChannelName`  
`0x007e9ba0` &middot; 215 instructions &middot; **no arity check** &middot; returns 1

writes `0x01204704`

> **No arity check.** Nothing stops you passing too few arguments; a missing one reads as zero/nil and the call proceeds.

#### `StreamingGetIsConnected`  
`0x007e98c0` &middot; 725 instructions &middot; enforces >= 0 args &middot; returns 1

Returns `bool`



> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `StreamingGetRandomViewerName`  
`0x007e9cd0` &middot; 215 instructions &middot; **no arity check** &middot; returns 1

writes `0x01204704`

> **No arity check.** Nothing stops you passing too few arguments; a missing one reads as zero/nil and the call proceeds.

#### `StreamingGetVotingCycleDurationFrames`  
`0x007e9c80` &middot; 80 instructions &middot; **no arity check** &middot; returns 1

reads `0x01053e20` = 1114636288 / 60, `0x01221bc0`

> **No arity check.** Nothing stops you passing too few arguments; a missing one reads as zero/nil and the call proceeds.

#### `StreamingSetCustomPhaseDurations`  
`0x007e9db0` &middot; 755 instructions &middot; enforces >= 2 args &middot; returns nothing

> Sets the duration of the next wait and voting phases. Use -1 for default duration.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `time_between_votes_seconds` | `number` | `lua_tonumber` | - |
| 2 | `time_voting_seconds` | `number` | `lua_tonumber` | - |



#### `StreamingSetVotingEnabled`  
`0x007ea140` &middot; 719 instructions &middot; enforces >= 1 arg &middot; returns nothing

> Turns the voting UI on or off.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `enabled` | `bool` | `lua_toboolean` | - |



### Input (12)

#### `InputGetJoystickAnalogButton`  
`0x007c1740` &middot; 1274 instructions &middot; enforces >= 2 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `joystick_index` | `int` | `lua_tointeger` | - |
| 2 | `analog_button_index` | `int` | - | - |

Returns `float (Debugish function - returns analog 'joystick' button value (0-1). analog_button_index 0 = left trigger, 1 = right trigger Does not depend on state. E.g. player could be in menus. See data/scripts/debug/keycodes.lua for the constants)`

Runtime messages:
- `) is not valid`
- `joystick analog button index (`



#### `InputGetJoystickAnalogStick`  
`0x007c1f80` &middot; 997 instructions &middot; enforces >= 1 arg &middot; returns 2

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `joystick_index` | `int` | `lua_tointeger` | - |
| 2 | `stick_id` | `int` | `lua_tointeger` | documented `0` |

Returns `float x, float y (Debugish function - returns analog stick positions (-1,+1). stick_id 0 = left, 1 = right, Does not depend on state. E.g. player could be in menus. See data/scripts/debug/keycodes.lua for the constants)`



#### `InputGetMousePosOnScreen`  
`0x007c0230` &middot; 126 instructions &middot; **no arity check** &middot; returns 2

reads `0x01221bc0`

> **No arity check.** Nothing stops you passing too few arguments; a missing one reads as zero/nil and the call proceeds.

#### `InputIsJoystickButtonDown`  
`0x007c0dc0` &middot; 1203 instructions &middot; enforces >= 2 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `joystick_index` | `int` | `lua_tointeger` | - |
| 2 | `joystick_button` | `int` | `lua_tointeger` | - |

Returns `bool (Debugish function - returns if 'joystick' button is down. Does not depend on state. E.g. player could be in menus. See data/scripts/debug/keycodes.lua for the constants)`

Runtime messages:
- `) is not valid`
- `joystick button index (`



#### `InputIsJoystickButtonJustDown`  
`0x007c1280` &middot; 1203 instructions &middot; enforces >= 2 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `joystick_index` | `int` | `lua_tointeger` | - |
| 2 | `joystick_button` | `int` | `lua_tointeger` | - |

Returns `bool (Debugish function - returns if 'joystick' button is just down. Does not depend on state. E.g. player could be in menus. See data/scripts/debug/keycodes.lua for the constants)`

Runtime messages:
- `) is not valid`
- `joystick button index (`



#### `InputIsJoystickConnected`  
`0x007c1c40` &middot; 830 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `joystick_index` | `int` | `lua_tointeger` | - |
| 2 | (unnamed) | - | `lua_tointeger` | - |

Returns `bool (Debugish function - returns true if 'joystick' at that index is connected. Does not depend on state. E.g. player could be in menus. See data/scripts/debug/keycodes.lua for the constants)`



#### `InputIsKeyDown`  
`0x007bf8d0` &middot; 786 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `key_code` | `int` | `lua_tointeger` | - |

Returns `bool (Debugish function - returns if a key is down, does not depend on state. E.g. player could be in menus or inputting text. See data/scripts/debug/keycodes.lua for the constants).`

reads `0x01221bc0`

#### `InputIsKeyJustDown`  
`0x007bfbf0` &middot; 786 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `key_code` | `int` | `lua_tointeger` | - |

Returns `bool (Debugish function - returns if a key is down this frame, does not depend on state. E.g. player could be in menus or inputting text. See data/scripts/debug/keycodes.lua for the constants)`

reads `0x01221bc0`

#### `InputIsKeyJustUp`  
`0x007bff10` &middot; 786 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `key_code` | `int` | `lua_tointeger` | - |

Returns `bool (Debugish function - returns if a key is up this frame, does not depend on state. E.g. player could be in menus or inputting text. See data/scripts/debug/keycodes.lua for the constants)`

reads `0x01221bc0`

#### `InputIsMouseButtonDown`  
`0x007c02b0` &middot; 767 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `mouse_button` | `int` | `lua_tointeger` | - |

Returns `bool (Debugish function - returns if mouse button is down. Does not depend on state. E.g. player could be in menus. See data/scripts/debug/keycodes.lua for the constants)`

reads `0x01221bc0`

#### `InputIsMouseButtonJustDown`  
`0x007c05b0` &middot; 767 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `mouse_button` | `int` | `lua_tointeger` | - |

Returns `bool (Debugish function - returns if mouse button is down. Does not depend on state. E.g. player could be in menus. See data/scripts/debug/keycodes.lua for the constants)`

reads `0x01221bc0`

#### `InputIsMouseButtonJustUp`  
`0x007c08b0` &middot; 767 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `mouse_button` | `int` | `lua_tointeger` | - |

Returns `bool (Debugish function - returns if mouse button is down. Does not depend on state. E.g. player could be in menus. See data/scripts/debug/keycodes.lua for the constants)`

reads `0x01221bc0`

### Randomness (7)

#### `ProceduralRandom`  
`0x007ca810` &middot; 1705 instructions &middot; enforces >= 2 args &middot; returns 1 (of 2 push sites)

> This is kinda messy. If given 2 arguments, returns number between 0.0 and 1.0. If given 3 arguments, returns int between 0 and 'a'. If given 4 arguments returns number between 'a' and 'b'.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `number` | `lua_tonumber` | - |
| 2 | `y` | `number` | `lua_tonumber` | - |
| 3 | `a` | `int|number` | `lua_tonumber` | documented `optional` |
| 4 | `b` | `int|number` | `lua_tonumber` | documented `optional` |

Returns `int|number`

reads `0x01205004`, `0x01205024`

#### `ProceduralRandomf`  
`0x007caec0` &middot; 1674 instructions &middot; enforces >= 2 args &middot; returns 1

> This is kinda messy. If given 2 arguments, returns number between 0.0 and 1.0. If given 3 arguments, returns a number between 0 and 'a'. If given 4 arguments returns a number between 'a' and 'b'.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `number` | `lua_tonumber` | - |
| 2 | `y` | `number` | `lua_tonumber` | - |
| 3 | `a` | `number` | `lua_tonumber` | documented `optional` |
| 4 | `b` | `number` | `lua_tonumber` | documented `optional` |

Returns `number`

reads `0x01205004`, `0x01205024`

#### `ProceduralRandomi`  
`0x007cb550` &middot; 1684 instructions &middot; enforces >= 2 args &middot; returns 1

> This is kinda messy. If given 2 arguments, returns 0 or 1. If given 3 arguments, returns an int between 0 and 'a'. If given 4 arguments returns an int between 'a' and 'b'.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `number` | `lua_tonumber` | - |
| 2 | `y` | `number` | `lua_tonumber` | - |
| 3 | `a` | `int` | - | documented `optional` |
| 4 | `b` | `int` | `lua_tointeger` | documented `optional` |

Returns `number`

reads `0x01053968`, `0x01205004`, `0x01205024`

#### `Random`  
`0x007c9610` &middot; 1314 instructions &middot; enforces >= 0 args &middot; returns 1 (of 2 push sites)

> This is kinda messy. If given 0 arguments, returns number between 0.0 and 1.0. If given 1 arguments, returns int between 0 and 'a'. If given 2 arguments returns int between 'a' and 'b'.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `a` | `int` | `lua_tointeger` | documented `optional` |
| 2 | `b` | `int` | `lua_tointeger` | documented `optional` |

Returns `number|int.`

writes `0x01221d18` &middot; reads `0x01053510` = 1859432

> The arity guard is **vacuous** (`lua_gettop` is never negative), so even the 2 documented parameter(s) are not actually enforced.

#### `RandomDistribution`  
`0x007ca080` &middot; 988 instructions &middot; enforces >= 3 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `min` | `int` | `lua_tointeger` | - |
| 2 | `max` | `int` | `lua_tointeger` | - |
| 3 | `mean` | `int` | `lua_tointeger` | - |
| 4 | `sharpness` | `number` | `lua_tonumber` | documented `1` |
| 5 | `baseline` | `number` | `lua_tonumber` | documented `0.005` |

Returns `int`

reads `0x01053780` = 1065353216 / 1, `0x01221d18`

#### `RandomDistributionf`  
`0x007ca460` &middot; 942 instructions &middot; enforces >= 3 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `min` | `number` | `lua_tonumber` | - |
| 2 | `max` | `number` | `lua_tonumber` | - |
| 3 | `mean` | `number` | `lua_tonumber` | - |
| 4 | `sharpness` | `number` | `lua_tonumber` | documented `1` |
| 5 | `baseline` | `number` | `lua_tonumber` | documented `0.005` |

Returns `number`

reads `0x01053780` = 1065353216 / 1

#### `Randomf`  
`0x007c9b40` &middot; 1344 instructions &middot; enforces >= 0 args &middot; returns 1

> This is kinda messy. If given 0 arguments, returns number between 0.0 and 1.0. If given 1 arguments, returns number between 0.0 and 'a'. If given 2 arguments returns number between 'a' and 'b'.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `min` | `number` | `lua_tonumber` | documented `optional` |
| 2 | `max` | `number` | `lua_tonumber` | documented `optional` |

Returns `number`

writes `0x01221d18` &middot; reads `0x01053510` = 1859432

> The arity guard is **vacuous** (`lua_gettop` is never negative), so even the 2 documented parameter(s) are not actually enforced.

### Raytracing (5)

#### `GetSurfaceNormal`  
`0x007bb030` &middot; 941 instructions &middot; enforces >= 4 args &middot; returns 4

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `pos_x` | `number` | `lua_tonumber` | - |
| 2 | `pos_y` | `number` | `lua_tonumber` | - |
| 3 | `ray_length` | `number` | `lua_tonumber` | - |
| 4 | `ray_count` | `int` | `lua_tointeger` | - |

Returns `found_normal:bool,normal_x:number,normal_y:number,approximate_distance_from_surface:number`



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `Raytrace`  
`0x007ba130` &middot; 724 instructions &middot; enforces >= 4 args &middot; returns nothing

> Does a raytrace that stops on any cell it hits.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x1` | `number` | - | - |
| 2 | `y1` | `number` | - | - |
| 3 | `x2` | `number` | - | - |
| 4 | `y2` | `number` | - | - |

Returns `did_hit:bool,hit_x:number,hit_y:number`



#### `RaytracePlatforms`  
`0x007ba9d0` &middot; 724 instructions &middot; enforces >= 4 args &middot; returns nothing

> Does a raytrace that stops on any cell a character can stand on.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x1` | `number` | - | - |
| 2 | `y1` | `number` | - | - |
| 3 | `x2` | `number` | - | - |
| 4 | `y2` | `number` | - | - |

Returns `did_hit:bool,hit_x:number,hit_y:number`



#### `RaytraceSurfaces`  
`0x007ba410` &middot; 724 instructions &middot; enforces >= 4 args &middot; returns nothing

> Does a raytrace that stops on any cell that is not fluid, gas (yes, technically gas is a fluid), or fire.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x1` | `number` | - | - |
| 2 | `y1` | `number` | - | - |
| 3 | `x2` | `number` | - | - |
| 4 | `y2` | `number` | - | - |

Returns `did_hit:bool,hit_x:number,hit_y:number`



#### `RaytraceSurfacesAndLiquiform`  
`0x007ba6f0` &middot; 724 instructions &middot; enforces >= 4 args &middot; returns nothing

> Does a raytrace that stops on any cell that is not gas or fire.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x1` | `number` | - | - |
| 2 | `y1` | `number` | - | - |
| 3 | `x2` | `number` | - | - |
| 4 | `y2` | `number` | - | - |

Returns `did_hit:bool,hit_x:number,hit_y:number`



### Herd & genome (4)

#### `GenomeSetHerdId`  
`0x007bd810` &middot; 864 instructions &middot; enforces >= 2 args &middot; returns nothing

> Deprecated, use GenomeStringToHerdID() and ComponentSetValue2() instead.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `new_herd_id` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GetHerdRelation`  
`0x007bca20` &middot; 799 instructions &middot; enforces >= 2 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `herd_id_a` | `int` | `lua_tointeger` | - |
| 2 | `herd_id_b` | `int` | `lua_tointeger` | - |

Returns `number`



#### `HerdIdToString`  
`0x007bc700` &middot; 786 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `herd_id` | `int` | `lua_tointeger` | - |

Returns `string`



#### `StringToHerdId`  
`0x007bc3c0` &middot; 830 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `herd_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Returns `int`



> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

### Polymorph (4)

#### `PolymorphTableAddEntity`  
`0x007b8430` &middot; 897 instructions &middot; enforces >= 2 args &middot; returns nothing

> Adds the entity to the polymorph random table

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_xml` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `is_rare` | `bool` | `lua_toboolean` | documented `false` |
| 3 | `add_only_one_copy` | `bool` | `lua_toboolean` | documented `true` |

reads `0x012094e0`

> **The binary enforces 2 arguments, more than the 1 the signature marks as required** (3 declared). Trust the enforced number.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `PolymorphTableGet`  
`0x007b8b50` &middot; 754 instructions &middot; enforces >= 0 args &middot; returns via a helper

> Returns a list of all the entities in the polymorph random table

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `rare_table` | `bool` | `lua_toboolean` | documented `false` |

Returns `{string}`

reads `0x012094e0`

> The arity guard is **vacuous** (`lua_gettop` is never negative), so even the 1 documented parameter(s) are not actually enforced.

> Returns through a push-helper rather than a `lua_push*` call, so the return value does not appear in the decompiled body.

#### `PolymorphTableRemoveEntity`  
`0x007b87c0` &middot; 902 instructions &middot; enforces >= 1 arg &middot; returns nothing

> Removes the entity from the polymorph random table

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_xml` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `from_common_table` | `bool` | `lua_toboolean` | documented `true` |
| 3 | `from_rare_table` | `bool` | `lua_toboolean` | documented `true` |

reads `0x012094e0`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `PolymorphTableSet`  
`0x007b8e50` &middot; 906 instructions &middot; enforces >= 1 arg &middot; returns via a helper

> Set a list of all entities sas the polymorph random table

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | (unnamed) | `{table_of_xml_entities}` | - | - |
| 2 | `rare_table` | `bool` | `lua_toboolean` | documented `false` |

reads `0x012094e0`

> Returns through a push-helper rather than a `lua_push*` call, so the return value does not appear in the decompiled body.

### Debug (6)

#### `DEBUG_GetMouseWorld`  
`0x007beb00` &middot; 878 instructions &middot; enforces >= 0 args &middot; returns 2

Returns `x:number,y:number`

reads `0x01221bc0`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `DEBUG_MARK`  
`0x007bee70` &middot; 1149 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `number` | `lua_tonumber` | - |
| 2 | `y` | `number` | `lua_tonumber` | - |
| 3 | `message` | `string` | `lua_isstring`, `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |
| 4 | `color_r` | `number` | `lua_tonumber` | documented `1` |
| 5 | `color_g` | `number` | `lua_tonumber` | documented `0` |
| 6 | `color_b` | `number` | `lua_tonumber` | documented `0` |

reads `0x01053780` = 1065353216 / 1

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 3) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `DebugBiomeMapGetFilename`  
`0x007e4a80` &middot; 997 instructions &middot; enforces >= 0 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `number` | `lua_tonumber` | documented `camera_x` |
| 2 | `y` | `number` | `lua_tonumber` | documented `camera_y` |

Returns `string`

reads `0x0105361c` = 1056964608 / 0.5

> The arity guard is **vacuous** (`lua_gettop` is never negative), so even the 2 documented parameter(s) are not actually enforced.

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `DebugEnableTrailerMode`  
`0x007e44e0` &middot; 683 instructions &middot; enforces >= 0 args &middot; returns nothing

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `DebugGetIsDevBuild`  
`0x007e4210` &middot; 711 instructions &middot; enforces >= 0 args &middot; returns 1

Returns `bool`

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `Debug_SaveTestPlayer`  
`0x007e4a60` &middot; 17 instructions &middot; **no arity check** &middot; returns nothing



> **No arity check.** Nothing stops you passing too few arguments; a missing one reads as zero/nil and the call proceeds.

### Everything else (30)

#### `AutosaveDisable`  
`0x007c4ad0` &middot; 692 instructions &middot; enforces >= 0 args &middot; returns nothing

writes `0x011532f5` = 257

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `BiomeMapConvertPixelFromUintToInt`  
`0x007c7f10` &middot; 752 instructions &middot; enforces >= 1 arg &middot; returns 1

> Swaps red and blue channels of 'color'. This can be used make sense of the BiomeMapGetPixel() return values. E.g. if( BiomeMapGetPixel( x, y ) == BiomeMapConvertPixelFromUintToInt( 0xFF36D517 ) ) then print('hills') end 

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `color` | `int` | `lua_tointeger` | - |

Returns `int`

#### `BiomeMapGetName`  
`0x007c8f50` &middot; 943 instructions &middot; enforces >= 0 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `number` | `lua_tonumber` | documented `camera_x` |
| 2 | `y` | `number` | `lua_tonumber` | documented `camera_y` |

Returns `name`

reads `0x0105361c` = 1056964608 / 0.5

> The arity guard is **vacuous** (`lua_gettop` is never negative), so even the 2 documented parameter(s) are not actually enforced.

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `BiomeMapGetPixel`  
`0x007c7bf0` &middot; 800 instructions &middot; enforces >= 2 args &middot; returns 1

> This is available if BIOME_MAP in magic_numbers.xml points to a lua file, in the context of that file.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `int` | `lua_tointeger` | - |
| 2 | `y` | `int` | `lua_tointeger` | - |

Returns `color:int`

reads `0x01204720`

#### `BiomeMapGetSize`  
`0x007c75d0` &middot; 760 instructions &middot; enforces >= 0 args &middot; returns 2

> if BIOME_MAP in magic_numbers.xml points to a lua file returns that context, if not will return the biome_map size

Returns `width:int,height:int`

writes `0x01204720`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `BiomeMapGetVerticalPositionInsideBiome`  
`0x007c8c10` &middot; 827 instructions &middot; enforces >= 2 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `number` | `lua_tonumber` | - |
| 2 | `y` | `number` | `lua_tonumber` | - |

Returns `number`



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `BiomeMapLoad`  
`0x007a75c0` &middot; 1327 instructions &middot; enforces >= 1 arg &middot; returns nothing

> Deprecated. Might trigger various bugs. Use BiomeMapLoad_KeepPlayer() instead.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `filename` | `string` | - | -<br>hardcoded *this function's own usage string* |

Runtime messages:
- `BiomeMapLoad - this function will be deprecated in the future! Using it probably causes all kinds of bugs already.`
- `data/biome/_biomes_all.xml`
- `data/biome/_pixel_scenes.xml`

writes `0x01207f30`, `0x01207f34` &middot; reads `0x0000000f`, `0x01152708` = 1129447424 / 210, `0x011532ec` = -1029046272 / -85, `0x01204b98`, `0x01204bc0`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `BiomeMapLoadImage`  
`0x007c8200` &middot; 1232 instructions &middot; enforces >= 3 args &middot; returns nothing

> This is available if BIOME_MAP in magic_numbers.xml points to a lua file, in the context of that file.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `int` | `lua_tointeger` | - |
| 2 | `y` | `int` | `lua_tointeger` | - |
| 3 | `image_filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Runtime messages:
- `couldn't load image file:`

reads `0x01204720`

> A missing/invalid string argument (argument 3) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `BiomeMapLoadImageCropped`  
`0x007c86d0` &middot; 1334 instructions &middot; enforces >= 7 args &middot; returns nothing

> This is available if BIOME_MAP in magic_numbers.xml points to a lua file, in the context of that file.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `int` | `lua_tointeger` | - |
| 2 | `y` | `int` | `lua_tointeger` | - |
| 3 | `image_filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 4 | `image_x` | `int` | `lua_tointeger` | - |
| 5 | `image_y` | `int` | `lua_tointeger` | - |
| 6 | `image_w` | `int` | `lua_tointeger` | - |
| 7 | `image_h` | `int` | `lua_tointeger` | - |

Runtime messages:
- `couldn't load image file:`

reads `0x01204720`

> A missing/invalid string argument (argument 3) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `BiomeMapLoad_KeepPlayer`  
`0x007a71d0` &middot; 1000 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `filename` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `pixel_scenes` | `string` | `lua_isstring`, `lua_tolstring` | documented `"data/biome/_pixel_scenes.xml"`<br>hardcoded *this function's own usage string* |

reads `0x01154a20`

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `BiomeMapSetPixel`  
`0x007c78d0` &middot; 793 instructions &middot; enforces >= 3 args &middot; returns nothing

> This is available if BIOME_MAP in magic_numbers.xml points to a lua file, in the context of that file.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `int` | `lua_tointeger` | - |
| 2 | `y` | `int` | `lua_tointeger` | - |
| 3 | `color_int` | `int` | `lua_tointeger` | - |

reads `0x01204720`

#### `BiomeMapSetSize`  
`0x007c72b0` &middot; 785 instructions &middot; enforces >= 2 args &middot; returns nothing

> This is available if BIOME_MAP in magic_numbers.xml points to a lua file, in the context of that file.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `width` | `int` | `lua_tointeger` | - |
| 2 | `height` | `int` | `lua_tointeger` | - |

reads `0x01204720`

#### `ConvertEverythingToGold`  
`0x007e53f0` &middot; 1503 instructions &middot; enforces >= 0 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `material_dynamic` | `string` | `lua_isstring`, `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |
| 2 | `material_static` | `string` | `lua_isstring`, `lua_tolstring` | documented `""`<br>hardcoded *this function's own usage string* |

Runtime messages:
- `couldn't find material:`

reads `0x01205010`

> The arity guard is **vacuous** (`lua_gettop` is never negative), so even the 2 documented parameter(s) are not actually enforced.

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `CreateItemActionEntity`  
`0x007c5f10` &middot; 983 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `action_id` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 2 | `x` | `number` | `lua_tonumber` | documented `0` |
| 3 | `y` | `number` | `lua_tonumber` | documented `0` |

Returns `entity_id:int`



> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `DoesWorldExistAt`  
`0x007bc070` &middot; 846 instructions &middot; enforces >= 4 args &middot; returns 1

> [Returns true if the area inside the bounding box defined by the parameters has been streamed in and no pixel scenes are loading in the area.

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `min_x` | `int` | `lua_tointeger` | - |
| 2 | `min_y` | `int` | `lua_tointeger` | - |
| 3 | `max_x` | `int` | `lua_tointeger` | - |
| 4 | `max_y` | `int` | `lua_tointeger` | - |

Returns `bool`



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `EntitiesGetMaxID`  
`0x00797b10` &middot; 778 instructions &middot; enforces >= 0 args &middot; returns 1

> Returns the max entity ID currently in use. Entity IDs are increased linearly. 

Returns `entity_max_id:number`

reads `0x01054630`, `0x01204b98`

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GetDailyPracticeRunSeed`  
`0x007e65a0` &middot; 1122 instructions &middot; enforces >= 0 args &middot; returns 1

Returns `int`

writes `0x01205030`

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `GetGameEffectLoadTo`  
`0x007b7ee0` &middot; 1357 instructions &middot; enforces >= 3 args &middot; returns 2 (of 4 push sites)

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |
| 2 | `game_effect_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `always_load_new` | `bool` | `lua_toboolean` | - |

Returns `effect_component_id:int,effect_entity_id:int`

Runtime messages:
- `couldn't find entity with id:`

writes `0x01152ff0` = 1 &middot; reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `GetParallelWorldPosition`  
`0x007a6800` &middot; 913 instructions &middot; enforces >= 2 args &middot; returns 2

> x = 0 normal world, -1 is first west world, +1 is first east world, if y < 0 it is sky, if y > 0 it is hell 

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `world_pos_x` | `number` | `lua_tonumber` | - |
| 2 | `world_pos_y` | `number` | `lua_tonumber` | - |

Returns `x, y`



> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `GetRandomAction`  
`0x007c6680` &middot; 886 instructions &middot; enforces >= 3 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `number` | `lua_tonumber` | - |
| 2 | `y` | `number` | `lua_tonumber` | - |
| 3 | `max_level` | `number` | `lua_tointeger` | - |
| 4 | `i` | `int` | `lua_tointeger` | documented `0` |

Returns `string`



#### `GetRandomActionWithType`  
`0x007c62f0` &middot; 911 instructions &middot; enforces >= 4 args &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `number` | `lua_tonumber` | - |
| 2 | `y` | `number` | `lua_tonumber` | - |
| 3 | `max_level` | `int` | `lua_tointeger` | - |
| 4 | `type` | `int` | `lua_tointeger` | - |
| 5 | `i` | `int` | `lua_tointeger` | documented `0` |

Returns `string`



#### `GetUpdatedEntityID`  
`0x007a1050` &middot; 716 instructions &middot; enforces >= 0 args &middot; returns 1

Returns `entity_id:int`

reads `0x01204ff0`

> Takes no arguments; the arity guard is `lua_gettop() < 0` and never fires.

#### `IsInvisible`  
`0x007c2670` &middot; 788 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |

Returns `bool`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `IsPlayer`  
`0x007c2370` &middot; 763 instructions &middot; enforces >= 1 arg &middot; returns 1

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `entity_id` | `int` | `lua_tointeger` | - |

Returns `bool`

reads `0x01204b98`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `RegisterSpawnFunction`  
`0x007a3100` &middot; 1126 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `color` | `int` | `lua_tointeger` | - |
| 2 | `function_name` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |

Runtime messages:
- `couldn't find BiomeSpawnScript for us... no color registered`

reads `0x01224ad4`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `SetPlayerSpawnLocation`  
`0x007b91e0` &middot; 753 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `number` | `lua_tonumber` | - |
| 2 | `y` | `number` | `lua_tonumber` | - |

reads `0x01205010`

> Goes through the entity-manager singleton `DAT_01204b98` (lazy-built, then a **linear** id scan), so bulk loops over entities are O(n) per lookup.

#### `SetRandomSeed`  
`0x007c9300` &middot; 770 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `x` | `number` | `lua_tonumber` | - |
| 2 | `y` | `number` | `lua_tonumber` | - |

reads `0x01205004`, `0x01205024`, `0x01221d18`

#### `SetTimeOut`  
`0x007a2d10` &middot; 993 instructions &middot; enforces >= 2 args &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `time_to_execute` | `number` | `lua_tonumber` | - |
| 2 | `file_to_execute` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |
| 3 | `function_to_call` | `string` | `lua_isstring`, `lua_tolstring` | documented `nil`<br>hardcoded *this function's own usage string* |

reads `0x00fe3c84`

> A missing/invalid string argument (argument 2) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

#### `SetWorldSeed`  
`0x007c3d90` &middot; 713 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `new_seed` | `int` | `lua_tointeger` | - |

writes `0x01205004`, `0x01206faa`

#### `UnlockItem`  
`0x007b94e0` &middot; 797 instructions &middot; enforces >= 1 arg &middot; returns nothing

| # | parameter | declared | read as | if absent |
|---|-----------|----------|---------|-----------|
| 1 | `action_id` | `string` | `lua_isstring`, `lua_tolstring` | -<br>hardcoded *this function's own usage string* |



> A missing/invalid string argument (argument 1) is replaced by **this function's own signature text**, so a malformed call silently yields the usage string as its value.

<!-- END GENERATED reference -->
