# Editable image API implementation

`ModImageMakeEditable` places an image in a global native registry.
`ModImageGetPixel` and `ModImageSetPixel` access that registry's pixel buffers
directly. The buffers are ordinary uncompressed heap allocations, separate from
textures or other image copies made by game systems.

Addresses refer to the Jan 25 2025 build. The C-like pseudocode below preserves
the relevant branches, memory accesses and Lua return counts; descriptive names
replace unnamed symbols, and compiler exception handling and logging-string
construction are omitted.

## Pixel storage

Each image has a 16-byte descriptor in a contiguous vector:

```c
struct EditableImage {
    uint32_t width;       // +0x00
    uint32_t height;      // +0x04
    int32_t  owner_mod;   // +0x08
    uint32_t *pixels;    // +0x0c; 32-bit pointer
};

struct ImageResult {
    uint32_t id;
    EditableImage *image;
};
```

The vector has three native pointers:

| pointer | retail address | meaning |
|---|---|---|
| `begin` | `0x01207d18` | first descriptor |
| `end` | `0x01207d1c` | one past the last descriptor |
| `capacity_end` | `0x01207d20` | end of descriptor allocation |

An image ID is the descriptor's index, with no generation counter or per-mod
namespace. The count is `(end - begin) / 0x10` when calculated in bytes. ID zero
is reserved for `{width=0, height=0, owner_mod=-1, pixels=NULL}`.

Pixels are stored in row order, with no row padding, per-pixel objects or
separate alpha plane:

```text
descriptor address = begin + id * 0x10
pixel index        = y * descriptor.width + x
pixel address      = descriptor.pixels + pixel_index * 4   (byte arithmetic)
```

For example, in an image four pixels wide, `(x=2, y=1)` selects word 6, at byte
offset 24 from the pixel buffer. Width, height and ownership live in the
descriptor, outside that allocation.

## Pixel format and Lua values

Each word has numeric format `0xAABBGGRR`. On x86 its four bytes in memory are
`R, G, B, A`, in that order. The setter stores the resulting 32-bit word without
blending, premultiplication, channel swapping or alpha-dependent handling.

The getter passes the word to `lua_pushinteger`, whose integer argument is
signed 32-bit in the shipped LuaJIT DLL. Consequently, words with the high bit
set become negative Lua numbers: `0xffffffff` is returned as `-1`. The declared
`uint` result does not change this conversion. `bit.tobit(color)` gives a packed
color the same signed representation for comparison.

The numeric arguments use `lua_tointeger`, not checked-integer helpers. The
resulting ID and coordinates are compared as unsigned 32-bit values. Negative
indices therefore fail the bounds checks. General argument-conversion and
logging rules are described in [how the API works](how-the-api-works.md).

## `ModImageGetPixel`

Retail entry: `0x0082f8b0`.

```c
int ModImageGetPixel(lua_State *L) {
    if (lua_gettop(L) < 3) {
        report_arity_error(L, 3);
        return 0;                         // no Lua results
    }

    uint32_t id = (uint32_t)lua_tointeger(L, 1);
    uint32_t x  = (uint32_t)lua_tointeger(L, 2);
    uint32_t y  = (uint32_t)lua_tointeger(L, 3);
    uint32_t color = 0;

    if (id < image_count()) {
        EditableImage *image = begin + id; // typed pointer arithmetic
        if (x < image->width && y < image->height)
            color = image->pixels[image->width * y + x];
    }

    lua_checkstack(L, 1);
    lua_pushinteger(L, (int32_t)color);
    return 1;
}
```

The successful path performs one 32-bit pixel load. It does not look up a
filename, decode an image or allocate a pixel buffer. Invalid IDs and
coordinates return one Lua value, zero, without a diagnostic. This is
indistinguishable from reading a pixel whose word is zero.

The stack-capacity check's return value is not tested. The retail pixel path
does not separately check the backing pointer for NULL after accepting the
descriptor and coordinates.

## `ModImageSetPixel`

Retail entry: `0x0082fbd0`.

```c
int ModImageSetPixel(lua_State *L) {
    if (lua_gettop(L) < 4) {
        report_arity_error(L, 4);
        return 0;
    }

    uint32_t id    = (uint32_t)lua_tointeger(L, 1);
    uint32_t x     = (uint32_t)lua_tointeger(L, 2);
    uint32_t y     = (uint32_t)lua_tointeger(L, 3);
    uint32_t color = (uint32_t)lua_tointeger(L, 4);

    if (id < image_count()) {
        EditableImage *image = begin + id;
        if (x < image->width && y < image->height)
            image->pixels[image->width * y + x] = color;
    }
    return 0;                             // no Lua results
}
```

