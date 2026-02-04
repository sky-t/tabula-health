# AI Evaluation Framework for Tabula Health Generator

## Overview

An evaluation system to measure quality of generated synthetic healthcare data across three dimensions: **format validity**, **terminology accuracy**, and **clinical coherence**.

---

## File Structure

```
lib/eval/
├── index.ts                    # Main exports
├── types.ts                    # Evaluation types and interfaces
├── validators/
│   ├── index.ts
│   ├── fhir-validator.ts       # FHIR R4 JSON validation
│   ├── ccda-validator.ts       # C-CDA XML validation
│   ├── hl7v2-validator.ts      # HL7v2 structure validation
│   └── terminology-validator.ts # Code system validation
├── judges/
│   ├── index.ts
│   ├── clinical-coherence.ts   # LLM-as-judge evaluator
│   └── prompts.ts              # Evaluation prompt templates
├── suites/
│   ├── index.ts
│   └── test-prompts.ts         # Curated test cases
├── runners/
│   ├── index.ts
│   └── eval-runner.ts          # Orchestration
└── reporters/
    ├── index.ts
    ├── console-reporter.ts
    └── json-reporter.ts

scripts/
└── run-eval.ts                 # CLI entry point
```

---

## Scoring Approach

### Overall Score Formula

```
Overall = (Format × 0.25) + (Terminology × 0.25) + (Clinical × 0.50)
```

### Thresholds

| Status | Score Range |
|--------|-------------|
| Pass   | ≥ 80        |
| Warn   | 60-79       |
| Fail   | < 60        |

### Clinical Dimension Weights

| Dimension                    | Weight |
|------------------------------|--------|
| Medication-Condition Alignment | 25%   |
| Lab Value Plausibility        | 20%   |
| Narrative Coherence           | 25%   |
| Temporal Consistency          | 15%   |
| Demographic Appropriateness   | 15%   |

### Finding Severity Penalties

| Severity | Penalty |
|----------|---------|
| Critical | -20     |
| Major    | -10     |
| Minor    | -2      |
| Info     | 0       |

---

## Implementation Details

### Phase 1: Types and Core Validators

**Files:** `lib/eval/types.ts`, `lib/eval/validators/*.ts`

- `EvaluationResult`, `Score`, `CheckResult` types
- FHIR validator: JSON parse, Bundle structure, resource checks, watermarks
- C-CDA validator: XML structure, template IDs, sections, watermarks
- HL7v2 validator: Segment presence, field separators, watermarks
- Terminology validator: Leverages `lib/terminology/` for ICD-10, RxNorm, LOINC, SNOMED validation

### Phase 2: LLM-as-Judge

**Files:** `lib/eval/judges/clinical-coherence.ts`, `lib/eval/judges/prompts.ts`

Uses OpenAI API (gpt-4o-mini) with function calling for structured output.

**Evaluates 5 dimensions (1-5 scale each):**
1. Medication-Condition Alignment - Are medications appropriate for conditions?
2. Lab Value Plausibility - Are lab values clinically reasonable?
3. Patient Narrative Coherence - Does the patient story make sense?
4. Temporal Consistency - Are dates and timelines logical?
5. Demographic Appropriateness - Are conditions age/sex appropriate?

### Phase 3: Test Suite

**Files:** `lib/eval/suites/test-prompts.ts`

12 curated test prompts across 7 categories:

| Category            | Tests | Description                          |
|---------------------|-------|--------------------------------------|
| diabetes_management | 2     | Type 2 with complications, Type 1    |
| pregnancy_care      | 2     | Normal prenatal, high-risk           |
| elderly_complex     | 2     | Multiple chronic conditions, dementia |
| pediatric_screening | 2     | Well child, pediatric asthma         |
| mental_health       | 1     | Depression and anxiety               |
| emergency_care      | 1     | Acute chest pain                     |
| edge_case           | 2     | Minimal info, complex multi-condition |

Each test case includes:
- Expected characteristics (conditions, medications, labs, age range)
- Minimum score thresholds

