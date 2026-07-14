# Kids' Pokemon Game — Design

**Date:** 2026-07-14
**Audience:** Two kids, ages 8 and 4, playing on a laptop/desktop browser.

## Goals

- Playable by a 4-year-old alone (no reading required to navigate — big picture buttons, icons everywhere).
- Engaging for an 8-year-old (collection building, light battle strategy, rare Pokemon).
- Zero frustration: catching always succeeds within a couple of throws, losing a battle costs nothing.
- Zero friction to launch: double-click a single file, no install, no server, no build step.

## Architecture

One self-contained `index.html` — vanilla JavaScript and CSS, no dependencies, no build tooling. Pokemon artwork loads at runtime from the PokeAPI sprites CDN (needs internet while playing; private home use only — the artwork is Nintendo's). Saves persist in the browser's `localStorage`.

## Screens

1. **Who's playing?** — two big trainer cards. First time, each kid picks an avatar Pokemon from six starters; the avatar is auto-added to their collection so Battle always works. Each kid has an independent save.
2. **Hub** — four big picture buttons: Catch 🌿, Battle ⚔️, Match 🎴, Pokedex 📕, plus a switch-player button. Header shows avatar and caught count.
3. **Catch** — a grass field with rustling patches. Click a patch → wild Pokemon pops out → click the Pokeball → throw + wobble animation → caught (85% first throw, guaranteed second). ~15% of encounters are rare Pokemon (Charizard, Mew, Snorlax, Dragonite, Lapras...). Caught Pokemon join the collection; confetti on success.
4. **Battle** — pick one of your caught Pokemon, face a random opponent. Three big move buttons (typed moves with emoji). Turn-based; the numbers favor the kid (player HP 100 / damage 15–28, enemy HP 70 / damage 6–14). Win → confetti. Lose → "Great try!" + rematch. No penalties ever.
5. **Match** — memory pairs using sprites of Pokemon the player has caught (padded from the general pool if the collection is small). Two sizes: 6 cards (little kid) or 12 cards (big kid), chosen by picture buttons.
6. **Pokedex** — grid of everything caught, with a ×N badge for duplicates and a caught counter.

## Data

- Curated pool of ~40 kid-favorite Gen-1 Pokemon hardcoded as `{id, name, type}`; sprite URL derived from id (`PokeAPI official-artwork` CDN path). No runtime API JSON calls — just image loads.
- Save shape: `{ profiles: [{avatar, caught: [ids...]}, {avatar, caught: [...]}] }` under one localStorage key.
- If a sprite fails to load (offline), a ❓ placeholder shows instead of a broken image.

## Non-goals

- No audio (deliberate — parents' sanity).
- No losing/trading Pokemon, no in-game text input, no networking beyond image loads.
- Not for publishing/distribution (copyrighted artwork).
