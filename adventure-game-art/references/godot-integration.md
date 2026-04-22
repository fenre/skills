# Godot 4 Integration — Adventure Game Art Pipeline

## Project Setup for Pixel Art

### Project Settings

Configure these settings in `Project > Project Settings` for pixel-perfect rendering:

```
Display > Window:
  Viewport Width:  320   (MI1/MI2) or 640 (MI3)
  Viewport Height: 200   (MI1/MI2) or 480 (MI3)
  Stretch > Mode:  viewport
  Stretch > Aspect: keep

Rendering > Textures:
  Canvas Textures > Default Texture Filter: Nearest
```

**Nearest** filtering is critical for pixel art — it preserves hard pixel edges. For MI3 cartoon style, use **Linear** instead for smoother scaling.

### Scaling for Modern Displays

The game viewport is small (320x200 or 640x480) but displays on modern monitors at much higher resolutions. The `viewport` stretch mode with `keep` aspect handles this by integer-scaling the viewport to fill the window while maintaining aspect ratio.

For non-square pixel emulation (MI1/MI2 at 1:1.2 aspect ratio):
- Set viewport to 320x200
- Stretch aspect to `keep` — this will scale to fill the window
- Alternatively, render at 320x240 and use a shader to squish to 320x200 appearance

### Project Folder Structure

```
project/
  assets/
    characters/
      guybrush/
        guybrush_walk.png
        guybrush_walk.json
        guybrush_idle.png
        guybrush_talk.png
        guybrush_portrait.png
    rooms/
      melee_dock/
        bg_sky.png
        bg_far.png
        bg_mid.png
        bg_main.png
        bg_foreground.png
        walkable_mask.png
    items/
      rubber_chicken.png
      map_piece.png
    ui/
      cursors/
        cursor_walk.png
        cursor_look.png
        cursor_use.png
        cursor_talk.png
      verb_bar.png
      inventory_panel.png
      dialog_box.png
    palettes/
      ega_16.gpl
      vga_256.gpl
  scenes/
    characters/
      guybrush.tscn
    rooms/
      melee_dock.tscn
    ui/
      inventory.tscn
      dialog.tscn
  scripts/
    characters/
      character_base.gd
    rooms/
      room_base.gd
    ui/
      inventory.gd
      dialog.gd
```

### Import Settings Per Asset Type

After adding images to the project, select them in the FileSystem panel and configure import settings in the Inspector:

| Asset Type | Filter | Mipmaps | Repeat |
|-----------|--------|---------|--------|
| Character sprites | Nearest | Off | Disabled |
| Room backgrounds | Nearest (pixel) / Linear (cartoon) | Off | Disabled |
| UI elements | Nearest | Off | Disabled |
| Parallax sky/clouds | Nearest | Off | Enabled (for seamless tiling) |

Click "Reimport" after changing settings.

---

## AnimatedSprite2D — Complete Setup

### Method 1: Individual Frame Images

Best when you have separate PNG files per frame (e.g., exported from Aseprite as a sequence).

1. Add `AnimatedSprite2D` node to your character scene
2. In Inspector, click `Sprite Frames` > `New SpriteFrames`
3. Click the SpriteFrames resource to open the editor panel (bottom of editor)
4. Rename the default animation from "default" to `idle_down`
5. Drag frame images into the animation panel
6. Set **Speed (FPS)**: 8-10 for pixel art, 12-15 for cartoon
7. Toggle **Loop** on for repeating animations (idle, walk)
8. Create additional animations: click the "Add Animation" button, name appropriately

**Naming convention for animations:**
```
idle_down, idle_up, idle_left, idle_right
walk_down, walk_up, walk_left, walk_right
talk_down, talk_up, talk_left, talk_right
interact_down, interact_up, interact_left, interact_right
```

### Method 2: Sprite Sheet (Grid)

Best when you have a single sprite sheet with all frames arranged in a grid.