### Phase 4: Runner and Reporters

**Files:** `lib/eval/runners/eval-runner.ts`, `lib/eval/reporters/*.ts`

**Pipeline:**
1. Generate persona from prompt (via `/api/plan-persona`)
2. Enrich persona with terminology codes
3. Generate messages (FHIR, C-CDA, HL7v2)
4. Run format validation
5. Run terminology validation
6. Run LLM-as-judge clinical evaluation
7. Aggregate scores and produce results

**Reporters:**
- Console: Colorized CLI output with progress
- JSON: Structured output for CI/artifacts (saved to `./eval-results/`)

### Phase 5: CLI Integration

**File:** `scripts/run-eval.ts`

```bash
# Run full test suite
pnpm eval:batch

# Run specific category
pnpm eval --category diabetes_management

# Evaluate single prompt
pnpm eval --prompt "Create a 45-year-old with heart failure"

# Options
pnpm eval --help
  --batch, -b              Run full test suite
  --category, -c <name>    Run category subset
  --prompt, -p <text>      Single evaluation
  --formats, -f <list>     Comma-separated (default: fhir,ccda,hl7v2)
  --verbose, -v            Detailed output
  --output, -o <dir>       JSON output directory
```

---

## Key Dependencies

| Purpose              | File                          |
|----------------------|-------------------------------|
| Existing validation  | `lib/generators/index.ts`     |
| Terminology service  | `lib/terminology/index.ts`    |
| OpenAI pattern       | `app/api/plan-persona/route.ts` |
| Core types           | `lib/types.ts`                |
| Persona enrichment   | `lib/generators/enrich-persona.ts` |

---

## Environment Variables

Required in `.env.local`:

```
OPENAI_API_KEY=sk-...      # For LLM-as-judge clinical evaluation
UMLS_API_KEY=...           # Optional: For SNOMED code validation
```

---

## Verification

```bash
# 1. Build check
pnpm build

# 2. Run full suite
pnpm eval:batch

# 3. Check JSON output
ls ./eval-results/

# 4. Review pass rate and identify regressions
```

---

## Sample Output

```
════════════════════════════════════════════════════════════
  EVALUATION RESULTS
════════════════════════════════════════════════════════════

Summary:
  Total Tests: 12
  ✓ Passed: 8
  ! Warned: 3
  ✗ Failed: 1

  Pass Rate: 67%
  Average Score: 74

  Duration: 57.1s

────────────────────────────────────────────────────────────
```

Individual test output:
```
Overall: 82
  Format: 100
  Terminology: 100
  Clinical: 64

Findings:
  Critical:
    • No lab tests listed for diabetic patient
  Major:
    • Condition onset date is in the future
```

---

## Enterprise Validation Roadmap

The current validators perform structural checks sufficient for development and demo use. For generating test data that will be ingested into enterprise EHR or HIE clinical data repositories, the following improvements are needed.

### Current Validator Limitations

| Aspect | FHIR | HL7v2 | C-CDA | Terminology |
|--------|------|-------|-------|-------------|
| Parsing | JSON structural | Regex/string split | Heuristic tag counting | N/A |
| Data type validation | None | None | None | N/A |
| Cardinality enforcement | Minimal | None | None | N/A |
| Value set binding | None | Sex only | None | Partial |
| Identifier format validation | None | None | None | N/A |
| Date/time timezone handling | None | Basic regex | None | N/A |
| Profile/IG conformance | None | None | None | N/A |

### Priority 1: Replace Heuristic Parsing With Real Parsers

The single most impactful change. Current validators use regex and string matching instead of proper parsers, which means malformed output can pass validation.

- **C-CDA**: Use a real XML parser (e.g., `fast-xml-parser` or `libxmljs2`) instead of heuristic tag counting
- **FHIR**: Integrate the official FHIR Validator or use a JSON Schema for FHIR R4 resources for profile-level conformance
- **HL7v2**: Use an HL7v2 parser library (e.g., `hl7js` or `node-hl7-complete`) that understands segment/field/component/subcomponent hierarchy

