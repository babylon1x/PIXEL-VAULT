# AGENT BUILD PROMPT — Pixel Vault

Copy everything below into your coding agent as a single message.

---

## ROLE

You are a senior front-end engineer with deep experience shipping small, polished, single-file browser games. You do not add features you weren't asked for, you do not "improve" a spec by guessing at intent, and you do not simplify a spec by dropping requirements you find inconvenient. When a spec is explicit, you follow it exactly, including exact numbers, exact state names, and exact file structure. When a spec is silent on something you need to decide to write working code, you make the smallest reasonable decision, implement it, and flag it in a short "Assumptions" section at the end of your response — you do not stop and ask, and you do not silently invent scope.

## CONTEXT

I am building a small browser game called **Pixel Vault**. The full product spec is below, inside the `<spec>` tags. It has already been audited for ambiguity, contradictions, and scope creep — treat every line inside `<spec>` as a locked requirement, not a suggestion, unless this prompt's constraints section explicitly says otherwise.

The core loop, in one sentence: a player uploads an image, watches it hide inside one of 6 shuffling vault tiles, then has 10 seconds and 2 tries to click the correct tile before the vault explodes.

## TASK

Build the complete, working game as specified. Output a single `index.html` file that runs correctly when opened directly in a browser with no build step, no server, and no network dependency beyond one optional Google Fonts `<link>`.

## HARD CONSTRAINTS (non-negotiable, in priority order if any tension arises)

1. **One file.** Everything — HTML, CSS, JS — lives in one `index.html`. Inline `<style>` and `<script>` tags. No separate `.css` or `.js` files. No bundler, no build step, no npm, no package.json.
2. **Zero external libraries.** No React, Vue, jQuery, GSAP, anacreon, anime.js, or any animation/utility library. Vanilla JS and CSS only. The only permitted external network call is the Google Fonts `<link>` for `Press Start 2P`, and it must have a `monospace` fallback so the game still works if that request fails or is blocked.
3. **State machine is authoritative.** Implement exactly this state machine, using these exact state names, as the single source of truth for what input is accepted:
   ```
   IDLE → PREVIEW → HIDING → SHUFFLING → PLAYING → WIN
                                             ↓
                                           LOSE
   ```
   Input (clicks) must be locked/ignored in every state except `PLAYING`. Do not add, rename, split, or merge states. Do not add a separate `REVEAL` state — `PREVIEW` covers both the pixel-dissolve-in animation and the full 2.0s hold.
4. **Tile architecture — this is the most important constraint in this entire prompt.** Do NOT shuffle tiles by reordering DOM nodes or by changing CSS `order`. Tile DOM order must stay fixed for the lifetime of the page. Model each tile as a JS object `{ id, position, isTarget }`. Shuffling means: animate `transform: translate()` to visually swap two tiles' positions, and only after that animation completes, update the `position` values in the data model. `isTarget` is a property of the tile object, never derived from screen position or DOM index. If you catch yourself writing code that reorders `.children` or sets `style.order`, stop and rewrite using the transform + position-map approach instead.
5. **Canvas is for resize and composite only — not pixelation.** Use Canvas to: downscale the uploaded image to a max of 1200px on the longest side, and composite transparent PNGs onto a `#1a1a2e` background. Do not apply any pixelation, block-averaging, or color-quantization filter to the uploaded image. The pixel-art aesthetic comes from the UI (font, palette, tile chrome), not from degrading the user's photo — degrading it would make it harder for the player to recognize during the heist preview, which works against the core loop.
6. **Audio and scanlines are dev-set constants, not player-facing settings.** Implement them as `const AUDIO_ENABLED = false;` and `const SCANLINES = false;` (or equivalent) near the top of the script. Do not build any settings menu, toggle button, icon, or UI control for these. Flipping them is a code edit, not a runtime action.
7. **Timing must use `performance.now()` timestamps for the countdown timer**, not `setInterval` tick-counting, so the timer doesn't drift if the tab is backgrounded.
8. **No persistence of any kind.** No `localStorage`, no `sessionStorage`, no cookies, no server calls beyond the Google Fonts link. Refreshing the page must be a full reset to `IDLE`.

## PRODUCT SPEC (locked)

<spec>

