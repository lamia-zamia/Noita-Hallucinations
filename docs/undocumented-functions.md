# The eight functions the game never documents

Eight of the 375 API functions have **no usage string in the binary**. The game does not know
what they are called or what they return, so there is no signature to quote, no arity to
cross-check, and nothing in the community stub set to compare against. Everything on this page
is read out of the disassembly.

They are listed in [api-reference.md](api-reference.md) as a bare table. This is what they
actually do.

All eight are also among the nine functions with **no arity guard** — they never check how many
arguments they were given, so passing extra arguments is harmless and passing too few is not
detected. See [how-the-api-works.md](how-the-api-works.md).

Addresses are for one Steam build and will move on update.

## `GameGetDateAndTimeLocal` — 8 values, two of them are holiday flags

The widest call in the API, and the widest *undocumented* one: it returns
**year, month, day, hour, minute, second** as integers, then **two booleans**.

The six integers are straightforward. The two booleans are the interesting part. Both are
`GetLocalTime` day/month comparisons against two hardcoded tables in `.rdata`, and both are
**gated on the current year**:

```c
iVar1 = wYear - 2021;                    /* 0x7e5 */
if (iVar1 >= 18) iVar1 = 18; else if (iVar1 < 1) iVar1 = 0;

/* boolean 1: only ever true in June */
if (wMonth == 6) { is_a = (wDay == TABLE_A[iVar1]); } else { is_a = false; }

/* boolean 2: a month/day pair per year */
if ((wDay != TABLE_B[iVar1 * 2]) || (wMonth != TABLE_B[iVar1 * 2 + 1])) { is_b = 0; }
else { is_b = 1; }
```

So the answer is **table-driven and expires**. The index is `year - 2021` clamped to `0..18`, so
the tables cover 2021 through 2039 and then hold their last entry forever. In 2040 and beyond
both booleans compare against the 2039 row and nothing else will ever match.

`TABLE_A` (`0x00fe0c98`) is a single dword per year — a **day of June**:

| 2021 | 2022 | 2023 | 2024 | 2025 | 2026 | 2027 | 2028 | 2029 |
|-----|-----|-----|-----|-----|-----|-----|-----|-----|
| 25 | 24 | 23 | 21 | 20 | 19 | 25 | 23 | 22 |

| 2030 | 2031 | 2032 | 2033 | 2034 | 2035 | 2036 | 2037 | 2038 | 2039 |
|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|
| 21 | 20 | 19 | 25 | 24 | 23 | 22 | 20 | 19 | 25 |

`TABLE_B` (`0x011535c0`) is a **month/day pair per year**, so this one is not June-bound:

| year | pair | year | pair | year | pair | year | pair | year | pair |
|------|------|------|------|------|------|------|------|------|------|
| 2021 | 3/28 | 2022 | 4/10 | 2023 | 4/2 | 2024 | 3/24 | 2025 | 4/13 |
| 2026 | 3/29 | 2027 | 3/21 | 2028 | 4/9 | 2029 | 3/25 | 2030 | 4/14 |
| 2031 | 4/6 | 2032 | 3/21 | 2033 | 4/10 | 2034 | 4/2 | 2035 | 3/24 |
| 2036 | 4/13 | 2037 | 3/29 | 2038 | 3/21 | 2039 | 3/18 | | |

The dates drift earlier each year, which is what you expect from an anniversary-style event
observed on a fixed calendar day rather than a fixed week. The binary does not say what the
event is; the tables are the whole of the mechanism.

**If you are writing a seasonal or anniversary mod, do not hardcode these.** Read them out of
the return values, and treat both booleans as false outside 2021–2039.

Note the year clamp is `if (iVar1 < 1) iVar1 = 0` — so **2020 and earlier also read row 0**,
the 2021 row. A machine with a wrong clock reading 2020 gets 2021's answer rather than
"never".

## `GameGetDateAndTimeUTC` — 6 values, no flags

The same shape minus the booleans, and it calls `GetSystemTime` rather than `GetLocalTime`:

```
year, month, day, hour, minute, second      -- all integers
```

No year table, no gating, nothing conditional. It is the plain wall clock in UTC and the least
surprising function in this list.

The difference between the two matters: the local variant's month/day are the *local* calendar
day, so for a mod keyed to a date the UTC variant can disagree with it for part of each day.

## `InputGetMousePosOnScreen` — 2 values, and it can crash

