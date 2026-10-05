# API notes by family

Behaviour that holds across a whole family rather than a single function, read off the
disassembly rather than the signatures. The per-function evidence is in
[api-reference.md](api-reference.md).

### Entity

54 functions. Resolving an `entity_id` is a **linear scan**: 30 call the lookup `0x0056eba0`
directly and 11 more go through the argument helper `0x0078da80`, which does the same lookup
— 41 of 54 in total. The remaining ones work on a component or on a coordinate, not an id.

`EntityLoad` and `EntityLoadEndGameItem` are the exception: they create a new entity rather
than looking one up, and can fail with `Error! Couldn't create a new entity, out of memory?`.

Passing `0` is meaningful, not an error: `GameShootProjectile` documents
`shooter_entity` "can be 0", and the lookup returns null rather than raising, so the
functions that dereference the result are the ones that need guarding.

### Component

30 functions, and 29 of them resolve their `component_id` through the same lookup,
`0x008a8a00`. That is the component registry and the single choke point for the family, so
a bad component id surfaces as the caller's own message — `ComponentObjectGetValue2` reports
`isn't a MetaObject or doesn't exist` — rather than as a Lua error.

`ComponentObjectGetValue2` and `ComponentObjectSetValue2` are the multi-value entry points;
the `table_of_component_values` form used by `EntityAddComponent2` does not support
multivalue types.

### Game

71 functions and the widest spread of return shapes in the API. The zero-argument
accessors (`GameGetFrameNum`, `GameGetCameraPos`, `GameIsDailyRun`, …) all share the
`lua_gettop() < 0` guard, which never fires; 21 of the API's 52 argument-less functions are
in this family, and calling them with stray arguments is harmless.

`GameGetDateAndTimeLocal` (8 values) and `GameGetDateAndTimeUTC` (6) are the widest calls in the
family, and both are two of only eight functions in the API with neither an arity guard
nor an embedded usage string. The local variant's two extra values are booleans compared
against hardcoded per-year date tables that stop matching in 2040 — see
[undocumented-functions.md](undocumented-functions.md).

`GameSetPostFxTextureParameter` carries the longest embedded documentation string in the
binary; the caution in it about one texture per unique `parameter_name` is a real memory
concern, not boilerplate.

### Gui

41 functions. 40 of them validate the `gui` handle through `0x0078c030`; the exception is
`GuiCreate`, which has no handle to validate. A stale handle is a **silent no-op**.

Exactly **10** functions commit a widget record to `gui+0x38` — the per-`gui` "previous widget"
that `GuiGetPreviousWidgetInfo` returns — and write the shared scratch block
`0x01154b98`–`0x01154ba8` that `GuiTooltip` reads:

    GuiBeginScrollContainer   GuiButton              GuiEndAutoBoxNinePiece
    GuiImage                  GuiImageButton         GuiImageNinePiece
    GuiSlider                 GuiText                GuiTextCentered
    GuiTextInput

That is the set `GuiGetPreviousWidgetInfo` reads. Note what is *not* in it: the layout
functions and the scope functions do not, so a layout-only frame leaves stale information
behind. See [gui-internals.md](gui-internals.md) for the id-hashing and layout internals.

### Physics

31 functions, all with an arity guard. The `*SetTransform` family
(`PhysicsComponentSetTransform`, `PhysicsBodyIDSetTransform`) and `PhysicsAddJoint` enforce
only **3** arguments against 6-7 documented parameters — the remaining values are read
unconditionally and default to zero if you omit them. `PhysicsBodyIDSetTransform` says so
in its own documentation; the other two do not.

Two functions return the full 6-value transform (`PhysicsBodyIDGetTransform`,
`PhysicsComponentGetTransform`) in **Box2D units**, which is why the velocity results need
`PhysicsVecToGameVec` before use.

### Materials & cells

The `CellFactory_*` family is the material enum: `GetType`, `GetName`, `GetTags` and
`GetUIName` each take one argument (a material name or id), `HasTag` takes two, and the five
`GetAll*` dumps take only optional flags (`include_statics`, `include_particle_fx_materials`)
and enumerate the cell material table at runtime. `CellFactory_GetType` is the name-to-id conversion used
throughout the API; there is no numeric literal for a material anywhere in the Lua surface,
which is why so many signatures spell it `material_type:number`.

### World & loading

Pixel-scene, background-sprite, ragdoll and projectile spawning. `LoadPixelScene` enforces
only **4** arguments against 5 required in its signature, and at 1672 instructions it is
one of the larger functions in the API — it parses and composes an entire pixel scene in a
single call.

### Mod file & image overrides

Eight functions (`ModImageDoesExist`, `ModImageGetPixel`, `ModImageIdFromFilename`,
`ModImageSetPixel`, `ModImageWhoSetContent`, `ModLuaFileGetAppends`, `ModTextFileGetContent`,
`ModTextFileWhoSetContent`) carry the caveat *"Unlike most Mod\* functions, this one is
available everywhere."* The file-patching `Mod*` functions exist only during
initialisation; these stay callable afterwards. `ModImageGetPixel` and
`ModImageSetPixel` silently fail and return 0 on an invalid id — no log line.

### Mod settings

Seven functions over one global settings vector at `0x01207ef4` with a count at
`0x01207ef8`. `ModSettingGetAtIndex` has a **vacuous** arity guard (`lua_gettop() < 0`), so
its `index` argument is not enforced; it also has 7 push sites for 3 return values, because
the value and next-value pushes each sit behind a 4-way type switch.

