# Font files

Noita's bitmap fonts (`data/fonts/*.xml`, and any font a mod passes to `GuiText`-style
functions) are read by one C++ loader. This page lists everything it reads.

## XML or binary: decided by the extension

The loader takes the text after the last `.` in the path. `xml` is parsed as described below;
`bin` goes to a separate binary loader (the `data/fonts/generated/*.bin` files the font builder
produces). Any other extension leaves the font empty. Loaded fonts are cached by path.

## The XML format

```xml
<FontData color_r="1" color_g="0.8" color_b="0.2" color_a="1">
  <Texture>data/fonts/font_pixel.png</Texture>
  <LineHeight>10</LineHeight>
  <CharSpace>-1</CharSpace>
  <WordSpace>4</WordSpace>
  <QuadChar id="65" rect_x="0" rect_y="0" rect_w="5" rect_h="9" offset_x="0" offset_y="0" width="5"/>
  <Char id="66" rect_x="6" rect_y="0" rect_w="5" rect_h="9"/>
</FontData>
```

- **The root element's name is not checked.** Most files use `<FontData>`; the vanilla
  `ubuntu_condensed_*.xml` files use `<Font>`. Both work.
- **The four settings are child elements, not attributes**, parsed with `istream >> float` (or
  read as a string for `Texture`). Surrounding whitespace and newlines are fine, which is why
  vanilla files put each value on its own line. A missing element leaves the value at 0 (or
  empty for `Texture`).
- **Glyphs are found by walking every child** and matching its name against `Char` and
  `QuadChar`; anything else is ignored. Order does not matter.
- **The colour is four attributes on the root**, floats, read only if present.

| item | kind | type | default | meaning |
|---|---|---|---|---|
| `Texture` | element | string | `""` | glyph atlas image |
| `LineHeight` | element | float | `0` | line height in pixels |
| `CharSpace` | element | float | `0` | added after every glyph; negative tightens |
| `WordSpace` | element | float | `0` | width of a space |
| `color_r`, `color_g`, `color_b` | root attribute | float | `0` | font colour, copied into the text renderer when the font is applied |
| `color_a` | root attribute | float | `1` | colour alpha, clamped to `0..1` when applied |

### Glyphs

| attribute | `QuadChar` | `Char` | type | meaning |
|---|---|---|---|---|
| `id` | yes | yes | int | Unicode code point |
| `rect_x`, `rect_y`, `rect_w`, `rect_h` | yes | yes | float | source rectangle in the texture |
| `offset_x`, `offset_y` | yes | - | float | draw offset |
| `width` | yes | - | float | advance before `CharSpace` |

Every glyph attribute is optional and defaults to 0.

`Char` is the short form. For a new code point it creates a glyph with `width = rect_w` and both
offsets 0. If the code point already exists, `Char` replaces only the rectangle and keeps the
existing offsets and width. `QuadChar` always replaces the whole glyph. Either way the last
definition of an `id` wins.

The font also records the largest `-offset_y` over all its glyphs.

## Evidence

| claim | address |
|---|---|
| loader: extension switch `xml`/`bin`, cache by path | `0x00db18c0` (strings `0x01004c4c`, `0x01004c58`) |
| XML entry: parse file, hand root to the reader | `0x00df0950` |
| reader: `Texture`/`LineHeight`/`CharSpace`/`WordSpace`, `color_*`, child walk | `0x00da4000` (source file `poro/source/game_utils/font/cfont.cpp`) |
| `QuadChar` reader (`id` int, the rest float) | `0x00da3520` |
| `Char` reader | `0x00da2c30`; insert with `width = rect_w` at `0x00da2b00` |
| reset before load: metrics and texture zeroed | `0x00da20b0` |
| constructor: colour `0,0,0,1` | `0x00da28e0` (`CFont::vftable`) |
| colour copied, alpha clamped to `min(a, 1.0)`, `>= 0` | `0x00db18c0` (`1.0` at `0x01053780`) |
