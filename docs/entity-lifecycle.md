# Entity lifetime: kills, children and CameraBoundComponent

How an entity dies in Noita, when it actually disappears, what happens to its children, and what
`CameraBoundComponent` does to entities near the edge of the loaded world. Addresses are for the
retail `noita.exe` (imagebase `0x00400000`).

> **Status.** Read from the executable, then checked with one in-game test run (retail build,
> November 2026). Everything the test covered behaved as described; the parts it did not cover are
> marked.

## The short version

- `EntityKill` does not remove anything. It **marks** the entity, and every entity that is its
  child *at that moment*, for removal.
- A marked entity is still "alive" to Lua: `EntityGetIsAlive`, `EntityHasTag`, `EntityGetTransform`
  and `EntityAddChild` all keep working on it until the engine physically removes it.
- The engine removes a marked entity during the next system update that touches one of its
  components. Removal **does not kill children**. It detaches them.
- So a child added to an already-marked entity survives the parent as an orphan: a root entity,
  frozen at its last position.
- `CameraBoundComponent` can mark an entity **inside `EntityLoad`** (the `max_count` rule), so the
  id `EntityLoad` returns can already be doomed. In the test, `EntityGetIsAlive` and `EntityHasTag`
  both returned true for such an id, and it was gone by `OnWorldPostUpdate`.