### Priority 2: Add HL7v2 Data Type Validation

HL7v2 fields have structured data types that are completely unvalidated. Interface engines (Rhapsody, Mirth Connect) will reject messages with malformed fields before they reach the EHR.

| Data Type | Structure | Used In |
|-----------|-----------|---------|
| CX (Composite ID) | `ID^^^AssigningAuthority^^^IDType` | PID-3 (Patient Identifier) |
| XPN (Person Name) | `Family^Given^Middle^Prefix^Suffix^Degree` | PID-5 (Patient Name) |
| CWE (Coded With Exceptions) | `Code^Text^CodingSystem^AltCode^AltText^AltSystem` | OBX-3, OBR-4 (Codes) |
| XAD (Address) | `Street^Other^City^State^Zip^Country` | PID-11 (Address) |
| XTN (Telephone) | `PhoneNumber^TelecomUseCode^...` | PID-13 (Phone) |

### Priority 3: Enforce Value Set Bindings

Enterprise systems validate coded fields against specific value sets. Currently only PID-8 (sex) is validated. Required value sets:

- Administrative gender (FHIR: `http://hl7.org/fhir/ValueSet/administrative-gender`)
- Marital status, race/ethnicity (required for US Realm / Meaningful Use)
- Encounter type, discharge disposition
- Problem status (active, inactive, resolved)
- Medication route (PO, IV, IM, etc.)
- Allergy severity/criticality
- Result interpretation codes (H, L, N, A for OBX-8 in HL7v2)

Consider using the VSAC (Value Set Authority Center) API or embedding required value sets.

### Priority 4: Add US Core Profile Validation (FHIR)

Most US EHRs require US Core Implementation Guide compliance. The FHIR validator currently does not check:

- Must Support elements
- Required value set bindings (e.g., Patient.gender)
- Extensions for race, ethnicity, birth sex
- Resource reference resolution patterns

The official HL7 FHIR Validator JAR can be wrapped as a CLI call for full US Core conformance checking.

### Priority 5: Validate Identifiers

EHRs and HIEs route and match patients based on identifiers. Invalid identifiers cause duplicate records or failed matches.

- **OID syntax** validation (e.g., `2.16.840.1.113883.x.x.x`)
- **NPI** format (10-digit with Luhn check digit)
- **MRN** patterns per facility conventions
- **Assigning Authority** OIDs must be properly formed

### Priority 6: Date/Time Timezone Enforcement

Missing or incorrect timezones cause real ingestion failures.

- **FHIR**: `instant` type requires timezone offset; `dateTime` has variable precision rules
- **HL7v2**: Timestamps should include `+/-ZZZZ` offset (e.g., `20240115120000-0500`)
- **C-CDA**: `effectiveTime` requires precision appropriate to context

### Priority 7: Validate C-CDA Entry-Level Content

The current C-CDA validator only checks section presence. Enterprise CDRs validate deeper:

- Entry templates (e.g., Problem Observation `2.16.840.1.113883.10.20.22.4.4`)
- Required Act/Observation structure within each section
- StatusCode elements (completed, active, etc.)
- EffectiveTime with proper precision
- Narrative-to-entry consistency

### Priority 8: UCUM Unit Validation for Lab Results

Lab results with non-standard units get rejected or misinterpreted. The terminology validator checks unit presence but not validity. Validate against UCUM using `@lhncbc/ucum-lhc`.

### Priority 9: Clinical Value Plausibility Checks

Beyond range validity (min < max), check physiological plausibility:

- Glucose: 0-2000 mg/dL
- Heart rate: 0-300 bpm
- Blood pressure: systolic > diastolic
- Age-appropriate lab reference ranges

### Priority 10: Cross-Format Consistency Validation

When generating the same patient across FHIR, HL7v2, and C-CDA, validate that:

- Patient demographics match across formats
- Clinical codes map correctly between formats
- Identifiers are consistent
- Timestamps align

### Suggested Libraries

