# Maze Generation Algorithms — Research for Carrot Maze

> **Project context:** Grid-based browser game (HTML5 Canvas). Snake hunts rabbits in a maze of carrot beds (грядки). Grid ~20×20 or 30×30. Need corridors + open pocket rooms.

---

## 1. Recursive Backtracking (DFS)

### How It Works
1. Start with a grid where every cell has all 4 walls.
2. Pick a random starting cell, mark it visited, push to stack.
3. While stack not empty:
   - Look at current cell (top of stack). Find all unvisited neighbors.
   - If unvisited neighbors exist: pick one randomly, remove the wall between current and chosen, mark chosen visited, push chosen.
   - If no unvisited neighbors: pop stack (backtrack).
4. Done when stack is empty — all cells reachable.

### Complexity
| Metric | Value |
|--------|-------|
| Time | O(N) where N = cells (each cell visited once, each wall checked once) |
| Space | O(N) for visited array + stack (worst case: stack holds entire path, ~N for a perfect snake) |

### Maze Characteristics
- **Long, winding corridors** with very few branches — the classic "perfect maze" look.
- High "river factor": solving the maze means following long passages with few decisions.
- No loops — a perfect spanning tree. Exactly one path between any two cells.
- Feels claustrophobic and exploratory. Great for dungeon crawlers.

### JS/Canvas Implementation Difficulty: ★★★☆☆ (Medium)
- Straightforward recursion/stack logic. ~50–70 lines.
- Stack-based iteration avoids call-stack overflow for 30×30 grids.
- Wall rendering requires storing 2 bits per cell (N/S/E/W walls) or a separate wall array.

### Suitability for Carrot Beds: ★★★★☆
- Long corridors are natural "connectors" between pockets.
- After generating, selectively widen groups of cells into rooms (see §5).
- The backbone is simple and predictable — easy to post-process.

---

## 2. Prim's Algorithm

### How It Works
1. Start with all cells having all 4 walls. All cells initially "unvisited."
2. Pick a random cell, mark it as part of the maze. Add all its walls to a "frontier" list.
3. While frontier not empty:
   - Pick a random wall from the frontier.
   - If exactly one of the two cells on either side is in the maze: remove the wall (connect them), mark the new cell as part of the maze, add all its walls to the frontier.
   - If both cells are already in the maze: discard the wall (it would create a loop).
4. Done when frontier is exhausted.

### Complexity
| Metric | Value |
|--------|-------|
| Time | O(N log N) if using a heap for random selection, effectively O(N) with array + random index |
| Space | O(N) for visited + frontier list |

### Maze Characteristics
- **Many short dead-ends**, more "branchy" than DFS.
- Corridors tend to be shorter; maze feels more open and organic.
- Still a perfect maze (no loops).
- More "choices" at intersections — less linear than DFS.

### JS/Canvas Implementation Difficulty: ★★★☆☆ (Medium)
- Frontier list management is straightforward. ~60–80 lines.
- Random wall selection from array by index is O(1).
- Must avoid adding duplicate walls to frontier (use a Set or check before push).

### Suitability for Carrot Beds: ★★★★☆
- Branchy structure means beds can sit naturally at dead-ends.
- More organic feel — corridors radiate outward rather than snake linearly.
- Post-processing to widen pockets is easy (similar to DFS).

---

## 3. Kruskal's Algorithm

### How It Works
1. Initialize: every cell is its own set (disjoint-set / union-find). All internal walls are in a list.
2. Shuffle the list of all walls randomly.
3. Iterate through the shuffled walls. For each wall:
   - If the two cells it separates belong to different sets: remove the wall (connect them) and union the two sets.
   - If they're already in the same set: skip (would create a loop).
4. Done when all cells are in one set (n-1 walls removed for n cells).

### Complexity
| Metric | Value |
|--------|-------|
| Time | O(N α(N)) with union-find + path compression ≈ O(N) |
| Space | O(N) for disjoint-set arrays + wall list |

### Maze Characteristics
- **Fair, unbiased** corridors — all possible perfect mazes are equally likely.
- Visually similar to Prim's but slightly less organic; more "checkerboard" feel.
- Many short dead-ends, moderate corridor length.
- The "fairest" algorithm — no structural bias.

### JS/Canvas Implementation Difficulty: ★★★★☆ (Slightly harder)
- Union-Find data structure adds ~30 lines of boilerplate.
- Wall list is large (≈2wh cells for a w×h grid, ~1800 for 30×30).
- Overall ~80–110 lines.

