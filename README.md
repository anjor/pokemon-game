# Gotta Catch 'Em! ⚡

A simple Pokemon game for young kids (built for ages 4 and 8) — one
self-contained HTML file, no install, no build step.

**Play it:** open `index.html` in a browser, or visit the GitHub Pages site.

## What's in the game

- **Two trainer profiles** — each kid picks an avatar buddy, types their name,
  and gets their own collection (saved in the browser's localStorage).
- **Catch** 🌿 — click rustling grass, throw a Pokeball, build your collection.
  Rare Pokemon (Charizard, Mew, Snorlax...) appear occasionally.
- **Battle** ⚔️ — turn-based battles with typed moves, kid-favored difficulty,
  and no penalties for losing. Once per battle you can **Terastallize** 💎 for
  a crystal transformation and boosted moves.
- **Match** 🎴 — memory pairs using the Pokemon you've caught, in small (easy)
  or big grids.
- **Pokedex** 📕 — everything you've caught, with duplicate badges. Tap a Pokemon for its type, a fun fact, its real size, and its evolution chain. Some
  favorites have special-form chips — tap to watch them **Mega Evolve** ⚡,
  **Gigantamax** 🔴, or show off an **Alolan/Galarian** regional form 🗺️.

Designed so a 4-year-old can navigate alone: big picture buttons, minimal
reading, and a reset that requires holding a button for 3 seconds behind a
very loud warning screen.

## Notes

- Pokemon artwork loads at runtime from the
  [PokeAPI sprites](https://github.com/PokeAPI/sprites) CDN, so the game needs
  an internet connection while playing.
- Pokemon is © Nintendo / Creatures Inc. / GAME FREAK inc. This is a
  non-commercial fan project; the code is MIT-licensed (see LICENSE), the
  artwork is not.
