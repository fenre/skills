# Art Styles and Color Palettes — Detailed Reference

## Era Comparison Table

| Attribute | MI1 (EGA/VGA) | MI2 (VGA) | MI3 (Cartoon) |
|-----------|--------------|-----------|---------------|
| Year | 1990 | 1991 | 1997 |
| Resolution | 320x200 | 320x200 | 640x480 |
| Pixel aspect | 1:1.2 (non-square) | 1:1.2 (non-square) | 1:1 (square) |
| Colors on screen | 16 (EGA) / 256 (VGA) | 256 | 256 |
| Character height | 32-40px | 40-48px | 100-140px |
| Art method | Pixel-by-pixel in Deluxe Paint | Pixel-by-pixel in Deluxe Paint | Drawn on paper, scanned, vectorized |
| Outline style | No outlines; color shapes only | No outlines; color shapes only | Bold dark outlines (2-3px) |
| Shading | Dithering + color ramps | Impressionist color blending | Flat fill + 2-3 cel-shade steps |
| Animation | Palette cycling, minimal frame anim | Palette cycling, moderate frame anim | Full cel animation (8-12+ frames) |
| Mood/lighting | Night-heavy, atmospheric | Varied (day, dusk, underground) | Bright, saturated, cartoon |
| Background style | Painterly with dithered gradients | Richer painterly, warmer tones | Illustrated, brushwork visible |

---

## EGA 16-Color Palette

The standard IBM EGA palette used in the original Secret of Monkey Island (EGA version):

| Index | Name | Hex | RGB | Usage Notes |
|-------|------|-----|-----|-------------|
| 0 | Black | `#000000` | 0, 0, 0 | Outlines, deep shadows, night sky |
| 1 | Blue | `#0000AA` | 0, 0, 170 | Dark water, night sky accents |
| 2 | Green | `#00AA00` | 0, 170, 0 | Dark foliage, swamp areas |
| 3 | Cyan | `#00AAAA` | 0, 170, 170 | Water highlights, moonlit surfaces |
| 4 | Red | `#AA0000` | 170, 0, 0 | Dark fabrics, blood, fire shadows |
| 5 | Magenta | `#AA00AA` | 170, 0, 170 | Dusk skies, magical effects, flowers |
| 6 | Brown | `#AA5500` | 170, 85, 0 | Wood, earth, leather, hair |
| 7 | Light Gray | `#AAAAAA` | 170, 170, 170 | Stone, metal, neutral surfaces |
| 8 | Dark Gray | `#555555` | 85, 85, 85 | Secondary shadows, stone details |
| 9 | Light Blue | `#5555FF` | 85, 85, 255 | Bright water, sky highlights |
| 10 | Light Green | `#55FF55` | 85, 255, 85 | Bright foliage, tropical plants |
| 11 | Light Cyan | `#55FFFF` | 85, 255, 255 | Water reflections, moonlight |
| 12 | Light Red | `#FF5555` | 255, 85, 85 | Fire, lava, warm light |
| 13 | Light Magenta | `#FF55FF` | 255, 85, 255 | Bright magic, sunset accents |
| 14 | Yellow | `#FFFF55` | 255, 255, 85 | Gold, sunlight, candle glow, text |
| 15 | White | `#FFFFFF` | 255, 255, 255 | Highlights, text, brightest points |

### EGA Art Tips

- **Dithering is essential** — with only 16 colors, smooth gradients are impossible without mixing
- **Checkerboard dithering** between adjacent colors creates ~120 effective color combinations
- **Night scenes dominate** because Black + Blue + Cyan + Dark Gray give 4 colors for shadows/atmosphere, leaving 12 for actual content
- **Palette cycling works on index slots** — animate by rotating which RGB values occupy indices 1-3 (water effect) without redrawing pixels

---

## VGA 256-Color Palette Construction

VGA games used custom palettes per scene (or globally with scene-specific sub-ranges). Build palettes using **color ramps** — graduated sequences from dark to light for each material.

### Color Ramp Structure

A typical ramp has 4-8 entries:

```
Darkest shadow → Shadow → Midtone → Light → Highlight → Specular
```

**Crucial principle:** Ramps should shift hue, not just brightness:
- Shadows shift toward **cool (blue/purple)**
- Highlights shift toward **warm (yellow/white)**
- This creates life and depth that pure brightness ramps lack

### Example VGA Palette Organization (256 colors)

