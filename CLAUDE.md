# Off the Leash! — Game Design & Development Guide

## Overview

A browser-based 2D pixel-art platformer in a single `index.html` file (~4400 lines). Three real dogs — Mango, Violet, and Mochi — escape the dogcatcher across 3 worlds (13 levels) plus a final Home level. Built with Phaser 3.90.0 via CDN, all art/sound generated procedurally in code.

Live at: `mderdzinski.github.io/otl.html` (hidden easter egg, no link from main site)

## Architecture

Everything is in one HTML file. No build tools, no server, no external assets.

| Section | Content |
|---------|---------|
| CSS (lines ~10-45) | Layout, touch controls styling |
| Touch JS (lines ~60-205) | Floating joystick, action buttons, haptics |
| Constants (lines ~210-260) | Tile IDs, palettes, dog stats, save key |
| Sprite Data (lines ~260-400) | createDogFrames(), getTileColor(), enemy colors |
| Level Data (lines ~400-700) | createLevel() with 2D tile arrays + spawn data |
| Sound Manager (lines ~700-1200) | Web Audio oscillators for SFX + multi-voice music |
| Texture Generator (lines ~1200-1950) | Canvas-drawn sprites for dogs, enemies, tiles, UI |
| Save System (lines ~1950-2000) | localStorage read/write |
| Scenes (lines ~2000-4050) | Boot, Title, CharSelect, WorldMap, Game, Pause, GameOver |
| Game Config (lines ~4050-4080) | Phaser.Game initialization |

## The Three Dogs

