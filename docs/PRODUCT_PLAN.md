# Tabula Health Generator - Product Plan

## Vision

**Every health record begins with a human story.**

Real life isn't structured data—it's a 45-year-old father managing diabetes while caring for aging parents. It's a college student's first ER visit far from home. We start with the story, then build the interoperability scaffolding around it.

Tabula Health Generator transforms narrative patient descriptions into clinically coherent, standards-compliant healthcare test data.

---

## Problem Statement

Healthcare software developers need realistic test data to build and validate their applications, but:

1. **Real patient data is off-limits** - HIPAA and privacy regulations prevent using actual records
2. **Manual test data is tedious** - Creating clinically coherent scenarios by hand is time-consuming
3. **Existing synthetic data lacks quality** - Generic tools produce unrealistic combinations
4. **Terminology codes are complex** - ICD-10, SNOMED CT, LOINC, RxNorm require domain expertise

## Solution

An AI-powered generator that takes natural language patient stories and outputs them in standard healthcare formats (FHIR, C-CDA, HL7v2) with properly coded terminology.

---

## Target Users

| User Type | Need | How We Help |
|-----------|------|-------------|
| Healthcare developers | Test EHR integrations | Generate valid FHIR/C-CDA/HL7v2 messages |
| QA engineers | Test edge cases | Create specific clinical scenarios on demand |
| Students/learners | Understand health data standards | See real examples with proper codes |
| Demo/sales teams | Show product capabilities | Generate realistic patient data quickly |

---

## Current Status

### Completed Features
- [x] AI-generated patient personas from natural language prompts
- [x] Pre-built persona templates for common scenarios
- [x] FHIR R4 Bundle generation (Patient, Condition, Observation, MedicationRequest, Encounter)
- [x] C-CDA 2.1 document generation with structured entries
- [x] HL7v2 ADT message generation
- [x] Automatic terminology enrichment (ICD-10, SNOMED CT, LOINC, RxNorm)
- [x] SNOMED CT crosswalk with local mapping fallbacks
- [x] Synthetic data watermarking
- [x] Deployed to Vercel with access code protection
- [x] Landing page with story-first narrative

### Remaining Tasks

| Task | Description | Status |
|------|-------------|--------|
| Demo video | Record 2-minute walkthrough | Pending |
| Portfolio repo | Public case study for GitHub Pages | Blocked by demo video |
| UX polish | Improve error states and loading indicators | Pending |
| AI eval design | Document evaluation framework for quality | Pending |
| AI eval implementation | Build test suite for quality measurement | Blocked by design |

---

## Technical Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Next.js Frontend                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ Landing     │  │ Persona     │  │ Output Preview      │  │
│  │ Hero        │  │ Builder     │  │ & Export            │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Server Actions                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ Persona     │  │ Terminology │  │ Message             │  │
│  │ Generation  │  │ Enrichment  │  │ Generators          │  │
│  │ (OpenAI)    │  │ (UMLS API)  │  │ (FHIR/CDA/HL7)      │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    External APIs                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ OpenAI      │  │ UMLS/NLM    │  │ RxNav               │  │
│  │ GPT-4       │  │ (SNOMED,    │  │ (RxNorm)            │  │
│  │             │  │  ICD-10)    │  │                     │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Product Decisions

### Why "story-first"?
**Decision**: Lead with narrative patient descriptions, not form fields
**Rationale**: Real clinical scenarios are stories first—a developer testing diabetes workflows thinks "45-year-old with uncontrolled diabetes" not "ICD-10 E11.9". Meeting users where they are.

### Why AI-generated personas?
**Decision**: Use LLM to generate clinically coherent patient scenarios
**Rationale**: Manual templates are limited; AI can create infinite variations while maintaining clinical logic (e.g., appropriate medications for diagnoses).

### Why multiple output formats?
**Decision**: Support FHIR, C-CDA, and HL7v2
**Rationale**: Different systems use different standards; FHIR is modern but legacy systems still use HL7v2 and C-CDA.

### Why real terminology codes?
**Decision**: Integrate with UMLS/NLM APIs for actual ICD-10, SNOMED CT codes
**Rationale**: Placeholder codes don't test real-world scenarios; valid codes are essential for meaningful testing.

### Why watermark synthetic data?
**Decision**: Embed clear synthetic markers in all outputs
**Rationale**: Prevent synthetic data from being mistaken for real patient records; critical for compliance.

---

## AI Quality Considerations

### Challenges
1. **Clinical coherence** - AI might generate illogical combinations
2. **Terminology accuracy** - Display names must map to correct codes
3. **Format validity** - Generated messages must pass standard validators

### Planned Evaluation Approach
- **Automated**: Run outputs through FHIR/C-CDA validators
- **LLM-as-judge**: Use AI to check clinical coherence
- **Human review**: Sample review for edge cases
- **Test suite**: Curated prompts with expected characteristics

---

## Links

- **Live Demo**: https://v0-tabula-health.vercel.app (access code required)
- **Vercel Project**: https://vercel.com/sky2tse-6641s-projects/v0-tabula-health

---

## Changelog

### 2026-01-30
- Deployed to Vercel with access code protection
- Added landing page with story-first narrative
- Completed SNOMED CT integration with crosswalk and local mappings
- Added Encounter resources to FHIR output
- Expanded C-CDA with structured medication, lab, and encounter entries
