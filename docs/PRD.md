# Tabula Health Generator - Product Requirements Document (MVP)

## 1. Product Vision

Create a simple, magical web application that allows users to generate clinically plausible, standards-conformant synthetic patient data per persona at the push of a button.

This MVP focuses on an interactive UI — no APIs or CI/CD integrations yet — to demonstrate immediate value and delight.

## 2. Goals (MVP)

- **Persona realism** – provide curated clinical personas (e.g., Diabetes, Pregnancy w/ social risks, Acute head injury)
- **Standards conformance** – generate HL7v2, FHIR R4, and C-CDA messages that pass validation
- **Watermarking & safety** – embed visible and machine-detectable watermarks in all outputs so synthetic data cannot be confused with PHI
- **Delightful UX** – one-click generation, clear preview, download, and validator results

## 3. Non-Goals (Deferred to Later Phases)

- API keys, programmatic access, and CI/CD integration
- Cohort generation at scale (hundreds/thousands)
- Deterministic seeds for reproducibility
- Bulk exports, webhooks, or enterprise features

## 4. Target Users

| User Type | Need |
|-----------|------|
| Healthcare app builders | Sample data for testing |
| QA analysts | Validating HL7/FHIR message parsing |
| Educators/trainers | Teaching standards with safe data |

## 5. User Stories

1. As a user, I can select a persona and one or multiple output formats (HL7v2, FHIR, C-CDA)
2. As a user, I can generate messages instantly and preview them in both human-readable and raw formats
3. As a user, I can validate each message and see clear pass/fail feedback
4. As a user, I can download each artifact individually or as a bundled ZIP

## 6. Functional Requirements (MVP)

### 6.1 Persona Catalog
- Provide at least three predefined personas
- Users must be able to select one persona at a time to drive message generation

### 6.2 Message Generation
- Users must be able to select one or more output formats simultaneously:
  - HL7v2 (ADT^A01, ORU^R01)
  - FHIR R4 (Patient, Condition, Observation, MedicationRequest Bundles)
  - C-CDA (Continuity of Care Document)
- Generated artifacts must be clinically consistent across all formats from the same persona
- Preview panel with tabs per format showing human-readable summary and raw message

### 6.3 Watermarking & Safety
- **HL7v2:** MSH-11=T, generator ID in MSH-3/4, synthetic assigning authority in PID-3, optional Z segment
- **C-CDA:** Document title suffix [SYNTHETIC TEST DATA], custom templateId, synthetic IDs
- **FHIR:** meta.tag + meta.security labels, synthetic profile, Provenance resource

### 6.4 Validation
- Each artifact validated against schema + basic business rules
- Validation results shown per format with pass/fail indicators

## 7. Non-Functional Requirements

- Generate a single message in <10 seconds
- ≥90% of generated artifacts pass schema validation on first attempt
- No real PHI stored or processed

## 8. Success Metrics

- 70% of first-time users generate a valid artifact within 5 minutes
- ≥90% validator pass rate across all formats on first try
- ≥80% of feedback respondents describe the experience as "easy" or "magical"

## 9. Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Data mistaken for PHI | Multi-layer watermarking |
| Clinically implausible data | Rules engine + curated terminology subsets |
| Artifacts fail validation | Pre-validation via open-source tooling |

## 10. Roadmap

- **Phase 1 (MVP):** Persona picker, multi-output generation, validation, watermarking, download
- **Phase 2:** Developer APIs, seeds, programmatic reproducibility
- **Phase 3:** Cohorts, bulk exports, CI/CD integration, enterprise features
