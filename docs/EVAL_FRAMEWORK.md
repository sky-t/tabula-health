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
│   ├── fhir-validator-cli.ts   # HL7 FHIR Validator JAR wrapper (US Core)
│   ├── ccda-validator.ts       # C-CDA XML validation
│   ├── hl7v2-validator.ts      # HL7v2 structure validation
│   ├── identifier-validator.ts # OID, NPI, UUID, URI validation
│   ├── datetime-validator.ts   # DateTime/timezone validation
│   ├── valueset-validator.ts   # Value set bindings validation
│   ├── ucum-validator.ts       # UCUM unit validation + normalizeUcumUnit()
│   ├── plausibility-validator.ts # Clinical value plausibility
│   ├── ccda-site-validator.ts # ONC SITE C-CDA conformance (--conformance)
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
    ├── json-reporter.ts
    └── trace-reporter.ts       # Full traces with persona + content

scripts/
└── run-eval.ts                 # CLI entry point

lib/traces/
└── production-trace.ts         # Vercel Blob trace persistence (save, list, get)

app/api/traces/
├── route.ts                    # GET list + POST enrich
└── [traceId]/
    └── route.ts                # GET single + PATCH annotations/feedback

tools/
└── trace-viewer/               # Shiny for Python annotation tool
    ├── app.py                  # Supports local + blob trace sources
    ├── requirements.txt
    └── README.md

eval-data/                      # Local prompt files (gitignored)
└── prompts/                    # CSV files for bulk trace generation

eval-results/                   # Generated artifacts (gitignored)
└── traces/                     # Full trace JSON files (local eval traces)
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
- FHIR validator: JSON parse, Bundle structure, resource checks, US Core profile validation (async, requires Java), watermarks
- C-CDA validator: `fast-xml-parser` for XML validation, DOM traversal for template IDs/sections, watermarks
- HL7v2 validator: `hl7v2` library for parsing, segment/field validation, data type validation (CX, XPN, CWE, XAD, XTN), watermarks
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
- Trace: Full traces with persona + content for manual annotation (saved to `./eval-results/traces/`)

### Phase 5: CLI Integration

**File:** `scripts/run-eval.ts`

```bash
# Run full test suite
pnpm eval:batch

# Run specific category
pnpm eval --category diabetes_management

# Evaluate single prompt
pnpm eval --prompt "Create a 45-year-old with heart failure"

# Evaluate single prompt and save trace
pnpm eval --prompt "Create a 45-year-old with heart failure" --save-traces

# Bulk evaluation from CSV file
pnpm eval --prompts-file eval-data/prompts/my-prompts.csv --save-traces

# Generate traces for full test suite
pnpm eval:traces

# Options
pnpm eval --help
  --batch, -b              Run full test suite
  --category, -c <name>    Run category subset
  --prompt, -p <text>      Single evaluation
  --prompts-file, -P <file> Bulk evaluation from CSV (one prompt per line)
  --formats, -f <list>     Comma-separated (default: fhir,ccda,hl7v2)
  --verbose, -v            Detailed output
  --output, -o <dir>       JSON output directory
  --save-traces, -t        Save full trace files with persona + content
  --no-judge               Skip LLM-as-judge clinical evaluation (saves API calls)
  --conformance            Enable external conformance validation (FHIR Validator JAR, ONC SITE C-CDA API)
```

---

## Key Dependencies

| Purpose              | File / Package                |
|----------------------|-------------------------------|
| Existing validation  | `lib/generators/index.ts`     |
| Terminology service  | `lib/terminology/index.ts`    |
| OpenAI pattern       | `app/api/plan-persona/route.ts` |
| Core types           | `lib/types.ts`                |
| Persona enrichment   | `lib/generators/enrich-persona.ts` |
| XML parsing (C-CDA)  | `fast-xml-parser` (npm)       |
| HL7v2 parsing        | `hl7v2` (npm)                 |
| Production traces    | `lib/traces/production-trace.ts`, `@vercel/blob` (npm) |

---

## Environment Variables

Required in `.env.local`:

