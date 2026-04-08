# Auto-Fill from Clinical Narrative
### Feature Specification — Tiro.health Calculators v2

| | |
|---|---|
| **Author** | Andries Clinckaert |
| **Date** | April 2026 |
| **Status** | Draft |
| **Version** | 1.0 |
| **Product** | Calculators v2 |
| **Target release** | Q2 2026 |
| **Primary users** | Cardiologists (CHA₂DS₂-VASc, EuroSCORE II, HAS-BLED, TRI) |
| **Analytics** | PostHog (EU) |

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Goals](#2-goals)
3. [Non-Goals](#3-non-goals)
4. [User Stories](#4-user-stories)
5. [Requirements](#5-requirements)
6. [UX Flow](#6-ux-flow)
7. [PostHog Event Instrumentation](#7-posthog-event-instrumentation)
8. [Success Metrics](#8-success-metrics)
9. [Open Questions](#9-open-questions)
10. [Timeline Considerations](#10-timeline-considerations)
11. [Appendix: CHA₂DS₂-VASc Fields Reference](#appendix-cha₂ds₂-vasc-fields-reference)

---

## 1. Problem Statement

Cardiologists using Tiro.health calculators today must manually select each field in the form, even when all the relevant clinical information is already present in a prior report or discharge letter. This double-entry is slow, error-prone, and adds friction exactly at the point of care where time matters most.

- **Who:** Cardiologists computing CHA₂DS₂-VASc and EuroSCORE II scores during clinical consultations or pre-operative assessments.
- **Frequency:** Every time a calculator is opened for a patient who already has a clinical text on file (estimated: majority of calculator uses).
- **Cost of inaction:** Without auto-fill, each calculator takes 60-120 seconds of manual data entry. Across many patients per day, this compounds into significant wasted time and increases the risk of transcription errors that could affect clinical decision-making.

> **Context: existing Magic Clipboard component**
> The Tiro Web SDK already ships a `tiro-magic-clipboard` web component that reads the system clipboard and attempts to fill form fields. The current implementation exposes this component in a left-column card alongside the calculator form. The user pastes their clinical report into the textarea and clicks **"Verslag verwerken"** to trigger extraction. After extraction, unanswered fields are highlighted in orange to guide manual completion.

---

## 2. Goals

### User goals

- Reduce the time to fill a calculator from a clinical narrative to under 10 seconds.
- Make auto-fill discoverable without any onboarding or training: a cardiologist seeing the interface for the first time should know exactly what to do.
- Never silently fill a field with wrong data: only populate fields when the system is confident; leave uncertain fields empty for the user to complete manually.

### Business goals

- Achieve a 60%+ auto-fill adoption rate (users who paste text) within 30 days of launch.
- Achieve an average field fill-rate of >= 70% for CHA₂DS₂-VASc (7 fields) from a standard cardiology discharge letter.
- Generate a complete PostHog event dataset that lets us measure funnel completion, field-level extraction accuracy, and downstream score calculation, enabling data-driven iteration.

---

## 3. Non-Goals

The following are explicitly out of scope for v1.

- **Fully automatic paste-triggered fill** (no button). A single "Verslag verwerken" button click is required after pasting; this keeps the user in control and prevents accidental extraction on partial pastes.
- **Source highlighting** — showing which words in the pasted text drove which field value. Valuable for trust-building but complex to implement correctly; deferred to v2.
- **Auto-fill for calculators beyond current scope.** All four calculators (CHA₂DS₂-VASc, EuroSCORE II, HAS-BLED, TRI) are supported in v1. Additional calculators can be added by extending `CALCULATOR_FIELDS` in `index.html`.
- **Automatic report import from the EHR / FHIR server.** The paste-in-textarea model is intentional: it puts the cardiologist in control, avoids EHR integration complexity, and works in any deployment context.
- **Storing or logging the pasted clinical text.** No free-text clinical data will leave the browser. Only structured field values and PostHog events (no PHI) will be transmitted.

---

## 4. User Stories

### Primary flow

1. As a cardiologist, I want to paste the patient's clinical report into a text area on the calculator page so that the calculator fields are filled automatically, saving me from manual data entry.
2. As a cardiologist, I want to click a single button to trigger extraction after pasting, so that I stay in control and can review the pasted text before it runs.
3. As a cardiologist, I want fields that could not be extracted to be clearly highlighted so I can complete them quickly without hunting through the form.
4. As a cardiologist, I want fields that could not be extracted to remain empty rather than show a wrong pre-filled value, so that I stay in control and do not accidentally accept incorrect data.

### Error and edge cases

4. As a cardiologist, I want to see a clear indicator when the auto-fill is processing (for example, a subtle spinner) so that I know the system has received my text and is working.
5. As a cardiologist, I want to be able to clear the pasted text and reset the form fields to their empty state, so that I can start over if I pasted the wrong report.
6. As a cardiologist, I want to manually override any auto-filled field without friction, so that I can correct any extraction errors before calculating the score.
7. As a cardiologist, I want the paste area to work whether my report is in Dutch or English, so that I am not forced to translate or rewrite my own text.

---

## 5. Requirements

### 5.1 Must-Have (P0)

| Priority | Requirement | Acceptance criteria |
|---|---|---|
| P0 | **Paste area on calculator page.** A clearly labelled `tiro-magic-clipboard` textarea appears in the left column alongside the calculator form. Placeholder text (NL): "Plak hier uw verslag en extraheer automatisch data voor de berekening". Both columns are equal width (50/50). | Given the calculator page loads, the textarea is visible. The placeholder text disappears on focus. |
| P0 | **Button-triggered field population.** After pasting, the user clicks "Verslag verwerken". The `tiro-magic-clipboard` component extracts data and fills the form. | Within 3 seconds (p95 on standard clinical text) all confidently extractable fields are populated. |
| P0 | **Highlight unanswered fields.** After extraction, fields not filled by autofill are highlighted with an orange outline. As the user fills each field, the highlight transitions to a green flash then clears. | After extraction completes, all unfilled input fields have a visible orange highlight. Filling a field removes it from the highlighted set. |
| P0 | **Empty-on-uncertainty policy.** Fields for which the extraction model is not confident are left empty (not pre-filled). There is no ambiguous partial fill. | After paste, every empty field either: (a) had no relevant information in the text, or (b) had ambiguous information. No field contains a value that contradicts the pasted text. |
| P0 | **Manual override at all times.** Auto-filled fields remain fully editable. Clicking a chip or changing a value does not require clearing the auto-fill state. | After auto-fill, each field can be changed by a single click/tap with no extra steps. The change is reflected immediately in the score. |
| P0 | **Clear and reset.** A "Clear" action (icon button or text link) empties the textarea and resets all form fields to their empty state. | Clicking Clear empties the textarea and all form fields. The calculator score resets. PostHog fires `autofill_text_cleared`. |
| P0 | **PostHog event instrumentation** (see Section 7). All specified events fire with correct properties on every triggered action. | Events appear in PostHog within 5 seconds of the corresponding user action. Properties match the schema in Section 7. |

### 5.2 Nice-to-Have (P1)

| Priority | Requirement | Acceptance criteria |
|---|---|---|
| P1 | **Visual fill feedback.** Fields that were just auto-filled show a brief highlight animation (e.g., a 0.5s blue background fade). | After paste, each auto-filled field flashes the brand blue colour once then returns to its normal state. |
| P1 | **Fill summary badge.** After extraction, a small inline message shows "X of Y fields filled" directly below the textarea. | After paste completes, a message such as "5 of 7 fields filled" appears below the textarea. It disappears when the user manually edits any field. |
| P1 | **Language auto-detection.** The extraction pipeline correctly handles both Dutch (nl-BE) and English clinical text without any user toggle. | Given a Dutch discharge letter, Dutch field values are extracted. Given an English report, English values are extracted. Both produce the same SNOMED-coded answers. |
| P1 | **Graceful error state.** If extraction fails (network timeout, SDK error), the textarea shows a non-blocking inline error message and all fields remain in their pre-paste state. | On error, a message appears below the textarea: "Auto-fill kon het verslag niet verwerken. Vul de velden manueel in." No field is partially filled. |

### 5.3 Future Considerations (P2)

These items are out of scope for v1 but should be considered during implementation to avoid architectural lock-in.

- **Source highlighting:** showing which sentence in the pasted text drove each field value. The textarea component should support overlaid highlight spans in v2.
- **Low-confidence hints:** a visual flag (e.g., yellow chip) on fields filled with lower-confidence values. The data model should preserve a confidence score per extracted field even if it is not displayed in v1.
- **Multi-calculator support:** the extraction pipeline should be designed as a per-calculator configuration, not hard-coded to CHA₂DS₂-VASc or EuroSCORE II.
- **Keyboard shortcut:** a global shortcut (e.g., Cmd+Shift+V) to paste and auto-fill without clicking into the textarea first.

---

## 6. UX Flow

> **Design principle:** One interaction. Zero surprises. The cardiologist pastes once and the form is done. Every design decision should be evaluated against this principle.

### 6.1 Page layout

The calculator page uses a two-column grid with **equal width columns (50/50)**. The left column contains the `tiro-magic-clipboard` card; the right column contains the calculator form.

**Left column: Paste area card**

- Card header: "Klinisch verslag" with a missing-fields badge that shows the count of unanswered fields after extraction.
- `tiro-magic-clipboard` textarea: resize vertical, placeholder text localised per language.
  - NL: "Plak hier uw verslag en extraheer automatisch data voor de berekening"
  - EN: "Paste the patient's clinical report here..."
- "Verslag verwerken" button (NL) / "Process report" (EN): triggers extraction on click.

**Right column: Calculator form**

No change to layout. Fields auto-populate in place. The optional brief highlight animation (P1) is the only visual change to this column.

### 6.2 States

| State | What the user sees |
|---|---|
| **Empty** | Textarea with placeholder. Score panel empty. No field highlights. |
| **Pasted** | Textarea shows report. "Verslag verwerken" button available. No field changes yet. |
| **Processing** | SDK extracting data. Form fields being populated by SDK. |
| **Filled** | Form fields populated. Missing-fields badge visible in card header. Unanswered fields have orange outline. |
| **Completing** | User fills highlighted fields one by one. Each filled field flashes green then clears. Badge count decrements. |
| **Done** | All fields filled. Badge shows "Alle velden ingevuld" briefly then hides. Score panel visible. |

---

## 7. PostHog Event Instrumentation

All events are sent via the existing PostHog instance (`eu.i.posthog.com`). **No personal health information (PHI) is included in any event property. The pasted clinical text must never be sent to PostHog.**

| Event name | Trigger | Key properties |
|---|---|---|
| `autofill_text_pasted` | User pastes text into the textarea | `calculator_name`, `text_char_length`, `has_existing_fill` (bool) |
| `autofill_extraction_started` | SDK begins processing the pasted text | `calculator_name`, `text_char_length` |
| `autofill_extraction_completed` | SDK returns extracted values | `calculator_name`, `fields_filled`, `fields_total`, `fill_rate` (0-1), `duration_ms` |
| `autofill_extraction_failed` | SDK returns an error | `calculator_name`, `error_type`, `duration_ms` |
| `autofill_field_overridden` | User manually changes an auto-filled field | `calculator_name`, `field_id`, `was_autofilled` (bool), `new_value_code` |
| `autofill_text_cleared` | User clicks the Clear button | `calculator_name`, `fields_were_filled` (bool), `time_since_paste_ms` |
| `calculator_score_calculated` | Score computation completes (all fields filled) | `calculator_name`, `score`, `used_autofill` (bool), `autofilled_fields_count`, `manual_fields_count` |

> `field_id` values must match the FHIR Questionnaire `linkId` values (e.g., `age`, `sex`, `chf`, `hypertension`, `stroke`, `vascular`, `diabetes` for CHA₂DS₂-VASc).

---

## 8. Success Metrics

Metrics are evaluated at 1 week, 30 days, and 90 days post-launch using PostHog dashboards.

| Metric | Type | Target (30 days) | PostHog query |
|---|---|---|---|
| Adoption rate | Leading | >= 60% of sessions | `autofill_text_pasted` / `calculator_page_viewed` |
| Field fill-rate (CHA₂DS₂-VASc) | Leading | >= 70% avg | `avg(fill_rate)` from `autofill_extraction_completed` |
| Score calculation rate | Leading | >= 80% of auto-fill sessions | `calculator_score_calculated` where `used_autofill = true` |
| Extraction error rate | Leading | <= 5% | `autofill_extraction_failed` / `autofill_extraction_started` |
| Field override rate | Leading | <= 20% of auto-filled fields | `autofill_field_overridden` / total fields filled |
| Clear rate | Leading | <= 10% of paste sessions | `autofill_text_cleared` / `autofill_text_pasted` |
| p95 extraction latency | Leading | <= 3 000 ms | `p95(duration_ms)` from `autofill_extraction_completed` |

---

## 9. Open Questions

| Question | Owner | Blocking? | Notes |
|---|---|---|---|
| Does `tiro-magic-clipboard` natively support a textarea as a trigger, or does it require refactoring to fire on a paste event? | Engineering | Yes | The current component uses a button click. A paste event listener on a textarea may require a new component API. |
| What is the extraction accuracy baseline on real CHA₂DS₂-VASc discharge letters (nl-BE)? | Engineering / Data | Yes (for target-setting) | Run a small offline eval on 20-30 anonymised reports before launch to calibrate `fill_rate` targets. |
| Should the textarea be pre-populated when the calculator page is opened from a deep link that includes a report ID? | Product | No (v2 scope) | Deferred; the paste-in model covers 100% of v1 use cases. |
| Which EuroSCORE II fields are most commonly present in cardiology reports, and what is the expected fill-rate? | Clinician / Data | No | Inform targets for the EuroSCORE II calculator once it is added to the feature. |
| Should the error message text be localisable (nl-BE / EN)? | Engineering / Design | No | App currently defaults to nl-BE; English support is a v2 concern. Hardcode nl-BE strings for now. |

---

## 10. Timeline Considerations

- **Hard deadlines:** None identified. No contractual or compliance dates tied to this feature.
- **Dependencies:** Tiro Web SDK team must confirm whether `tiro-magic-clipboard` supports paste-event triggering or needs a new API surface. This is the only blocking external dependency.

### Suggested phasing

1. **Phase 1 (complete):** Paste-and-button UX for all 4 calculators. Field highlighting after autofill. 50/50 layout. Localised placeholder text.
2. **Phase 2 (1 week after Phase 1 data):** Review PostHog extraction accuracy data, calibrate confidence threshold, improve field count accuracy in badge.
3. **Phase 3 (post-launch):** Full PostHog re-instrumentation, `autofill_field_overridden` with field-level diff, graceful error states.

---

## Appendix: CHA₂DS₂-VASc Fields Reference

| linkId | Label (nl-BE) | SNOMED CT code | Answer options (weight) |
|---|---|---|---|
| `age` | Leeftijd | 397669002 | <65 (0), 65-74 (1), ≥75 (2) |
| `sex` | Geslacht | 184100006 | man (0), vrouw (1) |
| `chf` | Hartfalen (CHF) | 161505003 | nee (0), ja (1) |
| `hypertension` | Hypertensie | 161501007 | nee (0), ja (1) |
| `stroke` | Beroerte/TIA/TE | 275526006 | nee (0), ja (2) |
| `vascular` | Vaatziekte (MI/PAD) | 266995000 | nee (0), ja (1) |
| `diabetes` | Diabetes mellitus | 161445009 | nee (0), ja (1) |
