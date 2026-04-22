# Environment Art, UI/HUD & Visual Effects

## Tileset Design

### Tile Size Selection

| Game Type | Tile Size | Notes |
|-----------|----------|-------|
| Retro platformer | 8x8, 16x16 | Classic NES/SNES feel |
| Modern platformer | 16x16, 32x32 | Room for detail |
| Top-down RPG | 16x16 | Standard for overworld and dungeon |
| Isometric | 32x16, 64x32 | 2:1 width-to-height ratio |
| Strategy | 16x16, 32x32 | Must read when zoomed out |

### Core Tileset Components

Every environment tileset needs at minimum:

```
Ground/Floor tiles:
├── Interior (fully surrounded)
├── Edges (top, bottom, left, right)
├── Outer corners (4 corners where edges meet)
├── Inner corners (4 corners for concave shapes)
└── Variants (2–3 visual variations of interior for organic look)

Plus:
├── Transition tiles (grass-to-dirt, stone-to-wood)
├── Decorative overlays (flowers, cracks, moss)
└── Special tiles (doors, ladders, chests, signs)
```

### Designing Tileable Tiles

Rules for tiles that connect seamlessly:

1. **Edges must match**: Left edge of tile A must pixel-match right edge of tile B
2. **Avoid center-focused patterns**: Details that draw the eye to tile centers create visible grid
3. **Break the grid**: Use decorative overlay tiles that span 2x1 or 2x2 to disguise the grid
4. **Vary the interior**: 2–4 interior variants placed randomly break repetition
5. **Test in a grid**: Always test tiles in a real layout, not in isolation

### Autotiling Systems

**16-tile (4-bit bitmask)**: Checks 4 cardinal neighbors. Produces blocky edges without inner corners. Simple to implement. Good for retro/grid-based styles.

**47-tile (8-bit bitmask)**: Checks all 8 neighbors (cardinal + diagonal). Produces smooth edges with proper inner corners. Industry standard for modern 2D games. Supported natively in Godot, LDtk, and most modern engines.

Implementation:
1. Design the fully-surrounded interior tile first
2. Use a bitmask template generator to see all required variants
3. Draw edges systematically (top, right, bottom, left)
4. Draw outer corners, then inner corners
5. Draw isolated and end-cap tiles
6. Configure bitmask values in your engine/editor

### Tileset Palette Discipline

- Share palette with the game's global palette
- Each tileset biome gets its own color emphasis (forest = greens, cave = grays/browns)
- Background tiles: lower saturation, less contrast
- Foreground/interactive tiles: higher saturation, more contrast
- Hazards: use colors that signal danger (lava = orange/red glow)

## Background & Parallax Design

### Layer Structure

A standard parallax background uses 3–5 layers:

```
Layer 0 (farthest): Sky / gradient / static color
Layer 1:            Distant landscape (mountains, clouds)
Layer 2:            Mid-ground (hills, treeline, buildings)
Layer 3:            Near background (fences, bushes, close structures)
Layer 4 (closest):  Foreground overlay (particles, fog, vines)
```

### Parallax Speed Ratios

| Layer | Scroll Speed | Example |
|-------|-------------|---------|
| Sky | 0x (static) or 0.05x | Gradient, stars |
| Far background | 0.1x–0.2x | Mountains, distant city |
| Mid background | 0.3x–0.5x | Hills, forest |
| Near background | 0.6x–0.8x | Fences, nearby trees |
| Gameplay layer | 1.0x | Player, enemies, tiles |
| Foreground overlay | 1.2x–1.5x | Fog, particles, lens effects |

### Background Art Tips

- **Decrease detail with distance**: Far layers are simpler with fewer colors
- **Atmospheric perspective**: Far layers shift toward blue/purple and lower contrast
- **Tile horizontally**: Background layers should tile seamlessly horizontally for infinite scroll
- **Limit vertical parallax**: In platformers, vertical parallax can cause disorientation — use sparingly
- **Time of day**: Design backgrounds in neutral tone, then tint with color overlays for dawn/dusk/night

