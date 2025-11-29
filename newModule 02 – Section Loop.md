# Module 02 – Section Loop

## Change Log
- Added `summary_level` configuration variable to control short vs detailed section summaries.
- Implemented conditional behavior inside the section loop for `summary_level = "short"` vs `summary_level = "detailed"`.
- Ensured that in detailed mode each section produces BOTH a paragraph and a bullet list of 3–5 key points.

---

## Purpose

Module 02 is responsible for looping over all normalized paper sections and generating section-level summaries that respect word limits, target audience, and guardrail rules.

This module assumes that:

- Module 01 (Intake & Setup) has already:
  - Normalized the section names.
  - Built an ordered list of sections to summarize.
  - Detected which sections are missing or very short.
- Module 03 (Guardrails) will be used to:
  - Enforce evidence and hallucination rules.
  - Provide standardized warning messages where needed.

---

## Inputs

- `paper_text`  
  Full research paper text, already segmented by section where possible.

- `normalized_sections`  
  An ordered list of canonical section names and their associated text blocks, for example:  
  `["Abstract", "Introduction", "Methodology", "Results", "Discussion", "Conclusion"]`.

- `target_audience`  
  One of: `domain expert`, `interdisciplinary researcher`, `educated layperson`.

- `summary_level`  
  Controls how detailed each section’s summary should be. Must be one of:
  - `"short"` – compact summary (1–2 sentences).
  - `"detailed"` – short paragraph + bullet list of 3–5 key points.

- `evidence_mode` (forwarded from higher-level configuration)  
  Passed through to Module 03 so that strict evidence rules can be applied if needed.

- `section_metadata`  
  Optional metadata from Module 01 and Module 03 indicating whether a section is:
  - present / missing
  - empty
  - word count (to detect `< 50` word sections).

---

## Outputs

For each section in `normalized_sections`, this module produces:

- A structured section summary object containing:
  - `section_name`
  - `summary_paragraph` (may be empty in some modes)
  - `summary_bullets` (list of bullet strings; may be empty in short mode)
  - `word_count_estimate`
  - Any guardrail warnings provided by Module 03.

All section summary objects are passed to Module 04 (Rendering & Refinement) to be formatted into:

- The Section-by-Section Table.
- Parts of the Paper Summary, Expert Summary, Layperson Summary, and Checks & Warnings.

---

## High-Level Behavior

1. Receive `normalized_sections`, `paper_text`, `target_audience`, `summary_level`, and `section_metadata`.
2. For each section in order:
   - Check if the section is missing, empty, or very short using `section_metadata`.
   - If the section has no usable text, call Module 03 to produce a standardized warning and do not attempt a normal summary.
   - Otherwise, generate the section summary according to `summary_level`.
3. Send the raw summary text and metadata to Module 03 for evidence checks and hallucination mitigation.
4. Store the final, guardrail-safe summary for Module 04.

---

## Section Loop – Detailed Logic

### 1. Initialize

- Create an empty list `section_summaries`.
- For each `section` in `normalized_sections`, retrieve:
  - `section_name`
  - `section_text`
  - `section_word_count`
  - Flags such as `is_missing`, `is_empty`, `is_very_short`.

### 2. Handle Missing / Empty / Very Short Sections

For each section:

- If `is_missing == true` OR `is_empty == true`:
  - Call Module 03 (Guardrails) to generate a standardized warning, for example:  
    `"Section skipped: no usable text was provided."`
  - Create a section summary object with:
    - `summary_paragraph = ""`
    - `summary_bullets = []`
    - `warning` set to the guardrail message.
  - Append this object to `section_summaries` and move to the next section.

- Else if `is_very_short == true` (e.g., `< 50` words):
  - Continue summarization but request an additional warning from Module 03, e.g.:  
    `"Section very short: summary may be incomplete."`
  - Attach this warning to the final summary output for this section.

### 3. Apply `summary_level` Modes

For sections with usable text:

- **If `summary_level = "short"`:**
  - Generate a **compact summary** of the section in **1–2 sentences**, with:
    - Maximum length: **≤ 150 words**.
    - Focus on the **central purpose, method, and key findings** of the section.
    - Language tuned to `target_audience` (more technical for experts, more explanatory for laypersons).
  - Do **not** generate a bullet list in this mode.
  - Store the result as:
    - `summary_paragraph = short_summary`
    - `summary_bullets = []`

- **If `summary_level = "detailed"`:**
  - Generate a **short paragraph** (up to 150 words) that:
    - Summarizes the main idea of the section.
    - Provides context appropriate to the `target_audience`.
  - Then generate a **bullet list of 3–5 key points** that:
    - Highlight important methods, results, assumptions, or implications.
    - Are written as concise bullet items, not full paragraphs.
  - Store the result as:
    - `summary_paragraph = detailed_paragraph`
    - `summary_bullets = [bullet_1, bullet_2, bullet_3, ...]`

In both modes:

- Respect the PS2 constraints on word counts.
- Do not introduce content that is not supported by the section text.

### 4. Guardrail Integration

After generating the draft summary for a section:

1. Pass `section_text`, `summary_paragraph`, `summary_bullets`, `evidence_mode`, and any existing warnings to Module 03.
2. Module 03 will:
   - Enforce hallucination and evidence rules.
   - Potentially revise or reject summaries in strict modes.
   - Add additional warnings if needed.
3. This module accepts the guardrail-checked summary and stores it in `section_summaries`.

---

## Output to Module 04

At the end of the loop:

- Return `section_summaries` as a structured list to Module 04 (Rendering & Refinement).
- Module 04 is responsible for:
  - Converting `section_summaries` into the Section-by-Section Table.
  - Reusing paragraphs and bullet lists in the Paper Summary, Expert Summary, Layperson Summary, Mini-Glossary cues, and Checks & Warnings.
