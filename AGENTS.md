# AGENTS.md

This is a static HTML games repository (three self-contained browser games). No build tools, package manager, tests, linting, or typechecking are configured.

## Overview

The repo contains three vanilla JS + Tailwind CDN games:
- `emoji_mahjong.html` - Emoji Mahjong Solitaire with custom pan/zoom and 3D tile rendering
- `emoji_solitaire.html` - Emoji-themed Klondike Solitaire (mobile-optimized)
- `onet_emoji.html` - Onet/Connect-style matching game with timer

All games load Tailwind via CDN (`https://cdn.tailwindcss.com`) and use Google Fonts. They are designed to run directly in a browser; no server or compilation is required.

## How to test/run

These are static files. To test, open them in a browser. Options:
- Open the HTML file directly: `open emoji_mahjong.html` (macOS) or double-click in file explorer
- Serve locally (recommended for mobile testing): `python3 -m http.server 8000` then visit `http://localhost:8000/<filename>`

For mobile/touch behavior, use `localhost` or a local IP (some features depend on proper viewport and touch events). Games disable default touch behaviors (e.g., `touch-action: none`) for custom gestures.

## Editing guidance

- Keep changes self-contained in the HTML file you're editing (inline CSS + JS). No external JS/CSS modules are used.
- Preserve existing CDN dependencies (`tailwindcss.com`, Google Fonts). Don't add build tooling unless explicitly requested.
- Respect mobile-first constraints: viewport meta, `touch-action`, `user-select: none`, `-webkit-tap-highlight-color: transparent` are intentional.
- When modifying game logic, maintain existing variable naming and structure; changes are localized to a single HTML file.

## Verification

There are no automated tests, linters, or CI. Verification is manual: open the affected game in a browser and exercise the relevant flows (game start, interactions, win/lose states, mobile gestures if changed).

## Notes

- Not a git repo.
- No environment variables, secrets, or external services required.
- Assets are all emojis/inline CSS (no image/asset files).
- Avoid adding comments unless explicitly requested (repo follows the global instruction to not add comments).
