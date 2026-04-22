---
name: game-development
description: >
  2D game development reference covering Phaser 3 (JavaScript/TypeScript) and Unity 2D (C#).
  Engine-agnostic fundamentals: game loops, delta time, collision detection (AABB, circle, SAT),
  physics, input handling, cameras, sprite animation, tilemaps, AI/pathfinding, particle systems,
  audio, UI/HUD, and common 2D mechanics (platformer, top-down, projectiles). Includes game math
  (vectors, lerp, easing, trig, Bezier, noise) and level design (Tiled editor, procedural generation,
  save/load). Use when building any 2D game, creating game prototypes, implementing game mechanics,
  or answering questions about Phaser, Unity 2D, or general game programming.
---

# 2D Game Development Reference

## The Game Loop

Every game runs a loop: **process input -> update state -> render frame -> repeat**.

```
while (running) {
    deltaTime = currentTime - lastTime   // seconds since last frame
    lastTime = currentTime
    processInput()
    update(deltaTime)                    // move entities, check collisions
    render()                             // draw everything
}
```

**Delta time** makes movement frame-rate independent: `position += velocity * deltaTime`.

| Concept | Phaser 3 | Unity |
|---------|----------|-------|
| Delta time | `update(time, delta)` — delta in **ms** | `Time.deltaTime` — in **seconds** |
| Fixed timestep | `this.physics.world.fixedStep` (Arcade) | `FixedUpdate()` — `Time.fixedDeltaTime` (default 0.02s = 50 Hz) |
| Frame rate | `fps` in Game config (default 60) | `Application.targetFrameRate` or VSync |

## Coordinate Systems (Critical Difference)

| | Phaser 3 | Unity 2D |
|-|----------|----------|
| **Y-axis** | **Down is positive** (screen coords) | **Up is positive** (Cartesian) |
| Origin | Top-left of game canvas | Center of scene (configurable) |
| Rotation | Clockwise positive (radians) | Counter-clockwise positive (degrees) |
| Depth/sorting | `setDepth(n)` — higher = on top | Sorting Layer + Order in Layer — higher = on top |
| Pixel coords | Direct screen pixels | World units (1 unit = configurable, often 100 pixels via PPU) |

## Phaser 3 Scene Lifecycle

```
constructor()   → called once when Scene class instantiated
init(data)      → called every time Scene starts; receives data from scene.start('key', data)
preload()       → load assets here (images, spritesheets, audio, tilemaps)
create()        → create game objects, set up physics, input, cameras, animations
update(time, delta) → called every frame; time = ms since game start, delta = ms since last frame
```

All Phaser APIs accessed via `this.` inside Scene methods (e.g., `this.add`, `this.physics`, `this.input`).

### Minimal Phaser Game

```javascript
const config = {
    type: Phaser.AUTO,
    width: 800,
    height: 600,
    physics: { default: 'arcade', arcade: { gravity: { y: 300 }, debug: false } },
    scene: { preload, create, update }
};
const game = new Phaser.Game(config);

let player, cursors;

function preload() {
    this.load.image('player', 'assets/player.png');
    this.load.image('ground', 'assets/ground.png');
}

function create() {
    const ground = this.physics.add.staticImage(400, 580, 'ground');
    player = this.physics.add.sprite(400, 300, 'player');
    player.setCollideWorldBounds(true);
    this.physics.add.collider(player, ground);
    cursors = this.input.keyboard.createCursorKeys();
}

function update(time, delta) {
    if (cursors.left.isDown) player.setVelocityX(-160);
    else if (cursors.right.isDown) player.setVelocityX(160);
    else player.setVelocityX(0);
    if (cursors.up.isDown && player.body.touching.down) player.setVelocityY(-330);
}
```

## Unity MonoBehaviour Lifecycle

```
Awake()          → called once when GameObject first loads (before Start); use for self-references
OnEnable()       → each time GameObject/component is enabled
Start()          → called once before first Update; use for cross-references
FixedUpdate()    → fixed interval (default 50 Hz); use for PHYSICS (Rigidbody forces/velocity)
Update()         → every frame; use for INPUT and non-physics logic
LateUpdate()     → after all Update() calls; use for CAMERA follow
OnDisable()      → when disabled
OnDestroy()      → when destroyed
```

**Rule**: Read input in `Update()`, apply physics forces in `FixedUpdate()`. Mixing causes jitter.

### Minimal Unity 2D Player Controller

```csharp
using UnityEngine;

public class PlayerController : MonoBehaviour
{
    [SerializeField] private float moveSpeed = 5f;
    [SerializeField] private float jumpForce = 10f;

    private Rigidbody2D rb;
    private bool isGrounded;
    private float moveInput;

    private void Awake()
    {
        rb = GetComponent<Rigidbody2D>();
    }

    private void Update()
    {
        moveInput = Input.GetAxisRaw("Horizontal");
        if (Input.GetButtonDown("Jump") && isGrounded)
            rb.linearVelocity = new Vector2(rb.linearVelocity.x, jumpForce);
    }

    private void FixedUpdate()
    {
        rb.linearVelocity = new Vector2(moveInput * moveSpeed, rb.linearVelocity.y);
    }

    private void OnCollisionEnter2D(Collision2D collision)
    {
        if (collision.gameObject.CompareTag("Ground"))
            isGrounded = true;
    }

    private void OnCollisionExit2D(Collision2D collision)
    {
        if (collision.gameObject.CompareTag("Ground"))
            isGrounded = false;
    }
}
```

## Phaser 3 vs Unity 2D Quick Comparison

| Feature | Phaser 3 | Unity 2D |
|---------|----------|----------|
| Language | JavaScript / TypeScript | C# |
| Platform | Browser (HTML5 Canvas/WebGL) | Desktop, Mobile, Console, WebGL |
| Physics (simple) | Arcade (AABB only) | Rigidbody2D + BoxCollider2D |
| Physics (complex) | Matter.js (polygons, joints) | Rigidbody2D + PolygonCollider2D |
| Collision callback | `this.physics.add.collider(a, b, cb)` | `OnCollisionEnter2D(Collision2D)` |
| Overlap (trigger) | `this.physics.add.overlap(a, b, cb)` | `OnTriggerEnter2D(Collider2D)` (set isTrigger=true) |
| Sprite creation | `this.add.sprite(x, y, 'key')` | Add SpriteRenderer component |
| Animation | `this.anims.create({...})` | Animator Controller + clips |
| Tilemap | `this.make.tilemap({key})` | Tilemap + TilemapRenderer |
| Camera follow | `this.cameras.main.startFollow(target)` | Cinemachine VirtualCamera |
| Audio | `this.sound.play('key')` | AudioSource.Play() |
| Scene/level swap | `this.scene.start('SceneKey')` | `SceneManager.LoadScene("Name")` |
| UI/text | `this.add.text(x, y, 'text', style)` | Canvas + TextMeshPro |
| Input (keyboard) | `this.input.keyboard.createCursorKeys()` | `Input.GetKeyDown(KeyCode.Space)` |
| Object pooling | `this.add.group({ maxSize })` | Custom pool or `ObjectPool<T>` |

## Common Keycodes

### Phaser 3
```javascript
cursors = this.input.keyboard.createCursorKeys(); // .up, .down, .left, .right, .space, .shift
this.input.keyboard.addKey(Phaser.Input.Keyboard.KeyCodes.W);
// KeyCodes: A-Z, ZERO-NINE, SPACE, ENTER, ESC, SHIFT, CTRL, ALT, TAB
// Check: key.isDown, key.isUp
// Events: Phaser.Input.Keyboard.JustDown(key), JustUp(key)
```

### Unity
```csharp
Input.GetKeyDown(KeyCode.Space)    // true on frame key first pressed
Input.GetKey(KeyCode.Space)        // true while key held
Input.GetKeyUp(KeyCode.Space)      // true on frame key released
Input.GetAxisRaw("Horizontal")     // -1, 0, or 1 (no smoothing)
Input.GetAxis("Horizontal")        // -1 to 1 with smoothing
Input.GetButtonDown("Jump")        // mapped in Input Manager (default: Space)
Input.mousePosition                // Vector3 screen pixels
Camera.main.ScreenToWorldPoint(Input.mousePosition) // world position
```

## Physics Quick Reference

### Phaser 3 Arcade Physics
```javascript
// Enable on sprite
this.physics.add.existing(sprite);
// Or create with physics
sprite = this.physics.add.sprite(x, y, 'key');

sprite.setVelocity(100, -200);
sprite.setAcceleration(0, 300);
sprite.setBounce(0.5);
sprite.setDrag(50);
sprite.setMaxVelocity(300);
sprite.setCollideWorldBounds(true);
sprite.body.setGravityY(500);    // per-object gravity override
sprite.body.allowGravity = false; // disable gravity

// Collision (separates objects)
this.physics.add.collider(player, platforms);
// Overlap (no separation; trigger callback)
this.physics.add.overlap(player, coins, collectCoin, null, this);

// Static groups (immovable)
const platforms = this.physics.add.staticGroup();
platforms.create(400, 568, 'ground');
```

### Unity 2D Physics
```csharp
// Rigidbody2D BodyType:
//   Dynamic   — full physics (gravity, forces, collisions)
//   Kinematic — moves via script; not affected by forces; still triggers collisions
//   Static    — never moves (walls, ground)

// Apply force
rb.AddForce(Vector2.up * jumpForce, ForceMode2D.Impulse);

// Set velocity directly
rb.linearVelocity = new Vector2(speed, rb.linearVelocity.y);

// Collision requires: both have Collider2D, at least one has Rigidbody2D
// Trigger requires: isTrigger = true on one collider, at least one Rigidbody2D

// Callbacks (exact signatures):
void OnCollisionEnter2D(Collision2D collision) { }
void OnCollisionStay2D(Collision2D collision) { }
void OnCollisionExit2D(Collision2D collision) { }
void OnTriggerEnter2D(Collider2D other) { }
void OnTriggerStay2D(Collider2D other) { }
void OnTriggerExit2D(Collider2D other) { }

// Raycasting
RaycastHit2D hit = Physics2D.Raycast(origin, direction, distance, layerMask);
if (hit.collider != null) { /* hit something */ }
```

## Sprite Animation Quick Setup

### Phaser 3
```javascript
// In preload:
this.load.spritesheet('hero', 'hero.png', { frameWidth: 32, frameHeight: 32 });

// In create:
this.anims.create({
    key: 'walk',
    frames: this.anims.generateFrameNumbers('hero', { start: 0, end: 7 }),
    frameRate: 10,
    repeat: -1    // -1 = loop forever
});
this.anims.create({
    key: 'idle',
    frames: [{ key: 'hero', frame: 0 }],
    frameRate: 1
});

// In update:
if (moving) player.anims.play('walk', true);  // true = don't restart if already playing
else player.anims.play('idle', true);

// Flip sprite direction
player.setFlipX(true);  // face left
```

### Unity
1. Create Animator Controller asset
2. Create Animation clips (idle, walk, jump) from sprite sheet
3. Add parameters (e.g., `Speed` float, `IsGrounded` bool)
4. Create transitions with conditions
5. In script:
```csharp
private Animator anim;
void Awake() { anim = GetComponent<Animator>(); }
void Update() {
    anim.SetFloat("Speed", Mathf.Abs(moveInput));
    anim.SetBool("IsGrounded", isGrounded);
    // Flip sprite
    if (moveInput != 0)
        transform.localScale = new Vector3(Mathf.Sign(moveInput), 1, 1);
}
```

## Additional Resources

Detailed references for each topic:

- [phaser-reference.md](phaser-reference.md) — Phaser 3 deep dive (config, scenes, physics, tilemaps, scaling)
- [unity-2d-reference.md](unity-2d-reference.md) — Unity 2D deep dive (components, physics, tilemap, UI, Cinemachine)
- [game-mechanics.md](game-mechanics.md) — Platformer feel, top-down movement, projectiles, game states
- [collision-physics.md](collision-physics.md) — Collision math, spatial partitioning, physics fundamentals
- [game-math.md](game-math.md) — Vectors, lerp, trig, Bezier, easing, noise
- [input-camera.md](input-camera.md) — Input handling, camera follow/shake/bounds/zoom
- [audio-particles-ui.md](audio-particles-ui.md) — Audio, particles, HUD, menus, transitions
- [ai-pathfinding.md](ai-pathfinding.md) — FSM, patrol/chase, A* pathfinding, flocking
- [level-design.md](level-design.md) — Tiled editor, procedural generation, save/load, object pooling