```
OPENAI_API_KEY=sk-...        # For LLM-as-judge clinical evaluation
UMLS_API_KEY=...             # Optional: For SNOMED code validation
BLOB_READ_WRITE_TOKEN=...   # Required for production trace storage (Vercel Blob)
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
| Parsing | JSON structural | ✅ `hl7v2` library | ✅ `fast-xml-parser` | N/A |
| Data type validation | None | ✅ CX, XPN, CWE, XAD, XTN | None | N/A |
| Cardinality enforcement | ✅ US Core (requires Java) | None | None | N/A |
| Value set binding | ✅ Gender, status, severity, etc. | ✅ Sex, class, interpretation, route | ✅ Gender, marital, status | Partial |
| Identifier format validation | ✅ OID, URI, system | ✅ OID, NPI, assigning authority | ✅ OID (root attrs) | N/A |
| Date/time timezone handling | ✅ instant/dateTime, periods | ✅ DTM fields, timezone enforcement | ✅ effectiveTime, birthTime | N/A |
| UCUM unit validation | ✅ Observation valueQuantity, referenceRange, component | ✅ OBX-6 units | ✅ PQ value units | N/A |
| Clinical value plausibility | ✅ Observation values, BP consistency | ✅ OBX values, age-specific | ✅ PQ values, LOINC context | N/A |
| Profile/IG conformance | ✅ US Core 6.1.0 (requires Java) | None | None | N/A |

### Priority 1: Replace Heuristic Parsing With Real Parsers ✅ COMPLETED

Upgraded validators to use proper parsing libraries instead of regex/string matching. Malformed output that previously passed validation is now caught.

**Implementation (Feb 2026):**

| Format | Library | Changes |
|--------|---------|---------|
| C-CDA | `fast-xml-parser` | `XMLValidator.validate()` for syntax checking, `XMLParser` for DOM traversal |
| HL7v2 | `hl7v2` | `HL7Message.parse()` for structured parsing, proper field/component access |
| FHIR | `JSON.parse()` (unchanged) | Already adequate; schema validation deferred to Priority 4 |

**Key improvements:**
- C-CDA: Malformed XML (unclosed tags, invalid syntax) now fails with specific line/column error location
- HL7v2: Escaped delimiters handled correctly, proper 1-based field indexing per HL7v2 spec
- Both: Parse once at top, pass structured data to sub-validators (more efficient)

### Priority 2: Add HL7v2 Data Type Validation ✅ COMPLETED

HL7v2 fields have structured data types that interface engines (Rhapsody, Mirth Connect) validate before messages reach the EHR. The `hl7v2` library's component access APIs enable proper validation.

**Implementation (Feb 2026):**

| Data Type | Location | Components Validated |
|-----------|----------|---------------------|
| CX (Composite ID) | PID-3 | ID (required), Assigning Authority (required), ID Type Code (recommended) |
| XPN (Person Name) | PID-5 | Family Name, Given Name (at least one required), Name Type Code |
| XAD (Address) | PID-11 | Street, City, State (2-letter), Zip (US format), Address Type |
| XTN (Telephone) | PID-13 | Phone (deprecated vs structured format), Use Code, Equipment Type |
| CWE (Coded) | OBX-3, OBR-4 | Identifier + Coding System (required together), known coding system validation |

**Validation features:**
- Component-level validation using `field.getValue(componentPos)`
- Known value validation (ID types, name types, address types, phone use codes, coding systems)
- Severity-appropriate findings (major for required, minor for recommended, info for best practices)
- Actionable suggestions for each issue

**Example output:**
```
data_types: PASSED (score: 67)
Findings:
  [major] PID-3: CX.4 (Assigning Authority) is required for patient matching
  [major] OBX[1]-3: CWE.3 (Coding System) required when CWE.1 (Identifier) is present
  [minor] PID-5: XPN.2 (Given Name) is missing
  [info] PID-11: XAD.7 (Address Type) not specified