### Core Loop
1. Player uploads one image (or uses a built-in pixel default).
2. Image is revealed full-screen for 2 seconds — the "heist preview" (this is the entire `PREVIEW` state: dissolve-in animation + hold).
3. Image animates *into* one random vault tile out of 6. The other 5 tiles are decoys.
4. All 6 tiles become visually identical (pixel vault doors).
5. Tiles shuffle (shell-game style) for ~2.4 seconds total (3 rounds × 0.8s each).
6. Timer starts **after shuffle ends, not before**: 10 seconds, 2 tries.
7. Click the tile hiding the image → WIN.
8. Click a decoy → tile cracks, shows static, one try left. That tile is now permanently out of play — clicking it again is ignored and does not consume the remaining try.
9. Second wrong click (on a different, uncracked tile) OR timer hits 0 → all tiles explode. LOSE.
10. Refresh = full reset. No persistence, no database, no server.

### Grid
3 columns × 2 rows = 6 tiles, fixed. Same grid on desktop and mobile, scaled proportionally via percentage layout (no separate mobile layout).

### Image Handling
- Accept: jpg, png, webp, gif (use first frame only for gif).
- Reject with a pixel-styled error toast: files >10MB, images <100×100px, non-image files.
- Downscale to max 1200px on the longest side before use (Canvas resize).
- Composite transparent PNGs onto `#1a1a2e` background (Canvas composite).
- Image data never leaves the browser — no upload to any server.
- If no image is uploaded, use a built-in pixel-art treasure chest as the default so the game is always playable.
- If multiple files are selected/dropped in quick succession, the last one wins; discard the rest.

### Visual Style
- Font: `Press Start 2P` via Google Fonts CDN link, `monospace` fallback.
- Palette — exactly these 8 colors, no gradients anywhere:
  - Background: `#1a1a2e`
  - Panel: `#16213e`
  - Accent hot: `#e94560`
  - Accent cool: `#0f3460`
  - Highlight: `#ffd369`
  - Text: `#f5f5f5`
  - Shadow: `#0a0a15`
  - Static: `#533483`
- Tiles: 3D pixel vault doors — beveled edges, 2 rivets, center keyhole graphic, 4px solid borders, zero `border-radius` anywhere in the entire app.
- Buttons: chunky, 4px border. Hover state = 1px translate + color invert. Pressed state = 2px translate down + color invert.
- Scanline overlay: single CSS rule, gated behind `SCANLINES` constant (see Hard Constraints §6), off by default.

### Animations (this is the complete list — do not add any animation not on this list)
| Event | Animation | Duration |
|---|---|---|
| Image reveal | Pixel-dissolve in | 2.0s |
| Image hides into tile | Scale + translate to target tile | 0.6s |
| Tile shuffle | `transform: translate()` swaps, ease-in-out | 0.8s × 3 rounds, 2–3 random tile swaps per round |
| Wrong click | Shake + crack overlay + fade to static | 0.3s |
| Correct click | Flip reveal + pixel-square burst | 0.5s |
| Explosion | 6 tiles → 8–16 chunks each, gravity fall, screen shake | 1.2s |
| Timer bar | Width decreases continuously; color shifts green → yellow → red | continuous |
| Button hover | 1px translate + color invert | 0.1s |
| Button press | 2px translate down | instant |

Win confetti = pixel squares (8×8px), never circles or any other shape.

### Audio (Web Audio API oscillators only — no audio files, no CDN, no downloads)
Gated behind `AUDIO_ENABLED` constant, off by default (see Hard Constraints §6).
- Click: square wave, 200Hz, 40ms
- Wrong: sawtooth wave, 120Hz, 150ms
- Correct: two square-wave notes, ascending pitch
- Explosion: white noise burst, 400ms
- Tick (final 3 seconds of timer): square wave, 800Hz, 30ms per tick

