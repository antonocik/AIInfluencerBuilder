# Lumio-Style UI Redesign — Design

Date: 2026-11-25
Status: draft, awaiting approval
Chosen approach: **B — skin + nav rail + CSS backdrop**

## Intent

Restyle `index.html` so it looks and feels like the Lumio demo
(https://demo.codeandchill.store/lumio/): dark glass cards floating over a
soft blurred backdrop, lemon accent, pill-shaped controls, floating left
navigation rail, Urbanist type. Original CSS only — Lumio is a paid
licensed template, so no CSS text is copied from it. The look is
recreated from tokens observed on the live demo.

Why: current UI is a flat blue-on-white SaaS form. User wants the premium
dashboard feel of Lumio while keeping the app's behavior identical.

## Goals

- Visual match to Lumio's design language: glass surfaces, lemon accent,
  pill radii, floating rail, sticky pill topbar, gradient+grain backdrop.
- Both themes restyled (dark is the signature look; light is the warm
  gray Lumio variant).
- Zero behavior change: same DOM ids, same JS, same storage keys, same
  copy language (Indonesian).
- 27-test harness stays green.

## Non-goals

- No new features, no JS logic changes (CSS-only + minimal HTML class
  hooks if needed).
- No framework, no build step, no npm, no external assets other than the
  Google Fonts stylesheet (already the pattern).
- No copying of Lumio CSS source. Recreate from observed tokens only.
- No restructuring of the saved-page artifact (duplicate `<style>` blocks
  after line ~2613, `<qb-toolbar>` junk, stray `<html>` fragments stay
  untouched — they don't win the cascade today and tests pass).

## Anti-goals

- Do not break test contract selectors: `#stepBar`, `#mobileNav`,
  `#themeToggle`, `#profileBar`, `.step-pill[data-target]`,
  `#sectionCount`.
- Do not let the rail or topbar cover content or inputs at any width.
- Do not require network for the backdrop (pure CSS, `file://` must
  render fully offline; Google Fonts may silently fall back to
  `system-ui`).

## Constraints

- Single file: `C:\Users\david\AIInfluencerBuilder\index.html`.
- App CSS lives in ONE `<style>` block, lines 12–490. All edits go
  there. Everything after line ~2613 is saved-page junk that must not be
  edited.
- Required fields `#charName`, `#charTypeSelect`, `#nicheSelect` untouched.
- Storage keys verbatim: `aib_state_v1`, `aib_current_profile`,
  `aib_profiles_v1`, `aib_theme`. Autosave debounce 400 ms.
- Test harness: `tests.html` + run-tests.sh (headless Chrome). Suite must
  print ALL PASS.
- Test `theme: dark palette applied` asserts `--ink` is `#e2e8f0` (dark)
  and `#0f172a` (light). If the redesign changes `--ink` values, update
  those two hex assertions in `tests.html` in the same task. Test IDs
  themselves are contract and must not be renamed.
- Existing screenshots (`screenshot.png`, `screenshot-dark.png`) become
  stale; regenerate after implementation.

## Approaches considered

**A — Skin only.** Change tokens, colors, radii, fonts, card glass.
Keep the horizontal sticky step bar under the header.
- Trades: smallest diff, zero layout risk. No rail, no topbar pill —
  the two most recognizable Lumio features missing. Rejected as too
  conservative for "gunakan UI SEPERTI lumio".

**B — Skin + nav rail + CSS backdrop (CHOSEN).** A, plus:
- `#stepBar` becomes a floating vertical icon rail fixed on the left on
  desktop (same pills, same ids, same JS); stays hidden ≤600px where
  `#mobileNav` bottom bar already takes over.
- Header becomes a floating sticky pill topbar (rounded, glass, inset
  ring, inset from page edge) instead of full-width bar.
- Body gets a pure-CSS backdrop: layered radial gradients (lemon glow +
  cool tint) plus SVG `feTurbulence` grain as an inline data-URI.
- Trades: moderate CSS diff; `.layout` needs left margin on desktop so
  the rail never overlaps the form; sticky offsets (`top` values,
  `scroll-margin-top`) need retuning. All CSS-only, DOM ids preserved,
  tests unaffected beyond the optional `--ink` hex update.

**C — Full dashboard restructure.** Rebuild markup into Lumio's app
shell (sidebar with logo, content pane, topbar with breadcrumbs).
- Trades: biggest visual fidelity, but rewrites DOM → test harness,
  section-nav JS, and validation wiring all need rework. High risk for a
  single-file app. Rejected: B delivers the look without touching
  behavior.

## Design (approach B)

### 1. Token remap (both themes in the one style block)

Structure stays as it is today: light tokens in `:root`, dark tokens in
`[data-theme=dark]`. No dark-first flip — first paint must keep matching
system preference with no flash, and `theme: first visit matches system`
must keep passing untouched. Only token *values* change.

Dark (`[data-theme=dark]`, the signature Lumio look):

| Token | New value | Meaning |
|---|---|---|
| `--bg` | `#101210` | ground |
| `--surface` | `rgba(38,40,38,0.62)` | glass card fill |
| `--surface-alt` | `rgba(255,255,255,0.04)` | subtle raise |
| `--line` | `rgba(255,255,255,0.10)` | hairline / inset ring |
| `--primary` | `#f4e31c` | accent lemon |
| `--on-primary` (new) | `#151714` | text on accent |
| `--primary-dark` | `#ded11a` | accent hover |
| `--primary-light` | `rgba(244,227,28,0.14)` | accent tint |
| `--ink` | `#f2f2ee` | main text (update test hex) |
| `--ink-muted` | `#a8aaa4` | secondary |
| `--ink-faint` | `#75776f` | tertiary |
| `--success` / `--danger` | keep `#10B981` / `#EF4444` | unchanged semantics |

Light (warm gray Lumio variant): `--bg:#e9e8e2`, `--surface:rgba(255,255,255,0.72)`,
`--line:rgba(21,23,20,0.10)`, `--ink:#151714`, `--ink-muted:#5d5f58`,
accent tokens identical to dark. `--ink` light likely stays `#0f172a` or
moves to `#151714` — builder's call, test hex updated to match.

Radii: `--radius-lg:20px` (keep), `--radius-md:16px`, `--radius-sm:12px`,
add `--radius-pill:999px`.

Shadows: cards use
`box-shadow: inset 0 0 0 1px var(--line), var(--shadow)` (Lumio ring),
`--shadow: 0 18px 50px rgba(0,0,0,0.35)` dark / soft neutral light.

Grain: one inline SVG data-URI (`feTurbulence`, baseFrequency ~0.8,
low opacity ~0.05) applied as a fixed pseudo-element on `body::before`
or layered in `body` background. No external file.

### 2. Backdrop

`body` background = stacked CSS gradients over `--bg`:
radial lemon glow top-left (large, low alpha), cool neutral glow
bottom-right, then grain layer, `background-attachment: fixed`.
Pure CSS — works from `file://` offline.

### 3. Pill topbar (header)

`.site-header` stops being full-width bar with border-bottom. Becomes:
sticky `top:12px`, `max-width` ~1180px, `margin:12px auto 0`,
`border-radius: var(--radius-pill)`, glass background
(`--surface` + `backdrop-filter: blur(14px)`), inset ring instead of
border-bottom, horizontal padding 18–22px. Brand mark gets lemon fill
with `--on-primary` letters. `#themeToggle` becomes a pill button
(999 radius). Header status text keeps its dot; hide the longer status
string below 768px if it crowds the pill.

DOM unchanged: same `.site-header > .header-inner > .brand / .header-right`,
same `#themeToggle` id.

### 4. Floating left rail (`#stepBar`)

Desktop (min-width: 601px):
- `.step-bar` → `position: fixed; left: 18px; top: 50%; transform:
  translateY(-50%);` vertical flex column, `gap: 10px`, `z-index: 30`.
- Rail container itself gets glass pill: `border-radius: 999px`,
  `padding: 10px 8px`, `background: var(--surface)`, inset ring,
  `backdrop-filter: blur(14px)`.
- `.step-pill` stays 34×34, becomes fully rounded (999). Inactive:
  muted ink on transparent. Active: lemon fill, `--on-primary` number,
  soft lemon glow shadow. Hover: accent border.
- Labels: keep pills number-only (current design), `title`/`aria-label`
  unchanged (JS already sets them).

Layout compensation (settled after builder check): `.progress-section`,
`.results-section`, and `footer` sit outside `<main class="layout">`, so
per-container insets would drift. Instead, apply ONE left inset to the
whole page: `body{padding-left:96px}` at `min-width:601px`. Header pill,
layout, progress, results, and footer all shift together and stay
aligned; the fixed rail floats in the created gutter and can never cover
content at any width ≥601px. Below 601px the padding is removed and
`#mobileNav` takes over (existing rule).

Header side gap: `.site-header` uses
`width:calc(100% - 32px); max-width:1180px; margin:12px auto 0;` so the
pill never touches viewport edges below 1180px+32px.

Mobile (≤600px): `.step-bar{display:none}` — unchanged. `#mobileNav`
bottom bar restyled to match: glass pill bar, lemon active state, 999
radius on items.

Scroll offsets: `.form-section{scroll-margin-top}` retuned for the new
floating header height (~120px still likely right; verify by test).

### 5. Cards and controls

- `.form-section`, `.profile-bar`, `.output-card`: `background:
  var(--surface)`, no solid border (ring via inset box-shadow),
  `border-radius: 20px`, `backdrop-filter: blur(10px)`.
- Buttons (`.btn-primary`, `.btn-secondary`, profile actions):
  `border-radius: 999px`. Primary = lemon fill + `--on-primary` text.
  Secondary = ghost with inset ring.
- Inputs/selects: single-line inputs and selects → pill radius 999,
  `background: var(--surface-alt)`, focus ring lemon
  (`outline: 2px solid rgba(244,227,28,0.55)`). Textareas keep
  `--radius-md` (pill shape reads wrong on multi-line) — builder's call,
  note in diff.
- `.eyebrow`, `.section-num`: lemon tint background, dark lemon text on
  light theme / lemon text on dark.
- `.step-pill.active` and all active states share one accent recipe.

### 6. Typography

- `--font-display` and `--font-body` → `'Urbanist', system-ui, sans-serif`.
- Font link: replace broken saved-page link (`./AI Influencer Builder_files/css2`)
  with a real Google Fonts URL for Urbanist (weights 400/500/600/700).
  Fallback `system-ui` keeps `file://` usable offline.
- `--font-mono`: keep JetBrains Mono only if the link is extended;
  otherwise fall back to `ui-monospace, monospace` for `.header-status`
  and `.section-num`. Builder's call — smallest link is fine.
- Heading sizes stay as-is (30px h1, 17px h2); Urbanist is rounder, no
  size change needed.

### 7. Theme toggle behavior

Unchanged logic (`aib_theme`, `data-theme` on `<html>`, system
preference first visit). Only the icon button's skin changes (pill,
lemon hover).

## What's left to the builder

- Exact gradient stop values and grain opacity (taste; verify visually
  via screenshot).
- Whether textarea gets pill or `--radius-md`.
- Whether `--ink` light stays `#0f172a` (zero test edits) or moves to
  warm `#151714` (two-line test edit) — prefer warm for palette
  consistency; update test hex in same commit.
- Mono font choice (extend Google Fonts link vs. system mono).
- Sticky offsets retune (`preview-column top`, `scroll-margin-top`)
  verified by running the harness + manual scroll test.
- Regenerate `screenshot.png` / `screenshot-dark.png` at 1440px.

## Builder-check round (2026-11-25)

Two questions came back from the reviewer; both settled here:

1. **Rail overlap below ~1260px** (`.progress-section`, `.results-section`,
   `footer` sit outside `.layout`). Settled: single `body` left inset
   `96px` at `min-width:601px` instead of per-container margins — see
   §4. Header pill side gap via `width:calc(100% - 32px)` — see §4.
2. **Token block structure** ("`:root` swap" ambiguity). Settled: keep
   current structure, light in `:root`, dark in `[data-theme=dark]`; no
   first-paint change — see §1.

## Verification plan (for the plan doc)

1. Headless screenshot before/after at 1440px and 375px, both themes.
2. Full harness: `run-tests.sh` → 27 ALL PASS (update `--ink` hex
   assertions if palette moves).
3. Overlap check: rail must not cover any content (form, progress,
   results, footer) at 601px, 1024px, 1280px, 1440px; mobile bottom nav
   must not cover last section (existing `padding-bottom` rule keeps
   this true — re-verify).
4. Offline check: `file://` render with network blocked → backdrop and
   layout intact, font falls back to `system-ui`.
