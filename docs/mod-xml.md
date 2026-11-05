# `mod.xml`

Every mod folder carries a `mod.xml`. The game reads exactly twelve attributes from it, all
optional, all from the root element. This page is the complete list, with the defaults the
executable actually uses.

## What the parser does

- **The root element's name is not checked.** The reader is handed whatever the root element
  is. Every known mod uses `<Mod>`; keep doing so.
- **Only attributes are read.** Child elements and text content are ignored.
- **Every attribute is optional.** Each one is read only if present; otherwise the field keeps
  its constructor value. The constructor zero-initialises the whole record, so a missing string
  is `""` and a missing flag is `0`. That includes `name` and `description`.
- **Unknown attributes are silently ignored.** A misspelt attribute name produces no error and
  no log line.
- **Booleans are `0` or `1`.** The value goes through a C++ `istream >> bool` without
  `boolalpha`. `"1"` is true, `"0"` is false, and **`"true"` is false** (the extraction fails
  and leaves `false`).

Both code paths that load a `mod.xml` call this same reader.

## The attributes

| attribute | type | default | effect |
|---|---|---|---|
| `name` | string | `""` | name shown in the mod list |
| `description` | string | `""` | description shown in the mod list |
| `request_no_api_restrictions` | bool | `0` | the mod's Lua state gets the full LuaJIT library set (`package`/`require`, `io`, `os`, `debug`, FFI) instead of the restricted six - see [lua-execution.md](lua-execution.md). The player must allow unsafe mods, and such mods do not work through the Steam Workshop |
| `is_game_mode` | bool | `0` | the mod is a game mode: started from the new-game menu rather than enabled in the mod list |
| `game_mode_supports_save_slots` | bool | `0` | a game mode supports multiple save slots |
| `ui_newgame_name` | string | `""` | game mode title in the new-game menu; `$key` translation keys work |
| `ui_newgame_description` | string | `""` | game mode description in the new-game menu; `$key` works |
| `ui_newgame_gfx_banner_bg` | string | `""` | path to the banner background image, e.g. `mods/<id>/files/menu_banner_background.png` |
| `ui_newgame_gfx_banner_fg` | string | `""` | path to the banner overlay image |
| `is_translation` | bool | `0` | the mod stops being a normal mod and is listed under Language in the options menu |
| `translation_xml_path` | string | `""` | path to the `<Language>` file: language key, display name, fonts, UI offsets ([language-xml.md](language-xml.md)) |
| `translation_csv_path` | string | `""` | path to the translation CSV |

Record layout, for anyone reading memory (`std::string` is 0x18 bytes in this build):

| offset | field |
|---|---|
| `+0x00` | `name` |
| `+0x18` | `description` |
| `+0x30` | `ui_newgame_name` |
| `+0x48` | `ui_newgame_description` |
| `+0x60` | `ui_newgame_gfx_banner_bg` |
| `+0x78` | `ui_newgame_gfx_banner_fg` |
| `+0x90` | `request_no_api_restrictions` (byte) |
| `+0x91` | `is_game_mode` (byte) |
| `+0x92` | `game_mode_supports_save_slots` (byte) |
| `+0x93` | `is_translation` (byte) |
| `+0x94` | `translation_xml_path` |
| `+0xac` | `translation_csv_path` |

The record continues past `+0xc4` with fields that are not filled from `mod.xml`.

## Attributes that do nothing here

A survey of 100 real `mod.xml` files found these, none of which `noita.exe` reads from
`mod.xml`:

| attribute | where it really belongs |
|---|---|
| `tags`, `dont_upload_folders`, `dont_upload_files` | `workshop.xml`, read by the Workshop uploader in `noita_dev.exe` (see `tools_modding/READ_ME_FIRST.txt`). Neither `dont_upload_*` string exists in `noita.exe` at all |
| `download_link`, `version`, `author`, `icon` | nowhere - no executable contains a reader for them. Harmless as notes for humans or third-party tools |

There is no `directory` attribute either; the string does not occur in the executable.

## The other two `<Mod>` files

`workshop.xml` and `compatibility.xml` sit next to `mod.xml` and also use a `<Mod>` root, but are
read by different code:

| file | attributes | read by |
|---|---|---|
| `workshop.xml` | `name`, `description`, `tags`, `dont_upload_folders`, `dont_upload_files` | the Workshop uploader in `noita_dev.exe` |
| `compatibility.xml` | `_format_version`, `version_built_with` (integers) | `0x00831ad0`, called from the mod scan `0x008365e0` |

A few mods write `_forversion` instead of `_format_version`; it is not read.

## Evidence

| claim | address |
|---|---|
| the reader: twelve attribute names, offsets, `has-attribute` guard on each | `0x00832360` |
| record constructor, zero-initialises everything | `0x0080a060` |
| local scan: `"\mod.xml"`, `"\mod_id.txt"`, `"\compatibility.xml"` per mod, reader called on the document root | `0x008365e0` |
| second load path (`"\mod.xml"` built from a different id list), same reader | `0x0095e4d0` |
| string to bool: `istream >> bool` | `0x00446280` (via `0x00442280`) |
| attribute strings | `0x00fe7740`-`0x00fe7814` |
