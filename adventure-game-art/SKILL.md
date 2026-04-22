---
name: adventure-game-art
description: Comprehensive guide for creating 2D adventure game art in the style of Monkey Island 1, 2, and 3. Covers three art eras (EGA pixel, VGA pixel, hand-drawn cartoon), manual pixel art creation in Aseprite, AI image generation with Stable Diffusion and DALL-E, sprite sheets, character animation, room backgrounds, parallax scrolling, inventory items, UI elements, color palettes, dithering, palette cycling, and Godot 4 integration including Popochiu addon. Use when Claude needs to help with (1) creating pixel art or cartoon sprites for adventure games, (2) AI-generating game art assets with proper prompts, (3) Aseprite workflow and sprite sheet export, (4) Godot 4 sprite import and animation setup, (5) adventure game room/background design, (6) retro color palettes and dithering techniques, (7) character walk cycles and animation, (8) point-and-click game art pipeline, (9) any Monkey Island-style game art question.
---

# 2D Adventure Game Art — Complete Reference

> Art creation guide for point-and-click adventure games in the tradition of Monkey Island. Covers three distinct art eras, manual and AI-assisted workflows, and a complete asset pipeline from concept to Godot Engine.

---

## 1. Art Asset Inventory

Every point-and-click adventure game needs these asset categories:

| Asset Type | Description | Typical Count per Room |
|------------|-------------|----------------------|
| **Room backgrounds** | Static or parallax-layered scene art | 1 per room (3-5 parallax layers for scrolling rooms) |
| **Character sprites** | Animated sprite sheets: idle, walk, talk, interact | 1 main character + NPCs per room |
| **Inventory items** | Small icons (32x32 to 64x64) for collected objects | 20-60 total across game |
| **UI elements** | Verb bar / interaction coin, inventory panel, dialog box, cursors | 1 global set |
| **Portraits** | Close-up faces for dialog (optional, MI3 style) | 1 per speaking character |
| **Props** | Interactive objects with animation (levers, doors, candles) | 3-10 per room |
| **Cutscene art** | Full-screen or partial illustrations for story beats | 5-15 total |
| **Effects** | Palette-cycled water/fire, particle-like sparkles, lighting overlays | Per room as needed |

---

## 2. Three Art Eras

### Era 1: EGA/VGA Pixel Art (MI1 — The Secret of Monkey Island, 1990)

| Spec | Value |
|------|-------|
| Resolution | 320x200 |
| Pixel aspect ratio | 1:1.2 (non-square, taller than wide) |
| Colors | 16 (EGA) or 256 (VGA) |
| Character height | ~32-40 pixels |
| Key techniques | Heavy dithering, palette cycling (water, fire, sunsets), night-dominant scenes to mask color limits |
| Key artists | Mark Ferrari (backgrounds, palette cycling), Steve Purcell (character design) |

**Defining characteristics:** Atmospheric use of limited colors. Dithered gradients. Palette cycling creates animated water, fire, and lighting without changing pixel data. Night and interior scenes predominate to work within color constraints.

### Era 2: VGA Pixel Art (MI2 — LeChuck's Revenge, 1991)

| Spec | Value |
|------|-------|
| Resolution | 320x200 |
| Pixel aspect ratio | 1:1.2 |
| Colors | 256 (VGA) |
| Character height | ~40-48 pixels |
| Key techniques | Impressionist color blending (adjacent complementary colors), richer scene palettes, more expressive character animation |
| Key artists | Steve Purcell, Peter Chan |

**Defining characteristics:** Full 256-color palette creates richer, more painterly scenes. Adjacent complementary colors blend optically (technique inspired by Impressionism). Characters are more detailed with distinct facial features. Wider variety of scene lighting (day, dusk, underground, tropical).

### Era 3: Hand-Drawn Cartoon (MI3 — The Curse of Monkey Island, 1997)

| Spec | Value |
|------|-------|
| Resolution | 640x480 |
| Colors | 256 |
| Character height | ~100-140 pixels |
| Key techniques | Cel animation drawn on paper, vectorized via US Animation software, anti-aliased lines, Disney-influenced character proportions |
| Key artists | Bill Tiller (backgrounds), Larry Ahern (character animation) |

