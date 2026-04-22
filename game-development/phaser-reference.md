# Phaser 3 Deep Reference

## Game Configuration

```javascript
const config = {
    type: Phaser.AUTO,          // AUTO, CANVAS, or WEBGL
    width: 800,                 // game width in pixels
    height: 600,                // game height in pixels
    parent: 'game-container',   // DOM element ID (optional)
    backgroundColor: '#2d2d2d',
    pixelArt: true,             // prevents anti-aliasing on sprites (crisp pixel art)
    physics: {
        default: 'arcade',     // 'arcade' or 'matter'
        arcade: {
            gravity: { y: 300 },
            debug: false        // set true to see collision boxes
        }
    },
    scale: {
        mode: Phaser.Scale.FIT,           // FIT, RESIZE, ENVELOP, NONE
        autoCenter: Phaser.Scale.CENTER_BOTH,
        min: { width: 400, height: 300 },
        max: { width: 1600, height: 1200 }
    },
    scene: [BootScene, GameScene, UIScene]  // array of Scene classes or single config
};
const game = new Phaser.Game(config);
```

### Scale Modes
| Mode | Behavior |
|------|----------|
| `NONE` | No scaling; game stays at configured size |
| `FIT` | Scale to fit parent, maintaining aspect ratio (letterbox) |
| `ENVELOP` | Scale to fill parent, cropping overflow |
| `RESIZE` | Canvas resizes to match parent; game dimensions change |
| `WIDTH_CONTROLS_HEIGHT` | Width fills; height adjusts proportionally |
| `HEIGHT_CONTROLS_WIDTH` | Height fills; width adjusts proportionally |

## Scene Class Structure

```javascript
class GameScene extends Phaser.Scene {
    constructor() {
        super('GameScene');  // scene key for switching
    }

    init(data) {
        // Runs each time scene starts. data = passed from scene.start()
        this.score = data.score || 0;
    }

    preload() {
        // Load assets (ONLY load here; runs before create)
        this.load.image('sky', 'assets/sky.png');
        this.load.spritesheet('player', 'assets/player.png', {
            frameWidth: 32, frameHeight: 48
        });
        this.load.audio('jump', 'assets/jump.mp3');
        this.load.tilemapTiledJSON('map', 'assets/level1.json');
        this.load.image('tiles', 'assets/tileset.png');

        // Loading progress
        this.load.on('progress', (value) => { /* 0 to 1 */ });
        this.load.on('complete', () => { /* all loaded */ });
    }

    create() {
        // Create game objects, physics, input, etc.
    }

    update(time, delta) {
        // Called every frame. time = ms since start, delta = ms since last frame
        // Convert delta to seconds: const dt = delta / 1000;
    }
}
```

## Scene Management

```javascript
// Start a new scene (stops current scene)
this.scene.start('GameScene', { level: 2, score: 100 });

// Launch a scene in parallel (both run; useful for UI overlay)
this.scene.launch('UIScene');

// Pause/resume
this.scene.pause('GameScene');
this.scene.resume('GameScene');

// Stop a scene
this.scene.stop('UIScene');

// Restart current scene
this.scene.restart();

// Bring scene to top (rendering order)
this.scene.bringToTop('UIScene');

// Listen for events from another scene
this.scene.get('GameScene').events.on('scoreUpdated', (score) => { });

// Emit events
this.events.emit('scoreUpdated', this.score);
```

## Game Objects

### Creating Objects
```javascript
// Static image (no physics)
const bg = this.add.image(400, 300, 'sky');
bg.setOrigin(0.5, 0.5);  // default origin is center (0.5, 0.5)
bg.setScale(2);
bg.setAlpha(0.8);
bg.setDepth(0);           // rendering order (higher = on top)
bg.setScrollFactor(0);    // 0 = fixed to camera (HUD); 1 = scrolls normally

// TileSprite (repeating background)
const bg = this.add.tileSprite(400, 300, 800, 600, 'stars');
// In update: bg.tilePositionX += 1; // scrolling background

// Sprite with physics
const player = this.physics.add.sprite(100, 100, 'player');

// Text
const score = this.add.text(16, 16, 'Score: 0', {
    fontFamily: 'Arial',
    fontSize: '24px',
    color: '#ffffff',
    stroke: '#000000',
    strokeThickness: 3
});
score.setScrollFactor(0);  // fixed to camera
```

### Groups
```javascript
// Dynamic physics group
const enemies = this.physics.add.group({
    maxSize: 20,          // for object pooling
    runChildUpdate: true  // calls update() on children
});

// Create children
enemies.create(200, 100, 'enemy');

// Get from pool
const enemy = enemies.get(x, y, 'enemy');
if (enemy) {
    enemy.setActive(true);
    enemy.setVisible(true);
    enemy.body.enable = true;
}

// Return to pool
enemy.setActive(false);
enemy.setVisible(false);
enemy.body.enable = false;

// Static group (immovable; great for platforms)
const platforms = this.physics.add.staticGroup();
platforms.create(400, 568, 'ground');

// Iterate group
enemies.getChildren().forEach(enemy => { /* ... */ });
```

## Arcade Physics