| Purpose | Library | Notes |
|---------|---------|-------|
| XML parsing (C-CDA) | `fast-xml-parser` or `libxmljs2` | Replace heuristic tag counting |
| HL7v2 parsing | `hl7js` or `node-hl7-complete` | Full data type support |
| FHIR validation | HL7 FHIR Validator JAR (CLI wrapper) | US Core conformance |
| UCUM units | `@lhncbc/ucum-lhc` | Lab result unit validation |
| Value sets | VSAC API | Authoritative US value sets |

---

## Clinical Coherence Evaluation Methodology

Based on the methodology recommended by Hamel Husain and Shreya Shankar in *AI Evals FAQ* (Nov 2025). The core insight: **start with error analysis, not infrastructure.** Build evaluation criteria from observed failures, not hypothesized ones. Use binary (pass/fail) evaluators per failure mode.

The generation pipeline has one non-deterministic step (LLM persona generation) and two deterministic steps (terminology enrichment, format generation). These require different evaluation strategies:

| Layer | Nature | Eval method |
|-------|--------|-------------|
| **Persona generation** (LLM) | Non-deterministic, subjective quality | Manual trace annotation → scoped binary judges |
| **Terminology enrichment** | Deterministic lookup | Automated tests with fixed inputs and expected outputs |
| **Format generators** (FHIR/C-CDA/HL7v2) | Deterministic transformation | Automated tests — known-good persona in, valid output asserted |

The manual annotation effort — the expensive, judgment-intensive work — is scoped to the LLM-generated persona. That's where non-determinism lives and where human judgment is needed to establish what "good" means. Format validity and terminology accuracy are deterministic and testable with traditional automated tests; mixing generator bugs into the annotation workflow would dilute the error analysis with noise that's cheaper to catch automatically.

The methodology below builds clinical coherence ground truth through manual persona annotation, then uses it to develop and calibrate scoped automated judges.

### High-Level Plan (7 Steps)

| Step | Activity | Output |
|------|----------|--------|
| 1 | **Generate Trace Dataset** | 100+ traces saved to `eval-results/traces/` |
| 2 | **Build Custom Trace Viewer** | Annotation tool with structured data capture |
| 3 | **Error Analysis (Open Coding)** | Free-text annotations on 100+ traces via the viewer |
| 4 | **Build Failure Taxonomy (Axial Coding)** | Prioritized failure modes with frequency counts |
| 5 | **Build Binary Per-Failure-Mode Evaluators** | Scoped judges, each answering one question: "does this trace have [problem X]?" |
| 6 | **Validate Judges Against Human Labels** | TPR/TNR metrics per judge using annotations as ground truth |
| 7 | **Iterate: Use Eval Findings to Improve Generators** | Prompt fixes, enrichment improvements, regression tests |

### Annotation Schema

The annotations produced during trace review are the primary output of the eval process. They must be structured, tied to traces by ID, and stored separately from traces so traces remain immutable and annotations can be re-coded as the failure taxonomy evolves.

Annotations are saved to `eval-results/annotations.json` as an array of records:

```json
{
  "traceId": "eval-2024-01-15-abc123",
  "verdict": "fail",
  "notes": "Diabetes patient has no HbA1c in labs. Metformin start date is 6 months before diabetes onset.",
  "failureMode": null,
  "severityTag": "clinician_would_notice",
  "reviewer": "sky-t",
  "timestamp": "2025-02-03T14:30:00Z",
  "reviewPass": 1
}
```

#### Field Definitions

