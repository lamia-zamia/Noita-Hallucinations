# Sprite stains

Noita's sprite-stain system is separate from the material-stain settings used by liquids. An entity receives visible sprite stains through a `SpriteStainsComponent` attached to a sprite the engine can resolve and map.

## Minimum entity setup

The engine needs:

- An enabled `SpriteStainsComponent`.
- A valid `SpriteComponent` selected by the stain component's `sprite_id`. This is a zero-based index into the entity's ordered sprite-component list; it defaults to `0`.
- A loaded sprite resource with frame data and a UV map that the stain system can load.

For one sprite, the component can be explicit:

```xml
<SpriteStainsComponent _enabled="1" sprite_id="0" />
```

`SpriteAnimatorComponent`, `HitboxComponent`, `CharacterDataComponent`, `StatusEffectDataComponent`, and the `mortal`/`hittable` tags are not prerequisites for building the sprite-stain map. The stain initializer resolves the selected `SpriteComponent` directly. It exits when that sprite resource or its frame data is unavailable, and stains cannot be added until the component's stain texture has initialized (`0x00c22080`, `0x00c23290`).

Each `SpriteStainsComponent` targets one sprite index. If an entity has two sprites and both should stain, give it two stain components, with `sprite_id="0"` and `sprite_id="1"`. Vanilla meatmaggot follows this pattern for its nine sprites (indices 0–8). The index refers to the final entity's ordered SpriteComponents, including inherited components; it is not an ID to guess. The engine returns no sprite for a negative or out-of-range index, or when the selected list entry is not active (`0x00c50680`).

The Lua function `EntityAddRandomStains(entity, material_type, amount)` is a useful direct check: it is a no-op when the entity has no `SpriteStainsComponent` (`0x007b52d0`). Use a valid material ID, such as `CellFactory_GetType("water")`, and a positive amount. The amount is a number of stain-cell placement attempts, not a pixel size. The selected sprite needs opaque pixels for the stain routine to land a cell (`0x00c23290`).

## UV-map location and filename

The game looks for generated sprite UV maps under:

```text
data/generated/sprite_uv_maps/
```

The map filename is based on the full virtual path of the sprite's underlying PNG: replace each `/` with `.` and retain the `.png` extension. For example:

```text
Sprite PNG:  mods/example/files/whatever.png
UV map:     data/generated/sprite_uv_maps/mods.example.files.whatever.png
```

Use the PNG path from the sprite definition's `filename`, not the entity XML path or the `SpriteComponent.image_file` path when that points to a sprite XML. The shipped player sprite is an example: its sprite definition names `data/enemies_gfx/player.png`, and the corresponding generated map is `data.enemies_gfx.player.png`.

The init-only `ModDevGenerateSpriteUVsForDirectory(directory_path, override_existing)` API generates maps for a directory. Its path must start with `mods/YOUR_MOD_HERE/` or `data/`; it is available during initialization in `noita_dev.exe`. Pass `true` for `override_existing` when an old map must be regenerated (`0x008317d0`).

## If the entity still receives no stains

Check these in order:

1. Confirm `_enabled` is not `0`.
2. Count the final entity's SpriteComponents, including those inherited through `<Base>`, and confirm `sprite_id` selects the intended entry. An extra decorative or emissive sprite can shift the index. Values `0` and `1` only select the first and second entries; to stain both, use a stain component for each index.
3. Confirm the selected `SpriteComponent` resolves to the sprite XML and that XML's `filename` points to the PNG whose UV map was generated.
4. Confirm the UV-map file exists at the derived path and matches the current PNG. Regenerate it with override enabled after changing the source image.
5. Test with `EntityAddRandomStains` and a known stain-capable material. If that works but contact with a particular liquid does not, the remaining issue is the liquid's stain configuration, not the entity's sprite setup.

The shader samples the UV map to find the reference-pose stain texture (`data/shaders/sprite_stains.frag`). The material XML field named `stainable` is a material property; it does not make an entity's sprite stainable.
