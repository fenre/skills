---
name: adventure-game-2d
description: >
  2D point-and-click adventure game development reference in the style of Monkey Island,
  Day of the Tentacle, Thimbleweed Park, and other LucasArts/Sierra classics.
  Covers: room/scene architecture, verb-based interaction systems, inventory management,
  dialogue trees with branching conversations, puzzle design (combinatorial, lateral thinking,
  inventory, environmental), character pathfinding over walkable area polygons, pixel art
  scene composition with parallax layers, game state and flag tracking, save/load systems,
  cutscene scripting, humor and comedy writing for games, and SCUMM-inspired UI layouts.
  Primarily targets Phaser 3 (TypeScript) but principles are engine-agnostic.
  Use when building a point-and-click adventure, creating room-based exploration,
  implementing dialogue systems, designing adventure game puzzles, or asking about
  Monkey Island, SCUMM, or classic adventure game design.
---

# 2D Point-and-Click Adventure Game Development

## Architecture Overview

A point-and-click adventure is a **data-driven, story-first game** where gameplay emerges from exploring rooms, talking to characters, collecting items, and solving puzzles through logical (and sometimes lateral) reasoning.

### Core Systems

```
┌─────────────────────────────────────────────┐
│                Game Manager                 │
│  (current room, global flags, inventory)    │
├──────────┬──────────┬──────────┬────────────┤
│  Room    │ Dialogue │ Inventory│  Puzzle    │
│  Engine  │ System   │ System   │  Logic     │
├──────────┼──────────┼──────────┼────────────┤
│ Pathfind │ Cutscene │  Verb/   │   Save/    │
│ System   │ Engine   │ Action UI│   Load     │
└──────────┴──────────┴──────────┴────────────┘
```

| System | Responsibility |
|--------|---------------|
| **Room Engine** | Load/render rooms, manage hotspots, objects, exits, walkable areas |
| **Dialogue System** | Branching conversation trees, NPC speech, typewriter text |
| **Inventory** | Item collection, combination, use-on-object |
| **Puzzle Logic** | Flag-based state checks, item requirements, multi-step solutions |
| **Pathfinding** | Character navigation over walkable area polygons |
| **Cutscene Engine** | Scripted sequences — walk, say, wait, animate, camera |
| **Verb/Action UI** | Player interaction model (verb bar, context menu, or simplified) |
| **Save/Load** | Serialize game state, inventory, flags, room states, positions |

---

## Room / Scene System

Each room is a self-contained scene with layered visuals and interactive elements.

### Room Data Structure

```typescript
interface Room {
  id: string;
  name: string;
  background: string;           // main background image
  parallaxLayers?: ParallaxLayer[];
  walkableArea: number[][];     // polygon vertices [[x,y], ...]
  hotspots: Hotspot[];          // background regions (non-movable)
  objects: RoomObject[];        // interactive items (can be taken/moved)
  characters: NPC[];
  exits: Exit[];
  music?: string;
  ambience?: string;
  onEnter?: string;             // script ID to run on room entry
}

interface Hotspot {
  id: string;
  name: string;                 // "the window", "a painting"
  polygon: number[][];          // clickable region
  look: string;                 // response to "Look at"
  use?: string;                 // response to "Use"
  interactions: Record<string, string>;  // verb → script/response
}

interface RoomObject {
  id: string;
  name: string;
  sprite: string;
  x: number; y: number;
  takeable: boolean;
  look: string;
  interactions: Record<string, string>;
  state: string;                // "open", "closed", "broken", etc.
  requiredFlag?: string;        // only visible when flag is set
}

interface Exit {
  id: string;
  polygon: number[][];          // walk-to region
  targetRoom: string;
  targetX: number;
  targetY: number;
  direction: 'left' | 'right' | 'up' | 'down';
}
```

### Parallax Background Layers

Layer rooms with 3-5 depth planes for visual richness:

| Layer | Scroll Rate | Content |
|-------|------------|---------|
| Sky / far BG | 0.0–0.1 | Sky gradient, distant mountains, stars |
| Mid BG | 0.2–0.4 | Trees, buildings, horizon elements |
| Main BG | 1.0 | Primary room art (character walks here) |
| Near FG | 1.3–1.5 | Foreground objects (tables, fences) |
| Overlay | 1.5–2.0 | Close foliage, frame elements |

Use **atmospheric perspective**: far layers are desaturated and low-contrast; near layers are vivid.

---

## Interaction System

### SCUMM-Style Verb Bar (Classic)

The original Monkey Island used 9–12 verbs at screen bottom:

```
 ┌──────────────────────────────────────────┐
 │            Game Viewport (room)          │
 │                                          │
 ├──────────────────────────────────────────┤
 │ Open │ Close │ Push │ Pull │ Give │ ...  │  ← Verb bar
 ├──────────────────────────────────────────┤
 │  [key] [rubber chicken] [map] [coin]    │  ← Inventory
 └──────────────────────────────────────────┘
```

Standard verbs: **Give, Open, Close, Pick up, Look at, Talk to, Use, Push, Pull**.

### Modern Simplified (Recommended for New Games)

Most modern adventures reduce to 2–3 actions:
- **Left-click**: Context action (walk / use / take)
- **Right-click**: Look at / examine
- **Drag item**: Use inventory item on world object

Or a **verb coin** — radial menu appearing on right-click with 3–4 icons (eye, hand, mouth, item).

### Interaction Resolution

```typescript
function resolveInteraction(verb: string, target: Hotspot | RoomObject, item?: InventoryItem): void {
  const key = item ? `${verb}_${item.id}` : verb;

  if (target.interactions[key]) {
    executeScript(target.interactions[key]);
  } else if (item) {
    say(player, `I can't use the ${item.name} with that.`);
  } else if (verb === 'look') {
    say(player, target.look);
  } else {
    say(player, defaultResponses[verb]);  // "I can't do that." etc.
  }
}
```

---

## Inventory System

### Data Model

```typescript
interface InventoryItem {
  id: string;
  name: string;
  description: string;         // "Look at" text
  sprite: string;              // inventory icon
  combinable: string[];        // IDs of items this can combine with
  useOnSelf?: string;          // script when "Use" alone
}

interface InventoryState {
  items: InventoryItem[];
  maxSlots: number;            // typically unlimited in adventures
}
```

### Item Combination

```typescript
const COMBINATIONS: Record<string, { result: string; script?: string }> = {
  'rubber_chicken+pulley': { result: 'rubber_chicken_pulley', script: 'combine_chicken_pulley' },
  'key+door': { result: null, script: 'unlock_door' },
};

function combineItems(a: string, b: string): CombineResult | null {
  return COMBINATIONS[`${a}+${b}`] || COMBINATIONS[`${b}+${a}`] || null;
}
```

---

## Dialogue System

### Dialogue Tree Structure

```typescript
interface DialogueNode {
  id: string;
  speaker: string;
  text: string;
  choices?: DialogueChoice[];
  next?: string;               // auto-advance to this node
  onEnter?: string;            // script to run when node is reached
  condition?: string;          // flag expression — skip if false
}

interface DialogueChoice {
  text: string;                // what the player sees
  targetNode: string;
  condition?: string;          // only show if flag is set
  onSelect?: string;           // script to run when chosen
  once?: boolean;              // hide after selecting once
}
```

### Dialogue Patterns

**Hub-and-Spokes** (Monkey Island style): Player returns to a central hub of questions after each NPC response. Player can explore all topics.

```
     ┌─ "Tell me about the treasure" → response → back to hub
HUB ─┼─ "Who are you?" → response → back to hub  
     ├─ "Nice hat." → response → back to hub
     └─ "Goodbye" → exit dialogue
