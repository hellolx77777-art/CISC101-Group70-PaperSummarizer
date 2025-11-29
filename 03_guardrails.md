# Module 03 – Guardrails

## Change Log
- Added `evidence_mode` configuration with support for `evidence_mode = "strict"` to restrict summaries to only the provided source text.
- Implemented standardized warning messages for missing, empty, and very short sections (e.g., `< 50` words).
- Strengthened hallucination mitigation behavior and integrated it with section-level summaries and system-wide Checks & Warnings.

---

## Purpose

Module 03 enforces all safety, evidence, and robustness rules for the Research Paper Summarizer. It ensures that:

- Summaries are grounded in the source text.
- Missing or low-quality input sections are clearly flagged.
- Length constraints and content boundaries are respected.
- The system’s behavior is transparent to the user.

This module is called by both:

- Module 01 (for initial detection of problems).
- Module 02 (to validate and refine individual section summaries).

---

## Inputs

For each section or summary, this module may receive:

- `section_name`  
  Canonical name of the section (e.g., `"Introduction"`).

- `section_text`  
  Raw text for the section (may be empty or missing).

- `section_word_count`  
  Word count of `section_text`.

- `draft_summary_paragraph`  
  Section summary paragraph produced by Module 02 (may be empty).

- `draft_summary_bullets`  
  List of bullet points produced by Module 02 (may be empty).

- `evidence_mode`  
  Controls how strictly evidence rules are enforced. Expected values:
  - `"normal"` – default behavior.
  - `"strict"` – strict evidence mode (no extrapolation beyond the text).

- `global_constraints`  
  Overall limits (e.g., max 150 words per section summary) and other relevant PS2 rules.

---

## Outputs

For each section, this module outputs a **guardrail-checked summary object** containing:

- `section_name`
- `final_summary_paragraph`
- `final_summary_bullets`
- `warnings` (list of warning strings, possibly empty)
- `evidence_notes` (optional, e.g., messages generated in strict mode)

These outputs are then consumed by Module 04 (Rendering & Refinement) to populate:

- The Section-by-Section Table.
- The Checks & Warnings section.
- Any visible notes about strict evidence behavior.

---

## Core Guardrail Components

### 1. Missing / Empty / Short Section Detection

This module standardizes how problematic sections are handled.

**Rules:**

- **Missing or empty section**  
  If `section_text` is missing, null, or only whitespace:
  - Do **not** attempt to generate a normal summary.
  - Emit a standard warning message, for example:  
    - `"Section skipped: no usable text was provided."`
  - Set:
    - `final_summary_paragraph = ""`
    - `final_summary_bullets = []`
    - `warnings = ["Section skipped: no usable text was provided."]`

- **Very short section (< 50 words)**  
  If `section_word_count < 50` but not empty:
  - Allow summary generation but attach an extra warning:  
    - `"Section very short: summary may be incomplete."`
  - Keep the summary from Module 02 but add this string to `warnings`.

These standardized messages must be reused consistently across the system so that:

- The Checks & Warnings section can list them clearly.
- Users understand why some sections look different or weaker.

---

### 2. Evidence Mode and Hallucination Mitigation

This module controls how strictly the system is allowed to interpret and paraphrase the source text.

#### 2.1 Evidence Mode Configuration

- `evidence_mode = "normal"`  
  - Allow concise paraphrasing and mild interpretation.
  - Still **forbid** introducing facts, results, or citations that are **not present** in the source text.
  - Encourage neutral wording and accurate representation of the section.

- `evidence_mode = "strict"`  
  - Activate **Strict Evidence Mode** (see below).
  - Summaries must be tightly bound to explicit statements in `section_text`.

#### 2.2 Strict Evidence Mode (Key Requirement)

When `evidence_mode = "strict"`:

1. Summaries must:
   - Only include claims, equations, results, and interpretations that **explicitly appear** in the provided `section_text`.
   - Avoid speculative language, extrapolation, or references to external models, papers, or prior knowledge.

2. If the model cannot confidently reconstruct a summary that is fully supported by the text, it must:
   - Avoid guessing.
   - Produce an explicit explanatory message, such as:  
     - `"The source text does not provide enough detail to summarize this section in strict evidence mode."`

3. Behavior in strict mode should be transparent and consistent:
   - Attach any such messages to `evidence_notes` and/or `warnings`.
   - Make it clear in the final output that the limitation arises from strict evidence rules, **not** from a model failure.

---

### 3. Length and Constraint Enforcement

This module also checks that:

- Each section summary paragraph respects the **≤ 150 word** limit.
- Bullet lists remain concise and relevant.
- The unified summary (managed mainly by other modules) respects the **≤ 250 word** limit.

If a draft summary from Module 02 violates these constraints:

- This module may:
  - Ask for a shorter rewrite (if running interactively), or
  - Instruct the system to trim or compress the summary while preserving factual correctness.
- Add a warning if heavy compression may have removed nuance, for example:  
  - `"Summary was shortened to satisfy word limit constraints; some details may be omitted."`

---

### 4. Integration with Other Modules

- **With Module 01 (Intake & Setup)**  
  - Share detection rules for missing / empty / very short sections.
  - Use consistent thresholds (e.g., `< 50` words) for classification.

- **With Module 02 (Section Loop)**  
  - Receive draft summaries and metadata from the section loop.
  - Apply evidence rules, strict mode, and standardized warnings.
  - Return a cleaned and fully validated section summary object.

- **With Module 04 (Rendering & Refinement)**  
  - Provide all warning messages and evidence notes so that:
    - The **Checks & Warnings** section can list them.
    - Users can see, for each section, if strict evidence mode or short text impacted the quality of the summary.

---

## Standard Warning Message Templates

This module defines the canonical wording for warnings, which should be reused everywhere:

- Missing / empty section:  
  - `"Section skipped: no usable text was provided."`

- Very short section (< 50 words):  
  - `"Section very short: summary may be incomplete."`

- Strict evidence mode insufficient information:  
  - `"The source text does not provide enough detail to summarize this section in strict evidence mode."`

By centralizing these messages here, the system remains consistent and easy to maintain.
