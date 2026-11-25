# UI/UX Improvements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add autosave+reset, validation, section navigation, dark mode, and multi-character profiles to the single-file `index.html` app.

**Architecture:** Patch `index.html` in place: one `AIB` namespace in the existing inline script, small HTML additions (Reset button, step bar, mobile nav, profile bar, theme toggle), CSS variable overrides for dark mode. Tests live in a new `tests.html` that loads `index.html` in an iframe and asserts against its DOM/localStorage; run headless Chrome with `--dump-dom`.

**Tech Stack:** Vanilla HTML/CSS/JS in one file. No framework, no npm, no build step. Tests: headless Chrome only.

**Spec:** `docs/superpowers/specs/2026-11-25-ui-ux-improvements-design.md`

## Global Constraints

- Single `index.html`; must run from `file://` (fonts from `./AI Influencer Builder_files/`). No modules, no fetch of local files without `--allow-file-access-from-files`.
- Test command (all tasks): `& 'C:\Program Files\Google\Chrome\Application\chrome.exe' --headless --allow-file-access-from-files --dump-dom file:///C:/Users/david/AIInfluencerBuilder/tests.html` — pass = output contains `ALL PASS`, fail = a `FAIL ` line.
- Storage keys verbatim: `aib_state_v1`, `aib_current_profile`, `aib_profiles_v1`, `aib_theme`.
- Autosave debounce: 400 ms.
- Required fields verbatim: `#charName`, `#charTypeSelect`, `#nicheSelect`.
- Existing selectors kept: `#builderForm`, `#generateBtn`, `.form-section` (ids `section1`..`section10`), `#resultsSection`, `#previewName`.
- UI copy Indonesian. Every storage write wrapped so a throw shows one toast (`Autosave tidak tersedia di browser ini`) and disables that feature for the session — never breaks the page.
- `AIB` namespace (single global object) holds all new functions; existing functions untouched except where a step names them.

## Review Focus

1. **localStorage unavailable (private mode)** — autosave/profiles must degrade with one toast, page fully usable. Test in Task 2 (`autosave: storage failure degrades, no throw`).
2. **Section 06 hidden for non-human characters** — step bar must skip it and count visible sections only. Test in Task 4 (`nav: hidden section 06 skipped, count is 9`).
3. **Profile switch must flush current edits, and reload restores working state over profile snapshot** — data-loss risk. Tests in Task 6 (`profile: switch flushes edits` and `profile: reload restores working state`).
4. **Reset must never delete saved profiles** — destructive action. Test in Task 2 (`reset: keeps saved profiles`).
5. **Dark mode freezes at user choice after toggle** — no longer follows system. Test in Task 5 (`theme: choice persists across reload, system light ignored`).

---

### Task 1: Test harness

**Files:**
- Create: `tests.html`

**Interfaces:**
- Produces: test API used by every later task — `t(name, fn)` registers a test (fn may return a Promise), `assert(cond, msg)`, `assertEq(actual, expected, msg)`, `withFreshPage(fn)` reloads the `index.html` iframe and runs `fn(iframeWindow)` after `AIB` is ready. Runner writes one line per test to `#results` (`PASS <name>` / `FAIL <name>: <msg>`), then `ALL PASS (n)` or `FAILURES (n)`.

- [ ] **Step 1: Write `tests.html`**

Iframe `src="index.html"`, poll until `iframe.contentWindow.AIB && iframe.contentWindow.document.readyState === 'complete'`, then run registered tests sequentially in the iframe's window context. One smoke test: `t('smoke: page loads with form', ...)` asserting `iframe.contentWindow.document.querySelector('#builderForm')` and `#generateBtn` exist.

- [ ] **Step 2: Run harness**

Run the Global Constraints test command. Expected: `ALL PASS (1)`.

- [ ] **Step 3: Commit**

```bash
git add tests.html
git commit -m "test: add browser test harness"
```

---

### Task 2: Autosave + Reset