1. Add `AnimatedSprite2D` node
2. Create `SpriteFrames` resource
3. Click the **grid icon** ("Add frames from Sprite Sheet") in the SpriteFrames panel
4. Select your sprite sheet PNG
5. Set **Horizontal** and **Vertical** frame count to match your grid layout
6. Click individual cells to select frames for the current animation
7. Click "Add frames" to add selected frames
8. Repeat for each animation

### Method 3: Sprite2D + AnimationPlayer

More control for complex animations (syncing sounds, hitboxes, particle effects).

1. Add `Sprite2D` node, assign your sprite sheet texture
2. Set `hframes` and `vframes` to match your grid (e.g., 6 columns, 4 rows)
3. Add `AnimationPlayer` node as a sibling
4. Create animations in AnimationPlayer that keyframe the `frame` property of Sprite2D
5. Each keyframe sets the `frame` index (0-based, left-to-right, top-to-bottom)

**When to use which:**

| Method | Best For |
|--------|----------|
| AnimatedSprite2D + individual frames | Simple characters, easy to manage |
| AnimatedSprite2D + sprite sheet | Most adventure game characters |
| Sprite2D + AnimationPlayer | Complex animations with sound/event sync |

---

## Animation State Management — GDScript

### Basic State Machine

```gdscript
extends CharacterBody2D

@onready var sprite: AnimatedSprite2D = $AnimatedSprite2D

var current_direction: String = "down"
var is_moving: bool = false
var is_talking: bool = false
var is_interacting: bool = false
var target_position: Vector2 = Vector2.ZERO
var move_speed: float = 80.0

func _process(delta: float) -> void:
    if is_interacting:
        return

    if is_moving:
        var direction = (target_position - global_position).normalized()
        velocity = direction * move_speed
        current_direction = _get_direction_name(direction)

        if global_position.distance_to(target_position) < 2.0:
            is_moving = false
            velocity = Vector2.ZERO

        move_and_slide()

    _update_animation()

func _update_animation() -> void:
    var anim_name: String

    if is_interacting:
        anim_name = "interact_" + current_direction
    elif is_talking:
        anim_name = "talk_" + current_direction
    elif is_moving:
        anim_name = "walk_" + current_direction
    else:
        anim_name = "idle_" + current_direction

    if sprite.animation != anim_name:
        sprite.play(anim_name)

func _get_direction_name(dir: Vector2) -> String:
    if abs(dir.x) > abs(dir.y):
        return "left" if dir.x < 0 else "right"
    else:
        return "up" if dir.y < 0 else "down"

func walk_to(pos: Vector2) -> void:
    target_position = pos
    is_moving = true

func start_talking() -> void:
    is_talking = true

func stop_talking() -> void:
    is_talking = false

func interact() -> void:
    is_interacting = true
    sprite.play("interact_" + current_direction)
    await sprite.animation_finished
    is_interacting = false
```

### Sprite Flipping for Left/Right

If you only have right-facing sprites, mirror for left:

```gdscript
func _update_animation() -> void:
    # ... (same as above)
    
    if current_direction == "left":
        sprite.flip_h = true
        # Use right-facing animation
        anim_name = anim_name.replace("_left", "_right")
    else:
        sprite.flip_h = false

    if sprite.animation != anim_name:
        sprite.play(anim_name)
```

---

## Point-and-Click Input Handling

### Click-to-Move

```gdscript
extends Node2D

@onready var player: CharacterBody2D = $Player
@onready var navigation: NavigationRegion2D = $NavigationRegion2D

func _input(event: InputEvent) -> void:
    if event is InputEventMouseButton and event.pressed:
        if event.button_index == MOUSE_BUTTON_LEFT:
            var click_pos = get_global_mouse_position()
            var nav_map = navigation.get_navigation_map()
            var nearest_point = NavigationServer2D.map_get_closest_point(nav_map, click_pos)
            player.walk_to(nearest_point)
```

### Cursor State Management

