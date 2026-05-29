# Visual Design Research: Carrot Maze

> **Context:** Browser-based pixel-art game on HTML5 Canvas. Snake hunts rabbits in a grid maze of carrot beds.
> Canvas ~600×600px, grid 20–30 cells. Dark green maze, orange carrots, white rabbits, green snake.

---

## 1. Color Palette — Best Practices for Game Readability

### 1.1 Guiding Principles

| Principle | What It Means for Carrot Maze |
|-----------|-------------------------------|
| **Contrast ratio ≥ 4.5:1** (WCAG AA) | Snake body vs. maze floor must be clearly distinguishable |
| **Hue separation** | Each entity type (snake, rabbit, carrot, wall, floor) should use distinct hue families |
| **Value/brightness hierarchy** | Interactive elements (snake, rabbits) should pop against background |
| **Mood consistency** | Dark + earthy + natural = "nighttime garden hunt" feel |
| **Color-blind safe** | Avoid red-green-only differentiation; use brightness + pattern as secondary channel |

### 1.2 Recommended Palette for Carrot Maze

```css
:root {
  /* --- Background & Page --- */
  --color-page-bg:      #1a1a2e;   /* Deep navy-black page surround */
  --color-canvas-border: #2d2d44;  /* Subtle canvas border */

  /* --- Maze Floor --- */
  --color-floor:        #2d5a1e;   /* Dark green — dirt/grass base */
  --color-floor-alt:    #264d18;   /* Slightly darker checker variant */
  --color-wall:         #1b3a0f;   /* Very dark green — impassable walls */

  /* --- Grid Lines (subtle) --- */
  --color-grid:         #3a6b28;   /* Slightly lighter green for cell borders */

  /* --- Carrot Beds --- */
  --color-bed:          #4a3520;   /* Dark brown tilled soil */
  --color-carrot:       #f7931e;   /* Bright orange */
  --color-carrot-top:   #4caf50;   /* Green carrot tops */

  /* --- Snake (Player) --- */
  --color-snake-head:   #00e676;   /* Bright green head */
  --color-snake-body:   #4caf50;   /* Medium green body segments */
  --color-snake-eye:    #1b3a0f;   /* Dark eye */
  --color-snake-tongue: #ff5252;   /* Red flicker */

  /* --- Rabbits (Targets) --- */
  --color-rabbit-body:  #ffffff;   /* White fur */
  --color-rabbit-ear:   #ffcdd2;   /* Pink inner ear */
  --color-rabbit-eye:   #212121;   /* Dark eye */
  --color-rabbit-nose:  #f48fb1;   /* Pink nose */

  /* --- UI --- */
  --color-score:        #ffd54f;   /* Warm yellow score text */
  --color-overlay-bg:   rgba(0,0,0,0.75);  /* Game-over backdrop */
  --color-button:       #4caf50;   /* Green restart button */
  --color-button-hover: #66bb6a;
  --color-button-text:  #ffffff;
}
```

### 1.3 Contrast Check (Approximate)

| Foreground | Background | Contrast Ratio | Pass WCAG AA? |
|-----------|-----------|----------------|---------------|
| Snake head `#00e676` | Floor `#2d5a1e` | ~3.8:1 | Marginal — use thicker snake or outline |
| Carrot `#f7931e` | Bed `#4a3520` | ~5.2:1 | ✅ Yes |
| Rabbit `#ffffff` | Floor `#2d5a1e` | ~7.8:1 | ✅ Yes (AAA) |
| Score `#ffd54f` | Page `#1a1a2e` | ~10.5:1 | ✅ Yes (AAA) |

**Recommendation:** Add a 1px dark outline (or shadow) around the snake to ensure it pops against dark green floor. This is a classic pixel-art trick.