**Defining characteristics:** Every animation frame drawn on paper first, then scanned and vectorized. Smooth anti-aliased outlines. Exaggerated cartoon proportions (large heads, expressive hands). Backgrounds are richly painted with visible brushwork. Bold color choices with strong silhouettes.

For full palette data and technical details, see [references/art-styles-and-palettes.md](references/art-styles-and-palettes.md).

---

## 3. Color Palettes and Core Techniques

### EGA Palette (16 Colors)

The canonical EGA palette: Black, Blue, Green, Cyan, Red, Magenta, Brown, Light Gray, Dark Gray, Light Blue, Light Green, Light Cyan, Light Red, Light Magenta, Yellow, White. Exact hex values in [references/art-styles-and-palettes.md](references/art-styles-and-palettes.md).

### VGA Palette Construction

Build custom 256-color palettes using **color ramps** — sequences of 4-8 colors from shadow to highlight for each material:
- **Skin tones**: 6-8 colors from deep shadow to highlight
- **Foliage**: 6 greens from near-black to bright lime
- **Stone/wood**: 6-8 earth tones
- **Sky gradient**: 8-12 blues from horizon haze to zenith
- **Water**: 6-8 blues/teals reserved for palette cycling
- Reserve 4-8 slots for UI and effects

### Dithering Patterns

| Pattern | Use Case |
|---------|----------|
| **Checkerboard** | Classic 50/50 mix of two colors; strongest retro feel |
| **Ordered (Bayer matrix)** | Smooth gradients with structured dot patterns |
| **Diagonal lines** | Directional shading on surfaces |
| **Random/stipple** | Organic textures (sand, bark, stone) |

### Palette Cycling

Animate scenes without changing pixel data — rotate a range of color indices each frame:
- **Water**: Cycle 6-8 blue shades in sequence; creates flowing/shimmering effect
- **Fire/torches**: Cycle red-orange-yellow indices; irregular offset between flames
- **Sunset/sunrise**: Slowly rotate sky and water palette ranges simultaneously
- **Lights**: Pulse brightness of light-source colors

### MI3 Cartoon Color Approach

- **Flat base fill** with 2-3 shading steps (shadow, midtone, highlight)
- **Strong dark outlines** (2-3px at 640x480) with varying line weight for depth
- **Rim lighting** on characters facing light sources
- **Warm/cool contrast**: warm foreground, cool background (or vice versa for mood)

---

## 4. Manual Art Workflow (Aseprite)

### Setup

1. **New file**: Set canvas to target resolution (320x200 for MI1/MI2, 640x480 for MI3)
2. **Load palette**: Import `.pal` or `.gpl` file matching target era. For custom VGA palettes, build color ramps in Aseprite's palette editor.
3. **Enable pixel-perfect mode**: Pencil tool > check "Pixel Perfect" to prevent diagonal doubles
4. **Set grid**: View > Grid Settings > match sprite cell size (e.g., 48x64 for character cells)

### Character Sprite Sheet Layout

Organize frames in a grid. Standard layout for a 4-direction character:

```
Row 0: Idle      [down] [left] [right] [up]
Row 1: Walk      [down x6] [left x6] [right x6] [up x6]
Row 2: Talk      [down x3] [left x3] [right x3] [up x3]
Row 3: Interact  [down x4] [left x4] [right x4] [up x4]
```

**Frame counts per action:**

| Action | MI1/MI2 (pixel) | MI3 (cartoon) |
|--------|-----------------|---------------|
| Idle | 1-2 frames | 4-6 frames (breathing, blinking) |
| Walk cycle | 4-6 frames per direction | 8-12 frames per direction |
| Talk | 2-4 frames (mouth + gesture) | 4-8 frames (lip sync + body) |
| Interact/Use | 3-5 frames | 6-10 frames |
| Pick up | 3-4 frames | 4-6 frames |

**FPS:** 8-10 FPS for pixel art walk cycles, 12-15 FPS for cartoon animation.

### Layer Organization

```
Layer stack (top to bottom):
  - Highlight    (brightest spots, specular)
  - Shading      (shadow areas)
  - Fill         (flat base colors)
  - Outline      (black or near-black lines)
  - Guide        (hidden; proportions, center marks)
```

### Export for Godot