```

### Priority 3: Enforce Value Set Bindings ✅ COMPLETED

Enterprise systems validate coded fields against specific value sets.

**Implementation (Feb 2026):**

Created `lib/eval/validators/valueset-validator.ts` with embedded value sets and validation functions, integrated into all three format validators.

**Value Sets Implemented:**

| Category | Value Set | Source |
|----------|-----------|--------|
| Gender | Administrative Gender | FHIR ValueSet, HL7 Table 0001 |
| Marital Status | Marital Status | HL7 Table 0002 |
| Race/Ethnicity | OMB Race/Ethnicity | CDC/OMB categories |
| Condition Status | Clinical/Verification Status | FHIR ValueSet |
| Medication Status | MedicationRequest Status | FHIR ValueSet |
| Medication Route | Route codes | SNOMED CT, HL7 Table 0162 |
| Allergy | Clinical status, criticality, severity | FHIR ValueSet |
| Encounter | Status, class | FHIR ValueSet, ActEncounterCode |
| Observation | Status, interpretation | FHIR ValueSet, HL7 Table 0078 |
| Result Status | OBX-11 status codes | HL7 Table 0085 |
| Patient Class | PV1-2 codes | HL7 Table 0004 |
| Discharge Disposition | PV1-36 codes | HL7 Table 0112 |

**Integration by format:**

| Format | Fields Validated |
|--------|------------------|
| FHIR | Patient.gender, maritalStatus; Condition clinicalStatus/verificationStatus; MedicationRequest.status; AllergyIntolerance status/criticality/severity; Encounter status/class; Observation status/interpretation |
| HL7v2 | PID-8 (sex), PV1-2 (patient class), OBX-8 (interpretation), OBX-11 (result status), RXR-1 (route) |
| C-CDA | administrativeGenderCode, maritalStatusCode, statusCode (entries), encounter codes, severity observations |

**Example output:**
```
valuesets: PASSED (score: 85)
Findings:
  [major] Patient.gender: Invalid code 'X'. Valid values: male, female, other, unknown
  [major] OBX[1]-8: Invalid code 'AB'. Valid values: N, A, AA, H, HH, L, LL...
  [minor] Condition[0].clinicalStatus: Invalid code 'current'. Valid values: active, recurrence, relapse, inactive, remission, resolved
```

### Priority 4: Add US Core Profile Validation (FHIR) ✅ COMPLETED

Most US EHRs require US Core Implementation Guide compliance.

**Implementation (Feb 2026):**

Wrapped the official HL7 FHIR Validator JAR (v6.2.6) as a CLI tool for comprehensive US Core validation.

**Files added:**
- `lib/eval/validators/fhir-validator-cli.ts` - JAR download, caching, execution, output parsing

**Features:**
- Automatic JAR download and caching to `~/.fhir-validator-cache/`
- Validates against US Core IG 6.1.0
- Graceful fallback when Java is unavailable (informational finding, validation continues)
- `skipUSCore` option to bypass US Core validation when not needed
- Parses validator output into structured findings with severity levels

**Checks performed (when Java 11+ available):**
- Must Support elements
- Required value set bindings (e.g., Patient.gender)
- Extensions for race, ethnicity, birth sex
- Resource reference resolution patterns
- Full FHIR R4 structural conformance

**Usage:**
```typescript
// With US Core validation (default)
const results = await validateFHIRBundle(content)

// Skip US Core validation
const results = await validateFHIRBundle(content, { skipUSCore: true })