```js
// Snake rendering with dark outline for contrast
function drawSnakeSegment(ctx, x, y, w, h, isHead) {
    // 1px dark outline
    ctx.fillStyle = '#1b3a0f';
    ctx.fillRect(x - 1, y - 1, w + 2, h + 2);
    // Fill body
    ctx.fillStyle = isHead ? '#00e676' : '#4caf50';
    ctx.fillRect(x, y, w, h);
}
```

---

## 2. Pixel-Art Rendering Techniques on Canvas

### 2.1 The Critical CSS Property

Pixel art MUST use nearest-neighbor interpolation. Without this, your crisp pixel edges turn into blurry mush:

```css
canvas {
    image-rendering: pixelated;      /* Modern browsers (Chrome, Edge) */
    image-rendering: crisp-edges;    /* Firefox, Safari */
    image-rendering: -moz-crisp-edges;
    image-rendering: -webkit-crisp-edges;
}
```

### 2.2 Internal Canvas Strategy: Draw at Native Resolution, Scale via CSS

```html
<!-- Canvas internal resolution matches the grid you draw -->
<canvas id="game" width="600" height="600"></canvas>
```

```css
/* CSS scales the canvas visually — keep pixel-art crisp */
#game {
    width: 600px;
    height: 600px;
    image-rendering: pixelated;
    image-rendering: crisp-edges;
}
```

**For Carrot Maze:** 600×600 internal resolution, 20–30 cell grid = each cell is 20–30px. This is the perfect size for pixel art — you can draw 20×20px sprites with 1px detail.

### 2.3 Sub-Pixel Rendering on Canvas: Integer Coordinates Only

Pixel art breaks when coordinates land on fractional pixels. Always floor/round draw positions:

```js
// BAD — fractional coords cause blurry edges
ctx.fillRect(x, y, w, h); // x = 100.7 → blur

// GOOD
const ix = Math.floor(x);
const iy = Math.floor(y);
ctx.fillRect(ix, iy, w, h);
```

### 2.4 Scaled-Up Internal Canvas (Retro Look)

If you want a CHUNKY pixel look (like old 8-bit games), draw at a LOW internal resolution and scale up:

```html
<!-- Internal: 120×120 (10px cells for 12×12 grid) — scale up to 600px -->
<canvas id="game" width="120" height="120"></canvas>
```

```css
#game {
    width: 600px;  /* 5× scale — chunky pixels */
    height: 600px;
    image-rendering: pixelated;
    image-rendering: crisp-edges;
}
```

**Recommendation for Carrot Maze:** Use 600×600 native resolution (not scaled-up) — the 20–30px cells give a nice retro feel without being too chunky. This also allows smooth sub-cell animation for the rabbit jump arcs and snake slither.

### 2.5 Disable Anti-Aliasing on Lines

```js
// Turn off smoothing for pixel-perfect lines
ctx.imageSmoothingEnabled = false;
```

This matters if you draw images/sprites. For `fillRect`, Canvas does NOT anti-alias integer coordinates, so it's naturally crisp.

---

## 3. Visual Polish

### 3.1 Screen Shake (On Snake Death or Rabbit Capture)

Screen shake adds visceral feedback. Apply as a translate offset on the root canvas context:

```js
class ScreenShake {
    constructor() {
        this.intensity = 0;   // Current shake magnitude (pixels)
        this.duration = 0;    // Remaining time (ms)
        this.decay = 0.9;     // How fast intensity fades per frame
    }

    trigger(intensity = 8, duration = 200) {
        this.intensity = intensity;
        this.duration = duration;
    }

    update(deltaTime) {
        if (this.duration <= 0) {
            this.intensity = 0;
            return;
        }
        this.duration -= deltaTime;
        this.intensity *= this.decay;
    }

    getOffset() {
        if (this.intensity < 0.5) return { x: 0, y: 0 };
        return {
            x: (Math.random() - 0.5) * this.intensity * 2,
            y: (Math.random() - 0.5) * this.intensity * 2
        };
    }
}

// Usage in render loop:
const shake = screenShake.getOffset();
ctx.save();
ctx.translate(shake.x, shake.y);

drawMaze(ctx);
drawSnake(ctx);
drawRabbits(ctx);

ctx.restore();
```

