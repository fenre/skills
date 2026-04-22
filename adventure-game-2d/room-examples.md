# Room Data Examples

## Complete Room Definition

```typescript
const TAVERN: Room = {
  id: 'tavern',
  name: 'The Rusty Anchor Tavern',
  background: 'tavern_bg',
  parallaxLayers: [
    { image: 'tavern_window_view', scrollFactor: 0.2, y: 0 },
    { image: 'tavern_shelves', scrollFactor: 0.5, y: 100 },
  ],
  walkableArea: [
    [80, 340], [720, 340], [720, 480], [600, 480],
    [580, 420], [200, 420], [180, 480], [80, 480],
  ],
  hotspots: [
    {
      id: 'fireplace',
      name: 'the fireplace',
      polygon: [[50, 200], [180, 200], [180, 340], [50, 340]],
      look: "A roaring fire. Very cozy.",
      interactions: {
        use: "I'd rather not burn my hands.",
        'use_marshmallow': 'script:toast_marshmallow',
      },
    },
    {
      id: 'notice_board',
      name: 'the notice board',
      polygon: [[300, 180], [420, 180], [420, 300], [300, 300]],
      look: "Several notices are pinned up. One catches my eye...",
      interactions: {
        look: 'script:read_notices',
        use: "I don't have anything to pin up.",
      },
    },
  ],
  objects: [
    {
      id: 'mug',
      name: 'a half-empty mug',
      sprite: 'mug_sprite',
      x: 350, y: 360,
      takeable: true,
      look: "Somebody left their grog. Waste not, want not.",
      state: 'full',
      interactions: {
        pick_up: 'script:take_mug',
        use: "I take a swig. Not bad, if you like battery acid.",
      },
    },
    {
      id: 'locked_chest',
      name: 'a locked chest',
      sprite: 'chest_locked',
      x: 650, y: 390,
      takeable: false,
      look: "A heavy chest with an ornate lock.",
      state: 'locked',
      interactions: {
        use: "It's locked tight.",
        'use_rusty_key': 'script:unlock_chest',
        push: "It's too heavy to push.",
        pull: "It won't budge.",
      },
    },
  ],
  characters: [
    {
      id: 'barkeeper',
      name: 'the barkeeper',
      sprite: 'barkeeper_sheet',
      x: 500, y: 350,
      facing: 'left',
      dialogue: 'barkeeper_dialogue_root',
      look: "A burly fellow with an impressive collection of scars.",
      interactions: {
        'give_coin': 'script:buy_drink',
        'give_flowers': "He raises an eyebrow. 'I'm flattered, but no.'",
        'use_sword': "That seems like a terrible idea. He's twice my size.",
      },
    },
  ],
  exits: [
    {
      id: 'exit_street',
      polygon: [[0, 340], [40, 340], [40, 480], [0, 480]],
      targetRoom: 'harbor_street',
      targetX: 700, targetY: 400,
      direction: 'left',
    },
    {
      id: 'exit_upstairs',
      polygon: [[620, 200], [700, 200], [700, 280], [620, 280]],
      targetRoom: 'tavern_upstairs',
      targetX: 400, targetY: 400,
      direction: 'up',
    },
  ],
  music: 'tavern_theme',
  ambience: 'crowd_murmur',
};
```

## Dialogue Tree Example

```typescript
const BARKEEPER_DIALOGUE: DialogueTree = {
  id: 'barkeeper_dialogue_root',
  startNode: 'greeting',
  nodes: {
    greeting: {
      id: 'greeting',
      speaker: 'barkeeper',
      text: "What'll it be, stranger?",
      choices: [
        { text: "I'm looking for the governor.", targetNode: 'governor_info' },
        { text: "Tell me about this island.", targetNode: 'island_info' },
        {
          text: "I hear you know something about the treasure.",
          targetNode: 'treasure_info',
          condition: 'heard_about_treasure',
        },
        {
          text: "Here's that package you were expecting.",
          targetNode: 'deliver_package',
          condition: 'has_package',
          once: true,
        },
        { text: "Never mind.", targetNode: 'exit' },
      ],
    },
    governor_info: {
      id: 'governor_info',
      speaker: 'barkeeper',
      text: "The governor? Ha! Good luck getting past her guard dogs. And I mean actual dogs.",
      next: 'greeting',
      onEnter: 'script:set_flag:knows_about_dogs',
    },
    island_info: {
      id: 'island_info',
      speaker: 'barkeeper',
      text: "This here's Cableport. Used to be a fishing village. Now it's mostly pirates and people who sell things to pirates.",
      next: 'greeting',
    },
    treasure_info: {
      id: 'treasure_info',
      speaker: 'barkeeper',
      text: "Keep your voice down! ...Meet me behind the tavern at midnight. And bring something to trade.",
      next: 'greeting',
      onEnter: 'script:set_flag:midnight_meeting',
    },
    deliver_package: {
      id: 'deliver_package',
      speaker: 'barkeeper',
      text: "Finally! You have no idea how long I've been waiting for this. Here, take this key as thanks.",
      onEnter: 'script:deliver_package_reward',
      next: 'exit',
    },
    exit: {
      id: 'exit',
      speaker: 'barkeeper',
      text: "Come back if you need anything.",
    },
  },
};
```

