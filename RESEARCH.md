# RESEARCH: Rabbit AI for Carrot Maze

## Domain Overview
Carrot Maze is a browser game (HTML5 Canvas + vanilla JS) where a snake (python) hunts rabbits on a grid of carrot beds. Rabbits jump between adjacent beds at intervals. This research covers pathfinding, behavior patterns, implementation approach, and spawning for the rabbit AI.

---

## 1. Pathfinding Algorithms for Grid Movement

### Algorithm Comparison

| Algorithm | Weighted | Heuristic | Complexity | Best Use Case |
|-----------|----------|-----------|------------|---------------|
| **BFS** | No | No | O(V+E) | Unweighted grids, equal-cost moves |
| **Dijkstra** | Yes | No | O((V+E)log V) | Weighted terrain (variable costs) |
| **A\*** | Yes | Yes | O(E) typically | Goal-directed search with heuristic |

### Analysis for This Game

**Key insight: Rabbits make SINGLE-JUMP decisions, not multi-hop routes.** A rabbit only needs to evaluate its immediate adjacent beds (neighbors). Full pathfinding is overkill 90% of the time because:

- The bed graph is small and sparse (beds are grid-connected pockets)
- Jump interval is short (2-4s), so rabbits rarely need to plan multi-hop escapes
- Evaluating neighbors with a score function is O(degree) — trivial

### Recommendations

| Scenario | Algorithm | Rationale |
|----------|-----------|-----------|
| **Pick next bed (default)** | Weighted scoring of adjacent beds | No pathfinding needed — score each neighbor by distance to snake, carrot density, randomness |
| **Flee from snake (local)** | Evaluate adjacent beds, pick furthest from snake | Single-hop escape decision |
| **Flee from snake (long-range)** | BFS from each adjacent bed to measure safe distance | BFS gives shortest-path distance. Run one BFS from snake position, then check distances to each candidate bed |
| **Patrol / return to home bed** | A\* (Manhattan heuristic) | Only when rabbit needs to navigate back to a specific bed across multiple hops |

### Why NOT Dijkstra?
All bed-to-bed transitions have uniform cost (one jump). Dijkstra's priority queue adds overhead with zero benefit when edge weights are equal. BFS is strictly better for unweighted grids.

### Why A\* for patrol?
A\* with Manhattan distance heuristic prunes the search space. On a grid maze, it explores far fewer nodes than BFS for targeted navigation. But for local "pick next neighbor" decisions, it's still heavier than needed.

### The Winning Approach: Snake-Distance BFS + Neighbor Scoring

Run **one BFS from the snake's position** across the bed graph. This gives every bed a `distanceFromSnake` value. Then each rabbit:
1. Looks at adjacent beds
2. Picks the one with highest `distanceFromSnake` (weighted with randomness)
3. If snake is far away (distance > 5), uses random/patrol behavior instead

This is O(V+E) once per frame for ALL rabbits, not per rabbit.

---

## 2. Behavior Patterns

### 2.1 Random Walk (Baseline)
Each jump interval, pick a random adjacent bed with uniform probability.

- **Pros:** Simplest, unpredictable, good baseline
- **Cons:** Looks robotic, may jump into snake accidentally
- **Best for:** Easy difficulty, distant rabbits

```js
// Pseudocode: Random walk
function randomWalk(rabbit) {
    const neighbors = getAdjacentBeds(rabbit.currentBed);
    return neighbors[Math.floor(Math.random() * neighbors.length)];
}
```

### 2.2 Flee from Snake
When the snake is within a danger radius, the rabbit should jump to the bed that maximizes distance from the snake.