### Suitability for Carrot Beds: ★★★☆☆
- Unbiased structure doesn't naturally lend itself to pocket carving.
- No inherent "flow" or "zones" — harder to predict where rooms will form.
- Still workable, but post-processing feels less natural than with DFS/Prim's.

---

## 4. Eller's Algorithm

### How It Works
1. Process the grid row by row, top to bottom. Only need to keep the current row in memory.
2. For each row:
   - **Randomly join adjacent cells** in the same row that are in different sets.
   - **Randomly open vertical connections** downward: for each set in the row, pick at least one cell to open a downward passage (connect to next row).
   - **Propagate sets** down. Cells without a vertical connection start new singleton sets.
3. On the final row: join all remaining disjoint sets (no downward connections possible).

### Complexity
| Metric | Value |
|--------|-------|
| Time | O(N) — each cell processed once per row, constant work per cell |
| Space | **O(W)** where W = width — only need current row state! (vs O(WH) for others) |

### Maze Characteristics
- **Horizontal bias** — corridors tend to run left-right more than up-down.
- Moderate dead-end count.
- Perfect maze, no loops.
- Feels "row-oriented" — less organic than Prim's, more predictable than DFS.

### JS/Canvas Implementation Difficulty: ★★★★☆ (Slightly harder)
- Row-by-row logic is conceptually trickier than global algorithms.
- Set management per row requires careful bookkeeping.
- ~80–100 lines. Harder to debug visually mid-generation.
- **Key advantage:** can generate arbitrarily tall mazes in O(W) memory.

### Suitability for Carrot Beds: ★★☆☆☆
- Horizontal bias may look awkward for a game where the snake moves in all directions.
- Row-by-row generation means pocket carving must happen after the full maze is built.
- Not the best fit for a symmetric, organic-looking playing field.

---

## 5. Carving Rooms / Pockets ("Carrot Beds") Into Mazes

A "carrot bed" is an open pocket — a 2×2, 3×3, or irregular group of cells with all internal walls removed, connected to the maze.

### Method A: Post-Generation Carving
After generating a perfect maze:

1. **Select seed cells randomly** — choose N cells scattered across the grid (e.g., 6–12 for a 20×20 grid).
2. **Grow each seed into a room:**
   - For each seed, attempt to expand to adjacent cells (up to a max room size like 3×3 or 4 cells).
   - Remove all internal walls within the room boundary.
   - Ensure rooms don't overlap.
3. **Reconnect:** removing walls creates disconnected components. For each room, ensure at least one external wall is removed so the room connects to the corridor network.
4. **Validate:** BFS from any cell — all cells must be reachable.

```js
// Pseudocode: post-generation room carving
function carveRooms(maze, gridW, gridH, numRooms, maxRoomSize) {
  const rooms = [];
  const occupied = new Set();

  for (let i = 0; i < numRooms; i++) {
    // Find a random seed not in any existing room
    let sx, sy;
    do {
      sx = randInt(1, gridW - maxRoomSize - 1);
      sy = randInt(1, gridH - maxRoomSize - 1);
    } while (occupied.has(`${sx},${sy}`));

    // Grow a rectangular room
    const rw = randInt(2, maxRoomSize);
    const rh = randInt(2, maxRoomSize);

    // Mark cells and remove internal walls
    for (let y = sy; y < sy + rh && y < gridH; y++) {
      for (let x = sx; x < sx + rw && x < gridW; x++) {
        occupied.add(`${x},${y}`);
        rooms.push({x, y});
      }
    }
    // Remove all internal walls within the room bounds
    removeInternalWalls(maze, sx, sy, rw, rh);
  }

  // Ensure connectivity: punch at least one wall from each room
  for (const room of rooms) {
    ensureConnected(maze, room);
  }
}
```

### Method B: Generation-Time Integration
Integrate room placement into the generation algorithm itself:

