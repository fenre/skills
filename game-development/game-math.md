# Essential Game Math

## Vector2 Operations

A 2D vector represents direction and magnitude: `(x, y)`.

### Basic Operations
```javascript
// Addition: position + velocity
result = { x: a.x + b.x, y: a.y + b.y };

// Subtraction: direction from a to b
direction = { x: b.x - a.x, y: b.y - a.y };

// Scale: multiply by scalar
scaled = { x: v.x * s, y: v.y * s };

// Magnitude (length)
magnitude = Math.sqrt(v.x * v.x + v.y * v.y);

// Magnitude squared (faster; use for comparisons to avoid sqrt)
magSq = v.x * v.x + v.y * v.y;

// Normalize (unit vector; length = 1)
len = Math.sqrt(v.x * v.x + v.y * v.y);
if (len > 0) normalized = { x: v.x / len, y: v.y / len };

// Dot product: a.x*b.x + a.y*b.y
// Result > 0: same direction; = 0: perpendicular; < 0: opposite direction
dot = a.x * b.x + a.y * b.y;

// Perpendicular (rotate 90 degrees)
perp = { x: -v.y, y: v.x };

// Distance between two points
dist = Math.sqrt((b.x-a.x)**2 + (b.y-a.y)**2);
```

### Unity Vector2 API
```csharp
Vector2 a = new Vector2(3, 4);
a.magnitude;           // 5
a.sqrMagnitude;        // 25 (faster for comparisons)
a.normalized;          // (0.6, 0.8)
Vector2.Distance(a, b);
Vector2.Dot(a, b);
Vector2.Perpendicular(v);  // rotate 90° counterclockwise
Vector2.Reflect(direction, normal);
Vector2.zero;   // (0, 0)
Vector2.one;    // (1, 1)
Vector2.up;     // (0, 1)
Vector2.down;   // (0, -1)
Vector2.left;   // (-1, 0)
Vector2.right;  // (1, 0)
```

---

## Lerp (Linear Interpolation)

Blend between two values based on t (0 to 1):
```
lerp(a, b, t) = a + (b - a) * t
```

When t=0 → returns a. When t=1 → returns b. When t=0.5 → returns midpoint.

### Uses
- Smooth camera follow
- Smooth movement to target
- Color transitions
- Health bar fill animation
- Fade in/out

### Frame-Rate Independent Lerp
```javascript
// Smooth approach (frame-rate independent)
// smoothing: 0.01 = very smooth, 0.1 = medium, 0.5 = fast
current = lerp(current, target, 1 - Math.pow(smoothing, deltaTime));
```

```csharp
// Unity: built-in Lerp
float val = Mathf.Lerp(current, target, speed * Time.deltaTime);
Vector2 pos = Vector2.Lerp(transform.position, target, speed * Time.deltaTime);
Color c = Color.Lerp(Color.red, Color.blue, t);

// MoveTowards: linear approach at fixed speed (reaches target exactly)
float val = Mathf.MoveTowards(current, target, speed * Time.deltaTime);
Vector2 pos = Vector2.MoveTowards(current, target, speed * Time.deltaTime);

// SmoothDamp: critically damped spring (best for camera; no overshoot)
float velocity = 0f;
current = Mathf.SmoothDamp(current, target, ref velocity, smoothTime);
```

### Lerp vs MoveTowards
| Method | Behavior | Reaches Target | Use For |
|--------|----------|---------------|---------|
| Lerp | Exponential approach (slows down) | Asymptotic (never exactly) | Camera, smooth follow |
| MoveTowards | Linear approach (constant speed) | Yes (exact) | Moving to a point |
| SmoothDamp | Spring-like (natural deceleration) | Yes (smooth stop) | Camera, UI elements |

---

## Trigonometry for Games

### Angle Between Two Points
```javascript
// Returns angle in radians (-PI to PI)
const angle = Math.atan2(targetY - originY, targetX - originX);

// Convert to degrees
const degrees = angle * (180 / Math.PI);
```

```csharp
// Unity
float angle = Mathf.Atan2(dir.y, dir.x) * Mathf.Rad2Deg;
transform.rotation = Quaternion.Euler(0, 0, angle);
```

