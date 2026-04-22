# Common 2D Game Mechanics

## Platformer Mechanics

### Basic Jump
```
On jump press AND grounded:
    velocityY = jumpForce  (negative in Unity since Y-up; positive-up in Phaser with gravity pulling down)
```

### Variable Jump Height (release early = lower jump)
```javascript
// Phaser
update(time, delta) {
    if (cursors.up.isDown && player.body.touching.down) {
        player.setVelocityY(-jumpForce);
    }
    // Cut jump short when button released
    if (cursors.up.isUp && player.body.velocity.y < 0) {
        player.setVelocityY(player.body.velocity.y * 0.5); // halve upward velocity
    }
}
```
```csharp
// Unity
void Update() {
    if (Input.GetButtonDown("Jump") && isGrounded)
        rb.linearVelocity = new Vector2(rb.linearVelocity.x, jumpForce);
    if (Input.GetButtonUp("Jump") && rb.linearVelocity.y > 0)
        rb.linearVelocity = new Vector2(rb.linearVelocity.x, rb.linearVelocity.y * 0.5f);
}
```

### Better Jump Curve (heavier fall)
Apply higher gravity when falling than when rising for a snappier feel.
```csharp
// Unity
void FixedUpdate() {
    if (rb.linearVelocity.y < 0)
        rb.linearVelocity += Vector2.up * Physics2D.gravity.y * (fallMultiplier - 1) * Time.fixedDeltaTime;
    else if (rb.linearVelocity.y > 0 && !Input.GetButton("Jump"))
        rb.linearVelocity += Vector2.up * Physics2D.gravity.y * (lowJumpMultiplier - 1) * Time.fixedDeltaTime;
}
// Typical values: fallMultiplier = 2.5f, lowJumpMultiplier = 2f
```

### Coyote Time (grace period after leaving ledge)
Player can still jump for a few frames after walking off a platform. NOT built into any engine.

```javascript
// Phaser
let coyoteTime = 0;
const COYOTE_DURATION = 100; // ms

update(time, delta) {
    if (player.body.touching.down) {
        coyoteTime = COYOTE_DURATION;
    } else {
        coyoteTime -= delta;
    }
    if (cursors.up.isDown && coyoteTime > 0) {
        player.setVelocityY(-jumpForce);
        coyoteTime = 0;
    }
}
```
```csharp
// Unity
private float coyoteCounter;
private const float CoyoteDuration = 0.1f; // seconds

void Update() {
    if (isGrounded) coyoteCounter = CoyoteDuration;
    else coyoteCounter -= Time.deltaTime;

    if (Input.GetButtonDown("Jump") && coyoteCounter > 0f) {
        rb.linearVelocity = new Vector2(rb.linearVelocity.x, jumpForce);
        coyoteCounter = 0f;
    }
}
```

### Jump Buffering (press jump slightly before landing)
Remembers the jump press and executes it when the player lands.

```csharp
// Unity
private float jumpBufferCounter;
private const float JumpBufferDuration = 0.15f;

void Update() {
    if (Input.GetButtonDown("Jump"))
        jumpBufferCounter = JumpBufferDuration;
    else
        jumpBufferCounter -= Time.deltaTime;

    if (jumpBufferCounter > 0f && isGrounded) {
        rb.linearVelocity = new Vector2(rb.linearVelocity.x, jumpForce);
        jumpBufferCounter = 0f;
    }
}
```

### Combined Coyote Time + Jump Buffering
```csharp
void Update() {
    if (isGrounded) coyoteCounter = CoyoteDuration;
    else coyoteCounter -= Time.deltaTime;

    if (Input.GetButtonDown("Jump")) jumpBufferCounter = JumpBufferDuration;
    else jumpBufferCounter -= Time.deltaTime;

    if (coyoteCounter > 0f && jumpBufferCounter > 0f) {
        rb.linearVelocity = new Vector2(rb.linearVelocity.x, jumpForce);
        coyoteCounter = 0f;
        jumpBufferCounter = 0f;
    }
}
```

