# Unity 2D Deep Reference

## Project Setup

### Essential Packages (Window > Package Manager)
- **Cinemachine** — smart 2D camera (follow, confiner, deadzone)
- **Input System** — modern action-based input (recommended over legacy)
- **TextMeshPro** — crisp text rendering (import TMP Essentials on first use)
- **2D Tilemap Editor** — included by default; tile painting tools
- **2D Sprite** — sprite editor, atlasing
- **2D Animation** — skeletal/bone animation for sprites

### 2D Settings
- Sprites: set **Pixels Per Unit** (PPU) on import (default 100; 16 or 32 common for pixel art)
- Pixel art: Filter Mode = **Point**, Compression = **None** on sprite import settings
- Camera: set **Orthographic**; Size = half the vertical world units visible

## GameObject / Component Model

Everything in Unity is a **GameObject** with **Components** attached.

```
GameObject "Player"
├── Transform           (position, rotation, scale — always present)
├── SpriteRenderer      (draws the sprite)
├── Rigidbody2D         (physics body)
├── BoxCollider2D       (collision shape)
├── Animator            (animation state machine)
└── PlayerController    (your C# script)
```

### Finding and Referencing
```csharp
// Get component on same GameObject
rb = GetComponent<Rigidbody2D>();

// Get component on child
var childRenderer = GetComponentInChildren<SpriteRenderer>();

// Find by tag
GameObject player = GameObject.FindWithTag("Player");

// Find by type (expensive; avoid in Update)
Camera cam = FindFirstObjectByType<Camera>();

// Serialized field (set in Inspector — preferred over Find)
[SerializeField] private Transform spawnPoint;
```

## Rigidbody2D

### Body Types
| Type | Use | Gravity | Forces | Collisions |
|------|-----|---------|--------|------------|
| Dynamic | Moving objects (player, enemies) | Yes | Yes | Full |
| Kinematic | Scripted movers (platforms, doors) | No | No | Detects but not affected |
| Static | Immovable (walls, ground) | No | No | Full |

### Key Properties
```csharp
rb.linearVelocity = new Vector2(5f, rb.linearVelocity.y);  // set velocity directly
rb.AddForce(Vector2.up * 10f, ForceMode2D.Impulse);        // instant impulse
rb.AddForce(Vector2.right * 5f, ForceMode2D.Force);        // continuous force
rb.gravityScale = 1f;     // multiplier on Physics2D.gravity (default gravity: 0, -9.81)
rb.linearDamping = 0.5f;         // slows velocity over time (friction-like)
rb.constraints = RigidbodyConstraints2D.FreezeRotation;     // prevent tumbling
rb.bodyType = RigidbodyType2D.Kinematic;
rb.simulated = false;     // disable physics without removing component
```

### Common Gotchas
- Set velocity in `FixedUpdate()`, read input in `Update()`
- `AddForce` accumulates; `linearVelocity =` overrides
- Use `ForceMode2D.Impulse` for jumps, `ForceMode2D.Force` for continuous push
- Kinematic bodies don't respond to collisions but DO trigger OnCollision/OnTrigger
- `rb.MovePosition()` for kinematic movement (respects interpolation)

## Colliders

### Types
| Collider | Shape | Use Case |
|----------|-------|----------|
| BoxCollider2D | Rectangle | Most objects, tiles |
| CircleCollider2D | Circle | Balls, circular enemies, range checks |
| CapsuleCollider2D | Capsule | Characters (rounded bottom for slopes) |
| PolygonCollider2D | Custom polygon | Complex shapes (auto-generated from sprite) |
| EdgeCollider2D | Line segments | Ground contours, boundaries |
| CompositeCollider2D | Merged children | Tilemap collision optimization |

### Collision vs Trigger
```csharp
// Collision: physical separation occurs
// Requirements: both have Collider2D, at least one has Rigidbody2D
void OnCollisionEnter2D(Collision2D collision)
{
    if (collision.gameObject.CompareTag("Enemy"))
    {
        // collision.contacts[0].point — contact point
        // collision.contacts[0].normal — surface normal
        // collision.relativeVelocity — impact speed
    }
}
void OnCollisionStay2D(Collision2D collision) { }
void OnCollisionExit2D(Collision2D collision) { }

// Trigger: no physical separation; one collider has isTrigger = true
// Requirements: at least one isTrigger, at least one Rigidbody2D
void OnTriggerEnter2D(Collider2D other)
{
    if (other.CompareTag("Coin"))
    {
        Destroy(other.gameObject);
        score++;
    }
}
void OnTriggerStay2D(Collider2D other) { }
void OnTriggerExit2D(Collider2D other) { }
```

