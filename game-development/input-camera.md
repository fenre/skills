# Input & Camera Systems

## Keyboard Input

### Key States
| State | Phaser 3 | Unity |
|-------|----------|-------|
| Just pressed (one frame) | `Phaser.Input.Keyboard.JustDown(key)` | `Input.GetKeyDown(KeyCode.X)` |
| Held down | `key.isDown` | `Input.GetKey(KeyCode.X)` |
| Just released (one frame) | `Phaser.Input.Keyboard.JustUp(key)` | `Input.GetKeyUp(KeyCode.X)` |

### Phaser Keyboard Setup
```javascript
// Cursor keys (arrows + space + shift)
const cursors = this.input.keyboard.createCursorKeys();

// WASD
const wasd = this.input.keyboard.addKeys({
    up: Phaser.Input.Keyboard.KeyCodes.W,
    down: Phaser.Input.Keyboard.KeyCodes.S,
    left: Phaser.Input.Keyboard.KeyCodes.A,
    right: Phaser.Input.Keyboard.KeyCodes.D
});

// Single key
const fireKey = this.input.keyboard.addKey(Phaser.Input.Keyboard.KeyCodes.SPACE);

// Event-based
this.input.keyboard.on('keydown-SPACE', () => { });
this.input.keyboard.on('keydown-ESC', () => { });

// Prevent key from propagating to browser
this.input.keyboard.addCapture('SPACE'); // prevents page scroll
```

### Unity Keyboard
```csharp
// Legacy Input
float h = Input.GetAxisRaw("Horizontal"); // -1, 0, or 1
float v = Input.GetAxisRaw("Vertical");
if (Input.GetKeyDown(KeyCode.Escape)) { }

// Common KeyCodes
// KeyCode.Space, KeyCode.LeftShift, KeyCode.LeftControl
// KeyCode.Return (Enter), KeyCode.Escape, KeyCode.Tab
// KeyCode.A through KeyCode.Z
// KeyCode.Alpha0 through KeyCode.Alpha9
// KeyCode.UpArrow, DownArrow, LeftArrow, RightArrow
```

---

## Mouse / Pointer Input

### Phaser 3
```javascript
// Active pointer (mouse or first touch)
const pointer = this.input.activePointer;
pointer.x;           // screen X
pointer.y;           // screen Y
pointer.worldX;      // world X (accounts for camera scroll)
pointer.worldY;      // world Y

// Click state
pointer.isDown;
pointer.leftButtonDown();
pointer.rightButtonDown();
pointer.middleButtonDown();

// Click on game object
sprite.setInteractive();
sprite.on('pointerdown', (pointer) => { });
sprite.on('pointerup', (pointer) => { });
sprite.on('pointerover', () => { }); // hover
sprite.on('pointerout', () => { });  // unhover

// Click anywhere on scene
this.input.on('pointerdown', (pointer) => { });

// Right-click context menu (disable browser default)
this.input.mouse.disableContextMenu();
```

### Unity
```csharp
// Mouse position (screen pixels)
Vector3 mouseScreen = Input.mousePosition;

// Convert to world position (essential for 2D)
Vector3 mouseWorld = Camera.main.ScreenToWorldPoint(mouseScreen);
mouseWorld.z = 0; // MUST zero z for 2D

// Mouse buttons: 0=left, 1=right, 2=middle
if (Input.GetMouseButtonDown(0)) { }  // click
if (Input.GetMouseButton(0)) { }      // held
if (Input.GetMouseButtonUp(0)) { }    // released

// Click on object (with collider)
void Update() {
    if (Input.GetMouseButtonDown(0)) {
        Vector2 mousePos = Camera.main.ScreenToWorldPoint(Input.mousePosition);
        RaycastHit2D hit = Physics2D.Raycast(mousePos, Vector2.zero);
        if (hit.collider != null) {
            // clicked on hit.collider.gameObject
        }
    }
}

// Scroll wheel
float scroll = Input.mouseScrollDelta.y; // +1 up, -1 down
```

---

## Touch Input