// Check validator status
import { getValidatorStatus } from './lib/eval/validators/fhir-validator'
const status = getValidatorStatus()
// { javaAvailable: true, validatorCached: true, validatorVersion: '6.2.6', cachePath: '...' }
```

**Note:** The `validateFHIRBundle` function is now async to support the validator subprocess.

### Priority 5: Validate Identifiers ✅ COMPLETED

EHRs and HIEs route and match patients based on identifiers. Invalid identifiers cause duplicate records or failed matches.

**Implementation (Feb 2026):**

Created `lib/eval/validators/identifier-validator.ts` with comprehensive identifier validation utilities, integrated into all three format validators.

**Validation functions:**

| Function | Purpose | Key Checks |
|----------|---------|------------|
| `validateOID()` | OID syntax validation | First arc 0-2, second arc rules (0-39 when first is 0-1), no leading zeros, proper dot separation |
| `validateNPI()` | NPI format validation | 10 digits, first digit 1-2, Luhn check digit with 80840 prefix |
| `validateUUID()` | UUID format validation | 36 chars, proper hyphenation, version digit, variant bits |
| `validateURI()` | URI format validation | Scheme required, validates OID portion for `urn:oid:` URIs |
| `validateFHIRIdentifiers()` | FHIR Patient identifiers | System URI format, value presence |
| `validateHL7v2AssigningAuthority()` | HL7v2 CX data type | Assigning authority OID, ID type codes, NPI validation when type=NPI |
| `validateCCDARoot()` | C-CDA root attributes | OID format, known root identification |

**Helper utilities:**
- `generateNPICheckDigit()` - Generate valid NPI from 9-digit prefix (for synthetic data)
- `identifyOIDRoot()` - Identify known OID namespaces (HL7, ISO US, Enterprise, Synthetic)
- `KNOWN_OID_ROOTS` - Registry of common healthcare OID roots

**Integration by format:**

| Format | Fields Validated |
|--------|------------------|
| FHIR | Patient.identifier (system URI, value), meta.profile URIs |
| HL7v2 | PID-3 (Patient ID), PV1-19 (Visit Number), ORC-12 (Ordering Provider), OBR-16 (Provider) |
| C-CDA | Document templateId roots, section templateId roots, all `root=""` attributes |

**Example output:**
```
identifiers: PASSED (score: 85)
Findings:
  [major] PID-3: Assigning authority OID invalid - OID first arc must be 0, 1, or 2
  [minor] Patient.identifier[0].system: URI must contain a scheme (e.g., http:, urn:, oid:)
  [info] PID-3: Assigning authority uses non-standard OID root
```

### Priority 6: Date/Time Timezone Enforcement ✅ COMPLETED

Missing or incorrect timezones cause real ingestion failures in enterprise systems.

**Implementation (Feb 2026):**

Created `lib/eval/validators/datetime-validator.ts` with comprehensive datetime validation, integrated into all three format validators.

**Validation functions:**

| Function | Purpose | Key Checks |
|----------|---------|------------|
| `validateFHIRDateTime()` | FHIR dateTime validation | Precision levels (year/month/day/time/instant), timezone requirement |
| `validateFHIRInstant()` | FHIR instant validation | Requires full precision with timezone (Z or +/-HH:MM) |
| `validateHL7v2DateTime()` | HL7v2 DTM validation | YYYYMMDDHHMMSS format, +/-ZZZZ timezone |
| `validateCCDADateTime()` | C-CDA effectiveTime | HL7v2 format with minimum precision enforcement |
| `validateFHIRResourceDateTimes()` | Bulk FHIR validation | Validates all datetime fields in a resource |
| `validateHL7v2SegmentDateTime()` | Segment field validation | Context-aware timezone requirements |

**Helper utilities:**
- `isFutureDate()` - Detect future dates (potential data quality issue)
- `isUnreasonablyOld()` - Detect dates before 1900

**Integration by format:**

| Format | Fields Validated | Timezone Required |
|--------|------------------|-------------------|
| FHIR | instant fields (issued, sent, received, recorded, lastUpdated), dateTime fields (birthDate, effectiveDateTime, etc.), period.start/end | instant: yes, dateTime with time: recommended |
| HL7v2 | MSH-7, PID-7 (DOB), PID-29 (death), PV1-44/45 (admit/discharge), OBR-7/22, OBX-14 | Clinical events: yes, DOB: no |
| C-CDA | Document effectiveTime, birthTime, entry effectiveTime, low/high ranges | Document & clinical events: yes, birthTime: no |

**Example output:**
```
datetimes: PASSED (score: 85)
Findings:
  [major] MSH-7: Timezone offset required (use +/-ZZZZ format, e.g., -0500)
  [minor] Patient.effectiveDateTime: Time without timezone may cause interpretation issues
  [minor] OBX[1]-14: Timezone recommended for clinical timestamps
