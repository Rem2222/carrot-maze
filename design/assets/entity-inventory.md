# Entity Inventory — Carrot Maze v3

> Создан по методологии CCGS asset-spec. Задача: MUL-75 «Доработка дизайна».
> Требования: более детализированные спрайты, милый стиль, 8-bit ретро, 2-3 варианта на тип, сохранить анимацию.

## Progress Summary

| Priority | Count | Status |
|----------|-------|--------|
| Critical | 6 | 6 proposed |
| High | 15 | 15 proposed |
| Medium | 6 | 6 proposed |
| **Total** | **27** | **All proposed** |

---

## Entity Inventory Table

| ID | Name | Category | Current | Dimensions | Palette | Frames | Priority | Status |
|----|------|----------|---------|------------|---------|--------|----------|--------|
| `SNAKE_HEAD` | Snake Head | player | fillRect | 16→24px | NES/PICO-8 | 4 anim | Critical | Proposed |
| `SNAKE_BODY` | Snake Body | player | fillRect | 16×16 | NES/PICO-8 | 1 static | Critical | Proposed |
| `SNAKE_TAIL` | Snake Tail | player | fillRect | 16×16 | NES/PICO-8 | 2 anim | Critical | Proposed |
| `RABBIT_WHITE` | White Rabbit | enemy | fillRect | 24×24 | PICO-8 | 4+4 | Critical | Proposed |
| `RABBIT_BROWN` | Brown Rabbit | enemy | — | 24×24 | PICO-8 | 4+4 | High | Proposed |
| `RABBIT_GREY` | Grey Rabbit | enemy | — | 24×24 | PICO-8 | 4+4 | High | Proposed |
| `CARROT` | Carrot | item_collectible | fillRect | 12×20 | NES | 4 sway | Critical | Proposed |
| `CARROT_GOLDEN` | Golden Carrot | item_collectible | — | 12×20 | NES | 4+sparkle | High | Proposed |
| `BONUS_SLOW` | Slow bonus | item_bonus | Emoji | 16×16 | NES | 1+4 | High | Proposed |
| `BONUS_SPEED` | Speed bonus | item_bonus | Emoji | 16×16 | NES | 1+4 | High | Proposed |
| `BONUS_KILL` | Kill bonus | item_bonus | Emoji | 16×16 | NES | 1+4 | High | Proposed |
| `BONUS_LIFE` | +Life bonus | item_bonus | Emoji | 16×16 | NES | 1+4 | High | Proposed |
| `BONUS_CHOP` | Chop bonus | item_bonus | Emoji | 16×16 | NES | 1+4 | High | Proposed |
| `BONUS_GHOST` | Ghost bonus | item_bonus | Emoji | 16×16 | NES | 1+4 | High | Proposed |
| `BONUS_RAGE` | Rage bonus | item_bonus | Emoji | 16×16 | NES | 1+4 | High | Proposed |
| `BONUS_WARP` | Warp bonus | item_bonus | Emoji | 16×16 | NES | 1+4 | High | Proposed |
| `BONUS_FREEZE` | Freeze bonus | item_bonus | Emoji | 16×16 | NES | 1+4 | High | Proposed |
| `HEART_FULL` | Full Heart | ui_element | Text | 12×12 | NES/PICO-8 | 1+2 | Critical | Proposed |
| `HEART_EMPTY` | Empty Heart | ui_element | Text | 12×12 | NES/PICO-8 | 1 | High | Proposed |
| `PARTICLE_SPARK` | Spark | vfx | fillRect | 4×4 | NES | 1 | Medium | Proposed |
| `PARTICLE_STAR` | Star | vfx | fillRect | 6×6 | NES | 4 | Medium | Proposed |
| `BED_TILE` | Bed/Wall | environment | fillRect | 30×30 | NES | 1 | Medium | Proposed |
| `FLOOR_TILE` | Floor | environment | fillRect | 30×30 | NES | 1 | Medium | Proposed |
| `BED_GRASS` | Grass deco | environment | fillRect | 8×8 | NES | 2 | Medium | Proposed |

---

## SNAKE HEAD — 3 Variants

### Variant A: Chunky Cute (милый толстячок)

