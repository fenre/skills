# AI & Pathfinding

## Finite State Machine (FSM)

The most common AI pattern for 2D games. Each enemy has a current state and transitions between states based on conditions.

### States and Transitions
```
         [Idle]
        /      \
  sees player   timer expires
      ↓            ↓
   [Chase] ←→ [Patrol]
      ↓
  in range
      ↓
   [Attack]
      ↓
  health low
      ↓
    [Flee]
```

### JavaScript Implementation (Phaser)
```javascript
class Enemy {
    constructor(scene, x, y) {
        this.sprite = scene.physics.add.sprite(x, y, 'enemy');
        this.state = 'patrol';
        this.patrolPoints = [{x: 100, y: 300}, {x: 500, y: 300}];
        this.patrolIndex = 0;
        this.detectionRange = 200;
        this.attackRange = 50;
    }

    update(time, delta, player) {
        const distToPlayer = Phaser.Math.Distance.Between(
            this.sprite.x, this.sprite.y, player.x, player.y
        );

        switch (this.state) {
            case 'patrol':
                this.patrol(delta);
                if (distToPlayer < this.detectionRange) this.state = 'chase';
                break;
            case 'chase':
                this.chase(player, delta);
                if (distToPlayer < this.attackRange) this.state = 'attack';
                if (distToPlayer > this.detectionRange * 1.5) this.state = 'patrol';
                break;
            case 'attack':
                this.attack(player);
                if (distToPlayer > this.attackRange * 1.5) this.state = 'chase';
                break;
            case 'flee':
                this.flee(player, delta);
                break;
        }
    }

    patrol(delta) {
        const target = this.patrolPoints[this.patrolIndex];
        const dist = Phaser.Math.Distance.Between(this.sprite.x, this.sprite.y, target.x, target.y);
        if (dist < 10) {
            this.patrolIndex = (this.patrolIndex + 1) % this.patrolPoints.length;
        }
        this.scene.physics.moveToObject(this.sprite, target, 60);
    }

    chase(player, delta) {
        this.scene.physics.moveToObject(this.sprite, player, 100);
    }

    flee(player, delta) {
        const angle = Phaser.Math.Angle.Between(player.x, player.y, this.sprite.x, this.sprite.y);
        this.sprite.setVelocity(Math.cos(angle) * 120, Math.sin(angle) * 120);
    }
}
```

### C# Implementation (Unity)
```csharp
public class EnemyAI : MonoBehaviour
{
    private enum State { Idle, Patrol, Chase, Attack, Flee }

    [SerializeField] private float detectionRange = 5f;
    [SerializeField] private float attackRange = 1.5f;
    [SerializeField] private float patrolSpeed = 2f;
    [SerializeField] private float chaseSpeed = 4f;
    [SerializeField] private Transform[] waypoints;

    private State currentState = State.Patrol;
    private Transform player;
    private Rigidbody2D rb;
    private int waypointIndex;

    void Start()
    {
        player = GameObject.FindWithTag("Player").transform;
        rb = GetComponent<Rigidbody2D>();
    }

    void Update()
    {
        float distToPlayer = Vector2.Distance(transform.position, player.position);

        switch (currentState)
        {
            case State.Patrol:
                Patrol();
                if (distToPlayer < detectionRange) currentState = State.Chase;
                break;
            case State.Chase:
                Chase();
                if (distToPlayer < attackRange) currentState = State.Attack;
                if (distToPlayer > detectionRange * 1.5f) currentState = State.Patrol;
                break;
            case State.Attack:
                Attack();
                if (distToPlayer > attackRange * 1.5f) currentState = State.Chase;
                break;
            case State.Flee:
                Flee();
                break;
        }
    }

    void Patrol()
    {
        if (waypoints.Length == 0) return;
        Transform target = waypoints[waypointIndex];
        Vector2 dir = (target.position - transform.position).normalized;
        rb.linearVelocity = dir * patrolSpeed;

        if (Vector2.Distance(transform.position, target.position) < 0.5f)
            waypointIndex = (waypointIndex + 1) % waypoints.Length;
    }

    void Chase()
    {
        Vector2 dir = (player.position - transform.position).normalized;
        rb.linearVelocity = dir * chaseSpeed;
    }

    void Attack() { rb.linearVelocity = Vector2.zero; /* attack logic */ }

    void Flee()
    {
        Vector2 dir = (transform.position - player.position).normalized;
        rb.linearVelocity = dir * chaseSpeed;
    }
}
```

---

## Patrol Patterns

### Waypoint Patrol
Move between predefined points in order. See FSM examples above.

### Edge Detection Patrol (Platformer)
Enemy walks until it detects an edge (no ground ahead) or a wall, then turns around.