```gdscript
extends Node

enum CursorMode { WALK, LOOK, USE, TALK, PICKUP }

var current_mode: CursorMode = CursorMode.WALK

var cursor_textures: Dictionary = {
    CursorMode.WALK: preload("res://assets/ui/cursors/cursor_walk.png"),
    CursorMode.LOOK: preload("res://assets/ui/cursors/cursor_look.png"),
    CursorMode.USE: preload("res://assets/ui/cursors/cursor_use.png"),
    CursorMode.TALK: preload("res://assets/ui/cursors/cursor_talk.png"),
    CursorMode.PICKUP: preload("res://assets/ui/cursors/cursor_walk.png"),
}

func set_cursor_mode(mode: CursorMode) -> void:
    current_mode = mode
    Input.set_custom_mouse_cursor(cursor_textures[mode])

func cycle_cursor() -> void:
    current_mode = (current_mode + 1) % CursorMode.size() as CursorMode
    set_cursor_mode(current_mode)
```

---

## Parallax Background Setup

### Scene Tree Structure

```
Room (Node2D)
  ParallaxBackground
    SkyLayer (ParallaxLayer)
      Sprite2D [texture: bg_sky.png]
    FarLayer (ParallaxLayer)
      Sprite2D [texture: bg_far.png]
    MidLayer (ParallaxLayer)
      Sprite2D [texture: bg_mid.png]
    MainLayer (ParallaxLayer)
      Sprite2D [texture: bg_main.png]
    ForegroundLayer (ParallaxLayer)
      Sprite2D [texture: bg_foreground.png]
  NavigationRegion2D
    [walkable polygon]
  Player (CharacterBody2D)
  Hotspots (Node2D)
    DoorHotspot (Area2D)
    ChestHotspot (Area2D)
  Camera2D
```

### ParallaxLayer Motion Scale Settings

| Layer | motion_scale | Notes |
|-------|-------------|-------|
| Sky | (0.0, 0.0) to (0.1, 0.0) | Nearly static; very slow drift |
| Far background | (0.3, 0.0) | Slow movement; distant feel |
| Mid-ground | (0.6, 0.0) | Moderate movement |
| Main ground | (1.0, 1.0) | Locked to camera; where characters walk |
| Foreground | (1.2, 0.0) to (1.5, 0.0) | Moves faster than camera for depth |

Set `motion_scale` in the Inspector for each `ParallaxLayer` node.

### Camera Setup for Scrolling Rooms

```gdscript
extends Camera2D

@export var follow_target: Node2D
@export var room_width: int = 960
@export var room_height: int = 200
@export var viewport_width: int = 320
@export var viewport_height: int = 200

func _ready() -> void:
    limit_left = 0
    limit_top = 0
    limit_right = room_width
    limit_bottom = room_height
    position_smoothing_enabled = true
    position_smoothing_speed = 5.0

func _process(delta: float) -> void:
    if follow_target:
        global_position = follow_target.global_position
```

---

## Walkable Areas

### Using NavigationRegion2D

1. Add `NavigationRegion2D` node to the room scene
2. In the Inspector, create a new `NavigationPolygon`
3. Click the polygon in the editor to enter edit mode
4. Draw the walkable area polygon by clicking to place points
5. Click "Bake NavigationPolygon" (or it auto-bakes)

### Character Pathfinding with NavigationAgent2D

Add `NavigationAgent2D` as a child of the character:

```gdscript
extends CharacterBody2D

@onready var nav_agent: NavigationAgent2D = $NavigationAgent2D
@onready var sprite: AnimatedSprite2D = $AnimatedSprite2D

var move_speed: float = 80.0

func _ready() -> void:
    nav_agent.path_desired_distance = 4.0
    nav_agent.target_desired_distance = 4.0

func walk_to(target: Vector2) -> void:
    nav_agent.target_position = target

func _physics_process(delta: float) -> void:
    if nav_agent.is_navigation_finished():
        velocity = Vector2.ZERO
        _update_animation(false)
        return

    var next_pos = nav_agent.get_next_path_position()
    var direction = (next_pos - global_position).normalized()
    velocity = direction * move_speed
    _update_direction(direction)
    _update_animation(true)
    move_and_slide()

func _update_direction(dir: Vector2) -> void:
    if abs(dir.x) > abs(dir.y):
        current_direction = "left" if dir.x < 0 else "right"
    else:
        current_direction = "up" if dir.y < 0 else "down"

var current_direction: String = "down"

func _update_animation(moving: bool) -> void:
    var anim = ("walk_" if moving else "idle_") + current_direction
    if sprite.animation != anim:
        sprite.play(anim)
```