- **Danger radius:** 2-3 grid units (tunable per difficulty)
- **Tie-breaking:** Add randomness among equally safe options
- **Imperfect flee:** 20-30% chance of suboptimal choice (rabbits aren't geniuses)

```js
// Pseudocode: Flee behavior
function fleeFromSnake(rabbit, snakePos, bedDistances) {
    const neighbors = getAdjacentBeds(rabbit.currentBed);
    let best = neighbors[0];
    let bestScore = -Infinity;

    for (const bed of neighbors) {
        const dist = bedDistances.get(bed); // from precomputed BFS
        // Add slight randomness for unpredictability
        const score = dist + (Math.random() * 0.5);
        if (score > bestScore) {
            bestScore = score;
            best = bed;
        }
    }
    return best;
}
```

### 2.3 Patrol Between Beds
Each rabbit is assigned 2-4 "favorite beds" in a region. It cycles between them.

- Creates natural-looking movement
- Different rabbits patrol different zones
- Mix with flee: if snake enters patrol zone, flee overrides patrol

```js
// Pseudocode: Patrol
function patrol(rabbit) {
    const route = rabbit.patrolRoute; // array of bed IDs
    const currentIdx = route.indexOf(rabbit.currentBed);
    const nextIdx = (currentIdx + 1) % route.length;
    // Find path to next patrol bed using A*
    return aStarNextStep(rabbit.currentBed, route[nextIdx]);
}
```

### 2.4 Weighted Movement (Recommended Default)
Score each adjacent bed on multiple weighted factors, then pick the highest-scoring one (with some randomness).

**Scoring factors:**

| Factor | Weight Range | Description |
|--------|-------------|-------------|
| Snake proximity | -1.0 to 0 | Inverse distance to snake (higher = safer). Dominant when snake is close. |
| Carrot bed affinity | +0.2 to +0.5 | Preference for staying on/near beds with carrots |
| Recent visit penalty | -0.3 | Avoid returning to bed just visited (prevents ping-pong) |
| Random jitter | ±0.3 | Makes behavior less robotic |
| Edge avoidance | -0.5 | Penalize beds near maze edges (optional) |
| Crowding | -0.3 | Avoid beds already occupied by other rabbits |

```js
// Pseudocode: Weighted neighbor scoring
function scoreBed(bed, rabbit, snakePos, bedDistances) {
    let score = 0;

    // Flee component (dominant when snake is close)
    const snakeDist = bedDistances.get(bed);
    score += Math.min(snakeDist / 5, 1.0) * 2.0; // cap at distance 5

    // Carrot affinity
    if (bed.hasCarrots) score += 0.3;

    // Recent visit penalty
    if (bed.id === rabbit.lastBed) score -= 0.5;

    // Randomness
    score += (Math.random() - 0.5) * 0.6;

    // Avoid other rabbits
    if (bed.occupiedBy) score -= 0.4;

    return score;
}
```

### 2.5 Behavior State Machine
Rabbits transition between states based on snake proximity:

```
         snake far (>4 tiles)
  PATROL ──────────────────────► RANDOM
    │                                │
    │ snake near (≤3 tiles)          │ snake near (≤3 tiles)
    ▼                                ▼
  FLEE ◄─────────────────────────────┘
    │
    │ snake very close (≤1 tile)
    ▼
  FREEZE/PANIC (1s delay, then jump randomly)
```

### 2.6 Believable Imperfection
Rabbits should NOT be optimal. The player needs to catch them. Add imperfection:
- **Reaction delay:** 200-500ms before responding to snake proximity
- **Wrong choice rate:** 10-25% chance of picking a suboptimal bed when fleeing
- **Freeze chance:** 5-10% chance of not jumping at all (rabbit "freezes")
- **Variable jump intervals:** Some rabbits are slower than others
- **Personality traits:** Assign each rabbit a "boldness" (0-1), "skittishness" (0-1), "speed" (0-1)

---

## 3. Implementation Approach

### 3.1 Architecture Overview

```
Game Loop (requestAnimationFrame)
├── update(deltaTime)
│   ├── updateSnake()
│   ├── updateRabbits(deltaTime)   ← Rabbit AI
│   │   ├── updateTimers(deltaTime)
│   │   ├── selectTargets()
│   │   └── animateMovement(deltaTime)
│   └── checkCollisions()
└── render()
    ├── drawMaze()
    ├── drawCarrots()
    ├── drawSnake()
    └── drawRabbits()   ← lerp positions for smooth animation
```

### 3.2 Rabbit Data Structure

```js
class Rabbit {
    constructor(bed, personality) {
        this.currentBed = bed;        // Bed object (grid position)
        this.previousBed = null;      // For visit penalty
        this.targetBed = null;        // Destination during jump
        this.visualX = bed.x;         // Current visual position (for animation)
        this.visualY = bed.y;         // Current visual position (for animation)
        this.animProgress = 0;        // 0→1 lerp progress
        this.animDuration = 350;      // ms for jump animation
        this.jumpTimer = 0;           // Countdown to next jump
        this.jumpInterval = random(2000, 4000); // ms between jumps
        this.state = 'idle';          // idle | jumping | frozen
        this.personality = personality || {
            boldness: random(0.3, 0.8),
            skittishness: random(0.2, 0.9),
            speed: random(0.5, 1.0)
        };
        this.patrolRoute = [];        // Optional patrol beds
        this.lastBed = null;          // Anti-ping-pong
    }
}
```

### 3.3 Timer System

```js
function updateRabbitTimers(rabbit, deltaTime) {
    if (rabbit.state === 'jumping') return; // already in air
    if (rabbit.state === 'frozen') {
        rabbit.freezeTimer -= deltaTime;
        if (rabbit.freezeTimer <= 0) rabbit.state = 'idle';
        return;
    }

    rabbit.jumpTimer -= deltaTime;
    if (rabbit.jumpTimer <= 0) {
        // Time to jump!
        const target = selectTarget(rabbit, snake, allRabbits);
        initiateJump(rabbit, target);
    }
}

function initiateJump(rabbit, targetBed) {
    rabbit.targetBed = targetBed;
    rabbit.previousBed = rabbit.currentBed;
    rabbit.animProgress = 0;
    // Adjust animation duration based on rabbit speed personality
    rabbit.animDuration = 350 / rabbit.personality.speed;
    rabbit.state = 'jumping';
}

function resetJumpTimer(rabbit) {
    // Base interval adjusted by personality
    const base = 2500 + rabbit.personality.boldness * 1500; // 2.5-4s
    rabbit.jumpInterval = base + (Math.random() - 0.5) * 1000; // ±500ms jitter
    rabbit.jumpTimer = rabbit.jumpInterval;
}
```

### 3.4 Target Selection (Complete)

```js
function selectTarget(rabbit, snake, allRabbits, bedDistances) {
    const neighbors = getAdjacentBeds(rabbit.currentBed);
    if (neighbors.length === 0) return rabbit.currentBed; // no escape

    const snakeDist = bedDistances.get(rabbit.currentBed);
    const dangerThreshold = 3 + rabbit.personality.skittishness * 2; // 3-5 range

    let candidates;

    if (snakeDist <= dangerThreshold) {
        // FLEE: score by distance from snake
        candidates = neighbors.map(bed => ({
            bed,
            score: fleeScore(bed, rabbit, bedDistances, allRabbits)
        }));
    } else {
        // ROAM: random-ish with slight carrot bias
        candidates = neighbors.map(bed => ({
            bed,
            score: roamScore(bed, rabbit, allRabbits)
        }));
    }

    // Sort by score descending, then pick with weighted randomness from top 2
    candidates.sort((a, b) => b.score - a.score);
    const topPool = candidates.slice(0, Math.min(2, candidates.length));
    // Weighted random: higher score = more likely
    return weightedRandomPick(topPool);
}

function fleeScore(bed, rabbit, bedDistances, allRabbits) {
    let score = bedDistances.get(bed) * 2.0; // distance from snake

    // Imperfection: 20% chance of random override
    if (Math.random() < 0.2) score += (Math.random() - 0.5) * 10;

    // Avoid beds with other rabbits
    if (allRabbits.some(r => r.currentBed === bed)) score -= 3;

    // Anti-ping-pong: penalize returning to previous bed
    if (bed === rabbit.previousBed) score -= 5;

    return score;
}

function roamScore(bed, rabbit, allRabbits) {
    let score = 0;

    // Carrot affinity
    if (bed.hasCarrots) score += 0.5;

    // Avoid last bed
    if (bed === rabbit.previousBed) score -= 0.4;

    // Avoid crowded beds
    if (allRabbits.some(r => r.currentBed === bed)) score -= 0.5;

    // Randomness
    score += (Math.random() - 0.5) * 0.8;

    return score;
}
```

### 3.5 Movement Animation (Smooth Lerp)

```js
function animateRabbitMovement(rabbit, deltaTime) {
    if (rabbit.state !== 'jumping') return;

    rabbit.animProgress += deltaTime / rabbit.animDuration;

    if (rabbit.animProgress >= 1.0) {
        // Landing
        rabbit.animProgress = 1.0;
        rabbit.visualX = rabbit.targetBed.x;
        rabbit.visualY = rabbit.targetBed.y;
        rabbit.currentBed = rabbit.targetBed;
        rabbit.targetBed = null;
        rabbit.state = 'idle';
        resetJumpTimer(rabbit);
        return;
    }

    // Ease-in-out for natural arc
    const t = easeInOutQuad(rabbit.animProgress);

    // Horizontal lerp
    rabbit.visualX = lerp(rabbit.previousBed.x, rabbit.targetBed.x, t);

    // Vertical lerp + arc (parabolic jump)
    rabbit.visualY = lerp(rabbit.previousBed.y, rabbit.targetBed.y, t);
    // Add arc offset: sin curve, peaks at t=0.5
    const arcHeight = 20; // pixels above bed
    rabbit.jumpOffset = Math.sin(t * Math.PI) * arcHeight;
}

function easeInOutQuad(t) {
    return t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t;
}

function lerp(a, b, t) {
    return a + (b - a) * t;
}
```

### 3.6 Render

```js
function drawRabbit(ctx, rabbit) {
    const x = rabbit.visualX;
    const y = rabbit.visualY - (rabbit.jumpOffset || 0);

    // Draw rabbit body (white with pixel-art style)
    ctx.fillStyle = '#ffffff';
    ctx.fillRect(x - 8, y - 8, 16, 16);

    // Ears
    ctx.fillStyle = '#ffcccc'; // pink inner ear
    ctx.fillRect(x - 6, y - 14, 4, 8);
    ctx.fillRect(x + 2, y - 14, 4, 8);

    // Eyes (direction based on movement)
    ctx.fillStyle = '#000000';
    ctx.fillRect(x - 3, y - 3, 2, 2);
    ctx.fillRect(x + 1, y - 3, 2, 2);
}
```

---

## 4. Rabbit Spawning

### 4.1 Where to Spawn

Rabbits SHALL spawn only on carrot beds. Spawn selection rules (in priority order):

1. **Not on the snake.** Minimum 3 tiles away from snake head.
2. **Spread out.** Minimum 2 tiles from other rabbits (use BFS distance).
3. **Prefer interior beds.** Avoid edge/perimeter beds (they have fewer escape routes).
4. **Prefer beds with ≥2 neighbors.** More escape options = fairer for the rabbit.

```js
function findSpawnBed(beds, snake, allRabbits) {
    // Filter eligible beds
    const eligible = beds.filter(bed => {
        // Must have carrots
        if (!bed.hasCarrots) return false;

        // Not too close to snake
        if (bfsDistance(bed, snake.headBed) < 3) return false;

        // Not occupied
        if (allRabbits.some(r => r.currentBed === bed)) return false;

        // Not too close to other rabbits
        if (allRabbits.some(r => bfsDistance(bed, r.currentBed) < 2)) return false;

        // Has at least 2 neighbors (escape routes)
        if (getAdjacentBeds(bed).length < 2) return false;

        return true;
    });

    if (eligible.length === 0) {
        // Fallback: relax constraints
        return beds.filter(b => b.hasCarrots && !allRabbits.some(r => r.currentBed === b))
                   .sort(() => Math.random() - 0.5)[0];
    }

    // Pick randomly from eligible
    return eligible[Math.floor(Math.random() * eligible.length)];
}
```

### 4.2 When to Spawn

| Event | Action |
|-------|--------|
| **Game start** | Spawn `initialCount` rabbits immediately |
| **Rabbit eaten** | Start a respawn timer (5-10s delay) |
| **Count below minimum** | Spawn one rabbit when timer fires |
| **Count at/above max** | Stop respawn timer |

```js
class RabbitSpawner {
    constructor(config) {
        this.minRabbits = 3;
        this.maxRabbits = 8;
        this.initialRabbits = 5;
        this.respawnDelay = 7000; // ms
        this.respawnTimer = 0;
        this.respawnQueued = false;
    }

    onRabbitEaten() {
        if (!this.respawnQueued) {
            this.respawnTimer = this.respawnDelay;
            this.respawnQueued = true;
        }
    }

    update(deltaTime, allRabbits, beds, snake) {
        if (this.respawnQueued) {
            this.respawnTimer -= deltaTime;
            if (this.respawnTimer <= 0) {
                this.respawnQueued = false;
                if (allRabbits.length < this.minRabbits) {
                    const bed = findSpawnBed(beds, snake, allRabbits);
                    if (bed) {
                        allRabbits.push(new Rabbit(bed, randomPersonality()));
                    }
                }
            }
        }
    }
}
```

### 4.3 Max Count & Difficulty Scaling

| Difficulty | Initial Rabbits | Min Rabbits | Max Rabbits | Jump Speed | Flee Distance | Wrong Choice Rate |
|------------|----------------|-------------|-------------|------------|---------------|-------------------|
| **Easy** | 6 | 4 | 10 | Slow (3-5s) | 2 tiles | 30% |
| **Medium** | 5 | 3 | 8 | Normal (2-4s) | 3 tiles | 20% |
| **Hard** | 4 | 3 | 6 | Fast (1.5-3s) | 4 tiles | 10% |

```js
const DIFFICULTY_CONFIG = {
    easy: {
        initialRabbits: 6, minRabbits: 4, maxRabbits: 10,
        jumpIntervalBase: 4000, jumpIntervalRange: 2000,
        fleeDistance: 2, wrongChoiceRate: 0.3,
        snakeSpeed: 0.7, // snake moves slower on easy
    },
    medium: {
        initialRabbits: 5, minRabbits: 3, maxRabbits: 8,
        jumpIntervalBase: 3000, jumpIntervalRange: 2000,
        fleeDistance: 3, wrongChoiceRate: 0.2,
        snakeSpeed: 1.0,
    },
    hard: {
        initialRabbits: 4, minRabbits: 3, maxRabbits: 6,
        jumpIntervalBase: 2250, jumpIntervalRange: 1500,
        fleeDistance: 4, wrongChoiceRate: 0.1,
        snakeSpeed: 1.3,
    },
};
```

### 4.4 Dynamic Difficulty (Optional Enhancement)
Track player's score rate. If they're catching rabbits too fast:
- Slightly increase flee distance
- Slightly decrease jump intervals
- Add one extra rabbit to max count

If they're struggling (no catches in 30s):
- Slightly decrease flee distance
- Slightly increase wrong choice rate
- Reduce max count

---

## 5. Complete JS Pseudocode: Rabbit AI Module

```js
// ============================================================
// rabbit-ai.js — Complete Rabbit AI Module
// ============================================================

// --- Rabbit Class ---
class Rabbit {
    constructor(bed, config) {
        this.currentBed = bed;
        this.previousBed = null;
        this.targetBed = null;
        this.visualX = bed.x;
        this.visualY = bed.y;
        this.animProgress = 0;
        this.animDuration = 350;
        this.jumpTimer = 0;
        this.state = 'idle';           // idle | jumping | frozen
        this.freezeTimer = 0;

        // Personality (0-1 range)
        this.personality = {
            boldness: randomRange(0.3, 0.8),
            skittishness: randomRange(0.2, 0.9),
            speed: randomRange(0.5, 1.0),
        };

        // Jump interval from config + personality modifier
        this.baseInterval = config.jumpIntervalBase +
            (1 - this.personality.boldness) * config.jumpIntervalRange;
        this.resetJumpTimer();

        // Patrol route (optional, for richer behavior)
        this.patrolRoute = [];
        this.patrolIndex = 0;
    }

    resetJumpTimer() {
        this.jumpInterval = this.baseInterval + randomRange(-500, 500);
        this.jumpTimer = this.jumpInterval;
    }
}

// --- Rabbit Spawner ---
class RabbitSpawner {
    constructor(config) {
        this.config = config;
        this.respawnDelay = 7000;
        this.respawnTimer = 0;
        this.respawnQueued = false;
    }

    spawnInitial(beds, snake) {
        const rabbits = [];
        for (let i = 0; i < this.config.initialRabbits; i++) {
            const bed = findSpawnBed(beds, snake, rabbits);
            if (bed) rabbits.push(new Rabbit(bed, this.config));
        }
        return rabbits;
    }

    queueRespawn() {
        if (!this.respawnQueued) {
            this.respawnTimer = this.respawnDelay;
            this.respawnQueued = true;
        }
    }

    update(dt, rabbits, beds, snake) {
        if (!this.respawnQueued) return;
        this.respawnTimer -= dt;
        if (this.respawnTimer <= 0) {
            this.respawnQueued = false;
            if (rabbits.length < this.config.minRabbits) {
                const bed = findSpawnBed(beds, snake, rabbits);
                if (bed) {
                    rabbits.push(new Rabbit(bed, this.config));
                }
            }
        }
    }
}

// --- Spawn Bed Selection ---
function findSpawnBed(beds, snake, rabbits) {
    const snakeBed = snake.segments[0]; // head
    const occupied = new Set(rabbits.map(r => r.currentBed));

    let candidates = beds.filter(b => {
        if (!b.hasCarrots) return false;
        if (occupied.has(b)) return false;
        if (bfsDistance(b, snakeBed, beds) < 3) return false;
        if (getNeighbors(b, beds).length < 2) return false;
        for (const r of rabbits) {
            if (bfsDistance(b, r.currentBed, beds) < 2) return false;
        }
        return true;
    });

    if (candidates.length === 0) {
        candidates = beds.filter(b => b.hasCarrots && !occupied.has(b));
    }
    if (candidates.length === 0) {
        candidates = beds.filter(b => b.hasCarrots);
    }

    return candidates[Math.floor(Math.random() * candidates.length)] || null;
}

// --- Bed Graph Helpers ---
function getNeighbors(bed, allBeds) {
    // Returns beds reachable in one jump from this bed
    // Neighbors are defined by adjacency in the bed graph
    return bed.neighbors || []; // precomputed during maze generation
}

function getAdjacentBeds(bed, allBeds) {
    return getNeighbors(bed, allBeds);
}

// --- BFS Distance Computation (run once per frame for all rabbits) ---
function computeSnakeDistances(snakeBed, allBeds) {
    const distances = new Map();
    const queue = [snakeBed];
    distances.set(snakeBed, 0);

    while (queue.length > 0) {
        const current = queue.shift();
        const currentDist = distances.get(current);

        for (const neighbor of getNeighbors(current, allBeds)) {
            if (!distances.has(neighbor)) {
                distances.set(neighbor, currentDist + 1);
                queue.push(neighbor);
            }
        }
    }

    // Unreachable beds get a high distance
    for (const bed of allBeds) {
        if (!distances.has(bed)) {
            distances.set(bed, 999);
        }
    }

    return distances;
}

// --- Behavior: Target Selection ---
function selectTarget(rabbit, snake, allRabbits, bedDistances, config) {
    const neighbors = getAdjacentBeds(rabbit.currentBed);
    if (neighbors.length === 0) return rabbit.currentBed;

    const snakeDist = bedDistances.get(rabbit.currentBed);
    const fleeThreshold = config.fleeDistance +
        Math.floor(rabbit.personality.skittishness * 2); // 3-5 range

    const isFleeing = snakeDist <= fleeThreshold;

    // Score all candidates
    const scored = neighbors.map(bed => ({
        bed,
        score: isFleeing
            ? fleeScore(bed, rabbit, bedDistances, allRabbits, config)
            : roamScore(bed, rabbit, allRabbits),
    }));

    scored.sort((a, b) => b.score - a.score);

    // Pick from top candidates with weighted randomness
    const poolSize = Math.min(2, scored.length);
    const pool = scored.slice(0, poolSize);

    // Weighted pick (roulette wheel)
    const totalWeight = pool.reduce((sum, c) => sum + Math.max(c.score, 0.1), 0);
    let roll = Math.random() * totalWeight;
    for (const c of pool) {
        roll -= Math.max(c.score, 0.1);
        if (roll <= 0) return c.bed;
    }
    return pool[pool.length - 1].bed;
}

function fleeScore(bed, rabbit, bedDistances, allRabbits, config) {
    let score = bedDistances.get(bed) * 2.0;

    // Imperfection: wrong choice rate
    if (Math.random() < config.wrongChoiceRate) {
        score += randomRange(-5, 5);
    }

    // Avoid beds with other rabbits
    if (allRabbits.some(r => r.currentBed === bed && r !== rabbit)) score -= 3;

    // Anti-ping-pong
    if (bed === rabbit.previousBed) score -= 5;

    return score;
}

function roamScore(bed, rabbit, allRabbits) {
    let score = 0;
    if (bed.hasCarrots) score += 0.5;
    if (bed === rabbit.previousBed) score -= 0.4;
    if (allRabbits.some(r => r.currentBed === bed && r !== rabbit)) score -= 0.5;
    score += randomRange(-0.4, 0.4);
    return score;
}

// --- Animation ---
function animateRabbit(rabbit, dt) {
    if (rabbit.state !== 'jumping') return;

    rabbit.animProgress += dt / rabbit.animDuration;

    if (rabbit.animProgress >= 1.0) {
        // Land
        rabbit.animProgress = 1.0;
        rabbit.visualX = rabbit.targetBed.x;
        rabbit.visualY = rabbit.targetBed.y;
        rabbit.previousBed = rabbit.currentBed;
        rabbit.currentBed = rabbit.targetBed;
        rabbit.targetBed = null;
        rabbit.state = 'idle';
        rabbit.resetJumpTimer();
        return;
    }

    const t = easeInOutQuad(rabbit.animProgress);
    rabbit.visualX = lerp(rabbit.previousBed.x, rabbit.targetBed.x, t);
    rabbit.visualY = lerp(rabbit.previousBed.y, rabbit.targetBed.y, t);
    rabbit.jumpOffset = Math.sin(t * Math.PI) * 20; // arc height
}

function initiateJump(rabbit, targetBed) {
    rabbit.targetBed = targetBed;
    rabbit.animProgress = 0;
    rabbit.animDuration = 350 / rabbit.personality.speed;
    rabbit.state = 'jumping';
}

// --- Main Update (called each frame) ---
function updateRabbits(dt, rabbits, snake, allBeds, spawner, config) {
    // 1. Compute shared snake-distance map (once per frame)
    const snakeBed = snake.segments[0];
    const bedDistances = computeSnakeDistances(snakeBed, allBeds);

    // 2. Update each rabbit
    for (const rabbit of rabbits) {
        if (rabbit.state === 'jumping') {
            animateRabbit(rabbit, dt);
            continue;
        }

        if (rabbit.state === 'frozen') {
            rabbit.freezeTimer -= dt;
            if (rabbit.freezeTimer <= 0) {
                rabbit.state = 'idle';
                // Panic jump after unfreezing
                const panicTarget = selectTarget(rabbit, snake, rabbits, bedDistances, config);
                initiateJump(rabbit, panicTarget);
            }
            continue;
        }

        // Countdown to next jump
        rabbit.jumpTimer -= dt;
        if (rabbit.jumpTimer <= 0) {
            // Small chance to freeze if snake is very close
            const snakeDist = bedDistances.get(rabbit.currentBed);
            if (snakeDist <= 1 && Math.random() < 0.15) {
                rabbit.state = 'frozen';
                rabbit.freezeTimer = 800 + Math.random() * 500; // 0.8-1.3s
                continue;
            }

            const target = selectTarget(rabbit, snake, rabbits, bedDistances, config);
            if (target && target !== rabbit.currentBed) {
                initiateJump(rabbit, target);
            } else {
                rabbit.resetJumpTimer(); // no valid target, wait
            }
        }
    }

    // 3. Spawner
    spawner.update(dt, rabbits, allBeds, snake);
}

// --- Utility Functions ---
function randomRange(min, max) {
    return min + Math.random() * (max - min);
}

function easeInOutQuad(t) {
    return t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t;
}

function lerp(a, b, t) {
    return a + (b - a) * t;
}

function bfsDistance(bedA, bedB, allBeds) {
    // Short BFS between two beds (for spawning checks)
    if (bedA === bedB) return 0;
    const visited = new Set([bedA]);
    const queue = [{ bed: bedA, dist: 0 }];
    while (queue.length > 0) {
        const { bed, dist } = queue.shift();
        for (const n of getNeighbors(bed, allBeds)) {
            if (n === bedB) return dist + 1;
            if (!visited.has(n)) {
                visited.add(n);
                queue.push({ bed: n, dist: dist + 1 });
            }
        }
    }
    return 999;
}

// --- Render ---
function renderRabbit(ctx, rabbit) {
    const x = rabbit.visualX;
    const y = rabbit.visualY - (rabbit.jumpOffset || 0);

    // Body
    ctx.fillStyle = '#ffffff';
    ctx.fillRect(x - 8, y - 4, 16, 12);

    // Head
    ctx.fillRect(x + 4, y - 10, 10, 8);

    // Ears
    ctx.fillStyle = '#ffffff';
    ctx.fillRect(x + 2, y - 18, 3, 10);
    ctx.fillRect(x + 9, y - 18, 3, 10);

    // Inner ears
    ctx.fillStyle = '#ffaaaa';
    ctx.fillRect(x + 3, y - 16, 1, 6);
    ctx.fillRect(x + 10, y - 16, 1, 6);

    // Eye
    ctx.fillStyle = '#000000';
    ctx.fillRect(x + 10, y - 8, 2, 2);

    // Tail
    ctx.fillStyle = '#ffffff';
    ctx.fillRect(x - 10, y - 2, 4, 4);
}
```

---

## 6. Summary of Recommendations

### Pathfinding
- **Do NOT use Dijkstra** — all jumps cost the same, BFS is strictly better
- **Use BFS once per frame** from the snake position to compute danger distances for ALL rabbits
- **Use A\*** only if patrol/return-to-home behavior is needed (multi-hop navigation)
- **Default behavior: score adjacent beds** — no pathfinding needed for single-jump decisions

### Behavior
- **Default state:** Weighted roam (carrot affinity + randomness + avoid ping-pong)
- **Flee trigger:** Snake within `fleeDistance` tiles → score neighbors by distance-from-snake
- **Freeze state:** 15% chance when snake is adjacent → brief pause, then panic jump
- **Personality traits:** Boldness, skittishness, speed — assign randomly, vary behavior

### Implementation
- **One BFS per frame** from snake head → shared distance map
- **Timer per rabbit** with personality-adjusted intervals and random jitter
- **Lerp animation** with easeInOutQuad + parabolic arc for jump feel
- **State machine:** idle → jumping → idle, with frozen as interrupt

### Spawning
- **On carrot beds only**, away from snake and other rabbits
- **Respawn on delay** (5-10s) when count drops below minimum
- **Difficulty scales:** rabbit count, speed, flee intelligence, wrong-choice rate
- **Optional:** dynamic difficulty based on player catch rate

### Key Design Principle: Believable Imperfection
Rabbits should feel like they have survival instincts but are NOT optimal. The 10-25% wrong-choice rate and personality variance ensure the player can catch them with smart positioning. A perfectly optimal rabbit AI would be impossible to catch and ruin the game.

---

## Sources & References
- Red Blob Games — Introduction to A*: https://www.redblobgames.com/pathfinding/a-star/introduction.html
- Game AI Pro — "Behavior Trees for AI": behavior selection patterns for game agents
- "The Nature of Code" by Daniel Shiffman — steering behaviors and autonomous agents
- MDN — requestAnimationFrame: https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame
- RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels (SHALL/MUST/SHOULD/MAY)