```

### Priority 7: Validate C-CDA Entry-Level Content ✅ COMPLETED

The C-CDA generator was rewritten for full ONC SITE validator conformance (Feb 2026).

**Implementation:**

Rewrote `lib/generators/ccda-generator.tsx` to produce CCD documents that pass the ONC SITE validator with 0 errors. Changes include dual R1.1/R2.1 templateIds on all entries, proper Act/Observation structure within each section, statusCode elements, effectiveTime with timezone precision, and three new required sections (Allergies, Social History, Vital Signs). Empty sections use `nullFlavor="NI"` to avoid XML Schema violations.

**Validation results:**
- ONC SITE validator: 0 errors (down from 43), 35 warnings (SHOULD-level), ~100 info (MAY-level)
- Eval suite C-CDA format scores: 96-99 across all test cases

### Priority 8: UCUM Unit Validation for Lab Results ✅ COMPLETED

Lab results with non-standard units get rejected or misinterpreted by EHRs.

**Implementation (Feb 2026):**

Created `lib/eval/validators/ucum-validator.ts` using the `@lhncbc/ucum-lhc` library from the National Library of Medicine for UCUM (Unified Code for Units of Measure) validation.

**Validation functions:**

| Function | Purpose | Key Checks |
|----------|---------|------------|
| `validateUcumUnit()` | Async unit validation | Uses UCUM library, returns validity and suggestions |
| `validateUcumUnitSync()` | Sync validation with fallback | Pattern-based validation when library not loaded |
| `validateFHIRObservationUnits()` | FHIR Observation resources | valueQuantity.unit/code, referenceRange units, component units |
| `validateHL7v2ObservationUnits()` | HL7v2 OBX-6 | CE/CWE data type unit code (first component) |
| `validateCCDAObservationUnits()` | C-CDA observation values | PQ value/@unit attributes |
| `preloadUcumLibrary()` | Library initialization | Pre-load for better performance |

**Key features:**
- Lazy-loads the UCUM library on first use
- Provides correction suggestions for common errors (e.g., `mg/dl` → `mg/dL`, `mmHg` → `mm[Hg]`)
- Pattern-based fallback validation for common units when library unavailable
- Case-sensitive validation (UCUM is case-sensitive)
- `normalizeUcumUnit()` exported for upstream use in `enrichLab()` — fixes units before they reach generators, so all three formats benefit without generator changes

**Common unit corrections:**
- Case errors: `mg/dl` → `mg/dL`, `mmol/l` → `mmol/L`, `meq/l` → `mEq/L`
- Human-readable → UCUM: `bpm` → `/min`, `°C` → `Cel`, `mmHg` → `mm[Hg]`
- Unit names → codes: `inches` → `[in_i]`, `pounds` → `[lb_av]`, `hours` → `h`

**Integration by format:**

| Format | Fields Validated |
|--------|------------------|
| FHIR | Observation.valueQuantity.unit/code, referenceRange[].low/high.unit, component[].valueQuantity.unit |
| HL7v2 | OBX-6 (Units) - first component of CE/CWE data type |
| C-CDA | `<value xsi:type="PQ" unit="..."/>`, `<low unit="..."/>`, `<high unit="..."/>` |

**Example output:**
```
units: PASSED (score: 85)
Findings:
  [major] Observation[0].valueQuantity.unit: Invalid UCUM unit: 'mg/dl'
          Suggestion: Use 'mg/dL' instead of 'mg/dl' (UCUM is case-sensitive)
  [major] OBX[1]-6: Invalid UCUM unit: 'mmHg'
          Suggestion: Use 'mm[Hg]' instead of 'mmHg'
  [minor] referenceRange: Unit not in common list, may need verification
