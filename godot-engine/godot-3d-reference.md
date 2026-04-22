# Godot 4 — 3D Reference

## Core 3D Nodes

| Node | Purpose |
|------|---------|
| `Node3D` | Base for all 3D nodes (position, rotation, scale) |
| `MeshInstance3D` | Renders 3D geometry |
| `Camera3D` | Viewport camera |
| `DirectionalLight3D` | Sun-like light (parallel rays) |
| `OmniLight3D` | Point light (radiates in all directions) |
| `SpotLight3D` | Cone-shaped light |
| `WorldEnvironment` | Sky, fog, ambient light, tonemap, SSAO, SSR |
| `CharacterBody3D` | Player/NPC movement (code-driven) |
| `RigidBody3D` | Physics-driven objects |
| `StaticBody3D` | Immovable collision (floors, walls) |
| `Area3D` | Trigger zones, damage areas |
| `NavigationRegion3D` | Walkable surfaces for pathfinding |
| `GPUParticles3D` | GPU-accelerated particle effects |

## MeshInstance3D & Primitives

Built-in mesh types: `BoxMesh`, `SphereMesh`, `CylinderMesh`, `CapsuleMesh`, `PlaneMesh`, `PrismMesh`, `TubeTrailMesh`, `RibbonTrailMesh`.

```gdscript
var mesh_inst := MeshInstance3D.new()
mesh_inst.mesh = BoxMesh.new()
mesh_inst.mesh.size = Vector3(2, 1, 3)
add_child(mesh_inst)
```

## Importing 3D Models

| Format | Recommendation |
|--------|---------------|
| **glTF 2.0** (.gltf/.glb) | Recommended — open standard, best support |
| **FBX** | Supported via FBX2glTF or ufbx |
| **OBJ** | Static meshes only, no animation |
| **Collada** (.dae) | Legacy, use glTF instead |
| **Blend** | Direct import if Blender installed |

## Materials — StandardMaterial3D

PBR properties:

| Property | Range | Description |
|----------|-------|-------------|
| `albedo_color` | Color | Base color |
| `albedo_texture` | Texture2D | Base color map |
| `metallic` | 0.0–1.0 | 0 = dielectric, 1 = metal |
| `roughness` | 0.0–1.0 | 0 = smooth/glossy, 1 = rough/matte |
| `normal_map` | Texture2D | Adds surface detail without geometry |
| `emission` | Color | Self-illumination color |
| `emission_energy` | float | Emission brightness |
| `rim` | 0.0–1.0 | Fresnel rim lighting |
| `ao_texture` | Texture2D | Ambient occlusion map |

**ORMMaterial3D**: Packs Occlusion (R), Roughness (G), Metallic (B) into one texture for efficiency.

```gdscript
var mat := StandardMaterial3D.new()
mat.albedo_color = Color.CORNFLOWER_BLUE
mat.metallic = 0.8
mat.roughness = 0.2
mesh_inst.material_override = mat
```

## Camera3D

```gdscript
# Perspective (default)
camera.projection = Camera3D.PROJECTION_PERSPECTIVE
camera.fov = 75.0

# Orthographic (isometric, strategy games)
camera.projection = Camera3D.PROJECTION_ORTHOGONAL
camera.size = 10.0

# Look at target
camera.look_at(target.global_position)
```

## Lighting Setup

Typical 3-point setup:
1. `DirectionalLight3D` — main sun, enable shadows
2. `OmniLight3D` — fill light
3. `WorldEnvironment` — ambient light via `Environment` resource

```gdscript
# Dynamic light
var light := OmniLight3D.new()
light.light_color = Color(1.0, 0.8, 0.6)
light.light_energy = 2.0
light.omni_range = 10.0
light.shadow_enabled = true
```

## Environment & Post-Processing

Configure via `WorldEnvironment` → `Environment` resource:

| Setting | Effect |
|---------|--------|
| Sky | Background (ProceduralSkyMaterial, PanoramaSkyMaterial) |
| Ambient Light | Fill shadows with color |
| Fog | Distance-based or volumetric fog |
| Tonemap | ACES, Reinhard, Filmic |
| SSAO | Screen-space ambient occlusion |
| SSR | Screen-space reflections |
| Glow/Bloom | Bright area bleed |
| Adjustments | Brightness, contrast, saturation |

## 3D Physics

Same body types as 2D with `3D` suffix. Collision shapes use `CollisionShape3D` with 3D shape resources:

| Shape | Use |
|-------|-----|
| `BoxShape3D` | Crates, walls |
| `SphereShape3D` | Projectiles, simple objects |
| `CapsuleShape3D` | Characters |
| `ConcavePolygonShape3D` | Complex static geometry |
| `ConvexPolygonShape3D` | Moving complex objects |

```gdscript
# Raycast from camera (mouse picking)
func _unhandled_input(event: InputEvent) -> void:
    if event is InputEventMouseButton and event.pressed:
        var from := camera.project_ray_origin(event.position)
        var to := from + camera.project_ray_normal(event.position) * 1000.0
        var query := PhysicsRayQueryParameters3D.create(from, to)
        var result := get_world_3d().direct_space_state.intersect_ray(query)
        if result:
            print("Hit: ", result.collider.name)
```

## Navigation 3D

```gdscript
# NavigationRegion3D holds the walkable mesh
# NavigationAgent3D child of character handles pathfinding

@onready var nav := $NavigationAgent3D

func navigate_to(target: Vector3) -> void:
    nav.target_position = target

func _physics_process(delta: float) -> void:
    if nav.is_navigation_finished(): return
    var next := nav.get_next_path_position()
    var dir := global_position.direction_to(next)
    velocity = dir * SPEED
    move_and_slide()
```