**Files:**
- Modify: `index.html` (inline `<script>` after `init()`, form header markup, CSS)
- Test: `tests.html`

**Interfaces:**
- Produces (all on global `AIB`): `AIB.getState()` → object|null, `AIB.setState(state)` → void, `AIB.saveState()` → void (immediate write of current form to `aib_state_v1`), `AIB.scheduleSave()` → void (400 ms debounce), `AIB.resetForm()` → void (clears form controls to defaults, removes `aib_state_v1`, clears output card bodies, hides `#resultsSection`). Reset button id `resetBtn`, placed in the form header next to `#generateBtn`.

- [ ] **Step 1: Write failing tests in `tests.html`**

- `autosave: input writes aib_state_v1 after debounce` — set `#charName` value in iframe, wait 600 ms, assert `localStorage.getItem('aib_state_v1')` contains that value.
- `autosave: reload restores state` — `withFreshPage`: set `#charName`, wait 600 ms, `withFreshPage` again, assert input value restored and preview name updated.
- `reset: clears form, state key, and outputs` — set `#charName`, save state, click `#resetBtn` (confirm stubbed to return true), assert input empty, `localStorage.getItem('aib_state_v1')` null, output card `pre` bodies empty.
- `reset: keeps saved profiles` — write a profile object to `aib_profiles_v1`, click `#resetBtn`, assert profile still present in `aib_profiles_v1`.
- `autosave: storage failure degrades, no throw` — replace `Storage.prototype.setItem` with a thrower in the iframe, type into `#charName`, assert no exception escapes and the toast element is visible exactly once.

- [ ] **Step 2: Run tests to verify they fail**

Run test command. Expected: `FAIL autosave: ...` lines (AIB/state functions undefined).

- [ ] **Step 3: Implement in `index.html`**

Add `AIB` namespace with the five functions; wire `input`/`change` listeners on `#builderForm` → `AIB.scheduleSave()`; call `AIB.setState(...)` restore inside existing `init()` before `updateLivePreview()`; add `#resetBtn` markup + `confirm()` + toast; wrap `localStorage.setItem` in a try/catch that toasts once and sets a session flag disabling further saves.

- [ ] **Step 4: Run tests to verify they pass**

Run test command. Expected: `ALL PASS` including the five new tests.

- [ ] **Step 5: Commit**

```bash
git add index.html tests.html
git commit -m "feat: add autosave and reset"
```

---

### Task 3: Validation

**Files:**
- Modify: `index.html` (`#generateBtn` click path, field markup, CSS)
- Test: `tests.html`

**Interfaces:**
- Produces: `AIB.validate()` → `{ok: boolean, firstInvalid: HTMLElement|null}`. Error elements: `<p class="field-error" id="charNameError">`, `charTypeError`, `nicheError` (empty, `hidden` by default). Required fields get `aria-required="true"` and a `*` in their `<label>`.

- [ ] **Step 1: Write failing tests**

- `validation: required markers present` — `#charName`, `#charTypeSelect`, `#nicheSelect` each have `aria-required="true"` and a visible `*` in the associated label.
- `validation: empty required blocks generate` — clear all three, stub `confirm`, click `#generateBtn`, assert `#charNameError` visible, `#charName` is `document.activeElement`, `#resultsSection` unchanged (its textContent equals before-click snapshot).
- `validation: error clears on input` — after the failing case, type into `#charName`, assert `#charNameError` hidden.
- `validation: valid form passes` — fill the three fields, assert `AIB.validate().ok === true` and `firstInvalid === null`.

- [ ] **Step 2: Run tests to verify they fail**

Expected: marker/behavior FAILs.

- [ ] **Step 3: Implement in `index.html`**

Add `*` + `aria-required` to the three labels; add the three error `<p>` elements after their fields; add `AIB.validate()`; prepend a check to the existing `#generateBtn` click handler: if `!AIB.validate().ok` → focus `firstInvalid`, show its error, toast `Lengkapi field wajib dulu`, and return without generating; clear error on `input` in the field.

