---
name: godot-engine
description: >
  Godot 4 game engine reference covering GDScript, scene/node architecture, 2D and 3D
  development, sprites, pixel art, animation (AnimatedSprite2D, AnimationPlayer,
  AnimationTree state machines), physics (CharacterBody2D, RigidBody2D, Area2D),
  tilemaps with terrains, shaders (CanvasItem, visual), UI/Control nodes, audio buses,
  navigation/pathfinding, particles, camera systems, input handling, save/load,
  project structure, autoloads, state machines, export/build, and Godot 3→4 migration.
  Use when building any game in Godot, writing GDScript, configuring Godot project
  settings, creating sprites or animations in Godot, working with Godot shaders,
  or asking about Godot Engine features and best practices.
---

# Godot 4 Engine Reference

## Scene Tree & Node Architecture

Everything in Godot is a **Node** organized in a tree. Scenes are reusable node subtrees saved as `.tscn` files.

### Node Lifecycle

```gdscript
func _enter_tree() -> void:    # node added to tree
func _ready() -> void:         # node + all children ready (init here)
func _process(delta: float) -> void:        # every frame
func _physics_process(delta: float) -> void: # fixed timestep (default 60Hz)
func _exit_tree() -> void:     # node removed from tree
```

### Scene Communication Rules

- **Downward** (parent→child): call methods directly (`$Child.do_thing()`)
- **Upward** (child→parent): emit signals
- **Sideways** (sibling→sibling): through parent or autoload

Keep scenes self-contained with no external dependencies when possible. Use dependency injection for flexibility.

---

## GDScript Quick Reference

### Variables & Typing

```gdscript
var speed := 200.0              # inferred float
var name: String = "Player"     # explicit type
const MAX_HP: int = 100         # constant
@export var damage: float = 10.0  # editable in Inspector
@export_range(0, 100) var health: int = 100
@onready var sprite := $Sprite2D   # resolved when node enters tree
```

**Static typing** gives 28–59% speedups and catches errors at parse time. Always use `:=` or `: Type`.

### Signals

```gdscript
signal health_changed(new_hp: int)
signal died

# Emit
health_changed.emit(hp)

# Connect (in _ready)
button.pressed.connect(_on_button_pressed)
enemy.died.connect(_on_enemy_died)
```

Name signals in past tense (`health_changed`, `died`) or with `_started`/`_finished`.

### Script Ordering Convention

```
@tool / class_name / extends
signals
enums
constants
static variables
@export variables
regular variables
@onready variables
_ready() / lifecycle methods
public methods
private methods (_prefixed)
```

### Key Syntax

```gdscript
for item in array:                          # foreach
for i in range(10):                         # counted loop
match state:                                # switch/match
    State.IDLE: pass
    State.RUN: handle_run()
if node is CharacterBody2D:                 # type check
var enemy := node as Enemy                  # cast
await get_tree().create_timer(1.0).timeout  # coroutine
await animation_player.animation_finished
```

---

## Project Structure

```
project/
├── assets/           # raw art, audio, fonts
│   ├── sprites/
│   ├── audio/
│   └── fonts/
├── source/           # scenes, scripts, resources
│   ├── autoload/     # global singletons
│   ├── player/       # player scene + scripts
│   ├── enemies/
│   ├── levels/
│   ├── ui/
│   └── shaders/
├── data/             # saves, configs, translations
└── project.godot
```

Use `snake_case` for all files/folders. Group by feature, not by type.

### Autoloads (Singletons)

Register in Project Settings → Autoload. Always loaded, accessible globally by script name (e.g., `GameManager.score += 100`). Use for cross-scene state, audio managers, scene transitions.

---

## Sprites & Graphics (2D)

### Sprite2D

Basic static sprite display. Key properties: `texture`, `offset`, `flip_h`/`flip_v`, `hframes`/`vframes` (for spritesheets), `frame`.

```gdscript
# Spritesheet with 8 columns, 4 rows
sprite.hframes = 8
sprite.vframes = 4
sprite.frame = 12   # specific frame index
```

