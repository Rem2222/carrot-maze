# 🐍 Carrot Maze

**Процедурно-генерируемый лабиринт.** Управляйте змейкой, охотящейся на зайцев в лабиринте с морковными грядками. Браузерная игра на чистом Canvas.

**Играть:** https://rem2222.github.io/carrot-maze/

---

## 🎮 Как играть

- **WASD / Стрелки** — управление змейкой
- **Цель:** съедать зайцев на морковных грядках
- **Внимание:** змейка не может врезаться в стены или в саму себя
- **Скорость** растёт с каждым съеденным зайцем
- **Высший счёт** сохраняется в localStorage

## 🧱 Технический стек

- **Язык:** Чистый JavaScript (ES6+), HTML5, CSS3
- **Рендеринг:** Canvas 2D — 600×600 пикселей
- **Сборка:** Один `index.html` — без фреймворков и зависимостей
- **Деплой:** GitHub Pages (ветка `gh-pages`)

## ✨ Особенности

| Особенность | Описание |
|---|---|
| **Процедурная генерация лабиринта** | DFS + room carving — уникальный лабиринт в каждой игре |
| **Snake-механика** | WASD/стрелки, self-collision, ускорение |
| **Bunny AI** | flee/roam с личностями (смелость, пугливость, скорость) |
| **Частицы + Screen shake** | Визуальная обратная связь при событиях |
| **High Score** | Сохраняется в localStorage |
| **Pixel-art стиль** | `image-rendering: pixelated`, fading tail, glow |

## 🏗 Архитектура

Фиксированный игровой тик (10 Гц) + `requestAnimationFrame` для плавного рендеринга.

```
Game Loop (requestAnimationFrame)
├── update()
│   ├── updateSnake()        — движение, коллизии
│   ├── updateRabbits()      — Bunny AI (таймеры, цели, анимация)
│   └── checkCollisions()    — поедание зайцев, death
└── render()
    ├── drawMaze()           — DFS-лабиринт + комнаты
    ├── drawCarrot()         — морковные грядки
    ├── drawSnake()          — змейка с fading tail
    └── drawRabbit()         — зайцы с прыжковой анимацией
```

### Bunny AI (Weighted Movement)

- **Flee:** когда змейка ближе порога опасности — прыжок в самую дальнюю грядку
- **Roam:** случайное перемещение с предпочтением морковных грядок
- **Личности:** у каждого зайца свои параметры (смелость, пугливость, скорость)
- **Несовершенство:** 10–25% шанс субоптимального выбора

## 📂 Состав проекта

```
carrot-maze/
├── index.html                # Игра (900 строк, 42 функции)
├── README.md
├── AGENTS.md                 # GSD Leader инструкции
├── RESEARCH.md               # Rabbit AI Research
└── docs/
    ├── rabbit-ai-research.md
    ├── snake-mechanics-research.md
    ├── visual-design-research.md
    └── spec-sync-mechanisms-research.md
```

## 🚀 Деплой

**URL:** https://rem2222.github.io/carrot-maze/
**Репозиторий:** https://github.com/Rem2222/carrot-maze