### Edge Cases (all must be handled exactly as specified)
| Case | Required behavior |
|---|---|
| No image uploaded | Use built-in pixel treasure chest |
| Image too large (>10MB) | Reject, pixel-styled error toast |
| Image too small (<100×100px) | Reject, pixel-styled error toast |
| Non-image file | Reject, pixel-styled error toast |
| GIF uploaded | Use first frame only |
| Transparent PNG | Composite onto `#1a1a2e` |
| Window resize | Percentage-based layout reflows; canvas rescales |
| Rapid/spam clicking | Blocked by state machine input lock (see Hard Constraint §3) |
| Click during `SHUFFLING` state | Ignored |
| Click on an already-cracked tile | Ignored, no try consumed (see Core Loop step 8) |
| Timer hits exactly 0 while a click is mid-processing | Timer wins — treat as LOSE |
| Tab backgrounded during countdown | No drift — timer driven by `performance.now()` timestamps, not tick counts |
| Multiple rapid uploads before game starts | Last file wins, earlier ones discarded |
| Page refresh at any point | Full reset to `IDLE` state |

### Copy (exact strings, use verbatim)
- Win: `VAULT CRACKED!` — buttons: `Play Again` / `New Image`
- Lose: `VAULT BLOWN!` — buttons: `Try Again` / `New Image`
- Timer warning at 3 seconds remaining: `HURRY!` (flashing)
- Wrong click popup: `DECOY!`

</spec>

## OUT OF SCOPE — DO NOT IMPLEMENT ANY OF THE FOLLOWING

Do not add these even if they seem like natural, small, or helpful additions. Each one is a deliberate exclusion, not an oversight:

- Multiplayer of any kind
- Leaderboard or score tracking
- User accounts or authentication
- Sharing (social, link, or otherwise)
- Saving/loading game state
- Difficulty levels or difficulty settings
- Power-ups or special abilities
- Themes, skins, or palette swaps
- Any mobile-specific gesture beyond a plain tap-as-click
- Any backend, API call, or database
- Analytics or telemetry of any kind
- PWA manifest or service worker
- A "peek" button or any hint mechanic
- Any animation not listed in the Animations table above
- Any settings menu or toggle UI for audio or scanlines (these are code constants — see Hard Constraint §6)

If you find yourself about to add something from this list "because it would only take a minute" — do not. Stop and omit it.

## ESCAPE HATCH — WHEN THE SPEC IS SILENT

The spec above has already been audited and is intentionally exhaustive. If you hit a genuine gap it does not cover (for example: exact pixel dimensions of the built-in treasure chest artwork, or the precise easing curve to use for a transform), do not stop and ask me — make the smallest, most conservative decision that satisfies the surrounding constraints and the visual style section, implement it, and list it under an "Assumptions" heading at the end of your response so I can review it. Do not use a gap in the spec as license to add scope from the Out of Scope list above.

## OUTPUT FORMAT

1. One complete `index.html` file, fully working, pasteable and runnable with no edits.
2. Code should be organized with clear section comments (e.g. `// ---- STATE MACHINE ----`, `// ---- TILE SHUFFLE LOGIC ----`, `// ---- CANVAS IMAGE PROCESSING ----`) so the structure is easy to audit against this spec.
3. After the code, include:
   - A short **"Spec compliance checklist"** confirming, line by line, that each Hard Constraint (1–8) is satisfied in the code you wrote.
   - An **"Assumptions"** section (only if you had to make any judgment calls per the Escape Hatch above; omit this section entirely if there were none — do not pad it).

## DEFINITION OF DONE

Before returning your answer, verify silently against this checklist and only return code that passes all of it:

- [ ] Tiles are never reordered in the DOM; shuffling is transform-only with a separate position-map update.
- [ ] `isTarget` lives on the tile object, never inferred from position or DOM index.
- [ ] Clicks are ignored in every state except `PLAYING`.
- [ ] Timer uses `performance.now()`, not `setInterval` tick counts.
- [ ] `AUDIO_ENABLED` and `SCANLINES` exist as constants, default `false`, with no UI to change them.
- [ ] No pixelation/quantization filter is applied to the uploaded image — only resize and composite.
- [ ] No feature from the Out of Scope list appears anywhere in the code.
- [ ] The file runs correctly when opened directly from disk in a browser (double-click, no server).
- [ ] All copy strings match the spec verbatim (`VAULT CRACKED!`, `VAULT BLOWN!`, `HURRY!`, `DECOY!`, button labels).
- [ ] All 8 palette colors are used and no gradient or `border-radius` appears anywhere.