| Property | Value |
|----------|-------|
| Dimensions | 24×24 px, sprite sheet 4 frames |
| Palette | NES: #00A800 green, #005800 dark, #F8B800 yellow, #F83800 red, #FFFFFF white, #000000 outline |
| Frames | 4 (idle bob + tongue flicker), 4 fps |
| Visual | Круглая голова, большие белые глаза (4px) с чёрными зрачками (2px), милая улыбка, красный раздвоенный язык (1 фрейм из 4), тёмная обводка 1px, сверху два «ушка»-бугорка 2×3px |
| AI Prompt | "24x24 pixel art cute snake head, round chunky shape, top-down view, NES color palette, bright green body with dark green outline, big white eyes with black pupils, cute smile, red forked tongue flicking, small ear-like bumps on top, 8-bit retro game style, no anti-aliasing, transparent background, 4-frame horizontal sprite sheet for idle animation" |
| **Фишка** | Максимально милый — большие глаза, улыбка, «ушки» |

### Variant B: Classic Pixel Snake

| Property | Value |
|----------|-------|
| Dimensions | 20×20 px, 4 frames |
| Palette | NES: #58D854 lime, #009800 mid, #004000 dark, #F83800 red, #000000 |
| Visual | Классическая ромбовидная голова, два жёлтых глаза (2×2px) по бокам, красный язык на 2 из 4 фреймов, чешуя — 2-3 пиксельные точки |
| AI Prompt | "20x20 pixel art snake head, diamond shape, top-down view, NES palette, lime green with dark shading, two yellow snake eyes on sides, red forked tongue extended, subtle scale texture dots, classic retro arcade style, no anti-aliasing, transparent background, 4-frame sprite sheet" |
| **Фишка** | Классический аркадный стиль, ближе к оригинальной змейке |

### Variant C: Baby Python (малыш-питон)

| Property | Value |
|----------|-------|
| Dimensions | 22×22 px, 4 frames, 5 fps |
| Palette | PICO-8: #00A800, #FFEC27, #FF004D, #FFFFFF, #000000 |
| Visual | Голова-сердечко (перевёрнутое), большие глаза с белым бликом (1px), розовые щёчки (2×2px), язык-сердечко на конце |
| AI Prompt | "22x22 pixel art baby python head, heart-shaped inverted, top-down view, PICO-8 palette, soft green with yellow belly, huge sparkly eyes with white highlight, pink blush cheeks, tiny heart-shaped tongue tip, ultra cute kawaii 8-bit pixel art, no anti-aliasing, transparent background, 4-frame sprite sheet" |
| **Фишка** | Ультра-кавайный, язык-сердечко, щёчки |

---

## RABBIT — 3 Variants

### Variant A: Fluffy Bunny (пушистый)

| Property | Value |
|----------|-------|
| Dimensions | 24×24 px, 8 frames (4 idle + 4 jump) |
| Palette | PICO-8: #FFF1E8 cream, #FFA0C5 pink, #FF004D dark pink, #000000 |
| Frames | 8 total: idle=ear twitch+nose wiggle (3fps), jump=squash→stretch→air→land (8fps) |
| Visual | Пухлый заяц, большие висячие уши (10px), круглое тело, хвостик-помпон 4×4px, большие глаза с бликом, розовый нос-треугольник, лапки 2px |
| AI Prompt | "24x24 pixel art cute fluffy bunny, side view, PICO-8 color palette, cream white round body, long floppy pink ears 10px, fluffy pom-pom tail, big sparkly eyes, pink triangle nose, tiny paws, 8-bit retro kawaii, no anti-aliasing, transparent background, 8-frame horizontal sprite sheet: 4 idle ear-twitch + 4 jump squash-stretch" |
| **Фишка** | Максимальная пушистость, большие висячие уши |

### Variant B: Ninja Rabbit (заяц-ниндзя)

| Property | Value |
|----------|-------|
| Dimensions | 22×22 px, 8 frames |
| Palette | NES: #A8A8A8 grey, #F8B800 yellow, #000000, #585858 |
| Visual | Поджарый серый заяц, стоячие острые уши (8px), жёлтая повязка ниндзя на голове, узкие глаза, боевая стойка, резкий прыжок с группировкой |
| AI Prompt | "22x22 pixel art ninja rabbit, side view, NES palette, sleek grey body, tall pointed ears, yellow ninja headband trailing, narrow determined eyes, athletic combat stance, 8-bit retro arcade, no anti-aliasing, transparent background, 8-frame sprite sheet: 4 idle + 4 jump" |
| **Фишка** | Самый динамичный, для hardcore |