The actual store is one `MOV` of a 32-bit value to the calculated address
(`0x0082feb9`). It does not compare the previous value. Writing the same value
still performs the store.

There is no image allocation, copy-on-write, dirty flag, texture upload,
filename lookup or image-wide operation in this path. An invalid ID or
coordinate silently skips the store. The function returns no Lua values on
both success and rejection; it does not return a success flag or numeric zero.

## `ModImageMakeEditable`

Retail Lua entry: `0x0082edf0`; find/create helper: `0x0044c610`.

The wrapper performs these operations in order:

1. Require at least three Lua arguments. Too few arguments produce an arity
   diagnostic and no Lua results.
2. Require both initialization flags, at `0x01207f10` and `0x01207efd`, to be
   nonzero. Otherwise log that the init API does nothing at this point and
   return no Lua results. This check precedes any filename lookup.
3. Read the filename through `lua_isstring` / `lua_tolstring`, with the shared
   argument-error helper on the other branch. Copy it into a native string,
   then lowercase its bytes using `0x00ddfab0`.
4. Read fallback width and height with `lua_tointeger`.
5. Require a current initializing-mod pointer at `0x01207f0c`. A missing pointer
   produces the `called outside mod initialization` diagnostic and three zero
   results.
6. Extract the filename extension and require `png`. A different extension
   produces a diagnostic and three zero results. The full name was already
   lowercased, so extension capitalization is normalized.
7. Call the find/create helper. Its owner argument is the current mod's index:
   `(current_mod - mods_begin) / 0x60`, with `mods_begin` at `0x01207e9c`.
8. Reserve room for three stack values and push `id`, `width`, `height`. A
   failed find/create result has no descriptor and becomes `(0, 0, 0)`.

