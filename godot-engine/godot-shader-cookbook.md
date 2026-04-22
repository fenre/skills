# Godot 4 — Shader Cookbook

Copy-paste shader examples for common 2D and 3D effects.

## Shader Types

| Type | Keyword | Use |
|------|---------|-----|
| CanvasItem | `shader_type canvas_item;` | 2D sprites, UI, tilemaps |
| Spatial | `shader_type spatial;` | 3D meshes |
| Particles | `shader_type particles;` | Custom particle behavior |
| Sky | `shader_type sky;` | Procedural sky |
| Fog | `shader_type fog;` | Volumetric fog |

---

## Hit Flash (2D)

White flash on damage. Control `flash_amount` from GDScript (0.0 → 1.0 → tween back).

```glsl
shader_type canvas_item;

uniform vec4 flash_color : source_color = vec4(1.0);
uniform float flash_amount : hint_range(0.0, 1.0) = 0.0;

void fragment() {
    vec4 tex = texture(TEXTURE, UV);
    COLOR = mix(tex, flash_color, flash_amount);
    COLOR.a = tex.a;
}
```

```gdscript
# From GDScript:
func flash() -> void:
    material.set_shader_parameter("flash_amount", 1.0)
    var t := create_tween()
    t.tween_property(material, "shader_parameter/flash_amount", 0.0, 0.2)
```

## Outline (2D)

Draws a colored border around sprite (requires transparent padding around the sprite).

```glsl
shader_type canvas_item;

uniform vec4 outline_color : source_color = vec4(0.0, 0.0, 0.0, 1.0);
uniform float outline_width : hint_range(0.0, 10.0, 1.0) = 1.0;

void fragment() {
    vec4 tex = texture(TEXTURE, UV);
    if (tex.a < 0.5) {
        vec2 size = TEXTURE_PIXEL_SIZE * outline_width;
        float a = texture(TEXTURE, UV + vec2(-size.x, 0)).a;
        a += texture(TEXTURE, UV + vec2(size.x, 0)).a;
        a += texture(TEXTURE, UV + vec2(0, -size.y)).a;
        a += texture(TEXTURE, UV + vec2(0, size.y)).a;
        if (a > 0.0) {
            COLOR = outline_color;
        } else {
            COLOR = tex;
        }
    } else {
        COLOR = tex;
    }
}
```

## Dissolve / Burn (2D)

Burn-away effect using a noise texture.

```glsl
shader_type canvas_item;

uniform sampler2D noise_texture;
uniform float dissolve_amount : hint_range(0.0, 1.0) = 0.0;
uniform float edge_width : hint_range(0.0, 0.2) = 0.05;
uniform vec4 edge_color : source_color = vec4(1.0, 0.5, 0.0, 1.0);

void fragment() {
    vec4 tex = texture(TEXTURE, UV);
    float noise = texture(noise_texture, UV).r;
    float edge = smoothstep(dissolve_amount, dissolve_amount + edge_width, noise);
    if (noise < dissolve_amount) {
        discard;
    }
    COLOR = mix(edge_color, tex, edge);
    COLOR.a = tex.a;
}
```

## Water Distortion (2D)

Wavy refraction effect for water surfaces.

```glsl
shader_type canvas_item;

uniform sampler2D screen_texture : hint_screen_texture, filter_linear;
uniform float wave_speed : hint_range(0.0, 5.0) = 1.5;
uniform float wave_strength : hint_range(0.0, 0.1) = 0.02;
uniform float wave_frequency : hint_range(0.0, 50.0) = 15.0;
uniform vec4 water_tint : source_color = vec4(0.2, 0.4, 0.8, 0.3);

void fragment() {
    vec2 offset;
    offset.x = sin(UV.y * wave_frequency + TIME * wave_speed) * wave_strength;
    offset.y = cos(UV.x * wave_frequency + TIME * wave_speed) * wave_strength * 0.5;
    vec4 screen = texture(screen_texture, SCREEN_UV + offset);
    COLOR = mix(screen, water_tint, water_tint.a);
}
```

