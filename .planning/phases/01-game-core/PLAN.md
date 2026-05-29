# PLAN: Phase 01 — Game Core

> **Parent:** MUL-16 — Carrot Maze Game
> **Planner:** GSD Planner (MUL-19)
> **Based on:** RESEARCH.md — GSD Researcher (2026-05-29)
> **Date:** 2026-05-29

---

## Objective

Реализовать играбельное ядро Carrot Maze: лабиринт с грядками, питона (змейку) и зайцев с AI. На выходе — один HTML-файл, открывающийся в браузере, с полным игровым циклом (IDLE → PLAYING → DEAD), счётом и рестартом.

## Tech Stack (подтверждено Research)

| Параметр | Значение |
|---|---|
| Язык | Vanilla JavaScript (no ES modules) |
| Рендеринг | HTML5 Canvas, 600×400px |
| Grid | 30×20 cells × 20px |
| Game Loop | Fixed timestep 10 Hz на requestAnimationFrame |
| Maze | Recursive Backtracking (DFS) + room carving |
| Rabbit AI | State machine (ROAM / FLEE / FREEZE) + shared BFS |
| Code | `index.html` + `game.js` (один файл через `<script>`) |
| Зависимости | Ноль (чистый браузер, открывается file://) |

---

## Tasks

### Task 1: Maze + Canvas Setup

**Files:** `index.html`, `game.js` (CONSTANTS, GameState, Maze, Renderer)

**Action:**
1. Создать `index.html`: Canvas 600×400 `#game-canvas`, скрытый `#game-over` div с кнопкой Restart, загрузка Press Start 2P
2. DFS maze generator (итеративный стек): 30×20 клеток, все стены подняты
3. Room carving: 6–10 грядок 2×2–3×3, не у краёв, без пересечений, удаление внутренних стен
4. BFS-валидация связности; при disconnect — повторить carving
5. Prerender фона на offscreen canvas: пол, стены, грядки, морковки (3-5 на грядку)
6. Основной `draw()`: фон + змейка + зайцы + HUD

**Verify:** Canvas отображается, лабиринт с грядками рендерится, BFS подтверждает связность.

**Done:** Maze рендерится на Canvas; все клетки достижимы; грядки с морковками видны.

---

### Task 2: Snake — Движение и коллизии

**Files:** `game.js` (Snake, Input, Collision)

**Action:**
1. Snake body: `[{x,y},...]` (head = index 0), старт в центре лабиринта
2. Direction queue: FIFO до 3, защита от 180° (против `lastQueuedDir`), отбрасывание дубликатов
3. Keyboard: стрелки + WASD, `preventDefault()` на стрелки
4. Movement: `unshift` головы → `pop` хвоста; `growthQueue > 0` → хвост не удаляется, `growthQueue--`
5. Collision checks: Move head → Wall → Self → Rabbit → Move tail
6. Wall: `head.x < 0 || head.x >= COLS || head.y < 0 || head.y >= ROWS` + maze grid проверка
7. Self: линейный проход `body.slice(1)`
8. Death → DEAD state

**Verify:** Змейка движется (10 Hz), не разворачивается на 180°, умирает о стены/себя, растёт.

**Done:** Змейка управляется с клавиатуры, движется по лабиринту, умирает о стены и себя, корректно растёт.

---

### Task 3: Rabbits + Game Loop + Game States

**Files:** `game.js` (GameLoop, Rabbit, BFS, GameStates, Scoring, Restart)

**Action:**
1. **Game Loop:** Fixed timestep: accumulator → while (acc >= TICK_DURATION) → update(); render() каждый кадр с alpha
2. **Game States:** IDLE (press ENTER) → PLAYING → DEAD (overlay + score + restart)
3. **Rabbit State Machine:**
   - ROAM (snake > 4 cells): weighted random walk между грядками
   - FLEE (snake 2–4 cells): max distance грядка от змеи; 15–25% suboptimal
   - FREEZE (snake ≤ 1 cell): 15% шанс замереть
4. **Shared BFS:** Один BFS от змеи на граф грядок (≤30 узлов) за кадр
5. **Rabbit timers:** 2–4s + jitter ±0.5s; lerp-анимация с easeInOutQuad + parabolic arc (200-300ms)
6. **Spawning:** Min 3 зайца; начальный спавн на грядках вдали от змеи; respawn 5–10s
7. **Scoring:** +10 за зайца; speed ramp: каждые 5 зайцев → +1 Hz (10→15 max)
8. **Rabbit eating:** `growthQueue++`, score += 10, удаление зайца, particle burst (10–15 частиц, 300ms)
9. **Visual polish:** Jump animation, particles, score popup "+10", snake body fade, screen shake (2-4 кадра)
10. **Restart:** Новый лабиринт, змейка в центре, счёт 0, зайцы респавн

**Verify:** Полный цикл IDLE→PLAYING→DEAD→Restart; зайцы прыгают, flee от питона, freeze иногда; +10 очков; счёт; рестарт; speed ramp.

**Done:** Полный играбельный цикл: IDLE → PLAYING → DEAD → Restart.

---

## Dependency Graph

```
Task 1 (Maze + Canvas)
  └── Task 2 (Snake) — зависит от Canvas и лабиринта
        └── Task 3 (Rabbits + Loop + States) — зависит от змейки и поля
```

Все таски последовательные. Task 1 → Task 2 → Task 3.

---

## Success Criteria

| # | Критерий | Измеримость |
|---|---|---|
| SC1 | Лабиринт на Canvas 600×400 | Визуально: стены, коридоры, грядки |
| SC2 | Все клетки пола достижимы | BFS visited.size === total floor cells |
| SC3 | Змейка движется, 10 Hz | Плавное движение, нет двойных срабатываний |
| SC4 | Змейка умирает о стены и себя | Game Over при wall/self collision |
| SC5 | ≥3 зайцев на поле | Min 3 зайца всегда |
| SC6 | Зайцы прыгают с анимацией | lerp + parabolic arc визуально |
| SC7 | Зайцы flee от питона (imperfect) | Убегают, но 15-25% suboptimal |
| SC8 | +10 очков за зайца, счёт | Score обновляется при поедании |
| SC9 | Рестарт работает | Новый лабиринт, счёт 0, зайцы респавн |
| SC10 | Speed ramp каждые 5 зайцев | TICK_RATE растёт 10→15 Hz |

---

## File Structure (после фазы)

```
carrot-maze/
├── index.html       # Canvas + UI overlay
├── game.js          # Вся логика (~600-800 строк)
└── .planning/
    ├── REQUIREMENTS.md
    ├── ROADMAP.md
    └── phases/01-game-core/
        ├── CONTEXT.md
        └── PLAN.md
```

---

## Notes for Executor

1. **NO ES modules** — `file://` не поддерживает CORS для `type="module"`. Весь код в одном `game.js`.
2. **Offscreen canvas для фона** — prerender maze один раз.
3. **Integer coordinates** — `Math.floor()`, `imageSmoothingEnabled = false`.
4. **Порядок коллизий:** Move head → Wall → Self → Rabbit → Move tail.
5. **BFS на графе грядок** — ≤30 узлов, не 600 клеток.
6. **Suboptimal flee — ключевая фича:** 15-25% шанс неверного направления.
7. **Palette из RESEARCH.md:** тёмно-зелёный пол `#2d5a1e`, стены `#1b3a0f`, грядки `#4a3520`, морковки `#f7931e`, змея `#00e676`/`#4caf50`, зайцы `#ffffff`.