### Phaser 3
```javascript
// Enable multi-touch
this.input.addPointer(1); // adds second pointer (total 2)

// Access pointers
const p1 = this.input.pointer1; // first touch
const p2 = this.input.pointer2; // second touch

// Touch events
this.input.on('pointerdown', (pointer) => { pointer.x; pointer.y; });

// Swipe detection
let startX, startY;
this.input.on('pointerdown', (pointer) => { startX = pointer.x; startY = pointer.y; });
this.input.on('pointerup', (pointer) => {
    const dx = pointer.x - startX;
    const dy = pointer.y - startY;
    const dist = Math.sqrt(dx*dx + dy*dy);
    if (dist > 50) { // minimum swipe distance
        if (Math.abs(dx) > Math.abs(dy)) {
            // horizontal swipe: dx > 0 ? 'right' : 'left'
        } else {
            // vertical swipe: dy > 0 ? 'down' : 'up'
        }
    }
});
```

### Unity
```csharp
if (Input.touchCount > 0) {
    Touch touch = Input.GetTouch(0);
    Vector2 touchPos = Camera.main.ScreenToWorldPoint(touch.position);

    switch (touch.phase) {
        case TouchPhase.Began:    break; // finger down
        case TouchPhase.Moved:    break; // finger moving
        case TouchPhase.Ended:    break; // finger lifted
        case TouchPhase.Canceled: break;
    }

    // Swipe
    if (touch.phase == TouchPhase.Ended) {
        Vector2 delta = touch.position - touchStartPos;
        if (delta.magnitude > 50f) {
            if (Mathf.Abs(delta.x) > Mathf.Abs(delta.y))
                direction = delta.x > 0 ? "right" : "left";
            else
                direction = delta.y > 0 ? "up" : "down";
        }
    }
}
```

---

## Gamepad Input

### Phaser 3
```javascript
// Enable gamepad in config
input: { gamepad: true }

// In update
if (this.input.gamepad.total > 0) {
    const pad = this.input.gamepad.getPad(0);
    const axisH = pad.axes[0].getValue(); // -1 to 1 (left stick X)
    const axisV = pad.axes[1].getValue(); // -1 to 1 (left stick Y)

    // Buttons (Xbox layout)
    if (pad.buttons[0].pressed) { } // A
    if (pad.buttons[1].pressed) { } // B
    if (pad.buttons[2].pressed) { } // X
    if (pad.buttons[3].pressed) { } // Y
}

// Event-based
this.input.gamepad.on('down', (pad, button, value) => { });
```

### Unity
```csharp
// Legacy Input (configured in Input Manager)
float h = Input.GetAxis("Horizontal"); // left stick X
float v = Input.GetAxis("Vertical");   // left stick Y

if (Input.GetButtonDown("Fire1")) { }  // mapped to gamepad button

// New Input System: define Gamepad bindings in Input Actions asset
// Automatically handles gamepad + keyboard with same action
```

### Deadzone Handling
Apply deadzone to prevent stick drift (small values when centered):
```javascript
function applyDeadzone(value, deadzone = 0.15) {
    if (Math.abs(value) < deadzone) return 0;
    return (value - Math.sign(value) * deadzone) / (1 - deadzone);
}
```

---

## Input Buffering

Store recent inputs so the game feels responsive even if the player presses slightly early.

```javascript
// Track recent inputs with timestamps
const inputBuffer = [];
const BUFFER_WINDOW = 150; // ms

function bufferInput(action) {
    inputBuffer.push({ action, time: performance.now() });
}

function consumeBufferedInput(action) {
    const now = performance.now();
    const index = inputBuffer.findIndex(
        i => i.action === action && now - i.time < BUFFER_WINDOW
    );
    if (index >= 0) {
        inputBuffer.splice(index, 1);
        return true;
    }
    return false;
}

// Usage: buffer jump when pressed, consume when able
if (jumpPressed) bufferInput('jump');
if (canJump && consumeBufferedInput('jump')) performJump();
```

---

## Camera Systems

### Follow with Lerp (smooth follow)

