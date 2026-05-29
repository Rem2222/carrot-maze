# Snake Game Mechanics: HTML5 Canvas Research

## 1. Game Loop Architecture (requestAnimationFrame)

### Overview
`requestAnimationFrame` (rAF) is the standard for browser game loops. It fires ~60 times/sec (matching display refresh), automatically pauses when the tab is hidden, and saves battery. The callback receives a high-resolution `DOMHighResTimeStamp`.

### Fixed Timestep

The game logic runs in discrete "ticks" at a constant interval, decoupled from rendering.

```js
// Fixed-timestep game loop (deterministic, recommended for snake)
const TICK_RATE = 10;        // game updates per second
const TICK_DURATION = 1000 / TICK_RATE; // ms per tick (~100ms for 10Hz)
let lastTickTime = 0;
let accumulator = 0;

function gameLoop(timestamp) {
  requestAnimationFrame(gameLoop);

  // Accumulate elapsed time
  const delta = timestamp - lastTickTime;
  lastTickTime = timestamp;
  accumulator += delta;

  // Consume accumulated time in fixed-size chunks
  while (accumulator >= TICK_DURATION) {
    update();          // Fixed-step game logic
    accumulator -= TICK_DURATION;
  }

  render();            // Render every frame (smooth)
}
```

**Pros:**
- Deterministic — identical input produces identical results. Critical for replays, debugging, and multiplayer sync.
- Movement stays perfectly grid-aligned; snake never drifts between cells.
- Physics is trivial: each tick moves exactly 1 cell.

**Cons:**
- "Spiral of death": if `update()` takes longer than `TICK_DURATION`, the loop falls behind. Mitigated by capping accumulator (e.g., `accumulator = Math.min(accumulator, TICK_DURATION * 3)`).
- Slight visual stutter on low-refresh displays if render doesn't interpolate.

### Variable Timestep

Game logic runs every frame using the actual delta time.

```js
// Variable-timestep loop (NOT recommended for grid-based snake)
let lastTime = 0;

function gameLoop(timestamp) {
  requestAnimationFrame(gameLoop);
  const delta = (timestamp - lastTime) / 1000; // seconds
  lastTime = timestamp;

  update(delta); // Position += velocity * delta
  render();
}
```

**Pros:**
- Simple — no accumulator math.
- Smooth continuous movement for physics-based games (platformers, space shooters).

**Cons for Snake:**
- Non-deterministic. Frame drops cause variable-sized moves.
- Grid alignment is fragile. Snake can land between cells, complicating collision detection.
- Harder to tune difficulty (speed becomes coupled to framerate).

### Hybrid: Fixed-step logic + interpolated rendering

For smooth visuals, interpolate snake position between ticks during render:

```js
const CELL_SIZE = 20; // px per grid cell

function render(alpha) {
  // alpha = accumulator / TICK_DURATION (0.0 to 1.0)
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  // Interpolate head between previous and current position
  const headVisualX = (prevX + (currX - prevX) * alpha) * CELL_SIZE;
  const headVisualY = (prevY + (currY - prevY) * alpha) * CELL_SIZE;
  // ...
}
```

For a classic snake at 6–15 ticks/sec, interpolation is usually unnecessary — the eye accepts discrete cell movement.

---

## 2. Grid-Based Movement and Rendering

### Grid Model

Define a grid of columns × rows, each cell `CELL_SIZE` pixels:

```js
const COLS = 30;
const ROWS = 20;
const CELL_SIZE = 20; // px

const canvas = document.getElementById('game-canvas');
canvas.width = COLS * CELL_SIZE;
canvas.height = ROWS * CELL_SIZE;
```

### Rendering Strategy

**Layered approach** (efficient, minimal redraws):

1. **Static background layer** — draw grid lines once to an offscreen canvas, reuse.
2. **Dynamic layer** — clear each frame, draw snake + food only.