---

## Hotspots and Interactive Objects

### Hotspot Setup

Interactive objects use `Area2D` nodes with collision shapes:

```gdscript
extends Area2D

@export var object_name: String = "Old Door"
@export var look_description: String = "A sturdy wooden door with iron hinges."
@export var use_action: String = "It's locked. I need a key."

signal looked_at(description: String)
signal used(result: String)
signal picked_up(item_name: String)

func interact(mode: int) -> void:
    match mode:
        0: # WALK
            pass
        1: # LOOK
            looked_at.emit(look_description)
        2: # USE
            used.emit(use_action)
        3: # TALK
            looked_at.emit("I don't think it wants to talk.")
        4: # PICKUP
            looked_at.emit("I can't pick that up.")
```

### Hotspot Visual Highlight

Show interactive objects when hovering:

```gdscript
func _on_mouse_entered() -> void:
    modulate = Color(1.2, 1.2, 1.2)  # Slight brighten

func _on_mouse_exited() -> void:
    modulate = Color(1.0, 1.0, 1.0)
```

---

## Inventory System — Sprite Setup

### Inventory Item Display

```gdscript
extends TextureRect

@export var item_id: String = ""
@export var item_name: String = ""
@export var item_texture: Texture2D

func _ready() -> void:
    texture = item_texture
    custom_minimum_size = Vector2(32, 32)  # MI1/MI2
    # custom_minimum_size = Vector2(64, 64)  # MI3
    stretch_mode = TextureRect.STRETCH_KEEP_ASPECT_CENTERED
    
    mouse_filter = Control.MOUSE_FILTER_STOP
    tooltip_text = item_name
```

### Inventory Panel (HBoxContainer)

```
InventoryPanel (PanelContainer)
  ScrollContainer
    HBoxContainer  [alignment: center]
      ItemSlot1 (TextureRect)
      ItemSlot2 (TextureRect)
      ...
```

---

## Dialog Portrait System

### Portrait Display During Dialog

```gdscript
extends Control

@onready var portrait_sprite: TextureRect = $PortraitSprite
@onready var name_label: Label = $NameLabel
@onready var dialog_label: RichTextLabel = $DialogLabel

var portraits: Dictionary = {}

func show_dialog(character_id: String, expression: String, text: String) -> void:
    var portrait_path = "res://assets/characters/%s/%s_portrait_%s.png" % [
        character_id, character_id, expression
    ]
    portrait_sprite.texture = load(portrait_path)
    name_label.text = character_id.capitalize()
    dialog_label.text = ""
    
    # Typewriter effect
    for ch in text:
        dialog_label.text += ch
        await get_tree().create_timer(0.03).timeout
    
    # Wait for click to continue
    await _wait_for_click()
    hide()

func _wait_for_click() -> void:
    while true:
        var event = await self.gui_input
        if event is InputEventMouseButton and event.pressed:
            break
```

---

## Popochiu Addon — Quick Start