| Field | Type | When Populated | Description |
|-------|------|----------------|-------------|
| `traceId` | string | Step 3 (open coding) | Links to the source trace in `eval-results/traces/`. Immutable key. |
| `verdict` | `"pass"` \| `"fail"` | Step 3 (open coding) | Binary judgment. Forced choice — no "maybe". |
| `notes` | string | Step 3 (open coding) | Free-text open coding notes. Write what you see, not what category it belongs to. This is the raw material for axial coding. |
| `failureMode` | string \| null | Step 4 (axial coding) | Starts `null`. Populated when notes are categorized into failure taxonomy (e.g., `missing_condition_relevant_labs`, `temporal_inconsistency`). May be re-coded as taxonomy evolves. |
| `severityTag` | string \| null | Step 3 (open coding) | Optional quick severity signal. One of: `clinician_would_notice` (obvious to domain expert), `subtle_issue` (requires careful review), `cosmetic` (technically wrong but not clinically meaningful). |
| `reviewer` | string | Step 3 (open coding) | Who reviewed. For provenance and multi-reviewer agreement measurement. |
| `timestamp` | ISO 8601 string | Step 3 (open coding) | When the annotation was created or last modified. |
| `reviewPass` | integer | Step 3 (open coding) | Which review cycle this annotation belongs to (1 = initial, 2 = after axial coding re-review, etc.). Supports iterative refinement. |

#### Design Decisions

- **Annotations stored separately from traces.** Traces in `eval-results/traces/` are immutable source data. Annotations in `eval-results/annotations.json` can be re-coded, updated, and versioned independently.
- **`failureMode` starts null.** During open coding (Step 3), you write free-text notes. During axial coding (Step 4), you group notes into categories and backfill `failureMode`. This two-pass approach prevents premature categorization.
- **`reviewPass` supports iteration.** After axial coding, you'll re-review traces with new failure modes in mind. The pass number tracks which cycle produced each annotation.

### Step 1 (Detailed): Generate Trace Dataset

Run the eval suite across all 12 test cases to generate personas. Supplement with 15-20 custom prompts covering edge cases for a total of ~100 traces. Each trace captures one persona — the unit of manual annotation — not one per output format.

Each trace includes:

- The input prompt (e.g., "45-year-old male with Type 2 diabetes and CKD")
- The generated persona (demographics, conditions, medications, labs, encounters)
- Automated format/terminology scores (for reference, not the focus of annotation)

Save all traces to `eval-results/traces/` as JSON for reproducibility.

```bash
# Generate trace dataset
pnpm eval:batch --output eval-results/traces --verbose
```

#### Portfolio Deliverables from Step 1

- Trace dataset in `eval-results/traces/`
- Devlog entry documenting trace generation approach and prompt coverage

### Step 2 (Detailed): Build Custom Trace Viewer

Build the trace viewer **before** starting manual review. Per Husain/Shankar, custom annotation tools are "the single most impactful investment" for eval workflows, and teams with custom tools iterate ~10x faster. Reviewing 100+ traces in raw JSON is unsustainable — the viewer makes the difference between abandoning the process and completing it.

#### Minimum Viable Viewer

- **Main panel:** Rendered persona (demographics, conditions with codes, medications with codes, labs with ranges, encounters) — this is the primary review target
- **Reference panel:** Generated output available for inspection when needed (syntax-highlighted JSON for FHIR, formatted XML for C-CDA, segment-parsed view for HL7v2) — useful for spotting persona issues that only become visible in the rendered output, but not the annotation focus
- **Bottom bar:** Pass/Fail buttons, free-text notes field, `severityTag` selector
- **Progress:** "Trace 23 of 100" with completion percentage
- **Keyboard shortcuts:** `N` next, `P` previous, `1` pass, `2` fail, `S` save
- **Persistence:** Annotations auto-save to `eval-results/annotations.json` on every verdict

#### Not in V1

- Clustering, filtering, or search (add after you have enough annotated data)
- Auto-evaluator results displayed alongside traces (add in Step 5)
- `failureMode` dropdown (not needed until axial coding in Step 4)
- Multi-reviewer support (single reviewer is fine for initial pass)

#### Implementation Notes

- Build as a Next.js page within the existing app (reuse existing UI components)
- Load traces from `eval-results/traces/` via API route
- Write annotations to `eval-results/annotations.json` via API route
- Keep it minimal — only add features that earn their complexity

#### Portfolio Deliverables from Step 2

