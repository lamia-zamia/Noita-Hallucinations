# Language XML files

Each language has one small XML file with its display name, fonts and a few UI offsets. The
shipped ones are `data/translations/<id>.xml` (`en`, `de`, `jp`, ...); a translation mod points
at its own with `translation_xml_path` in [`mod.xml`](mod-xml.md).

```xml
<Language
  key="fi-es"
  name="Finnish"
  font_default="data/fonts/font_pixel.xml"
  font_inventory_title="data/fonts/font_pixel_big.xml"
  font_important_message_title="data/fonts/font_pixel_huge.xml"
  font_world_space_message="data/fonts/font_pixel.xml"
  fonts_utf8="1"
  fonts_pixel_font="1"
  ui_wand_info_offset1="10"
  ui_wand_info_offset2="20"
  ui_action_info_offset2="10">
</Language>
```

All thirteen attributes are optional and read only from the root element; there are no
children. A missing attribute keeps the constructor value.

| attribute | type | default | notes |
|---|---|---|---|
| `key` | string | `""` | language id. The shipped files omit it: the game already holds an id for each built-in language and loads `data/translations/<id>.xml` into that record. A translation mod sets it |
| `name` | string | `""` | name in the language list |
| `font_default` | string | `""` | font file - an XML font or a generated `.bin` ([fonts.md](fonts.md)) |
| `font_inventory_title` | string | `""` | |
| `font_important_message_title` | string | `""` | |
| `font_world_space_message` | string | `""` | |
| `fonts_utf8` | bool | `0` | every shipped language sets `1` |
| `fonts_pixel_font` | bool | `0` | set by the Latin and Cyrillic languages; `jp`, `ko` and `zh-cn` (generated Noto fonts) omit it |
| `fonts_dpi` | float | **`1`** | no shipped file sets it |
| `ui_wand_info_offset1` | float | `0` | per-language UI offsets |
| `ui_wand_info_offset2` | float | `0` | |
| `ui_action_info_offset2` | float | `0` | |
| `ui_configurecontrols_offset2` | float | `0` | |

The text itself is not here: it comes from `data/translations/common.csv` (and, for a translation
mod, `translation_csv_path`). What each UI offset moves on screen was not traced.

## Evidence

| claim | address |
|---|---|
| reader: the thirteen attributes, their types and record offsets `+0x00`..`+0xa4` | `0x00848430` |
| constructor: strings empty, bools 0, `fonts_dpi` 1.0, offsets 0 | `0x0084a320` |
| built-in languages: `"data/translations/" + id + ".xml"` read into an existing record | `0x0084a700` |
| registering an extra language (`Text_RegisterLanguage() error - Missing translation file: `) | `0x0084a510` |