- **Removing a `SpriteComponent` from a script that runs on `execute_on_added` leaves its image
  drawn at world (0,0), with no entity behind it.** See [the sprite leak](#the-00-sprite-leak).
  Polymorph triggers it for any child entity whose on-added script edits its sprites.
- A "frozen" `CameraBoundComponent` entity is not saved. The engine remembers its XML file name
  and position, and later loads that file again as a brand-new entity.

## Killing is a mark

Every entity has a removal counter at offset `+0x0C` (0 = alive). `EntityKill` (`0x0078F6C0`) looks
the id up and calls `0x0044DF60`, which raises the counter to at least 1 and then calls itself on
every entity in the child list (`+0x90`). `CameraBoundComponent` kills go through the same
function. Children added later are not in the list yet, so they are not marked.

`EntityGetIsAlive` (`0x0078F9A0`) only asks whether the id is still in the entity table
(`0x0056EBA0`). It never reads the counter. There is no Lua function that reports "marked for
removal".

`EntityAddChild` (`0x00793150`) checks that both ids exist and that the child has no parent yet
(`+0x94` is empty). It does not check whether the parent is marked.

## When the entity is actually removed

A gameplay frame runs `OnWorldPreUpdate`, the world simulation, every component system in sorted
order, then `OnWorldPostUpdate` ([details](lua-execution.md#when-each-callback-fires-and-in-what-order)).
Systems are sorted by priority, then name. Priorities that matter here:

| system | priority |
|---|---|
| `InheritTransformSystem` | 2 |
| `CameraBoundSystem` | 100 |
| `LuaSystem` | 120 |
| `SpriteSystem` | 500 |

Removal happens in two places:

1. **Inside each system's pass.** The shared component loop (`0x00C60490`, and the same code in
   systems with their own loop such as `SpriteSystem` `0x00C52B30`) checks every component's owner
   after updating it. An owner with counter 1 is set to 2 and collected; after the loop, every
   collected entity is destroyed (`0x0056E8D0`). The component does not have to be enabled for
   this; only the update is skipped for disabled components.
2. **After all systems.** `0x0057A4A0` looks through the entity table and destroys the first
   marked entity it finds: **one per frame**. This is what removes entities that have no
   component any system loops over.

"First" means first in the engine's internal slot table, not lowest id: slots of destroyed
entities are reused. In the test, two component-less entities killed in the same callback were
removed one frame apart, and the one with the higher id went first.

So an entity killed in `OnWorldPreUpdate` is normally gone by `OnWorldPostUpdate` of the same
frame. An entity killed *after* the last system that holds one of its components has run (for
example in `OnWorldPostUpdate`, or by a system late in the order) is still present, and still
"alive", in the next frame's `OnWorldPreUpdate`. It is removed during that frame's update.

## What removal does to children

`0x0056E8D0` sends an "entity about to be destroyed" event, removes the entity's components from
every component buffer (each system's remove handler runs), sends "entity destroyed", and calls the
entity destructor `0x0056CF80`. The destructor:

- removes the entity from its parent's child list, and
- **sets the parent pointer of each child to none**, then frees the list.

It does not kill the children. Only the recursive mark at kill time does that. A child that was
attached after the kill keeps running as a root entity.

`InheritTransformComponent` (`0x00B497F0`) does nothing when its entity has no parent, so an
orphan stays at the last position its parent gave it. It does not jump to (0,0).

### Reproduction

```lua
-- in OnWorldPreUpdate
local parent = EntityLoad("data/entities/animals/zombie.xml", x, y)
EntityKill(parent)
local child = EntityLoad("some_icon.xml", x, y)
EntityAddChild(parent, child)   -- succeeds, no error
-- OnWorldPostUpdate: parent is gone, child is alive with EntityGetParent(child) == 0
```

Swapping the two calls (attach first, then kill) kills both.

Tested in game, both with `EntityKill` and with a `max_count` kill. The orphan's own Lua script
already saw `EntityGetParent == 0` during the same frame's `LuaSystem` pass, which also confirms
that `CameraBoundSystem` runs before `LuaSystem`.

`GameGetFrameNum()` is one higher in `OnWorldPostUpdate` than in the `OnWorldPreUpdate` of the same
frame: the counter advances during the update. Keep that in mind when matching pre and post logs.

## CameraBoundComponent

Field layout of the component (offsets from the component start):

| field | offset | meaning, as implemented |
|---|---|---|
| `enabled` | `+0x48` | the distance and border checks below are skipped when false; `max_count` still applies |
| `distance` | `+0x4C` | kill when further than this from the camera centre |
| `distance_border` | `+0x50` | a frozen entity respawns within `distance - distance_border` |
| `max_count` | `+0x54` | most live entities loaded from the same file |
| `freeze_on_distance_kill` | `+0x58` | freeze before a `distance` kill |
| `freeze_on_max_count_kill` | `+0x59` | freeze before a `max_count` kill |

Vanilla enemies typically use `distance="160000"` (never reached) and `max_count` 20 or 30.

### Every frame: distance and world border (`0x00A5BD10`)

For each entity with an enabled `CameraBoundComponent`, during `CameraBoundSystem`'s pass:

- If the entity is **outside a world rectangle** (integer bounds read from an object owned by the
  camera; most likely the streamed-in area, not yet confirmed): freeze it and kill it. This
  happens **whatever `freeze_on_distance_kill` says**.
- Else, if it is further than `distance` from the camera centre: kill it, freezing first only if
  `freeze_on_distance_kill` is set.
- Else nothing.

Because `CameraBoundSystem`'s own loop then destroys what it marked, an entity removed this way is
gone before `LuaSystem` and `SpriteSystem` run in that frame. Children attached before this point
are marked with it and die too.

### On load: `max_count` (`0x00A5BBE0`)

This runs from the "component added" event, which `EntityLoad` sends before it returns (see
[below](#execute_on_added-runs-inside-entityload)). The engine keeps a list of live entities per
entity file (up to 4096 different files; beyond that it logs `Too many unique entities` and stops
counting). When the list is longer than `max_count`, it sorts the list by distance from the camera
centre and kills every entry past the first `max_count`, freezing each first if
`freeze_on_max_count_kill` is set.

Consequences:

- The entity just loaded can be killed before `EntityLoad` returns, if it is the furthest one.
- Loading one entity can kill **another** entity of the same file somewhere else in the world.
- The check is skipped while frozen entities are being respawned.

### What "freeze" stores (`0x00A58FE0`)

Only a small record: the entity's file, its position, `max_count` and
`(distance - distance_border)²`. It is filed under the 512 x 512 chunk the position falls in.
Positions beyond ±1,024,000 are not recorded at all.

Nothing else survives: not health, not component values changed at runtime, not components added
from Lua, not children added with `EntityAddChild`. This is also why `EntityGetFilename` is
documented as unreliable after "the world streaming system": the respawned entity is a new load.

### Respawn (`0x00A5BE10` → `0x00A590F0`)

At the start of `CameraBoundSystem`'s pass every frame, the records in the 3 x 3 chunks around the
camera centre are checked. A record is loaded again from its file (a new entity with a new id) when
all of these hold:

- fewer than `max_count` entities of that file are alive,
- the position is inside the world rectangle,
- its distance from the camera centre is below `distance - distance_border`, and
- above `0.6 x max(camera width, camera height)`, so it is off screen.

The respawned entity is checked by the same pass straight away. `execute_on_added` scripts run
again, because it is a fresh load.

Tested in game with an entity spawned outside its `distance` but inside its respawn radius: it was
frozen and respawned once per frame, each time with a new id, with no children and without a
component that had been added from Lua before the first freeze. The child attached before the
first freeze died with it.

## `execute_on_added` runs inside `EntityLoad`

`EntityLoad` builds all components and child entities first, then sends one "component added"
event per component (`0x00574C60`). Events are delivered immediately to every listener
(`0x00578550`); nothing is queued. `LuaSystem`'s listener (`0x00BA7C20`) runs the script at once if
`execute_on_added` is set and the component is enabled. The run (`0x00BA6E70`) executes the
script's main body first and then, on this first run only and if `call_init_function` is set,
calls `init(entity_id)`.

So such a script runs before `EntityLoad` returns, and before the caller's `EntityAddChild`.
At that moment the new entity is its own root. Confirmed in game: the script's first run, and then
`init()`, were logged between the `EntityLoad` call and the line after it.

## Sprites

When a `SpriteComponent` is added, `0x00C1B320` creates the render object and, for an ordinary
world sprite, registers it in the world's sprite list and remembers the handle. When the component
is removed, `0x00C1B7B0` unregisters it; changing the image re-creates it through `0x00C1AF80`,
which unregisters the old one first. The sprite's position is written only by `SpriteSystem`'s
per-frame update (`0x00C1AB60`), from the entity's position plus the offsets, and only for enabled
components.

Apart from [the sprite leak](#the-00-sprite-leak) below, no path was found that leaves an ordinary
world sprite drawn after its component is gone. The test tried: adding a sprite and killing the entity in the same callback (with and without a
`CameraBoundComponent`, so removed before and after `SpriteSystem` could position it); loading an
entity with a sprite in its XML and killing it at once; adding a sprite to an entity already marked
by `max_count` or by `EntityKill`; adding and removing a sprite in one callback; changing a sprite's
image, refreshing it and killing the entity. None left anything drawn, at the entity's position or
at (0,0). The only icons on screen were the ones on entities still alive (a control and the two
orphans), and they disappeared as soon as those entities were killed.

Emissive, UI-parented and fog-of-war-hole sprites are registered in other lists that this handle
does not cover; whether those are cleaned up is not established.

When looking for entities at (0,0), expect plenty: `EntityGetInRadius(0, 0, r)` returns every
child entity whose own transform is (0,0), such as the player's inventory containers and the items
in them.

## The (0,0) sprite leak

Confirmed in game. Load an entity like this:

```xml
<Entity>
	<LuaComponent script_source_file="remove_my_sprites.lua" execute_on_added="1" execute_every_n_frame="-1" />
	<SpriteComponent image_file="data/ui_gfx/perk_icons/bounce.png" />
</Entity>
```

```lua
-- remove_my_sprites.lua
local id = GetUpdatedEntityID()
for _, c in ipairs(EntityGetComponent(id, "SpriteComponent") or {}) do
	EntityRemoveComponent(id, c)
end
```

The entity ends up with no `SpriteComponent`, but the bounce icon is drawn at world (0,0) and
stays there. No entity owns it, so nothing in Lua can find or remove it. Putting the
`SpriteComponent` before the `LuaComponent` in the XML gives the same result.

Why:

1. `EntityLoad` creates every component first, then sends the "component added" event to each one
   in turn ([above](#execute_on_added-runs-inside-entityload)). The script runs during that loop.
   The `LuaComponent` gets its event before the `SpriteComponent`, whatever the XML order.
2. `EntityRemoveComponent` (`0x00790520` → `0x008E5140`) tells the systems at once that the component
   is gone, but only queues the component object for deletion at the end of the frame. The sprite
   system has nothing to clean up yet (`0x00C1B7B0`), because the sprite has not been created.
3. The event loop carries on and delivers "added" to the removed component. The sprite system
   creates its image and puts it in the world's draw list (`0x00C1B320`).
4. The component is no longer in the sprite system's component list, so the per-frame update that
   sets the image's position (`0x00C1AB60`) never runs for it. The image stays where it was
   created, at (0,0). The remove step has already happened, so nothing ever takes it out of the
   draw list.

The same thing happens when the engine rebuilds a saved entity, which it does at the end of
polymorph (below). Then the rebuilt children already have their parent when their on-added
scripts run, unlike a fresh `EntityLoad`, where the new entity is still its own root.

### Polymorph

Polymorph saves the target, children included, into the effect (`0x00B041E0` → `0x00A59D50`,
stored in the `GameEffectComponent`'s `mSerializedData`) and kills it. When the effect ends
(`0x00B015D0`), a new entity is built from that copy at the polymorphed creature's position
(`0x00A5A190`). Per entity, all components are created, then each gets "added", then the children
are created, attached to the parent, and built the same way.

So after polymorph:

- the creature and every child are new entities with new ids;
- every `execute_on_added` script runs again, and in a child it already sees its parent;
- a child whose on-added script removes `SpriteComponent`s leaks those images at (0,0). In the
  test, a deer with such a child was polymorphed: afterwards the child had no sprite, nothing was
  drawn above the deer, and the child's icon sat at (0,0).

### Avoiding it

Do not remove a `SpriteComponent` during the run that `execute_on_added` triggers. Either let
that run return early (check something that is only true on later runs), or keep the components
and change them in place: set `image_file` and the offsets, call `EntityRefreshSprite`, and hide
spare ones with `EntitySetComponentIsEnabled(entity, comp, false)`.

## For mods that decorate new entities

A common pattern scans new entity ids in `OnWorldPreUpdate` and attaches a child to each. Given the
above:

- An id can be marked and still pass every check: it was killed after its last system ran in the
  previous frame, or killed by `max_count` inside an `EntityLoad` that your own code (or another
  mod) just made. Attaching a child to it produces an orphan next update.
- There is no Lua way to see the mark. A child can detect its orphaning itself: on its own first
  update, `EntityGetRootEntity(self) == self` means the parent is gone.
- Loading more copies of an enemy than its `max_count` (clone or summon effects) kills the copies
  furthest from the camera, possibly ones that are already decorated.
- Enemies that wander out of the loaded area and come back are new entities with new ids. Anything
  attached to the old one is lost, and a mod that decorates by id will decorate the new one again.

## Open questions

- What the "world rectangle" is exactly (streamed area, or something else).
- Whether a leaked (0,0) image survives starting a New Game.
- Whether the same leak exists for other drawn components (particle emitters, lights, text sprites).
