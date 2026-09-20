# PIXEL VAULT

A single-file, shell-game-style browser memory game. Upload an image (or use the built-in pixel treasure chest), watch it hide inside one of six shuffling vault tiles, then find it within 10 seconds and 2 tries before the vault explodes.

Everything — HTML, CSS, and JavaScript — lives in one self-contained `index.html`. There is no build step, no server, and no dependencies beyond an optional Google Fonts link (with a local `monospace` fallback). Open the file in any modern browser and play.

---

## Table of Contents

1. [Workspace Layout](#workspace-layout)
2. [How to Run](#how-to-run)
3. [Tech Stack](#tech-stack)
4. [Core Game Loop](#core-game-loop)
5. [State Machine](#state-machine)
6. [Game Specs](#game-specs)
   - [Grid](#grid)
   - [Image Handling](#image-handling)
   - [Visual Style](#visual-style)
   - [Animations](#animations)
   - [Audio](#audio)
   - [Edge Cases](#edge-cases)
   - [Copy Strings](#copy-strings)
7. [Configuration Constants](#configuration-constants)
8. [Architecture Notes](#architecture-notes)
9. [Out of Scope (intentional omissions)](#out-of-scope-intentional-omissions)

---

## Workspace Layout

```
PIXEL VAULT/
├── index.html                 ← The entire game (HTML + CSS + JS, single file)
├── README.md                  ← This file
├── docs/
│   └── pixel-vault-agent-prompt.md   ← The original locked product spec
└── Sound effects/
    ├── On Button click.mp3    ← Button click sound
    ├── On Game Open.mp3       ← Game load/open sound
    ├── On Game Over.mp3       ← Loss (explosion / game over) sound
    ├── On Hover.mp3           ← Hover sound
    └── On Right guess.mp3     ← Correct guess / win sound
```

---

## How to Run

### Prerequisites

- Any modern browser (Chrome, Edge, Firefox, Safari). The game uses fetch-free ES6+ features, `<canvas>`, and the Web Audio API.
- No Node.js, no package manager, no build tools, no internet connection required (the `Press Start 2P` font link degrades gracefully to `monospace`).

### Run it

**Option A — double click (recommended):**

1. Open this repo folder
2. Double-click `index.html`. It opens in your default browser and is immediately playable.

**Option B — command line:**

```powershell
start "" ".\index.html"
```

or

```bash
cd /path/to/pixel-vault && start index.html
```

or, from inside the repo folder:

```
start index.html
```

**Option C — serve it locally (optional, not required):**

Any static server works fine since the game needs no backend:

```powershell
# from the PIXEL VAULT folder
python -m http.server 8000
# then open http://localhost:8000/
```

> **Important:** opening via a server (or the browser's file:// protocol from a folder where `Sound effects/` sits next to `index.html`) is what lets the mp3 effects load. If you copy `index.html` somewhere by itself, keep a `Sound effects/` folder next to it with the same file names.

### How to play

1. On the start screen click **UPLOAD IMAGE**, **USE PIXEL DEFAULT**, or drag-and-drop an image file onto the window.
2. Watch the 2-second **heist preview** — the image pixel-dissolves in full-screen.
3. The image animates into one of six random vault doors; all six then look identical.
4. The tiles shuffle shell-game style for ~2.4 seconds.
5. The 10-second timer starts (after the shuffle). Click the tile hiding your image.
6. **Win** when you pick the right tile. **Lose** if a second wrong tile is cracked, or the timer hits 0.
7. From the win/loss panel:
   - **Play Again** / **Try Again** — new round with the same image already in memory (no re-upload).
   - **New Image** — back to the upload screen.

---

## Tech Stack

| Layer | Technology | Notes |
|---|---|---|
| Markup | HTML5 | Entirely inline in `index.html` |
| Styling | Vanilla CSS3 | No preprocessor, no framework, no `border-radius` |
| Logic | Vanilla JavaScript (ES6+) | No libraries, no framework |
| Rendering | `<canvas>` 2D | Image resize/composite + pixel-dissolve preview only |
| Animation | CSS transitions/keyframes + `requestAnimationFrame` | Transform-only shuffles, canvas/dissolve and FX driven by rAF |
| Audio | HTML `<audio>` elements (effects) + Web Audio API (micro-SFX) | mp3 files from `Sound effects/` |
| Fonts | Google Fonts `Press Start 2P` | With `monospace` fallback; single allowed network call |
| State | None external | Refresh = full reset; zero persistence |

**Not used** (deliberately): frameworks, bundlers, npm, backend, database, `localStorage`/`sessionStorage`, cookies, service workers, or any analytics.

---

## Core Game Loop

1. Player uploads one image, or uses the built-in pixel chest default.
2. Image is revealed full-screen for 2 seconds — the **heist preview** (entirely the `PREVIEW` state: dissolve-in animation + hold).
3. Image animates *into* one random vault tile out of 6; the other 5 tiles are decoys.
4. All 6 vault doors look identical.
5. Tiles shuffle for **~2.4 seconds** total (3 rounds &times; 0.8s).
6. Timer starts **after the shuffle**: **10 seconds, 2 tries**.
7. Correct tile clicked → **WIN** (flip reveal + pixel burst + confetti).
8. Decoy clicked → tile cracks, shows static, 1 try remaining; that tile is permanently out of play (further clicks ignored, no try consumed).
9. Second wrong click on a different uncracked tile, **or** the timer hits 0 → **all 6 tiles explode** → **LOSE**.
10. Refresh anywhere = full reset to `IDLE`.

---

## State Machine

The state machine is the single authority for what input is accepted. Input (clicks) is locked/ignored in every state except `PLAYING`.

```
IDLE → PREVIEW → HIDING → SHUFFLING → PLAYING → WIN
                                              ↓
                                            LOSE
```

| State | Meaning | Input allowed |
|---|---|---|
| `IDLE` | Start/upload screen | Upload / use default only |
| `PREVIEW` | 2.0s full-screen pixel-dissolve reveal | None |
| `HIDING` | Image animates into the target tile (0.6s) | None |
| `SHUFFLING` | Shell-game shuffle (~2.4s) | None (clicks ignored) |
| `PLAYING` | Timer running, tiles clickable | Tile clicks only |
| `WIN` | Correct guess | Panel buttons |
| `LOSE` | Misses exhausted or timeout | Panel buttons |

There is intentionally **no** separate `REVEAL` state — `PREVIEW` covers the dissolve and the hold.

---

## Game Specs

### Grid

- **3 columns × 2 rows = 6 tiles**, fixed.
- Same layout on desktop and mobile — scaled proportionally via percentage layout (no separate mobile view, no reordering).
- Tile geometry: 30.5% wide, 45% tall; `left = col × 33.3333 + 1.6%`, `top = row × 50 + 2.5%`.

### Image Handling

- **Accepted:** `jpg`, `png`, `webp`, `gif` (GIF uses the **first frame** only).
- **Rejected with a pixel-styled error toast:**
  - Files **&gt;10MB**
  - Images **&lt;100×100px**
  - Non-image files / undecodable images
- **Downscaled** to max **1200px** on the longest side via Canvas.
- Transparent PNGs are **composited onto `#1a1a2e`** via Canvas.
- **No pixelation/quantization** is applied to the uploaded image — only resize + composite. The pixel aesthetic comes from the UI, not degrading the photo.
- Image data **never leaves the browser** — no server upload, no persistence.
- If no image is loaded, a built-in **pixel-art treasure chest** is drawn procedurally so the game is always playable.
- Multiple rapid uploads → **last one wins**, earlier ones discarded.

### Visual Style

- **Font:** `Press Start 2P` (Google Fonts CDN) with `monospace` fallback.
- **Palette — exactly 8 colors, no gradients anywhere:**
  - Background `#1a1a2e`
  - Panel `#16213e`
  - Accent hot `#e94560`
  - Accent cool `#0f3460`
  - Highlight `#ffd369`
  - Text `#f5f5f5`
  - Shadow `#0a0a15`
  - Static `#533483`
  - *(The timer bar's green is a locked animation-spec exception — the bar shifts green → yellow → red — and is documented as such.)*
- **Tiles:** 3D pixel vault doors — beveled edges, 2 rivets, center SVG keyhole, 4px solid borders, **zero `border-radius` anywhere**.
- **Buttons:** chunky, 4px border; hover = 1px translate + color invert; pressed = 2px translate down + invert.
- **Scanlines:** single CSS rule gated behind the `SCANLINES` constant, off by default (code edit to enable, no UI).
- **Win confetti:** 8×8px squares only — never circles or other shapes.

### Animations

| Event | Animation | Duration |
|---|---|---|
| Image reveal | Pixel-dissolve in | 2.0s |
| Image hides into tile | Scale + translate to target tile | 0.6s |
| Tile shuffle | `transform: translate()` swaps, ease-in-out | 0.8s × 3 rounds, 2–3 random swaps per round (~2.4s) |
| Wrong click | Shake + crack overlay + fade to static | 0.3s |
| Correct click | Flip reveal + pixel-square burst | 0.5s |
| Explosion | 6 tiles → 8–16 chunks each, gravity fall, screen shake | 1.2s |
| Timer bar | Width decreases continuously; color shifts green → yellow → red | continuous |
| Button hover | 1px translate + color invert | 0.1s |
| Button press | 2px translate down | instant |

### Audio

Real mp3 effects load from the `Sound effects/` folder next to `index.html`:

| Trigger | File |
|---|---|
| Button click | `On Button click.mp3` |
| Game load / open | `On Game Open.mp3` |
| Game over / wrong final guess | `On Game Over.mp3` |
| Hover on buttons and tiles | `On Hover.mp3` |
| Correct guess / win | `On Right guess.mp3` |

Micro-SFX that have no assigned file are generated with the Web Audio API (oscillator tones): the **wrong-guess decoy** buzz and the **HURRY final-3s tick**.

> **Browser autoplay policy:** the game-open sound is attempted on load, and retried automatically on the first pointer/keypress if the browser blocked autoplay. No sound plays until a user gesture is allowed.

### Edge Cases

| Case | Behavior |
|---|---|
| No image uploaded | Built-in pixel treasure chest used |
| Image &gt;10MB | Rejected, error toast |
| Image &lt;100×100px | Rejected, error toast |
| Non-image file | Rejected, error toast |
| GIF uploaded | First frame only |
| Transparent PNG | Composited onto `#1a1a2e` |
| Window resize | Percentage layout reflows; preview canvas rescales; tile bases re-snapped |
| Rapid/spam clicking | Blocked by state machine input lock |
| Click during `SHUFFLING` | Ignored |
| Click on an already-cracked tile | Ignored, no try consumed |
| Timer hits 0 mid-click | Timer wins → LOSE |
| Tab backgrounded during countdown | No drift — timer uses `performance.now()` timestamps |
| Multiple rapid uploads | Last file wins |
| Page refresh at any point | Full reset to `IDLE` |

### Copy Strings (verbatim)

- Win: `VAULT CRACKED!` — buttons `Play Again` / `New Image`
- Lose: `VAULT BLOWN!` — buttons `Try Again` / `New Image`
- Timer warning at 3s remaining: `HURRY!` (flashing)
- Wrong click popup: `DECOY!`

---

## Configuration Constants

All located near the top of the `<script>` block in `index.html`:

| Constant | Default | Purpose |
|---|---|---|
| `AUDIO_ENABLED` | `true` | Master switch for all sound. `false` mutes everything. |
| `SCANLINES` | `false` | Enables the CRT scanline overlay via a single CSS rule. |
| `TIMER_SECS` | `10` | Countdown length per round. |
| `TRIES_MAX` | `2` | Wrong guesses allowed per round. |
| `SHUFFLE_ROUNDS` | `3` | Shuffle rounds per round. |
| `ROUND_MS` | `800` | Duration of each shuffle round (0.8s). |
| `PREVIEW_MS` | `1500` | Pixel-dissolve animation portion of the preview. |
| `PREVIEW_HOLD` | `500` | Hold after dissolve (together = 2.0s `PREVIEW`). |
| `HIDE_MS` | `600` | Image-hides-into-tile animation length. |
| `TEXTURE_SIZE` | `128` | Canvas resolution for generated static/crack textures. |

---

## Architecture Notes

- **Tile model:** each tile is a JS object `{ id, position, isTarget, cracked, el }`. `isTarget` lives on the object — it is never derived from screen position or DOM index.
- **DOM order is fixed for the page lifetime** — tiles are built once (`buildTiles()`), never reordered, and CSS `order` is never touched.
- **Transform-only shuuffle:** a swap animates `transform: translate()` on two tiles, and only after that animation completes are the `position` values in the data model updated, then bases are re-applied and transforms cleared. `resetTiles()` resets positions, target flags, crack/win classes, and all inline tile styles before each round.
- **Timer:** `performance.now()` timestamps + `requestAnimationFrame`, so it stays accurate even when the tab is backgrounded; on zero it also force-wins against an in-flight click.
- **No persistence:** refresh is a hard reset to `IDLE`.
- **`Try Again` / `Play Again`** start a new round with the image already in memory — no re-upload, no page reload.

---

## Out of Scope (intentional omissions)

The following were deliberately excluded from the game and will not be added:

- Multiplayer, leaderboards, scores, user accounts, or authentication — none
- Social sharing, saving/loading game state, or any persistence
- Difficulty levels, power-ups, hint/peek buttons, or any second chance outside the 2-try rule
- Skins/palette swaps, settings menus, or toggles for audio/scanlines beyond the code constants
- Any animation beyond the table above, any mobile gesture other than tap-as-click
- Backend, API calls, databases, analytics, telemetry, PWA/service worker — none