The global function is removed from mod Lua states after initialization; see
[Lua execution](lua-execution.md#after-magic-numbers-nine-globals-disappear).

### Filename index

The filename index maps a native string to an image ID. Lookup at `0x008af530`
hashes the string bytes with 32-bit FNV-1a:

```c
uint32_t hash = 0x811c9dc5;
for (uint8_t byte : filename)
    hash = (hash ^ byte) * 0x01000193;
```

It selects a bucket and compares full strings within the bucket; the hash is
not the image ID. The native lookup result contains the ID at node offset
`+0x20`. Helper `0x0044ca20` returns that ID and computes its descriptor pointer
from the global vector. A miss returns `{0, NULL}`.

This index is used when resolving filenames. Neither pixel-access function
uses it: they already receive the vector index.

### Find-or-create helper

The helper's control flow is equivalent to:

```c
ImageResult find_or_create(const char *name, int32_t w, int32_t h,
                           int32_t owner) {
    if (name == NULL)
        return {0, NULL};

    ImageResult existing = find_by_filename(name);
    if (existing.id != 0) {
        existing.image->owner_mod = owner;
        return existing;
    }

    uint32_t *pixels = NULL;
    int32_t decoded_w = 0, decoded_h = 0;
    if (starts_with(name, "data/") || starts_with(name, "mods/"))
        pixels = load_image(name, &decoded_w, &decoded_h, 4,
                            preserve_rgb_order=true, bypass_editable=true);

    if (pixels != NULL) {
        w = decoded_w;
        h = decoded_h;
    } else {
        if (w <= 0 || h <= 0)
            return {0, NULL};
        w = min(w, 16384);
        h = min(h, 16384);
        size_t bytes = w * h * 4;
        pixels = malloc(bytes);
        if (pixels == NULL)
            return {0, NULL};
        memset(pixels, 0, bytes);
    }

    uint32_t id = image_count();
    append_descriptor({w, h, owner, pixels});
    filename_index[name] = id;
    return {id, begin + id};
}
```

Reusing a filename changes only `owner_mod`. It does not reload, resize,
initialize or copy the pixel buffer. The dimensions are creation fallbacks,
not a resize request. A successfully loaded file keeps its decoded dimensions.

The `data/` / `mods/` prefix test controls whether file loading is attempted.
It does not enclose the fallback allocation branch. A permitted extension and
positive fallback dimensions can therefore create an entry even when that
prefix test fails. Loading or decoding failure also reaches fallback creation.

The descriptor append (`0x00895aa0`) copies four words into the vector. The
filename index stores the new ID, not a second pixel buffer. Descriptor-vector
growth moves those small records; existing pixel allocations and IDs remain
unchanged. Native descriptor pointers may move when the vector grows.

## Allocation and ownership

Each retained buffer occupies exactly `width * height * 4` payload bytes.
The descriptor, filename index and heap allocator add separate overhead.
Pixel values and image compressibility do not affect this allocation size.
There is no lazy allocation of individual pixels.

New blank buffers are obtained with `malloc` and initialized with `memset` before
registration. Both fallback dimensions must be positive; each is capped at
16384. The largest fallback allocation is therefore 1 GiB. This fallback cap
is not applied to a successfully decoded file's dimensions.

Failure of the blank pixel allocation is checked and produces the zero-ID
result. Allocating descriptor storage and filename-index nodes is a separate
operation; the vector growth path can raise a native allocation exception.
The image registry owns the pixel buffer after registration. A Lua ID is only
an integer, so Lua garbage collection cannot release the allocation.

## Loading and consumer copies

MakeEditable invokes the image loader at `0x0044b410` with four requested
channels and both control flags set. Those flags bypass the editable-image
copy path and suppress its optional red/blue swap. The loader reads file bytes
through the virtual filesystem, decodes them, releases the input-file buffer
and returns the decoded pixel allocation for registration.

For PNG input, the path includes `0x00dc8f20`, `0x00dc8d60` and `0x00dd0600`.
Compressed-data accumulation, inflation, reconstruction and possible channel
conversion can require additional temporary allocations. Decoder work buffers
are released after the output allocation is detached. They are not part of
the retained editable descriptor.

Other game systems can request a copy of editable pixels through `0x0044c960`:

```c
ImageResult result = find_by_filename(name);
if (result.id == 0)
    return NULL;

EditableImage *image = result.image;
dst = malloc(image->width * image->height * requested_channels);
if (requested_channels == 4)
    memcpy(dst, image->pixels, image->width * image->height * 4);
else {
    for (uint32_t i = 0; i < image->width * image->height; ++i) {
        uint8_t *src_pixel = (uint8_t *)image->pixels + i * 4;
        uint8_t *dst_pixel = (uint8_t *)dst + i * 3;
        dst_pixel[0] = src_pixel[0];
        dst_pixel[1] = src_pixel[1];
        dst_pixel[2] = src_pixel[2];
    }
}

*width_out = image->width;
*height_out = image->height;
if (channels_out != NULL)
    *channels_out = requested_channels;
return dst;
```

The non-four-channel branch emits three bytes per pixel; it is not a general
channel converter. The observed requests are for three or four channels.
Unlike blank-image creation, this helper does not check `malloc` before using
the returned pointer. Allocation failure can fault during the copy.

Consumers own their copies independently. Pixel writes modify only the editable
source; they do not update previously copied image data or invalidate an
existing texture. Some texture-loading modes free the CPU upload buffer and
others retain it. That ownership belongs to the consumer, not to SetPixel.

## Registry reset

The reset routine at `0x0044cac0` executes this sequence:

```c
for (EditableImage *p = begin; p != end; ++p)
    if (p->pixels != NULL)
        free(p->pixels);

end = begin;
filename_index.clear();
reset_filename_buckets();
append_descriptor({0, 0, -1, NULL});
```

The new-game loading path reaches it through `0x006af010` -> `0x006e9cc0`.
It retains the small descriptor-vector capacity while releasing the pixel
buffers. IDs can subsequently be reused for different images. There is no
exposed Lua operation to free one editable image individually.

## Address reference

| operation | retail executable | developer executable |
|---|---|---|
| MakeEditable Lua entry | `0x0082edf0` | `0x008a7f50` |
| find/create helper | `0x0044c610` | `0x0044f8a0` |
| GetPixel Lua entry | `0x0082f8b0` | `0x008a8bf0` |
| GetPixel memory-access helper | in the Lua entry | `0x00450100` |
| SetPixel Lua entry | `0x0082fbd0` | `0x008a8ef0` |
| SetPixel memory-access helper | in the Lua entry | `0x004501a0` |
| registry begin/end | `0x01207d18` / `0x01207d1c` | `0x0132acbc` / `0x0132acc0` |
| copy editable pixels | `0x0044c960` | `0x0044fc50` |
| reset editable registry | `0x0044cac0` | not separately mapped |

The developer access helpers add assertions for invalid native descriptor or
backing-buffer pointers. Their valid pixel access uses the same descriptor
layout, unsigned bounds checks and word addressing. The shipped DLL's
`lua_pushinteger` at preferred address `0x10007360` uses signed `CVTDQ2PD`
conversion for the Lua result.
