# Collision Detection & Physics Fundamentals

## AABB (Axis-Aligned Bounding Box) Collision

Two rectangles overlap when ALL four conditions are true:
```
A.left < B.right   AND
A.right > B.left   AND
A.top < B.bottom   AND
A.bottom > B.top
```

### JavaScript Implementation
```javascript
function aabbCollision(a, b) {
    return (
        a.x < b.x + b.width &&
        a.x + a.width > b.x &&
        a.y < b.y + b.height &&
        a.y + a.height > b.y
    );
}
```

### With Overlap Resolution
```javascript
function resolveAABB(a, b) {
    const overlapX = Math.min(a.x + a.width, b.x + b.width) - Math.max(a.x, b.x);
    const overlapY = Math.min(a.y + a.height, b.y + b.height) - Math.max(a.y, b.y);

    if (overlapX < overlapY) {
        if (a.x < b.x) a.x -= overlapX; else a.x += overlapX;
    } else {
        if (a.y < b.y) a.y -= overlapY; else a.y += overlapY;
    }
}
```

### Phaser Arcade physics uses AABB only. No rotation support in Arcade mode.

---

## Circle-Circle Collision

Two circles collide when the distance between centers is less than the sum of their radii:
```
distance(a.center, b.center) < a.radius + b.radius
```

```javascript
function circleCollision(a, b) {
    const dx = a.x - b.x;
    const dy = a.y - b.y;
    const distSq = dx * dx + dy * dy;
    const radiusSum = a.radius + b.radius;
    return distSq < radiusSum * radiusSum; // compare squared to avoid sqrt
}
```

---

## Circle-AABB Collision

Find the closest point on the rectangle to the circle center, then check distance.

```javascript
function circleRectCollision(circle, rect) {
    const closestX = Math.max(rect.x, Math.min(circle.x, rect.x + rect.width));
    const closestY = Math.max(rect.y, Math.min(circle.y, rect.y + rect.height));
    const dx = circle.x - closestX;
    const dy = circle.y - closestY;
    return (dx * dx + dy * dy) < (circle.radius * circle.radius);
}
```

---

## Point-in-Rectangle

```javascript
function pointInRect(px, py, rect) {
    return px >= rect.x && px <= rect.x + rect.width &&
           py >= rect.y && py <= rect.y + rect.height;
}
```

## Point-in-Circle

```javascript
function pointInCircle(px, py, circle) {
    const dx = px - circle.x;
    const dy = py - circle.y;
    return dx * dx + dy * dy <= circle.radius * circle.radius;
}
```

---

## SAT (Separating Axis Theorem)

For convex polygons, two shapes do NOT collide if there exists any axis along which their projections don't overlap.

Algorithm:
1. Get all edge normals from both polygons
2. For each normal, project both polygons onto that axis
3. If projections don't overlap on ANY axis → no collision
4. If ALL axes overlap → collision

Use when: complex shapes, rotated rectangles, polygon collisions.
Avoid when: simple AABB or circles suffice (SAT is slower).

Phaser's Matter.js physics uses SAT internally.

---

## Spatial Partitioning

When you have many objects, checking every pair (O(n^2)) is expensive. Spatial partitioning reduces the number of checks.

### Grid-Based

Divide the world into a grid of cells. Only check collisions between objects in the same or adjacent cells.

```javascript
class SpatialGrid {
    constructor(cellSize) {
        this.cellSize = cellSize;
        this.cells = new Map();
    }

    getKey(x, y) {
        const cx = Math.floor(x / this.cellSize);
        const cy = Math.floor(y / this.cellSize);
        return `${cx},${cy}`;
    }

    insert(entity) {
        const key = this.getKey(entity.x, entity.y);
        if (!this.cells.has(key)) this.cells.set(key, []);
        this.cells.get(key).push(entity);
    }

    query(entity) {
        const cx = Math.floor(entity.x / this.cellSize);
        const cy = Math.floor(entity.y / this.cellSize);
        const nearby = [];
        for (let dx = -1; dx <= 1; dx++) {
            for (let dy = -1; dy <= 1; dy++) {
                const key = `${cx + dx},${cy + dy}`;
                if (this.cells.has(key)) nearby.push(...this.cells.get(key));
            }
        }
        return nearby;
    }

    clear() { this.cells.clear(); }
}
```

Best for: uniform distribution, most 2D games. Cell size = ~2x largest entity.

### Quadtree