### Circular Motion (orbit, rotation)
```javascript
// Move in a circle of radius r, at angular speed omega
x = centerX + r * Math.cos(angle);
y = centerY + r * Math.sin(angle);
angle += omega * deltaTime;
```

### Wave Patterns (sine wave for bobbing, pulsing)
```javascript
// Bobbing up and down
y = baseY + amplitude * Math.sin(time * frequency);

// Pulsing scale
scale = 1.0 + 0.1 * Math.sin(time * 5); // pulse between 0.9 and 1.1
```

### Direction from Angle
```javascript
// Convert angle (radians) to direction vector
dirX = Math.cos(angle);
dirY = Math.sin(angle);
```

### Common Angles
| Degrees | Radians | Direction (screen coords, Y-down) |
|---------|---------|----------------------------------|
| 0 | 0 | Right |
| 90 (π/2) | 1.5708 | Down |
| 180 (π) | 3.1416 | Left |
| 270 (3π/2) | 4.7124 | Up |

Unity (Y-up): 90° = Up, 270° = Down.

---

## Bezier Curves

Smooth curves defined by control points. Great for paths, trajectories, UI animations.

### Quadratic Bezier (3 points: start, control, end)
```javascript
function quadBezier(p0, p1, p2, t) {
    const u = 1 - t;
    return {
        x: u * u * p0.x + 2 * u * t * p1.x + t * t * p2.x,
        y: u * u * p0.y + 2 * u * t * p1.y + t * t * p2.y
    };
}
```

### Cubic Bezier (4 points: start, control1, control2, end)
```javascript
function cubicBezier(p0, p1, p2, p3, t) {
    const u = 1 - t;
    return {
        x: u*u*u*p0.x + 3*u*u*t*p1.x + 3*u*t*t*p2.x + t*t*t*p3.x,
        y: u*u*u*p0.y + 3*u*u*t*p1.y + 3*u*t*t*p2.y + t*t*t*p3.y
    };
}
// Sample points along curve: for (let t = 0; t <= 1; t += 0.01) { cubicBezier(..., t); }
```

Use for: projectile arcs, camera paths, enemy movement patterns, UI transitions.

---

## Easing Functions

Transform a linear 0-to-1 value into a curved one for natural motion.

```javascript
const Easing = {
    linear:      t => t,
    easeInQuad:  t => t * t,
    easeOutQuad: t => t * (2 - t),
    easeInOutQuad: t => t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t,

    easeInCubic:  t => t * t * t,
    easeOutCubic: t => 1 - Math.pow(1 - t, 3),
    easeInOutCubic: t => t < 0.5 ? 4 * t * t * t : 1 - Math.pow(-2 * t + 2, 3) / 2,

    easeInElastic: t => t === 0 ? 0 : t === 1 ? 1 :
        -Math.pow(2, 10 * t - 10) * Math.sin((t * 10 - 10.75) * (2 * Math.PI / 3)),
    easeOutElastic: t => t === 0 ? 0 : t === 1 ? 1 :
        Math.pow(2, -10 * t) * Math.sin((t * 10 - 0.75) * (2 * Math.PI / 3)) + 1,

    easeOutBounce: t => {
        if (t < 1/2.75) return 7.5625 * t * t;
        if (t < 2/2.75) return 7.5625 * (t -= 1.5/2.75) * t + 0.75;
        if (t < 2.5/2.75) return 7.5625 * (t -= 2.25/2.75) * t + 0.9375;
        return 7.5625 * (t -= 2.625/2.75) * t + 0.984375;
    },

    easeOutBack: t => { const c = 1.70158; return 1 + (c + 1) * Math.pow(t - 1, 3) + c * Math.pow(t - 1, 2); }
};

// Usage: apply to any 0-1 value
const easedT = Easing.easeOutQuad(t);
const value = lerp(start, end, easedT);
```