```javascript
// Phaser
this.cameras.main.startFollow(player, true, 0.08, 0.08); // lerpX, lerpY

// Manual (any engine)
function updateCamera(camera, target, lerpFactor, dt) {
    camera.x += (target.x - camera.x) * lerpFactor;
    camera.y += (target.y - camera.y) * lerpFactor;
    // Frame-rate independent:
    // camera.x += (target.x - camera.x) * (1 - Math.pow(1 - lerpFactor, dt * 60));
}
```

```csharp
// Unity (in LateUpdate)
void LateUpdate() {
    Vector3 targetPos = new Vector3(target.position.x, target.position.y, -10f);
    transform.position = Vector3.Lerp(transform.position, targetPos, smoothSpeed * Time.deltaTime);
}
// Better: use Cinemachine (see unity-2d-reference.md)
```

### Camera Deadzone
Camera doesn't move until the target exits a central rectangle. Reduces micro-movements.

```javascript
// Phaser
this.cameras.main.setDeadzone(200, 100); // width, height of deadzone

// Manual
function updateCameraDeadzone(camera, target, deadzone) {
    if (target.x > camera.x + deadzone.width / 2) camera.x = target.x - deadzone.width / 2;
    if (target.x < camera.x - deadzone.width / 2) camera.x = target.x + deadzone.width / 2;
    if (target.y > camera.y + deadzone.height / 2) camera.y = target.y - deadzone.height / 2;
    if (target.y < camera.y - deadzone.height / 2) camera.y = target.y + deadzone.height / 2;
}
```

### Camera Look-Ahead
Camera shifts slightly in the direction of movement so the player sees more ahead.

```javascript
// Add velocity-based offset to camera target
const lookAheadX = player.body.velocity.x * lookAheadFactor;
const cameraTargetX = player.x + lookAheadX;
// Then lerp camera toward cameraTargetX
```

### Camera Bounds (restrict to level)
```javascript
// Phaser
this.cameras.main.setBounds(0, 0, mapWidth, mapHeight);

// Manual clamp
camera.x = clamp(camera.x, 0, mapWidth - viewportWidth);
camera.y = clamp(camera.y, 0, mapHeight - viewportHeight);
```

### Camera Shake (Screen Shake)

```javascript
// Phaser
this.cameras.main.shake(200, 0.01); // duration ms, intensity

// Manual (apply random offset to camera position)
function screenShake(intensity, duration) {
    let remaining = duration;
    return function update(dt) {
        if (remaining <= 0) return { x: 0, y: 0 };
        remaining -= dt;
        const factor = remaining / duration; // decreases over time
        return {
            x: (Math.random() - 0.5) * 2 * intensity * factor,
            y: (Math.random() - 0.5) * 2 * intensity * factor
        };
    };
}
```

```csharp
// Unity with Cinemachine
// Add CinemachineImpulseSource component to source object
// Add CinemachineImpulseListener to Virtual Camera
impulseSource.GenerateImpulse(); // trigger shake
```

### Camera Zoom

```javascript
// Phaser
this.cameras.main.setZoom(2); // 2x zoom in

// Smooth zoom
this.cameras.main.zoomTo(2, 500); // target zoom, duration ms
```

```csharp
// Unity (Orthographic)
Camera.main.orthographicSize = targetSize; // half vertical view in world units
// Smaller = zoomed in, Larger = zoomed out

// Smooth zoom
Camera.main.orthographicSize = Mathf.Lerp(currentSize, targetSize, speed * Time.deltaTime);
```

---

## Camera Best Practices

1. **Use LateUpdate for camera** (Unity) — ensures camera follows AFTER all movement
2. **Lerp smoothing values**: 0.05-0.15 for lazy follow, 0.3-0.5 for tight follow
3. **Always clamp to bounds** to prevent showing empty space beyond the level
4. **Deadzone + look-ahead** together give the best feel for side-scrollers
5. **Screen shake duration**: keep short (100-300ms); intensity varies by event (0.005 for hits, 0.02 for explosions)
6. **Noise-based shake** (Perlin noise) feels more natural than random offset shake
7. **Cinemachine** (Unity) handles all of this with minimal code — prefer it over manual camera scripts