Recursively subdivides space into 4 quadrants when a node exceeds capacity.

Best for: non-uniform distribution, large worlds with clustered entities.

---

## Physics Fundamentals

### Frame-Rate Independent Movement

ALWAYS multiply by delta time:
```
position += velocity * deltaTime
velocity += acceleration * deltaTime
```

Without delta time, movement speed depends on frame rate. At 60 FPS the game runs correctly but at 30 FPS everything moves at half speed.

### Velocity and Acceleration

```javascript
// Basic physics update
entity.velocityX += entity.accelerationX * dt;
entity.velocityY += entity.accelerationY * dt;
entity.x += entity.velocityX * dt;
entity.y += entity.velocityY * dt;

// Apply friction (decelerate when no input)
entity.velocityX *= (1 - friction * dt);
// Or: entity.velocityX = moveTowards(entity.velocityX, 0, friction * dt);

// Gravity
entity.velocityY += gravity * dt;

// Clamp velocity
entity.velocityX = clamp(entity.velocityX, -maxSpeed, maxSpeed);
entity.velocityY = clamp(entity.velocityY, -maxFallSpeed, maxFallSpeed);
```

### Terminal Velocity
Clamp the downward velocity so objects don't fall infinitely fast:
```
velocityY = min(velocityY, terminalVelocity)
```

### Restitution (Bounciness)
```
// On collision:
velocityY = -velocityY * restitution   // 0 = no bounce, 1 = full bounce
```

---

## Fixed Timestep vs Variable Timestep

### Variable Timestep (deltaTime approach)
- `update(deltaTime)` where deltaTime varies each frame
- Simple; works for most 2D games
- Can cause issues with fast objects (tunneling) at low frame rates

### Fixed Timestep
- Physics updates at fixed interval regardless of frame rate
- Accumulate time, consume in fixed-size chunks
- Unity does this automatically: `FixedUpdate()` runs at fixed rate
- Phaser Arcade: `this.physics.world.fixedStep` controls this

```javascript
// Manual fixed timestep (framework-agnostic)
const FIXED_DT = 1 / 60; // 60 Hz
let accumulator = 0;

function gameLoop(realDt) {
    accumulator += realDt;
    while (accumulator >= FIXED_DT) {
        physicsUpdate(FIXED_DT);
        accumulator -= FIXED_DT;
    }
    render();
}
```

---

## Raycasting (2D)

Cast a line from A to B and find the first thing it hits.

### Use Cases
- Ground detection (cast down from player feet)
- Line-of-sight checks (can enemy see player?)
- Bullet hit detection (instant projectiles)
- Wall detection for AI

### JavaScript Ray vs Line Segment
```javascript
function rayVsRect(rayOrigin, rayDir, rect) {
    const invDirX = 1 / rayDir.x;
    const invDirY = 1 / rayDir.y;

    const t1 = (rect.x - rayOrigin.x) * invDirX;
    const t2 = (rect.x + rect.width - rayOrigin.x) * invDirX;
    const t3 = (rect.y - rayOrigin.y) * invDirY;
    const t4 = (rect.y + rect.height - rayOrigin.y) * invDirY;

    const tMin = Math.max(Math.min(t1, t2), Math.min(t3, t4));
    const tMax = Math.min(Math.max(t1, t2), Math.max(t3, t4));

    if (tMax < 0 || tMin > tMax) return null; // no hit

    return {
        t: tMin,
        point: { x: rayOrigin.x + rayDir.x * tMin, y: rayOrigin.y + rayDir.y * tMin }
    };
}
```

### Unity 2D Raycast
```csharp
RaycastHit2D hit = Physics2D.Raycast(origin, direction, maxDistance, layerMask);
if (hit.collider != null) {
    // hit.point, hit.normal, hit.distance, hit.collider.gameObject
}

// Ground check
bool grounded = Physics2D.Raycast(transform.position + Vector3.down * 0.5f,
    Vector2.down, 0.1f, groundLayer);
```

---

## Tunneling Prevention

Fast-moving objects can pass through thin colliders between frames.

Solutions:
1. **Continuous collision detection**: Unity `Rigidbody2D.collisionDetectionMode = Continuous`
2. **Sweep test**: cast the body's shape along its velocity vector
3. **Increase collider thickness**: make walls thicker than max velocity * dt
4. **Reduce fixed timestep**: smaller physics steps = less tunneling
5. **Raycast ahead**: cast a ray in the velocity direction to detect walls before moving
