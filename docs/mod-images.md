# Editable mod images: pixel access, clearing and memory

For clearing an existing image, **write zero directly with `ModImageSetPixel`**.
In a fresh developer-build process without Debug Helper, unconditional writes
were faster at every tested fill level, including when all pixels were already
zero. Reading first adds lookup, return-value and comparison work without
avoiding an expensive operation inside the native setter.

For a **new image whose filename does not exist**, neither loop is necessary:
`ModImageMakeEditable(filename, width, height)` already fills its pixels with zero.

The native implementation was traced in the Jan 25 2025 executables. Two live
developer-build runs verified allocation cost, filename reuse, signed colors
and pixel behavior. The timing results below distinguish runs with and without
Debug Helper, whose optional Lua call history treats reads and writes differently.

## What GetPixel and SetPixel actually do

Editable images live in one process-global vector. An image id is its index;
there is no filename search, hash lookup or entity lookup inside either pixel call.
Each descriptor is 16 bytes:

| offset | contents |
|---|---|
| `+0x00` | width |
| `+0x04` | height |
| `+0x08` | index of the mod that last called `ModImageMakeEditable` for this name |
| `+0x0c` | pointer to a dense four-byte-per-pixel buffer |

With valid arguments, `ModImageGetPixel` reads three integers, checks the id,
checks `x < width` and `y < height`, and reads:

```c
pixels[y * width + x]
```

It then checks the Lua stack and pushes an integer result. `ModImageSetPixel`
reads four integers, performs the same checks and stores the supplied 32-bit
color at that address. It does not compare against the existing pixel.

There is no allocation, color blending, compression, per-pixel dirty flag,
GPU synchronization or image-wide processing in the successful setter path.
The getter also does no image allocation.

The bounds checks are unsigned: negative ids or coordinates are rejected too.
An invalid read returns `0`, indistinguishable from a transparent black pixel;
an invalid write does nothing. **The setter returns no Lua values**, including
when the write is rejected. Id zero is a reserved empty descriptor.

## Clearing an existing image

For an unconditional clear:

```lua
-- Use the actual dimensions returned for this editable image.
local id, w, h = ModImageMakeEditable(filename, 0, 0)
if id ~= 0 then
    local set = ModImageSetPixel
    for y = 0, h - 1 do
        for x = 0, w - 1 do
            set(id, x, y, 0)
        end
    end
end
```

This traverses the buffer in row order and caches the Lua function reference.
It clears the complete color to transparent black (`0x00000000`). Any color
with an alpha byte of zero is transparent; its RGB bytes need not be zero.
Preserving RGB while clearing alpha requires reading the old color, so that
operation has a different cost.

For `N` pixels, a direct clear makes `N` setter calls. A conditional clear makes
`N` getter calls plus one setter call for every nonzero pixel, and performs the
Lua comparison and branch. Already-zero pixels avoid a store, but still pay for
a complete getter. The getter has its own result/stack work. These native
operations alone do not establish their full cost through Lua and any hooks.

If `G` is the measured getter cost, `S` the setter cost and `f` the fraction of
pixels that require a write, the rough comparison is:

```text
direct:       N * S
conditional:  N * G + f * N * S + comparisons/branches
```

There is no measured universal speed ratio or crossover point. If the image is
known to be entirely zero already, skip the pass altogether.

### Measured developer-build runs

Both runs on 2026-11-25 used LuaJIT 2.0.4 with JIT enabled, one 1024 x 1024
image, regularly distributed nonzero pixels and three repetitions per case.
Each clear started from the same restored pattern. Times below are medians for
clearing the complete image; preparation and verification were outside timing.

**Without Debug Helper, in a fresh process:**

| pixels initially unequal to zero | unconditional writes | read, compare, then write | conditional slowdown |
|---|---:|---:|---:|
| 0% | 33.374 ms | 36.417 ms | 9% |
| 1% | 34.807 ms | 44.816 ms | 29% |
| 50% | 31.958 ms | 51.423 ms | 61% |
| 100% | 31.620 ms | 73.592 ms | 133% |

Unconditional writes won every median comparison. The fully changing image
took about **2.33 times as long** when read/compare/write was used. At 0%
changes the difference was smaller and individual timings were noisy; this is
one three-repetition run, not a universal bound for every machine or workload.

**Earlier run with Debug Helper enabled:**