```js
// --- Background (drawn once) ---
const bgCanvas = document.createElement('canvas');
bgCanvas.width = canvas.width;
bgCanvas.height = canvas.height;
const bgCtx = bgCanvas.getContext('2d');

function drawBackground() {
  // Alternating checkerboard for visibility
  for (let r = 0; r < ROWS; r++) {
    for (let c = 0; c < COLS; c++) {
      bgCtx.fillStyle = (r + c) % 2 === 0 ? '#a3d977' : '#91c765';
      bgCtx.fillRect(c * CELL_SIZE, r * CELL_SIZE, CELL_SIZE, CELL_SIZE);
    }
  }
  // Optional grid lines
  bgCtx.strokeStyle = 'rgba(0,0,0,0.05)';
  bgCtx.lineWidth = 0.5;
  for (let r = 0; r <= ROWS; r++) {
    bgCtx.beginPath();
    bgCtx.moveTo(0, r * CELL_SIZE);
    bgCtx.lineTo(COLS * CELL_SIZE, r * CELL_SIZE);
    bgCtx.stroke();
  }
  for (let c = 0; c <= COLS; c++) {
    bgCtx.beginPath();
    bgCtx.moveTo(c * CELL_SIZE, 0);
    bgCtx.lineTo(c * CELL_SIZE, ROWS * CELL_SIZE);
    bgCtx.stroke();
  }
}
drawBackground();

// --- Per-frame render ---
function render() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  ctx.drawImage(bgCanvas, 0, 0);         // blit background

  // Draw food
  ctx.fillStyle = '#e74c3c';
  ctx.beginPath();
  ctx.arc(
    food.x * CELL_SIZE + CELL_SIZE / 2,
    food.y * CELL_SIZE + CELL_SIZE / 2,
    CELL_SIZE / 2 - 2, 0, Math.PI * 2
  );
  ctx.fill();

  // Draw snake (head distinct from body)
  snake.forEach((segment, i) => {
    const isHead = i === 0;
    ctx.fillStyle = isHead ? '#2d5016' : '#4a7c28';
    roundRect(
      segment.x * CELL_SIZE + 1,
      segment.y * CELL_SIZE + 1,
      CELL_SIZE - 2,
      CELL_SIZE - 2,
      4
    );
  });
}

// Helper: rounded rectangles
function roundRect(x, y, w, h, r) {
  ctx.beginPath();
  ctx.moveTo(x + r, y);
  ctx.lineTo(x + w - r, y);
  ctx.quadraticCurveTo(x + w, y, x + w, y + r);
  ctx.lineTo(x + w, y + h - r);
  ctx.quadraticCurveTo(x + w, y + h, x + w - r, y + h);
  ctx.lineTo(x + r, y + h);
  ctx.quadraticCurveTo(x, y + h, x, y + h - r);
  ctx.lineTo(x, y + r);
  ctx.quadraticCurveTo(x, y, x + r, y);
  ctx.closePath();
  ctx.fill();
}
```

