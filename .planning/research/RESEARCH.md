# RESEARCH: Carrot Maze — Game Design & Tech Stack

> **Parent issue:** MUL-16 — «Питон охотится на зайцев в лабиринте из грядок»
> **Researcher:** GSD Researcher (MUL-18)
> **Date:** 2026-05-29

---

## Executive Summary

**Tech Stack подтверждён:** HTML5 Canvas + vanilla JavaScript — оптимальный выбор для браузерной игры без сервера, без зависимостей, без сборки. Открывается по двойному клику в любом современном браузере.

**Архитектурные рекомендации:**
- **Maze:** Recursive Backtracking (DFS) + post-generation room carving для carrot beds
- **Game Loop:** Fixed timestep (10 Hz) на requestAnimationFrame
- **Snake:** Array-based body, direction queue с защитой от 180° разворота
- **Rabbit AI:** State machine (ROAM / FLEE / FREEZE) + shared BFS distance map
- **Visual:** Dark earthy palette, pixel-art рендеринг, HTML overlay для UI
- **Grid:** 30×20 cells × 20px = 600×400px canvas

---

## 1. Maze Generation (Лабиринт из грядок)

### 1.1 Сравнение алгоритмов

| Алгоритм | Стиль коридоров | Сложность JS | Подходит для грядок | Характер |
|---|---|---|---|---|
| **Recursive Backtracking (DFS)** | Длинные, извилистые, мало развилок | ★★★☆☆ (50-70 строк) | ★★★★☆ | «Данжен-кроулер» |
| **Prim's Algorithm** | Короткие, много тупиков | ★★★☆☆ (60-80 строк) | ★★★★☆ | Органичный, естественный |
| **Kruskal's** | Справедливые, много тупиков | ★★★★☆ (80-110 строк) | ★★★☆☆ | Нейтральный |
| **Eller's** | Горизонтальный уклон | ★★★★☆ (80-100 строк) | ★★☆☆☆ | Row-oriented |

