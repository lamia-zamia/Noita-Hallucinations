# GUI bugs and sharp edges

Every item here is from the disassembly of one Steam build. Each is labelled with how solid the
evidence is:

- **verified** — read directly out of the decompilation while writing this page.
- **reported** — found by a systematic sweep of the 207-function GUI call graph; specific, with
  a named function, but not individually re-checked.
- **disproved** — a claim that was published here and has since been checked and found wrong.
  The heading keeps the original wording so you can recognise it if you read an older copy; the
  body says what is actually true.

Addresses are build-specific.

If you are reimplementing the GUI rather than modding it, this page is the list of behaviours
you must either copy or fix deliberately. Several of them are load-bearing: the game itself
depends on the quirks.

## Wrong values

### `GuiLayoutAddHorizontalSpacing` ignores its amount
**verified** — the most consequential one, because it is a silent wrong answer rather than a
crash.

```c
if (1 < iVar1) { lua_tonumber(param_1,2); }                        /* discarded */
iVar1 = *(int *)(state + 0x1a4);
iVar6 = *(int *)(iVar1 + -0xc);
if (*(int *)(iVar1 + -0x10) != iVar6) {
    *(float *)(iVar6 - 0x24) = *(float *)(iVar6 - 0x1c) + *(float *)(iVar6 - 0x24);
}
```

The amount is fetched from Lua and thrown away; the x cursor always advances by the layout's
`margin_x`. `GuiLayoutAddVerticalSpacing` uses its amount correctly, including the documented
margin fallback. **Workaround:** set the layout's `margin_x` to the spacing you want, or use a
zero-size widget.

### Percentage layout positions are truncated
**reported** — the wrappers compute `... * (float)(int)x * 0.01f`, so `50.7` is treated as `50`.

### `mirrorize_over_x_axis` is dead
**verified** — the Lua wrapper calls `lua_toboolean(param_1,5);` and discards the result; no
other read of that argument exists. Only `x_axis` mirrors anything.

### `GuiGetScreenDimensions` returns 0, 0 on every failure
**verified** — wrong argument type, stale handle, or a null `+0x88` all leave the two locals at
their initialised `0.0`, and both zeros are pushed. No error, no log.

```c
local_w = 0.0; local_h = 0.0;
if (lua_type(L,1) == 2) { h = validate(topointer(L,1));
  if (h != 0) { size = window_size();
                 scale = state->ui_scale;                 /* state + 0x2b0 */
                 local_w = size[0] / scale; local_h = size[1] / scale; } }
```

## Memory safety

### Layout and scroll End functions underflow unchecked
**verified** for the layout case, **reported** for the scroll case.

`GuiLayoutEnd` reads the record *below* the end pointer before testing whether a record
exists, then decrements the end pointer unconditionally:

```c
iVar1 = state->layout_end;
iVar2 = *(int *)(iVar1 - 0xc);          /* read first  */
this_00 = (int *)(iVar1 - 0x10);
if (*this_00 != iVar2) { ... }           /* test after */
...
LAB: *(int *)(state->layout_end) = *(int *)(state->layout_end) - 0x30;   /* always */
```

One `GuiLayoutEnd` without a matching Begin leaves `end < begin`. Every later "is a layout
active" test compares `begin != end`, which is still true, so the layout code then reads memory
it does not own. `GuiLayoutEndLayer` has the same shape and additionally passes the record's
first dword to `operator_delete` — which, if the top of the stack is a layout frame rather than
a layer, is the frame's mode word `1` or `2`.

`GuiEndScrollContainer` decrements three end pointers with no emptiness test.

**Workaround:** keep your own Begin/End depth counter. Never rely on the engine to catch it.

### A first-in-frame layout reads past the record it just pushed
**verified** — when the layout stack is empty, `GuiLayoutBegin*` auto-pushes a 16-byte layer
record as the base, then immediately reads it as a 48-byte layout record:

```c
if (begin == end) FUN_0081e7e0(this,1);        /* pushes 16 bytes at [begin, begin+0x10) */
...
if (*(int *)(end - 0x10) != begin_of_top) {
    param_2 = param_2 + *(float *)(begin_of_top - 0x24);   /* begin + 0x0c: inside  */
    param_3 = *(float *)(begin_of_top - 0x20) + param_3;   /* begin + 0x10: 4 PAST  */
}
```

The x term reads the record's own flag byte as a float — a denormal near zero, harmless. The
**y term reads 4 bytes past the record**, i.e. whatever follows it in the heap, and that value
is added to the frame's y origin and stored as the new frame's y cursor. In practice the first
layout in a frame can start at a slightly wrong y. The guard does not help: the record's first
dword is `0` and `begin` is a non-null heap pointer, so the branch is taken.