```csharp
// Unity
[SerializeField] private float speed = 2f;
[SerializeField] private LayerMask groundLayer;
private int direction = 1; // 1 = right, -1 = left

void FixedUpdate()
{
    rb.linearVelocity = new Vector2(direction * speed, rb.linearVelocity.y);

    // Check for edge (raycast down-forward)
    Vector2 edgeCheckPos = (Vector2)transform.position + new Vector2(direction * 0.5f, -0.5f);
    bool groundAhead = Physics2D.Raycast(edgeCheckPos, Vector2.down, 1f, groundLayer);

    // Check for wall (raycast forward)
    bool wallAhead = Physics2D.Raycast(transform.position, Vector2.right * direction, 0.6f, groundLayer);

    if (!groundAhead || wallAhead)
    {
        direction *= -1;
        transform.localScale = new Vector3(direction, 1, 1);
    }
}
```

```javascript
// Phaser
update() {
    this.sprite.setVelocityX(this.direction * this.speed);

    // Edge detection: check tile below + ahead
    const tileAhead = map.getTileAtWorldXY(
        this.sprite.x + this.direction * 20,
        this.sprite.y + 20
    );
    if (!tileAhead) {
        this.direction *= -1;
        this.sprite.setFlipX(this.direction < 0);
    }
}
```

---

## Line of Sight

Check if enemy can actually see the player (no walls blocking).

```csharp
// Unity
bool HasLineOfSight(Vector2 from, Vector2 to, LayerMask obstacleLayer)
{
    Vector2 direction = to - from;
    RaycastHit2D hit = Physics2D.Raycast(from, direction.normalized, direction.magnitude, obstacleLayer);
    return hit.collider == null; // true if no obstacles between
}

// Usage: only chase if line of sight is clear
if (distToPlayer < detectionRange && HasLineOfSight(transform.position, player.position, wallLayer))
    currentState = State.Chase;
```

```javascript
// Phaser (using Arcade ray)
hasLineOfSight(from, to) {
    const line = new Phaser.Geom.Line(from.x, from.y, to.x, to.y);
    const tiles = groundLayer.getTilesWithinShape(line);
    return !tiles.some(tile => tile.index !== -1); // -1 = empty tile
}
```

---

## A* Pathfinding

Grid-based pathfinding algorithm. Finds shortest path from start to goal.

### Algorithm
```
1. Create open set (nodes to evaluate) with start node
2. Create closed set (already evaluated)
3. While open set is not empty:
   a. Pick node with lowest f = g + h from open set
   b. If it's the goal → reconstruct path
   c. Move it to closed set
   d. For each neighbor:
      - Skip if in closed set or not walkable
      - Calculate tentative g (cost from start)
      - If new g is lower OR neighbor not in open set:
        - Update g, h, parent
        - Add to open set if not already there
4. If open set empty → no path exists
```

- **g** = actual cost from start to current node
- **h** = estimated cost from current to goal (heuristic)
- **f** = g + h (total estimated cost)

### Heuristics
| Heuristic | Formula | Use When |
|-----------|---------|----------|
| Manhattan | `abs(dx) + abs(dy)` | 4-directional movement (no diagonals) |
| Euclidean | `sqrt(dx² + dy²)` | Any-direction movement |
| Chebyshev | `max(abs(dx), abs(dy))` | 8-directional (diagonal cost = 1) |
| Octile | `max(abs(dx), abs(dy)) + 0.414 * min(abs(dx), abs(dy))` | 8-directional (diagonal cost = √2) |

### JavaScript Implementation
```javascript
function aStar(grid, start, goal) {
    const openSet = [start];
    const closedSet = new Set();
    const cameFrom = new Map();
    const gScore = new Map();
    const fScore = new Map();

    const key = (n) => `${n.x},${n.y}`;
    gScore.set(key(start), 0);
    fScore.set(key(start), heuristic(start, goal));

    while (openSet.length > 0) {
        openSet.sort((a, b) => (fScore.get(key(a)) || Infinity) - (fScore.get(key(b)) || Infinity));
        const current = openSet.shift();

        if (current.x === goal.x && current.y === goal.y)
            return reconstructPath(cameFrom, current);

        closedSet.add(key(current));

        for (const neighbor of getNeighbors(grid, current)) {
            if (closedSet.has(key(neighbor))) continue;

            const tentativeG = (gScore.get(key(current)) || 0) + 1;
            if (tentativeG < (gScore.get(key(neighbor)) || Infinity)) {
                cameFrom.set(key(neighbor), current);
                gScore.set(key(neighbor), tentativeG);
                fScore.set(key(neighbor), tentativeG + heuristic(neighbor, goal));
                if (!openSet.some(n => n.x === neighbor.x && n.y === neighbor.y))
                    openSet.push(neighbor);
            }
        }
    }
    return null; // no path
}

function heuristic(a, b) {
    return Math.abs(a.x - b.x) + Math.abs(a.y - b.y); // Manhattan
}

function getNeighbors(grid, node) {
    const dirs = [{x:0,y:-1},{x:0,y:1},{x:-1,y:0},{x:1,y:0}]; // 4-directional
    return dirs
        .map(d => ({x: node.x + d.x, y: node.y + d.y}))
        .filter(n => n.x >= 0 && n.y >= 0 && n.x < grid[0].length && n.y < grid.length
                     && grid[n.y][n.x] === 0); // 0 = walkable
}

function reconstructPath(cameFrom, current) {
    const path = [current];
    const key = (n) => `${n.x},${n.y}`;
    while (cameFrom.has(key(current))) {
        current = cameFrom.get(key(current));
        path.unshift(current);
    }
    return path;
}
```