| Dog | Color | Speed | Jump | HP | Ability |
|-----|-------|-------|------|----|---------|
| Mango | Cream (#F0E0B8) | 200 (fastest) | -380 (3 tiles) | 2 | **Bark** — scares mailmen/bears away, stuns other enemies. 2s cooldown. |
| Violet | Grey (#9898A8) | 170 (medium) | -440 (4 tiles) | 2 | **Passive double jump** — press jump again in mid-air. No shift ability. |
| Mochi | Orange (#D07030) | 130 (slowest) | -400 (3.3 tiles) | 4 | **Slam** — ground pound that breaks cracked floors, stuns enemies, calms cats. 0.8s cooldown. |

**Mochi is the bottleneck for level design.** Every platform and collectible must be reachable by Mochi.

### Dog Particle Trails
- Mango: tiny paw print circles
- Violet: horizontal speed lines
- Mochi: ground-shake dust puffs (only when on ground)

## Physics Constants

| Parameter | Value | Notes |
|-----------|-------|-------|
| TILE_SIZE | 24px | Everything snaps to this grid |
| GAME_WIDTH | 640px | |
| GAME_HEIGHT | 432px | = 18 tiles |
| GRAVITY | 1000 | Applied per-body, not global |
| Level width | 50 tiles (1200px) | Except 1-4 which is 80 tiles |
| Level height | 17 tiles (408px) | |
| Ground | y=14 (grass), y=15-16 (dirt) | |

## Level Design Rules

### Platform Reachability (Mochi = bottleneck)

| Constraint | Mochi Limit | Safe Value |
|-----------|-------------|------------|
| Max vertical jump | 3.33 tiles | **3 tiles** |
| Max horizontal gap | 4.33 tiles | **4 tiles** |
| Jump arc time | 0.8 seconds | |

1. **Chain rule**: Every platform must be reachable from the previous one, starting from ground (y=14). Trace the full path.
2. **Max vertical**: From surface at y=M, can reach y = M-3 (e.g., ground y=14 → platform y=11)
3. **Max horizontal**: Gap between right edge of one platform and left edge of next must be <= 4 tiles
4. **Stepping stones**: If a gap is too wide, add an intermediate platform at y=11 to bridge it
5. **Staircase pattern**: For height, use ground(y=14) → y=11 → y=8 (two 3-tile jumps)
6. **No dead ends**: Always a way back down without dying
7. **Minimum clearance**: Platforms above ground MUST be at y=11 or higher (never y=12 or y=13). Violet is 27px tall — a platform at y=12 leaves only 24px of clearance, blocking her. Minimum gap = 2 tiles (48px). **Never use y=12 or y=13 for platforms over walkable ground.**

### Collectible Placement

1. **Never overlap**: No two collectibles at the same (x, y)
2. **Never inside terrain**: Must be 1+ tiles ABOVE any solid tile at that x
3. **Never over pits**: Must have solid ground/platform within 3 tiles below
4. **Pit check**: If ground was removed with `tiles[y][x] = T.EMPTY`, no collectibles at that x with y >= 13
5. **Treats on ground**: `addTreats(x, y, count)` at y=13 for ground-level bones
6. **Special items on platforms**: Squeaky toys (3 per level) and golden treats (1 per level) go ON platforms — they're the reward for exploration
7. **Spread squeaky toys**: Place in early, middle, and late sections of the level
8. **Water bowls near danger**: Before boss fights or hard sections, on platforms

### Level Layout Guidelines

- **Vertical variety**: Every level needs platforms at 2+ different heights
- **Moving platforms**: Must be the ONLY way across a gap (don't make them redundant)
- **Boss arenas**: Need elevated platforms (y=8+) for stomping down on boss. Staircase: ground → y=11 → y=8
- **Chase level (1-4)**: Checkpoint must be on safe ground (not over a pit). Chase boundary must start near checkpoint on respawn.

## Enemies

| Enemy | World | Behavior | Stompable? |
|-------|-------|----------|-----------|
| Vacuum | 1 | Patrols back & forth | Yes |
| Mailman | 1-2 | Patrols + throws letter projectiles | Yes |
| Squirrel | 1-3 | Darts away when player gets close | Yes (tight timing) |
| Sprinkler | 1 | Stationary, shoots water arc every 3s | Yes |
| Cat | 2 | Patrols + blocks paths. Mochi can calm them. | Yes |
| Pigeon | 2 | Flies sine-wave pattern, divebombs | Yes |
| Bear | 3 | Charges at player when close, lunges | Yes |
| Porcupine | 3 | Slow patrol. **CANNOT be stomped** — always hurts. | **No** |
| Dogcatcher | Boss | Chases player, throws projectiles, jumps. Multi-phase. | Yes (multiple stomps) |

### Enemy Design Rules
- Enemies reverse at platform edges (won't walk off ledges), except boss
- Enemies that fall off the world are destroyed
- Defeated enemies drop a bone treat
- `enemy.dying = true` + `enemy.body.enable = false` immediately on defeat to prevent double-hit
- Player gets 200ms invincibility after each stomp

## Boss Fights

| Level | Phase | HP | Shoot Interval | Special |
|-------|-------|-----|----------------|---------|
| 2-4 | 2 | 6 | 1300ms, double-shot | Medium speed |
| 3-4 | 3 | 9 | 700ms, triple-shot | Screen shake on landing, flashes red when low HP |

- Doghouse is hidden until boss is defeated (boss sign shown instead)
- Boss HP text floats above the boss (disappears on death via `boss.dying` check)
- 1-4 is a chase level, not a boss fight (dogcatcher chases from behind)

## Collectibles & Scoring

| Item | Effect | Per Level |
|------|--------|-----------|
| Bone Treat | Currency. 100 = extra life. | ~15-25 |
| Squeaky Toy | Collect all 3 = extra life. Tracked on HUD with sprite icons. | 3 |
| Golden Treat | +25 bonus treats + camera flash. | 1 |
| Water Bowl | Heal 1 HP. | 1-3 |

### Star Rating
- Under 30s = 3 stars
- Under 60s = 2 stars
- Otherwise = 1 star
- Finding all squeaky toys bumps rating up by 1 (max 3)
- Best rating saved per dog per level, shown on world map

## Music System

5-voice procedural chiptune via Web Audio API. Each world has unique 8-bar (32-beat) composition.

| World | BPM | Key | Feel | Melody Voice |
|-------|-----|-----|------|-------------|
| 1: Neighborhood | 126 | C major | Bright, bouncy (Yoshi's Island) | Triangle |
| 2: Downtown | 138 | E minor | Driving, urgent (Mega Man) | Square |
| 3: Outdoors | 92 | A minor | Atmospheric, haunting (Celeste) | Sine |
| Boss | 152 | D minor | Intense, relentless (Castlevania) | Sawtooth |
| Home | 108 | G major | Warm, triumphant (credits music) | Triangle |

Voices: melody, counter-melody, bass, chord pads, percussion (kick + hi-hat)

### Audio Rules
- `sfx.stopMusic()` disconnects the gain node (kills all pre-scheduled oscillators)
- Music stops on ALL scene transitions (Title, CharSelect, WorldMap, GameOver)
- SFX volume: 0.015-0.03 range (quiet, behind music)
- Music gain: 0.18

## Save System

localStorage key: `doggie_game_save`

```javascript
{
    unlockedLevels: { mango: ['1-1'], violet: ['1-1'], mochi: ['1-1'] },
    completed: { mango: [], violet: [], mochi: [] },
    goldenTreats: [],        // levelKey strings
    squeakyToys: [],         // 'levelKey-N' strings
    totalTreats: 0,
    lives: 3,                // MAX_LIVES = 3
    hardcore: false,
    stars: {}                // 'dog-levelKey': starCount
}
```

- **Normal mode**: Game over refills lives, keeps progress
- **Hardcore mode**: Game over wipes ALL progress. Also wipes on every new game start from title screen.
- Toggle with H key on title screen

## Touch Controls

Shown on touch devices only (`@media (hover: none) and (pointer: coarse)`).

- **Left 45%**: Floating joystick (appears at touch point, 15px deadzone, 45px max travel)
- **Right side**: JUMP (90px green) + ACT (76px red) buttons
- **Top-right**: Pause button
- **Top-left**: Fullscreen button (fallback to `scrollTo(0,1)` on iOS)
- Joystick + action buttons hidden on menu screens (`showGameTouchControls(false)`)
- Haptic feedback via `navigator.vibrate()` on all touches

## Controls (Keyboard)

| Key | Action |
|-----|--------|
| WASD / Arrows | Move |
| Space / W | Jump |
| Shift | Ability (Mango: bark, Violet: nothing, Mochi: slam) |
| M | Mute toggle |
| P / ESC | Pause |
| H | Toggle hardcore mode (title screen only) |

## Checkpoints

Red flag poles that turn green when activated. Currently on:
- Level 1-4 (chase, at x=35)
- Level 3-3 (cave, at x=25)

On death, player respawns at last checkpoint with:
- Fade-in animation
- ~700ms invincibility (flashing)
- Chase level: boundary resets near checkpoint, 500ms delay before chase resumes

## Versioning

The version is stored in `const VERSION` near the top of `index.html` and displayed on the title screen.

**Always bump the version when committing changes:**

| Change Type | Bump | Example |
|------------|------|---------|
| Bug fixes, collectible repositioning, tuning values | Patch (x.x.+1) | 1.1.0 → 1.1.1 |
| New features, new enemies, UI changes, level redesigns | Minor (x.+1.0) | 1.1.0 → 1.2.0 |
| New worlds, major content additions (doubling levels, new dog) | Major (+1.0.0) | 1.1.0 → 2.0.0 |

Update the `const VERSION = 'x.x.x';` line in the constants section before committing.

## Known Issues / Future Work

### Polish
- No idle animations (dogs are static when standing still)
- No walk cycle animation (dogs are single-frame sprites)
- No screen transition between title and character select
- Victory screen could show all 3 dogs together
- Moving platforms are plain brown rectangles — could use log/cloud textures

### Gameplay
- Moving platforms only in World 2 and 3-1 — could add to more levels
- No wall-jumping mechanic
- No underwater/swimming sections
- Barriers (grey blocks) exist in levels but only Violet could dash through them (now Violet has no dash — barriers are currently impassable for everyone)
- Soft dirt tiles are decorative only (Mango's dig was removed)

### Technical
- Background `Math.random()` calls in building window generation mean backgrounds look different each time a level loads

### Content Ideas
- Narrative intro (comic panels)
- New Game+ with harder enemy speeds
- World-specific environmental hazards (wind, conveyor belts)
- Achievement system
- Time trial ghost (race your previous best)