### AnimatedSprite2D + SpriteFrames

For frame-by-frame animation:

1. Add `AnimatedSprite2D` node
2. Create `New SpriteFrames` in Inspector
3. In SpriteFrames panel: add animations, drag in frames or import from spritesheet (grid icon)
4. Set FPS per animation, toggle loop

```gdscript
@onready var anim := $AnimatedSprite2D

func _ready() -> void:
    anim.play("idle")

func set_movement(vel: Vector2) -> void:
    if vel.length() > 0:
        anim.play("run")
        anim.flip_h = vel.x < 0
    else:
        anim.play("idle")
```

**Signals**: `animation_finished`, `animation_looped`, `frame_changed`.

### AtlasTexture (Manual Spritesheet Regions)

For extracting regions from a packed texture:

```gdscript
var atlas := AtlasTexture.new()
atlas.atlas = preload("res://assets/sprites/characters.png")
atlas.region = Rect2(0, 0, 32, 32)   # x, y, w, h
sprite.texture = atlas
```

Use Auto Slice: Sprite2D → Texture → New AtlasTexture → Edit Region → Snap Mode: Auto Slice.

### Pixel Art Setup

| Setting | Value | Location |
|---------|-------|----------|
| Default Texture Filter | **Nearest** | Project Settings → Rendering → Textures |
| Window Stretch Mode | **canvas_items** | Project Settings → Display → Window → Stretch |
| Window Stretch Aspect | **keep** | Same location |
| Snap 2D Transforms to Pixel | **On** | Project Settings → Rendering → 2D |
| Snap 2D Vertices to Pixel | **On** | Same location |
| Base resolution | **320×180** or **640×360** | Display → Window |

Per-import: set Filter to Off, Mipmaps Off, Compression to Lossless.

---

## Animation Systems

### AnimationPlayer

Keyframe-based animation of **any** node property — position, scale, modulate, shader params, method calls.

```gdscript
@onready var anim_player := $AnimationPlayer

anim_player.play("attack")
await anim_player.animation_finished
anim_player.play("idle")
```

### AnimationTree + State Machine

For complex character animation with blending and transitions:

```gdscript
@onready var anim_tree := $AnimationTree
@onready var state_machine := anim_tree["parameters/playback"] as AnimationNodeStateMachinePlayback

func update_animation(velocity: Vector2) -> void:
    if velocity.length() > 10:
        state_machine.travel("run")
    else:
        state_machine.travel("idle")
```

Transitions between states: set switch mode to **Immediate** or **At End**. Enable **Auto** advance for automatic transitions based on conditions.

### Tweens

One-shot property interpolation (not a node in Godot 4):

```gdscript
var tween := create_tween()
tween.tween_property($Sprite2D, "modulate:a", 0.0, 0.5)
tween.tween_callback(queue_free)

# Parallel tweens
var t := create_tween().set_parallel(true)
t.tween_property(self, "scale", Vector2(1.5, 1.5), 0.2)
t.tween_property(self, "modulate", Color.RED, 0.2)
```

Chain with `.set_trans(Tween.TRANS_BOUNCE)`, `.set_ease()`. Sequential by default; call `.set_parallel(true)` for simultaneous.

---

## Physics & Collision

### Body Types

| Node | Use For | Movement |
|------|---------|----------|
| `StaticBody2D` | Walls, floors, platforms | Does not move |
| `CharacterBody2D` | Player, NPCs | You control via code |
| `RigidBody2D` | Crates, projectiles, ragdolls | Physics engine controls |
| `Area2D` | Triggers, pickups, damage zones | Detection only (no blocking) |

Every body needs a `CollisionShape2D` child with a shape resource.

### CharacterBody2D Movement

Set `velocity` in `_physics_process()`, then call `move_and_slide()`. Use `is_on_floor()`, `is_on_wall()`, `is_on_ceiling()` for state checks. Get gravity from `ProjectSettings.get_setting("physics/2d/default_gravity")`.

### Collision Layers & Masks