## Lighting Techniques (2D)

### Faux Lighting Methods

Since 2D games lack real light calculation, simulate lighting with:

**Color overlays**: Multiply-blend a dark blue/purple layer over the scene for nighttime. Mask the overlay around light sources.

**Sprite-based lights**: Radial gradient sprites (additive blend) placed at light sources. Simple and performant.

**Normal map lighting**: For advanced 2D lighting, pair sprite color maps with normal maps. Engines like Unity and Godot can compute per-pixel lighting on 2D sprites using normal data.

**Shadow sprites**: Pre-drawn shadow shapes placed beneath characters and objects. Offset based on light direction.

### Light Source Types for 2D

| Source | Visual Treatment |
|--------|-----------------|
| Sun/ambient | Global tint; consistent shadow direction |
| Torch/fire | Warm radial glow; flicker animation (random opacity/size) |
| Magic | Saturated radial glow matching magic color |
| Neon/tech | Sharp-edged glow with bloom bleed |
| Underwater | Caustic overlay patterns; blue-green tint |

## Particle & VFX Design

### Particle Building Blocks

Most game particles are built from simple shapes:

```
●  Circle/dot     — sparks, dust, bubbles, rain
★  Star           — magic, sparkle, impact flash
─  Line/streak    — speed lines, rain, tracers
◆  Diamond        — crystals, ice, shatter
▓  Square         — debris, confetti, smoke chunks
```

### Common VFX Recipes

**Hit Impact**: 3–5 streak particles radiating outward + 1 frame white flash + screen shake

**Explosion**: Expanding circle sprite (3–5 frames) + radial spark particles + smoke cloud (fading circles) + debris particles (gravity-affected)

**Dust Cloud**: 3–5 small circles spawned at feet, expand and fade over 10–15 frames. Color matches ground.

**Trail Effect**: Spawn fading copies of the sprite behind the character at regular intervals (ghost trail) or draw a shrinking line following the movement path.

**Heal/Buff**: Rising sparkle particles + expanding ring sprite + brief green/golden tint on character

**Fire**: Layered approach — yellow core (small, fast), orange mid (medium, medium speed), red outer (large, slow). Particles rise and shrink. Add glow overlay.

**Water Splash**: Upward arc particles + expanding ripple ring on water surface

### VFX Timing

- Impact effects: 3–8 frames (instant, punchy)
- Ambient effects: Continuous loop (fire, rain, sparkle)
- Transition effects: 15–30 frames (portal opening, teleport)
- UI feedback: 5–15 frames (button press, item collect)

## UI/HUD Design

### HUD Principles

1. **Minimal footprint**: HUD should occupy < 15% of screen area
2. **Consistent placement**: Health always in same spot; don't move UI during gameplay
3. **Readable at speed**: Players glance at HUD — use color and shape, not text, for status
4. **Match game style**: Pixel art game = pixel art UI. Don't mix hi-res UI with pixel gameplay
5. **Consider safe zones**: Keep critical HUD elements away from screen edges (mobile notches, TV overscan)

### Health Bar Design Patterns

| Style | Look | Best For |
|-------|------|----------|
| **Segmented bar** | Divided into discrete chunks | Action games, clear damage feedback |
| **Continuous bar** | Smooth fill | RPGs, bosses with large HP pools |
| **Hearts/icons** | Discrete units (Zelda-style) | Low max HP, easy to count |
| **Radial/ring** | Circular fill | Minimal UI, stamina gauges |
| **Boss bar** | Full-width at top/bottom | Boss encounters |

Health bar features:
- Red fill on colored background (or green → yellow → red gradient)
- Delayed drain (white "ghost" bar showing recent damage)
- Flash/shake on damage
- Outline that matches UI frame style

### Inventory & Menu Design

