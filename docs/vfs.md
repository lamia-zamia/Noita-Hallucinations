# The virtual file system

How Noita turns a string like `data/scripts/init.lua` into bytes, why your mod's copy of
a file wins over the vanilla one, and what happens when you ask for a file the game would
rather you did not.

Everything here is from one Steam build of the 32-bit `noita.exe`, imagebase `0x00400000`.
Addresses move when the game updates; the *behaviour* does not. Treat each address as a
pointer into one build.

- [lua-execution.md](lua-execution.md) - the other half: how the Lua states that call into
  this file system are created, and when.
- [api-reference.md](api-reference.md) - the 375 registered functions.
- [how-the-api-works.md](how-the-api-works.md) - the global behaviour rules the file
  functions inherit.

## The short version

| question | answer |
|---|---|
| What is it? | a device list plus a mount table, owned by a static singleton |
| Search order | **first device that answers wins**, in registration order |
| Do mods shadow `data/`? | **yes** - the mod device is inserted at the *front* of the list |
| Is it case-insensitive? | for most reads, yes - the path is lowercased first. But not all |
| Can Lua read anything? | no - only `data/` and `mods/`, and only 7 file extensions |
| Can Lua read a mod's own `mod.xml`? | **no** - explicitly blacklisted; you get `""`, which is truthy |
| Is there a `.pak`/`.wak` archive? | **yes, and the retail game uses it** - `data/data.wak` holds 14 745 files including all of `data/scripts` |

## The layer stack

The file system is not bespoke Noita code. It is part of **`poro`**, the engine framework
the game is built on - the same framework that owns the OpenGL renderer, the joystick, the
event recorder and the font and tween utilities. A leaked source path names it:

```
d:\projects\nollagames\fallingeverything\build\vc12\..\..\source\misc_utils\wizard_app_config.cpp
```

