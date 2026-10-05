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
    /* builds "requires 4 parameters, only 2 given" and sends it to the error sink,
       which logs  "Lua error - GuiText( … )"  followed by the indented message  */
}
else {
    /* the real work */
}
```

Note the shape of that log line: the message and the signature are logged as *two* lines,
the second indented, because the signature string is passed to the logger as context rather
than being formatted into the message.

The failing branch has **no `return` in it**. So an under-supplied call is not reported and
then partially executed — it performs no work and leaves **no values on the stack**. A function
documented as `-> entity_id` returns zero values, which in Lua shows up as `nil` at the
assignment, one frame away from the actual mistake.

366 of the 375 functions have this guard. The 9 that do not are listed in the
[reference](api-reference.md), along with the 7 functions whose enforced minimum disagrees with
their own signature.

## A missing string argument becomes an empty string, and says so

When a documented string argument is absent, or present but not a string, the function logs

```
 param <N> wasn't a string, string was expected
```

and the argument **becomes the empty string `""`**. The helper that logs the message returns a
pointer to a zero-length string, and the caller uses that pointer as the argument.

```lua
-- AddFlagPersistent expects one string. Give it a number.
AddFlagPersistent(5)
--> logs:  param 1 wasn't a string, string was expected
--> then searches the persistent-flag store for the key ""
```

Numbers and booleans are looser still: they are read with a plain `lua_tonumber` /
`lua_toboolean`, so a wrong type is coerced silently with no check and no log line at all.
The practical summary is the same either way — **a bad string argument does not raise, it quietly
becomes an empty string**, and the only reliable way to notice is the log.

The [reference](api-reference.md) marks every affected function under **if absent**.

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

Three things are worth knowing about cross-object state.

**`gui` handles are validated, and a stale one is silent.** Every `Gui*` call runs the handle
through a validator that returns `0` for a handle that is not in the registry of live `Gui`
objects. The caller then does nothing — no log line. Using a `Gui` object after `GuiDestroy` it
fails invisibly, so a destroyed-object bug looks exactly like a widget that stopped drawing. The
validator has no vtable probe and no generation counter, so a stale handle is accepted if it
happens to match a one-entry cache of the last valid handle.

**Entity lookups are a linear scan.** Resolving an `entity_id` walks a vector of entity
pointers comparing stored ids. Loops over many entities are therefore quadratic, and this is
visible in the disassembly rather than inferred: 41 of the 54 `Entity*` functions reach
the same scan. If you are iterating entities, prefer the bulk forms (`EntityGetInRadius`,
`EntityGetAllComponents`, `EntityGetAllChildren`) over repeated per-id calls.

**The previous-widget block is per-`gui`, at `gui+0x38`.** Ten functions record a drawn widget
into it - `GuiBeginScrollContainer`, `GuiButton`, `GuiEndAutoBoxNinePiece`, `GuiImage`,
`GuiImageButton`, `GuiImageNinePiece`, `GuiSlider`, `GuiText`, `GuiTextCentered` and
`GuiTextInput` - and `GuiBeginAutoBox` commits a freshly initialised record through the same
path. `GuiGetPreviousWidgetInfo` reads it back. Two mods with their own `Gui` objects therefore do not
interfere. Note that the layout and scope functions are not among the writers, so a frame that
only moves widgets leaves the previous frame's information in place.

**It only ever holds a Lua-drawn widget.** The commit is `FUN_007d97d0(gui, record)` and it has
exactly 11 callers (the ten widget functions and `GuiBeginAutoBox`), every one of them a Lua wrapper in the `0x007d`-`0x007e` band. The game's own
widgets are built by a different set of functions (`0x008245d0`, `0x00823580`, `0x00825cb0`, ...)
which never call it. So passing the game's `gui` to `GuiGetPreviousWidgetInfo` returns the last
widget *you* drew, not a vanilla one - see [ui-modding-2.md](ui-modding-2.md).

## Return values

The reference reports the number of values each function leaves on the stack, taken from the
`lua_checkstack` reservation the generated code emits immediately before its push run. Where a
function has several return paths the count is shown as *returns N (of M push sites)*; N is
what one call gives you, M is how many push calls exist in the body.

A handful of functions return their result through a helper rather than a `lua_push*` call of
their own, and are marked *returns via a helper*.
