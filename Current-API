# Luanti Fork

> A high-performance Luanti/Minetest fork built for Android-first gameplay, cinematic modding, and modern engine capabilities.

---

## What's Different

This fork is not a balance patch or a content mod â€” it is a deep engine rework targeting four areas: **rendering performance**, **animation quality**, **mobile-first UX**, and **mod developer power**. It ships with a fully rewritten physics model, a bone transform system, glTF multi-clip animation, an Android HTML overlay layer, and a server-side world management API.

---

## Feature Overview

| Area | What's New |
|---|---|
| Physics | Honest gravity, snappy movement, lava physics, ladder overhaul |
| Animation | glTF multi-clip, smoothstep blending, bone API, animation events |
| Camera | Server-controlled FOV, tilt, free-look, smooth orientation sync |
| Android | HTMLView overlays, headless workers, shared memory IPC |
| Fog | Volumetric layers, height density, biome atmosphere, turbulence |
| Networking | Look direction packet separated from position (no more teleport freeze) |
| World API | Server-side world creation, switching, and synchronisation |
| Accessibility | Sprint toggle, auto-climb, controlled ladder descent |

---

## Physics Model

The physics have been rebuilt around a "snappy and heavy" feel inspired by high-performance mobile voxel engines.

**Key changes from upstream:**

- **Gravity is 32.0 nodes/sÂ²** â€” the upstream "factor of 2" internal hack has been removed. Gravity now behaves exactly as defined. Falling feels fast and physical.
- **Jump speed is 9.5 nodes/s** â€” tuned specifically for 32.0 gravity. Consistently clears 1-block heights with a small margin.
- **Ground acceleration is 3.0** â€” eliminates the "ice-skating" feel. Players start and stop fast.
- **Built-in sprint** â€” engine detects â‰¥95% joystick input and applies a 1.3Ã— speed multiplier automatically. No mod needed. Can be disabled via accessibility settings.
- **Lava physics** â€” nodes in `group:lava` apply exponential velocity decay and a forced 0.5 block/s sink. Moving through lava feels genuinely thick.
- **Ladder overhaul** â€” crouch to descend, forward to boost-climb, optional `auto_climb` mode.
- **Smoother edge-grab** â€” sneak at ledge edges now reduces velocity 50% per frame instead of hitting an invisible wall.

**Default movement settings:**

| Setting | Value |
|---|---|
| `movement_gravity` | 32.0 |
| `movement_speed_walk` | 4.3 |
| `movement_speed_jump` | 9.5 |
| `movement_speed_crouch` | 1.3 |
| `movement_speed_climb` | 3.0 |
| `movement_acceleration_default` | 3.0 |
| `movement_acceleration_air` | 1.25 |

All values are multiplied by `set_physics_override`, so existing mods remain compatible.

**New `set_physics_override` fields:**

```lua
player:set_physics_override({
    speed_sprint           = 1.0,  -- Multiplies the built-in 1.3x sprint boost
    speed_walk             = 1.0,  -- Multiplies base walk speed independently
    auto_climb             = false, -- Auto-climb and auto-descend on ladders
    acceleration_default   = 1.0,
    acceleration_air       = 1.0,
})
```

**New node groups:**

- `group:lava` â€” enables high-viscosity liquid physics on any node
- `group:disable_jump` â€” prevents jumping while standing on the node

**New callbacks:**

```lua
core.register_on_jump(function(player) end)
core.register_on_land(function(player) end)
```

---

## Animation System

### glTF Multi-Clip Support

Each `animations[i]` entry in a glTF file is loaded as a selectable clip. Switch between them by index or name.

```lua
-- By index
obj:set_animation({ clip = 0, speed = 1.0 })

-- By name
obj:set_animation({ clip = "Run", speed = 1.0, loop = true })

-- Explicit form
obj:set_animation_clip("Walk", {x=0, y=2.0}, 1.0, 0.1, true)
```

### Time Mode

glTF animations use seconds; B3D/X use frames. The `time_mode` parameter handles this automatically.

