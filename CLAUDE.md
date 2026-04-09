# CLAUDE.md — ClinIQ Calculators v2

Developer guide and implementation tracking for this repository.

---

## Project Overview

**ClinIQ** is a single-page HTML application (`index.html`) that hosts FHIR-based clinical calculators (CHA₂DS₂-VASc, EuroSCORE II, HAS-BLED, TRI). It uses the **Tiro Web SDK** to render questionnaire forms and extract structured data from clinical text.

**Architecture:** Single-file app — all CSS, HTML, and JavaScript live inline in `index.html`. No build step, no bundler. Open the file directly in a browser or serve it with any static file server.

**Key external dependencies:**
- `https://cdn.tiro.health/sdk/next/tiro-web-sdk.iife.js` — Tiro Web SDK (web components)
- `https://sdc-staging.tiro.health/fhir/r5` — SDC FHIR endpoint for questionnaire rendering
- Google Fonts (Open Sans, Fira Mono)

**Web components used:**
- `<tiro-form-filler>` — renders a FHIR Questionnaire as a form; emits `tiro-update` on change
- `<tiro-magic-clipboard>` — reads clipboard text and fills a `<tiro-form-filler>` via AI extraction; its child `<tiro-magic-clipboard-button>` changes `data-state` (`idle` → `pending` → `success` | `error`)

**Shadow DOM notes:**
- `tiro-form-filler` and other Tiro components use **open** shadow roots (`mode: "open"`) — accessible via `element.shadowRoot`
- Questionnaire items are rendered with a `linkId` attribute on a wrapper element inside the shadow DOM
- Highlighting uses `deepWalk()` to recursively descend into all open shadow roots and locate elements by attribute value

---

## Autofill Feature — Implemented UX

The current implementation keeps `<tiro-magic-clipboard>` as a **visible component** in the left card. The user:
1. Pastes their clinical report into the `tiro-magic-clipboard` textarea (placeholder: "Plak hier uw verslag en extraheer automatisch data voor de berekening")
2. Clicks **"Verslag verwerken"** to trigger extraction
3. The SDK fills the questionnaire fields automatically
4. Questions not filled by autofill are highlighted in orange; they clear to green as the user fills them in

The layout is **50/50** — both columns share equal width (`grid-template-columns: 1fr 1fr`).

---

## Implementation Checklist

### Phase 1 — Core UX (all calculators)

- [x] **`tiro-magic-clipboard` card** — visible in left column with placeholder text and "Verslag verwerken" button
- [x] **Remove Tiro.health marketing CTA card** from left column
- [x] **50/50 layout** — `grid-template-columns: 1fr 1fr` (was `440px 1fr`)
- [x] **Autofill button styling** — blue full-width button via `tiro-magic-clipboard-button` CSS
- [x] **i18n strings** — `autofillLabel`, `autofillPlaceholder`, `autofillBtn` in both `en` and `nl`
- [x] **Dutch placeholder** — "Plak hier uw verslag en extraheer automatisch data voor de berekening"
- [x] **Placeholder uses `setAttribute`** — web components require `setAttribute('placeholder', ...)` not `.placeholder` property

### Phase 1 — Field Highlighting (all calculators)

- [x] **`CALCULATOR_FIELDS` map** — linkId arrays for all 4 calculators: `chadsvasc` (7), `euroscore2` (17), `hasbled` (9), `tri` (8)
- [x] **`autofillActivated` flag** — highlights only appear after the autofill button is clicked; reset on calculator change
- [x] **`deepWalk(root, callback)`** — recursively walks shadow DOM; checks `root.shadowRoot` before walking children
- [x] **`discoverQuestionItems()`** — walks entire `tiro-form-filler` shadow tree, matches any attribute value to a known linkId; retries at 300ms / 800ms / 1.5s / 3s if elements not yet rendered
- [x] **`setEmptyStyle(el)`** — orange outline (`#f97316`) on unanswered fields
- [x] **`flashFilledStyle(el)`** — green flash (`#16a34a`) when a field transitions from empty → filled, then clears
- [x] **`applyFieldHighlights(response)`** — called on every `tiro-update` after autofill is activated
- [x] **`resetHighlights()`** — clears all styles and `linkIdToEl` on calculator switch
- [x] **Missing badge** — `#missing-badge` in card header shows count of unanswered fields; auto-hides when all filled
- [x] **Calculator change resets** — `loadCalculator()` resets `highlightReady`, `autofillActivated`, clears badge

### Phase 2 — Polish

- [ ] **Field count accuracy** — improve `countFilledChadsvascFields()` / badge using FHIR QuestionnaireResponse `item` traversal instead of result row proxy
- [ ] **`autofill_field_overridden` field_id** — implement FHIR item diff to populate `field_id` and `new_value_code`
- [ ] **Graceful error state** — verify SDK error surfaces as `data-state="error"` and show user-facing message
- [ ] Language auto-detection for NL/EN clinical text

### Phase 3 — Expansion

- [ ] Validate field coverage for HAS-BLED and TRI from real clinical letters
- [ ] Review EuroSCORE II field list (some fields like `eGFR` decimal input may need separate handling)

---

## How to Test

1. Open `index.html` in a browser (no build step needed)
2. Select a calculator (e.g. **CHA₂DS₂-VASc Score**) in the dropdown
3. Paste a clinical cardiology report into the left textarea
4. Click **"Verslag verwerken"**
5. Verify:
   - Questionnaire fields auto-populate
   - Unfilled fields get an orange outline highlight
   - Filling a highlighted field flashes green then clears
   - Missing badge in card header counts remaining fields
6. Open browser DevTools → Console to see `[ClinIQ] highlight:` debug messages confirming element discovery
7. Switch language (EN ↔ NL) — verify placeholder and button label translate correctly
8. Switch calculator — verify highlights and autofill state reset

---

## Open Questions / Known Limitations

- **`tiro-magic-clipboard` clear API:** Clicking Clear (if added) needs to reset the SDK component state. Currently there is no explicit clear on `tiro-magic-clipboard`. Investigate SDK API.
- **Field count accuracy:** Badge count is based on FHIR QuestionnaireResponse traversal but score fields (decimal result items) may count as answered — filter to input-type fields only.
- **EuroSCORE II decimal fields:** `Leeftijd` and `eGFR` are `decimal` type inputs — verify that `deepWalk` finds them correctly and that `isAnswered()` handles `valueDecimal` (it does, but confirm in browser).