### Body Properties
```javascript
sprite.setVelocity(vx, vy);
sprite.setVelocityX(vx);
sprite.setVelocityY(vy);
sprite.setAcceleration(ax, ay);
sprite.setBounce(0.5);           // 0 = no bounce, 1 = full bounce
sprite.setDrag(100);             // deceleration when no acceleration applied
sprite.setFriction(0.5);        // friction with other bodies
sprite.setMaxVelocity(300, 600);
sprite.setMass(2);
sprite.setImmovable(true);      // won't move on collision (but still detects)
sprite.setCollideWorldBounds(true);
sprite.body.allowGravity = false;
sprite.body.setGravityY(500);   // per-object gravity override
sprite.body.setSize(24, 32);    // collision box size
sprite.body.setOffset(4, 16);   // collision box offset from sprite origin

// Check grounded
sprite.body.touching.down       // true if touching something below
sprite.body.blocked.down        // true if blocked by world bounds below
sprite.body.onFloor()           // touching OR blocked down
```

### Collisions and Overlaps
```javascript
// Collider: objects physically separate on contact
this.physics.add.collider(player, platforms);
this.physics.add.collider(player, enemies, hitEnemy, null, this);

// Overlap: objects pass through; callback fires
this.physics.add.overlap(player, coins, collectCoin, processCheck, this);
// processCheck(player, coin) => return true to trigger callback, false to skip

function collectCoin(player, coin) {
    coin.disableBody(true, true); // disableBody(disableGameObject, hideGameObject)
    score += 10;
}

// Group vs Group
this.physics.add.collider(bullets, enemies, bulletHitEnemy, null, this);

// World bounds collision callback
player.body.onWorldBounds = true;
this.physics.world.on('worldbounds', (body) => { /* body hit world edge */ });
```

### Move To / Velocity From Angle
```javascript
// Move toward a point at speed
this.physics.moveToObject(enemy, player, 100); // 100 = speed

// Move to coordinates
this.physics.moveTo(bullet, targetX, targetY, 300);

// Velocity from angle
this.physics.velocityFromAngle(angle, speed); // returns {x, y}
this.physics.velocityFromRotation(rotation, speed); // rotation in radians

// Angle between objects
const angle = Phaser.Math.Angle.Between(a.x, a.y, b.x, b.y);
```

## Matter.js Physics

Use when you need polygon collisions, joints, constraints, or realistic physics.

```javascript
// Config
physics: {
    default: 'matter',
    matter: { gravity: { y: 1 }, debug: true }
}

// Create body
const box = this.matter.add.sprite(200, 100, 'box');
box.setBody({ type: 'rectangle', width: 32, height: 32 });
// or: type: 'circle', radius: 16
// or: type: 'polygon', sides: 6, radius: 16

// Constraints (joints)
this.matter.add.constraint(bodyA, bodyB, length, stiffness);

// Collision events
this.matter.world.on('collisionstart', (event) => {
    event.pairs.forEach(pair => {
        const { bodyA, bodyB } = pair;
    });
});

// Labels for identification
box.setData('type', 'enemy');
```

## Input

### Keyboard
```javascript
// Cursor keys (arrow keys + space + shift)
const cursors = this.input.keyboard.createCursorKeys();
if (cursors.left.isDown) { }
if (cursors.space.isDown) { }

// WASD
const wasd = this.input.keyboard.addKeys('W,A,S,D');
if (wasd.W.isDown) { }

// Single key
const spaceKey = this.input.keyboard.addKey(Phaser.Input.Keyboard.KeyCodes.SPACE);

// Just pressed (true for one frame only)
if (Phaser.Input.Keyboard.JustDown(spaceKey)) { }
if (Phaser.Input.Keyboard.JustUp(spaceKey)) { }

// Key events
this.input.keyboard.on('keydown-SPACE', () => { });
```

### Pointer (Mouse / Touch)
```javascript
// Pointer position (screen coordinates)
const pointer = this.input.activePointer;
pointer.x; pointer.y;

// World coordinates (accounts for camera scroll)
pointer.worldX; pointer.worldY;

// Click/tap
if (pointer.isDown) { }

// Click on game object
sprite.setInteractive();
sprite.on('pointerdown', (pointer) => { });
sprite.on('pointerover', () => { /* hover */ });

// Drag
this.input.setDraggable(sprite);
this.input.on('drag', (pointer, gameObject, dragX, dragY) => {
    gameObject.x = dragX;
    gameObject.y = dragY;
});
```

### Gamepad
```javascript
this.input.gamepad.on('connected', (pad) => { });
// In update:
const pad = this.input.gamepad.getPad(0);
if (pad) {
    const axisH = pad.axes[0].getValue(); // -1 to 1
    if (pad.buttons[0].pressed) { /* A button */ }
}
```

## Camera