### One vector, two record sizes
**verified** — layout frames (48 bytes) and layers (16 bytes) share the vector at
`state+0x1a0`, pushed by two helpers that differ only in stride. Every reader hardcodes one
stride. The engine's own code always nests layer-outside-layout; a layout containing a layer, or
any other interleaving, will be read with the wrong stride.

**This is load-bearing.** The scroll container and the tooltip both do
*layer → layout → … → layout end → layer end*, and code that computes "the current layout" as
`end - 0x30` gets the right answer only because the layer is never on top when that code runs.

### `GuiCreate` can hand back a null state pointer
**reported** — on the failure path the engine sets `+0x88` to null and the Lua wrapper still
pushes the handle. Any later call dereferences null plus an offset.

## Silent no-ops

### Id pushes past 1024 are dropped
**verified** — `GuiIdPush`'s bounds test has no `else` branch. Once the stack holds 1024
entries, further pushes are discarded and every subsequent id resolution in that frame returns
the wrong value, with no diagnostic.

### Nothing unwinds an unbalanced Begin or Push
**verified** — `GuiStartFrame` resets the option, colour, z and previous-widget fields. It does
**not** touch the layout stack, the id stack, the layer stack or the scroll stacks. `GuiDestroy`
does not unwind them either. A `GuiIdPush` you forget to pop, or a layout you forget to end,
persists into the next frame and the next frame after that.

### A stale handle does nothing, silently
**verified** — 40 of 41 `Gui*` functions run the handle through the validator and, on `0`, skip
the entire body. No log line, no return value, no error. This is the single most common reason
"my GUI stopped working".

### `GuiStartFrame` wipes global state even with a bad handle
**reported** — the reset of the process-global previous-widget block happens outside the
handle check, so a call with an invalid handle skips the real frame reset but still clears
global state belonging to whatever mod was drawing.

### Passing something that is not a `gui` handle
**verified pattern** across the wrappers: they test `lua_type(L,1) == 2` and then
`if (handle != 0)`, with no `else`. A string in the first position does nothing, silently.

### `GuiOptionsAddForNextWidget` drops values of 64 or more
**verified** — the option index is split across two 32-bit halves:

```c
uVar8 = 1 << (v & 0x1f);
uVar9 = 0;
if (0x1f < v) { uVar9 = uVar8; }     /* v >= 32: move to the high half */
uVar8 = uVar8 ^ uVar9;              /* v >= 32: clear the low half  */
if (0x3f < v) { uVar9 = uVar8; }    /* v >= 64: high half becomes 0 */
pending_lo |= uVar8;  pending_hi |= uVar9;
```

`0 ≤ v < 32` sets a low-half bit, `32 ≤ v < 64` sets the corresponding high-half bit, and
**`v ≥ 64` sets neither** — no error, no log. A negative value reaches the `& 0x1f` path and
sets an arbitrary low bit. The game defines far fewer than 32 options, so this only bites on a
mistake.

Note this is *not* modulo-32 aliasing: `GuiOptionsAdd(gui, 32)` is a different option from
`GuiOptionsAdd(gui, 0)`.

## Uninitialised data

### The committed widget record carries stack garbage
**verified** — the record constructor writes through byte 75; the commit copies 80:

```c
*(undefined4 *)(gui + 0x84) = param_2[0x13];   /* never initialised */
```

Bytes 5 and 18–23 are also left untouched by the constructor. Four to eleven bytes of the
"previous widget" record — which `GuiGetPreviousWidgetInfo` reads — are stack garbage. In
practice the fields modders read are the ones that do get written, which is why this has never
been noticed.

### The per-widget colour's RGB channels are never initialised — *not a bug, a decompiler artefact*
**disproved** — this was published here earlier as a possible uninitialised read. It is not one.

`FUN_0042b6e0` builds a 20-byte RGBA value at `this`. The decompiler types it as taking a
single float, because the four-float signature is not in the call graph it infers:

```c
*(undefined4 *)(this + 0)   = 4;              /* a type tag, not a colour channel */
*(undefined4 *)(this + 4)   = in_XMM1_Da;     /* r */
*(undefined4 *)(this + 8)   = in_XMM2_Da;     /* g */
*(undefined4 *)(this + 0xc) = in_XMM3_Da;     /* b */
*(undefined4 *)(this + 0x10)= param_1;        /* a — the only stack argument */
```

The instructions are an ordinary `__thiscall` four-float constructor, `r`/`g`/`b` in XMM1–XMM3
and `a` on the stack:

```
0042b6e9  XORPS XMM0,XMM0                  ; zero +0x04..+0x13 first
0042b6ec  MOVDQU xmmword ptr [ECX + 0x4],XMM0
0042b6f3  MOVSS XMM0,dword ptr [EBP + 0x8] ; a, from the stack
0042b6f8  MOVSS dword ptr [ECX + 4],XMM1
0042b6fd  MOVSS dword ptr [ECX + 8],XMM2
0042b702  MOVSS dword ptr [ECX + 0xc],XMM3
0042b707  MOVSS dword ptr [ECX + 0x10],XMM0
```

It has **174 call sites, and all 174 define XMM1, XMM2 and XMM3 before the call** — measured
by walking back from each call to the previous control transfer and recording which argument
registers are written. So the default next-widget colour is genuinely opaque white
(`1.0, 1.0, 1.0, 1.0`), and `GuiColorSetForNextWidget` genuinely sets all four channels.

Worth knowing if you reimplement this: the same prototype-inference trap produces
`in_XMM1_Da`-style names all over the decompilation, and they are **not** evidence of an
uninitialised read. Check the call sites before believing one.

### Multi-line text ORs in an unassigned local
**reported** — `GuiText` declares a byte, never assigns it, and ORs it into the record's
mouse-over flag in the multi-line path. So hover and focus reporting for text containing a line
break includes a garbage bit. The single-line path is unaffected.

## Leaks and unbounded growth

### The widget-state map is never pruned
**verified** — the map at `state+0x64` is a red-black tree keyed by the 64-bit widget id. The
only tree operation anywhere in the GUI code is get-or-insert, whose "key already present" path
destroys the incoming payload and returns the existing node; **no erase path exists**.

A node is at least 0x200 bytes. A mod that varies its widget id per frame adds one node per id,
permanently, for the lifetime of that `Gui` object.

**Workaround:** keep your id set small and stable. A common mod pattern of
`GuiIdPush(entity_id)` or a frame counter as the id grows this without bound.

### The draw-command list only grows
**disproved** — this was published here earlier as a leak. It is not a leak.

The glyph/text draw builder `FUN_0081bbe0` appends 100-byte commands to a process-global
three-pointer vector (`begin`/`end`/`capacity`) and has no removal, which is what made it look
unbounded. But the list **is** drained, every frame, by `FUN_008279c0` — and that function has
exactly **one** caller in the whole binary, `FUN_006b3b10`, a virtual method. It is the only
thing that reads the list.

```c
/* FUN_008279c0, at the top: snapshot the list */
FUN_0095f260(begin, end, (end - begin) / 100, ...);   /* count is the command count */
...
for (cmd = begin; cmd != end; cmd += 100) { ... }     /* 100 bytes per command */

/* at the bottom, unconditionally: the drain */
end = begin;
```

The capacity check on the append path is a normal geometric grow, not an abort:
`FUN_00932e60` grows by 1.5x via `FUN_00948840` when the list is full, and the
`_Xlength_error` path is only reachable at 700 million commands.

`FUN_00efdee0` is the static destructor: it walks the list calling `FUN_0081dba0` per command,
then frees the buffer and zeroes all three pointers.

So the corrected statement is: the draw list is per-frame scratch space, allocated once and
reused, and it does not grow without bound. The one real cost is that the buffer's high-water
mark is never shrunk.

### Layout and layer stacks are unbounded
**verified** — there is no depth cap on the layout stack. A loop that begins layouts without
ending them grows the vector until memory runs out.

## Behaviour you may be relying on by accident

### Widget ids depend on the whole id stack
**verified** — a widget's effective id is its own id **verbatim** when the id stack is empty,
and an FNV-1a-64 hash chain seeded by the stack top when it is not. The same literal id
therefore identifies different widgets depending on whether anything is pushed. Anything that
"works" only inside a particular `GuiIdPush` scope will break when the scope changes.

### Two mods with the same id share state, not just pixels
**verified** — hover, focus, click latch, animation phase and scroll offset all live in the
id-keyed map. Sharing an id at the same stack depth means writing to each other's widget state.

### Only one widget can be hovered per frame
**verified** — the first widget to claim the mouse sets a flag that suppresses the rest. With
two mods drawing overlapping widgets, the second one cannot be hovered at all, and which one
wins is draw order.

### Clicking requires the pointer to still be inside
**reported** — a click is the mouse-down flag and a current hover test. There is no
press-then-drag-out case, so a button cannot be clicked by pressing it and sliding off.

### The commit resets pending state, which is why it is called after every widget
**verified** — `FUN_007d97d0` writes the record to `gui+0x38` and then resets the pending
option, colour and z fields, setting the option set to `1` (option bit 0, the default
alignment) rather than `0`. Layout and scope functions do not commit, so a frame that only
pushes layout or opens a scope leaves the previous frame's "previous widget" record in place.
`GuiGetPreviousWidgetInfo` can then report a widget that was never drawn.
