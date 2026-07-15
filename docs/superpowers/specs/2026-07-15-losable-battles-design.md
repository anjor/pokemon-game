# Losable Battles + Missing Evolutions — Design

**Date:** 2026-07-15
**Status:** Approved

## Problem

1. Battles are mathematically unloseable: player 100 HP vs enemy 70 HP, player
   deals 12–30/turn, enemy deals only 6–14/turn. The enemy can deal at most
   ~70 damage before fainting. The kids win every battle, so there is no
   tension and Tera has no tactical purpose.
2. Onix (and Chansey) cannot evolve — their evolutions (Steelix, Blissey) are
   Gen 2 and were skipped when the Gen-1-only evolution table was built. The
   8-year-old noticed ("Onix has had 3 battles but hasn't evolved").

## Goals

- Battles end in a loss roughly 1 in 4 times (parent-approved rate).
- Losses are dramatic and legible (a visible "power move" event), not a slow
  RNG grind.
- Losing stays penalty-free: gentle message, evolution win stars are kept
  (a loss simply doesn't add a star).
- Onix → Steelix and Chansey → Blissey become reachable evolutions.

## Design

### Battle rebalance (simulation-tuned)

| Knob | Before | After |
|---|---|---|
| Enemy HP | 70 | 100 |
| Enemy base damage | 6–14 | 12–18 |
| Enemy power move | — | 30% chance/turn, 26–34 damage |

Power move presentation: message "💥 {enemy} unleashes a POWERFUL attack!"
plus a bigger, longer shake animation (`big-hit` class) on the player box.

Tuning was validated by Monte Carlo simulation (100k battles, player picks a
random move each turn, no Tera): **25.0% loss rate, ~5.2 player turns per
battle**. Tera's 1.5× damage tilts the odds back toward the player, making it
a genuine comeback tool.

### New evolutions

- `EVOLUTIONS`: add `95: 208` (Onix → Steelix), `113: 242` (Chansey → Blissey).
- `EVOLVED`: add Steelix (id 208, type steel) and Blissey (id 242, type normal).
- `TYPE_COLORS`: add `steel` (silvery) so Steelix's evolution sparkles tint
  correctly. Steel has no move table; the existing `MOVES.normal` fallback
  covers it.
- Existing win counts already recorded for Onix/Chansey mean a kid with prior
  wins evolves on their next victory — an immediate payoff.

## Out of scope

- Evolutions for Lapras, Snorlax, Mew (they have none).
- Any penalty on loss (HP carryover, star resets, cooldowns).

## Verification

Headless Chrome driver injected into a copy of index.html (repo convention):
forced power-move turn shows the message and big-hit animation; a forced loss
shows the gentle lose message and does not increment win stars; Onix with one
prior win evolves into Steelix on the next victory and the collection updates.