```
Indices 0-7:     UI & system (black, white, cursor colors, text)
Indices 8-23:    Skin tones (2 ramps: fair and tanned, 8 colors each)
Indices 24-39:   Earth tones (wood, dirt, stone — 2 ramps of 8)
Indices 40-55:   Greens (foliage, jungle — 2 ramps of 8)
Indices 56-71:   Blues (sky, water — 2 ramps of 8)
Indices 72-83:   Reds/oranges (fire, clothing — 12 colors)
Indices 84-95:   Purples/magentas (night, magic — 12 colors)
Indices 96-111:  Warm neutrals (sand, rope, parchment — 16 colors)
Indices 112-127: Cool neutrals (metal, shadows — 16 colors)
Indices 128-159: Scene-specific colors (tropical, cave, town — 32 colors)
Indices 160-191: Palette cycling ranges (water: 8, fire: 8, misc: 16)
Indices 192-239: Secondary materials and props (48 colors)
Indices 240-255: Reserved for effects and transparency (16 colors)
```

### Impressionist Color Blending (MI2 Technique)

Used by artists at LucasArts for MI2's VGA art:

1. Place two **complementary colors** adjacent to each other at the pixel level
2. The viewer's eye blends them at a distance, perceiving a third color
3. Example: alternate blue and orange pixels → eye sees a warm gray-brown
4. Works because CRT monitors softened pixel edges naturally
5. Creates richer, more vibrant surfaces than using a single "mixed" color from the palette

**Best for:** Skin tones, fabric folds, tropical vegetation, stone/wood grain

---

## MI3 Cartoon Color Theory

### Approach

MI3 uses a **flat fill + cel shading** approach rather than pixel-level dithering:

1. **Base fill**: Solid color for each area (skin, coat, pants, etc.)
2. **Shadow**: One darker shade per material, applied as a flat shape (not gradient)
3. **Deep shadow**: Optional third darkest shade for creases, under-chin, etc.
4. **Highlight**: One lighter shade for rim lighting or light-facing surfaces
5. **Outline**: Dark (not always pure black) line art at consistent weight

### Color Harmony Patterns

| Pattern | Use | Example |
|---------|-----|---------|
| **Complementary** | High contrast, vibrant scenes | Blue sky + orange sunset |
| **Analogous** | Cohesive, moody scenes | Green jungle: yellow-green, green, teal |
| **Split complementary** | Rich but balanced | Purple night + yellow-green + orange lights |
| **Triadic** | Cartoon energy | Red character + blue water + yellow treasure |

### Outline Color Guidelines

- **Characters**: Near-black (`#1A1A2E` or `#2D1B2E`) — slightly warm or cool tinted
- **Warm objects**: Dark brown outlines (`#3D1F00`)
- **Cool objects**: Dark blue outlines (`#0A1628`)
- **Foreground**: Heavier outlines (2-3px at 640x480)
- **Background**: Thinner or no outlines (1px or just color edges)

### Line Weight for Depth

- **Closest objects** (characters, foreground props): Thickest lines
- **Mid-ground**: Medium lines
- **Background elements**: Thinnest lines or no outlines
- This simulates atmospheric perspective and helps readability

---

## Dithering Techniques — Detailed Reference

### Checkerboard Dithering

The most common retro technique. Alternating pixels of two colors in a grid:

```
A B A B A B
B A B A B A
A B A B A B
```

**When to use:** Large areas needing a 50/50 blend (sky gradients, water surfaces, large shadow areas)

### Ordered Dithering (Bayer Matrix)

Uses a structured threshold pattern. Pixels are assigned to color A or B based on position within a repeating matrix:

**2x2 Bayer:**
```
0 2
3 1
```

**4x4 Bayer:**
```
 0  8  2 10
12  4 14  6
 3 11  1  9
15  7 13  5
```

Threshold each pixel: if brightness > matrix value at that position, use the lighter color.

**When to use:** Smooth gradients with a retro feel (skies, water, fog). Produces more structured patterns than checkerboard.

### Diagonal/Line Dithering

```
A A B A A B
A B A A B A
B A A B A A
```

**When to use:** Directional shading — angle the lines to suggest surface direction (vertical for walls, horizontal for floors).

### Stipple/Random Dithering

Randomly placed pixels of two colors with controlled density.

**When to use:** Organic textures (sand, gravel, bark, dirt). Avoid for geometric surfaces.

### Floyd-Steinberg Error Diffusion

An algorithm (not a hand-drawn pattern) that distributes quantization error to neighboring pixels. Applied automatically when reducing an image to a limited palette.