```javascript
const cam = this.cameras.main;

// Follow target
cam.startFollow(player, true, 0.1, 0.1); // lerp values for smoothing

// Deadzone (camera doesn't move unless target leaves this area)
cam.setDeadzone(200, 100);

// Bounds (don't scroll beyond level edges)
cam.setBounds(0, 0, levelWidth, levelHeight);

// Zoom
cam.setZoom(2); // 2x zoom in

// Shake
cam.shake(200, 0.01); // duration ms, intensity

// Flash
cam.flash(250, 255, 255, 255); // duration, r, g, b

// Fade
cam.fadeOut(500); // ms
cam.fadeIn(500);
cam.once('camerafadeoutcomplete', () => { this.scene.start('NextScene'); });

// Scroll manually
cam.scrollX = 100;
cam.scrollY = 200;
```

## Tweens

```javascript
this.tweens.add({
    targets: sprite,
    x: 400,
    y: 300,
    alpha: 0,
    scale: 2,
    angle: 360,
    duration: 1000,       // ms
    ease: 'Power2',       // Linear, Power1-4, Quad, Cubic, Sine, Elastic, Bounce, Back
    yoyo: true,           // reverse after completing
    repeat: -1,           // -1 = forever
    delay: 500,           // ms before starting
    onComplete: () => { },
    onUpdate: (tween) => { }
});

// Common eases: 'Linear', 'Quad.easeIn', 'Quad.easeOut', 'Quad.easeInOut',
//   'Cubic.easeIn', 'Sine.easeInOut', 'Elastic', 'Bounce', 'Back'

// Tween chain
this.tweens.chain({
    targets: sprite,
    tweens: [
        { x: 200, duration: 500 },
        { y: 400, duration: 500 },
        { alpha: 0, duration: 300 }
    ]
});
```

## Timers

```javascript
// Delayed call
this.time.delayedCall(2000, () => { /* runs after 2 seconds */ }, [], this);

// Repeating timer
this.time.addEvent({
    delay: 1000,
    callback: spawnEnemy,
    callbackScope: this,
    loop: true       // or repeat: 5 for fixed count
});

// Stop timer
const timer = this.time.addEvent({ ... });
timer.remove();
```

## Tilemap (Tiled Editor Integration)

```javascript
// In preload:
this.load.tilemapTiledJSON('map', 'level1.json');
this.load.image('tiles', 'tileset.png');

// In create:
const map = this.make.tilemap({ key: 'map' });
const tileset = map.addTilesetImage('tileset-name-in-tiled', 'tiles');

// Create layers (names must match Tiled layer names)
const groundLayer = map.createLayer('Ground', tileset, 0, 0);
const platformLayer = map.createLayer('Platforms', tileset, 0, 0);

// Set collision by tile property (set 'collides' in Tiled)
platformLayer.setCollisionByProperty({ collides: true });
// Or by tile index range
platformLayer.setCollisionBetween(1, 50);

// Physics collision with layer
this.physics.add.collider(player, platformLayer);

// Object layer (spawn points, etc.)
const spawnPoints = map.getObjectLayer('Spawns');
spawnPoints.objects.forEach(obj => {
    if (obj.name === 'playerSpawn') {
        player.setPosition(obj.x, obj.y);
    }
});

// Camera bounds from map size
this.cameras.main.setBounds(0, 0, map.widthInPixels, map.heightInPixels);
this.physics.world.setBounds(0, 0, map.widthInPixels, map.heightInPixels);
```

## Audio

```javascript
// In preload:
this.load.audio('bgm', 'music.mp3');
this.load.audio('jump', ['jump.ogg', 'jump.mp3']); // fallback formats

// In create:
const music = this.sound.add('bgm', { loop: true, volume: 0.5 });
music.play();

// Sound effect
this.sound.play('jump', { volume: 0.8, rate: 1.0 });

// With pitch variation (prevents repetitive feel)
this.sound.play('hit', { rate: Phaser.Math.FloatBetween(0.9, 1.1) });

// Pause all
this.sound.pauseAll();
```

## Particles

```javascript
// Phaser 3.60+ particle system
const emitter = this.add.particles(x, y, 'particle', {
    speed: { min: 50, max: 200 },
    angle: { min: 0, max: 360 },
    scale: { start: 1, end: 0 },
    alpha: { start: 1, end: 0 },
    lifespan: 800,
    quantity: 10,
    frequency: -1,       // -1 = explode mode (one burst)
    gravityY: 200
});
emitter.explode(20);     // emit 20 particles at once

// Follow a sprite
emitter.startFollow(player);
```

## Useful Math Utilities

```javascript
Phaser.Math.Between(min, max)            // random integer between min and max
Phaser.Math.FloatBetween(min, max)       // random float
Phaser.Math.Clamp(value, min, max)       // clamp value to range
Phaser.Math.Distance.Between(x1,y1,x2,y2) // Euclidean distance
Phaser.Math.Angle.Between(x1,y1,x2,y2)  // angle in radians
Phaser.Math.DegToRad(degrees)
Phaser.Math.RadToDeg(radians)
Phaser.Math.Linear(a, b, t)             // lerp
Phaser.Math.RND.pick(array)             // random element from array
Phaser.Math.RND.shuffle(array)          // shuffle in place
Phaser.Math.Wrap(value, min, max)       // wrap value within range
Phaser.Math.Snap.To(value, snap)        // snap to grid (e.g., snap to 32px grid)
```