**Sprite sheet export (Ctrl+Alt+Shift+S):**
- Layout: **By Rows** or **Grid** (match your frame arrangement)
- Check **Trim Cels** only if exporting with JSON metadata
- Always export **JSON data** alongside the PNG — Godot and tools can parse frame positions
- Format: PNG (lossless, transparency support)
- Padding: 1-2px between frames to prevent bleed with texture filtering

**CLI batch export:**
```bash
aseprite -b character.aseprite \
  --sheet character_sheet.png \
  --sheet-type rows \
  --data character_sheet.json \
  --format json-array
```

---

## 5. AI Art Generation Workflow

### Overview

AI generation accelerates the concepting and drafting phase. Final game art still requires manual post-processing for consistency, palette compliance, and proper sprite sheet formatting.

### Recommended Models

| Style Target | Best Models | Notes |
|-------------|-------------|-------|
| MI1/MI2 pixel art | SDXL + pixel art LoRA, Flux | Best with "16-bit pixel art" or "VGA pixel art" style tags |
| MI3 cartoon | DALL-E 3, Midjourney, SDXL | Cartoon cel-shading prompts; DALL-E 3 for clean line art |
| Backgrounds | Any model | Largest benefit; backgrounds are the most time-intensive manual asset |

### Prompt Formula

**Five-part structure:**
1. **Art style and era**: "VGA pixel art, 256-color palette, 1990s adventure game style"
2. **Subject description**: "pirate captain standing on dock, blue coat, tricorn hat"
3. **Pose and framing**: "side view, idle standing pose, full body"
4. **Color/palette constraint**: "limited color palette, warm tropical colors"
5. **Background**: "plain white background" (for sprites) or describe scene (for backgrounds)

### Post-Processing Pipeline

```
AI output (high-res, unconstrained palette)
  1. Scale down to target resolution (nearest-neighbor for pixel art, bicubic for cartoon)
  2. Reduce to target palette (Aseprite: Sprite > Color Mode > Indexed, load palette)
  3. Manual cleanup: fix artifacts, align outlines, adjust dithering
  4. Ensure consistency: compare with existing assets for proportion/style match
  5. Export as sprite sheet or individual frame
```

For complete prompt templates per era and asset type, see [references/ai-generation-prompts.md](references/ai-generation-prompts.md).

---

## 6. Room Background Creation

### Composition Principles

- **Rule of thirds**: Place focal points (doors, characters, key objects) at intersection points
- **Walkable area guidance**: Ground plane should clearly read as walkable; use value contrast to separate walkable floor from walls/obstacles
- **Depth cues**: Overlap objects, use value gradient (lighter = farther), scale reduction
- **Exit indicators**: Visually signal room exits (paths leading off-screen, doors, archways)
- **Hotspot clarity**: Interactive objects should contrast slightly with background — not hidden, but not glowing

### Parallax Layer Breakdown

For scrolling rooms (wider than viewport), split the background into depth layers:

```
Layer 5 (foreground):  Closest objects (vines, railing, lamp posts)
Layer 4 (main ground): Primary walkable area, furniture, NPCs
Layer 3 (mid-ground):  Buildings, trees at medium distance
Layer 2 (far bg):      Distant mountains, skyline
Layer 1 (sky):         Sky gradient, clouds, moon/sun
```

**Movement ratios** (relative to camera):
- Layer 5 (foreground): 1.2x-1.5x (moves faster than camera for parallax)
- Layer 4 (main ground): 1.0x (locked to camera)
- Layer 3: 0.6x-0.8x
- Layer 2: 0.3x-0.5x
- Layer 1 (sky): 0.0x-0.1x (nearly static)

### Room Sizing

| Era | Base viewport | Scrolling room width |
|-----|--------------|---------------------|
| MI1/MI2 | 320x200 | 320px to 960px wide (1-3 screens) |
| MI3 | 640x480 | 640px to 1920px wide (1-3 screens) |

### Walkable Area Mask

Create a separate monochrome image (white = walkable, black = blocked) at the same resolution as the room background. This mask feeds into Godot's `NavigationRegion2D` or Popochiu's walkable area system.

---

## 7. Character Design

### Proportions by Era