### Mod queries

`ModGetActiveModIDs`, `ModGetAPIVersion`, `ModIsEnabled`, `ModSettingGetCount`. All
zero-argument or trivially guarded. `ModGetActiveModIDs` returns its table through the
push-helper `0x0078d9f0`, so it has no `lua_push*` of its own.

### Stats

Five functions over the run/global/biome stat stores. `StatsGetValue` is the only one whose
signature admits `nil`; the other two getters always return a string, empty if the key is unknown.

### Flags & globals

`AddFlagPersistent` / `HasFlagPersistent` / `RemoveFlagPersistent` and the `Globals*` pair.
`AddFlagPersistent` is the clearest example in the API of the string-fallback trap: called
with a non-string it logs `1 param wasn't a string, string was expected` and then carries on with
the key `""` (the empty string), so it records a flag with an empty name.

`GlobalsGetValue` / `GlobalsSetValue` are **not** a preferences API, despite the name. The store is
a flat string→string map on the *WorldState entity* — the same one `GameGetWorldStateEntity()`
returns — and it is saved into the run's `world_state.xml`. Three consequences worth knowing:

- **It does not exist in the front-end.** Before a run exists there is no WorldState, so reads
  return your default and writes do nothing. Main-menu code cannot use it.
- **It is per-run, and reset by a new game.** It is where the game keeps its own progression
  bookkeeping (`HOLY_MOUNTAIN_VISITS`, `GLOBAL_BOSS_KILL_COUNT`, `visited_biomes`,
  `fungal_shift_iteration`, the perk and essence pickup counters).
- **It holds no UI or screen state at all.** Actual user preferences live in a separate
  `config.xml` structure with no Lua binding whatsoever.

There is no way to enumerate it, keys are a flat unnamespaced namespace, and both functions are
string-only. See [ui-modding-2.md](ui-modding-2.md).

### Persistent values

The `GetValue*` / `SetValue*` / `SessionNumbers*` / `MagicNumbers*` family. The
`SetValueBool`, `SetValueInteger` and `SetValueNumber` entries are **not** general setters:
their embedded message is `called outside LuaComponent context`, and they only work from
inside a LuaComponent's own callbacks, where the component supplies the implicit key.

### Streaming

Seven functions for the Twitch-integration surface, and the least documented corner of the
API. Three (`StreamingGetIsConnected`, `StreamingSetCustomPhaseDurations`,
`StreamingSetVotingEnabled`) carry both a guard and a usage string; the other **four** have
**neither**, putting them among only eight functions in the API that the binary documents not
at all. `StreamingGetConnectedChannelName`, `StreamingGetRandomViewerName` and
`StreamingGetVotingCycleDurationFrames` all return a single string with no way to
distinguish "no stream" from "empty name". All four construct the streaming manager lazily and
dereference it without a null check; `StreamingForceNewVoting` is additionally gated on a flag
with no `else` branch, so it does nothing at all when no stream is connected. Details in
[undocumented-functions.md](undocumented-functions.md).

### Input

Twelve functions wrapping the input state, all marked *"Debugish function … does not depend
on state. E.g. player could be in menus."* in their own documentation. The key codes they
expect come from `data/scripts/debug/keycodes.lua`; the binary does not carry the table, so
the mapping cannot be recovered from the executable alone. `InputGetMousePosOnScreen` is the
one member with neither an arity guard nor a usage string, and it is also the one that does
not return what a modder would expect: **unscaled screen pixels, not GUI coordinates**, with
no null check on the renderer it reads from. See
[undocumented-functions.md](undocumented-functions.md).

### Randomness

`Random`, `Randomf`, `RandomDistribution*` and the `ProceduralRandom*` trio. The
`Procedural*` variants are position-keyed: the same `(x, y)` yields the same value, which is
what makes them usable inside world-generation callbacks that may be re-entered.

### Raytracing

`Raytrace`, `RaytracePlatforms`, `RaytraceSurfaces`, `RaytraceSurfacesAndLiquiform` and
`GetSurfaceNormal`. The four `Raytrace*` functions return the same 3-value shape (`did_hit`,
then the point), and differ only in which cells count as a hit — the documentation strings state
the predicate for each. `GetSurfaceNormal` is different: it takes a position, a ray length and
a ray count, and returns `found_normal`, `normal_x`, `normal_y` and
`approximate_distance_from_surface`.

### Herd & genome

Four functions. `EntityGetHerdRelation` is *not* an alias for `GetHerdRelation`: it is a
separate, larger implementation with its own validation messages
(`entity_a doesn't have an enabled GenomeComponent`), which is the clearest demonstration
that this API has no shared entry points.

### Polymorph

Four functions over the polymorph random tables. `PolymorphTableAddEntity` enforces **2**
arguments where its signature marks only 1 as required, so a one-argument call logs an arity
error and adds nothing.

### Debug

Six functions. Five are ordinary guarded calls; `Debug_SaveTestPlayer` has neither an arity
guard nor a usage string — one of only eight functions in the API the binary does not
document at all — and in a release build it is a **17-instruction stub that does nothing**.
`EntitySave` sits in the same position from the other side: it *has* a usage string but no
guard, and its whole body logs that string to the error log without saving anything. Its own
documentation's note that it "works only in dev builds" is the entire implementation. Both are
covered in [undocumented-functions.md](undocumented-functions.md).

## Reading a per-function entry