### Layer-Based Collision
- Edit > Project Settings > Physics 2D > Layer Collision Matrix
- Assign GameObjects to layers (e.g., Player, Enemy, Bullet, Ground)
- Uncheck boxes in matrix to disable collision between specific layers

## Raycasting

```csharp
// Raycast (returns first hit)
RaycastHit2D hit = Physics2D.Raycast(origin, direction, distance, layerMask);
if (hit.collider != null)
{
    Debug.Log($"Hit: {hit.collider.name} at {hit.point}");
}

// Ground check with raycast
bool isGrounded = Physics2D.Raycast(
    transform.position,
    Vector2.down,
    groundCheckDistance,
    groundLayer
);

// Circle cast (wider check)
RaycastHit2D hit = Physics2D.CircleCast(origin, radius, direction, distance, layerMask);

// Overlap checks
Collider2D[] results = Physics2D.OverlapCircleAll(position, radius, layerMask);
bool isInZone = Physics2D.OverlapPoint(position, layerMask);

// Debug visualization
Debug.DrawRay(origin, direction * distance, Color.red);
```

## Input

### Legacy Input (simple; works out of the box)
```csharp
float h = Input.GetAxisRaw("Horizontal"); // -1, 0, 1 (no smoothing)
float v = Input.GetAxisRaw("Vertical");

if (Input.GetKeyDown(KeyCode.Space)) { }   // single frame
if (Input.GetKey(KeyCode.Space)) { }       // held
if (Input.GetKeyUp(KeyCode.Space)) { }     // released

if (Input.GetMouseButtonDown(0)) { }       // 0=left, 1=right, 2=middle
Vector3 mouseWorld = Camera.main.ScreenToWorldPoint(Input.mousePosition);
mouseWorld.z = 0; // important for 2D
```

### New Input System (recommended for production)
1. Install Input System package
2. Create Input Actions asset
3. Define Action Maps (Player, UI)
4. Define Actions (Move, Jump, Fire) with bindings

```csharp
using UnityEngine.InputSystem;

public class PlayerController : MonoBehaviour
{
    private PlayerInputActions inputActions;
    private Vector2 moveInput;

    private void Awake()
    {
        inputActions = new PlayerInputActions();
    }

    private void OnEnable()
    {
        inputActions.Player.Enable();
        inputActions.Player.Jump.performed += OnJump;
    }

    private void OnDisable()
    {
        inputActions.Player.Jump.performed -= OnJump;
        inputActions.Player.Disable();
    }

    private void Update()
    {
        moveInput = inputActions.Player.Move.ReadValue<Vector2>();
    }

    private void OnJump(InputAction.CallbackContext context)
    {
        if (isGrounded) rb.linearVelocity = new Vector2(rb.linearVelocity.x, jumpForce);
    }
}
```

## SpriteRenderer and Sorting

### Sorting Order
1. **Sorting Layer** (string): Background < Default < Foreground < UI
2. **Order in Layer** (int): Within same sorting layer, higher = rendered on top
3. Configure in: Edit > Project Settings > Tags and Layers > Sorting Layers

```csharp
var sr = GetComponent<SpriteRenderer>();
sr.sortingLayerName = "Characters";
sr.sortingOrder = 5;
sr.flipX = true;  // mirror horizontally
sr.color = Color.red; // tint
sr.color = new Color(1, 1, 1, 0.5f); // semi-transparent
```

### Sprite Atlas
- Create: Right-click > Create > 2D > Sprite Atlas
- Drag sprites/folders into atlas
- Reduces draw calls by combining textures into single sheet

## Animation

### Setup (via Animator)
1. Select sprite sheet, set **Sprite Mode = Multiple**, open **Sprite Editor**, slice
2. Select frames in Project, drag onto Scene → auto-creates Animator Controller + clip
3. In Animator window: create states (Idle, Walk, Jump), transitions, parameters

### Scripting Animations
```csharp
private Animator anim;

void Awake() { anim = GetComponent<Animator>(); }

void Update()
{
    anim.SetFloat("Speed", Mathf.Abs(moveInput));
    anim.SetBool("IsGrounded", isGrounded);
    anim.SetTrigger("Jump");     // triggers transition once

    // Flip sprite based on direction
    if (moveInput != 0)
        transform.localScale = new Vector3(Mathf.Sign(moveInput), 1, 1);
    // Alternative: sr.flipX = moveInput < 0;
}
```