| Era | Head-to-body ratio | Character height | Hands | Detail level |
|-----|-------------------|-----------------|-------|-------------|
| MI1 (EGA/VGA) | ~1:3 | 32-40px | Mitten-shaped | Minimal; silhouette-driven |
| MI2 (VGA) | ~1:3.5 | 40-48px | Simple fingers | Moderate; visible features |
| MI3 (cartoon) | ~1:2.5 | 100-140px | Articulated fingers | High; full facial expressions |

### Walk Cycle Structure

**4-direction walk (MI1/MI2 pixel art):**
- Down, Left, Right (mirror of Left), Up
- 4-6 frames per direction: contact, passing, contact (opposite), passing (opposite)
- Arms swing opposite to legs; slight head bob (1px vertical)
- Left/Right sprites can be mirrored to save work — unless character has asymmetric features (eyepatch, sword side)

**8-direction walk (MI3 cartoon):**
- Down, Down-Left, Left, Up-Left, Up (and mirrors)
- 8-12 frames per direction with anticipation and follow-through
- Separate arm, body, and leg layers for smoother animation

### Talk Animation

- **Pixel art (MI1/MI2)**: 2-4 frames alternating open/closed mouth. Optional: arm gesture frame.
- **Cartoon (MI3)**: 4-8 frames with lip shapes (closed, open, wide, round) plus body movement. Head tilts and arm gestures on separate cycles.

### Expression Sheets (Portraits)

For dialog close-ups (MI3 style), create a portrait sheet:
- Neutral, Happy, Angry, Surprised, Sad, Thinking, Talking (2-3 mouth positions)
- Standard size: 128x128 or 160x160 pixels
- Consistent lighting direction across all expressions

---

## 8. Godot 4 Integration

### Project Settings for Pixel Art

```
Project > Project Settings:
  Display > Window:
    Viewport Width: 320  (MI1/MI2) or 640 (MI3)
    Viewport Height: 200 (MI1/MI2) or 480 (MI3)
    Stretch Mode: viewport
    Stretch Aspect: keep
    
  Rendering > Textures:
    Default Texture Filter: Nearest  (critical for pixel art — disables anti-aliasing on sprites)
```

For MI3 cartoon style, you may use `Linear` filtering instead of `Nearest` for smoother edges.

### AnimatedSprite2D Setup

1. Add `AnimatedSprite2D` node to your character scene
2. Create a new `SpriteFrames` resource in the Inspector
3. Open the SpriteFrames editor panel (bottom of editor)
4. Create named animations: `idle_down`, `walk_down`, `walk_left`, `talk_down`, etc.
5. For sprite sheets: click the grid icon, select your sheet, set frame grid (hframes x vframes)
6. Select frames for each animation, set FPS (8-10 for pixel, 12-15 for cartoon)
7. Toggle Loop for repeating animations (walk, idle); disable for one-shots (interact)

### Animation State Control (GDScript)

```gdscript
func update_animation(direction: Vector2, is_moving: bool, is_talking: bool):
    var dir_name = get_direction_name(direction)
    
    if is_talking:
        $AnimatedSprite2D.play("talk_" + dir_name)
    elif is_moving:
        $AnimatedSprite2D.play("walk_" + dir_name)
    else:
        $AnimatedSprite2D.play("idle_" + dir_name)

func get_direction_name(dir: Vector2) -> String:
    if abs(dir.x) > abs(dir.y):
        return "left" if dir.x < 0 else "right"
    else:
        return "up" if dir.y < 0 else "down"
```

### Parallax Background Setup

```
Scene tree:
  ParallaxBackground
    ParallaxLayer (sky)        → motion_scale = (0.1, 0.0)
    ParallaxLayer (far_bg)     → motion_scale = (0.3, 0.0)
    ParallaxLayer (mid_bg)     → motion_scale = (0.6, 0.0)
    ParallaxLayer (main_ground)→ motion_scale = (1.0, 1.0)
    ParallaxLayer (foreground) → motion_scale = (1.3, 0.0)
```

Each `ParallaxLayer` contains a `Sprite2D` with the layer image.

### Popochiu Addon