```

### Priority 9: Clinical Value Plausibility Checks ✅ COMPLETED

Beyond range validity (min < max), validates that clinical values are within physiologically possible ranges.

**Implementation (Feb 2026):**

Created `lib/eval/validators/plausibility-validator.ts` with comprehensive clinical value plausibility checks based on medical reference ranges.

**Validation functions:**

| Function | Purpose | Key Checks |
|----------|---------|------------|
| `checkValuePlausibility()` | Single value check | Physiological limits, critical values, age-specific ranges |
| `checkBloodPressureConsistency()` | BP component check | Systolic > diastolic, pulse pressure validation |
| `validateFHIRObservationPlausibility()` | FHIR Observation | valueQuantity, components (BP), referenceRange consistency |
| `validateHL7v2ObservationPlausibility()` | HL7v2 OBX segments | OBX-5 value against OBX-3 identifier |
| `validateCCDAObservationPlausibility()` | C-CDA observations | PQ value elements with LOINC code context |

**Clinical values covered (40+ parameters):**

| Category | Parameters |
|----------|------------|
| Vital Signs | Heart rate, blood pressure (systolic/diastolic), respiratory rate, temperature, SpO2, weight, height, BMI |
| Glucose/Diabetes | Blood glucose, HbA1c |
| Renal Function | Creatinine, BUN, eGFR |
| Electrolytes | Sodium, potassium, chloride, bicarbonate, calcium, magnesium, phosphorus |
| Hematology | Hemoglobin, hematocrit, WBC, platelets |
| Liver Function | ALT, AST, alkaline phosphatase, bilirubin, albumin |
| Lipids | Total cholesterol, LDL, HDL, triglycerides |
| Thyroid | TSH, free T4 |
| Cardiac | Troponin, BNP, NT-proBNP |
| Coagulation | PT, INR, aPTT |

**Key features:**
- LOINC code matching for precise parameter identification
- Alternate name matching (e.g., "glucose", "blood sugar", "fasting glucose")
- Age-specific ranges (pediatric vs adult heart rate, weight, etc.)
- Critical value thresholds (separate from physiological limits)
- Blood pressure consistency (systolic must exceed diastolic, pulse pressure checks)
- Reference range validation (low must be less than high)

**Severity levels:**
- `critical`: Value outside physiologically possible range (data error)
- `major`: Value in critical range (clinically concerning but possible)
- `minor`: Value unusual but within possible range

**Example output:**
```
plausibility: PASSED (score: 77)
Findings:
  [critical] Observation[0].valueQuantity: Heart Rate value 350 is outside physiologically possible range (20-300 /min)
  [major] Observation[1].valueQuantity: Blood Glucose value 650 is critically high (> 600 mg/dL)
  [major] Observation[2].component: Systolic (110) must be greater than diastolic (120) blood pressure
```

### Priority 10: Cross-Format Consistency Validation

When generating the same patient across FHIR, HL7v2, and C-CDA, validate that:

- Patient demographics match across formats
- Clinical codes map correctly between formats
- Identifiers are consistent
- Timestamps align

### Libraries In Use

| Purpose | Library | Status |
|---------|---------|--------|
| XML parsing (C-CDA) | `fast-xml-parser` | ✅ Implemented (Priority 1) |
| HL7v2 parsing | `hl7v2` | ✅ Implemented (Priority 1) |
| FHIR validation | HL7 FHIR Validator JAR v6.2.6 (CLI wrapper) | ✅ Implemented (Priority 4) |
| Identifier validation | Custom (OID, NPI, UUID, URI) | ✅ Implemented (Priority 5) |
| DateTime validation | Custom (FHIR/HL7v2/C-CDA formats, timezone) | ✅ Implemented (Priority 6) |
| Value set validation | Custom (embedded HL7/FHIR value sets) | ✅ Implemented (Priority 3) |
| UCUM units | `@lhncbc/ucum-lhc` | ✅ Implemented (Priority 8) |
| Clinical plausibility | Custom (40+ clinical parameters) | ✅ Implemented (Priority 9) |

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

### Step 1 (Detailed): Generate Trace Dataset ✅ IMPLEMENTED

Run the eval suite across all 12 test cases to generate personas. Supplement with 15-20 custom prompts covering edge cases for a total of ~100 traces. Each trace captures one persona — the unit of manual annotation — not one per output format.

Each trace includes:

- The input prompt (e.g., "45-year-old male with Type 2 diabetes and CKD")
- The generated persona (demographics, conditions, medications, labs, encounters)
- Generated outputs for each format (FHIR, C-CDA, HL7v2) with full content preserved
- Automated format/terminology/clinical scores and findings
- Timing information (persona generation, total)

Save all traces to `eval-results/traces/` as JSON for reproducibility.

```bash
# Generate traces for full test suite
pnpm eval:traces

# Generate trace for single prompt
pnpm eval --prompt "45-year-old male with Type 2 diabetes" --save-traces