### Animation Events
- In Animation window, add event at specific frame
- Calls a method on any MonoBehaviour on the same GameObject
- Use for: footstep sounds, spawn projectile at attack frame, VFX triggers

## Tilemap

### Setup
1. Create > 2D Object > Tilemap > Rectangular
2. Opens Tile Palette window; create palette from tileset sprite
3. Paint tiles onto layers

### Structure
```
Grid (parent)
├── Tilemap_Ground     (Sorting Layer: Background, Order: 0)
├── Tilemap_Platforms   (Sorting Layer: Default, Order: 0)
├── Tilemap_Foreground  (Sorting Layer: Foreground, Order: 0)
```

### Collision
```
Tilemap_Platforms
├── Tilemap Renderer
├── Tilemap Collider 2D   (generates collider per tile)
├── Composite Collider 2D (merges adjacent tiles — better performance)
└── Rigidbody2D (Static)  (required by Composite Collider)
```
Set `Tilemap Collider 2D > Used By Composite = true` to merge.

### Rule Tiles
- Auto-tile based on neighbors (e.g., ground tiles auto-connect edges)
- Create > 2D > Tiles > Rule Tile
- Define rules: if neighbor exists above/left/right/below, use this sprite

## Canvas UI

### Canvas Render Modes
| Mode | Use |
|------|-----|
| Screen Space - Overlay | Always on top; scales with screen. Default for HUD. |
| Screen Space - Camera | Rendered by specific camera; can have 3D effects |
| World Space | UI exists in game world (e.g., health bar above enemy) |

### Common UI Setup
```csharp
using TMPro;
using UnityEngine.UI;

public class GameUI : MonoBehaviour
{
    [SerializeField] private TextMeshProUGUI scoreText;
    [SerializeField] private Image healthBar; // set Image Type = Filled
    [SerializeField] private Button restartButton;

    private void Start()
    {
        restartButton.onClick.AddListener(OnRestart);
    }

    public void UpdateScore(int score)
    {
        scoreText.text = $"Score: {score}";
    }

    public void UpdateHealth(float current, float max)
    {
        healthBar.fillAmount = current / max; // 0 to 1
    }

    private void OnRestart()
    {
        SceneManager.LoadScene(SceneManager.GetActiveScene().buildIndex);
    }
}
```

### Anchors
- **Anchor** determines how UI element repositions when screen resizes
- Top-left anchor: element stays relative to top-left corner
- Stretch: element fills available space
- Set via Rect Transform anchor presets (hold Alt to also set pivot+position)

## Cinemachine 2D Camera

### Setup
1. Add package: Cinemachine
2. Create > Cinemachine > 2D Camera
3. Set **Follow** to player Transform
4. Configure **Body > Framing Transposer**: Dead Zone, Soft Zone, Lookahead

### Key Settings
```
CinemachineVirtualCamera
├── Follow: Player Transform
├── Body: Framing Transposer
│   ├── Dead Zone Width/Height: 0.1 (area where camera doesn't move)
│   ├── Soft Zone Width/Height: 0.8
│   ├── Screen X/Y: 0.5 (center)
│   └── Lookahead Time: 0.3 (camera leads movement)
└── Lens > Orthographic Size: 5 (half vertical view in world units)
```

### Confiner (Camera Bounds)
1. Create empty GameObject with PolygonCollider2D matching level bounds
2. Set collider `isTrigger = true`
3. Add `CinemachineConfiner2D` extension to Virtual Camera
4. Set Bounding Shape 2D to the collider

### Camera Shake
```csharp
using Cinemachine;

CinemachineImpulseSource impulse = GetComponent<CinemachineImpulseSource>();
impulse.GenerateImpulse(); // trigger shake
// Configure impulse definition in Inspector (amplitude, frequency, duration)
```

## Prefabs and Instantiation

```csharp
[SerializeField] private GameObject bulletPrefab;
[SerializeField] private Transform firePoint;

void Fire()
{
    GameObject bullet = Instantiate(bulletPrefab, firePoint.position, firePoint.rotation);
    Destroy(bullet, 3f); // auto-destroy after 3 seconds
}
```

