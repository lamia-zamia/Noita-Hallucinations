# How Noita's Lua API actually behaves

Noita's modding API has no documentation of its own beyond a one-line usage string per
function, embedded in the binary and used to build error messages. Those strings are accurate
about shape and silent about behaviour. This page is the behaviour, recovered from the
executable, that changes how you should write mod code.

Everything here applies to all 375 API functions unless a page says otherwise.

## Nothing is type-checked

The entire API is built on sixteen C functions from `lua51.dll`. What is missing from that list
matters more than what is in it: there is no `luaL_checkinteger`, no `luaL_checkstring`, no
`luaL_argerror`, no `luaL_error` and no `lua_error` anywhere in the surface.

Arguments are read with the permissive accessors only — `lua_tonumber`, `lua_tointeger`,
`lua_tolstring`, `lua_toboolean`, `lua_topointer` — which coerce and never complain.

| you pass | what happens |
|----------|--------------|
| a string where a number is wanted | coerced; `"5"` works, `"abc"` becomes `0` |
| a number where a string is wanted | coerced, because Lua 5.1 converts numbers to strings |
| `nil` for a number | `0` |
| `nil` or a non-string for a *documented string* argument | an error is **logged** and a hardcoded default string is substituted — see below |
| too few arguments | an error is **logged** and the function **does nothing at all** |

There is no path by which a bad argument raises an error your `pcall` can catch. Error
handling around API calls will not work; you have to check values yourself.

## Too few arguments makes the function do nothing

Most functions begin with the same guard:

```c
if (lua_gettop(L) < 4) {
    /* log:  "GuiText( … ) requires 4 parameters, only 2 given"  */
}
else {
    /* the real work */
}
```

The failing branch has **no `return` in it**. So an under-supplied call is not reported and
then partially executed — it performs no work and leaves **no values on the stack**. A function
documented as `-> entity_id` returns zero values, which in Lua shows up as `nil` at the
assignment, one frame away from the actual mistake.

366 of the 375 functions have this guard. The 9 that do not are listed in the
[reference](api-reference.md), along with the 7 functions whose enforced minimum disagrees with
their own signature.

## A missing string argument comes back as a string — often the signature itself

When a documented string argument is absent, or present but not a string, the function logs

```
<N> param wasn't a string, string was expected
```

and then **returns a hardcoded string that the call site baked in**. For many functions that
hardcoded string is the function's own usage text, which means a malformed call quietly hands
your code back a copy of the documentation:

```lua
-- AddFlagPersistent expects one string. Give it a number.
AddFlagPersistent(5)
--> logs: 1 param wasn't a string, string was expected
--> returns true, having searched the persistent-flag store for the key
--    "AddFlagPersistent( key:string ) -> bool_is_new"
```

Where the default is the empty string the substitution is silent. The
[reference](api-reference.md) marks every affected function under **if absent**.

## An error that repeats is only logged once

All of the above messages end up in one function, which writes `Lua error - <message>` to the
game log and remembers the last message it printed. If the new message is identical to the
previous one it is **suppressed**, printing `Lua error - (skipping logging of recurring lua
errors)` instead.

This bites in the obvious place: an error inside a per-frame callback gets logged exactly
once. If you are grepping the log for a repeating error and find a single hit, the error did
not stop happening — the logger decided not to say so again. Vary the message, or read it out
of the return value, if you need to see it every frame.

## There are no aliases, and no hidden functions

Not one of the 375 names is a jump to another implementation. Where two names look
interchangeable — `EntityGetHerdRelation` and `GetHerdRelation`, `EntityGetComponent` and
`EntityGetFirstComponent` — they are genuinely separate code with separate validation, and
they can diverge between game updates.

The game calls `lua_setfield` from fourteen places, but thirteen of them install functions on
engine-internal objects (the gun system, perks, status effects, the MetaObject tables), not on
the global environment. Nothing is hiding. See
[api-registration.md](api-registration.md) for what each of them is.

## Handles, ids, and other shared state

Three things are process-global rather than per-mod, and are the cause of most cross-mod
interference.

**`gui` handles are validated, and a stale one is silent.** Every `Gui*` call runs the handle
through a validator that returns `0` for a handle that is no longer live. The caller then does
nothing — no log line. Using a `Gui` object after `GuiDestroy` it fails invisibly, so a
destroyed-object bug looks exactly like a widget that stopped drawing.

**Entity lookups are a linear scan.** Resolving an `entity_id` walks a vector of entity
pointers comparing stored ids. Loops over many entities are therefore quadratic, and this is
visible in the disassembly rather than inferred: 41 of the 54 `Entity*` functions reach the
same scan. If you are iterating entities, prefer the bulk forms (`EntityGetInRadius`,
`EntityGetAllComponents`, `EntityGetAllChildren`) over repeated per-id calls.

**The previous-widget block is global, not per-`gui`.** Exactly ten functions write
`0x01154b98`–`0x01154ba8`, and `GuiGetPreviousWidgetInfo` reads it back. Two `Gui` objects, or
two mods, drawing in the same frame clobber each other's results. The ten are
`GuiBeginScrollContainer`, `GuiButton`, `GuiEndAutoBoxNinePiece`, `GuiImage`, `GuiImageButton`,
`GuiImageNinePiece`, `GuiSlider`, `GuiText`, `GuiTextCentered` and `GuiTextInput` — note that
the layout and scope functions are not among them, so a frame that only moves widgets leaves
the previous frame's information in place.

## Return values

The reference reports the number of values each function leaves on the stack, taken from the
`lua_checkstack` reservation the generated code emits immediately before its push run. Where a
function has several return paths the count is shown as *returns N (of M push sites)*; N is
what one call gives you, M is how many push calls exist in the body.

A handful of functions return their result through a helper rather than a `lua_push*` call of
their own, and are marked *returns via a helper*.