### When to Use Each
| Easing | Feel | Use For |
|--------|------|---------|
| easeOutQuad | Gentle deceleration | Camera movement, general UI |
| easeOutCubic | Quick start, smooth stop | Menu slides, popups |
| easeInOutQuad | Smooth both ends | Transitions, camera pans |
| easeOutElastic | Springy overshoot | Bouncy UI elements, pick-ups |
| easeOutBounce | Bouncing landing | Dropping objects, score popups |
| easeOutBack | Slight overshoot | Button presses, scale effects |

Phaser tweens accept ease names: `'Quad.easeOut'`, `'Cubic.easeIn'`, `'Bounce'`, `'Elastic'`, `'Back'`.

---

## Randomness

### Weighted Random Selection
```javascript
function weightedRandom(items) {
    // items = [{value: 'common', weight: 70}, {value: 'rare', weight: 25}, {value: 'epic', weight: 5}]
    const totalWeight = items.reduce((sum, item) => sum + item.weight, 0);
    let random = Math.random() * totalWeight;
    for (const item of items) {
        random -= item.weight;
        if (random <= 0) return item.value;
    }
    return items[items.length - 1].value;
}
```

### Shuffle (Fisher-Yates)
```javascript
function shuffle(array) {
    for (let i = array.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [array[i], array[j]] = [array[j], array[i]];
    }
    return array;
}
```

### Seeded Random (Reproducible)
```javascript
// Simple mulberry32 PRNG
function mulberry32(seed) {
    return function() {
        seed |= 0; seed = seed + 0x6D2B79F5 | 0;
        let t = Math.imul(seed ^ seed >>> 15, 1 | seed);
        t = t + Math.imul(t ^ t >>> 7, 61 | t) ^ t;
        return ((t ^ t >>> 14) >>> 0) / 4294967296;
    };
}
const rng = mulberry32(12345); // same seed = same sequence
rng(); // always returns the same first value for seed 12345
```

---

## Perlin / Simplex Noise

Smooth, continuous random values for terrain generation, cloud textures, organic movement.

Properties:
- Same input always returns same output (deterministic)
- Nearby inputs return similar values (smooth)
- Range: typically -1 to 1 or 0 to 1

### Use Cases
- Terrain height maps
- Cloud/fog generation
- Organic enemy movement (add noise to patrol path)
- Camera shake (noise-based is smoother than random)
- Procedural level generation

### Libraries
- JavaScript: `simplex-noise` npm package, or Phaser's `Phaser.Math.Noise`
- Unity: `Mathf.PerlinNoise(x, y)` (built-in; returns 0-1)

```csharp
// Unity terrain example
float height = Mathf.PerlinNoise(x * scale, y * scale); // 0 to 1
// Octaves for detail:
float h = 0;
float amplitude = 1, frequency = 1, maxValue = 0;
for (int i = 0; i < octaves; i++) {
    h += Mathf.PerlinNoise(x * frequency * scale, y * frequency * scale) * amplitude;
    maxValue += amplitude;
    amplitude *= persistence; // 0.5 typical
    frequency *= lacunarity;  // 2.0 typical
}
h /= maxValue;
```

---

## Distance Comparisons

### Euclidean Distance (accurate)
```
dist = sqrt((x2-x1)^2 + (y2-y1)^2)
```
Use for: radius checks, accurate distance.

### Manhattan Distance (faster, grid-aligned)
```
dist = abs(x2-x1) + abs(y2-y1)
```
Use for: grid pathfinding heuristic, rough proximity.

### Squared Distance (fastest, no sqrt)
```
distSq = (x2-x1)^2 + (y2-y1)^2
```
Use for: comparing distances (is A closer than B?), range checks.
```javascript
// Instead of: if (distance(a, b) < range)
// Use:        if (distSq(a, b) < range * range)
```

---

## Clamping and Wrapping

```javascript
// Clamp: keep value within bounds
function clamp(value, min, max) {
    return Math.max(min, Math.min(max, value));
}

// Wrap: loop value around (screen wrap, angle wrap)
function wrap(value, min, max) {
    const range = max - min;
    return ((value - min) % range + range) % range + min;
}
// wrap(370, 0, 360) → 10
// wrap(-10, 0, 360) → 350
```