### Variant C: Sleepy Rabbit (сонный заяц)

| Property | Value |
|----------|-------|
| Dimensions | 24×24 px, 8 frames |
| Palette | PICO-8: #AB7B5C brown, #FFEC27 cream, #FF004D pink, #7E2553, #000000 |
| Visual | Пухлый коричневый заяц, глаза-чёрточки (полузакрытые), зевает в idle, уши висят вниз, двигается медленно и лениво |
| AI Prompt | "24x24 pixel art sleepy lazy rabbit, side view, PICO-8 palette, chubby brown body, half-closed sleepy eyes as horizontal lines, yawning mouth in idle, droopy floppy ears, cute lazy vibe, 8-bit kawaii pixel art, no anti-aliasing, transparent background, 8-frame sprite sheet" |
| **Фишка** | Самый милый, зевает |

---

## CARROT — 3 Variants

### Variant A: Chunky Carrot

| Property | Value |
|----------|-------|
| Dimensions | 12×20 px, 4 frames |
| Palette | NES: #F89820 orange, #E86800 dark, #58A828 green, #387800 dark green, #FFFFFF highlight |
| Visual | Толстая морковка, 3 оттенка оранжевого для объёма, белый блик слева, тёмная тень справа, пышная ботва из 5 листиков (3×5px) веером, sway-анимация |
| AI Prompt | "12x20 pixel art cute chunky carrot, NES color palette, rounded tip, 3 orange shades for 3D volume with white highlight, lush green leafy top with 5 individual leaves fanning, 8-bit retro, no anti-aliasing, transparent background, 4-frame sway animation sprite sheet" |
| **Фишка** | Объёмная с бликом, пышная ботва |

### Variant B: Baby Carrot

| Property | Value |
|----------|-------|
| Dimensions | 10×14 px, 4 frames, 4 fps |
| Palette | PICO-8: #FFA300 orange, #FFEC27 yellow, #008751 green, #00A800 light green |
| Visual | Маленькая морковка, пропорционально огромная ботва (3 крупных листа, половина спрайта), ярко-оранжевая с жёлтым бликом, chibi-пропорции |
| AI Prompt | "10x14 pixel art baby carrot, PICO-8 palette, bright orange with yellow highlight, oversized cute green leafy top 3 large round leaves half the sprite, chibi proportions, 8-bit kawaii, no anti-aliasing, transparent background, 4-frame sway sprite sheet" |
| **Фишка** | Chibi, огромная ботва |

### Variant C: Golden Carrot (редкая)

| Property | Value |
|----------|-------|
| Dimensions | 12×20 px, 4 frames + sparkle |
| Palette | NES: #F8D800 gold, #F8B800 dark gold, #FFFFFF sparkle, #58A828 green |
| Visual | Золотая морковка, золотой градиент, белые звёздочки-блики (2×2px) на 1 фрейме из 4. ×3 очков |
| AI Prompt | "12x20 pixel art golden carrot, NES palette, gold gradient body, sparkle highlight dots 2x2px white, green leafy top, rare item variant, 8-bit retro, no anti-aliasing, transparent background, 4-frame sprite sheet with sparkle every 4th frame" |
| **Фишка** | Для редкого дропа, золотая + sparkle |

---

## BONUS ICONS — 9 Types

Все 16×16 px, NES palette, sprite sheet (4-frame pulse: scale 1.0→1.1→1.0→0.95).

| ID | Icon | Color | AI Prompt |
|----|------|-------|-----------|
| SLOW | Tiny turtle 4×6px or snowflake | #4FC3F7 ice blue | "16x16 NES palette, cute tiny turtle in rounded square, ice blue bg, 'SLOW' label, 4-frame pulse" |
| SPEED | Tiny wing 6×4px or lightning | #FFD54F yellow | "16x16 NES palette, lightning bolt, yellow bg, 'FAST' label, 4-frame pulse" |
| KILL | Tiny skull 6×6px | #E53935 red | "16x16 NES palette, tiny skull icon, red bg, 'KILL' label, 4-frame pulse" |
| LIFE | Heart with + symbol 6×6px | #EC407A pink | "16x16 NES palette, heart with plus, pink bg, '+1' label, 4-frame pulse" |
| CHOP | Scissors 6×8px | #9CCC65 lime | "16x16 NES palette, scissors icon, lime green bg, 'CUT' label, 4-frame pulse" |
| GHOST | Cute ghost 8×10px | #CE93D8 purple | "16x16 NES palette, cute ghost, purple bg, 'GHOST' label, 4-frame pulse" |
| RAGE | Flame 6×8px | #FF7043 orange | "16x16 NES palette, flame icon, orange-red bg, 'RAGE' label, 4-frame pulse" |
| WARP | Circular arrows 8×8px | #4DD0E1 cyan | "16x16 NES palette, circular arrows, cyan bg, 'WARP' label, 4-frame pulse" |
| FREEZE | Ice crystal 6×8px | #80DEEA light blue | "16x16 NES palette, ice crystal, light blue bg, 'ICE' label, 4-frame pulse" |