**Layer** = what this object IS. **Mask** = what it SCANS FOR. Name layers in Project Settings → Layer Names → 2D Physics.

### Area2D

Signals: `area_entered`, `body_entered`, `area_exited`, `body_exited`. **CharacterBody2D does NOT actively detect Area2D** — always connect from the Area2D side.

---

## TileMap & TileSet

### Setup

1. Create `TileMap` node, assign a new `TileSet`
2. In TileSet: add texture sources (your tileset image)
3. Configure **physics layers** (collision) and **terrain sets** (autotile)
4. Paint tiles in the TileMap editor

### Terrains (Autotile)

Replaces Godot 3 autotile. Automatically selects correct border tiles:

1. TileSet → Terrain Sets → add terrain set
2. Choose mode: **Match Corners and Sides** (complex) or **Match Sides** (simple)
3. Define terrains (grass, dirt, water) with colors
4. For each tile: set peering bits (3×3 grid showing neighbor connections)
5. Paint with Terrain brush — Godot auto-selects correct tiles

### Coordinate Conversion

```gdscript
var map_pos: Vector2i = tile_map.local_to_map(world_position)
var world_pos: Vector2 = tile_map.map_to_local(map_pos)
var tile_data: TileData = tile_map.get_cell_tile_data(layer, map_pos)
```

---

## Shaders (2D)

### CanvasItem Shader Basics

```glsl
shader_type canvas_item;

uniform vec4 flash_color : source_color = vec4(1.0, 1.0, 1.0, 1.0);
uniform float flash_amount : hint_range(0.0, 1.0) = 0.0;

void fragment() {
    vec4 tex = texture(TEXTURE, UV);
    COLOR = mix(tex, flash_color, flash_amount);
    COLOR.a = tex.a;
}
```

### Common 2D Effects

| Effect | Technique |
|--------|-----------|
| **Hit flash** | Mix sprite color with white using uniform |
| **Outline** | Sample neighboring pixels, draw if adjacent to transparent |
| **Dissolve** | Compare noise texture to threshold, discard below |
| **Water distortion** | Offset UV with noise + sine wave |
| **CRT scanlines** | Modulate by `sin(SCREEN_UV.y * line_count)` |
| **Drop shadow** | Offset UV, sample, tint dark |

Apply shaders via Material → New ShaderMaterial on any CanvasItem node. For reuse across nodes, use **CanvasGroup** or shared material resources.

---

## Audio

### Player Types

| Node | Use |
|------|-----|
| `AudioStreamPlayer` | Non-positional (music, UI sounds) |
| `AudioStreamPlayer2D` | Positional with 2D panning |
| `AudioStreamPlayer3D` | Positional with 3D spatialization |

### Audio Buses

Configure in bottom panel → Audio. Route players to specific buses via `bus` property. Add effects (reverb, distortion, EQ) per bus. Master bus should not exceed 0 dB.

### Music Crossfade

Use two `AudioStreamPlayer` nodes. Tween volume_db from -80→0 on new player and 0→-80 on old player in parallel. Swap references after fade completes.

### Interactive Music (4.3+)

- `AudioStreamInteractive`: multiple clips with configurable transitions
- `AudioStreamPlaylist`: sequential or shuffled playback
- `AudioStreamSynchronized`: layered stems with individual volume

---

## Camera2D

Enable `position_smoothing_enabled` + set `position_smoothing_speed` for smooth follow. Set `limit_left/top/right/bottom` for boundaries. For screen shake, use trauma-based approach: add trauma (0–1) on impact, decay over time, apply `trauma²` as offset/rotation. Anti-jitter: enable Snap 2D Transforms/Vertices to Pixel.

---

## Parallax Backgrounds

Use `Parallax2D` nodes (4.3+, replaces deprecated `ParallaxBackground`/`ParallaxLayer`). Set `scroll_scale` per layer: sky=0.1, mid=0.3, near=0.6. Set `repeat_size` for infinite scrolling.

---

## Navigation & Pathfinding

