# `weather_config.xml`

`data/global/weather_config.xml` tunes the sky background - storm clouds, the parallax mountain and
cloud layers, the sky gradient and the sunset line. The game loads that one path (`0x00888c80`)
into a `ConfigParallaxBackground` record.

## The attributes

All sixteen are optional attributes on the root `<WeatherConfig>`. Defaults are the constructor
values (`0x008835f0`); the shipped file restates them.

| attribute | type | default |
|---|---|---|
| `debug` | bool | `0` |
| `init_randomize` | bool | **`1`** |
| `storm_clouds_pos_y_0` | float | `130` |
| `storm_clouds_pos_y_1` | float | `40` |
| `storm_alpha_0` | float | `0` |
| `storm_alpha_1` | float | `0.6` |
| `parallax_offset_y` | float | `50` |
| `mountains_01_offset_y` | float | `0` |
| `mountains_02_offset_y` | float | `0` |
| `clouds_01_offset_y_min` | float | `0` |
| `clouds_01_offset_y_max` | float | `0` |
| `clouds_02_offset_y_min` | float | `0` |
| `clouds_02_offset_y_max` | float | `0` |
| `gradient_sky_alpha` | float | `0` |
| `sunset_alpha` | float | `1` |
| `cloud_coeff` | float | `0.001667` (`1/600`) |

What each value does on screen was not traced; the names are the in-source field names.

## Attributes that do nothing

The shipped file also sets `autoplay`, `speed`, `storm_value`, `time` and `windspeed`. None of them
is a field of this record: `autoplay`, `storm_value` and `windspeed` do not occur anywhere in the
executable, and `speed` and `time` occur only as unrelated strings. They look like a snapshot of
runtime weather state saved into the file.

`data/global/test_weather.xml` (sun and star settings) is not loaded by anything, and none of its
eighteen attribute names exists in the executable.

## Evidence

| claim | address |
|---|---|
| loads `data/global/weather_config.xml` as `WeatherConfig`, plus the `data/weather_gfx/` images | `0x00888c80` |
| `ConfigParallaxBackground` constructor and field registration (16 names) | `0x008835f0` |
| defaults: word `0x0100` at `+0x04` (`debug` 0, `init_randomize` 1), floats from `+0x08` in declaration order | `0x008835f0` |