**Trigger moments:**
- Snake eats a rabbit → `trigger(4, 150)` (mild thump)
- Snake dies → `trigger(12, 400)` (violent shake)

### 3.2 Particle Effects

Simple pixel-particle system for eating rabbits or death explosions:

```js
class Particle {
    constructor(x, y, color, speed, life) {
        this.x = x;
        this.y = y;
        this.color = color;
        this.vx = (Math.random() - 0.5) * speed;
        this.vy = (Math.random() - 0.5) * speed;
        this.life = life;       // ms
        this.maxLife = life;
        this.size = Math.random() * 4 + 2;
    }

    update(dt) {
        this.x += this.vx * (dt / 16);
        this.y += this.vy * (dt / 16);
        this.life -= dt;
    }

    draw(ctx) {
        const alpha = Math.max(0, this.life / this.maxLife);
        ctx.globalAlpha = alpha;
        ctx.fillStyle = this.color;
        ctx.fillRect(
            Math.floor(this.x - this.size / 2),
            Math.floor(this.y - this.size / 2),
            this.size, this.size
        );
        ctx.globalAlpha = 1;
    }
}

class ParticleSystem {
    constructor() {
        this.particles = [];
    }

    emit(x, y, color, count = 12, speed = 3, life = 400) {
        for (let i = 0; i < count; i++) {
            this.particles.push(new Particle(x, y, color, speed, life));
        }
    }

    update(dt) {
        this.particles = this.particles.filter(p => p.life > 0);
        this.particles.forEach(p => p.update(dt));
    }

    draw(ctx) {
        this.particles.forEach(p => p.draw(ctx));
    }
}

// Usage:
// When rabbit is eaten:
particles.emit(rabbitX, rabbitY, '#ffffff', 15, 4, 500); // white burst
particles.emit(rabbitX, rabbitY, '#f7931e', 8, 3, 400);  // carrot flecks
// When snake dies:
particles.emit(headX, headY, '#00e676', 25, 5, 600);
```

### 3.3 Smooth Transitions — Rabbit Jump Animation

Already partially designed in RESARCH.md. Add a **squash-and-stretch** for more polish:

```js
function drawRabbit(ctx, rabbit) {
    const x = rabbit.visualX;
    const y = rabbit.visualY - (rabbit.jumpOffset || 0);

    // Squash-and-stretch based on jump progress
    let scaleX = 1, scaleY = 1;
    if (rabbit.state === 'jumping') {
        const t = rabbit.animProgress;
        if (t < 0.2) {
            // Takeoff: squash horizontally, stretch vertically
            scaleX = 1.2;
            scaleY = 0.8;
        } else if (t > 0.8) {
            // Landing: squash vertically, stretch horizontally
            scaleX = 0.8;
            scaleY = 1.2;
        }
    }

    ctx.save();
    ctx.translate(x, y);
    ctx.scale(scaleX, scaleY);

    // Body
    ctx.fillStyle = '#ffffff';
    ctx.fillRect(-8, -8, 16, 16);
    // Ears
    ctx.fillStyle = '#ffcdd2';
    ctx.fillRect(-6, -14, 4, 8);
    ctx.fillRect(2, -14, 4, 8);
    // Eyes
    ctx.fillStyle = '#212121';
    ctx.fillRect(-3, -3, 3, 3);
    ctx.fillRect(1, -3, 3, 3);

    ctx.restore();
}
```

### 3.4 Carrot Wobble / Idle Animation

Static carrots are functional but boring. Add a subtle sine-based sway:

```js
function drawCarrot(ctx, cx, cy, size, time) {
    const sway = Math.sin(time * 0.003 + cx * 0.1) * 1.5; // ±1.5px wobble
    const rx = Math.floor(cx + sway);
    const ry = Math.floor(cy);

    // Carrot body (orange triangle approximated as rects)
    ctx.fillStyle = '#f7931e';
    ctx.fillRect(rx - 3, ry - 2, 6, 12);
    // Taper to point
    ctx.fillRect(rx - 2, ry + 10, 4, 4);

    // Green top
    ctx.fillStyle = '#4caf50';
    ctx.fillRect(rx - 4, ry - 5, 2, 4);  // left leaf
    ctx.fillRect(rx + 2, ry - 5, 2, 4);  // right leaf
    ctx.fillRect(rx - 1, ry - 6, 2, 5);  // center leaf
}
```

### 3.5 Snake Body Trail with Fading Tail

Instead of uniform green, fade the tail segments for depth:

```js
function drawSnake(ctx, segments, time) {
    const len = segments.length;
    for (let i = len - 1; i >= 0; i--) {
        const seg = segments[i];
        const isHead = (i === 0);

        // Fade: head brightest, tail darker
        const fadeRatio = 1 - (i / (len + 5)) * 0.35;
        const r = Math.floor(0x4c * fadeRatio);
        const g = Math.floor(0xaf * fadeRatio);
        const b = Math.floor(0x50 * fadeRatio);
        ctx.fillStyle = `rgb(${r},${g},${b})`;

        // Outline for contrast
        ctx.fillStyle = '#1b3a0f';
        ctx.fillRect(seg.x - 1, seg.y - 1, seg.w + 2, seg.h + 2);

        // Body fill
        ctx.fillStyle = `rgb(${r},${g},${b})`;
        ctx.fillRect(seg.x, seg.y, seg.w, seg.h);

        // Eyes on head
        if (isHead) {
            ctx.fillStyle = '#1b3a0f';
            ctx.fillRect(seg.x + 4, seg.y + 3, 3, 3);
            ctx.fillRect(seg.x + 13, seg.y + 3, 3, 3);
        }
    }
}
```

### 3.6 CSS Backdrop / Page Atmosphere

```css
body {
    margin: 0;
    padding: 0;
    /* Dark radial vignette — draws eye to center */
    background: radial-gradient(ellipse at center, #1e2a1e 0%, #0a0f0a 100%);
    min-height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
    font-family: 'Press Start 2P', monospace;
    /* Subtle pixel-art noise texture via CSS if desired */
}

/* Optional: scanline overlay for retro CRT feel */
body::after {
    content: '';
    position: fixed;
    top: 0; left: 0; right: 0; bottom: 0;
    background: repeating-linear-gradient(
        0deg,
        transparent,
        transparent 2px,
        rgba(0,0,0,0.03) 2px,
        rgba(0,0,0,0.03) 4px
    );
    pointer-events: none;
    z-index: 100;
}
```

---

## 4. UI Layer — Score, Game-Over, Restart

### 4.1 HTML Overlay vs. Canvas-Drawn — Recommendation

| Approach | Pros | Cons |
|----------|------|------|
| **HTML overlay** | Native text rendering (crisp), CSS animations, accessible, easier to style | Layering over canvas, z-index management |
| **Canvas-drawn** | Everything in one rendering pass, consistent pixel-art look | Text rendering is blurry at small sizes; re-rendering cost; no accessibility |

**Recommendation: Hybrid approach.**

- **Canvas-drawn:** In-game HUD (score, level indicator) — minimal text, rendered in pixel-art style
- **HTML overlay:** Game-over screen, restart button — needs clickable, styled elements

### 4.2 Score Display (Canvas-Drawn)

Pixel-art score in the top-left:

```js
function drawScore(ctx, score, highScore = 0) {
    const x = 12;
    const y = 12;

    // Score background (semi-transparent dark panel)
    ctx.fillStyle = 'rgba(0,0,0,0.5)';
    ctx.fillRect(x - 4, y - 4, 180, 52);
    // Border
    ctx.strokeStyle = '#4caf50';
    ctx.lineWidth = 1;
    ctx.strokeRect(x - 4, y - 4, 180, 52);

    // "SCORE" label
    ctx.fillStyle = '#a5d6a7';
    ctx.font = '10px monospace';
    ctx.textBaseline = 'top';
    ctx.fillText('🐍 SCORE', x, y);

    // Score value (large)
    ctx.fillStyle = '#ffffff';
    ctx.font = 'bold 20px monospace';
    ctx.fillText(String(score).padStart(6, '0'), x, y + 12);

    // High score
    ctx.fillStyle = '#ffd54f';
    ctx.font = '9px monospace';
    ctx.fillText('HI ' + String(highScore).padStart(6, '0'), x + 100, y + 1);
}
```

### 4.3 Game-Over Overlay (HTML + CSS)

```html
<div id="game-over" class="overlay hidden">
    <div class="overlay-content">
        <h1 class="game-over-title">GAME OVER</h1>
        <p class="final-score">Score: <span id="final-score">0</span></p>
        <p class="high-score">Best: <span id="high-score">0</span></p>
        <button id="restart-btn" class="restart-btn">
            ▶ PLAY AGAIN
        </button>
        <p class="hint">Press ENTER or SPACE to restart</p>
    </div>
</div>
```

```css
.overlay {
    position: absolute;
    top: 0; left: 0; right: 0; bottom: 0;
    background: rgba(0, 0, 0, 0.78);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 10;
    /* Fade in */
    animation: overlayFadeIn 0.4s ease-out;
}

.overlay.hidden {
    display: none;
}

@keyframes overlayFadeIn {
    from { opacity: 0; }
    to   { opacity: 1; }
}

.overlay-content {
    text-align: center;
    color: #ffffff;
    /* Pixel-art border */
    border: 4px solid #4caf50;
    padding: 40px 60px;
    background: #1a1a2e;
    /* Simulate pixel-art box shadow (stair-step) */
    box-shadow:
        4px 0   0 0 #0d0d1a,
        0   4px 0 0 #0d0d1a,
        4px 4px 0 0 #0d0d1a;
}

.game-over-title {
    font-family: 'Press Start 2P', monospace;
    font-size: 28px;
    color: #ff5252;
    margin: 0 0 24px 0;
    /* Pixel-art text shadow */
    text-shadow: 3px 3px 0 #b71c1c;
    /* Flicker animation */
    animation: titleFlicker 2s infinite;
}

@keyframes titleFlicker {
    0%, 100% { opacity: 1; }
    92%      { opacity: 0.85; }
    94%      { opacity: 1; }
    96%      { opacity: 0.9; }
}

.final-score {
    font-family: 'Press Start 2P', monospace;
    font-size: 16px;
    color: #ffd54f;
    margin: 12px 0;
}

.restart-btn {
    font-family: 'Press Start 2P', monospace;
    font-size: 14px;
    background: #4caf50;
    color: #ffffff;
    border: none;
    padding: 14px 32px;
    cursor: pointer;
    margin-top: 20px;
    transition: transform 0.1s, background 0.15s;
    /* Pixel-art look: hard corners, no rounding */
    border-radius: 0;
    /* Bottom border gives 3D pixel-art button depth */
    border-bottom: 4px solid #2e7d32;
}

.restart-btn:hover {
    background: #66bb6a;
    transform: translateY(-2px);
    border-bottom-width: 6px;
}

.restart-btn:active {
    transform: translateY(2px);
    border-bottom-width: 2px;
}

.hint {
    font-family: 'Press Start 2P', monospace;
    font-size: 9px;
    color: #757575;
    margin-top: 16px;
}
```

### 4.4 Wrapper Layout

```html
<div id="game-wrapper">
    <canvas id="game" width="600" height="600"></canvas>
    <div id="game-over" class="overlay hidden"><!-- ... --></div>
</div>
```

