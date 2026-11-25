# Design: UI/UX Improvements — AI Influencer Builder

Date: 2026-11-25
Repo: antonocik/AIInfluencerBuilder
Scope: `index.html` (single-file HTML app, 5.591 lines, no build tooling)

## Intent and why

`index.html` is a one-page form builder: user fills 10 sections (identity, personality, niche, audience, fashion, physical, visual, content type, platforms, goals), gets a live preview card, and 5 output cards (summary, personality profile, master/closeup/fullbody prompts) with copy/download. Today it loses all form data on refresh, has no validation, no way to keep more than one character, and no dark mode. Users filling long forms repeatedly need persistence, guidance, and control.

## Goals

1. **Autosave** — form state persists across refresh/close; Reset button clears it.
2. **Validation** — required fields marked, inline errors, Generate blocked when required fields empty.
3. **Section navigation** — step indicator 01–10 with active highlighting; mobile prev/next bar.
4. **Dark mode** — toggle in header, follows system preference on first visit, persists.
5. **Multi-character profiles** — save, switch, delete named profiles; import/export JSON.

## Non-goals

- No backend, accounts, or auth.
- No change to prompt-generation logic or output text.
- No visual redesign; keep current light-theme look as the base.
- No framework, no build step, no npm. Stays a single `index.html` + `AI Influencer Builder_files/`.
- No multi-language; UI copy stays Indonesian.

## Constraints

- Must run from `file://` (fonts loaded from local `./AI Influencer Builder_files/css2`).
- `localStorage` can be unavailable (private mode, quota). Features must degrade: warn once via toast, never break the page.
- Keep existing behavior: chips/multi-select, character-type (human/non-human) logic, accordions, copy buttons, progress animation, reduced-motion respect.
- Existing screenshot in README will need a refresh only if the visible layout changes.

## Approaches considered

| Approach | Trade-off | Decision |
|---|---|---|
| A. Patch `index.html` in place | Matches current architecture, zero deps, `file://` works, diff stays reviewable | **Chosen** |
| B. Split CSS/JS into separate files | Cleaner files, but breaks the saved-page portability and doubles the diff surface | Rejected |
| C. Rewrite with React/Vite | Overkill for a form tool; adds build tooling the repo has none of | Rejected |

## Design details

### 1. Autosave

- Storage key `aib_state_v1`: a snapshot of every form control (text inputs, textareas, selects, radios, checkboxes, chip selections, character type).
- Debounced save (400 ms) on `input`/`change`.
- On page load: restore state, then run existing `initFields()` + `updateLivePreview()`.
- **Reset button** in the form header: `confirm()` dialog, then clear `aib_state_v1`, clear the form to defaults, toast "Form direset".
- If storage throws (quota/private mode): show one toast "Autosave tidak tersedia di browser ini", disable autosave for the session, page keeps working.

### 2. Validation

- Required: **Nama Karakter**, **Jenis Karakter**, **Niche Konten**. Everything else optional.
- Required labels get `*` and `aria-required="true"`.
- On Generate click: check required fields; if any empty → focus first invalid field, inline error text under it, toast "Lengkapi field wajib dulu".
- Error clears on input in that field.

### 3. Section navigation

- Sticky step bar directly under the site header: 10 numbered pills (01–10) matching the section titles. Click scrolls to that section; active pill highlights while scrolling (IntersectionObserver).
- Under 600 px: bottom-fixed bar with "Sebelumnya" / "Berikutnya" buttons and "Bagian N dari 10" label; scrolls to adjacent section.
- Existing `scroll-behavior: smooth` stays.

### 4. Dark mode

- `<html data-theme="light|dark">`.
- Toggle button in header (sun/moon SVG), persists to `aib_theme`.
- First visit: match `prefers-color-scheme`.
- Dark palette defined as CSS variable overrides under `[data-theme=dark]` (bg, surface, ink, ink-muted, border, shadows, chips, output cards). No other markup changes.

### 5. Multi-character profiles

- Storage key `aib_profiles_v1`: `[{id, name, savedAt, state}]` — same `state` shape as autosave.
- Profile bar above the form: selector dropdown + "Simpan profil baru" + delete.
- Switching a profile loads its state into the form (and becomes the autosave target).
- **Export**: download all profiles as `aib-profiles.json`.
- **Import**: file input; merge by `id` (imported wins on conflict), toast with counts.
- Flow with no saved profile stays exactly as today.

## Decisions (settled)

- **Storage schema**: `aib_state_v1` = current working form state; `aib_current_profile` = id of active profile (or null); `aib_profiles_v1` = `[{id,name,savedAt,state}]`. On load, `aib_state_v1` wins (refresh restores unsaved edits, not a stale profile snapshot).
- **Profile auto-update**: edits write into the active profile (same debounced save path as autosave). "Simpan profil baru" duplicates the current form as a new profile. Switching profiles flushes the current one first.
- **Step bar vs hidden section**: pills render only for visible sections (section 06 "Penampilan Fisik" is hidden for non-human characters → skipped); count and "Bagian N dari M" use the visible count.
- **Reset semantics**: clears the form, `aib_state_v1`, and the output cards. Saved profiles are never deleted by Reset.
- **Dark mode**: follows `prefers-color-scheme` until the user toggles; then the choice persists and no longer follows the system.
- **No `localStorage`**: autosave disabled and profile bar hidden; one toast, page fully usable.

## What's left to the builder

Exact CSS values, SVG icons, Indonesian wording for toasts/errors/labels, and final placement of the profile bar and step bar. Anything not specified here is the builder's call.