## CRT Scanlines (2D)

Retro monitor effect.

```glsl
shader_type canvas_item;

uniform float line_count : hint_range(50.0, 500.0) = 200.0;
uniform float line_opacity : hint_range(0.0, 1.0) = 0.15;
uniform float curvature : hint_range(0.0, 0.1) = 0.02;

void fragment() {
    // Barrel distortion
    vec2 uv = UV * 2.0 - 1.0;
    uv *= 1.0 + curvature * dot(uv, uv);
    uv = uv * 0.5 + 0.5;

    vec4 tex = texture(TEXTURE, uv);
    float scanline = sin(uv.y * line_count * PI) * 0.5 + 0.5;
    tex.rgb -= scanline * line_opacity;
    COLOR = tex;
}
```

## Drop Shadow (2D)

```glsl
shader_type canvas_item;

uniform vec2 shadow_offset = vec2(2.0, 2.0);
uniform vec4 shadow_color : source_color = vec4(0.0, 0.0, 0.0, 0.5);

void fragment() {
    vec2 shadow_uv = UV - shadow_offset * TEXTURE_PIXEL_SIZE;
    float shadow_alpha = texture(TEXTURE, shadow_uv).a;
    vec4 shadow = vec4(shadow_color.rgb, shadow_alpha * shadow_color.a);
    vec4 tex = texture(TEXTURE, UV);
    COLOR = mix(shadow, tex, tex.a);
}
```

## Chromatic Aberration (2D, full-screen)

Apply to a `ColorRect` covering the viewport.

```glsl
shader_type canvas_item;

uniform sampler2D screen_texture : hint_screen_texture, filter_linear;
uniform float aberration : hint_range(0.0, 10.0) = 2.0;

void fragment() {
    vec2 offset = (UV - 0.5) * aberration * TEXTURE_PIXEL_SIZE;
    float r = texture(screen_texture, SCREEN_UV + offset).r;
    float g = texture(screen_texture, SCREEN_UV).g;
    float b = texture(screen_texture, SCREEN_UV - offset).b;
    COLOR = vec4(r, g, b, 1.0);
}
```

---

## Applying Shaders

### Per-Node (2D)

Inspector → CanvasItem → Material → New ShaderMaterial → New Shader → edit.

### Full-Screen Post-Processing

Add a `CanvasLayer` (layer 128) with a `ColorRect` that covers the viewport. Apply shader to ColorRect's material. Use `hint_screen_texture` to read the rendered frame.

### Sharing Materials

To share a ShaderMaterial across nodes without sharing parameter values: Inspector → Material → right-click → Make Unique. To intentionally share: assign the same `.tres` resource.

### Visual Shader Editor

For node-based shader authoring: New ShaderMaterial → New VisualShader. Connect nodes in the visual graph editor. Equivalent to code shaders but with a graph UI.

---

## Uniform Type Hints

| Hint | Type | Description |
|------|------|-------------|
| `source_color` | vec4 | Color picker in Inspector |
| `hint_range(min, max, step)` | float/int | Slider |
| `hint_screen_texture` | sampler2D | Screen buffer |
| `hint_normal` | sampler2D | Normal map |
| `hint_default_white` | sampler2D | Default white texture |

## Built-in Variables (CanvasItem)

| Variable | Type | Description |
|----------|------|-------------|
| `UV` | vec2 | Texture coordinates |
| `VERTEX` | vec2 | Vertex position (vertex shader) |
| `COLOR` | vec4 | Output color (fragment shader) |
| `TEXTURE` | sampler2D | Node's texture |
| `TEXTURE_PIXEL_SIZE` | vec2 | 1.0 / texture size |
| `SCREEN_UV` | vec2 | Screen-space UV |
| `TIME` | float | Elapsed time in seconds |
| `AT_LIGHT_PASS` | bool | True during 2D light pass |