```css
#game-wrapper {
    position: relative;   /* So overlay can be absolute over canvas */
    width: 600px;
    height: 600px;
    /* Retro pixel-art border */
    border: 6px solid #2d2d44;
    /* Pixel-art corner effect via box-shadow */
    box-shadow:
        0 0 0 6px #1a1a2e,
        0 0 0 8px #4caf50,
        0 0 20px rgba(0, 230, 118, 0.15);
}
```

---

## 5. Typography for Game UI

### 5.1 Pixel Font Options

| Font | Source | License | Characteristics |
|------|--------|---------|----------------|
| **Press Start 2P** | Google Fonts | OFL | Classic 8-bit feel, full Latin set, readable at 8–16px |
| **VT323** | Google Fonts | OFL | Terminal-style monospace, narrower, great for scoreboards |
| **Pixelify Sans** | Google Fonts | OFL | Modern pixel sans, very clean, good at small sizes |
| **Silkscreen** | Google Fonts | OFL | Very small bitmap font, legible at 8px |
| **DotGothic16** | Google Fonts | OFL | Japanese-inspired bitmap, distinctive |
| **Tiny5** | Google Fonts | OFL | Extremely compact 5px font, good for tiny HUD elements |

### 5.2 Recommendation for Carrot Maze

**Primary: Press Start 2P** — for game title, game-over, buttons. It's the most iconic pixel font.
**Secondary: VT323** — for score display. Narrower = fits more digits, more readable at smaller sizes.

```html
<!-- Google Fonts import -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Press+Start+2P&family=VT323&display=swap" rel="stylesheet">
```

```css
:root {
    --font-pixel: 'Press Start 2P', monospace;
    --font-mono:  'VT323', 'Courier New', monospace;
}

body {
    font-family: var(--font-pixel);
}

.score-display {
    font-family: var(--font-mono);
    font-size: 24px;
}
```

### 5.3 Canvas Text Rendering for Pixel Fonts

Google Fonts work on Canvas via `ctx.font`, but at small sizes they can blur. For Canvas-drawn scores, use a bitmap approach or system monospace:

```js
// Option A: Use system monospace (guaranteed crisp at integer sizes)
ctx.font = 'bold 20px monospace';

// Option B: Use loaded Google Font (may blur at small sizes on Canvas)
ctx.font = '24px "VT323", monospace';

// Option C: Draw pixel text manually (pure pixel-art approach)
// This is overkill for Carrot Maze — VT323 at 24px is crisp enough
```

**Recommendation:** For Canvas HUD, use `font: 'bold 20px monospace'` for maximum crispness. For HTML overlays, use the Google pixel fonts.

---

## 6. CSS Page Styling