Returns the mouse position in **screen** pixels, unscaled and un-offset:

```c
piVar3 = (**(code **)(DAT_01221bc0 + 0x28))();          /* get the window/renderer  */
pfVar4 = (float *)(**(code **)(*piVar3 + 0x1c))(auStack_48);
lua_pushnumber(L, pfVar4[0]);
lua_pushnumber(L, pfVar4[1]);
```

Three things to know:

- **It is not the GUI's mouse.** The GUI keeps its own scaled position at `state+0x1e4`/`+0x1e8`
  and divides by the UI scale. This returns the raw screen position. With a UI scale other than
  1 the two disagree, and positioning a widget from this value will be off.
- **There is no null check on either call.** If the renderer is not up, `*piVar3` is a
  dereference of null and the call returns garbage that is then dereferenced again. It is safe
  in a normal frame and unsafe in a pre-render callback.
- It writes into a 68-byte stack buffer and reads back the first two floats.

## `Debug_SaveTestPlayer` — a stub that does nothing

```c
undefined4 FUN_007e4a60(void) { return 0; }
```

Seventeen instructions of nothing: no arguments read, no Lua calls, no side effects. It is a
compiled-out debug helper. Calling it is a no-op — no error, no log line, no return value.

This is the clearest case in the API of a function that survives in the export table with no
implementation behind it. If you are porting a mod from a dev build, this is why nothing
happens.

## The four `Streaming*` functions

All four are Twitch-integration leftovers, and all four have the same shape: **lazily construct
a `StreamingIntegrationLuaManager` if it does not exist, then call through it**.

```c
if (DAT_01204704 == 0) {
    p = operator_new(0xc);                          /* a 12-byte manager */
    DAT_01204704 = p ? FUN_00438c50(p) : 0;
}
```

The `Streaming*` family that *is* documented (`StreamingGetIsConnected`,
`StreamingSetVotingEnabled`, `StreamingSetCustomPhaseDurations`) is harmless. These four are
worth avoiding:

- **`StreamingGetConnectedChannelName`** → one string, via `FUN_00439230`.
- **`StreamingGetRandomViewerName`** → one string, via `FUN_00439350`.
- **`StreamingForceNewVoting`** → no return value. It tests a flag at `+0x4c8` on the manager and,
  only if set, writes `+0x4e0 = 1` and a zeroed 64-bit value at `+0x4e8`. **The flag test has no
  `else`**, so with no stream connected the call does nothing at all — silently.
- **`StreamingGetVotingCycleDurationFrames`** → one integer. It sums two phase durations
  (`+0xea4` and `+0xea8`) and multiplies by `60.0`, i.e. **seconds to frames at 60 fps**, then
  truncates to an integer. The truncation is not rounding: a 2.5-second cycle reports 150, a
  2.99-second cycle also reports 179 rather than 180.

The two string getters return a single string with **no way to distinguish "no stream" from
"empty name"**, and neither checks whether the manager was successfully constructed — if
`operator_new` fails, `DAT_01204704` is null and the next line dereferences it.

## `EntitySave` — has a signature, still does nothing

Not one of the eight, but the same story and worth stating because it *looks* documented.
`EntitySave` has a usage string — `EntitySave( entity_id:int, filename:string ) [Note: works
only in dev builds.]` — and no arity guard. Its entire body builds that string and hands it to
the error sink:

```c
FUN_007efbc0(L, 0x1018690, usage_string);    /* log it, do nothing else */
```

So in a release build `EntitySave` **logs its own documentation to the error log and returns**.
Its documentation is accurate; the note about dev builds is the whole implementation. Because
of the error-dedup rule, you see this line once per session no matter how often you call it.

## Practical advice

None of these eight are useful to a mod author, and three of them are actively misleading:

- Do not use the two date functions for anything that must keep working. The holiday booleans
  are hardcoded to 2021–2039 and drift.
- Do not use `InputGetMousePosOnScreen` to position GUI widgets; it is unscaled screen space and
  the GUI already tracks its own.
- Avoid the `Streaming*` family entirely unless you are writing a Twitch integration, and treat
  the null-manager path as a crash risk.
- Do not expect `Debug_SaveTestPlayer` or `EntitySave` to do anything. Both are stubs in a
  release build.

Their value is as evidence: they are the boundary of what the game itself knows about its own
API, and a good test case for any reimplementation, because a reimplementation that implements
them *correctly* is more faithful to the original than one that implements them usefully.