(There are 56 `d:\projects\nollagames\` strings in the binary; 14 of them are `poro\source\...`
paths, which is what identifies the framework.)

The relevant RTTI classes (MSVC type descriptors, recovered from the `.rdata`):

| class | role |
|---|---|
| `poro::IApplication` / `poro::DefaultApplication` | the process-level owner of the engine's subsystems |
| `poro::IFileDevice` | **the VFS device interface** - one per place files can come from |
| `poro::DiskFileDevice` | the plain-filesystem device |
| `poro::IStreamInternal@platform_impl` | the Win32 stream implementation behind a device |
| `poro::ReadStream` | the handle a caller holds: a path, a size, an open inner stream |
| `poro::FileDataTempReleaser` | RAII wrapper for a stream backed by a temp file |
| `poro::AppConfig` | where the save/user directories are configured |
| `poro::IPlatform` / `PlatformDesktop` / `PlatformWin` | the OS layer |
| `poro::EventRecorder` / `EventPlaybackImpl` | the replay/demo system, same stream machinery |

Plus three device classes that are *not* in the `poro` namespace and are specific to the game:
`WizardPakFileDevice`, `ModDiskFileDeviceCaching` and `ModDiskFileDevice`.

A read, end to end:

```
ModTextFileGetContent("mods/mine/data/x.xml")     0x0082de60   Lua binding
  lowercase the path                              0x00ddfab0
  pass the access gate                             0x0082cf80
  open                                            0x00dbc560
      per-thread ReadStream, reused if the path is unchanged
      -> real open                                 0x00dbc940
          walk the device list, first hit wins      device vtable +0x0c
  read                                            0x00dbc820   ReadStream vtable +0x00
  close                                           0x00dbc6a0
push string, or nil
```

The read is a one-slot forward: the `poro::ReadStream` at `0x00ff8c1c` has exactly one entry,
which loads the inner stream and tail-jumps to its own slot `+0x0c`.

## The device list

One static singleton, at `0x01221bc0`, holds a pointer to the file system at offset `0x70`.
Its vtable is installed at runtime, so the global is all zeros in the file on disk; a single
instruction at `0x004139aa` writes the table address in, and the runtime table is at
`0x01051b84` and is 44 slots wide (`0x00` to `0xAC`, all 44 of them real code pointers; 28 distinct
slots are called from the rest of the program). Callers reach it as:

```asm
mov ecx, 0x1221bc0            ; this
mov eax, [0x1221bc0]          ; the object's VTABLE, not the object
call [eax + 0x24]             ; -> mov eax,[ecx+0x70]; ret
```

Slots that matter:

| slot | meaning |
|---|---|
| `+0x00` | scalar deleting destructor (`0x00db9000`) |
| `+0x14` | setter: stores one pointer argument at `this+0x04` |
| `+0x24` | return the file-system object - the device list + mount table |
| `+0x28`, `+0x30` | return a pointer to a small object whose `+0x04` begins a `{begin,size,capacity}` region - both are passed straight to vector mutators, so neither is a string |
| `+0x34` | return a count (`(end - begin) >> 2`) |
| `+0x38` | index -> return an element, which is itself a vector; the payload is `*(int *)([elem+8] + index*4)` |

The file-system object itself:

| offset | contents |
|---|---|
| `+0x00` | device vector begin |
| `+0x04` | device vector end |
| `+0x0c` | mount vector begin |
| `+0x10` | mount vector end |
| `+0x18` | a 4-byte MSVC `mtx_t` (a `std::mutex`), taken by both mount-table functions |

The binary imports no SRWLOCK and no critical-section symbol at all - only `_Mtx_init`,
`_Mtx_lock`, `_Mtx_unlock` and `_Mtx_destroy` from `MSVCP120.dll`, which is both why the field
is 4 bytes and why it is a `std::mutex`.

### The device interface

Each device is a vtable (the mod device's is 13 slots). Only **two** of them are used to
resolve a read:

| slot | meaning |
|---|---|
| `+0x00` | scalar deleting destructor |
| `+0x0c` | **open**: `(out_stream, path)`, returns null if this device does not have it |
| `+0x1c` | **exists**: does this device have this file |

Two further slots exist but are used only by the path-resolution / directory-walk function
(`0x00dbcd90`), which the read path never reaches:

| slot | meaning |
|---|---|
| `+0x18` | resolve to the real absolute path on disk |
| `+0x2c` | does this path belong to this device |

**First match wins.** Both the exists test (`0x00dbcf70`) and the open (`0x00dbc940`) walk
the vector from the front and stop at the first device that answers:

```c
// 0x00dbcf70 - "does any device have this file?"
for (p = fs->dev_begin; p != fs->dev_end; ++p)
    if (p->vtable[0x1c/4](path)) return 1;
return 0;
```

There is no longest-prefix match, no scoring, and no merge. Order is the whole story.

### Mods go to the front

At mod-discovery time the game inserts a mod device with `std::vector::insert` at
`devices.begin()` (`0x008af2b0`, called twice from the mod-initialisation function at
`0x00834620`, both times with the vector's own begin pointer as the position).
`vector::insert` places the new element *before* the position it is given, and the position
given is the first element. So the mod device is searched **before** the base filesystem
device, and a mod that ships `data/scripts/foo.lua` shadows the vanilla
`data/scripts/foo.lua`. That is the whole mod-override mechanism: there is no diffing, no
content merging, and no notion of "unshadowing".

There *is* one per-file structure on the read path, and it is worth not confusing with an
override table. `ModDiskFileDeviceCaching` is 12 bytes - vtable, a container at `+0x04`, and a
device pointer - and its `open` (`0x0082cc80`) lowercases the path and looks it up in that
container on **every** open. The container is built once per mod initialisation (the progress
line is `Mod filesys cache build time`). It is a path-resolution index, not a shadow table:
it does not change which device wins, only how that device finds its own files.

Across a mod hot-reload only `DiskFileDevice` and `WizardPakFileDevice` instances survive;
the mod device is torn down and rebuilt (`0x008358d0`, which deletes every element that is
neither, then calls the mod-initialisation function again). That the archive device is the
one explicitly preserved is the code's own acknowledgement that it is worth keeping - an
allowlist is suggestive rather than conclusive. The much stronger evidence is at
initialisation: `0x0085e240` asks the device list whether `data/data.wak` exists and, if so,
constructs a `WizardPakFileDevice` and indexes the archive *through the VFS* before the game
starts. See [the archive section](#the-datawak-archive).

## The mount table

Save and user data are not addressed by real paths. They use a five-character virtual
prefix: `??` plus a three-letter mount name, then `/`.

| prefix | directory when user data is in the per-user directory | with `-always_store_userdata_in_workdir` |
|---|---|---|
| `??SAV` | `save00` | `save00` |
| `??S00` | `save00` | `save00` |
| `??USR` | `save_shared` | `save_shared` |
| `??REC` | `save_rec` | *(empty - the work dir itself)* |
| `??STA` | `save00/stats` | `save_stats` |

All five are registered in one block of the start-up function (`0x0080a550`,
lines 155-295). The `root` argument is a single global, `0x01204ba0`: `0` means "resolve
against the per-user application directory", `1` means "resolve against the working
directory". It is set by **two** command-line flags, not one:

- `-always_store_userdata_in_appdata` - forces `0`
- `-always_store_userdata_in_workdir` - forces `1`

Which physical directory `0` resolves to is not settled here; the flag names point at AppData,
and a Steam install is also consistent with the Steam userdata directory.

**`??SAV` and `??S00` are the same directory.** The split is semantic, not physical:

- `??SAV/` is the *current run's* save - `player.xml`, `world_state.xml`,
  `world/world_*.png_petri`, `.autosave*`. A new run deletes six named files and walks
  `??SAV/world`; `??S00/*` is never touched.
- `??S00/` is *persistent* state that outlives that - `mod_config.xml`,
  `mod_settings.bin`, `persistent/flags/`, `persistent/orbs/`, `persistent/bones_new/`.

They are the same on-disk folder, so a mod that writes to `??S00/` and a mod that writes to
`??SAV/` can see each other.

### Mount entries

A mount entry is 0x34 bytes:

| offset | contents |
|---|---|
| `+0x00` | `std::string` prefix, e.g. `??SAV` (5 chars, compared in full) |
| `+0x18` | `int` root selector |
| `+0x1c` | `std::string` directory |

Registration (`0x00dbd6e0`) takes the mutex, walks the vector comparing prefixes **in full**,
replaces the `dir` and `root` of an exact match in place, and otherwise **appends**. So the
final order is registration order, and re-registering a prefix does not move it.

### Expansion

`0x00dbd470` is the expander, and its test is exact:

```asm
cmp  eax, 5          ; length must be > 5
jbe  no_mount
cmp  byte ptr [ecx], 0x3f   ; path[0] == '?'
jne  no_mount
cmp  byte ptr [ecx+1], 0x3f ; path[1] == '?'
jne  no_mount
```

then it takes `substr(0, 5)` as the key, linearly matches it against every registered
prefix, and on a hit joins `entry.dir` with `path.substr(5)`, resolving the result against
`entry.root`. On a miss the path is resolved as-is against the caller's own root.

Two consequences worth internalising:

- **`data/` and `mods/` are not mounts.** They are ordinary relative paths resolved against
   the game directory. `data/scripts/init.lua` is just a path; the only reason it always
   works is that a base disk device is in the list.
- **The mount name is always exactly three characters.** `substr(0, 5)` means a hypothetical
  `??SAVES/` would look up the key `??SAV` and then produce `ES/...`. The join adds no
  separator of its own - the `/` in a well-formed path is part of the tail - so the result is
  `save00ES/...`, not `save00/ES/...`. The five names above are the whole vocabulary, and the
  5th character is never actually tested, so `??SAVX/` would match `??SAV` too.

The expansion ends by widening to UTF-16 and building an absolute Windows path. Note that the
binary imports no `CreateFileA` and no `CreateFileW` at all: the wide open it does use is
`std::_Fiopen` from `MSVCP120.dll`, which reaches `CreateFileW` inside the CRT.

## What Lua is allowed to read

`ModTextFileGetContent` is the odd one out among the `Mod*` functions - its own signature
string says so ("Unlike most Mod* functions, this one is available everywhere"). It is also
the only file-reading entry point, and it is gated (`0x0082cf80`). A read succeeds only if
**all four** hold:

1. **The extension is one of seven** (`0x011526b0`):

   | | | | |
   |---|---|---|---|
   | `txt` | `csv` | `xml` | `lua` |
   | `frag` | `vert` | `json` | |

   No `.png`, no `.ogg`, no `.bin`, no extension at all. Images and sounds go through
   `ModImage*` and the music-bank API instead.

2. **The path starts with `data/` or `mods/`** (`0x011531ec`). Nothing else is reachable
   from Lua - not `??S00/`, not `..`, not an absolute path.

3. **After the first path component is stripped, the name is not one of three files**
   (`0x01152760`):

   | blacklisted | why it exists |
   |---|---|
   | `compatibility.xml` | the mod's declared compatibility list |
   | `mod.xml` | the mod's manifest |
   | `mod_id.txt` | the mod's id |

   `ModTextFileGetContent("mods/mine/mod.xml")` returns **an empty string** - not nil, and
   not the file. That is a real trap, because `""` is truthy in Lua: a mod that writes
   `if ModTextFileGetContent(p) then` will not notice. The log line is
   `file cannot be read: ` followed by the path, emitted by the caller rather than by the
   gate. These three files are read by the C++ side directly through the device list,
   bypassing this gate. A mod that wants to read its own manifest must ship the data
   somewhere else, or ask the game's own mod code.

4. The file has to actually resolve - see the search order above.

The gate's three pieces are three independent flags, combined with a genuine three-way AND:
if the extension is not allowed the result is 0; otherwise the prefix is tested, and if that
passes the blacklist is tested. A read that fails any one of them returns false. (The
decompilation of the gate reuses one register for all three and carries a "type propagation
algorithm not settling" warning, which makes it *look* like the last check overwrites the
first. It does not - the prefix result is deliberately copied aside before the blacklist call.
If you are reimplementing, do not write the lossy overwrite; write the AND.)

### The first component is stripped, not the mod directory

The gate calls a helper (`0x00de2760`) that finds the first `/` or `\` in the path - the two
marker strings are the single characters `"/"` and `"\"` - and returns everything after it:

```
"data/scripts/init.lua"        ->  "scripts/init.lua"
"mods/mine/data/x.xml"         ->  "mine/data/x.xml"
"mods\\mine\\data\\x.xml"      ->  "mine\\data\\x.xml"
```

A path with **no** separator is returned unchanged, not emptied. The two marker strings are
single characters, `/` and `\`, but they are built at run time so their identity is an
inference from the surrounding code rather than something the file image shows.

This is what makes the documented equivalence true: `mods/mod/data/file.xml` and
`data/file.xml` name the same file, because the canonical key is the lowercased path with
its leading component removed, and the mod's directory is matched separately, by the mod
device, not by the path string.

### Separator handling is not normalised

The engine's own code is inconsistent about separators - it builds `"/init.lua"` and
`"/mods/"` + id with forward slashes in one place and `"\mod.xml"` and `"\compatibility.xml"`
with backslashes in another (the Lua-visible path prefix test uses the separate literal
`mods/`, without a leading slash). No normalisation pass was found. On Windows this is invisible; it will bite a reimplementation that assumes one
separator.

## Case sensitivity

Most reads lowercase the entire path before doing anything else (`0x00ddfab0`, a plain
`tolower` loop over the string). So `Data/Scripts/Init.lua` and `data/scripts/init.lua` are
the same file for those calls. It is called from 39 sites in 30 distinct functions.

**This is not uniform, and the inconsistency is observable:**

| function | lowercases? | gated? |
|---|---|---|
| `ModTextFileGetContent` | yes | extension + prefix + blacklist |
| `ModTextFileSetContent` | yes | same gate |
| `ModTextFileWhoSetContent` | yes | **no gate** - it only consults the who-set registry |
| `ModDoesFileExist` | **no** | **no gate at all** |
| `ModLuaFileGetAppends` | **no** | - |

`ModDoesFileExist` is the odd one out twice over. It does not lowercase, and it applies
neither the extension allowlist nor the `data/`+`mods/` prefix check - it just asks the
device list. It also has a fallback: if the file-system singleton is null it calls
`_findfirst64i32` on the raw path, bypassing the VFS entirely.

Practical upshot: `ModDoesFileExist` and `ModTextFileGetContent` answer different questions.
The clearest case is not case at all - `ModDoesFileExist("mods/mine/x.png")` is `true` while
`ModTextFileGetContent("mods/mine/x.png")` is `""`, because only one of them applies the
extension allowlist.

## The per-thread stream cache

`0x00dbc560` keeps one `poro::ReadStream` in thread-local storage at `*(TLS+4)`, created
lazily. The stream caches its path at offset `+0x0c`. On entry it compares the requested
path with the cached one and, **if the strings are identical, does nothing at all** - no
re-open, no device walk. Only a different path (or a different length) tears down the inner
stream and re-opens.

That is a one-entry memo per thread, not a content cache. It matters because the same file
is re-opened constantly within a frame; it does not save reads across frames and it never
caches bytes. The wrapper object is 0x24 bytes with no buffer in it at all, and its single
vtable slot forwards straight to the inner stream.

`0x00dbc6a0` is the matching close, and note the polarity: it releases the inner stream and
clears the path **when the cached path is not empty**. An empty path is the already-closed
sentinel, so a double close is a no-op.

## Recording who changed what

When a mod replaces a file's content, the game remembers which mod did it, so that
`ModTextFileWhoSetContent` can answer. The registry is an object at `0x01207ec0` whose
`+0x04` is a hash container keyed on the lowercased path, holding a **mod index**; the node
layout (an `int` at `+0x50` with a `-1` "no mod" sentinel, a second string at `+0x54`, and a
secondary lookup that takes a node rather than a key) rules out a `std::map`. The index is
resolved back to a mod id by indexing the 0x60-stride mod table, which is where the `0x60`
from [lua-execution.md](lua-execution.md) comes from. A path that is not in the registry
returns an empty string, not an error.

## Directory enumeration

- `0x00dbcd90` - resolve a path by asking each device whether the path is inside it (device
  slot `+0x2c`), then asking that device for the real path (slot `+0x18`). Falls back to
  plain expansion when nothing matches.
- `0x00dbceb0` - resolve against the mount table alone. There is no device parameter and no
  device vtable call in it: it expands the `??XXX` prefix and then makes the result absolute.
  In other words it is the *fallback half* of `0x00dbcd90`, promoted to its own entry point.

Neither is reachable from Lua. Mod *discovery* is not a directory walk of `mods/` at all: it
starts from the enabled-mod list and probes each candidate's `mod.xml` through the device
list.

## Hot reload

The game watches the mods directory for changes with the Win32 change-notification API:

```c
FindFirstChangeNotificationW(L"mods/", TRUE, 0x15f)
```

`0x15f` is every change class except last-access and EA. The handle lives in
`lpHandles_011536b0` and the call is made once per process, from `0x0082b5d0`, at the very
start of mod initialisation. So dropping a folder into `mods/` mid-session is noticed
without a restart - see [ui-modding.md](ui-modding.md) for what that does and does not
reload.

## Writing

Writes do not go through the device *list* - there is no writable device and no read-side
shadowing to reason about. A write resolves the path to an absolute Windows path and then
makes one virtual call on an object the file system owns at `fs+0x1c`; that is the whole
layering. The general whole-file writer is `0x0056ccc0`, called from 13 places: it writes a
4-byte length twice, then the payload, zlib-compressing first when the payload is 0x80 bytes
or more.

Named writable locations:

| path | written by |
|---|---|
| `??S00/mod_settings.bin` | mod settings save, `0x00837f10` (the loader of the same file is `0x00837a10`) |
| `??S00/mod_config.xml` | `0x00837a10` - it both reads and writes |
| `??S00/mod_settings.xml` | **deleted** when the `.bin` is absent and the `.xml` is present - a live xml-to-binary migration |
| `??USR/config.xml` | the game options file; see [ui-settings.md](ui-settings.md) |
| `??SAV/world/*.bin`, `??SAV/*.xml` | the save payload |
| `tools_modding/*` | the two documentation generators - `0x007ec750` (`-build_lua_api_documentation_n_exit`) and `0x00579c50` (`-write_documentation_n_exit`) |
| `temptemp/*` | diagnostics **and** world save/load - see below |

The `mod_settings.xml` deletion is worth knowing about if you are reading a save directory
by hand. To be precise about the mechanism: the file is *existence-checked and deleted*, not
read and converted in the same place - the serialisation it belonged to is not in that
function.

## The `data.wak` archive

**The retail game ships an archive, and reads it at runtime.** A Steam install contains
`data/data.wak`, 42 467 033 bytes (`0x287FED9`), holding **14 745 files** - including all
1 032 files under the game's own `data/scripts/` directory, of which 952 are `.lua`. The
`data/` directory beside it is *not* the game data; it holds only 290 loose files. So the VFS
device list described above is normally serving most of `data/` out of the archive, not off
the disk.

That is worth knowing before you draw conclusions from a loose-file listing: an absent
`data/scripts/init.lua` on disk means nothing, because the file exists inside the archive.

### What ships loose, and why

Packing is a three-function chain behind the `-wizard_pak` flag: `0x0085e560` opens the
archive, `0x0085d240` writes the header, index and every blob, and `0x0085cef0` is the
recursive **file selector** - it walks `data/`, skips anything under five prefixes, and
prints `Not packing: ` followed by the path. The five prefixes live in a table at
`0x01152640`:

| excluded from packing | loose files in a retail install |
|---|---|
| `data/audio` | 26 |
| `data/fonts` | 83 |
| `data/translations` | 13 |
| `data/schemas` | 159 |
| `data/video` | 1 |

The counts are the proof: those five directories hold 282 loose files, and
**zero** of the archive's 14 745 entries are under any of them. It is not that they are
listed but unreferenced - they are absent from the index entirely.

The remaining loose files are not game data either:

- `data/scripts/gun/` holds three `*_generated.lua` files (`gun_generated.lua`,
  `gunaction_generated.lua`, `gunshoteffects_generated.lua`). They are **config-system default
  stubs** (`ConfigGun_Init` and friends), not tables derived from wand definitions. The archive
  holds a copy of each as well.
- `data/generated/material_icons/` holds two material-icon caches - but `data/generated/` is
  otherwise 628 packed entries, so it is a mixed shipped/runtime directory, not a cache dir.
- The rest are `data/data.wak` itself, `data/icon.bmp`, and
  `data/noita_engine_patcher_asm`.

### The format

Undocumented anywhere, so this was recovered from the shipped file and then checked against
it, and independently re-derived from the writer (`0x0085d240`). The header is 0x10 bytes -
four dwords, no more:

| offset | value in the shipped archive | meaning |
|---|---|---|
| `+0x00` | 0 | version, unused |
| `+0x04` | 14 745 | entry count |
| `+0x08` | `0x0C2CA3` (797 859) | offset where the file data begins |
| `+0x0C` | 0 | unused |

Then a variable-length index, 797 843 bytes, of entries that are simply concatenated:

```
u32 data_offset      absolute offset of this file's bytes
u32 plain_size       length of the file
u32 name_len         length of the name, no NUL
char name[name_len]  UTF-8, and always the full "data/..." path
```

(The dword at `+0x10`, immediately after the header, is the *first index entry's*
`data_offset`. It happens to equal the `index_end` field because entry 0's body starts where
the index ends - that coincidence is worth knowing about before you read it as a fifth header
field.)

From `0x0C2CA3` to end of file come the file bodies, concatenated, **plain and
uncompressed**, in index order. Each body is followed by exactly one pad byte - all 14 744 of
them are `0x00` - and the pad is *trailing*, not a per-file header, because the offsets
accumulate forward from `index_end`. So the stride from one `data_offset` to the next is
`plain_size + 1`.

Four independent checks all pass on the shipped archive:

| check | result |
|---|---|
| header entry count vs entries walked | 14 745 = 14 745 |
| index end vs the header's `index_end` field | `0x0C2CA3` = `0x0C2CA3` |
| every consecutive `data_offset` gap equals `plain_size + 1` | 14 744 of 14 744 |
| end of the last entry vs end of file | `0x287FED9` = `0x287FED9` |

And the bodies really are the files - reading `data/credits.txt` and `data/scripts/init.lua`
at their own offsets yields CRLF text with no bare LF or bare CR anywhere, byte-identical to
the same files in an unpacked tree. Note that there is no *loose* copy to compare against in
a retail install, for exactly the reason this section is about.

Entries are in **directory order, not sorted by name** - 176 of the 14 744 adjacent pairs are
out of order, and the grouping is the recursive walk's own output order - so a lookup cannot
binary-search; it has to walk. That is why `-wizard_pak` also dumps a plain name list to
`temptemp/data_wak_files.txt` so the archive can be diffed by eye. That file is a *pack-time*
artefact: it is not present in a normal install.

### Packing and unpacking

Both directions are behind command-line flags in the start-up function's exit-immediately
developer branch: `-wizard_pak` runs the packing chain above, `-wizard_unpak` runs the
extractor (`0x0085e7f0`), alongside `-clean_save`. The writer finishes by printing
`WizardPak - done`. All three branches jump straight to the function epilogue.

Unpacking is a supported user operation, and the game ships the batch file for it next to
the executable - 30 bytes, no trailing newline:

```
data_wak_unpack.bat
    noita.exe -wizard_unpak
    pause
```

Run it and you get a full loose `data/` tree, which is the easiest way to read or diff the
game's data. This is provably how the community trees were obtained: an unpacked tree checked
against the archive contains **all 14 745** entries plus exactly one extra file (a
pixel-scene splice output), and correctly lacks the five excluded prefixes - which
`-wizard_unpak` does not produce either. One caveat if you use such a tree: because the five
excluded directories are never unpacked, their absence from it tells you nothing about
whether they exist.

## `temptemp/`

A plain relative scratch directory at the game root. It is never a mount. It is **not** only
touched by diagnostics - ordinary world save and load write to it - so treat it as scratch
space rather than as a developer-only area. Twelve `temptemp/` paths exist in the binary:

| written by | path |
|---|---|
| `-wizard_pak` (`0x0085d240`) | `data_wak_files.txt` |
| world creation (`0x0087a900`) | `biome_wangs/`, `_biomes_all_wang.png`, `_biomes_all_wang2.png` |
| biome pathfinding (`0x008771a0`) | `paths/` |
| world-tree save and load (`0x00745830`) | `worldtree_saved.png`, `worldtree_loaded.png` |
| component system (`0x0057a060`) | `component_system_order.txt` |
| developer tools | `debug_settings.xml`, `world_big/`, `entities.txt`, `_particle_effect.xml` |

Deleting the directory is harmless - it is a scratch area and is recreated on demand - but the
premise is worth stating: the interesting-looking files in it are caches and dumps, and the
two `worldtree_*.png` in a stock install are written by normal play, not by a developer.

## For a reimplementation

The minimum that reproduces observed behaviour:

1. A list of devices, searched front to back, first hit wins. Put the mod device first.
2. A mount table of `(5-char prefix, root, directory)`, linearly matched, expanded by
   concatenating directory and `path[5:]`.
3. Lowercase the path before lookup - but keep the un-lowercased form for the mod-prefix
   match if you want `ModDoesFileExist` and `ModTextFileGetContent` to agree, which the
   original does *not* do.
4. Gate the Lua-facing read on extension and prefix, and blacklist the three mod descriptor
   filenames. AND the three checks; return `""`, not nil, when the gate rejects a path.
5. A one-entry-per-thread open stream keyed on the path, so repeated reads of the same path
   skip the device walk.
6. An archive device for `data/`, holding the bulk of the game data, so that a device-ordered
   list has both an archive-backed and a disk-backed source. `data/icon.bmp` is the worked
   example of a file both sources have.

## Undetermined

- **How the archive device orders against the *mod* devices.** The archive-vs-disk half of
  this question is settled: the archive device is inserted at `begin()` during start-up, so
  it precedes the disk device, and the only case where it matters is after
  `data_wak_unpack.bat` or in a development tree. Against mod devices it is not - mod devices
  are also inserted at `begin()`, and the mod-initialisation function re-runs on every save,
  so whichever insert happened last is first. Not traced.
- Whether the mod device matches its prefixes by longest or first-listed. The device list is
  ordered mod-first, so a file present in two enabled mods resolves to the one inserted
  first; the exact insertion order among several mods was not traced.
- The initial value of the user-data root global `0x01204ba0`, and which physical directory
  `root = 0` resolves to. It is set only by the two command-line flags and lives in BSS, so
  its load-time default is decided by start-up code outside the paths read here. The flag
  names say AppData; a Steam install is also consistent with the Steam userdata directory.
- The device vtables for `ModDiskFileDeviceCaching` and `WizardPakFileDevice`. Their RTTI type
  descriptors resolve but their complete-object-locators do not, so their slot layouts are
  inferred from the four slots the generic code calls rather than read directly. (The mod
  device's vtable has since been recovered at `0x01004df4` and matches the slot numbering
  above.)