**When to use:** Post-processing AI-generated art or high-color source images down to a retro palette. Apply in Aseprite (Sprite > Color Mode > Indexed > select palette) or GIMP (Image > Mode > Indexed > Floyd-Steinberg dithering).

---

## Palette Cycling — Implementation Details

Palette cycling animates scenes by rotating color index values without changing any pixel data. The illusion of motion comes from shifting which RGB colors are assigned to specific palette indices.

### Water Cycling

```
Frame 1: Index 160=#0044AA, 161=#0055BB, 162=#0066CC, 163=#0077DD, 164=#0088EE, 165=#0077DD, 166=#0066CC, 167=#0055BB
Frame 2: Index 160=#0055BB, 161=#0066CC, 162=#0077DD, 163=#0088EE, 164=#0077DD, 165=#0066CC, 166=#0055BB, 167=#0044AA
(rotate one step per frame)
```

- Use 6-8 indices for water
- Rotate forward for flowing water, ping-pong for gentle waves
- Speed: 4-8 shifts per second for calm water, 10-15 for rapids

### Fire Cycling

```
Indices: 168-175 (8 colors)
Colors: dark red → red → orange → yellow → white (peak) → yellow → orange → red
```

- Rotate at 8-12 shifts/second for flickering effect
- Offset multiple fire sources by 2-3 indices for natural variation
- Place fire pixels as a mix of these indices in an irregular pattern

### Sunset/Sunrise Cycling

- Use two ranges simultaneously: sky indices (8 colors) and water reflection indices (8 colors)
- Slowly rotate both ranges in sync (1-2 shifts/second)
- Sky range transitions: deep blue → purple → orange → pink → light blue
- Mark Ferrari's technique: a single background image produces day-to-night transition purely through palette rotation

### Implementation in Godot

Palette cycling requires shader-based implementation in modern engines. In Godot 4:

```gdscript
# Apply a palette cycling shader to a Sprite2D
# The shader maps indexed colors and rotates UV coordinates on a palette texture

# Shader approach:
# 1. Store your art as a grayscale index map (pixel value = palette index)
# 2. Use a 256x1 palette texture (lookup table)
# 3. Shift the lookup offset each frame to simulate cycling
```

Alternatively, for simpler scenes, swap `Sprite2D` textures at timed intervals with pre-baked palette variations.

---

## Aseprite Palette File Formats

### GPL (GIMP Palette) Format

```
GIMP Palette
Name: MI1 EGA
Columns: 16
#
  0   0   0	Black
  0   0 170	Blue
  0 170   0	Green
  0 170 170	Cyan
170   0   0	Red
170   0 170	Magenta
170  85   0	Brown
170 170 170	Light Gray
 85  85  85	Dark Gray
 85  85 255	Light Blue
 85 255  85	Light Green
 85 255 255	Light Cyan
255  85  85	Light Red
255  85 255	Light Magenta
255 255  85	Yellow
255 255 255	White
```

Save as `.gpl` and load in Aseprite via Palette > Load Palette.

### PAL (JASC) Format

```
JASC-PAL
0100
16
0 0 0
0 0 170
0 170 0
0 170 170
170 0 0
170 0 170
170 85 0
170 170 170
85 85 85
85 85 255
85 255 85
85 255 255
255 85 85
255 85 255
255 255 85
255 255 255
```

### Lospec Palette Resources

Community-curated retro palettes at https://lospec.com/palette-list:
- Search "EGA" for EGA-compatible palettes
- Search "VGA" for 256-color retro palettes
- Search "Monkey Island" for fan-made MI-specific palettes
- Download in `.gpl`, `.pal`, `.png`, or `.hex` formats

---

## Practical Palette Tips

1. **Design your palette before drawing** — changing palettes mid-production causes inconsistencies
2. **Limit ramp count, not color count** — 16 ramps of 8 colors is easier to manage than 128 random colors
3. **Test palette in context** — a color ramp that looks great in isolation may not work when sky meets water meets foliage
4. **Keep a palette swatch sheet** — a reference image showing all ramps labeled by material, used by all artists
5. **Reserve cycling ranges** — mark indices 160-191 (or similar) for animation; never use them for static art
6. **Warm shadows, cool highlights (or vice versa)** — never use pure gray; shift hue for life
7. **Transparent color**: In indexed mode, index 0 is typically transparent for sprites. Place your transparency color (usually magenta `#FF00FF` or designated color) at index 0.
