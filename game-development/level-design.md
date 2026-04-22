# Level Design, Procedural Generation & Systems

## Tiled Map Editor

[Tiled](https://www.mapeditor.org/) is the standard free tool for creating 2D tile-based levels.

### Tiled Concepts
| Concept | Description |
|---------|-------------|
| **Tileset** | Image containing all tiles in a grid (e.g., 16x16 or 32x32 per tile) |
| **Map** | Grid of tile references organized into layers |
| **Tile Layer** | Grid layer for visual tiles (ground, walls, decorations) |
| **Object Layer** | Free-form layer for spawn points, triggers, zones (rectangles, polygons, points) |
| **Properties** | Custom key-value pairs on tiles, objects, or layers (e.g., "collides": true) |

### Recommended Layer Structure
```
Map
├── Background    (tile layer — sky, distant scenery)
├── Ground        (tile layer — walkable ground, walls)
├── Platforms     (tile layer — platforms, bridges)
├── Foreground    (tile layer — decorations rendered in front of player)
├── Collisions    (tile layer — invisible collision tiles, or use properties on ground)
├── Spawns        (object layer — player spawn, enemy spawns)
└── Triggers      (object layer — doors, checkpoints, collectible positions)
```

### Tile Properties
In Tiled: select a tile in the tileset, add custom properties:
- `collides: true` — for collision detection
- `deadly: true` — for hazard tiles (spikes, lava)
- `oneWay: true` — for platforms you can jump through

### Export
- **Phaser**: Export as JSON (File > Export As > JSON)
- **Unity**: Export as TMX (default) → import via SuperTiled2Unity package

---

## Loading Tiled Maps in Phaser

```javascript
// Preload
this.load.tilemapTiledJSON('level1', 'maps/level1.json');
this.load.image('tileset', 'images/tileset.png');

// Create
const map = this.make.tilemap({ key: 'level1' });

// addTilesetImage(name-in-Tiled, key-in-Phaser, tileWidth, tileHeight, margin, spacing)
const tileset = map.addTilesetImage('my-tileset', 'tileset');

// Create layers by name (must match Tiled layer names exactly)
const bgLayer = map.createLayer('Background', tileset, 0, 0);
const groundLayer = map.createLayer('Ground', tileset, 0, 0);
const fgLayer = map.createLayer('Foreground', tileset, 0, 0);

// Set depth for rendering order
bgLayer.setDepth(0);
groundLayer.setDepth(1);
fgLayer.setDepth(10); // above player

// Collision
groundLayer.setCollisionByProperty({ collides: true });
this.physics.add.collider(player, groundLayer);

// Object layer — spawn points, collectibles
const objectLayer = map.getObjectLayer('Spawns');
objectLayer.objects.forEach(obj => {
    if (obj.type === 'PlayerSpawn') {
        player.setPosition(obj.x, obj.y);
    }
    if (obj.type === 'EnemySpawn') {
        enemies.create(obj.x, obj.y, 'enemy');
    }
    if (obj.type === 'Coin') {
        coins.create(obj.x, obj.y, 'coin');
    }
});

// Set camera and world bounds from map dimensions
this.cameras.main.setBounds(0, 0, map.widthInPixels, map.heightInPixels);
this.physics.world.setBounds(0, 0, map.widthInPixels, map.heightInPixels);
```

---

## Loading Tiled Maps in Unity

### Using SuperTiled2Unity (recommended)
1. Install SuperTiled2Unity via Package Manager (git URL or .unitypackage)
2. Place `.tmx` and tileset images in Assets folder
3. SuperTiled2Unity auto-imports and creates prefab
4. Drag map prefab into scene
5. Collisions auto-generated from Tiled tile properties or object layers

### Manual Tilemap (without Tiled)
1. Create > 2D Object > Tilemap > Rectangular
2. Window > 2D > Tile Palette → create palette
3. Drag tileset sprite (sliced) into palette
4. Paint tiles with brush, rectangle, fill tools

### Tilemap Collision (Unity)
```
Tilemap GameObject
├── TilemapRenderer
├── TilemapCollider2D          (generates per-tile colliders)
├── CompositeCollider2D        (merges adjacent colliders — much better performance)
└── Rigidbody2D (Body Type: Static)  (required by CompositeCollider2D)
```
Set `TilemapCollider2D → Used by Composite = true`.

---

## Procedural Level Generation

### Random Room Placement (Dungeon)
```javascript
function generateDungeon(width, height, numRooms) {
    const grid = Array.from({length: height}, () => Array(width).fill(1)); // 1 = wall
    const rooms = [];

    for (let i = 0; i < numRooms; i++) {
        const roomW = randInt(4, 10);
        const roomH = randInt(4, 8);
        const roomX = randInt(1, width - roomW - 1);
        const roomY = randInt(1, height - roomH - 1);

        // Check overlap with existing rooms
        const overlaps = rooms.some(r =>
            roomX < r.x + r.w + 1 && roomX + roomW + 1 > r.x &&
            roomY < r.y + r.h + 1 && roomY + roomH + 1 > r.y
        );
        if (overlaps) continue;

        // Carve room (set to 0 = floor)
        for (let y = roomY; y < roomY + roomH; y++)
            for (let x = roomX; x < roomX + roomW; x++)
                grid[y][x] = 0;

        // Connect to previous room with corridors
        if (rooms.length > 0) {
            const prev = rooms[rooms.length - 1];
            const cx = Math.floor(roomX + roomW / 2);
            const cy = Math.floor(roomY + roomH / 2);
            const px = Math.floor(prev.x + prev.w / 2);
            const py = Math.floor(prev.y + prev.h / 2);

            // Horizontal then vertical corridor
            for (let x = Math.min(cx, px); x <= Math.max(cx, px); x++) grid[cy][x] = 0;
            for (let y = Math.min(cy, py); y <= Math.max(cy, py); y++) grid[y][px] = 0;
        }
        rooms.push({x: roomX, y: roomY, w: roomW, h: roomH});
    }
    return { grid, rooms };
}
```

### BSP (Binary Space Partitioning) Dungeon
1. Start with entire map as one region
2. Split randomly (horizontal or vertical) into two children
3. Recurse until regions are small enough for rooms
4. Place one room in each leaf
5. Connect sibling rooms with corridors

More structured than random placement; guarantees connectivity and good room distribution.

### Wave Function Collapse (WFC) — Concept
- Define tiles with adjacency rules (which tiles can neighbor which on each side)
- Start with all cells "uncollapsed" (all tiles possible)
- Pick lowest-entropy cell (fewest possibilities), collapse to one tile
- Propagate constraints to neighbors
- Repeat until all cells resolved

Use for: natural-looking terrain, varied map layouts that respect tile connectivity rules.
Libraries: `wfc` npm for JS, `WaveFunctionCollapse` asset for Unity.

---

## Infinite / Endless Generation

### Chunk-Based Spawning
```javascript
// Spawn new content ahead of the player, despawn behind
const CHUNK_SIZE = 800; // pixels

update() {
    const playerChunk = Math.floor(player.x / CHUNK_SIZE);

    // Spawn new chunk ahead
    if (playerChunk > lastSpawnedChunk) {
        spawnChunk(playerChunk + 1);
        lastSpawnedChunk = playerChunk;
    }

    // Despawn old chunks (2+ chunks behind)
    cleanupChunks(playerChunk - 2);
}

function spawnChunk(chunkIndex) {
    const startX = chunkIndex * CHUNK_SIZE;
    // Generate platforms, enemies, coins for this section
    for (let i = 0; i < 5; i++) {
        const px = startX + Phaser.Math.Between(50, CHUNK_SIZE - 50);
        const py = Phaser.Math.Between(300, 500);
        platforms.create(px, py, 'platform');
    }
}

function cleanupChunks(belowChunk) {
    platforms.getChildren().forEach(p => {
        if (Math.floor(p.x / CHUNK_SIZE) < belowChunk) {
            p.destroy();
        }
    });
}
```

### Difficulty Over Distance
```javascript
function getChunkDifficulty(chunkIndex) {
    return {
        gapWidth: Math.min(50 + chunkIndex * 5, 200),
        platformWidth: Math.max(150 - chunkIndex * 3, 60),
        enemyChance: Math.min(0.1 + chunkIndex * 0.05, 0.7),
        movingPlatforms: chunkIndex > 5,
        hazards: chunkIndex > 10
    };
}
```

---

## Scene / Level Management

### Phaser
```javascript
// Pass data between scenes
this.scene.start('Game', { level: 2, score: 500 });

// In receiving scene's init():
init(data) {
    this.level = data.level || 1;
    this.score = data.score || 0;
}

// Persistent data across scenes (use Phaser registry)
this.registry.set('highScore', 9999);
const hs = this.registry.get('highScore');

// Or use a global data object
this.game.globals = { score: 0, lives: 3, level: 1 };
```

### Unity
```csharp
// Load scene
SceneManager.LoadScene("Level2");
SceneManager.LoadScene(SceneManager.GetActiveScene().buildIndex + 1);

// Persist object across scenes
DontDestroyOnLoad(gameObject); // put on a GameManager

// Pass data: use a static class or singleton
public class GameData {
    public static int score = 0;
    public static int lives = 3;
    public static int currentLevel = 1;
}
```

---

## Save / Load

### Phaser (Browser — localStorage)
```javascript
// Save
function saveGame() {
    const saveData = {
        level: currentLevel,
        score: score,
        position: { x: player.x, y: player.y },
        inventory: inventory,
        timestamp: Date.now()
    };
    localStorage.setItem('gameSave', JSON.stringify(saveData));
}

// Load
function loadGame() {
    const raw = localStorage.getItem('gameSave');
    if (!raw) return null;
    return JSON.parse(raw);
}

// Delete
function deleteSave() {
    localStorage.removeItem('gameSave');
}

// Check if save exists
function hasSave() {
    return localStorage.getItem('gameSave') !== null;
}
```

### Unity (PlayerPrefs — simple)
```csharp
// Save simple values
PlayerPrefs.SetInt("HighScore", score);
PlayerPrefs.SetFloat("MusicVolume", 0.7f);
PlayerPrefs.SetString("PlayerName", name);
PlayerPrefs.Save();

// Load
int hs = PlayerPrefs.GetInt("HighScore", 0); // 0 = default
bool hasSave = PlayerPrefs.HasKey("HighScore");

// Delete
PlayerPrefs.DeleteKey("HighScore");
PlayerPrefs.DeleteAll();
```

### Unity (JSON File — complex data)
```csharp
[System.Serializable]
public class SaveData {
    public int level;
    public int score;
    public float playerX;
    public float playerY;
    public List<string> inventory;
}

public static void Save(SaveData data) {
    string json = JsonUtility.ToJson(data, true);
    string path = Path.Combine(Application.persistentDataPath, "save.json");
    File.WriteAllText(path, json);
}

public static SaveData Load() {
    string path = Path.Combine(Application.persistentDataPath, "save.json");
    if (!File.Exists(path)) return null;
    string json = File.ReadAllText(path);
    return JsonUtility.FromJson<SaveData>(json);
}
```

---

## Object Pooling

Avoid `Instantiate`/`Destroy` (Unity) or `create`/`destroy` (Phaser) in hot paths. Reuse objects.

### Phaser Object Pool Pattern
```javascript
// Create group with max size
const bullets = this.physics.add.group({
    defaultKey: 'bullet',
    maxSize: 30
});

// Get from pool
function fireBullet(x, y, vx, vy) {
    const bullet = bullets.get(x, y);
    if (!bullet) return; // pool exhausted
    bullet.setActive(true).setVisible(true);
    bullet.body.enable = true;
    bullet.setVelocity(vx, vy);
}

// Return to pool (e.g., on world bounds or collision)
function deactivateBullet(bullet) {
    bullet.setActive(false).setVisible(false);
    bullet.body.enable = false;
    bullet.body.stop(); // zero velocity
}

// Auto-deactivate when leaving world
this.physics.world.on('worldbounds', (body) => {
    deactivateBullet(body.gameObject);
});
// Set on bullet: bullet.body.onWorldBounds = true; bullet.setCollideWorldBounds(true);
```

### Unity Object Pool
```csharp
using UnityEngine.Pool;

public class BulletSpawner : MonoBehaviour
{
    [SerializeField] private Bullet bulletPrefab;
    private ObjectPool<Bullet> pool;

    void Awake()
    {
        pool = new ObjectPool<Bullet>(
            createFunc: () => {
                var b = Instantiate(bulletPrefab);
                b.SetPool(pool);
                return b;
            },
            actionOnGet: b => b.gameObject.SetActive(true),
            actionOnRelease: b => b.gameObject.SetActive(false),
            actionOnDestroy: b => Destroy(b.gameObject),
            defaultCapacity: 20,
            maxSize: 50
        );
    }

    public void Fire(Vector2 position, Vector2 velocity)
    {
        var bullet = pool.Get();
        bullet.transform.position = position;
        bullet.Init(velocity);
    }
}

// Bullet script:
public class Bullet : MonoBehaviour
{
    private ObjectPool<Bullet> pool;
    public void SetPool(ObjectPool<Bullet> p) => pool = p;
    public void ReturnToPool() => pool.Release(this);

    void OnBecameInvisible() { ReturnToPool(); }
}
```

---

## Performance Tips

1. **Object pooling**: Always pool bullets, particles, enemies that spawn/die frequently
2. **Spatial partitioning**: Use grid/quadtree for collision when entity count > 50-100
3. **Minimize raycasts**: Cache ground checks; don't raycast every frame if not needed
4. **Sprite atlases**: Combine sprites into atlases to reduce draw calls
5. **Culling**: Don't update off-screen entities (Phaser does this with camera culling; Unity needs manual handling or Tilemap culling)
6. **Avoid Find in Update**: Cache references in Awake/Start (Unity)
7. **Tilemap CompositeCollider2D**: Always use composite for Tilemap collisions in Unity
8. **Phaser groups**: Use `runChildUpdate: false` on groups where you manually update children
