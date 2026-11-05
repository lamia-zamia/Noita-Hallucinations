# Sprite XML files

Animated sprites (`data/enemies_gfx/*.xml`, `data/items_gfx/*.xml`, and any `image_file` a
`SpriteComponent` points at that ends in `.xml`) are read by one C++ reader. This page lists
everything it reads, with the defaults it actually uses.

## Rules that apply everywhere

- **Every attribute is optional.** Each one is read only if present; otherwise the field keeps its
  constructor value, listed below.
- **Unknown attributes are ignored silently.** In particular `emissive`, `z_index` and
  `rect_animation` are `SpriteComponent` fields; on a sprite file or `<RectAnimation>` they do
  nothing.
- Booleans are `0`/`1`. Numbers are plain numbers; a unit suffix such as `frame_wait="0.05s"` is
  not part of the format.

## Structure

```xml
<Sprite filename="data/enemies_gfx/foo.png" offset_x="8" offset_y="16" default_animation="stand">
  <RectAnimation name="stand" frame_count="4" frame_width="16" frame_height="20"
                 frame_wait="0.1" pos_x="0" pos_y="0">
    <Event name="step" frame="2" probability="1" max_distance="500"/>
    <ChildAnimations sprite_name="arm" frame="0" position_x="3" position_y="4"/>
  </RectAnimation>
  <Hotspot name="hand" color="ff0000"/>
  <Sprite name="arm" filename="..."> ... </Sprite>
</Sprite>
```

`<Sprite>` reads three repeated children - `RectAnimation`, `Sprite` (nested child sprites, same
format) and `Hotspot`. `<RectAnimation>` reads two - `ChildAnimations` and `Event`.

### `<Sprite>`

| attribute | type | default |
|---|---|---|
| `name` | string | `""` |
| `filename` | string | `""` |
| `hotspots_filename` | string | `""` |
| `offset_x`, `offset_y` | float | `0` |
| `scale_x`, `scale_y` | float | `1` |
| `position_x`, `position_y` | float | `0` |
| `angle` | float | `0` |
| `default_animation` | string | `""` |
| `color_r`, `color_g`, `color_b`, `color_a` | float | `1` |

### `<RectAnimation>`

| attribute | type | default | note |
|---|---|---|---|
| `name` | string | `"unknown"` | |
| `frame_count` | int | `0` | |
| `frame_width`, `frame_height` | int | `0` | |
| `frames_per_row` | int | **`9999`** | i.e. "all frames on one row" |
| `shrink_by_one_pixel` | bool | `0` | |
| `pos_x`, `pos_y` | int | `0` | pixel position of the first frame |
| `frame_wait` | float | `0` | seconds per frame |
| `has_offset` | bool | `0` | use this animation's own `offset_x`/`offset_y` |
| `offset_x`, `offset_y` | float | `0` | |
| `loop` | bool | **`1`** | animations loop unless they say `loop="0"` |
| `next_animation` | string | `""` | |

### `<Event>` (inside `<RectAnimation>`)

| attribute | type | default |
|---|---|---|
| `name` | string | `""` |
| `frame` | int | `0` |
| `probability` | float | `1` |
| `max_distance` | float | `500` |
| `on_finished` | bool | `0` |
| `check_physics_material` | bool | `0` |

### `<ChildAnimations>` (inside `<RectAnimation>`)

| attribute | type | default |
|---|---|---|
| `frame` | int | `0` |
| `sprite_name` | string | `"unknown"` |
| `position_x`, `position_y` | float | `0` |
| `rotation` | float | `0` |
| `alpha` | float | `1` |
| `scale_x`, `scale_y` | float | `1` |

### `<Hotspot>`

| attribute | type | default |
|---|---|---|
| `name` | string | `""` |
| `color` | hex colour | - |

`color` is read with `std::hex` into a 32-bit value and its red and blue bytes are then swapped, so
any 1-8 hex digits are accepted (`ff0000` and `000000ff` both occur in shipped files).

## Evidence

| claim | address |
|---|---|
| `<Sprite>` reader: attributes, children `RectAnimation`, `Sprite`, `Hotspot` | `0x00daba20` |
| `<Sprite>` defaults (scale 1, colour 1, strings empty) | constructor `0x00dab5b0`, reset `0x00dab930` |
| `<RectAnimation>` reader: attributes, children `ChildAnimations`, `Event` | `0x00daf6e0` |
| `<RectAnimation>` defaults: `"unknown"`, `frames_per_row` 9999, `loop` **1**, `has_offset` 0 | constructor `0x00da80a0` (`mov dword [esi+0x24],0x270f`; `mov byte [esi+0x44],1`; `mov byte [esi+0x38],0`) |
| `<Event>` reader | `0x00da7300` |
| `<Event>` defaults: `probability` 1.0, `max_distance` 500.0 | list reader `0x00de61e0` |
| `<ChildAnimations>` reader / defaults | `0x00da6870` / `0x00da6810` |
| `<Hotspot>` colour: `stream >> std::hex >> uint`, R/B swap | `0x00da7b60` -> `0x00da7f60` -> `0x00da7e70` |