- The trace viewer itself (visible in the app)
- Devlog entry showing the viewer and explaining design decisions

### Step 3 (Detailed): Error Analysis (Open Coding)

This is the most important step. It builds domain intuition and determines what the evals should actually measure. Per Husain/Shankar: "Error analysis is the most important activity in evals. Error analysis helps you decide what evals to write in the first place."

The focus is on the LLM-generated persona — the non-deterministic output where human judgment is needed. Format and terminology issues in the deterministic pipeline are caught by automated tests, not manual annotation.

Use the trace viewer built in Step 2. For each trace:

1. Read the persona — does the clinical picture make sense as a whole?
2. If something looks off, check the generated output for confirmation — sometimes persona issues only become visible when rendered as a FHIR bundle or C-CDA document
3. Mark pass or fail (forced binary choice)
4. If fail, write notes describing what's wrong in the persona. Focus on clinical coherence — the thing only human judgment can assess.
5. Optionally set `severityTag`
6. Move to next trace

**What to look for:**
- Does this patient's clinical picture make sense as a whole?
- Would a clinician look at this persona and find it plausible?
- Are there missing, contradictory, or implausible clinical relationships?

**Example notes:**
- "Diabetes patient has no HbA1c in labs"
- "Metformin listed but no diabetes in conditions"
- "Patient is 8 years old but has hypertension meds"
- "Condition onset date is 2 years after medication start date"
- "No encounter generated despite hospitalization in the prompt"

**Reach theoretical saturation:** Keep reviewing until new traces stop revealing new failure types. Aim for 100+ traces. If you're still finding new categories after 100, keep going.

#### Portfolio Deliverables from Step 3

- Completed `eval-results/annotations.json` with 100+ annotated traces
- Devlog entry with observations and surprises from the review process

### Step 4: Build Failure Taxonomy (Axial Coding)

From Step 3's open coding notes:

1. Export all `notes` fields from annotations where `verdict === "fail"`
2. Group similar notes into failure mode categories (an LLM can propose initial groupings, but you make the final call)
3. Name each category clearly (e.g., `missing_condition_relevant_labs`, `temporal_inconsistency`)
4. Go back through annotations and populate the `failureMode` field (set `reviewPass` to 2)
5. Count frequency per category
6. Classify each as **specification failure** (the prompt/instructions are wrong — fix the generator) vs. **generalization failure** (the LLM fails to follow correct instructions — candidate for automated evaluator)
7. Fix specification failures immediately — don't build evals for things you can fix by improving the prompt

#### Portfolio Deliverables from Step 4

- `docs/ERROR_ANALYSIS.md` — Failure taxonomy with frequency counts and examples
- Updated `eval-results/annotations.json` with `failureMode` populated
- Devlog entry documenting the taxonomy and classification decisions

### Step 5: Build Binary Per-Failure-Mode Evaluators

For each **generalization failure** that persists after fixing prompts:
- Determine if it can be caught with code (assertion, regex, lookup table) — prefer this, it's cheaper and deterministic
- If it requires subjective judgment, build a scoped binary LLM-as-judge evaluator
- Each evaluator answers one question: "Does this trace exhibit [failure X]? Pass/Fail + critique"
- Start with the most capable model available, optimize for cost later

### Step 6: Validate Judges Against Human Labels

The annotations from Steps 3-4 become ground truth for validating automated judges:
- Split annotated traces into train (for prompt iteration) and test (for measuring judge accuracy)
- Run each automated judge against the test set
- Measure True Positive Rate (catches real failures) and True Negative Rate (doesn't flag good traces)
- Iterate on judge prompts until aligned with human annotations
- Document the alignment metrics — this is what gives the eval credibility

### Step 7: Iterate

Use eval findings to improve generators:
- Fix prompt gaps discovered in error analysis
- Add regression tests for fixed failures
- Re-run error analysis after changes to verify improvements and discover new failure modes
- Track experiments, not features — count how many eval cycles you've completed
- Each cycle produces a new `reviewPass` in annotations, showing evolution over time