**Grid inventory** (Diablo-style): Items as icons in grid slots
- Slot states: empty, occupied, selected, locked
- Item quality indicated by border color (gray/green/blue/purple/gold)
- Tooltip on hover/select showing item name, stats, description

**List inventory** (classic RPG): Text list with small icons
- Category tabs or scrollable list
- Equipped items highlighted
- Quantity display for stackable items

### Menu Screen Layout

```
┌─────────────────────────────────┐
│          GAME TITLE             │  ← Centered, largest text
│                                 │
│         ▶ New Game              │  ← Menu items centered or left-aligned
│           Continue              │
│           Settings              │
│           Credits               │
│                                 │
│                  v1.0           │  ← Version info, small, bottom corner
└─────────────────────────────────┘
```

Menu design rules:
- Clear selected item indicator (arrow, highlight, color change, animation)
- Consistent navigation (D-pad, arrow keys, mouse all work)
- Smooth transitions between screens (slide, fade)
- Sound feedback on navigate and select

### Text & Typography in Pixel Art Games

For pixel art games, use pixel fonts that match your resolution:
- **5x5 minimum**: Readable uppercase only
- **5x7**: Full alphanumeric with lowercase
- **8x8**: Comfortable reading, good for dialogue
- **Custom drawn**: Match your game's specific style

Rules:
- Integer scaling only (don't filter pixel fonts)
- Consistent font across all UI (one font for body, optionally one for titles)
- Adequate contrast (light text on dark panel or vice versa)
- Line spacing of at least 1–2 px between lines

## UI Panel & Frame Construction

### 9-Slice/9-Patch Panels

Build resizable UI panels from 9 parts:

```
┌──┬────────┬──┐
│TL│  Top   │TR│    Corners: fixed size
├──┼────────┼──┤    Edges: tile/stretch in one direction
│ L│ Center │ R│    Center: tile/stretch in both directions
├──┼────────┼──┤
│BL│ Bottom │BR│
└──┴────────┴──┘
```

Design the corner and edge tiles once; they scale to any panel size. Most engines support 9-slice natively.

### UI Color Palette

- **Panel background**: Dark, low-saturation (doesn't compete with game visuals)
- **Text**: High contrast against panel (white or light yellow on dark panels)
- **Highlight/selected**: Saturated accent color (gold, blue, or game's primary color)
- **Disabled/inactive**: Desaturated, lower contrast version of normal state
- **Danger/warning**: Red tint or red accent for destructive actions

## Art Asset Organization

### File Naming Convention

```
character_hero_idle_right_32x32.png
character_hero_walk_right_32x32.png
enemy_slime_idle_16x16.png
tile_forest_ground_16x16.png
ui_healthbar_full_64x8.png
vfx_explosion_48x48.png
bg_forest_layer1_far_320x180.png
```

Pattern: `category_name_action_direction_dimensions.extension`

### Folder Structure

```
assets/
├── characters/
│   ├── hero/
│   │   ├── hero_idle.png
│   │   ├── hero_walk.png
│   │   └── hero_attack.png
│   └── enemies/
│       ├── slime.png
│       └── skeleton.png
├── tiles/
│   ├── forest/
│   ├── cave/
│   └── castle/
├── backgrounds/
│   ├── forest/
│   └── cave/
├── ui/
│   ├── hud/
│   ├── menus/
│   └── icons/
├── vfx/
│   ├── particles/
│   └── impacts/
└── fonts/
```

### Asset Production Checklist

- [ ] Art style guide completed and followed
- [ ] All sprites at consistent resolution
- [ ] Palette shared across asset categories
- [ ] All animations exported as sprite sheets with metadata
- [ ] UI elements use 9-slice for resizable panels
- [ ] Backgrounds designed for parallax with correct layer ordering
- [ ] Tilesets tested in actual level layouts
- [ ] All assets named consistently
- [ ] Final exports are PNG with alpha (no lossy compression on pixel art)