| pixels initially unequal to zero | unconditional writes | read, compare, then write | faster in this run |
|---|---:|---:|---|
| 0% | 153.745 ms | 35.871 ms | conditional, 4.29x |
| 1% | 152.003 ms | 39.673 ms | conditional, 3.83x |
| 50% | 149.164 ms | 112.749 ms | conditional, 1.32x |
| 100% | 155.329 ms | 191.105 ms | unconditional, 1.23x |

**These are results for that mod configuration.** Debug Helper was enabled;
its Lua history filter forwards `ModImageGetPixel` without recording, while
`ModImageSetPixel` takes the recording path when history is active. That adds
source/argument inspection and other work to writes. The saved startup record
does not confirm successful history-hook installation, so this is a plausible
explanation for the large asymmetry, rather than a proven attribution.

Removing Debug Helper was the only change in the logged enabled-mod list;
the second run used a different process. The unconditional clear became
4.37-4.91 times faster across the four cases, while the all-zero conditional
median was similar (35.871 ms before, 36.417 ms after). This supports the
asymmetric instrumentation explanation. It does not independently identify
which optional hook was installed in the earlier process.

The second run retained the other five enabled mods alongside the measurement
mod, so it is a baseline without Debug Helper rather than an otherwise
unmodified game. For that configuration the implementation-based recommendation
is confirmed: use unconditional writes for a full clear. With selective
instrumentation active, the measured ranking can reverse; the earlier ratios
must not be presented as native API costs.

### Color equality has a signed-integer trap

The byte layout is `0xAABBGGRR`: alpha in the high byte, red in the low byte.
Despite the documented `uint` return type, the shipped `lua51.dll` pushes the
32-bit pixel as a **signed** integer converted to a Lua number. Thus a pixel
whose bits are `0xffffffff` is returned as `-1`.

```lua
local target = bit.tobit(0xffffffff)
if ModImageGetPixel(id, x, y) ~= target then
    ModImageSetPixel(id, x, y, target)
end
```

Normalize arbitrary packed colors with `bit.tobit` before equality comparisons.
Directly comparing a returned white pixel with the positive Lua number
`0xffffffff` gives a false mismatch. Comparison with zero has no such problem.
The setter accepts the corresponding signed bit pattern.

## What MakeEditable allocates

The memory cost follows the **pixel dimensions**, not the PNG file size or the
amount of visible content:

```text
retained pixel bytes = 4 * actual_width * actual_height
```

| dimensions | retained pixels per image |
|---|---:|
| 1024 x 1024 | 4 MiB |
| 2048 x 2048 | 16 MiB |
| 4096 x 4096 | 64 MiB |
| 8192 x 8192 | 256 MiB |
| 16384 x 16384 | 1024 MiB |

For twenty distinct 1024 x 1024 images, the retained pixel payload is exactly
**80 MiB**. Descriptors, filename bookkeeping and allocator overhead add to that.

Both live runs matched this:

| configuration | private memory before creation | after twenty images | increase |
|---|---:|---:|---:|
| Debug Helper enabled | 494.219 MiB | 574.645 MiB | 80.426 MiB |
| Debug Helper absent | 407.988 MiB | 488.410 MiB | 80.422 MiB |

All twenty returned the requested dimensions and succeeded in each run.
Reopening their filenames, collecting Lua garbage, and later dropping the Lua
ids and collecting again did not change private memory at those checkpoints.
The different process baselines are not a measurement of Debug Helper's own
memory footprint.

For a new blank image, the native path is a `malloc(width * height * 4)` followed
by a `memset(..., 0, ...)`. It does not create a second full-size blank buffer,
decode a PNG or upload a GPU texture. Zeroing touches the allocated pages; the
image is not represented as a sparse collection of nontransparent pixels.

Changing pixels to zero later does not shrink or release the buffer. Reading
first, writing fewer pixels, dropping the Lua id, or running Lua garbage
collection cannot remove this registry-owned allocation.

### Existing images and repeated filenames

The width and height arguments are **fallback creation sizes**, not a resize
request. An existing PNG is decoded at its own dimensions, with four output
channels. A previously editable filename returns its existing id and dimensions;
it is not decoded, resized or zeroed again. The call updates the owner metadata
even if no pixel was changed.

Always budget using the returned `w` and `h`. A call requesting 1024 x 1024 can
return a much larger image if that filename already exists. Different editable
names retain separate buffers; calling the same name repeatedly reuses one.
The registry is shared across mods.