1. Add `NavigationRegion2D` — draw/bake walkable polygon
2. Add `NavigationAgent2D` as child of character
3. Set `nav_agent.target_position`, then in `_physics_process`: call `get_next_path_position()`, move toward it, check `is_navigation_finished()`

Use `enter_cost`/`travel_cost` for weighted paths. RVO avoidance is built-in.

---

## Particles

| Node | Processing | Best For |
|------|-----------|----------|
| `GPUParticles2D` | GPU | Thousands of particles — fire, rain, explosions |
| `CPUParticles2D` | CPU | Hundreds of particles — per-particle script access |

Key `ParticleProcessMaterial` settings:
- `explosiveness`: 0 = gradual, 1 = all at once
- `one_shot`: emit once then stop
- `emission_shape`: Point, Sphere, Ring, Box
- `spread`: emission cone angle (0–180°)
- `gravity`, `damping`, `angular_velocity`

Godot 4 additions: manual `emit_particle()`, sub-emitters, attractors, SDF collision.

---

## Input Handling

```gdscript
# Polling (continuous actions, in _process/_physics_process)
var dir := Input.get_vector("move_left", "move_right", "move_up", "move_down")
if Input.is_action_pressed("fire"):

# Event-based (one-shot actions)
func _unhandled_input(event: InputEvent) -> void:
    if event.is_action_pressed("jump"):
        jump()
        get_viewport().set_input_as_handled()
```

Define actions in Project Settings → Input Map. Supports keyboard, mouse, gamepad, touch.

Prefer `_unhandled_input()` over `_input()` — it respects UI consumption.

---

## Save / Load

### Custom Resources (Recommended)

Create a `class_name SaveData extends Resource` with `@export` fields. Save with `ResourceSaver.save(data, "user://save.tres")`, load with `ResourceLoader.load("user://save.tres") as SaveData`. **Always** save to `user://` (not `res://` — read-only in exports).

| Method | Use Case |
|--------|----------|
| `ConfigFile` | Settings (volume, keybinds) — INI-style |
| `JSON` | External API data, web integration |
| `Resource` (.tres) | Game saves (recommended — full type support) |

---

## State Machine Pattern

Node-based FSM: create `State` base class extending `Node` with `enter()`, `exit()`, `process_input()`, `process_frame()`, `process_physics()` — each returns a `State` (or null to stay). `StateMachine` node manages `current_state`, delegates lifecycle calls, and calls `transition_to()` when a state returns non-null.

Scene tree: `Player → StateMachine → [IdleState, RunState, JumpState, FallState]`

For simple cases (2–3 states), an enum + `match` in `_process` is sufficient.

---

## Export / Build

| Platform | Notes |
|----------|-------|
| **Windows** | 32/64-bit; simplest setup |
| **macOS** | May require Apple notarization |
| **Linux** | x86_64 and ARM |
| **Web** | Needs COOP/COEP headers for threads; disable threads for simpler hosting |
| **Android** | JDK 17+, Android SDK, keystore signing |
| **iOS** | Requires macOS + Xcode |

Setup: Project → Export → Add Preset → configure per platform. Export templates must match Godot version.

---

## Godot 3 → 4 Migration Cheat Sheet

| Godot 3 | Godot 4 |
|----------|---------|
| `Spatial` | `Node3D` |
| `KinematicBody2D` | `CharacterBody2D` |
| `export var` | `@export var` |
| `onready var` | `@onready var` |
| `tool` | `@tool` |
| `.instance()` | `.instantiate()` |
| `translation` | `position` |
| `rand_range()` | `randf_range()` / `randi_range()` |
| `Tween` (node) | `create_tween()` (object) |
| `connect("sig", obj, "method")` | `sig.connect(callable)` |
| `yield()` | `await` |
| `move_and_slide(vel)` | `velocity = vel; move_and_slide()` |

---

## Additional Resources

- For 3D specifics (materials, PBR, lighting), see [godot-3d-reference.md](godot-3d-reference.md)
- For shader cookbook with copy-paste examples, see [godot-shader-cookbook.md](godot-shader-cookbook.md)