[Popochiu](https://github.com/carenalgas/popochiu) is a Godot addon specifically for point-and-click adventures (inspired by AGS and PowerQuest). It provides:
- Room, character, inventory item, and dialog management
- Built-in verb UI (9-verb, 2-click, Sierra-style)
- Walkable area system
- Save/load, audio, transitions
- Install via Godot AssetLib or clone from GitHub

### Walkable Areas

**With Popochiu:** Define walkable polygons directly in the room editor.

**Without Popochiu:** Use `NavigationRegion2D` with a hand-drawn polygon matching the walkable area mask. Characters use `NavigationAgent2D` for pathfinding.

For complete Godot integration guide, see [references/godot-integration.md](references/godot-integration.md).

---

## 9. Complete Asset Pipeline

### End-to-End Workflow

```
1. CONCEPT
   Paper sketches or digital thumbnails for rooms and characters
   Define art era (MI1/MI2/MI3) and lock palette

2. GENERATION
   Option A: Manual in Aseprite (full control, slower)
   Option B: AI-generate base → post-process in Aseprite (faster, needs cleanup)
   Option C: AI backgrounds + manual characters (hybrid, recommended)

3. POST-PROCESSING
   - Palette reduction to target (16/256 colors)
   - Dithering pass (manual or Floyd-Steinberg)
   - Consistency check (proportions, palette, outline weight)
   - Animation frame creation

4. SPRITE SHEET ASSEMBLY
   Aseprite export: PNG sheet + JSON metadata
   Naming convention: character_action_direction.png

5. GODOT IMPORT
   - Drag PNG + JSON into Godot project /assets/ folder
   - Create SpriteFrames resources
   - Configure animations (FPS, loop, frame selection)

6. SCENE INTEGRATION
   - Place backgrounds and parallax layers
   - Position walkable areas and hotspots
   - Wire up character animation states
   - Test walk cycles and interactions in-engine
```

### File Naming Convention

```
assets/
  characters/
    guybrush/
      guybrush_walk.png          (sprite sheet)
      guybrush_walk.json         (frame metadata)
      guybrush_idle.png
      guybrush_talk.png
      guybrush_portrait.png      (dialog close-up)
  rooms/
    melee_dock/
      melee_dock_bg.png          (main background or parallax layers)
      melee_dock_sky.png
      melee_dock_far.png
      melee_dock_mid.png
      melee_dock_fg.png
      melee_dock_walkable.png    (walkable area mask)
  items/
    rubber_chicken.png           (inventory icon)
  ui/
    cursor_walk.png
    cursor_look.png
    cursor_use.png
    cursor_talk.png
    verb_bar.png
    inventory_panel.png
    dialog_box.png
```

---

## 10. Quick Reference Tables

### Resolution and Scale Cheat Sheet

| Target Era | Viewport | Character Height | Walk Frames | Talk Frames | FPS |
|-----------|----------|-----------------|-------------|-------------|-----|
| MI1 (EGA) | 320x200 | 32-40px | 4-6 | 2-3 | 8-10 |
| MI2 (VGA) | 320x200 | 40-48px | 4-6 | 2-4 | 8-10 |
| MI3 (cartoon) | 640x480 | 100-140px | 8-12 | 4-8 | 12-15 |

### Tool Recommendations

| Task | Primary Tool | Alternative |
|------|-------------|-------------|
| Pixel art creation | Aseprite | Pyxel Edit, GIMP, GraphicsGale |
| Cartoon illustration | Krita, Clip Studio Paint | Procreate (iPad), Photoshop |
| AI generation (pixel) | Stable Diffusion (SDXL) + pixel LoRA | Flux |
| AI generation (cartoon) | DALL-E 3, Midjourney | Stable Diffusion |
| Palette editing | Aseprite palette editor | Lospec.com palette list |
| Sprite sheet packing | Aseprite export | TexturePacker, Shoebox |
| Game engine | Godot 4 + Popochiu | Adventure Game Studio (AGS) |

---

## 11. Additional References

- [Art Styles and Palettes](references/art-styles-and-palettes.md) — EGA/VGA/cartoon palette hex values, dithering patterns, palette cycling, era comparison
- [AI Generation Prompts](references/ai-generation-prompts.md) — Copy-paste prompt templates per era and asset type, model settings, post-processing
- [Godot Integration](references/godot-integration.md) — Full Godot 4 setup, AnimatedSprite2D, Popochiu, parallax, walkable areas, GDScript patterns