```lua
obj:set_animation({
    range = {x = 0, y = 2.0},
    speed = 1.0,
    time_mode = "seconds"  -- works for any format
})
```

| `time_mode` | Behaviour |
|---|---|
| `"auto"` (default) | Seconds for glTF, frames for B3D/X. Warns if speed > 5 on glTF. |
| `"seconds"` | Always seconds. Engine converts for B3D/X automatically. |
| `"frames"` | Always frames. Engine converts for glTF automatically. |

### Smoothstep Blending

Animation transitions use smoothstep (ease-in/out) instead of linear blending. Default blend time is `0.1s`. Existing mods benefit automatically with no code changes.

### glTF Animation Events

Define events inside the glTF `extras` field:

```json
"animations": [{
  "name": "Walk",
  "extras": {
    "events": [
      { "time": 0.5, "name": "footstep_l" },
      { "time": 1.0, "name": "footstep_r" }
    ]
  }
}]
```

Receive them in Lua:

```lua
core.animator.register_on_event(function(animator, object, event)
    if event.name == "footstep_l" then
        minetest.sound_play("step_left", { object = object })
    end
end)
```

### Model Scaling API

No more guessing export scale:

```lua
obj:set_properties({
    visual = "mesh",
    mesh = "character.gltf",
    auto_normalize = true,  -- 1 unit in file = 1 node in world
    target_height = 1.8,    -- Force exact height in nodes
})
```

### Model Introspection

```lua
obj:get_model_info()
-- Returns: { mesh, format, uses_time, default_speed }

core.gltf_get_animation_clips(path)
-- Returns: { {index, name, start, end, duration}, ... }

core.gltf_inspect(path)
-- Returns: { meshes, bones, animations }
```

---

## Bone Transform API

Each transform type (position, rotation, scale) is stored and synced **independently**. Calling `set_bone_rotation` only updates rotation â€” it does not touch position or scale.

```lua
-- Additive override (on top of animation)
obj:set_bone_rotation("Head", {x=0, y=45, z=0})

-- Absolute override (replaces animation)
obj:set_bone_position("RightHand", {x=1, y=0, z=0}, { absolute = true })

-- With smooth interpolation
obj:set_bone_rotation("Head", {x=0, y=45, z=0}, { interpolation = 0.1 })
```

### Persistent Smoothing

Set a default interpolation duration per bone so you don't have to repeat it on every call:

```lua
obj:set_part_smooth("Head", { rotation = 0.5 })
-- All subsequent set_bone_rotation calls on "Head" take 0.5s by default
```

### Per-Part Visibility

```lua
obj:set_part_visible("Helmet", false)  -- Hides bone without destroying scale
obj:set_part_visible("Helmet", true)
```

### Bulk Override

```lua
obj:set_bone_override("RightArm", {
    rotation = { vec = {x=0, y=0, z=0}, absolute = false, interpolation = 0.2 },
    visible  = true,
    color    = "#FF8800",
    glow     = 0.5,
})
```

> âš ï¸ `set_bone_override` rotation uses **radians**. `set_bone_rotation` uses **degrees**. This is intentional â€” override operates at a lower level.

### World Position

```lua
local pos = obj:get_bone_world_pos("RightHand")
-- Ideal for attaching particles or effects to body parts
```

---

## Lua Animator (`core.animator`)

A state machine with blending, events, and additive layers.

```lua
local anim = core.animator.humanoid(player, {
    idle  = "Idle",
    walk  = "Walk",
    run   = "Run",
    jump  = "Jump",
}, {
    walk_threshold = 0.05,
    run_threshold  = 2.5,
    on_event = function(animator, obj, event)
        if event.name == "land" then
            minetest.sound_play("land", { object = obj })
        end
    end
})

core.animator.register(anim)
```

**Animation end / cycle helpers:**

```lua
core.on_animation_end(obj, function(obj)
    -- fires when a non-looping animation finishes
end)

core.on_animation_cycle(obj, function(obj)
    -- fires each time a looping animation wraps
end)
```

---

## Camera API

