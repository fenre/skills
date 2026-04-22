# AI Art Generation Prompts — Adventure Game Assets

## Model Recommendations

| Target Style | Best Model | Settings | Notes |
|-------------|-----------|----------|-------|
| MI1 EGA pixel art | SDXL + pixel art LoRA | Steps: 30-40, CFG: 7-9, Sampler: DPM++ SDE Karras | Specify "16-color EGA" or "limited palette" |
| MI2 VGA pixel art | SDXL + pixel art LoRA | Steps: 30-40, CFG: 7-9, Sampler: DPM++ SDE Karras | Specify "VGA 256-color pixel art" |
| MI3 cartoon | DALL-E 3 | Standard quality, natural style | Best for clean line art and cartoon cel shading |
| MI3 cartoon (alt) | Midjourney v6+ | --style raw, --ar 4:3 | Good for painterly backgrounds |
| MI3 cartoon (alt) | SDXL | Steps: 40-50, CFG: 6-8 | Use with cartoon/cel-shade LoRA |
| Backgrounds (any era) | Flux or SDXL | Higher resolution, then downscale | Backgrounds benefit from detail at high res |

---

## Prompt Formula

### Five-Part Structure

Every prompt should include these five components:

```
[1. Art style and era] [2. Subject description] [3. Pose and framing] [4. Color/palette] [5. Background]
```

### Negative Prompts Library

**For pixel art (all eras):**
```
blurry, anti-aliased, smooth gradients, realistic, photorealistic, 3D render, high resolution, modern art style, watercolor, oil painting, pencil sketch, text, watermark, signature, border, frame, UI elements, HUD
```

**For cartoon/MI3 style:**
```
pixel art, pixelated, low resolution, blurry, photorealistic, 3D render, anime style, manga, chibi, dark gritty, horror, text, watermark, signature, border, frame
```

---

## Character Sprite Prompts

### MI1 EGA Style — Character

```
16-bit pixel art, EGA palette, 16 colors, retro adventure game character sprite,
[CHARACTER DESCRIPTION: e.g., young pirate man with blond ponytail, white shirt, brown pants],
idle standing pose, side view, full body,
limited color palette, dithered shading,
plain white background, single character, no scenery
```

**Negative:** `blurry, anti-aliased, smooth gradients, realistic, photorealistic, 3D, high resolution, multiple characters, text, watermark`

### MI2 VGA Style — Character

```
VGA pixel art, 256-color palette, 1990s LucasArts adventure game style,
[CHARACTER DESCRIPTION: e.g., pirate captain with blue coat, large beard, tricorn hat, peg leg],
idle standing pose, front-facing three-quarter view, full body,
rich warm color palette, detailed pixel shading, no outlines,
plain white background, single character
```

**Negative:** `blurry, anti-aliased, smooth gradients, realistic, photorealistic, 3D, modern style, outlines, cel-shaded, text, watermark`

### MI3 Cartoon Style — Character

```
Hand-drawn cartoon character design, cel-shaded, adventure game style similar to 1990s LucasArts animation,
[CHARACTER DESCRIPTION: e.g., tall lanky pirate with exaggerated features, big nose, confident grin, blue coat over white shirt],
idle standing pose, three-quarter view, full body,
bold dark outlines, flat colors with simple shadow shapes, Disney-influenced proportions,
plain white background, character design sheet, single character
```

**Negative:** `pixel art, pixelated, realistic, 3D, anime, manga, dark gritty, text, watermark, background scenery`

### Character Sheet (Consistency Reference)

Generate a reference sheet first, then use it for consistent individual poses:

```
Character design reference sheet, multiple views, front view, side view, three-quarter view, back view,
[STYLE TAG: "VGA pixel art" or "cartoon cel-shaded adventure game"],
[CHARACTER DESCRIPTION],
consistent proportions and colors across all views,
plain white background, turnaround sheet, character model sheet
```

---

## Walk Cycle Prompts

### Pixel Art Walk Cycle

```
Pixel art sprite sheet, [VGA 256-color / EGA 16-color] adventure game style,
[CHARACTER DESCRIPTION],
walk cycle animation frames, 6 frames, side view walking left to right,
consistent character design across all frames,
plain white background, horizontal strip layout, evenly spaced frames
```