### Following the Path
```javascript
// Move enemy along path points
if (path && path.length > 0) {
    const target = path[0];
    const worldX = target.x * tileSize + tileSize / 2;
    const worldY = target.y * tileSize + tileSize / 2;
    const dist = Phaser.Math.Distance.Between(enemy.x, enemy.y, worldX, worldY);

    if (dist < 4) {
        path.shift(); // arrived at waypoint; move to next
    } else {
        scene.physics.moveToObject(enemy, {x: worldX, y: worldY}, moveSpeed);
    }
}
```

### Performance Tips
- Recalculate path periodically (e.g., every 0.5s), not every frame
- Limit search area (max iterations) to prevent lag
- Use a priority queue (binary heap) instead of array sort for open set
- Cache paths when target hasn't moved significantly

---

## Flocking / Boids

Emergent swarm behavior from three simple rules applied per entity.

### Rules
1. **Separation**: Steer away from nearby neighbors (avoid crowding)
2. **Alignment**: Steer toward average heading of nearby neighbors
3. **Cohesion**: Steer toward average position of nearby neighbors

```javascript
function updateBoid(boid, allBoids, dt) {
    let separation = {x: 0, y: 0};
    let alignment = {x: 0, y: 0};
    let cohesion = {x: 0, y: 0};
    let neighborCount = 0;

    for (const other of allBoids) {
        if (other === boid) continue;
        const dist = distance(boid, other);

        if (dist < perceptionRadius) {
            neighborCount++;

            // Separation: steer away from close neighbors
            if (dist < separationRadius) {
                separation.x += (boid.x - other.x) / dist;
                separation.y += (boid.y - other.y) / dist;
            }

            // Alignment: match velocity of neighbors
            alignment.x += other.vx;
            alignment.y += other.vy;

            // Cohesion: move toward center of neighbors
            cohesion.x += other.x;
            cohesion.y += other.y;
        }
    }

    if (neighborCount > 0) {
        alignment.x /= neighborCount;
        alignment.y /= neighborCount;
        cohesion.x = cohesion.x / neighborCount - boid.x;
        cohesion.y = cohesion.y / neighborCount - boid.y;
    }

    // Apply weighted forces
    boid.vx += separation.x * sepWeight + alignment.x * aliWeight + cohesion.x * cohWeight;
    boid.vy += separation.y * sepWeight + alignment.y * aliWeight + cohesion.y * cohWeight;

    // Clamp speed
    const speed = Math.sqrt(boid.vx * boid.vx + boid.vy * boid.vy);
    if (speed > maxSpeed) {
        boid.vx = (boid.vx / speed) * maxSpeed;
        boid.vy = (boid.vy / speed) * maxSpeed;
    }

    boid.x += boid.vx * dt;
    boid.y += boid.vy * dt;
}
```

Use for: fish schools, bird flocks, swarm enemies, particle-like crowd movement.

---

## Difficulty Scaling

### Progressive Difficulty
```javascript
// Increase difficulty based on score, time, or wave number
function getDifficultySettings(wave) {
    return {
        enemyCount: Math.min(3 + wave * 2, 20),
        enemySpeed: Math.min(60 + wave * 10, 200),
        spawnInterval: Math.max(3000 - wave * 200, 500), // ms, minimum 500
        enemyHealth: 1 + Math.floor(wave / 3)
    };
}
```

### Director-Based (adaptive difficulty)
Track player performance and adjust:
- Player dying a lot → reduce enemy count/speed
- Player cruising → increase spawn rate, add new enemy types
- Track: deaths per minute, health when finishing wave, time to clear wave