# Bulk generation from CSV file
pnpm eval --prompts-file eval-data/prompts/my-prompts.csv --save-traces
```

**Trace File Format (eval traces):**
```json
{
  "traceId": "eval-2026-02-04-abc123",
  "timestamp": "2026-02-04T14:30:00Z",
  "prompt": "Create a 58-year-old Hispanic male with type 2 diabetes...",
  "testCaseId": "diabetes-001",
  "persona": { /* full Persona object */ },
  "outputs": {
    "fhir": { "content": "...", "scores": {...}, "findings": [...] },
    "ccda": { "content": "...", "scores": {...}, "findings": [...] },
    "hl7v2": { "content": "...", "scores": {...}, "findings": [...] }
  },
  "aggregateScores": { "format": 100, "terminology": 100, "clinical": 64, "overall": 82 },
  "timing": { "personaGeneration": 6482, "total": 18647 }
}
```

**Production Trace Format (Vercel Blob):**

Production traces use the `ProductionTrace` type (`lib/traces/production-trace.ts`) which extends the eval trace with:
- `source: 'production' | 'eval'` — distinguishes origin
- `annotations: Annotation[]` — array supporting multiple reviewers (each with `reviewerId`, `qualityRating`, `commonIssues`, `notes`, `annotatedAt`)
- `feedback?: { rating: 'up' | 'down', timestamp }` — user thumbs up/down
- `outputs` may lack `scores` and `findings` (no automated evaluation in production)

Production traces are saved to Vercel Blob at `traces/trace-{traceId}.json`. They can be viewed in the trace viewer with `TRACE_SOURCE=blob` or fetched via the `/api/traces` endpoints.

#### Portfolio Deliverables from Step 1

- Trace dataset in `eval-results/traces/`
- Devlog entry documenting trace generation approach and prompt coverage

### Step 2 (Detailed): Build Custom Trace Viewer ✅ IMPLEMENTED

Build the trace viewer **before** starting manual review. Per Husain/Shankar, custom annotation tools are "the single most impactful investment" for eval workflows, and teams with custom tools iterate ~10x faster. Reviewing 100+ traces in raw JSON is unsustainable — the viewer makes the difference between abandoning the process and completing it.

#### Implementation: Shiny for Python

Built as a standalone Shiny for Python app in `tools/trace-viewer/` rather than a Next.js page. This follows Hamel Husain's recommendation of lightweight Python frameworks (Shiny, Gradio, Streamlit) for rapid iteration on annotation tools.

```bash
# Setup
cd tools/trace-viewer
pip install -r requirements.txt

# Run
shiny run app.py
# Opens at http://localhost:8000
```

#### Features (Three-Panel Layout)

**Left Sidebar (Navigation):**
- Trace selector dropdown with score preview and annotation indicator (`+`)
- Min score filter slider
- Severity filter checkboxes
- Quick info panel showing scores and annotation status

**Center Panel (Content):**
- Prompt always visible at top
- Tabbed navigation: Overview, Persona, Outputs, Findings
- Overview: aggregate scores, metadata, timing
- Persona: demographics, conditions, medications, labs, encounters
- Outputs: sub-tabs for FHIR/C-CDA/HL7v2 with per-format scores and syntax-highlighted content
- Findings: filterable list sorted by severity

**Right Sidebar (Annotations - Always Visible):**
- Quality rating (1-5 scale)
- Common issues checkboxes (missing meds, irrelevant labs, terminology error, etc.)
- Free-text notes field
- Save button — writes annotations directly to trace JSON file

The split-panel layout keeps annotations visible while browsing content, eliminating context switching.

#### Annotation Storage

Annotations are saved directly into each trace JSON file (not a separate annotations.json):

```json
{
  "traceId": "eval-2026-02-04-abc123",
  "annotations": {
    "quality_rating": "good",
    "common_issues": ["missing_meds"],
    "notes": "LLM didn't prescribe antihistamines for allergies",
    "annotated_at": "2026-02-04T15:30:00"
  },
  ...
}
```

This simplifies the workflow — each trace is self-contained with its annotation.

#### Portfolio Deliverables from Step 2

- The trace viewer itself (`tools/trace-viewer/`)
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