Все алгоритмы генерируют **perfect maze** (ровно один путь между любыми двумя клетками, нет циклов). Сложность: O(N) по времени, O(N) по памяти (кроме Eller's: O(W)).

### 1.2 Рекомендация: Recursive Backtracking (DFS)

**Почему DFS:**
- Длинные коридоры — идеальный «поток» для chase-геймплея: питон движется по естественным маршрутам, зайцы прячутся в тупиках/грядках
- Простейшая отладка — итеративный стек, не нужно Union-Find или frontier-менеджмент
- Легко post-process: после генерации вырезаем комнаты-грядки, алгоритм не «сопротивляется»
- 50-70 строк JS — полностью контролируемый код

**Runner-up — Prim's** (на случай если захочется более «органичного» лабиринта с большим количеством тупиков-укрытий для зайцев).

### 1.3 Carrot Beds: Post-Generation Room Carving

После генерации perfect maze вырезаем 6-10 «грядок» — комнат 2×2 или 3×3 клетки с удалёнными внутренними стенами:

1. Random seed cells (разбросаны по полю, не вплотную к краям)
2. Grow room (2×2 – 3×3 клетки, не пересекаются)
3. Remove ALL internal walls внутри комнаты
4. Ensure connectivity (хотя бы одна внешняя стена удалена)
5. Validate BFS (все клетки достижимы)

Грядки помечаются особым типом клетки — на них рендерятся морковки и спавнятся зайцы.

### 1.4 Параметры сетки

| Параметр | Значение | Обоснование |
|---|---|---|
| Grid size | 30×20 cells | Достаточно для maze+комнат, 600 клеток — manageable |
| Cell size | 20px | Чёткий pixel-art размер, кратен 600 |
| Canvas | 600×400px | Компактный, но просторный |
| Carrot beds | 6-10 шт. | ~10-16% площади под комнатами |
| Bed size | 2×2 – 3×3 cells | Достаточно для спавна 1-2 зайцев на грядку |

---

## 2. Snake Game Mechanics (Питон на Canvas)

### 2.1 Game Loop: Fixed Timestep

**Рекомендация:** Fixed timestep (10 Hz) — детерминированный, grid-safe, простой.

```
requestAnimationFrame → accumulator → while (acc >= TICK): update() → render()
```

- **TICK_RATE = 10** (100ms на шаг) — комфортная скорость для chase-геймплея
- При поедании зайцев скорость может расти: 10 → 12 → 15 Hz (нарастающая сложность)
- Accumulator cap = 3×TICK_DURATION (защита от «спирали смерти»)
- Рендер каждый кадр (~60 FPS), логика — каждый тик (10 Hz)

**Variable timestep — НЕ рекомендуется** для grid-based игр: недетерминирован, змейка может проваливаться между клеток.

### 2.2 Snake Body: Plain Array

```js
const snake = [
  {x: 5, y: 10}, // head (index 0)
  {x: 4, y: 10}, // body
  {x: 3, y: 10}, // tail (index length-1)
];
```

- **Array vs Linked List:** Для 30×20 (600 клеток) и даже для полной длины 600 — array.shift() на 10 Hz не создаёт заметной задержки. Array проще, не нужен свой deque.
- **Growth:** `growthQueue` счётчик — змейка не растёт мгновенно, а добавленные сегменты «выезжают» из хвоста с задержкой.

### 2.3 Movement & Input

**Direction queue (FIFO, max 3):**

- Стрелки + WASD
- Защита от 180°: проверка против `lastQueuedDir`, а не `currentDir`
- Отбрасывание дубликатов подряд

### 2.4 Collision Detection

| Тип коллизии | Реализация |
|---|---|
| **Wall** | `head.x < 0 || head.x >= COLS || head.y < 0 || head.y >= ROWS` |
| **Self** | Линейный проход `snake.slice(1)` — для ≤600 клеток O(n) достаточно |
| **Rabbit** | `head.x === rabbit.x && head.y === rabbit.y` |

Порядок за тик: Move head → Wall check → Self check → Rabbit check → Move tail.

### 2.5 Rendering

- Offscreen background canvas (стены/грядки — prerender один раз)
- Main canvas: `ctx.drawImage(bgCanvas) + draw snake + draw rabbits + HUD`
- `imageSmoothingEnabled = false`; `Math.floor()` координаты

---

## 3. Rabbit AI (Зайцы между грядок)

### 3.1 State Machine

```
ROAM (Snake далеко: > 4 клеток)
  → Weighted random walk между грядками
  ↓ snake enters danger zone (2-4 cells)
FLEE
  → Выбор соседней грядки с max distance от змеи, 10-25% suboptimal
  ↓ snake adjacent (1 cell)
FREEZE
  → 15% шанс замереть — лёгкая добыча
```

### 3.2 Shared BFS Distance Map

**Один BFS от змеи на ВЕСЬ граф грядок за кадр** — не отдельный поиск для каждого зайца.

- BFS: O(V+E), V ≤ 30 (количество грядок), E ≤ 4V — тривиально
- Зайцы в ROAM: weighted random (carrot affinity + anti-ping-pong + crowd avoidance)

### 3.3 Параметры

| Параметр | Значение |
|---|---|
| Jump interval | 2-4s (+ jitter ±0.5s) |
| Danger radius | 2-3 cells |
| Freeze chance | 15% при соседстве |
| Suboptimal flee | 15-25% |
| Min rabbits | 3 |
| Respawn delay | 5-10s |

### 3.4 Why Imperfect AI?

Идеальный AI делает игру нефан. Suboptimal flee + freeze chance = заяц, который «почти спасся, но замер». Игрок должен чувствовать, что ОН переиграл зайца.

---

## 4. Visual Design

### 4.1 Color Palette

```
Page bg:      #1a1a2e   Deep navy-black
Floor:        #2d5a1e   Dark green
Wall:         #1b3a0f   Very dark green
Bed (soil):   #4a3520   Dark brown
Carrot:       #f7931e   Bright orange
Snake head:   #00e676   Bright green (+ 1px #1b3a0f outline)
Snake body:   #4caf50   Medium green
Rabbit:       #ffffff   White (+ #ffcdd2 pink ears)
Score:        #ffd54f   Warm yellow
Overlay:      rgba(0,0,0,0.75)
```

⚠️ Контраст snake vs floor ≈ 3.8:1 — нужна 1px тёмная обводка.

### 4.2 Pixel-Art Rendering

```css
canvas { image-rendering: pixelated; image-rendering: crisp-edges; }
```
```js
ctx.imageSmoothingEnabled = false;
```

### 4.3 Visual Polish (по приоритету)

| Эффект | Приоритет |
|---|---|
| Rabbit jump animation (lerp + parabolic arc, 200-300ms) | High |
| Particle burst при поедании (10-15 частиц, 300ms fade) | Medium |
| Score popup "+10" | Medium |
| Screen shake (2-4 кадра) | Medium |
| Carrot idle wobble | Low |
| Snake body fade (голова ярче хвоста) | Low |

### 4.4 UI Layer

- **Canvas HUD:** счёт в левом верхнем углу (monospace, полупрозрачная подложка)
- **HTML Overlay:** game over с Press Start 2P, кнопка Restart

### 4.5 Typography

- **Заголовки/кнопки:** Press Start 2P (Google Fonts, OFL)
- **Счёт на Canvas:** 14px monospace bold

---

## 5. Best Practices

### 5.1 Game Loop

```js
const TICK_RATE = 10;
const TICK_DURATION = 1000 / TICK_RATE;
let lastTime = 0, accumulator = 0;

function gameLoop(timestamp) {
  requestAnimationFrame(gameLoop);
  const delta = timestamp - lastTime;
  lastTime = timestamp;
  accumulator += delta;
  if (accumulator > TICK_DURATION * 3) accumulator = TICK_DURATION * 3;
  while (accumulator >= TICK_DURATION) {
    update();
    accumulator -= TICK_DURATION;
  }
  render(accumulator / TICK_DURATION);
}
requestAnimationFrame(gameLoop);
```

### 5.2 Game States

`IDLE → PLAYING → DEAD`

### 5.3 Code Organization

```
carrot-maze/
├── index.html    # Canvas + UI overlay + <script src="game.js">
├── game.js       # Вся игровая логика
└── README.md
```

Без ES modules (file:// не поддерживает CORS для `type="module"`).

### 5.4 Scoring & Difficulty

- +10 за зайца
- Speed ramp: каждые 5 зайцев → +1 Hz к TICK_RATE (10→15 Hz max)

---

## 6. Технические риски

| Риск | Митигация |
|---|---|
| Maze disconnected после carving | BFS-валидация + auto-reconnect |
| CORS блокирует шрифты при file:// | System monospace fallback |
| Разная частота rAF на 144Hz | Fixed timestep нормализует |

---

## 7. Подтверждение техстека

| Критерий | Canvas + Vanilla JS | Phaser.js | React + Canvas |
|---|---|---|---|
| Размер | 0 KB | ~1 MB | ~200 KB + React |
| Запуск | Двойной клик | Нужен сервер | Нужен бандлер |
| Сложность | Низкая (600-800 строк) | Средняя | Высокая |
| Pixel-art | Полный контроль | Абстракция | Контроль |

**Вердикт:** HTML5 Canvas + Vanilla JS — безальтернативный выбор.

---

## 8. Итоговые рекомендации для Planner (MUL-19)

1. **Maze:** DFS с room carving (6-10 грядок 2×2–3×3)
2. **Grid:** 30×20 × 20px = 600×400px Canvas
3. **Loop:** Fixed timestep 10 Hz на rAF
4. **Code:** `index.html` + `game.js`, без зависимостей
5. **Rabbit AI:** State machine + shared BFS
6. **Visual:** Dark palette + pixel-art + HTML overlay
7. **States:** IDLE → PLAYING → DEAD
8. **Phases:** MVP core loop → visual polish

---

## 9. References

- Maze Generation: https://weblog.jamisbuck.org/2010/12/27/maze-generation-recursive-backtracking
- Fix Your Timestep: https://gafferongames.com/post/fix_your_timestep/
- requestAnimationFrame: https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame
- CSS image-rendering: https://developer.mozilla.org/en-US/docs/Web/CSS/image-rendering
- Press Start 2P: https://fonts.google.com/specimen/Press+Start+2P

---

_Детальные под-документы:_
- `.planning/research/maze-algorithms.md` — алгоритмы лабиринтов (17 KB)
- `docs/snake-mechanics-research.md` — механика змейки (23 KB)
- `docs/rabbit-ai-research.md` — AI зайцев
- `docs/visual-design-research.md` — визуальный дизайн (25 KB)
