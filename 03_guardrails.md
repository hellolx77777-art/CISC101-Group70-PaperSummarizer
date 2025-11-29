# Module 3: Guardrails

## Purpose
Ensure academic integrity, prevent hallucinations, and maintain strict constraint enforcement.

## Responsibilities
- Enforce content fidelity:
  - Summaries must reflect only the provided text
  - No fabricated details or citations
- Detect empty or extremely short sections
- Apply long-text chunking strategies for papers exceeding token limits
- Verify word limits:
  - Section summaries ≤ 150 words

## Inputs
- Section summaries
- Raw section data
- Alerts from Module 1 and Module 2

## Outputs
- Cleaned, validated summaries
- Warnings:
  - Missing sections
  - Short sections
  - Potential inconsistency flags