### Cartoon Walk Cycle

```
2D animation walk cycle sprite sheet, cartoon cel-shaded style, adventure game character,
[CHARACTER DESCRIPTION],
8 frames of walking animation, side profile view,
bold outlines, flat colors, consistent character design per frame,
plain white background, horizontal strip layout
```

**Post-processing required:** AI-generated walk cycles rarely maintain consistent proportions across frames. Plan to use these as rough keyframe references, then refine manually in Aseprite.

---

## Room Background Prompts

### MI1 EGA Style — Background

```
16-bit pixel art background, EGA style, 16-color limited palette, retro adventure game scene,
[SCENE DESCRIPTION: e.g., moonlit Caribbean dock with wooden pier, anchored pirate ship, palm trees, starry night sky],
wide establishing shot, no characters, atmospheric lighting,
dithered shading, 320x200 pixel art style, moody night scene
```

### MI2 VGA Style — Background

```
VGA pixel art background, 256-color palette, 1990s LucasArts adventure game,
[SCENE DESCRIPTION: e.g., tropical island town square at sunset, colorful colonial buildings, hanging lanterns, cobblestone street],
wide establishing shot, no characters,
rich warm palette, painterly pixel art, detailed environment,
320x200 pixel art style, atmospheric
```

### MI3 Cartoon Style — Background

```
Hand-painted cartoon background, adventure game environment, 1990s animated film style,
[SCENE DESCRIPTION: e.g., pirate tavern interior, wooden beams, candle-lit tables, fireplace, bottles on shelves, treasure map on wall],
wide establishing shot, no characters,
rich color palette, visible brushwork, warm lighting with cool shadows,
detailed environment illustration, whimsical cartoon style
```

### Parallax Layer Backgrounds

Generate each layer separately with transparency in mind:

**Sky layer:**
```
[STYLE TAG], sky only, no ground, no foreground,
[SKY DESCRIPTION: e.g., sunset gradient from deep purple to warm orange with scattered clouds],
wide horizontal composition, seamless if possible,
plain transparent bottom edge
```

**Far background layer:**
```
[STYLE TAG], distant landscape silhouette,
[DESCRIPTION: e.g., mountain range and distant forest tree line],
horizontal panoramic, bottom half transparent,
atmospheric perspective, muted cool colors, low detail
```

**Mid-ground layer:**
```
[STYLE TAG], medium-distance environment elements,
[DESCRIPTION: e.g., row of colonial buildings with balconies and tiled roofs],
bottom portion transparent for foreground, moderate detail
```

---

## Inventory Item Prompts

### Pixel Art Items

```
VGA pixel art icon, 256-color palette, adventure game inventory item,
[ITEM DESCRIPTION: e.g., rubber chicken with a pulley in the middle],
single object centered, clear readable silhouette at small size,
32x32 pixels style, detailed but readable, slight shadow beneath,
plain transparent background
```

### Cartoon Items

```
Cartoon illustration, adventure game inventory icon, cel-shaded style,
[ITEM DESCRIPTION: e.g., golden key with ornate skull-shaped handle],
single object centered, bold dark outline, flat colors with simple highlight,
clean simple design readable at 64x64 pixels,
plain transparent background
```

---

## Dialog Portrait Prompts

### MI3 Style Portraits

```
Cartoon character portrait, adventure game dialog box style, close-up face and shoulders,
[CHARACTER DESCRIPTION: e.g., grizzled old pirate with eye patch, wild gray beard, weathered tan skin, mischievous grin],
front-facing or three-quarter view, expressive face,
bold outlines, flat cel-shaded colors, warm lighting,
square composition, simple neutral background
```

**Expression variations** — generate a base portrait, then modify the prompt:
- Replace expression: "neutral expression" / "angry scowl" / "surprised wide eyes" / "sly smirk" / "sad frown"
- Use seed locking (same seed) with only the expression changed for consistency

---

## UI Element Prompts

### Cursor Icons

```
Pixel art cursor icon set, adventure game UI, VGA style,
arrow cursor, magnifying glass (look), hand (use), speech bubble (talk), walking boots (walk),
each icon 16x16 or 24x24 pixels, clear silhouettes,
plain transparent background, icons arranged in a row
```

### Verb Bar / Interaction UI

