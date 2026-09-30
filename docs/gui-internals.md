# Noita GUI internals

The data structures behind the `Gui*` API. If you want to know **what the GUI does**, read
[gui-cookbook.md](gui-cookbook.md) and [gui-options.md](gui-options.md) first — those are
behaviour and formulas, and this page is the layout of the state underneath them.

This page is for when you need the offsets: reimplementing the GUI against the original's
structures, debugging a crash, or writing a Lua mod that has to reason about shared state.

Understanding it explains most of the ways GUI mods interfere with each other: a persistent
per-widget state map keyed by a hashed id, a layout stack that is never unwound automatically,
and a set of frame-scoped fields that some functions reset and others do not.

- [gui-cookbook.md](gui-cookbook.md) — the behaviour and the coordinate maths, no offsets.
- [gui-defaults.md](gui-defaults.md) — the hardcoded sprites, fonts, sounds, defaults and constants.
- [gui-options.md](gui-options.md) — what each `GUI_OPTION` value does.
- [gui-layout.md](gui-layout.md) — the layout engine and container functions in detail.
- [gui-bugs.md](gui-bugs.md) — the bugs and sharp edges, with severity.

Addresses are for one Steam build of the 32-bit `noita.exe` and will move on update. Function
names are stable; treat every address as a pointer into one build.

## Two objects, not one

`GuiCreate()` allocates a **144-byte** object tagged `"LuaImGui"` and hands it to Lua as
lightuserdata. It has no `__gc`, so a `Gui` that is never `GuiDestroy`ed leaks.

Almost nothing lives in it. At offset `+0x88` is a pointer to a second, much larger object
holding all the real state, allocated in the same call — **712 bytes** (`operator_new(0x2c8)`),
created eagerly rather than on the first frame.

Every other `Gui*` function takes the lightuserdata, runs it through a validator, and then
dereferences `+0x88`. A `Gui` whose `+0x88` is null — which `GuiCreate` will hand you if the
state allocation fails — is a null-pointer dereference waiting to happen, not an error.

### The handle validator

`FUN_0078c030` is called by 40 of the 41 `Gui*` functions (`GuiCreate` is the exception). It:

1. compares the handle against a one-entry cache in a process global,
2. otherwise looks the handle up in a global intrusive tree keyed by object address, and
   rejects it unless the node's sentinel matches the tree's root sentinel,
3. caches the result and returns either the handle or `0`.

On `0` the calling function **does nothing at all** — no log line, no error. Using a handle
after destroying it, or passing anything that is not a `gui` handle, is therefore
indistinguishable from a widget that declined to draw.

`GuiDestroy` is a single virtual call on the object; it does not clear that one-entry cache, so
a destroyed handle can still be returned by the cache path until some other handle is
validated.

### The 144-byte object

| offset | size | contents |
|--------|------|----------|
| `+0x00` | 4 | vtable pointer (`ImGuiContext`) |
| `+0x04` | 1 | a flag set from the `GuiCreate` argument |
| `+0x08` | 4 | frame option set, **low 32 bits** |
| `+0x0c` | 4 | frame option set, **high 32 bits** |
| `+0x10` | 4 | pending (next-widget) option set, low 32 bits |
| `+0x14` | 4 | pending option set, high 32 bits |
| `+0x18` | 4 | **never written by the frame reset** |
| `+0x1c` | 16 | next-widget colour: r, g, b, a as floats |
| `+0x2c` | 4 | frame z (float) |
| `+0x30` | 4 | pending z (float) |
| `+0x34` | 1 | "a pending z is set" flag |
| `+0x38` | 80 | the previous-widget record (20 dwords) |
| `+0x88` | 4 | pointer to the 712-byte state object |