```lua
player:set_camera({
    mode      = "thirdpersonback",
    fov       = 90,
    tilt      = 15,          -- roll in degrees
    smooth    = true,        -- smooth orientation on client
    free_look = true,        -- server orientation is additive, not override
    fov_transition = 0.3,    -- smooth FOV change over 0.3s
})
```

### Look Direction Fix

Previously, `set_look_vertical` / `set_look_horizontal` sent a full teleport packet, freezing the player if called frequently. On protocol version â‰¥ 52, these now send a dedicated orientation-only packet. The player can move freely while the server controls their camera. Falls back to legacy behaviour for older clients automatically.

---

## Fog API

```lua
core.set_fog(player, {
    color               = "#8899AA",
    fog_start           = 0.6,
    fog_end             = 0.9,
    blend_time          = 1.5,
    max_density         = 0.8,
    max_density_height  = 64,
    zero_density_height = 128,
    turbulence          = 0.2,
    layers = {
        { color = "#AABBCC", max_density = 0.3, max_density_height = 32 }
    },
})

-- Localised fog zone
core.set_fog_boundary(player, {
    pos    = {x=0, y=40, z=0},
    radius = 20,
    shape  = "sphere",
    fog    = { color = "#223344", max_density = 1.0 },
    sound  = { name = "wind", gain = 0.5, fade_in = 2.0 },
})

-- Per-biome atmosphere
core.register_biome_atmosphere(biome_id, {
    fog = { ... },
    boundary = { ... },
})
```

---

## Android: HTMLView

Android-only. Renders HTML/CSS/JS overlays directly inside the game.

```lua
-- Show a fullscreen HTML UI
htmlview.run("hud", [[<html>...</html>]])
htmlview.display("hud", { fullscreen = true })

-- Message passing
htmlview.on_message_json("hud", function(data, raw, err)
    if data then
        minetest.log("Received: " .. data.action)
    end
end)
htmlview.send_json("hud", { type = "update", health = 20 })

-- Shared memory (zero-copy IPC)
htmlview.shared_set("player_pos", minetest.serialize(pos))
-- In JS: luanti.shared_get("player_pos")

-- Headless worker (no visible view)
htmlview.run_worker("compute", [[<script>...</script>]])
```

> âš ï¸ `htmlview.shared_get` returns `nil` for both a missing key and an empty string value. These cases are indistinguishable from Lua. Avoid storing empty strings.

> âš ï¸ All HTMLViews are destroyed when leaving a world.

---

## World API

```lua
-- Create a world programmatically (server-side)
local ok, path = core.create_world("MyWorld", "minetest", {
    seed    = "12345",
    mg_name = "v7",
    mods    = { default = true, mymod = "path/to/mymod" },
    visible = "hidden",
})

-- Switch a player to another world
minetest.world_switch(playername, "MyWorld")

-- Client-side self-switch
core.world_switch("MyWorld")

-- Synchronised world data
core.get_synchronized_worldpath()
```

---

## Accessibility

- **Sprint toggle** â€” disables the automatic 1.3Ã— joystick sprint multiplier for players who prefer consistent walking speed. Available in the Accessibility menu under Movement, reactive in real-time.

```lua
core.settings:get_bool("accessibilitysprintenabled")
```

- **Auto-climb** â€” moving toward a ladder climbs it. Crouch descends. Set via `set_physics_override({ auto_climb = true })`.

---

## Compatibility Notes

| Feature | Platform |
|---|---|
| HTMLView | Android only |
| Protocol v52 look sync | Requires matching client build |
| glTF multi-clip | All platforms |
| Bone API | All platforms |
| Physics model | All platforms |
| World API | Server-side (all platforms) |

---

## Status

This is a **pre-release**. Core systems are functional but some APIs may change before 1.0. Known rough edges:

- `set_bone_override` uses radians; `set_bone_rotation` uses degrees â€” this inconsistency will not be changed (intentional low-level vs high-level split)
- `htmlview.shared_get` cannot distinguish missing key from empty string value
- `get_animation_info` fields `progress` and `bones` are placeholders and always return `nil`
- HTMLView is destroyed on world leave â€” persistent state must be written to mod storage before leaving

---