New blank-image dimensions are capped individually at 16384 after requiring
positive dimensions. That cap still permits a single 1 GiB allocation. It is
not a guarantee that enough memory exists, and it is not the sizing rule for
an already loaded PNG. Failure of the blank pixel allocation returns id zero
and zero dimensions.

### Decoding and consumers can add copies

Loading an existing PNG temporarily holds file bytes and decoder work buffers
in addition to the final RGBA image. PNG processing includes an accumulated
compressed-data buffer, an inflated buffer and reconstructed pixels; some input
formats also require channel conversion. The exact peak depends on the input,
and the scratch buffers are freed. This does not apply to creating a missing,
blank image.

When game code later loads an editable image, the shared image-copy helper
allocates a separate output buffer and copies the pixels. The editable source
stays in the registry. Consumers can therefore add another full-size CPU copy,
as well as rendering resources. Some texture paths free their upload buffer;
others retain it according to their loading mode. There is no universal
"two copies forever" or "three times the image size" rule.

The four-channel copy helper also calls `memcpy` without checking its `malloc`
result first. Allocation failure during that later consumer load can therefore
be a native fault rather than a clean id-zero return from `ModImageMakeEditable`.

The pixel setter does not refresh consumers that already loaded a copy. Make
edits before the relevant game system loads the image. This is why changing the
CPU pixels after initialization can read back correctly while the visible
texture remains unchanged.

## Lifetime and out-of-memory failures

There is no exposed `ModImageFree` operation. Images are retained until the
native registry reset. The new-game loading path explicitly frees every pixel
buffer, clears the filename map and reinserts the reserved id-zero descriptor.
Growing the descriptor vector copies only 16-byte descriptors, not the images.
This path establishes retention during a run, **not an accumulating image leak
across every new game**. Other caches and allocator retention need separate
measurement; native `free` does not guarantee an immediate fall in working set.

Both checked executables, including `noita_dev.exe`, are 32-bit and carry the
large-address-aware PE flag. On 64-bit Windows that permits up to **4 GiB of
user virtual address space**, shared by everything in the process. Available
physical RAM is a different limit. [Windows memory limits](https://learn.microsoft.com/en-us/windows/win32/memory/memory-limits-for-windows-releases).

Twenty newly created blank 1024 x 1024 images explain about 80 MiB of retained
image memory. An OOM from that allocation loop alone does not establish a
larger hidden image allocation or a leak. It requires the failing process's
baseline, remaining address space and commit availability to explain. Each
image needs one contiguous 4 MiB allocation; address-space fragmentation and
other engine/mod allocations can matter.

Both measured allocation runs completed without OOM. After creation the first
still had 2724.496 MiB of entirely free virtual address regions, including a
largest region of 1979.422 MiB. Without Debug Helper those values were
2856.313 MiB and 2034.984 MiB. Neither run provides evidence of address-space
exhaustion or an unexpectedly large image allocation. These results do not
explain a different, historical failing run.

To reduce this cost, reduce dimensions or the number of distinct retained
images. Reuse an existing editable image when its previous contents are no
longer needed. If an image can be authored as a normal PNG beforehand, doing
so avoids the additional editable source registry allocation, although its
normal loading and rendering costs still exist.

## Implementation evidence

Addresses below are specific to the checked Jan 25 2025 executables.

| operation | retail executable | developer executable |
|---|---|---|
| `ModImageMakeEditable` Lua entry | `0x0082edf0` | `0x008a7f50` |
| find/create editable image | `0x0044c610` | `0x0044f8a0` |
| `ModImageGetPixel` Lua entry | `0x0082f8b0` | `0x008a8bf0`, pixel helper `0x00450100` |
| `ModImageSetPixel` Lua entry | `0x0082fbd0` | `0x008a8ef0`, pixel helper `0x004501a0` |
| editable registry begin/end | `0x01207d18` / `0x01207d1c` | `0x0132acbc` / `0x0132acc0` |
| copy editable pixels for a consumer | `0x0044c960` | `0x0044fc50` |
| free/reset editable images | `0x0044cac0`, called by `0x006e9cc0` during `0x006af010` | not separately traced |

The signed result is established by `lua51.dll`'s `lua_pushinteger` at preferred
address `0x10007360`: it uses signed `CVTDQ2PD` conversion. The retail PNG path
is `0x0044b410` -> `0x00dc8f20` -> `0x00dc8d60` -> `0x00dd0600`;
the latter releases the temporary decoder buffers after returning the pixels.