```

**One-Shot Choices** (Thimbleweed Park style): Player picks ONE response; other options vanish. Creates replay value — players miss jokes on first playthrough.

**Conditional Choices**: Options appear/disappear based on game flags:

```typescript
// Only show "Ask about map" if player has found the torn page
{ text: "What about this map fragment?", condition: "has_torn_page", targetNode: "map_info" }
```

### Typewriter Text Display

```typescript
async function typewriterText(text: string, speed = 30): Promise<void> {
  for (let i = 0; i <= text.length; i++) {
    textObject.setText(text.substring(0, i));
    await delay(speed);
    if (skipPressed) { textObject.setText(text); return; }
  }
  await waitForClick();
}
```

---

## Puzzle Design

### Golden Rules

1. **Problem before solution** — Player encounters the locked door BEFORE finding the key.
2. **Fair clues** — Every puzzle solution should be hinted at through observation or dialogue.
3. **Multiple paths when possible** — Parallel puzzle chains prevent bottlenecks.
4. **No dead ends** — Player should never reach an unwinnable state (LucasArts philosophy).
5. **Logical within the game world** — Solutions should make sense in context, even if whimsical.

### Puzzle Types

| Type | Description | Example |
|------|-------------|---------|
| **Inventory** | Use/combine collected items | Combine rubber chicken + pulley |
| **Environmental** | Interact with room elements | Pull lever to open gate |
| **Dialogue** | Extract information via conversation | Learn the password from the barkeeper |
| **Observation** | Notice visual/audio clues | Read the note on the desk |
| **Combination** | Multi-step using several types | Learn recipe (dialogue) → gather ingredients (inventory) → cook (environmental) |
| **Lateral thinking** | Creative/unexpected logic | Use gopher repellent on the rat |
| **Sequence** | Actions in specific order | Open safe with overheard combination |
| **NPC-mediated** | Give item to / convince NPC | Give flowers to guard to distract them |

### Puzzle Dependency Chart

Design puzzles as a directed acyclic graph showing what unlocks what:

```
[Find map] ──→ [Decode map] ──→ [Navigate jungle]
     │                                   │
     └──→ [Get compass from shop] ──────→┘
              │
[Earn coins] ─┘
```

Parallel branches (multiple puzzles solvable at once) improve pacing. Strictly linear chains feel bottlenecked.

### Flag System

```typescript
const flags: Record<string, boolean | number | string> = {};

function setFlag(key: string, value: any): void { flags[key] = value; }
function getFlag(key: string): any { return flags[key]; }
function checkFlag(expr: string): boolean {
  // "has_key && !door_open" → evaluate against flags
  return evaluateExpression(expr, flags);
}
```

---

## Character Pathfinding

### Walkable Area Polygon

Define walkable regions as polygons. Characters navigate within these bounds.

```typescript
interface WalkableArea {
  polygon: number[][];       // [[x1,y1], [x2,y2], ...]
  holes?: number[][][];      // cutout regions (furniture, pits)
  active: boolean;
}
```

### Navigation Algorithm

1. **Point-in-polygon test** — verify destination is walkable
2. **If direct line-of-sight** — walk straight
3. **Otherwise, build visibility graph** from concave polygon vertices
4. **Run A*** on the visibility graph
5. **Smooth the path** with funnel algorithm

```typescript
function findPath(start: Vec2, end: Vec2, walkable: WalkableArea): Vec2[] {
  if (!isInPolygon(end, walkable.polygon)) {
    end = closestPointOnPolygon(end, walkable.polygon);
  }
  if (hasLineOfSight(start, end, walkable)) return [start, end];

  const nodes = getConcaveVertices(walkable.polygon);
  nodes.push(start, end);
  const graph = buildVisibilityGraph(nodes, walkable);
  return aStar(graph, start, end);
}
```

### Character Walking

Iterate waypoints from `findPath()`, set directional animation, tween to each point, then return to idle. Scale characters by Y-position for depth (higher on screen = smaller/further).

Required animation sets: `idle/walk/talk` × `left/right/front/back`, plus `pick_up`, `use`, `give`.

---

## Cutscene / Scripting Engine

Adventure games need a lightweight scripting system for cutscenes and interactions.

### Script Actions

```typescript
type ScriptAction =
  | { type: 'say'; character: string; text: string }
  | { type: 'walk'; character: string; x: number; y: number }
  | { type: 'animate'; character: string; anim: string }
  | { type: 'wait'; ms: number }
  | { type: 'setFlag'; flag: string; value: any }
  | { type: 'addItem'; item: string }
  | { type: 'removeItem'; item: string }
  | { type: 'removeObject'; object: string }
  | { type: 'changeRoom'; room: string; x: number; y: number }
  | { type: 'playSound'; sound: string }
  | { type: 'fadeOut' } | { type: 'fadeIn' }
  | { type: 'camera'; target: string; speed: number }
  | { type: 'choice'; dialogue: string }
  | { type: 'conditional'; flag: string; then: ScriptAction[]; else?: ScriptAction[] };