[Popochiu](https://github.com/carenalgas/popochiu) handles most adventure game systems out of the box.

### Installation

1. **Godot AssetLib**: Search "Popochiu" in AssetLib tab, install
2. **Manual**: Clone repo, copy `addons/popochiu/` into your project's `addons/` folder
3. Enable in `Project > Project Settings > Plugins > Popochiu`

### Core Concepts

| Popochiu Object | Purpose | Art Assets Needed |
|----------------|---------|-------------------|
| **Room** | A game scene/location | Background layers, walkable area polygon |
| **Character** | Player or NPC | Sprite sheets for walk/idle/talk |
| **Prop** | Interactive environment object | Static or animated sprite |
| **Hotspot** | Invisible interactive area | None (polygon only) |
| **Inventory Item** | Collectible object | Icon sprite (32x32 or 64x64) |
| **Dialog** | Branching conversation | Portrait sprites (optional) |

### Creating a Room

1. In Popochiu dock: click **Room** > type name > **Create**
2. Place background sprite in the room scene
3. Add **Walkable Areas**: draw polygon in the WalkableArea node
4. Add **Props**: create prop nodes, assign sprites, wire up interactions
5. Add **Hotspots**: invisible Area2D regions with interaction scripts

### Creating a Character

1. In Popochiu dock: click **Character** > type name > **Create**
2. Open the character scene
3. Assign sprite sheets to the AnimatedSprite2D
4. Create animations: `idle`, `walk_r`, `walk_l`, `walk_u`, `walk_d`, `talk`
5. Set `walk_speed` in the character script

### Adding Inventory Items

1. In Popochiu dock: click **Inventory Item** > type name > **Create**
2. Assign the item icon texture
3. Wire up the item's interaction script (use, combine, look)

### Popochiu Interaction Script Pattern

```gdscript
# In a Prop or Hotspot script:
func _on_click() -> void:
    await C.player.walk_to_clicked()
    await C.player.face_clicked()
    await D.say("Guybrush", "Interesting...")

func _on_right_click() -> void:
    await D.say("Guybrush", "It's a rusty old sword.")

func _on_item_used(item: PopochiuInventoryItem) -> void:
    if item.script_name == "Key":
        await D.say("Guybrush", "It fits!")
        await C.player.play_animation("interact")
        # Remove key from inventory, change room state
        I.Key.remove()
        self.hide()
```

---

## Palette Cycling Shader (Advanced)

For EGA/VGA-style palette cycling in Godot 4:

```gdscript
# palette_cycle.gdshader
shader_type canvas_item;

uniform sampler2D palette_texture : filter_nearest;
uniform float cycle_offset : hint_range(0.0, 1.0) = 0.0;
uniform float cycle_start : hint_range(0.0, 1.0) = 0.625;  // index 160/256
uniform float cycle_end : hint_range(0.0, 1.0) = 0.75;     // index 192/256

void fragment() {
    vec4 pixel = texture(TEXTURE, UV);
    
    // Use red channel as palette index (grayscale indexed image)
    float index = pixel.r;
    
    // Apply cycling offset to indices in the cycling range
    if (index >= cycle_start && index < cycle_end) {
        float range_size = cycle_end - cycle_start;
        float local_index = index - cycle_start;
        local_index = mod(local_index + cycle_offset * range_size, range_size);
        index = cycle_start + local_index;
    }
    
    // Look up color from palette texture (256x1 image)
    COLOR = texture(palette_texture, vec2(index, 0.5));
}
```

Control the cycling from GDScript:

```gdscript
extends Sprite2D

@export var cycle_speed: float = 4.0  # shifts per second

func _process(delta: float) -> void:
    var mat = material as ShaderMaterial
    var offset = mat.get_shader_parameter("cycle_offset")
    offset = fmod(offset + delta * cycle_speed / 8.0, 1.0)
    mat.set_shader_parameter("cycle_offset", offset)
```

---

## Performance Tips for Pixel Art Games

1. **Use texture atlases** — pack sprites into larger sheets to reduce draw calls
2. **Limit AnimatedSprite2D count** — share SpriteFrames resources between identical NPCs
3. **Pre-bake palette cycling** — for static scenes, generate 8-16 pre-cycled background frames and swap textures instead of using shaders
4. **NavigationRegion2D baking** — bake once at room load, not every frame
5. **Disable mipmaps** — unnecessary for pixel art and wastes memory
6. **Match viewport to art resolution** — never render at a higher internal resolution than your art supports