1. Before generating, mark N regions as "room zones" (e.g., 3×3 blocks).
2. During generation (e.g., in DFS/Prim's):
   - When the algorithm reaches a room zone, remove ALL internal walls within that zone.
   - Continue generating corridors from the room's perimeter cells.
3. This produces more organic transitions between corridors and rooms.

### Method C: Cellular Automata Smoothing (for organic pockets)
1. Generate standard maze.
2. Apply cellular automata rules to widen corridors into organic pockets:
   - Rule: if a wall cell has ≥3 open neighbors, remove the wall.
   - Repeat 2–4 iterations.
3. This creates organic, cave-like openings. Great for "carrot beds" that feel natural.
4. Downside: harder to control exact room shape/size.

---

## 6. Comparative Summary Table

| Criterion | DFS (Recursive Backtracking) | Prim's | Kruskal's | Eller's |
|---|---|---|---|---|
| **Time Complexity** | O(N) | O(N log N) / O(N) | O(N α(N)) | O(N) |
| **Space Complexity** | O(N) | O(N) | O(N) | **O(W)** ← best |
| **Corridor Style** | Long, winding, few branches | Short, many dead-ends, branchy | Fair, unbiased, many dead-ends | Horizontal bias, moderate |
| **Organic Feel** | Linear, dungeon-like | Very organic | Somewhat organic | Grid-like, row-oriented |
| **JS Impl. Difficulty** | ★★★☆☆ Medium | ★★★☆☆ Medium | ★★★★☆ Harder (UF) | ★★★★☆ Harder (row logic) |
| **Lines of JS (approx)** | 50–70 | 60–80 | 80–110 | 80–100 |
| **Infinite/Memory-constrained** | No | No | No | **Yes** (single row) |
| **Carrot Bed Suitability** | ★★★★☆ | ★★★★☆ | ★★★☆☆ | ★★☆☆☆ |
| **Post-processing Ease** | Easy | Easy | Moderate | Moderate |
| **Visual Appeal for Game** | Excellent corridors | Organic, game-like | Neutral | Less natural |

---

## 7. Recommendation

### Top Pick: **Recursive Backtracking (DFS)**
- Best corridor flow for a snake-chase game — long, clear paths the snake travels through.
- Easiest to implement and debug.
- Carrot beds carved as rooms create natural "zones" connected by long corridors.
- Most widely used in game jam / indie games for a reason.

### Runner-Up: **Prim's Algorithm**
- More organic, branchy feel — more interesting exploration.
- Dead-ends make natural hiding spots for rabbits.
- Slightly more complex frontier management, but still very manageable.

---

## 8. JS Pseudocode — Top 2 Candidates

### 8.1 Recursive Backtracking (DFS) — Iterative Stack

```js
/**
 * Generate a perfect maze using Recursive Backtracking (iterative stack).
 * @param {number} w - Grid width in cells
 * @param {number} h - Grid height in cells
 * @returns {object} Maze representation: cells[y][x] = { n, s, e, w } wall booleans
 */
function generateMazeDFS(w, h) {
  // Initialize: all walls present
  const cells = [];
  for (let y = 0; y < h; y++) {
    cells[y] = [];
    for (let x = 0; x < w; x++) {
      cells[y][x] = { n: true, s: true, e: true, w: true, visited: false };
    }
  }

  const stack = [];
  const startX = randInt(0, w - 1);
  const startY = randInt(0, h - 1);
  cells[startY][startX].visited = true;
  stack.push({ x: startX, y: startY });

  // Direction helpers: dx, dy, wall-to-remove, opposite-wall
  const dirs = [
    { dx:  0, dy: -1, wall: 'n', opp: 's' },
    { dx:  1, dy:  0, wall: 'e', opp: 'w' },
    { dx:  0, dy:  1, wall: 's', opp: 'n' },
    { dx: -1, dy:  0, wall: 'w', opp: 'e' },
  ];

  while (stack.length > 0) {
    const current = stack[stack.length - 1];
    const { x, y } = current;

    // Find unvisited neighbors
    const neighbors = [];
    for (const d of dirs) {
      const nx = x + d.dx;
      const ny = y + d.dy;
      if (nx >= 0 && nx < w && ny >= 0 && ny < h && !cells[ny][nx].visited) {
        neighbors.push({ nx, ny, ...d });
      }
    }

    if (neighbors.length > 0) {
      // Pick random unvisited neighbor
      const next = neighbors[randInt(0, neighbors.length - 1)];
      // Remove wall between current and next
      cells[y][x][next.wall] = false;
      cells[next.ny][next.nx][next.opp] = false;
      // Visit and push
      cells[next.ny][next.nx].visited = true;
      stack.push({ x: next.nx, y: next.ny });
    } else {
      // Backtrack
      stack.pop();
    }
  }

  // Clean up visited flags (optional — can leave for debugging)
  return cells;
}

function randInt(min, max) {
  return Math.floor(Math.random() * (max - min + 1)) + min;
}
```

### 8.2 Prim's Algorithm

```js
/**
 * Generate a perfect maze using Prim's algorithm.
 * @param {number} w - Grid width in cells
 * @param {number} h - Grid height in cells
 * @returns {object} Maze representation: cells[y][x] = { n, s, e, w } wall booleans
 */
function generateMazePrim(w, h) {
  // Initialize: all walls present, all cells unvisited
  const cells = [];
  for (let y = 0; y < h; y++) {
    cells[y] = [];
    for (let x = 0; x < w; x++) {
      cells[y][x] = { n: true, s: true, e: true, w: true, inMaze: false };
    }
  }

  // Pick random starting cell
  const startX = randInt(0, w - 1);
  const startY = randInt(0, h - 1);
  cells[startY][startX].inMaze = true;

  // Frontier: list of walls as { x, y, dir } where (x,y) is the cell IN the maze
  const frontier = [];
  addWallsToFrontier(frontier, startX, startY, w, h);

  while (frontier.length > 0) {
    // Pick random wall from frontier
    const idx = randInt(0, frontier.length - 1);
    const wall = frontier[idx];
    // Remove from frontier (swap with last + pop for O(1))
    frontier[idx] = frontier[frontier.length - 1];
    frontier.pop();

    const { x, y, dir } = wall;
    const nx = x + dir.dx;
    const ny = y + dir.dy;

    // If the neighbor is NOT in the maze, connect them
    if (nx >= 0 && nx < w && ny >= 0 && ny < h && !cells[ny][nx].inMaze) {
      // Remove wall
      cells[y][x][dir.wall] = false;
      cells[ny][nx][dir.opp] = false;
      // Mark as part of maze
      cells[ny][nx].inMaze = true;
      // Add its walls to frontier
      addWallsToFrontier(frontier, nx, ny, w, h);
    }
    // If both cells are already in the maze, wall is discarded (prevents loops)
  }

  return cells;
}

const DIRS = [
  { dx:  0, dy: -1, wall: 'n', opp: 's' },
  { dx:  1, dy:  0, wall: 'e', opp: 'w' },
  { dx:  0, dy:  1, wall: 's', opp: 'n' },
  { dx: -1, dy:  0, wall: 'w', opp: 'e' },
];

function addWallsToFrontier(frontier, x, y, w, h) {
  for (const d of DIRS) {
    const nx = x + d.dx;
    const ny = y + d.dy;
    if (nx >= 0 && nx < w && ny >= 0 && ny < h) {
      frontier.push({ x, y, dir: d });
    }
  }
}
```

---

## 9. Practical Notes for Carrot Maze Implementation

### Grid Sizing
- For a 20×20 grid rendered on a 600×600px Canvas: each cell = 30px.
- Carrot beds should be 2×2 or 3×3 cells (60–90px) — visually distinct.
- Corridors remain 1 cell wide for tight gameplay.

### Rendering Strategy
```js
// Render maze on Canvas
function renderMaze(ctx, cells, cellSize) {
  ctx.strokeStyle = '#2d5a1e'; // dark green walls
  ctx.lineWidth = 2;
  const w = cells[0].length;
  const h = cells.length;

  for (let y = 0; y < h; y++) {
    for (let x = 0; x < w; x++) {
      const px = x * cellSize;
      const py = y * cellSize;
      const c = cells[y][x];
      if (c.n) drawLine(ctx, px, py, px + cellSize, py);
      if (c.s) drawLine(ctx, px, py + cellSize, px + cellSize, py + cellSize);
      if (c.w) drawLine(ctx, px, py, px, py + cellSize);
      if (c.e) drawLine(ctx, px + cellSize, py, px + cellSize, py + cellSize);
    }
  }
}
```

### Room Carving Integration
After maze generation, carve rooms and **re-validate connectivity**. Use a flood-fill BFS: if any cell is unreachable, punch an additional wall connecting the isolated region.

### Snake Movement
Snake moves cell-by-cell on the grid. At each step, check the wall between current cell and target cell. If a wall exists → collision (death). This is the key game mechanic — walls are the movement constraint.

---

## Sources & References

- Buck, J. (2015). *Mazes for Programmers*. The Pragmatic Bookshelf. — definitive reference.
- Wikipedia: Maze Generation Algorithm (covers all 4 algorithms in detail).
- Jamis Buck's blog: weblog.jamisbuck.org — excellent visual walkthroughs of each algorithm.
- Red Blob Games: *Grid parts and relationships* — useful for grid coordinate handling in Canvas.