- [ ] **Step 4: Run tests to verify they pass**

Expected: `ALL PASS`.

- [ ] **Step 5: Commit**

```bash
git add index.html tests.html
git commit -m "feat: add required-field validation"
```

---

### Task 4: Section navigation

**Files:**
- Modify: `index.html` (markup after `.site-header`, CSS, JS)
- Test: `tests.html`

**Interfaces:**
- Produces: `AIB.visibleSections()` → `HTMLElement[]` (`.form-section` without `.hidden`); `AIB.initSectionNav()` → void. Markup: `<nav class="step-bar" id="stepBar">` with one `<button class="step-pill" data-target="sectionN">NN</button>` per visible section, inserted after `.site-header`; `<div class="mobile-nav" id="mobileNav">` with `#prevSectionBtn`, `#nextSectionBtn`, `#sectionCount` (text `Bagian N dari M`), fixed bottom, shown only under 600 px.

- [ ] **Step 1: Write failing tests**

- `nav: pill count equals visible sections` — default (human) → 10 pills, `#sectionCount` text `Bagian 1 dari 10`.
- `nav: hidden section 06 skipped, count is 9` — set `#charTypeSelect` to a non-human value (the value that triggers `section6` hidden), assert 9 pills, no pill with `data-target="section6"`, count text `dari 9`.
- `nav: click pill scrolls and activates` — click pill for `section3`, assert active class on that pill and `#section3` bounding rect top within 120 px of viewport top after scroll settles.
- `nav: prev/next move between sections` — click `#nextSectionBtn`, assert active target advances by one and count text updates.

- [ ] **Step 2: Run tests to verify they fail**

Expected: no `#stepBar` → FAILs.

- [ ] **Step 3: Implement in `index.html`**

Build pills from `AIB.visibleSections()`; IntersectionObserver (rootMargin `-40% 0px -55% 0px`) sets `.active`; pill click → `scrollIntoView({behavior:'smooth', block:'start'})`; prev/next clamp at ends and update `#sectionCount`; re-run pill build when character type changes (hook the existing type-change path that toggles `section6`).

- [ ] **Step 4: Run tests to verify they pass**

Expected: `ALL PASS`.

- [ ] **Step 5: Commit**

```bash
git add index.html tests.html
git commit -m "feat: add section step navigation"
```

---

### Task 5: Dark mode

**Files:**
- Modify: `index.html` (header markup, CSS `[data-theme=dark]` overrides, JS)
- Test: `tests.html`

**Interfaces:**
- Produces: `AIB.toggleTheme()` → void; `AIB.currentTheme()` → `'light'|'dark'`. Header button `#themeToggle` with sun/moon SVG. `data-theme` on `<html>`. Key `aib_theme`.

- [ ] **Step 1: Write failing tests**

- `theme: first visit matches system` — delete `aib_theme`, reload, assert `document.documentElement.dataset.theme` equals (`matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light'`).
- `theme: toggle sets dark and persists` — click `#themeToggle`, assert `data-theme="dark"` and `localStorage.aib_theme === 'dark'`.
- `theme: choice persists across reload, system light ignored` — with `aib_theme='dark'` set, `withFreshPage`, assert `data-theme` still `dark` (headless default is light).
- `theme: dark palette applied` — in dark, assert computed `background-color` of `body` differs from the light value and `--ink` variable resolves to the dark hex.

- [ ] **Step 2: Run tests to verify they fail**

Expected: FAILs (no `data-theme` logic yet).

- [ ] **Step 3: Implement in `index.html`**

On load: `aib_theme` if present, else `matchMedia` result, write to `<html data-theme>`; register a `matchMedia('(prefers-color-scheme: dark)').change` listener that only applies while `aib_theme` is absent; `#themeToggle` flips theme, persists, updates icon; add `[data-theme=dark]{...}` CSS variable overrides for `--bg`, `--surface`, `--surface-alt`, `--ink`, `--ink-muted`, `--ink-faint`, `--border`, `--shadow`, plus header background rgba.

