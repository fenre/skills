# Pathfinding Reference for 2D Adventure Games

## Polygon-Based Walkable Areas

Adventure games define walkable regions as polygons rather than grids. This gives smooth, natural movement.

### Point-in-Polygon Test (Ray Casting)

```typescript
function isInPolygon(point: Vec2, polygon: Vec2[]): boolean {
  let inside = false;
  for (let i = 0, j = polygon.length - 1; i < polygon.length; j = i++) {
    const xi = polygon[i].x, yi = polygon[i].y;
    const xj = polygon[j].x, yj = polygon[j].y;
    if (((yi > point.y) !== (yj > point.y)) &&
        (point.x < (xj - xi) * (point.y - yi) / (yj - yi) + xi)) {
      inside = !inside;
    }
  }
  return inside;
}
```

### Closest Point on Polygon Edge

When player clicks outside walkable area, snap to nearest walkable point:

```typescript
function closestPointOnPolygon(point: Vec2, polygon: Vec2[]): Vec2 {
  let closest: Vec2 = polygon[0];
  let minDist = Infinity;
  for (let i = 0; i < polygon.length; i++) {
    const a = polygon[i];
    const b = polygon[(i + 1) % polygon.length];
    const cp = closestPointOnSegment(point, a, b);
    const d = distance(point, cp);
    if (d < minDist) { minDist = d; closest = cp; }
  }
  return closest;
}
```

### Line-of-Sight Check

Test if a straight line between two points stays within the polygon:

```typescript
function hasLineOfSight(a: Vec2, b: Vec2, polygon: Vec2[]): boolean {
  for (let i = 0; i < polygon.length; i++) {
    const c = polygon[i];
    const d = polygon[(i + 1) % polygon.length];
    if (segmentsIntersect(a, b, c, d)) return false;
  }
  // Also check midpoint is inside polygon
  const mid = { x: (a.x + b.x) / 2, y: (a.y + b.y) / 2 };
  return isInPolygon(mid, polygon);
}
```

### Visibility Graph Construction

Build a graph of all concave vertices plus start/end points, connected by line-of-sight edges:

```typescript
function getConcaveVertices(polygon: Vec2[]): Vec2[] {
  const concave: Vec2[] = [];
  const n = polygon.length;
  for (let i = 0; i < n; i++) {
    const prev = polygon[(i - 1 + n) % n];
    const curr = polygon[i];
    const next = polygon[(i + 1) % n];
    const cross = (curr.x - prev.x) * (next.y - curr.y) -
                  (curr.y - prev.y) * (next.x - curr.x);
    if (cross < 0) concave.push(curr);  // clockwise winding = concave
  }
  return concave;
}

function buildVisibilityGraph(nodes: Vec2[], polygon: Vec2[]): Graph {
  const graph = new Graph();
  for (let i = 0; i < nodes.length; i++) {
    for (let j = i + 1; j < nodes.length; j++) {
      if (hasLineOfSight(nodes[i], nodes[j], polygon)) {
        const d = distance(nodes[i], nodes[j]);
        graph.addEdge(i, j, d);
      }
    }
  }
  return graph;
}
```

### A* on Visibility Graph

```typescript
function aStar(graph: Graph, startIdx: number, endIdx: number): Vec2[] {
  const open = new PriorityQueue<number>();
  const gScore: Record<number, number> = {};
  const fScore: Record<number, number> = {};
  const cameFrom: Record<number, number> = {};

  gScore[startIdx] = 0;
  fScore[startIdx] = heuristic(startIdx, endIdx);
  open.enqueue(startIdx, fScore[startIdx]);

  while (!open.isEmpty()) {
    const current = open.dequeue();
    if (current === endIdx) return reconstructPath(cameFrom, current, graph);

    for (const { neighbor, weight } of graph.neighbors(current)) {
      const tentative = (gScore[current] ?? Infinity) + weight;
      if (tentative < (gScore[neighbor] ?? Infinity)) {
        cameFrom[neighbor] = current;
        gScore[neighbor] = tentative;
        fScore[neighbor] = tentative + heuristic(neighbor, endIdx);
        open.enqueue(neighbor, fScore[neighbor]);
      }
    }
  }
  return [];  // no path found
}
```

## Walkable Area with Holes

For rooms with furniture or obstacles that block walking:

```typescript
interface WalkableArea {
  outer: Vec2[];       // outer boundary (clockwise)
  holes: Vec2[][];     // inner holes (counter-clockwise)
}
```

When building the visibility graph, include concave vertices from both the outer polygon AND all hole polygons. Line-of-sight checks must test against all polygon edges (outer + holes).

## Character Walk Speed and Depth Scaling

Characters higher on screen (smaller Y) are further away and should walk slower and render smaller:

```typescript
const BASE_SPEED = 200;        // pixels/sec at bottom of room
const MIN_SCALE = 0.5;         // scale at top of walkable area
const MAX_SCALE = 1.0;         // scale at bottom

function getCharScale(y: number, walkableMinY: number, walkableMaxY: number): number {
  const t = (y - walkableMinY) / (walkableMaxY - walkableMinY);
  return MIN_SCALE + t * (MAX_SCALE - MIN_SCALE);
}

function getCharSpeed(y: number, walkableMinY: number, walkableMaxY: number): number {
  return BASE_SPEED * getCharScale(y, walkableMinY, walkableMaxY);
}
```

## Phaser 3 Polygon Interaction

```typescript
// Define walkable area
const walkPoly = new Phaser.Geom.Polygon([x1,y1, x2,y2, ...]);

// Check if click is walkable
this.input.on('pointerdown', (pointer: Phaser.Input.Pointer) => {
  const worldPoint = this.cameras.main.getWorldPoint(pointer.x, pointer.y);
  if (Phaser.Geom.Polygon.Contains(walkPoly, worldPoint.x, worldPoint.y)) {
    walkPlayerTo(worldPoint.x, worldPoint.y);
  } else {
    // Snap to nearest edge point
    const nearest = closestPointOnPolygon(worldPoint, walkPolyVertices);
    walkPlayerTo(nearest.x, nearest.y);
  }
});
```