### Wall Slide and Wall Jump
```csharp
private bool isTouchingWall;
private bool isWallSliding;

void Update() {
    isTouchingWall = Physics2D.Raycast(transform.position, Vector2.right * facingDirection,
        wallCheckDistance, wallLayer);

    isWallSliding = isTouchingWall && !isGrounded && rb.linearVelocity.y < 0;

    if (isWallSliding)
        rb.linearVelocity = new Vector2(rb.linearVelocity.x, -wallSlideSpeed);

    if (Input.GetButtonDown("Jump") && isWallSliding) {
        rb.linearVelocity = new Vector2(-facingDirection * wallJumpForce.x, wallJumpForce.y);
        facingDirection *= -1;
    }
}
```

### One-Way Platforms (Phaser)
```javascript
// Phaser: use Platform Collider with custom process callback
this.physics.add.collider(player, platform, null, (player, platform) => {
    return player.body.velocity.y > 0 && player.body.bottom <= platform.body.top + 5;
}, this);
```

### One-Way Platforms (Unity)
Use `PlatformEffector2D` component on the platform's collider:
- Set `Surface Arc = 180` (only top side collides)
- Enable `Use One Way` on the Collider2D

---

## Top-Down Movement

### 8-Directional with Normalization
Without normalization, diagonal movement is ~41% faster.

```javascript
// Phaser
update(time, delta) {
    let vx = 0, vy = 0;
    if (cursors.left.isDown) vx = -1;
    else if (cursors.right.isDown) vx = 1;
    if (cursors.up.isDown) vy = -1;
    else if (cursors.down.isDown) vy = 1;

    // Normalize to prevent fast diagonals
    const len = Math.sqrt(vx * vx + vy * vy);
    if (len > 0) { vx /= len; vy /= len; }

    player.setVelocity(vx * speed, vy * speed);
}
```
```csharp
// Unity
void Update() {
    Vector2 input = new Vector2(Input.GetAxisRaw("Horizontal"), Input.GetAxisRaw("Vertical"));
    input = Vector2.ClampMagnitude(input, 1f); // normalize diagonal
    moveInput = input;
}
void FixedUpdate() {
    rb.linearVelocity = moveInput * moveSpeed;
}
```

### Smooth Acceleration/Deceleration
```csharp
// Unity
void FixedUpdate() {
    Vector2 targetVelocity = moveInput * maxSpeed;
    rb.linearVelocity = Vector2.MoveTowards(rb.linearVelocity, targetVelocity, acceleration * Time.fixedDeltaTime);
}
// Or use Rigidbody2D.linearDamping for natural deceleration
```

---

## Projectiles

### Basic Bullet Spawning
```javascript
// Phaser
fireBullet() {
    const bullet = bullets.get(player.x, player.y, 'bullet');
    if (bullet) {
        bullet.setActive(true).setVisible(true);
        bullet.body.enable = true;
        this.physics.moveTo(bullet, targetX, targetY, bulletSpeed);
        this.time.delayedCall(3000, () => {
            bullet.setActive(false).setVisible(false);
            bullet.body.enable = false;
        });
    }
}
```
```csharp
// Unity
void Fire() {
    GameObject bullet = Instantiate(bulletPrefab, firePoint.position, firePoint.rotation);
    Vector2 direction = (target.position - firePoint.position).normalized;
    bullet.GetComponent<Rigidbody2D>().linearVelocity = direction * bulletSpeed;
    Destroy(bullet, 3f);
}
```

### Aim Toward Mouse
```javascript
// Phaser
const angle = Phaser.Math.Angle.Between(player.x, player.y, pointer.worldX, pointer.worldY);
this.physics.velocityFromRotation(angle, bulletSpeed, bullet.body.velocity);
```
```csharp
// Unity
Vector3 mouseWorld = Camera.main.ScreenToWorldPoint(Input.mousePosition);
mouseWorld.z = 0;
Vector2 dir = (mouseWorld - transform.position).normalized;
float angle = Mathf.Atan2(dir.y, dir.x) * Mathf.Rad2Deg;
transform.rotation = Quaternion.Euler(0, 0, angle);
```

---

## Health and Damage

### Invincibility Frames (iframes)
```csharp
// Unity
private bool isInvincible = false;

public void TakeDamage(int amount) {
    if (isInvincible) return;
    health -= amount;
    StartCoroutine(InvincibilityFrames());
}

IEnumerator InvincibilityFrames() {
    isInvincible = true;
    float elapsed = 0f;
    while (elapsed < invincibilityDuration) {
        sr.enabled = !sr.enabled; // flicker
        yield return new WaitForSeconds(0.1f);
        elapsed += 0.1f;
    }
    sr.enabled = true;
    isInvincible = false;
}
```