**Optimization tips:**
- Only redraw changed cells if using "dirty rectangle" update (overkill for ≤30×20 grid).
- Avoid `canvas.getContext('2d')` every frame — cache the context reference.
- For the background, `putImageData` is faster than `fillRect` loops but the grid is small enough that it doesn't matter.
- Use `willReadFrequently: true` context option if you need pixel reads (you shouldn't for snake).

---

## 3. Collision Detection

### Wall Collision

Since the snake occupies a finite grid, check whether the next head position is outside bounds:

```js
function checkWallCollision(head) {
  return head.x < 0 || head.x >= COLS || head.y < 0 || head.y >= ROWS;
}
```

**Wrap-around variant** (optional game mode):

```js
function applyWrap(head) {
  head.x = (head.x + COLS) % COLS;
  head.y = (head.y + ROWS) % ROWS;
}
```

### Self-Collision

Check if the new head position overlaps any existing body segment. The tail will vacate its cell before the check (unless growing), so exclude it:

```js
function checkSelfCollision(head, body) {
  // body[0] is head; body[body.length-1] is tail (will move, except when growing)
  // Check from segment[1] because head==head trivially
  // If NOT growing, tail occupies its current cell, but it will move — safe to skip.
  // If GROWING, tail stays — must check full body.
  for (let i = 1; i < body.length; i++) {
    if (body[i].x === head.x && body[i].y === head.y) {
      return true;
    }
  }
  return false;
}
```

**Optimized with Set** (O(1) lookup for large snakes):

```js
const bodySet = new Set();
function rebuildBodySet(body) {
  bodySet.clear();
  for (const seg of body) {
    bodySet.add(key(seg));
  }
}
function key(seg) { return `${seg.x},${seg.y}`; }

// Then self-collision is:
// Rebuild set once per tick after tail removal but before head insertion
// Check: bodySet.has(key(newHead))
```

For a typical snake ≤100 segments, the O(n) loop is fine. The Set approach only matters at extreme lengths or if collision is checked multiple times per frame.

### Food (Rabbit) Collision

Simple coordinate equality check:

```js
function checkFoodCollision(head, food) {
  return head.x === food.x && head.y === food.y;
}
```

On collision: increment score, grow snake, spawn new food avoiding snake body.

```js
function spawnFood(body) {
  const occupied = new Set(body.map(seg => `${seg.x},${seg.y}`));
  let pos;
  do {
    pos = {
      x: Math.floor(Math.random() * COLS),
      y: Math.floor(Math.random() * ROWS)
    };
  } while (occupied.has(key(pos)));
  return pos;
}
```

Edge case: if the snake fills the whole board, `do/while` loops forever. Guard with a max-attempt counter or a total-cell check:

```js
if (occupied.size >= COLS * ROWS) {
  // Player wins! All cells filled.
  return null;
}
```

### Collision Resolution Order (per tick)

1. Compute next head position from current direction.
2. Check wall collision → game over.
3. Check self-collision → game over.
4. Check food collision → grow + score + spawn new food.
5. If no food: remove tail (move normally).
6. Insert new head into body.

---

## 4. Snake Body Management

### Array (Recommended for ≤30×20 grid)

The simplest and most common approach. The array represents segments head-first: `[head, ..., tail]`.

**Movement (no growth):**
```js
// pop tail, unshift new head
body.pop();
body.unshift(newHead);
```

**Movement (with growth):**
```js
// Don't pop tail — the snake lengthens by 1
body.unshift(newHead);
```

**Full update step:**
```js
function update() {
  // Save previous head for interpolation
  prevHead = { ...body[0] };

  // Compute new head
  const newHead = {
    x: body[0].x + direction.dx,
    y: body[0].y + direction.dy
  };

  // Collision checks (wall, self)
  if (checkWallCollision(newHead)) { gameOver(); return; }
  if (checkSelfCollision(newHead, body)) { gameOver(); return; }

  // Prepend new head
  body.unshift(newHead);

  // Food?
  if (checkFoodCollision(newHead, food)) {
    score++;
    food = spawnFood(body);   // growth: don't pop tail
  } else {
    body.pop();               // normal: remove tail
  }
}
```

**Array pros:**
- Simple, readable, easy to debug.
- JS engines optimize `unshift`/`pop` for small arrays.
- Direct iteration for rendering and collision checks.

**Array cons:**
- `unshift` is O(n) — shifts all elements. For a 100-segment snake at 10 fps, imperceptible. Only a concern above ~10,000 segments (impossible on a 30×20 grid).

### Queue (Deque via two stacks)

A proper queue gives O(1) push/pop at both ends. In JS, implement as two stacks:

```js
class Deque {
  constructor() {
    this.head = [];   // front stored in reverse
    this.tail = [];
  }
  pushFront(item) { this.head.push(item); }
  popBack() {
    if (this.tail.length === 0) {
      // Rebalance: move half from head to tail
      const mid = Math.ceil(this.head.length / 2);
      while (this.head.length > mid) {
        this.tail.push(this.head.pop());
      }
    }
    return this.tail.pop();
  }
  // For iteration (render/collision):
  [Symbol.iterator]() {
    const h = [...this.head].reverse();
    return h.concat(this.tail)[Symbol.iterator]();
  }
}
```

This is overkill for a 30×20 grid snake (max 600 cells). Use an array.

### Linked List

Each segment is a node with `{x, y, next}`. The snake holds `head` and `tail` references.

```js
class SnakeNode {
  constructor(x, y) { this.x = x; this.y = y; this.next = null; }
}

class Snake {
  constructor(startX, startY, length) {
    this.head = new SnakeNode(startX, startY);
    let node = this.head;
    for (let i = 1; i < length; i++) {
      node.next = new SnakeNode(startX - i, startY); // horizontal start
      node = node.next;
    }
    this.tail = node;
  }

  move(newX, newY, grow) {
    const newNode = new SnakeNode(newX, newY);
    newNode.next = this.head;
    this.head = newNode;
    if (!grow) {
      // Remove tail: traverse to second-last
      let prev = this.head;
      while (prev.next !== this.tail) prev = prev.next;
      prev.next = null;
      this.tail = prev;
    }
  }

  // Convert to array for rendering/collision
  toArray() {
    const arr = [];
    let node = this.head;
    while (node) { arr.push(node); node = node.next; }
    return arr;
  }
}
```

**Linked list pros:**
- O(1) head insertion, O(1) tail removal (if doubly-linked or with `prev` reference).
- No array shifting overhead.

**Linked list cons:**
- More code, more bug-prone.
- Iteration for rendering/collision is still O(n).
- GC pressure from object-per-node (memory overhead).
- Completely unnecessary for a classic snake.

**Verdict:** Use a plain array. The grid caps at 30×20=600 cells, and 600-element array shifts at 10 fps are trivial for any modern JS engine.

### Growth Mechanics

Two common approaches:

**1. Delayed growth (recommended):** Grow by 1 segment per food eaten. The body simply doesn't pop the tail for one tick per food item. Implement as a counter:

```js
let growthQueue = 0; // segments to grow

// On food eat:
growthQueue += 1; // or more for "super food"

// In update:
if (growthQueue > 0) {
  growthQueue--;
  // don't pop tail
} else {
  body.pop();
}
```

**2. Instant growth:** Append N segments to the tail immediately. Visually jarring unless animated.

---

## 5. Input Handling

### Keyboard Events

Listen on `keydown` for responsive input. Avoid `keyup` (slow) and `keypress` (deprecated).

```js
const DIRECTION = {
  UP:    { dx:  0, dy: -1 },
  DOWN:  { dx:  0, dy:  1 },
  LEFT:  { dx: -1, dy:  0 },
  RIGHT: { dx:  1, dy:  0 },
};

// Arrow keys + WASD
const keyMap = {
  ArrowUp:    'UP',    w: 'UP',    W: 'UP',
  ArrowDown:  'DOWN',  s: 'DOWN',  S: 'DOWN',
  ArrowLeft:  'LEFT',  a: 'LEFT',  A: 'LEFT',
  ArrowRight: 'RIGHT', d: 'RIGHT', D: 'RIGHT',
};

document.addEventListener('keydown', (e) => {
  if (keyMap[e.key]) {
    e.preventDefault(); // prevent page scrolling with arrows
    queueDirection(keyMap[e.key]);
  }
});
```

### Direction Queue (Prevent 180° Reversal)

The core problem: if the snake is moving RIGHT and the player presses LEFT then DOWN in the same tick, the snake would reverse into itself. Solution: buffer inputs in a queue.

```js
let dirQueue = [];    // FIFO queue of pending direction changes
let currentDir = DIRECTION.RIGHT;

function queueDirection(newDirName) {
  const newDir = DIRECTION[newDirName];
  // Determine the last queued direction (or current if queue empty)
  const lastDir = dirQueue.length > 0
    ? DIRECTION[dirQueue[dirQueue.length - 1]]
    : currentDir;

  // Prevent 180° reversal: new direction must not be opposite of last
  if (newDir.dx + lastDir.dx !== 0 || newDir.dy + lastDir.dy !== 0) {
    // Prevent duplicate consecutive directions (optional — prevents queue spam)
    if (newDir !== lastDir) {
      dirQueue.push(newDirName);
    }
  }
}
```

**Processing the queue (called once per game tick):**

```js
function consumeDirection() {
  while (dirQueue.length > 0) {
    const nextDirName = dirQueue.shift();
    const nextDir = DIRECTION[nextDirName];
    // Re-check against current active direction (not queue head)
    if (nextDir.dx + currentDir.dx !== 0 || nextDir.dy + currentDir.dy !== 0) {
      currentDir = nextDir;
      break; // only consume ONE input per tick
    }
    // If it was a reversal (stale), discard and check next in queue
  }
}
```

**Key design decisions:**
- **At most 1 direction change per tick.** This gives the player one turn per grid cell, which is the defining constraint of snake. Consuming more than one per tick lets the player zigzag inside a single cell, breaking the game feel.
- **Queue cap:** Limit to 2–3 entries to prevent unbounded buffering during lag spikes:

```js
const MAX_QUEUE = 3;

function queueDirection(newDirName) {
  if (dirQueue.length >= MAX_QUEUE) return;
  // ... rest of validation
}
```

- **Validate against last queued direction** (not current) to prevent the classic "double-tap reversal" where two rapid inputs cancel out within one tick.

### Touch/Swipe Support (Mobile)

```js
let touchStartX, touchStartY;

canvas.addEventListener('touchstart', (e) => {
  e.preventDefault();
  touchStartX = e.touches[0].clientX;
  touchStartY = e.touches[0].clientY;
});

canvas.addEventListener('touchend', (e) => {
  e.preventDefault();
  const dx = e.changedTouches[0].clientX - touchStartX;
  const dy = e.changedTouches[0].clientY - touchStartY;

  if (Math.abs(dx) > Math.abs(dy)) {
    queueDirection(dx > 0 ? 'RIGHT' : 'LEFT');
  } else if (Math.abs(dy) > 0) {
    queueDirection(dy > 0 ? 'DOWN' : 'UP');
  }
});
```

---

## Complete Game Loop Pattern

Tying it all together:

```js
// ============ CONSTANTS ============
const COLS = 30, ROWS = 20, CELL_SIZE = 20;
const TICK_RATE = 10; // 10 moves per second (adjustable difficulty)
const TICK_DURATION = 1000 / TICK_RATE;
const MAX_DIR_QUEUE = 3;

// ============ STATE ============
let body = [];          // [{x, y}, ...] head-first
let food = null;
let currentDir = { dx: 1, dy: 0 };  // start moving right
let dirQueue = [];
let score = 0;
let gameActive = false;
let growthQueue = 0;

// Timing
let lastTickTime = 0;
let accumulator = 0;

// DOM
const canvas = document.getElementById('game-canvas');
const ctx = canvas.getContext('2d');

// ============ INIT ============
function init() {
  const startX = Math.floor(COLS / 2);
  const startY = Math.floor(ROWS / 2);
  body = [
    { x: startX,     y: startY },
    { x: startX - 1, y: startY },
    { x: startX - 2, y: startY },
  ];
  currentDir = { dx: 1, dy: 0 };
  dirQueue = [];
  score = 0;
  growthQueue = 0;
  gameActive = true;
  food = spawnFood(body);
  lastTickTime = performance.now();
  accumulator = 0;

  requestAnimationFrame(gameLoop);
}

// ============ GAME LOOP ============
function gameLoop(timestamp) {
  if (!gameActive) return;

  const delta = timestamp - lastTickTime;
  lastTickTime = timestamp;
  accumulator += delta;

  // Clamp to prevent spiral of death
  if (accumulator > TICK_DURATION * 3) {
    accumulator = TICK_DURATION * 3;
  }

  while (accumulator >= TICK_DURATION) {
    update();
    if (!gameActive) return; // game ended mid-update
    accumulator -= TICK_DURATION;
  }

  render();
  requestAnimationFrame(gameLoop);
}

// ============ UPDATE ============
function update() {
  // Consume exactly one direction from the queue
  consumeDirection();

  // Compute new head
  const head = body[0];
  const newHead = {
    x: head.x + currentDir.dx,
    y: head.y + currentDir.dy
  };

  // Wall collision
  if (newHead.x < 0 || newHead.x >= COLS || newHead.y < 0 || newHead.y >= ROWS) {
    endGame('wall');
    return;
  }

  // Self collision (check all segments except the tail that will move)
  // If growing, tail stays — check all segments
  const checkLimit = growthQueue > 0 ? body.length : body.length - 1;
  for (let i = 0; i < checkLimit; i++) {
    if (body[i].x === newHead.x && body[i].y === newHead.y) {
      endGame('self');
      return;
    }
  }

  // Move: insert new head
  body.unshift(newHead);

  // Food collision
  if (newHead.x === food.x && newHead.y === food.y) {
    score += 10;
    growthQueue += 1;
    food = spawnFood(body);
    if (!food) { endGame('win'); return; }
  }

  // Tail management
  if (growthQueue > 0) {
    growthQueue--;
    // leave tail → snake grows
  } else {
    body.pop(); // remove tail
  }
}

// ============ INPUT ============
const KEY_MAP = {
  ArrowUp: 'UP', w: 'UP', W: 'UP',
  ArrowDown: 'DOWN', s: 'DOWN', S: 'DOWN',
  ArrowLeft: 'LEFT', a: 'LEFT', A: 'LEFT',
  ArrowRight: 'RIGHT', d: 'RIGHT', D: 'RIGHT',
};
const DIR_MAP = {
  UP:    { dx:  0, dy: -1 },
  DOWN:  { dx:  0, dy:  1 },
  LEFT:  { dx: -1, dy:  0 },
  RIGHT: { dx:  1, dy:  0 },
};

function queueDirection(dirName) {
  if (dirQueue.length >= MAX_DIR_QUEUE) return;
  const newDir = DIR_MAP[dirName];
  const lastDir = dirQueue.length > 0
    ? DIR_MAP[dirQueue[dirQueue.length - 1]]
    : currentDir;
  // Prevent 180° reversal
  if (newDir.dx + lastDir.dx === 0 && newDir.dy + lastDir.dy === 0) return;
  // Prevent consecutive duplicates
  if (newDir.dx === lastDir.dx && newDir.dy === lastDir.dy) return;
  dirQueue.push(dirName);
}

function consumeDirection() {
  while (dirQueue.length > 0) {
    const nextName = dirQueue.shift();
    const nextDir = DIR_MAP[nextName];
    if (nextDir.dx + currentDir.dx !== 0 || nextDir.dy + currentDir.dy !== 0) {
      currentDir = nextDir;
      break; // only one direction change per tick
    }
  }
}

document.addEventListener('keydown', (e) => {
  if (KEY_MAP[e.key]) {
    e.preventDefault();
    queueDirection(KEY_MAP[e.key]);
  }
});

// ============ RENDER ============
function render() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  // Background (assume bgCanvas is pre-rendered, see Section 2)
  ctx.drawImage(bgCanvas, 0, 0);

  // Food
  ctx.fillStyle = '#e74c3c';
  const fx = food.x * CELL_SIZE + CELL_SIZE / 2;
  const fy = food.y * CELL_SIZE + CELL_SIZE / 2;
  ctx.beginPath();
  ctx.arc(fx, fy, CELL_SIZE / 2 - 2, 0, Math.PI * 2);
  ctx.fill();

  // Snake
  body.forEach((seg, i) => {
    ctx.fillStyle = i === 0 ? '#2d5016' : '#4a7c28';
    const pad = 1;
    const x = seg.x * CELL_SIZE + pad;
    const y = seg.y * CELL_SIZE + pad;
    const w = CELL_SIZE - pad * 2;
    const h = CELL_SIZE - pad * 2;
    const r = 4;
    ctx.beginPath();
    ctx.moveTo(x + r, y);
    ctx.lineTo(x + w - r, y);
    ctx.quadraticCurveTo(x + w, y, x + w, y + r);
    ctx.lineTo(x + w, y + h - r);
    ctx.quadraticCurveTo(x + w, y + h, x + w - r, y + h);
    ctx.lineTo(x + r, y + h);
    ctx.quadraticCurveTo(x, y + h, x, y + h - r);
    ctx.lineTo(x, y + r);
    ctx.quadraticCurveTo(x, y, x + r, y);
    ctx.closePath();
    ctx.fill();
  });
}

// ============ FOOD SPAWNING ============
function spawnFood(body) {
  const occupied = new Set(body.map(s => `${s.x},${s.y}`));
  if (occupied.size >= COLS * ROWS) return null; // board full
  let pos, attempts = 0;
  do {
    pos = { x: Math.floor(Math.random() * COLS), y: Math.floor(Math.random() * ROWS) };
    attempts++;
  } while (occupied.has(`${pos.x},${pos.y}`) && attempts < 1000);
  return pos;
}

// ============ GAME STATE ============
function endGame(reason) {
  gameActive = false;
  console.log(`Game over: ${reason}. Score: ${score}`);
  // UI: show restart button
}

function restart() { init(); }

// ============ BOOT ============
canvas.width = COLS * CELL_SIZE;
canvas.height = ROWS * CELL_SIZE;
init();
```

---

## Summary of Recommendations

| Concern | Recommendation |
|---------|---------------|
| Game loop | Fixed timestep with `requestAnimationFrame` |
| Tick rate | 8–15 Hz (adjustable difficulty) |
| Snake data structure | Plain array (head-first) |
| Direction input | Queue with 1 consume/tick, max 3 buffered |
| 180° prevention | Validate against last queued direction |
| Rendering | Pre-rendered background + per-frame snake/food draw |
| Collision check | O(n) linear scan (grid caps at 600 cells) |
| Growth | `growthQueue` counter — don't pop tail that tick |
| Food spawning | Random + exclusion set, with board-full guard |