### 6.1 Complete Page Layout

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Carrot Maze</title>
    <link href="https://fonts.googleapis.com/css2?family=Press+Start+2P&family=VT323&display=swap" rel="stylesheet">
    <style>
        /* === RESET === */
        *, *::before, *::after {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
        }

        /* === PAGE === */
        html, body {
            width: 100%;
            height: 100%;
            overflow: hidden;
            background: #0a0f0a;
        }

        body {
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            font-family: 'Press Start 2P', monospace;
            color: #ffffff;

            /* Dark radial gradient — draws focus to center */
            background:
                radial-gradient(ellipse at 50% 50%, #1e2a1e 0%, #0a0f0a 70%);

            /* Subtle pixel-art texture (CSS-only noise) */
            /* Uncomment for grain effect:
            background-image:
                radial-gradient(ellipse at 50% 50%, #1e2a1e 0%, #0a0f0a 70%),
                url("data:image/svg+xml,...");
            */
        }

        /* === GAME TITLE === */
        .game-title {
            font-size: 18px;
            color: #f7931e;
            text-shadow: 2px 2px 0 #4a3520;
            margin-bottom: 16px;
            letter-spacing: 2px;
            user-select: none;
        }

        /* === GAME CONTAINER === */
        #game-wrapper {
            position: relative;
            /* Canvas size + border */
            width: 612px;  /* 600 + 2*6px border */
            height: 612px;
            /* Pixel border effect */
            border: 6px solid #2d2d44;
            outline: 2px solid #4caf50;
            outline-offset: 2px;
            /* Glow */
            box-shadow:
                0 0 20px rgba(76, 175, 80, 0.1),
                0 0 60px rgba(76, 175, 80, 0.05);
            /* Pixel-art crisp scaling */
            image-rendering: pixelated;
            image-rendering: crisp-edges;
        }

        /* === CANVAS === */
        #game {
            display: block;  /* Removes inline spacing gap */
            width: 600px;
            height: 600px;
            image-rendering: pixelated;
            image-rendering: crisp-edges;
        }

        /* === RESPONSIVE === */
        @media (max-width: 640px) {
            #game-wrapper {
                width: calc(100vw - 24px);
                height: calc(100vw - 24px);
            }
            #game {
                width: 100%;
                height: 100%;
            }
            .game-title {
                font-size: 12px;
            }
        }

        @media (min-width: 641px) and (max-width: 1024px) {
            #game-wrapper {
                width: 500px;
                height: 500px;
            }
            #game {
                width: 488px;
                height: 488px;
            }
        }

        /* === INSTRUCTIONS (below canvas) === */
        .instructions {
            margin-top: 16px;
            font-size: 9px;
            color: #757575;
            text-align: center;
            user-select: none;
        }

        .instructions kbd {
            display: inline-block;
            background: #2d2d44;
            color: #a5d6a7;
            padding: 2px 6px;
            border: 1px solid #4a4a6a;
            font-family: 'VT323', monospace;
            font-size: 11px;
        }
    </style>
</head>
<body>
    <h1 class="game-title">🥕 CARROT MAZE 🐍</h1>
    <div id="game-wrapper">
        <canvas id="game" width="600" height="600"></canvas>
        <div id="game-over" class="overlay hidden">
            <!-- ...overlay content... -->
        </div>
    </div>
    <p class="instructions">
        <kbd>WASD</kbd> or <kbd>↑↓←→</kbd> to move &nbsp;|&nbsp;
        <kbd>SPACE</kbd> to restart
    </p>
</body>
</html>
```

### 6.2 Responsive Layout Strategy

The game runs at a fixed 600×600 logical resolution. For responsive behavior:

1. **Desktop (≥1025px):** Full 600×600 with generous margins
2. **Tablet (641–1024px):** Scale to 500×500, keep aspect ratio
3. **Mobile (<640px):** Fill viewport width minus padding, maintain square aspect ratio

```css
/* Ensure square aspect ratio on mobile */
#game-wrapper {
    aspect-ratio: 1 / 1;
    max-width: 600px;
    width: min(600px, calc(100vw - 32px));
    height: auto;
}

#game {
    width: 100%;
    height: auto;
    aspect-ratio: 1 / 1;
}
```

### 6.3 Dark Mode / System Preference

Since the design is inherently dark, it works in both light and dark OS modes. No special handling needed.

### 6.4 Favicon for Tab

```html
<link rel="icon" href="data:image/svg+xml,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 32 32'><rect width='32' height='32' fill='%232d5a1e'/><rect x='8' y='12' width='16' height='16' fill='%23f7931e' rx='1'/></svg>">
```

---

## Summary of Recommendations

| Area | Recommendation |
|------|---------------|
| **Palette** | Dark green floor (#2d5a1e), orange carrots (#f7931e), white rabbits, green snake with outline for contrast |
| **Pixel rendering** | 600×600 native canvas, `image-rendering: pixelated`, integer coordinates only |
| **Polish** | Screen shake on death/capture, particle bursts, squash-and-stretch rabbit jumps, carrot idle wobble |
| **UI** | Hybrid — Canvas HUD for score (monospace font), HTML overlay for game-over/restart (Press Start 2P) |
| **Typography** | Press Start 2P (titles), VT323 (scores), system monospace for Canvas text |
| **Page CSS** | Dark radial-gradient background, pixel-art border on wrapper, centered flexbox layout, responsive down to mobile |