### Object Pooling
```csharp
using UnityEngine.Pool;

private ObjectPool<GameObject> bulletPool;

void Awake()
{
    bulletPool = new ObjectPool<GameObject>(
        createFunc: () => Instantiate(bulletPrefab),
        actionOnGet: (obj) => obj.SetActive(true),
        actionOnRelease: (obj) => obj.SetActive(false),
        actionOnDestroy: (obj) => Destroy(obj),
        defaultCapacity: 20,
        maxSize: 50
    );
}

void Fire()
{
    GameObject bullet = bulletPool.Get();
    bullet.transform.position = firePoint.position;
}

// Return to pool (call from bullet script):
// pool.Release(gameObject);
```

## Coroutines

```csharp
// Start
StartCoroutine(FlashRed());

// Stop
StopCoroutine(flashCoroutine);
StopAllCoroutines();

IEnumerator FlashRed()
{
    sr.color = Color.red;
    yield return new WaitForSeconds(0.1f);
    sr.color = Color.white;
    yield return new WaitForSeconds(0.1f);
    sr.color = Color.red;
    yield return new WaitForSeconds(0.1f);
    sr.color = Color.white;
}

// Useful yields:
yield return null;                           // wait one frame
yield return new WaitForSeconds(1f);         // wait 1 second (affected by timeScale)
yield return new WaitForSecondsRealtime(1f); // unaffected by timeScale (pause menus)
yield return new WaitForFixedUpdate();       // wait for next FixedUpdate
yield return new WaitUntil(() => isReady);   // wait until condition true
```

## ScriptableObjects

Reusable data containers that live as assets (not on GameObjects).

```csharp
[CreateAssetMenu(fileName = "NewWeapon", menuName = "Game/Weapon Data")]
public class WeaponData : ScriptableObject
{
    public string weaponName;
    public int damage;
    public float fireRate;
    public float bulletSpeed;
    public Sprite icon;
    public AudioClip fireSound;
}

// Usage in MonoBehaviour:
[SerializeField] private WeaponData weapon;
void Fire()
{
    // Use weapon.damage, weapon.fireRate, etc.
}
```

## Scene Management

```csharp
using UnityEngine.SceneManagement;

// Load by name
SceneManager.LoadScene("GameScene");

// Load by build index
SceneManager.LoadScene(1);

// Reload current
SceneManager.LoadScene(SceneManager.GetActiveScene().buildIndex);

// Additive loading (overlay scenes)
SceneManager.LoadScene("UIScene", LoadSceneMode.Additive);

// Async loading
AsyncOperation asyncLoad = SceneManager.LoadSceneAsync("GameScene");
asyncLoad.allowSceneActivation = false; // hold until ready
// When ready: asyncLoad.allowSceneActivation = true;
// Progress: asyncLoad.progress (0 to 0.9, then activation)
```

## Audio

```csharp
// AudioSource component on GameObject
[SerializeField] private AudioClip jumpSound;
[SerializeField] private AudioClip hitSound;
private AudioSource audioSource;

void Awake() { audioSource = GetComponent<AudioSource>(); }

void PlayJump()
{
    audioSource.PlayOneShot(jumpSound, 0.8f); // volume 0-1
}

// Pitch variation (prevents repetitive feel)
void PlayHit()
{
    audioSource.pitch = Random.Range(0.9f, 1.1f);
    audioSource.PlayOneShot(hitSound);
    audioSource.pitch = 1f;
}

// Background music (separate AudioSource with Loop = true)
// Use AudioMixer for volume control and ducking
```

## Useful Utilities

```csharp
// Time
Time.deltaTime           // seconds since last frame (use in Update)
Time.fixedDeltaTime      // fixed timestep (use in FixedUpdate)
Time.timeScale = 0f;     // pause (set to 1 to resume)
Time.unscaledDeltaTime   // not affected by timeScale

// Math
Mathf.Clamp(value, min, max)
Mathf.Lerp(a, b, t)              // t = 0 to 1
Mathf.MoveTowards(current, target, maxDelta)
Mathf.SmoothDamp(current, target, ref velocity, smoothTime)
Mathf.Sign(value)                 // -1 or 1
Mathf.Abs(value)
Mathf.Approximately(a, b)        // float comparison

// Vector2
Vector2.Distance(a, b)
Vector2.Lerp(a, b, t)
Vector2.MoveTowards(current, target, maxDistanceDelta)
(target - transform.position).normalized  // direction to target
Vector2.Angle(from, to)          // unsigned angle in degrees
Vector2.SignedAngle(from, to)    // signed angle

// Random
Random.Range(0, 10)        // int: 0-9 (exclusive max)
Random.Range(0f, 1f)       // float: 0-1 (inclusive max)
Random.insideUnitCircle    // random Vector2 inside unit circle
```