```
[STYLE TAG] adventure game verb interface,
row of action buttons: Open, Close, Push, Pull, Give, Use, Look At, Pick Up, Talk To,
retro adventure game UI design, readable text, consistent button style,
horizontal bar layout
```

---

## Cutscene Illustration Prompts

### Dramatic Scene

```
[STYLE TAG: pixel art or cartoon],
dramatic cinematic scene, adventure game cutscene illustration,
[SCENE DESCRIPTION: e.g., pirate ship battle under stormy skies, cannon fire, lightning, waves crashing against hull],
wide cinematic composition, dramatic lighting, high contrast,
no UI elements, no text, full-screen illustration
```

---

## Consistency Techniques

### Seed Locking (Stable Diffusion)

When generating multiple assets for the same character:
1. Generate the best base image, note its **seed** value
2. Reuse the same seed for pose/expression variations
3. Change only the pose/expression description in the prompt
4. Keep all other parameters identical (model, CFG, steps, sampler)

### Style Consistency Across Assets

1. **Use the same style prefix** for all prompts in a project (copy-paste your style tag)
2. **Generate a style reference sheet** first: a collage of 4-6 elements (character, building, tree, item, sky) in your target style
3. **Use img2img** with your reference sheet as input for subsequent generations
4. **Consider LoRA/fine-tuning** if producing 50+ assets — train a small LoRA on 10-20 hand-approved reference images

### Character Consistency Checklist

After generating each character asset:
- [ ] Proportions match reference sheet (head size, body length, limb proportions)
- [ ] Color palette matches (compare skin tone, clothing colors side by side)
- [ ] Outline weight is consistent (if MI3 cartoon style)
- [ ] Detail level matches other assets in the same scene
- [ ] Character is recognizable at game-scale resolution

---

## Post-Processing Pipeline

### Step 1: Scale to Target Resolution

| Source | Target | Method |
|--------|--------|--------|
| AI output (1024x1024) | 320x200 pixel art | Nearest-neighbor downscale in Aseprite/GIMP |
| AI output (1024x1024) | 640x480 cartoon | Bicubic downscale, then sharpen edges |
| AI output (1024x1024) | Sprite (48px tall) | Nearest-neighbor to exact pixel dimensions |

### Step 2: Palette Reduction

**In Aseprite:**
1. Open the scaled image
2. Load your target palette (Palette > Load Palette)
3. Sprite > Color Mode > Indexed
4. Select dithering algorithm:
   - **None**: Flat reduction (best for cartoon/MI3)
   - **Ordered**: Structured dithering (good for MI1/MI2)
   - **Old**: Aseprite's legacy algorithm (sometimes preferred for pixel art)
5. Review and manually fix any problem areas

**In GIMP:**
1. Image > Mode > Indexed
2. Select "Use custom palette" and load your palette
3. Choose Floyd-Steinberg dithering (or None for cartoon)
4. Colors > Map > Rearrange Colormap if needed

### Step 3: Manual Cleanup

- Fix broken outlines (dithering can fragment character edges)
- Ensure key silhouette features are intact (face, hands, distinctive props)
- Align pixel grid if the downscale produced sub-pixel offsets
- Verify transparent background is clean (no stray pixels)

### Step 4: Consistency Pass

- Place new asset alongside existing assets in a test composition
- Check: color harmony, scale consistency, outline weight, detail level
- Adjust if anything feels out of place

### Step 5: Export

- **Sprites**: Export individual frames or sprite sheets from Aseprite (PNG + JSON)
- **Backgrounds**: Export as single PNG per layer (or combined for non-parallax)
- **Items**: Export as individual transparent PNGs at icon size

---

## Batch Prompting Strategy

When generating a full room's worth of assets:

1. **Background first** — this sets the color palette and mood for everything else
2. **Key characters** — main character and primary NPCs for the room
3. **Props and interactive objects** — ensure they read clearly against the background
4. **Inventory items** — smaller, simpler; generate in batches of 4-6
5. **UI elements** — last; match the overall art style

Generate 3-5 variations of each asset. Select the best, then post-process. Budget approximately:
- 10-20 generations per background (to get one good result)
- 5-10 per character pose (then heavy manual refinement)
- 3-5 per inventory item
- 3-5 per UI element
