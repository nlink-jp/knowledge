# Web UI (CSS / HTML)

CSS/layout traps from WebView frontends (Wails etc.) and self-contained HTML
report generation. Each entry follows **symptom → why → how to apply**.

---

## Thin visual elements need an extended hit area for hover

**Symptom:** A 2 px bar with `pointer-events: none` never showed its hover
tooltip.

**Why:** 2–3 px cannot hold hover at normal mouse precision, and
`pointer-events: none` kills `:hover` itself — doubly unfirable.

**How to apply:**
1. Make the element at least 6 px tall (ideally 8 px).
2. `linear-gradient(to bottom, transparent 0, transparent <pad>px, <color> <pad>px, ...)`
   keeps the visual to a thin band at the bottom (looks identical, larger hit
   area).
3. Never add `pointer-events: none`.
4. Signal affordance with `cursor: help`.
5. Give tooltip `::after` elements enough `z-index` to overlay rows below.

## In row-transformed virtual tables, cell z-index only works within the row

**Symptom:** In a virtualized table (rows positioned with
`transform: translateY()`), no amount of `z-index` on an auto-growing cell editor
stopped the next row's text from painting over it.

**Why:** **transform creates a stacking context**, so each row is its own
stacking context. A cell's z-index only applies within its row's context and
cannot beat later DOM siblings (rows below). Easily mistaken for a
transparent-background issue; it's actually paint order.

**How to apply:**
- Lift **the whole row containing the overlay**: `.vt-row-editing { z-index: 20; }`
  (rows already have stacking contexts via transform, so z-index applies).
- Design the whole z-index hierarchy (e.g. context menu 100 / editing row 20 /
  header 2).
- Debugging lesson: "elements below show through" is not necessarily
  transparency — a later sibling stacking context painting on top looks the same.

## Undefined CSS variables fail silently and break UI

**Symptom:** New styles referenced `var(--accent-primary)` which no theme defined.
It happened to look fine in dark theme, but in light theme "the button disappears
after clicking".

**Why:** An undefined custom property falls back to the initial value
(transparent for background, black for color, …) — **with no error**.

**How to apply:**
- When writing `var(--xxx)`, always check it exists in the theme definition file.
- Inline fallbacks (`var(--x, #5078c8)`) defeat theme switching — not
  recommended.
- With multiple themes, verify **every theme** (dark merely "happening to look
  right" is common).
- Prefer existing theme tokens over inventing new variables.

## CSS notes for self-contained HTML reports

**Symptom:** Generating a CDN-free single-file HTML report took 7 layout-bug
iterations. Recurring issues:

1. **overflow-x: auto and overflow-y: visible cannot coexist** — per spec, one
   being auto/scroll converts the other's visible to auto. Negative-margin
   indicators on an overflow-x tab bar are impossible → move the border to the
   button side.
2. **No margins inside grid cards** — a margin-bottom on `.card` needs
   cancellation inside grids and yields zero spacing outside them. Unify spacing
   with the container's `gap`
   (`display: flex; flex-direction: column; gap: 1rem`).
3. **grid align-items: start over stretch** — stretch interacts with
   margins/paddings to create steps; start's natural top alignment is stabler.
4. **Normalize LLM output used in CSS classes** — LLMs return "Excellent" etc.
   capitalized while classes are lowercase; apply `|lower` to every badge class
   in the template.
5. **Prevent double @ on usernames** — LLMs sometimes return names already
   @-prefixed; guard with a helper that checks the first character.
