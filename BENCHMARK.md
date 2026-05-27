# Benchmark — Firefox (Example Domain page)

Live test of `screenview` against a real Firefox window showing `example.com`.
Goal: answer "what page/content is in Firefox?" using each method and compare token cost.

## How tokens are estimated
- **Text output:** tokens ≈ characters ÷ 4 (standard rough ratio).
- **Images:** Anthropic caps the long edge at 1568 px, then tokens ≈ (width × height) ÷ 750.
  - Full PNG 2560×1405 → scaled to 1568×860 → ≈ **1,798 tokens**
  - Downscaled JPG 1000×549 (under cap) → ≈ **732 tokens**

## Measured results

| Method | Output | Raw size | Est. tokens | Got the answer? |
|--------|--------|----------|-------------|-----------------|
| `read` (OCR) | text | 275 chars | **~69** | ✅ full page text + title + bookmark bar |
| `windows` | text | 685 chars | ~171 | ✅ title "Example Domain — Mozilla Firefox" |
| `tree --app Firefox` | text | 39 chars | ~10 | ❌ Firefox a11y disabled by default (see note) |
| `shot` (downscaled JPG) | image | 11.7 KB | **~732** | ✅ full visual, legible |
| **full PNG (the OLD way)** | image | 53.7 KB | **~1,798** | ✅ full visual |

## Before vs After

**Task: "read what's on the Firefox page"**

| | Method | Tokens | Δ vs old |
|--|--------|--------|----------|
| BEFORE | full-res PNG | ~1,798 | — |
| AFTER (text) | OCR `read` | ~69 | **−96%** |
| AFTER (visual) | downscaled `shot` | ~732 | **−59%** |

- Reading the page as **text via OCR costs ~26× fewer tokens** than the old full screenshot
  (~69 vs ~1,798) and still captured everything: title, body text, and the bookmark bar.
- When a **visual** is genuinely needed, the downscaled JPG still cuts cost by **~59%**
  (~732 vs ~1,798) with no loss of legibility (confirmed by eye).
- Knowing just *which page is open* via `windows` is **~171 tokens** — the window title
  alone ("Example Domain — Mozilla Firefox") answers it without any capture.

## Note: Firefox & accessibility
`tree` returned nothing because **Firefox ships with its accessibility engine off** until an
assistive client forces it on (pref `accessibility.force_disabled = -1`, or env
`GNOME_ACCESSIBILITY=1` before launch). So for Firefox out of the box, **OCR is the right
tool**, and `screenview look` falls back to it automatically. The `tree` method shines on
native GTK/Qt apps (file managers, system dialogs) that expose accessibility by default.

## Takeaway
For "what's on screen" questions, the cheapest-first ladder turns a ~1,800-token look into a
~70-token one — a **~96% reduction** — while keeping a downscaled image in reserve for the
rare cases where pixels actually matter.

---

# Benchmark 2 — Thunar file manager (a11y-enabled native app)

Unlike Firefox, Thunar exposes its full accessibility tree, so this shows `tree` working.
Window was 640×480 → full PNG ≈ (640×480)/750 ≈ **410 tokens**.

| Method | Tokens | What you get |
|--------|--------|--------------|
| `read` (OCR) | ~96 | visible text only (filenames, labels) |
| `tree --depth 4` | ~144 | top-level controls + roles |
| full PNG | ~410 | pixels only, no semantics |
| `tree --depth 8` | ~1,463 | most controls, nested |
| `tree --depth 12+` | ~1,560 | every control, fully nested |

`tree` returned exact, role-tagged controls — e.g. `button: Back`, `button: New Tab`,
`toggle button: Split View`, `menu item: Icon View / List View / Compact View` — the kind of
semantic detail OCR and screenshots cannot provide, and which is what you need to click
reliably.

## Lesson: cheapest-first is task-dependent, not "always text"
- **"What can I click?"** → `tree` (shallow `--depth`) — precise control labels + roles.
- **"What does it say?"** → `read` (OCR) — cheapest for pure content.
- **"What's open?"** → `windows` — near-free.
- **"Does the layout look right?"** → downscaled `shot`.
- `tree` is the most *accurate* but not always the *cheapest* — at full depth it can exceed a
  small window's PNG. Tune `--depth`: shallow for an overview, deep only for a nested control.
