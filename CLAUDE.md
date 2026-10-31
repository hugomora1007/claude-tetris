# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

There is no build, package manager, linter, or test runner. The game is static files.

- Run: open `index.html` directly, or serve the folder (`python3 -m http.server 8000`, `npx serve .`) and open `http://localhost:8000`.
- Tests: none exist, so there is no single-test command. Verify changes by playing in the browser; browser console errors are the main signal.

## Architecture

Three files, no modules, no globals beyond the script scope:

- `index.html` loads `style.css` and `game.js` (the script sits at the end of `<body>`, so DOM lookups at the top of `game.js` run after the elements exist). It defines the board `<canvas id="board">` (300×600), the `next-canvas` preview, the score/lines/level spans, and the `#overlay` used for PAUSA and GAME OVER. The element IDs are the contract with `game.js`; renaming one breaks the script silently at startup.
- `game.js` is the whole game, in one `'use strict'` file. Everything is a top-level `function` or `const`, and the mutable state lives in a single `let` line (`board, current, next, score, lines, level, paused, gameOver, lastTime, dropAccum, dropInterval, animId`). `init()` resets all of it and is also the restart button's handler.
- `style.css` is presentation only; it does not affect game logic.

Data model:
- The board is `ROWS` arrays of `COLS` numbers, where `0` is empty and `1..7` is a piece type that indexes both `COLORS` and `PIECES`. Piece shapes are small square matrices using the same numbering, so a shape cell's value is written straight into the board on lock (`merge`).
- A piece is `{ type, shape, x, y }`. `x,y` is the top-left of the shape matrix on the board, and `y` may be negative at spawn.

Control flow:
- A single `requestAnimationFrame` loop (`loop`) accumulates `dt` and steps the piece down when `dropAccum >= dropInterval`. Every other mutation (keys, hard drop) happens in the `keydown` handler and then calls `updateHUD()`. Drawing is a full redraw each frame (`draw`), with the ghost piece drawn at 0.2 alpha.
- Locking flows `lockPiece()` → `merge()` → `clearLines()` → `spawn()`. `spawn()` promotes `next` to `current`, rolls a new `next`, and calls `endGame()` if the new piece collides immediately.
- Collision (`collide`) is the single source of truth for legality. Movement, rotation (`tryRotate`, which tries horizontal kicks 0, ±1, ±2), soft drop, and the ghost position (`ghostY`) all go through it.

## Gotchas

- `endGame()` calls `cancelAnimationFrame`, but when the game ends from inside `loop()` (the lock happens in the loop's gravity step), `loop()` still runs `draw()` and re-requests a frame afterwards. Do not assume the loop has stopped after game over; guard new per-frame logic with `gameOver`.
- `clearLines()` scans bottom-up and does `r++` after removing a row, which cancels the loop's `r--` so the same index is re-checked (stacked full rows clear correctly). Keep that when editing it.
- Scoring uses the level at the time of clearing, and `level` is recomputed from `lines` (`floor(lines / 10) + 1`) before `dropInterval` is updated.
- Pause is handled before the game-state guard in `keydown` (`KeyP` always works); all other keys are ignored while paused or after game over.
- Strings in the UI and README are in Spanish; code comments and identifiers are in English. Keep new user-facing text in Spanish to match.
- `README.md` is the user-facing doc (in Spanish) and has a "Personalización" section listing the tunable constants. Update it when those constants or controls change.