---

## HEARTS — 2 Variants

### Variant A: Pixel Heart Classic

| Property | Value |
|----------|-------|
| Dimensions | 12×12 px, 2 frames (Full beat) + 1 (Empty static) |
| Palette | #FF5252 red, #D32F2F dark, #FF8A80 highlight, #555555 empty |
| Visual | Классическое пиксельное сердечко: 2 дуги сверху, треугольник снизу. Full = красное с розовым бликом, Empty = тёмно-серый контур. Beat = расширение на 1px |
| AI Prompt | "12x12 pixel art heart icon, NES palette, classic heart shape two curves top triangle bottom, red fill with pink highlight dot, dark outline, 8-bit retro UI, no anti-aliasing, transparent background, 2-frame beat animation pulse" |
| **Фишка** | Классика, читаемо на любом фоне |

### Variant B: Chibi Heart

| Property | Value |
|----------|-------|
| Dimensions | 14×14 px, 5 frames (3 states: full/half/empty, full has 2 beat frames) |
| Palette | PICO-8: #FF004D, #FF77A8, #FFF1E8, #7E2553 |
| Visual | Округлое «толстое» сердечко. Full = ярко-розовое с белым бликом-звёздочкой (2×2px). Half = правая половина закрашена. Empty = контур. Beat = блик пульсирует |
| AI Prompt | "14x14 pixel art chibi heart icon, PICO-8 palette, round plump heart shape, bright pink with white star highlight, three states full/half/empty, 8-bit cute UI, no anti-aliasing, transparent background, 5-frame sprite sheet" |
| **Фишка** | Три состояния, звёздочка-блик |

---

## VFX & ENVIRONMENT

### Particles
- **PARTICLE_SPARK** 4×4px: white square with diagonal highlight line
- **PARTICLE_STAR** 6×6px: 4-pointed cross star, 4-frame rotation

### Tiles
- **BED_TILE** 30×30px: green bed with dark soil border, small plant decorations, tileable
- **FLOOR_TILE** 30×30px: brown floor, subtle texture, tileable  
- **BED_GRASS** 8×8px: grass tuft decoration, 2-frame sway

---

## Asset Manifest

```
player (3):    SNAKE_HEAD [A/B/C], SNAKE_BODY, SNAKE_TAIL — Critical, Proposed
enemy (3):     RABBIT_WHITE [A/B/C], RABBIT_BROWN [A/B/C], RABBIT_GREY [A/B] — Critical+High, Proposed
collectible:   CARROT [A/B/C], CARROT_GOLDEN [C] — Critical+High, Proposed
bonus (9):     SLOW, SPEED, KILL, LIFE, CHOP, GHOST, RAGE, WARP, FREEZE — High, Proposed
ui (2):        HEART_FULL [A/B], HEART_EMPTY [A/B] — Critical+High, Proposed
vfx (2):       PARTICLE_SPARK, PARTICLE_STAR — Medium, Proposed
environment:   BED_TILE, FLOOR_TILE, BED_GRASS — Medium, Proposed

TOTAL: 24 unique assets, 27 with variants
```

---

## Implementation Notes

**Порядок:** Critical (snake, rabbit, carrot, hearts) → High (bonuses, variants) → Medium (vfx, tiles).

**Код:** замена fillRect на drawImage, спрайты как data URLs или sprite sheet PNG. Существующие SPRITES.kawaii/nes — дополнить drawImage.

**Палитры:** player=одна палитра, enemy=одна, UI=NES, bonuses=NES.

*Создано GSD Researcher по скиллу ccgs-entity-inventory, MUL-75.*