```

### Script Runner

Iterate over the action array sequentially with `async/await`. Disable player input at start, re-enable on completion. Switch on `action.type` and delegate to the appropriate system (dialogue, pathfinding, inventory, flags). Support `conditional` by recursively calling `runScript` on the `then`/`else` branches.

---

## Save / Load System

### What to Persist

```typescript
interface SaveData {
  version: number;
  timestamp: number;
  currentRoom: string;
  playerPosition: { x: number; y: number };
  inventory: string[];          // item IDs
  flags: Record<string, any>;   // all game state flags
  roomStates: Record<string, {
    removedObjects: string[];
    objectStates: Record<string, string>;
    visitCount: number;
  }>;
  dialogueState: Record<string, string[]>; // exhausted dialogue choices
  npcPositions: Record<string, { room: string; x: number; y: number }>;
}
```

Serialize to JSON. For web: `localStorage`. Keep a version field to handle format migrations.

---

## Comedy & Writing (Monkey Island Style)

### Key Techniques from Ron Gilbert

1. **Long-fuse jokes** — Setup and punchline can be separated by 30+ minutes of gameplay. ("I can hold my breath for 10 minutes" pays off when Guybrush is underwater.)
2. **Absurd logic** — Solutions should be funny AND logical within the game's world.
3. **Fourth-wall breaks** — Characters aware they're in a game ("I'm selling these fine leather jackets").
4. **Verb humor** — Funny responses to unusual verb+object combinations.
5. **Character voice** — Every character has a distinct speaking style and worldview.

### Writing Default Responses

Every unhandled verb+object combo needs a witty fallback. Keep an array per verb and randomize. Vary tone per character — a pirate responds differently than a librarian. Make failed interactions entertaining, not frustrating.

---

## Phaser 3 Implementation Notes

### Scene Structure

```
BootScene → MenuScene → GameScene (persistent)
                           ├── RoomScene (swapped per room)
                           ├── UIScene (overlay — verbs, inventory)
                           └── DialogueScene (overlay — conversation)
```

Use Phaser's scene stacking: `GameScene` stays active while `RoomScene` swaps. UI scenes run in parallel with `scene.launch()`.

### Key Phaser Features for Adventures

| Need | Phaser API |
|------|-----------|
| Click detection on shapes | `this.add.zone()` or `polygon.contains()` with `input.on('pointerdown')` |
| Sprite depth sorting | `sprite.setDepth()` or `sprite.y` as depth for auto-sorting |
| Camera follow/pan | `this.cameras.main.pan(x, y, duration)` |
| Typewriter text | `this.time.addEvent({ delay, callback, repeat })` |
| Tweened walking | `this.tweens.add({ targets, x, y, duration })` |
| Parallax | Multiple `tileSprite` layers with different `scrollFactorX` |
| Masking walkable area | `Phaser.Geom.Polygon` + `Phaser.Geom.Polygon.Contains()` |
| Scene overlay | `this.scene.launch('UIScene')` runs in parallel |

### Y-Depth Sorting

Characters further from camera (higher Y = further up screen) should render behind closer characters:

```typescript
this.children.sort('y', Phaser.GameObjects.GROUP_SORT_ASCENDING);
// or per-frame:
character.setDepth(character.y);
```

---

## Additional Resources

- For puzzle dependency analysis methodology, see the Monkey Island 2 puzzle dependency article on gamedeveloper.com
- For pathfinding implementation details, see [pathfinding-reference.md](pathfinding-reference.md)
- For complete room data examples, see [room-examples.md](room-examples.md)
