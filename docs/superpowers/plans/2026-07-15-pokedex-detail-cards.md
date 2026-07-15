# Pokedex Detail Cards Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Tapping a Pokemon in the Pokedex grid opens a detail card showing its type badge, a kid-friendly fun fact, real height/weight, and its evolution chain.

**Architecture:** Everything lives in the single `index.html` (vanilla JS/CSS, no build, no dependencies). A new hardcoded `DEX_INFO` map (`id → { fact, h, w }`) supplies the per-Pokemon data; evolution chains are derived at render time from the existing `EVOLUTIONS` map. The detail card is a fixed-position overlay toggled by a CSS class, opened from a click handler added to each dex grid cell.

**Tech Stack:** Vanilla HTML/CSS/JS in one file. Verification via a driver script injected into a copy of `index.html`, run under headless Chrome (repo convention — there is no test framework).

**Spec:** `docs/superpowers/specs/2026-07-15-pokedex-detail-cards-design.md`

## Global Constraints

- Single file: all changes go in `index.html`. No new runtime files, no network calls, no libraries.
- Kid-first: facts must be short, cheerful, readable-aloud; no dark lore (e.g. nothing about Cubone's mother). Metric units only.
- Do not commit sprite images to the repo (MIT license covers code only; artwork stays on the PokeAPI CDN via the existing `art(id)` helper).
- Follow existing visual language: white `var(--card)` cards, rounded corners, hard drop shadows `0 Npx 0 rgba(0,0,0,.1x)`, emoji icons.
- Work on branch `feature/pokedex-detail-cards` (already created).
- Line numbers below refer to `index.html` at commit `bd037df`.

## Test harness (used by every task)

Each task's test is a `driver.js` written to the session scratchpad, injected before `</body>` of a copy of `index.html`, and run under headless Chrome. The driver appends a `<pre id="test-log">` with `TEST: <name> PASS|FAIL` lines; the runner greps them out of `--dump-dom` output.

Runner script — each task recreates it verbatim (idempotent):

```bash
SCRATCH="/private/tmp/claude-501/-Users-anjor-repos-anjor-pokemon-game/8118ea75-4c35-46f3-bd73-23b885120500/scratchpad"
mkdir -p "$SCRATCH"
cat > "$SCRATCH/run-dex-test.sh" <<'SH'
#!/bin/bash
# Usage: run-dex-test.sh  (expects driver.js in scratchpad, run from repo root)
set -e
SCRATCH="/private/tmp/claude-501/-Users-anjor-repos-anjor-pokemon-game/8118ea75-4c35-46f3-bd73-23b885120500/scratchpad"
python3 - "$SCRATCH" <<'PYEOF'
import sys, pathlib
scratch = pathlib.Path(sys.argv[1])
html = pathlib.Path("index.html").read_text()
driver = (scratch / "driver.js").read_text()
assert "</body>" in html
(scratch / "test.html").write_text(
    html.replace("</body>", "<script>\n" + driver + "\n</script>\n</body>"))
PYEOF
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --virtual-time-budget=8000 \
  --dump-dom "file://$SCRATCH/test.html" 2>/dev/null | grep -o "TEST: [^<]*"
SH
chmod +x "$SCRATCH/run-dex-test.sh"
```

Every driver starts with this preamble (appends results even if an assertion throws):

```js
const LOG = [];
const t = (name, ok) => LOG.push(`TEST: ${name} ${ok ? "PASS" : "FAIL"}`);
const finish = () => {
  const pre = document.createElement("pre");
  pre.id = "test-log";
  pre.textContent = LOG.join("\n");
  document.body.appendChild(pre);
};
```

---

### Task 1: DEX_INFO data map

**Files:**
- Modify: `index.html` — insert after the `TYPE_COLORS` closing `};` (line 778), before `const art = ...` (line 780)
- Test: `$SCRATCH/driver.js` (scratchpad, not committed)

**Interfaces:**
- Consumes: existing globals `ALL` (array of `{id, name, type}`), `byId(id)`.
- Produces: global `const DEX_INFO = { <id>: { fact: string, h: number /* meters */, w: number /* kg */ }, ... }` with exactly one entry per Pokemon in `ALL` (69 entries). Task 2 reads `DEX_INFO[id].fact/.h/.w`.

- [ ] **Step 1: Write the failing test**

Create the runner (see "Test harness" above), then write `$SCRATCH/driver.js`:

```js
const LOG = [];
const t = (name, ok) => LOG.push(`TEST: ${name} ${ok ? "PASS" : "FAIL"}`);
const finish = () => {
  const pre = document.createElement("pre");
  pre.id = "test-log";
  pre.textContent = LOG.join("\n");
  document.body.appendChild(pre);
};
try {
  const missing = ALL.filter(p => !DEX_INFO[p.id]);
  t("every roster id has a DEX_INFO entry", missing.length === 0);
  const bad = ALL.filter(p => {
    const d = DEX_INFO[p.id];
    return !d || typeof d.fact !== "string" || d.fact.trim().length < 10
      || !(d.h > 0) || !(d.w > 0);
  });
  t("every entry has non-empty fact and positive h/w", bad.length === 0);
  const orphans = Object.keys(DEX_INFO).filter(id => !byId(Number(id)));
  t("no DEX_INFO entries for unknown ids", orphans.length === 0);
} catch (e) {
  LOG.push("TEST: exception FAIL " + e.message);
}
finish();
```

- [ ] **Step 2: Run test to verify it fails**

Run (from repo root): `"$SCRATCH/run-dex-test.sh"`
Expected output: `TEST: exception FAIL DEX_INFO is not defined`

- [ ] **Step 3: Add the DEX_INFO map**

Insert into `index.html` between the `TYPE_COLORS` block and the `const art` line:

```js
// Pokedex detail info: kid-friendly fact, height h (m), weight w (kg).
// Facts are original one-liners (official flavor text can be dark);
// heights/weights are the real game values.
const DEX_INFO = {
  1:   { fact: "It grows a plant on its back from the day it's born!", h: 0.7, w: 6.9 },
  3:   { fact: "Its giant flower smells amazing after rainy days!", h: 2.0, w: 100.0 },
  4:   { fact: "The flame on its tail shows how it's feeling!", h: 0.6, w: 8.5 },
  6:   { fact: "It can breathe fire hot enough to melt rocks!", h: 1.7, w: 90.5 },
  7:   { fact: "It hides in its shell and squirts water at foes!", h: 0.5, w: 9.0 },
  9:   { fact: "The water cannons on its shell blast like rockets!", h: 1.6, w: 85.5 },
  10:  { fact: "It munches leaves all day and dreams of flying!", h: 0.3, w: 2.9 },
  12:  { fact: "Its wings are covered in sparkly waterproof scales!", h: 1.1, w: 32.0 },
  16:  { fact: "It's very gentle and loves napping in tall grass!", h: 0.3, w: 1.8 },
  18:  { fact: "It can fly at the speed of sound — whoosh!", h: 1.5, w: 39.5 },
  25:  { fact: "It stores lightning in its little red cheeks!", h: 0.4, w: 6.0 },
  26:  { fact: "Its tail works like a lightning rod in storms!", h: 0.8, w: 30.0 },
  27:  { fact: "It rolls into a ball to bounce away from danger!", h: 0.6, w: 12.0 },
  28:  { fact: "It can curl into a spiky ball like a pinecone!", h: 1.0, w: 29.5 },
  35:  { fact: "Some say it dances under the full moon!", h: 0.6, w: 7.5 },
  36:  { fact: "It walks so quietly, its footsteps make no sound!", h: 1.3, w: 40.0 },
  37:  { fact: "It's born with one white tail that splits into six!", h: 0.6, w: 9.9 },
  38:  { fact: "Its nine tails are said to hold magical power!", h: 1.1, w: 19.9 },
  39:  { fact: "Its lullaby song makes everyone fall asleep!", h: 0.5, w: 5.5 },
  40:  { fact: "Its body is soft and squishy like a balloon!", h: 1.0, w: 12.0 },
  43:  { fact: "By day it hides in soil; at night it goes for walks!", h: 0.5, w: 5.4 },
  45:  { fact: "It has the biggest flower petals in the world!", h: 1.2, w: 18.6 },
  50:  { fact: "Nobody knows what its bottom half looks like!", h: 0.2, w: 0.8 },
  51:  { fact: "Three Digletts that dig super-deep tunnels together!", h: 0.7, w: 33.3 },
  52:  { fact: "It loves anything shiny, especially coins!", h: 0.4, w: 4.2 },
  53:  { fact: "The jewel on its head glitters when it prowls!", h: 1.0, w: 32.0 },
  54:  { fact: "It always has a headache — quack, ouch!", h: 0.8, w: 19.6 },
  55:  { fact: "It swims faster than a champion swimmer!", h: 1.7, w: 76.6 },
  58:  { fact: "A loyal puppy that barks at anything sneaky!", h: 0.7, w: 19.0 },
  59:  { fact: "It can run a whole day and night without stopping!", h: 1.9, w: 155.0 },
  60:  { fact: "The swirl on its tummy is its insides showing through!", h: 0.6, w: 12.4 },
  62:  { fact: "A super swimmer that never gets tired!", h: 1.3, w: 54.0 },
  63:  { fact: "It sleeps 18 hours a day — even while teleporting!", h: 0.9, w: 19.5 },
  65:  { fact: "Its brain is so big it remembers everything ever!", h: 1.5, w: 48.0 },
  66:  { fact: "It lifts boulders like they're beach balls!", h: 0.8, w: 19.5 },
  68:  { fact: "Four arms mean four punches at once!", h: 1.6, w: 130.0 },
  74:  { fact: "It looks just like a rock — watch your step!", h: 0.4, w: 20.0 },
  76:  { fact: "It rolls down mountains like a boulder — rumble!", h: 1.4, w: 300.0 },
  77:  { fact: "Its mane is made of fire but it won't burn friends!", h: 1.0, w: 30.0 },
  78:  { fact: "It gallops almost as fast as a race car!", h: 1.7, w: 95.0 },
  79:  { fact: "It's so relaxed it takes 5 seconds to feel anything!", h: 1.2, w: 36.0 },
  80:  { fact: "A Shellder bit its tail and never let go!", h: 1.6, w: 78.5 },
  81:  { fact: "It floats using magnets and beeps softly!", h: 0.3, w: 6.0 },
  82:  { fact: "Three Magnemite stuck together with magnet power!", h: 1.0, w: 60.0 },
  90:  { fact: "Its shell is harder than diamond — snap snap!", h: 0.3, w: 4.0 },
  91:  { fact: "Its shell can block any attack — even rockets!", h: 1.5, w: 132.5 },
  92:  { fact: "It's made of gas and loves playing spooky tricks!", h: 1.3, w: 0.1 },
  94:  { fact: "It hides in shadows and giggles in the dark!", h: 1.5, w: 40.5 },
  95:  { fact: "A rock snake longer than a school bus!", h: 8.8, w: 210.0 },
  104: { fact: "It wears a bone helmet and never takes it off!", h: 0.4, w: 6.5 },
  105: { fact: "It throws bones like boomerangs — they come back!", h: 1.0, w: 45.0 },
  113: { fact: "It shares its yummy eggs with hurt Pokemon!", h: 1.1, w: 34.6 },
  116: { fact: "It shoots ink at anything that gets too close!", h: 0.4, w: 8.0 },
  117: { fact: "It swims backwards — and it's fast at it!", h: 1.2, w: 25.0 },
  120: { fact: "Its red core blinks like a little star at night!", h: 0.8, w: 34.5 },
  121: { fact: "Its core glows in seven rainbow colors!", h: 1.1, w: 80.0 },
  129: { fact: "It just splashes... but wait until it evolves!", h: 0.9, w: 10.0 },
  130: { fact: "The mightiest splash-turned-dragon of the sea!", h: 6.5, w: 235.0 },
  131: { fact: "It loves ferrying people across the sea on its back!", h: 2.5, w: 220.0 },
  133: { fact: "Nobody knows what it will evolve into — surprise!", h: 0.3, w: 6.5 },
  134: { fact: "It can melt into water and disappear!", h: 1.0, w: 29.0 },
  135: { fact: "Its fur crackles with tiny lightning bolts!", h: 0.8, w: 24.5 },
  136: { fact: "It has a fiery furnace inside its fluffy body!", h: 0.9, w: 25.0 },
  143: { fact: "It eats a mountain of food, then sleeps all day!", h: 2.1, w: 460.0 },
  147: { fact: "A rare dragon that sheds its skin as it grows!", h: 1.8, w: 3.3 },
  149: { fact: "It flies around the whole world in just 16 hours!", h: 2.2, w: 210.0 },
  151: { fact: "The rarest of all — it can learn any move!", h: 0.4, w: 4.0 },
  208: { fact: "Its body is harder than diamond and super long!", h: 9.2, w: 400.0 },
  242: { fact: "The kindest Pokemon — it heals anyone who's sad!", h: 1.5, w: 46.8 },
};
```

- [ ] **Step 4: Run test to verify it passes**

Run: `"$SCRATCH/run-dex-test.sh"`
Expected output (all three lines):

```
TEST: every roster id has a DEX_INFO entry PASS
TEST: every entry has non-empty fact and positive h/w PASS
TEST: no DEX_INFO entries for unknown ids PASS
```

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Add DEX_INFO map: facts, heights, weights for all 69 Pokemon"
```

---

### Task 2: Detail card overlay (open/close, type badge, fact, size)

**Files:**
- Modify: `index.html` —
  - CSS: insert after the `.dex-cell .dup` rule (line 497), before the `/* ---------- confetti ---------- */` comment (line 499)
  - HTML: insert after `<div class="dex-grid" id="dex-grid"></div>` (line 645), inside `#screen-dex`
  - JS: add functions after `showDex()` (line 1295) and one line inside `showDex()`
- Test: `$SCRATCH/driver.js` (overwrite Task 1's driver)

**Interfaces:**
- Consumes: `DEX_INFO[id]` from Task 1 (`{ fact, h, w }`); existing `byId(id)`, `art(id)`, `TYPE_COLORS` (`type → [color1, color2]`), `me()`, `showDex()`.
- Produces: `showDexDetail(id)` opens the card for a Pokemon id; `closeDexDetail()` closes it. Overlay element `#dex-overlay` gets/loses class `open`. An empty `<div id="detail-evo">` exists inside the card — Task 3 fills it via a `renderEvoRow(id)` call appended to `showDexDetail`.

- [ ] **Step 1: Write the failing test**

Overwrite `$SCRATCH/driver.js`:

```js
const LOG = [];
const t = (name, ok) => LOG.push(`TEST: ${name} ${ok ? "PASS" : "FAIL"}`);
const finish = () => {
  const pre = document.createElement("pre");
  pre.id = "test-log";
  pre.textContent = LOG.join("\n");
  document.body.appendChild(pre);
};
try {
  // Seed a profile and open the dex (caught ids render in ascending order)
  save.profiles[0] = { avatar: 25, name: "Test", caught: [25, 26, 131, 133], wins: {} };
  current = 0;
  showDex();
  const overlay = document.getElementById("dex-overlay");
  t("overlay exists and starts closed", !!overlay && !overlay.classList.contains("open"));

  document.querySelector(".dex-cell").click(); // first cell = Pikachu (id 25)
  t("tap opens overlay", overlay.classList.contains("open"));
  t("shows name", document.getElementById("detail-name").textContent === "Pikachu");
  const badge = document.getElementById("detail-type");
  t("shows type badge", badge.textContent === "electric" && badge.style.background !== "");
  t("shows fact", document.getElementById("detail-fact").textContent === DEX_INFO[25].fact);
  const size = document.getElementById("detail-size").textContent;
  t("shows height and weight", size.includes("0.4") && size.includes("6"));

  document.querySelector(".dex-detail").click();
  t("tapping the card itself does NOT close", overlay.classList.contains("open"));
  document.querySelector(".dex-close").click();
  t("close button closes", !overlay.classList.contains("open"));

  document.querySelectorAll(".dex-cell")[1].click(); // Raichu
  t("reopens for another Pokemon", overlay.classList.contains("open")
    && document.getElementById("detail-name").textContent === "Raichu");
  overlay.click(); // backdrop
  t("backdrop click closes", !overlay.classList.contains("open"));
} catch (e) {
  LOG.push("TEST: exception FAIL " + e.message);
}
finish();
```

- [ ] **Step 2: Run test to verify it fails**

Run: `"$SCRATCH/run-dex-test.sh"`
Expected: `TEST: overlay exists and starts closed FAIL` then `TEST: exception FAIL ...` (overlay is null).

- [ ] **Step 3: Implement the overlay**

CSS (into the pokedex CSS section):

```css
  .dex-overlay {
    position: fixed; inset: 0;
    background: rgba(0,0,0,.45);
    display: none;
    align-items: center; justify-content: center;
    z-index: 50;
  }
  .dex-overlay.open { display: flex; }
  .dex-detail {
    position: relative;
    background: var(--card);
    border-radius: 22px;
    box-shadow: 0 8px 0 rgba(0,0,0,.2);
    padding: 26px 28px 22px;
    width: min(420px, 92vw);
    max-height: 88vh;
    overflow-y: auto;
    display: flex; flex-direction: column; align-items: center; gap: 10px;
    text-align: center;
  }
  .dex-detail h2 { font-size: 30px; }
  #detail-art { width: 160px; height: 160px; object-fit: contain; }
  .dex-close {
    position: absolute; top: 10px; right: 12px;
    width: 44px; height: 44px;
    border: none; border-radius: 50%;
    background: var(--red); color: #fff;
    font-size: 20px; font-family: inherit;
    box-shadow: 0 3px 0 rgba(0,0,0,.2);
    cursor: pointer;
  }
  .type-badge {
    border-radius: 999px;
    padding: 4px 18px;
    font-size: 18px;
    box-shadow: 0 3px 0 rgba(0,0,0,.12);
  }
  .dex-fact {
    background: rgba(0,0,0,.06);
    border-radius: 14px;
    padding: 10px 14px;
    font-size: 17px;
  }
  .dex-size { display: flex; gap: 20px; font-size: 17px; }
```

HTML (inside `#screen-dex`, right after the dex grid):

```html
  <div class="dex-overlay" id="dex-overlay" onclick="closeDexDetail()">
    <div class="dex-detail" onclick="event.stopPropagation()">
      <button class="dex-close" onclick="closeDexDetail()">✖</button>
      <img id="detail-art" src="" alt="">
      <h2 id="detail-name"></h2>
      <span class="type-badge" id="detail-type"></span>
      <p class="dex-fact" id="detail-fact"></p>
      <div class="dex-size" id="detail-size"></div>
      <div class="evo-row" id="detail-evo"></div>
    </div>
  </div>
```

JS — inside `showDex()`, after `cell.innerHTML = ...` add:

```js
    cell.onclick = () => showDexDetail(id);
```

JS — new functions after `showDex()`:

```js
function showDexDetail(id) {
  const p = byId(id), d = DEX_INFO[id];
  document.getElementById("detail-art").src = art(id);
  document.getElementById("detail-name").textContent = p.name;
  const badge = document.getElementById("detail-type");
  badge.textContent = p.type;
  badge.style.background = (TYPE_COLORS[p.type] || ["#cfd8dc"])[0];
  document.getElementById("detail-fact").textContent = d.fact;
  document.getElementById("detail-size").innerHTML =
    `<span>📏 ${d.h} m</span><span>⚖️ ${d.w} kg</span>`;
  document.getElementById("detail-evo").innerHTML = "";
  document.getElementById("dex-overlay").classList.add("open");
}

function closeDexDetail() {
  document.getElementById("dex-overlay").classList.remove("open");
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `"$SCRATCH/run-dex-test.sh"`
Expected: all 9 `TEST:` lines end in `PASS`.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Add Pokedex detail card: tap a Pokemon for type, fact, and size"
```

---

### Task 3: Evolution row

**Files:**
- Modify: `index.html` —
  - CSS: append to the pokedex CSS added in Task 2 (after the `.dex-size` rule)
  - JS: add `evoPrev`/`renderEvoRow` after `closeDexDetail()`; replace one line in `showDexDetail()`
- Test: `$SCRATCH/driver.js` (overwrite)

**Interfaces:**
- Consumes: `EVOLUTIONS` (base id → evolved id, or array for Eevee), `byId(id)`, `art(id)`, `#detail-evo` div and `showDexDetail(id)` from Task 2.
- Produces: `renderEvoRow(id)` fills `#detail-evo` with the Pokemon's full chain (`.evo-sprite` imgs joined by `.evo-arrow` spans; current Pokemon gets class `current`; Eevee branches render as a `.evo-branch` group). Empty for Pokemon with no evolution relation.

- [ ] **Step 1: Write the failing test**

Overwrite `$SCRATCH/driver.js`:

```js
const LOG = [];
const t = (name, ok) => LOG.push(`TEST: ${name} ${ok ? "PASS" : "FAIL"}`);
const finish = () => {
  const pre = document.createElement("pre");
  pre.id = "test-log";
  pre.textContent = LOG.join("\n");
  document.body.appendChild(pre);
};
const evo = () => document.getElementById("detail-evo");
const sprites = () => evo().querySelectorAll(".evo-sprite");
const currentAlt = () => evo().querySelector(".evo-sprite.current")?.alt;
try {
  save.profiles[0] = { avatar: 25, name: "Test", caught: [25], wins: {} };
  current = 0;

  showDexDetail(25); // Pikachu: base of a 2-chain
  t("Pikachu chain has 2 sprites", sprites().length === 2);
  t("Pikachu chain has 1 arrow", evo().querySelectorAll(".evo-arrow").length === 1);
  t("Pikachu is highlighted", currentAlt() === "Pikachu");

  showDexDetail(26); // Raichu: evolved form shows where it came from
  t("Raichu shows full chain", sprites().length === 2);
  t("Raichu is highlighted", currentAlt() === "Raichu");
  t("chain starts at Pikachu", sprites()[0].alt === "Pikachu");

  showDexDetail(131); // Lapras: no evolution
  t("Lapras has empty evo row", evo().children.length === 0);

  showDexDetail(133); // Eevee: branch
  t("Eevee shows base + 3 branches", sprites().length === 4);
  t("Eevee branch group exists", evo().querySelectorAll(".evo-branch").length === 1);
  t("Eevee is highlighted", currentAlt() === "Eevee");

  showDexDetail(134); // Vaporeon: member of a branch
  t("Vaporeon shows Eevee + 3 branches", sprites().length === 4
    && sprites()[0].alt === "Eevee");
  t("Vaporeon is highlighted", currentAlt() === "Vaporeon");
} catch (e) {
  LOG.push("TEST: exception FAIL " + e.message);
}
finish();
```

- [ ] **Step 2: Run test to verify it fails**

Run: `"$SCRATCH/run-dex-test.sh"`
Expected: `TEST: Pikachu chain has 2 sprites FAIL` (evo row is always empty), several more FAILs, and `TEST: Lapras has empty evo row PASS`.

- [ ] **Step 3: Implement the evolution row**

CSS (after `.dex-size`):

```css
  .evo-row { display: flex; align-items: center; justify-content: center; gap: 6px; flex-wrap: wrap; }
  .evo-sprite { width: 54px; height: 54px; object-fit: contain; }
  .evo-sprite.current { background: var(--accent); border-radius: 12px; }
  .evo-arrow { font-size: 22px; }
  .evo-branch { display: flex; gap: 4px; }
```

JS — in `showDexDetail()`, replace `document.getElementById("detail-evo").innerHTML = "";` with:

```js
  renderEvoRow(id);
```

JS — after `closeDexDetail()`:

```js
// Walk EVOLUTIONS backwards: who evolves INTO this id? (null if nobody)
function evoPrev(id) {
  for (const k in EVOLUTIONS) {
    const v = EVOLUTIONS[k];
    if (Array.isArray(v) ? v.includes(id) : v === id) return Number(k);
  }
  return null;
}

function renderEvoRow(id) {
  const row = document.getElementById("detail-evo");
  row.innerHTML = "";
  let base = id, prev = evoPrev(base);
  while (prev !== null) { base = prev; prev = evoPrev(base); }
  const chain = [base];
  let branches = null;
  let next = EVOLUTIONS[base];
  while (next != null) {
    if (Array.isArray(next)) { branches = next; break; }
    chain.push(next);
    next = EVOLUTIONS[next];
  }
  if (chain.length === 1 && !branches) return; // no evolution relation
  const sprite = pid =>
    `<img class="evo-sprite${pid === id ? " current" : ""}" src="${art(pid)}" alt="${byId(pid).name}">`;
  const arrow = `<span class="evo-arrow">→</span>`;
  let html = chain.map(sprite).join(arrow);
  if (branches) html += arrow + `<span class="evo-branch">${branches.map(sprite).join("")}</span>`;
  row.innerHTML = html;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `"$SCRATCH/run-dex-test.sh"`
Expected: all 12 `TEST:` lines end in `PASS`.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Show evolution chain in Pokedex detail card"
```

---

### Task 4: Full regression run + README

**Files:**
- Modify: `README.md:19` (Pokedex feature bullet)
- Test: `$SCRATCH/driver.js` (overwrite with combined suite)

**Interfaces:**
- Consumes: everything from Tasks 1–3.
- Produces: nothing new — this is the final gate.

- [ ] **Step 1: Run the combined regression driver**

Overwrite `$SCRATCH/driver.js` with the concatenation of the three previous drivers' bodies inside ONE preamble/`finish()` pair — i.e. start with the shared preamble, then paste Task 1's `try{...}` block, Task 2's `try{...}` block (including its seed lines), Task 3's helper consts (`evo`, `sprites`, `currentAlt`) and `try{...}` block, and end with a single `finish();`.

Run: `"$SCRATCH/run-dex-test.sh"`
Expected: all 24 `TEST:` lines end in `PASS`.

- [ ] **Step 2: Sanity-check the game still boots clean**

Run:

```bash
SCRATCH="/private/tmp/claude-501/-Users-anjor-repos-anjor-pokemon-game/8118ea75-4c35-46f3-bd73-23b885120500/scratchpad"
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --virtual-time-budget=5000 \
  --dump-dom "file://$(pwd)/index.html" 2>/dev/null | grep -c "profile-card"
```

Expected: a number ≥ 1 (profile screen rendered — no JS error killed the page).

- [ ] **Step 3: Update the README feature bullet**

In `README.md`, change line 19 from:

```markdown
- **Pokedex** 📕 — everything you've caught, with duplicate badges.
```

to:

```markdown
- **Pokedex** 📕 — everything you've caught, with duplicate badges. Tap a Pokemon for its type, a fun fact, its real size, and its evolution chain.
```

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "Mention Pokedex detail cards in README"
```
