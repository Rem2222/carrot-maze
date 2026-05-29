# Requirements: Carrot Maze

> Живой документ. Обновляется по мере изменения проекта (Spec Sync).

## R1: Game Field & Maze Generation

- R1.1: The game SHALL render a grid-based maze on an HTML5 Canvas (30×20 cells, 20px each).
- R1.2: The maze SHALL be generated procedurally using Recursive Backtracking (DFS) algorithm.
- R1.3: After generation, 6-10 carrot beds (rooms 2×2–3×3) SHALL be carved by removing internal walls.
- R1.4: BFS connectivity check SHALL ensure all beds are reachable from the snake's start position.
- R1.5: Walls, floors, and beds SHALL have distinct visual rendering with earthy pixel-art palette.
- R1.6: The canvas SHALL use `image-rendering: pixelated / crisp-edges` for sharp pixel display.

## R2: Python Snake

- R2.1: The player SHALL control a snake (python) using arrow keys or WASD.
- R2.2: The snake SHALL grow longer with each rabbit eaten (+1 segment per rabbit).
- R2.3: The snake SHALL die if it hits a wall or itself.
- R2.4: The snake SHALL start in a safe position with at least 2 non-fatal movement directions.
- R2.5: Initial snake movement direction SHALL be RIGHT; LEFT at start SHALL be blocked to prevent immediate self-collision.
- R2.6: Speed SHALL increase with score: 125ms at 0-4 rabbits eaten, 100ms at 5-9, 67ms at 10+.
- R2.7: A 180-degree reversal (e.g. RIGHT->LEFT) SHALL be blocked via direction queue.

## R3: Rabbits (Зайцы)

- R3.1: Rabbits SHALL spawn on carrot beds.
- R3.2: Rabbits SHALL hop between adjacent beds at random intervals.
- R3.3: Each rabbit eaten by the snake SHALL award 10 points.
- R3.4: At least 3 rabbits SHALL be present on the field at all times.
- R3.5: When a rabbit is eaten, a new rabbit SHALL spawn on a random bed after a short delay.
- R3.6: Rabbits SHALL be visually distinct (ear shapes, white/brown colors).

## R4: User Experience & Visual

- R4.1: The game SHALL display current score on the canvas HUD (top-left, monospace 20px).
- R4.2: High score SHALL be stored in `localStorage` and displayed on the game-over screen.
- R4.3: The game SHALL show a "GAME OVER" overlay with final score and restart button on death.
- R4.4: Restart SHALL be available via button click, Enter, or Space key.
- R4.5: A title screen SHALL display on first load ("CARROT MAZE" + "PRESS SPACE TO START").
- R4.6: Screen shake effect SHALL play on snake death.
- R4.7: Particle effects SHALL play when eating a rabbit.
- R4.8: CRT scanline overlay SHALL be rendered via CSS pseudo-element.
- R4.9: The snake body SHALL fade from head to tail (dark green to light green).
- R4.10: Sprites SHALL have 1px dark outlines for contrast on dark backgrounds.
- R4.11: All interactive elements SHALL meet WCAG AA contrast (>=4.5:1).
- R4.12: Carrots SHALL be rendered with visible green tops on orange bodies.

## R5: Technical Constraints

- R5.1: Zero external dependencies — vanilla JS only.
- R5.2: Single `index.html` file — opens directly in any modern browser.
- R5.3: Game loop SHALL use `requestAnimationFrame` with fixed-timestep accumulator.
- R5.4: Code SHALL be minified and self-contained in one file for deployment.