### Knockback
```csharp
// Unity
public void TakeDamage(int amount, Vector2 knockbackSource) {
    health -= amount;
    Vector2 knockDir = ((Vector2)transform.position - knockbackSource).normalized;
    rb.linearVelocity = Vector2.zero;
    rb.AddForce(knockDir * knockbackForce, ForceMode2D.Impulse);
}
```

---

## Parallax Scrolling

```javascript
// Phaser — using TileSprite
// In create:
this.bg1 = this.add.tileSprite(400, 300, 800, 600, 'mountains');
this.bg2 = this.add.tileSprite(400, 300, 800, 600, 'trees');
this.bg1.setScrollFactor(0); // fixed to camera
this.bg2.setScrollFactor(0);

// In update:
this.bg1.tilePositionX = this.cameras.main.scrollX * 0.2; // slow
this.bg2.tilePositionX = this.cameras.main.scrollX * 0.5; // medium
```

```csharp
// Unity — move layers relative to camera
public class ParallaxLayer : MonoBehaviour {
    [SerializeField] private float parallaxFactor = 0.5f; // 0=fixed, 1=moves with camera
    private float startPosX;
    private Transform cam;

    void Start() {
        cam = Camera.main.transform;
        startPosX = transform.position.x;
    }

    void LateUpdate() {
        float dist = cam.position.x * parallaxFactor;
        transform.position = new Vector3(startPosX + dist, transform.position.y, transform.position.z);
    }
}
```

---

## Game State Management

### Simple State Machine (Phaser)
```javascript
// Use separate Scenes for each state
// Boot -> MainMenu -> Game -> GameOver

class MainMenu extends Phaser.Scene {
    constructor() { super('MainMenu'); }
    create() {
        this.add.text(400, 250, 'My Game', { fontSize: '48px' }).setOrigin(0.5);
        const startBtn = this.add.text(400, 350, 'Start', { fontSize: '32px' })
            .setOrigin(0.5).setInteractive();
        startBtn.on('pointerdown', () => this.scene.start('Game'));
    }
}
```

### Pause (Phaser)
```javascript
// Launch overlay scene
this.input.keyboard.on('keydown-ESC', () => {
    this.scene.pause();
    this.scene.launch('PauseMenu');
});
// In PauseMenu scene:
this.input.keyboard.on('keydown-ESC', () => {
    this.scene.stop();
    this.scene.resume('Game');
});
```

### Pause (Unity)
```csharp
public void TogglePause() {
    if (Time.timeScale == 0f) {
        Time.timeScale = 1f;
        pauseMenu.SetActive(false);
    } else {
        Time.timeScale = 0f;
        pauseMenu.SetActive(true);
    }
}
// Use WaitForSecondsRealtime in coroutines during pause
// UI animations: use unscaledDeltaTime
```

---

## Screen Wrap

```javascript
// Phaser
update() {
    if (player.x < 0) player.x = 800;
    else if (player.x > 800) player.x = 0;
    if (player.y < 0) player.y = 600;
    else if (player.y > 600) player.y = 0;
}
// Or: this.physics.world.wrap(player, 16); // 16 = padding
```
```csharp
// Unity
void Update() {
    Vector3 pos = transform.position;
    Vector3 viewPos = Camera.main.WorldToViewportPoint(pos);
    if (viewPos.x < 0) viewPos.x = 1;
    else if (viewPos.x > 1) viewPos.x = 0;
    if (viewPos.y < 0) viewPos.y = 1;
    else if (viewPos.y > 1) viewPos.y = 0;
    transform.position = Camera.main.ViewportToWorldPoint(viewPos);
}
```

---

## Collectibles and Scoring

```javascript
// Phaser
const coins = this.physics.add.group();
for (let i = 0; i < 10; i++) {
    coins.create(Phaser.Math.Between(50, 750), Phaser.Math.Between(50, 550), 'coin');
}
this.physics.add.overlap(player, coins, (player, coin) => {
    coin.disableBody(true, true);
    score += 10;
    scoreText.setText('Score: ' + score);
    this.sound.play('pickup');
}, null, this);
```