- [ ] **Step 4: Run tests to verify they pass**

Expected: `ALL PASS`.

- [ ] **Step 5: Commit**

```bash
git add index.html tests.html
git commit -m "feat: add dark mode toggle"
```

---

### Task 6: Multi-character profiles

**Files:**
- Modify: `index.html` (profile bar markup above `#builderForm`, CSS, JS)
- Test: `tests.html`

**Interfaces:**
- Produces: `AIB.profiles.list()` → `[{id,name,savedAt,state}]`; `AIB.profiles.saveNew(name)` → profile (duplicates current form state, sets `aib_current_profile`); `AIB.profiles.load(id)` → void (flush current via `AIB.saveState()` first, set state into form, set `aib_current_profile`); `AIB.profiles.remove(id)` → void; `AIB.profiles.exportJSON()` → string; `AIB.profiles.importJSON(text)` → `{added:number, updated:number}`. Markup: `#profileBar` above the form with `#profileSelect`, `#saveProfileBtn`, `#deleteProfileBtn`, `#exportProfileBtn`, `#importProfileBtn` + hidden `<input type="file" id="importFile" accept=".json">`. Auto-update: the same debounced save writes into the active profile's `state` when `aib_current_profile` is set.

- [ ] **Step 1: Write failing tests**

- `profile: save new creates profile and activates it` — set `#charName='Aria'`, click `#saveProfileBtn` (prompt stub returns 'Aria'), assert one profile in `aib_profiles_v1` with `state` containing `Aria`, `aib_current_profile` set, `#profileSelect` shows it.
- `profile: switch flushes edits` — save profile A, edit `#charName='Beta'`, switch to profile B (create second first), switch back to A, assert A's stored state has `Beta`.
- `profile: reload restores working state` — with `aib_state_v1` containing `Gamma` and an active profile whose state differs, `withFreshPage`, assert form shows `Gamma`.
- `profile: export/import round-trip` — `exportJSON()` parses to an array of 2; `importJSON` of a payload with one new + one existing id returns `{added:1, updated:1}` and storage holds 3 (or 2 updated) — assert exact counts from the test's own fixture.
- `profile: delete removes only that profile` — two profiles, delete one, assert the other intact and `aib_current_profile` cleared if it pointed at the deleted one.

- [ ] **Step 2: Run tests to verify they fail**

Expected: FAILs (`AIB.profiles` undefined).

- [ ] **Step 3: Implement in `index.html`**

Implement the six `AIB.profiles` functions on top of `AIB.getState()`/`AIB.setState()` from Task 2; extend the debounced save to also write the active profile's `state`; render `#profileBar`; wire buttons (`prompt()` for new-profile name, `confirm()` for delete, `Blob`+`URL.createObjectURL` for export, `FileReader` for import).

- [ ] **Step 4: Run tests to verify they pass**

Expected: `ALL PASS`.

- [ ] **Step 5: Commit**

```bash
git add index.html tests.html
git commit -m "feat: add multi-character profiles"
```

---

### Task 7: Final QA and docs

**Files:**
- Modify: `screenshot.png`, `README.md`
- Test: `tests.html` (full suite), manual screenshots

- [ ] **Step 1: Run full test suite**

Run test command. Expected: `ALL PASS` with all tasks' tests.

- [ ] **Step 2: Manual smoke in real Chrome**

Open `index.html` from `file://`: fill form, verify autosave toast-free, toggle dark mode, save/switch a profile, generate prompts. Check mobile width 375 px: bottom nav bar present, step bar hidden.

- [ ] **Step 3: Refresh README screenshot**

Headless Chrome screenshot at 1280×2400 (light) replacing `screenshot.png`; optionally add a dark-mode shot `screenshot-dark.png` and reference both in README.

- [ ] **Step 4: Commit and push**

```bash
git add -A
git commit -m "feat: UI/UX improvements - autosave, validation, navigation, dark mode, profiles"
git push
```