## Puzzle Chain Example

```typescript
// Puzzle: Get into the governor's mansion

const MANSION_PUZZLE_FLAGS = {
  // Step 1: Learn about the guard dogs (dialogue)
  knows_about_dogs: false,

  // Step 2: Get meat from the butcher (inventory)
  has_meat: false,

  // Step 3: Get sleeping powder from the doctor (dialogue + inventory)
  has_sleeping_powder: false,

  // Step 4: Combine meat + sleeping powder (inventory combination)
  has_drugged_meat: false,

  // Step 5: Use drugged meat on dogs (use item on hotspot)
  dogs_asleep: false,

  // Step 6: Pick the lock OR find the back entrance
  has_lockpick: false,         // parallel path A
  knows_back_entrance: false,  // parallel path B

  // Goal
  inside_mansion: false,
};

// Dependency graph:
// knows_about_dogs → has_meat ──────────────┐
//                    has_sleeping_powder ────┤
//                                           ↓
//                                    has_drugged_meat
//                                           ↓
//                                      dogs_asleep
//                                           ↓
//                    has_lockpick ──→ inside_mansion  (path A)
//            knows_back_entrance ──→ inside_mansion  (path B)
```

## Cutscene Script Example

```typescript
const UNLOCK_CHEST_SCRIPT: ScriptAction[] = [
  { type: 'walk', character: 'player', x: 640, y: 390 },
  { type: 'animate', character: 'player', anim: 'use' },
  { type: 'say', character: 'player', text: "Let's see if this key fits..." },
  { type: 'playSound', sound: 'key_turn' },
  { type: 'wait', ms: 500 },
  { type: 'playSound', sound: 'chest_open' },
  { type: 'say', character: 'player', text: "Yes! It worked!" },
  { type: 'setFlag', flag: 'chest_opened', value: true },
  { type: 'removeItem', item: 'rusty_key' },
  { type: 'addItem', item: 'treasure_map' },
  { type: 'say', character: 'player', text: "A treasure map! This changes everything." },
  {
    type: 'conditional',
    flag: 'barkeeper_is_watching',
    then: [
      { type: 'say', character: 'barkeeper', text: "Hey! That's MY chest!" },
      { type: 'walk', character: 'barkeeper', x: 620, y: 390 },
      { type: 'say', character: 'player', text: "Uh oh." },
    ],
  },
];
```

## Interaction Map Pattern

For complex rooms, define a lookup table mapping `verb_item → script`:

```typescript
const TAVERN_INTERACTIONS: Record<string, string> = {
  // hotspot interactions
  'look_fireplace':          "A roaring fire. Very cozy.",
  'use_fireplace':           "I'd rather keep my eyebrows.",
  'use_marshmallow_fireplace': 'script:toast_marshmallow',

  // object interactions
  'pick_up_mug':             'script:take_mug',
  'look_mug':                "Somebody's unfinished grog.",
  'use_mug':                 "I take a swig. Blech.",
  'use_mug_fireplace':       "I'm not making hot grog. This isn't that kind of game.",

  // NPC interactions
  'talk_barkeeper':          'dialogue:barkeeper_dialogue_root',
  'give_coin_barkeeper':     'script:buy_drink',
  'use_sword_barkeeper':     "I like my head attached to my body, thanks.",

  // exit interactions
  'use_exit_street':         'script:leave_tavern',
};
```
