# Pokedex Detail Cards — Design

**Date:** 2026-07-15
**Status:** Approved

## Problem

The Pokedex (`showDex()`) is a bare grid: sprite, name, ×N duplicate badge.
The roster data only stores `id`/`name`/`type`, and even the type is never
shown. It doesn't feel like a real Pokedex — there is nothing to learn or
explore about a caught Pokemon.

## Goals

- Tapping a Pokemon in the dex grid opens a kid-friendly detail card with
  type, a fun fact, real height/weight, and its evolution chain.
- The dex grid itself is unchanged.
- Everything stays hardcoded in the single `index.html` — no network calls,
  no new files (kid-first rule: instant and offline-safe).

## Design

### Interaction

Tapping any `.dex-cell` opens a full-screen overlay card in the existing
visual language (rounded card, drop shadow, big ✖ close button). Tapping ✖
or the dark backdrop closes it. No other screens change.

### Detail card contents (top to bottom)

1. Big artwork via the existing `art(id)` helper.
2. Name + colored type badge, colors reused from `TYPE_COLORS` (first color
   of each pair as badge background).
3. One short kid-friendly fun fact in a speech-bubble style
   (e.g. "It stores lightning in its cheeks!").
4. 📏 height (m) and ⚖️ weight (kg) — real values from the games.
5. Evolution row: sprites joined by arrows, derived from the game's
   `EVOLUTIONS` map (inverted for "evolves from"). Shows the **game's**
   chains (Bulbasaur → Venusaur directly, matching what the kid sees
   happen). Eevee shows all three surprise outcomes. Pokemon with no
   evolution relation show no row.

### Data

New hardcoded `DEX_INFO` map: `id → { fact, h, w }` for all 69 roster
Pokemon (`POOL` + `RARES` + `EVOLVED`). Facts are original kid-friendly
one-liners (official flavor text can be dark); heights/weights are the real
game values. No changes to the roster arrays or `EVOLUTIONS`.

## Out of scope

- Type badges on the grid cells themselves.
- Stats, moves, cries, or shiny forms.
- Imperial units.

## Verification

Headless Chrome driver injected into a copy of index.html (repo convention):

- Tap a grid cell → card shows name, type badge, fact, height/weight.
- Base form (Pikachu) shows its forward chain; evolved form (Raichu) shows
  where it came from; no-evolution Pokemon (e.g. Lapras) shows no row;
  Eevee shows three outcomes.
- ✖ and backdrop both close the card.
- Data integrity: every id in `ALL` has a `DEX_INFO` entry with non-empty
  `fact` and positive `h`/`w`.