Options are a **64-bit set stored as two 32-bit halves**, in two places: `+0x08`/`+0x0c` is the
frame-wide set used by `GuiOptionsAdd`/`Remove`/`Clear`, and `+0x10`/`+0x14` is the pending set
used by `GuiOptionsAddForNextWidget`. A widget is handed
`(pending_low | frame_low, pending_high | frame_high)` as two separate arguments, which is why a
single option value can never affect both halves. See
[enums.md](enums.md#there-are-two-option-fields-and-only-one-is-cleared) for the per-function
field table and what `GuiOptionsClear` actually clears.

### The 712-byte state object

Everything frame-scoped and everything that accumulates lives here. The fields that matter
for writing mods:

| offset | contents |
|--------|----------|
| `+0x28` | font map: string -> measured text record |
| `+0x64` | red-black map: widget id -> persistent widget state |
| `+0xb8`, `+0x160` | initialised to `-1` (frame stamps) |
| `+0x1a0`, `+0x1a4`, `+0x1a8` | layout stack: begin, end, capacity |
| `+0x1ac`, `+0x1b0`, `+0x1b4` | child bounding-box accumulators (2 stacks, 40-byte records) |
| `+0x1b8`, `+0x1bc`, `+0x1c0` | scroll-container frames (40-byte records) |
| `+0x1d0`, `+0x1d4` | **id stack**: begin, end. `std::vector<uint64>` |
| `+0x1dc` | frame counter, initialised to `0x80000000` |
| `+0x1e4`, `+0x1e8` | mouse x, y |
| `+0x1ec`, `+0x1f0` | previous-frame mouse x, y |
| `+0x1f6` | mouse-down flag |
| `+0x1f7` | mouse-moved flag |
| `+0x1fd` | "a widget already claimed the mouse this frame" |
| `+0x21c` | frame stamp for the tooltip, initialised to `0x80000000` |
| `+0x220` | 1.0f |
| `+0x224` | clip/transform records |
| `+0x230`, `+0x234` | a `std::vector<int>` of layer ids |
| `+0x254` | default text colour |
| `+0x2b0` | **UI scale**, 1.0f |
| `+0x2b4` | second font size, 1.0f |

The state object is constructed by `FUN_00817db0`, which first calls a base constructor
`FUN_0065c7b0`. That base is a `GameMouseListener` subclass, and it is worth knowing about for
two reasons:

- **It does not close the layout gaps.** It writes only `+0x00`–`+0x34` (a vtable, two flags,
  a back-pointer, and a `std::string` at `+0x10`). The three ranges
  `+0xd8`–`+0x137`, `+0x170`–`+0x17f` and `+0x184`–`+0x197` are written by **no** constructor in
  the chain, so they start as whatever the allocator returned. If you are reimplementing this,
  those are your three uninitialised holes.
- **It registers the state object as a game mouse listener.** The base constructor adds `this`
  to the global `GameMouse` listener list, which is where the mouse fields at `+0x1e4`–`+0x1f0`
  come from: the engine dispatches input to registered listeners, the GUI does not poll.

A third constructor, `FUN_0081d580`, initialises the sub-object at `+0x23c`: a `std::string`
followed by four RGBA values (the built-in palette) and three scalars. The clip/transform
records at `+0x224` are set by `FUN_008a9f10` with a rectangle of `(1, 0, 1, 1, 1)`.

## Frame lifecycle

`GuiStartFrame` does three things, and one of them is not what you would guess:

1. Builds a 0x88-byte descriptor on the stack and bulk-copies it into `gui+0x08`. This is the
   per-frame reset — see the table below.
2. Calls the real `NewFrame`, which recomputes the UI scale, updates the frame counter, reads
   the mouse, clears the "a widget claimed the mouse" flag, and lazily allocates a
   process-global 416-byte container.
3. Separately builds a zeroed 40-byte widget record and copies it into the **process-global**
   block at `0x01154b98`–`0x01154bcc`. That block is the "previous widget" descriptor, and it
   is global rather than per-`gui`.

### What a frame does and does not reset

| state | reset by `GuiStartFrame`? |
|-------|---------------------------|
| `gui+0x08` frame options, low half | yes, to 0 |
| `gui+0x0c` frame options, high half | yes, to 0 |
| `gui+0x10` / `+0x14` pending options | yes — to 1 and 0, i.e. *option bit 0 on* |
| `gui+0x1c`–`+0x28` colour | fully: white with alpha 1.0 |
| `gui+0x2c` frame z | yes, to 0 |
| `gui+0x30` / `+0x34` pending z and its flag | yes |
| `gui+0x38`–`+0x84` previous-widget record | yes, fully zeroed |
| **layout stack** `state+0x1a0`/`+0x1a4` | **no** |
| **id stack** `state+0x1d0`/`+0x1d4` | **no** |
| **layer stack** `state+0x230`/`+0x234` | **no** |
| **scroll stacks** `state+0x1ac`/`+0x1b8` | **no** |
| persistent widget-state map | **no** — it is meant to persist |

This is the single most important thing to know about the GUI's error behaviour: **an
unbalanced `GuiLayoutBegin*` or `GuiIdPush` survives into the next frame**, and into the next
`GuiStartFrame` on the same object. Nothing warns you. See [gui-bugs.md](gui-bugs.md).

`GuiDestroy` does not unwind them either.

Note also that `GuiStartFrame` performs step 3 **outside** its handle check. Calling it with an
invalid handle skips the real frame reset but still wipes the process-global widget
descriptor.

## The id system

This is the root of mod incompatibility, so it is worth stating exactly.

The id stack is a `std::vector<uint64>` at `state+0x1d0`/`+0x1d4` with a hard cap of **1024**
entries. The hash is **FNV-1a-64** with the standard constants: offset basis
`0xcbf29ce484222325`, prime `0x100000001b3`. The hash step hashes the 8 raw bytes of a value,
one byte at a time, as `hash = (byte XOR hash) * prime`.

`GuiIdPush(gui, id)`:

```c
if ((end - begin) / 8 < 1024) {                  /* no else: see below */
    if (begin == end) seed = 0xcbf29ce484222325;  /* empty stack */
    else            seed = *(uint64 *)(end - 8);  /* top of stack */
    push(FNV1a64_over_8_bytes(id, seed));
}
```

A widget's effective id, computed just before it draws, is:

```c
if (begin == end) return id;                      /* verbatim, NOT hashed */
return FNV1a64_over_8_bytes(id, top_of_stack);
```

Three consequences that the API documentation does not mention:

- **With an empty id stack a widget's id is the raw integer you passed.** Push anything, and the
  same literal id becomes a hash chain instead. So the same `id` can be two different widgets
  depending on whether the mod happens to have wrapped it in a `GuiIdPush` that frame.
- **The hash is cumulative over everything pushed**, so what is on the stack *above* nothing
  matters only in that pushing and popping changes the value. A widget's identity depends on
  the whole current stack, not just its own id.
- **Past 1024 pushes the push is silently dropped** — the `if` has no `else`. Every subsequent
  id resolution in that frame is then wrong, with no diagnostic.

Because the hash is over the *bytes* of the integer, ids `1` and `257` also produce different
chains depending on the stack; and because the empty-stack case returns the id unhashed, a
literal small integer id is a globally predictable widget identity across all mods.

### The persistent widget-state map

Each widget's state is kept in a red-black map at `state+0x64`, **keyed by the 64-bit effective
id**. The map belongs to the `Gui` object, not to the process, but it is never pruned: the only
tree operation in the GUI code is get-or-insert, and no erase path exists. A map node holds at
least 0x200 bytes, including:

| entry offset | contents |
|--------------|----------|
| `+0x54` | a flag gating reuse of the entry |
| `+0x58` | frame counter at last commit |
| `+0x5c` | "drew something this pass" |
| `+0x65` | "entry was created this frame" |
| `+0x78`, `+0x7c` | scroll target and its animated value |
| `+0xd8`, `+0xdc` | measured content size |
| `+0xec` | smoothed 0..1 animation phase |
| `+0x208` | cached asset path (`std::string`) |

This is why id collisions are worse than they look: two mods passing the same id at the same
stack depth are not just drawing over each other, they are **reading and writing the same
persistent state** — hover, animation phase, click latch, scroll offset. A mod that varies its
id per frame grows the map by one node per id, permanently, for the lifetime of that `Gui`
object.

## The draw pipeline

The GUI is a **pure producer of draw commands**, and the pipeline around it is fully mapped.

`FUN_0081bbe0` builds a 100-byte draw command and appends it to a **process-global** three-
pointer vector: `begin` at `0x01208550`, `end` at `0x01208554`, `capacity` at `0x01208558`.
The command count is `(end - begin) / 100`. Growth is geometric (1.5x, via `FUN_00932e60` →
`FUN_00948840`) and the abort path is unreachable in practice.

**The consumer is `FUN_008279c0`, and it is the only one.** One caller in the entire binary:
`FUN_006b3b10`, a virtual method. It:

1. hands the whole list to `FUN_0095f260` (the actual render submission), passing the command
   count,
2. walks the list itself in 100-byte strides, reading each command's first dword as a
   render-resource pointer and testing `+0x48` on it before using it,
3. **resets `end = begin` unconditionally at the end** — this is the drain.

Because the walk is guarded by `if (handle != 0)`, a null resource handle produces no draw
call and no error. `FUN_00efdee0` is the static destructor: it walks the list calling
`FUN_0081dba0` per command, frees the buffer and zeroes all three pointers.

Two things follow for anyone reimplementing this outside the game: the GUI is a pure producer
of draw commands, and the list it produces is per-frame scratch that the renderer consumes and
resets. It is device-independent in the sense that matters — nothing in the GUI subsystem
touches Direct3D, and the command format is a flat 100-byte record.

### The colour helper is a false-positive machine

`FUN_0042b6e0` is worth calling out because it produces a **wrong reading** if taken at face
value. The decompiler types it as taking a single float and reports it reading three
uninitialised registers:

```c
*(undefined4 *)(this + 4)  = in_XMM1_Da;    /* looks uninitialised */
*(undefined4 *)(this + 8)  = in_XMM2_Da;    /* looks uninitialised */
*(undefined4 *)(this + 0xc)= in_XMM3_Da;    /* looks uninitialised */
```

It is an ordinary `__thiscall` four-float RGBA constructor — r/g/b in XMM1–XMM3, a on the
stack — and all **174** call sites define all three registers. If you are reading the
decompilation and see `in_XMM*_Da` in a colour helper, check the call sites before believing
it. See [gui-bugs.md](gui-bugs.md).

## The 41 functions and what they actually call

The Lua functions are thin — most of their instruction count is the shared arity-error string
builder. This is the mapping from the Lua name to the engine function that does the work,
which is the useful starting point for reading anything further.

| Lua function | engine function | notes |
|--------------|-----------------|-------|
| `GuiCreate` | `0x0081dbb0` | allocates 144 + 712 bytes, registers the handle |
| `GuiDestroy` | `0x007d9c90` | one virtual call; no unwinding |
| `GuiStartFrame` | `0x0081dd50` → `0x008187d0` | the real NewFrame is the second one |
| `GuiGetScreenDimensions` | `0x007e2da0` | `window_size / ui_scale`; 0,0 on any failure |
| `GuiGetPreviousWidgetInfo` | `0x00563630` + reads `gui+0x38` | returns 11 values from the global record |
| `GuiIdPush` | `0x008274d0` | FNV-1a-64, cap 1024 |
| `GuiIdPop` | `0x007dbf00` | `end -= 8`, unchecked |
| `GuiIdPushString` | `0x00817d50` + `0x008274d0` | hashes the string bytes, then pushes |
| `GuiOptionsAdd` / `Remove` / `Clear` | `0x007da320` / `0x007da630` / `0x007da940` | `+0x0c` only |
| `GuiOptionsAddForNextWidget` | `0x007dac30` | `+0x10` and `+0x14` |
| `GuiZSet` / `GuiZSetForNextWidget` | `0x007db2b0` / `0x007db5b0` | `+0x2c` / `+0x30`+`+0x34` |
| `GuiColorSetForNextWidget` | `0x0042b6e0` | writes `+0x1c`–`+0x28` |
| `GuiText` / `GuiTextCentered` | `0x00821630` | the text widget |
| `GuiButton` | `0x008245d0` | hit-testing and the id-keyed cache |
| `GuiSlider` | `0x00824e70` | |
| `GuiTextInput` | `0x00825cb0` | |
| `GuiImage` | `0x00820bf0` | |
| `GuiImageNinePiece` | `0x008221d0` | |
| `GuiImageButton` | `0x00822f00` | |
| `GuiGetImageDimensions` | `0x0081b8d0` | loads and caches a 232-byte image object |
| `GuiGetTextDimensions` | `0x0081de00` + `0x0081b4e0` | measurement + per-string cache |
| `GuiTooltip` | `0x008291b0` | reads the **global** previous-widget record |
| `GuiLayoutBeginHorizontal` / `Vertical` | `0x0081e480` | **the same function**, direction is an argument |
| `GuiLayoutEnd` | `0x0081e710` | |
| `GuiLayoutBeginLayer` / `EndLayer` | `0x0081e7e0` / `0x0081e8b0` | 16-byte records, see [gui-layout.md](gui-layout.md) |
| `GuiLayoutAddHorizontalSpacing` | `0x007e1e90` | **ignores its amount argument** |
| `GuiLayoutAddVerticalSpacing` | `0x007e21b0` | uses it |
| `GuiBeginAutoBox` | `0x008202d0` | pushes an empty bounding-box accumulator |
| `GuiEndAutoBoxNinePiece` | `0x008205a0` | measures, sizes, mirrors, draws |
| `GuiBeginScrollContainer` | `0x0081f870` | |
| `GuiEndScrollContainer` | `0x00820060` | |
| `GuiAnimateBegin` / `End` | `0x0081eb00` / `0x007dc4e0` | |
| `GuiAnimateAlphaFadeIn` / `ScaleIn` | `0x0081eb60` / `0x0081edf0` | |
| every widget | `0x00563630` … `0x007d97d0` | begin/commit bracket, see below |

### The widget bracket

Every drawing function is wrapped in the same pair:

- `FUN_00563630(w)` initialises an 80-byte widget record on the stack: zero, except word 13
  which is set to `1.0f` (a default scale).
- the widget's own function fills the record in and returns it,
- `FUN_007d97d0(gui, w)` copies the record to `gui+0x38` and **resets the pending option,
  colour and z fields**.

So "the previous widget" is a copy of the last record committed, and the pending per-widget
state is consumed by being reset at commit time. That is why `gui+0x10` is set to `1` at
commit rather than `0`: the reset value is "option bit 0 on", the default alignment.

The commit is also where the 80-byte record picks up its problem: the constructor writes
through byte 75, but the commit copies 80 bytes, so the last dword of every committed record
is stack garbage. See [gui-bugs.md](gui-bugs.md).

### Hit-testing and the "one widget owns the mouse" rule

A widget is hovered when the mouse is inside its rectangle, and the frame has a single flag at
`state+0x1fd` that the first hovering widget sets; later widgets are then not considered
hovered. Clicking additionally requires the mouse-down flag, or a forced option bit. There is
no "pressed then dragged off" case — click is gated on the pointer still being inside.
