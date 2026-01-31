# Tabula Health Generator

> Every health record begins with a human story.

**AI-powered synthetic healthcare data generator** that transforms natural language patient descriptions into standards-compliant FHIR, C-CDA, and HL7v2 messages.

*Tabula* — Latin for "tablet" or "slate" — evokes the blank canvas on which compelling patient stories are crafted. Like a tabula rasa, each generation starts fresh, shaped entirely by your narrative.

[![Live Demo](https://img.shields.io/badge/Live%20Demo-Visit%20App-blue?style=for-the-badge)](https://v0-tabula-health.vercel.app)
[![Built with Next.js](https://img.shields.io/badge/Built%20with-Next.js%2014-black?style=for-the-badge&logo=next.js)](https://nextjs.org)

---

## Why Not Just Use Synthea?

[Synthea](https://github.com/synthetichealth/synthea) is the gold standard for bulk synthetic patient generation — it simulates entire populations from birth to death using rule-based modules. It's excellent for what it does.

But there's a gap:

| Need | Synthea | Tabula Health Generator |
|------|---------|---------------------|
| Generate 10,000 patients for load testing | ✅ Perfect | ❌ Overkill |
| "I need a 45-year-old diabetic with recent chest pain" | ❌ Configure modules, filter output, hope for match | ✅ Describe it, get tailored records on demand |
| Test a specific edge case scenario | ❌ May require custom module development | ✅ Describe the scenario in plain English |
| Explore "what if" clinical variations | ❌ Requires code changes | ✅ Adjust your prompt |
| Time to first useful record | Minutes to hours (setup, run, filter) | Seconds |

**Synthea is a population simulator. Tabula Health Generator is a scenario fulfillment tool for medical records.**

They're complementary: use Synthea when you need volume and statistical realism across a population, use Tabula Health when you need a specific patient story on demand.

---

## The Problem

Healthcare software developers need realistic test data, but:

- **Real patient data is off-limits** — HIPAA prevents using actual records
- **Manual test data is tedious** — Creating clinically coherent scenarios by hand takes hours
- **Existing synthetic data lacks quality** — Generic tools produce unrealistic combinations (a 25-year-old with dementia medications)
- **Terminology codes are complex** — ICD-10, SNOMED CT, LOINC, RxNorm require domain expertise most developers don't have

## The Solution

Describe a patient in plain English. Get back clinically coherent, properly coded healthcare messages.

**Input:**
> "A 45-year-old father managing type 2 diabetes while caring for aging parents, recently hospitalized for chest pain"

**Output:**
- FHIR R4 Bundle with Patient, Condition, Observation, MedicationRequest, Encounter
- C-CDA 2.1 Continuity of Care Document
- HL7v2 ADT messages

All with real terminology codes (ICD-10, SNOMED CT, LOINC, RxNorm) and synthetic data watermarks for safe testing.

<!-- TODO: Add screenshot or GIF here -->
![Tabula Health Generator Demo](images/demo-placeholder.png)

---

## Why I Built This

Synthea is great for bulk data, but I saw a gap: **no tool let you describe a specific patient scenario and get back standards-compliant test data in seconds.**

QA engineers testing edge cases, developers debugging specific workflows, educators demonstrating clinical concepts — they all needed something more targeted than population simulation.

This project demonstrates:

1. **Healthcare domain expertise** — Understanding HL7 standards, clinical terminology systems, and the regulatory context around health data
2. **AI product thinking** — Using LLMs not just as chat interfaces but as structured data generators with validation and quality controls
3. **Shipping ability** — Taking a concept from PRD to deployed product with modern AI tooling (built with Vercel v0 + Claude)

This is a portfolio piece showing how I approach AI product development in a complex, regulated domain.

---

## Technical Highlights

### LLM as Structured Data Extractor (Not a Code Database)

Key architectural decision: **the LLM doesn't generate terminology codes. It extracts clinical intent from the narrative.**

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   Your Story     │───▶│   LLM extracts    │───▶│   Terminology    │───▶│   Format         │
│                  │     │   clinical facts │     │   service adds   │     │   generators     │
│ "45-year-old     │     │                  │     │   real codes     │     │   output         │
│  with diabetes   │     │ conditions:      │     │                  │     │                  │
│  on metformin"   │     │  "Type 2         │     │ E11.9 (ICD-10)   │     │ FHIR, C-CDA,     │
│                  │     │   diabetes"      │     │ 1043567 (RxNorm) │     │ HL7v2            │
└──────────────────┘     └──────────────────┘     └──────────────────┘     └──────────────────┘
```

**Why not have the LLM generate codes directly?**

LLMs hallucinate codes. They'll invent plausible-looking but invalid ICD-10 or RxNorm codes. The solution: separation of concerns.

| Component | Responsibility |
|-----------|----------------|
| **LLM (via function calling)** | Understands clinical context, normalizes terminology ("sugar problems" → "Type 2 diabetes mellitus"), structures narrative into conditions/meds/labs |
| **Terminology Service** | Deterministic lookups against real databases (UMLS, RxNav) guarantee valid codes |

**OpenAI Function Calling** constrains the LLM to return structured JSON via a schema (`emit_persona`), not prose. The LLM can't improvise—it must fill the defined fields with clinical facts extracted from your narrative.

This architecture lets each component do what it's best at: LLMs excel at understanding intent and normalizing language; lookup services guarantee correctness.

### Multi-Format Output Generation
Single AI-generated persona produces consistent data across three healthcare standards:
- **FHIR R4** — Modern REST-based standard that the digital health ecosystem is in the process of adopting. 
- **C-CDA 2.1** — Document-based exchange standard
- **HL7v2** — Legacy messaging standard still used in 90%+ of US hospitals

### Real Terminology Integration
Not placeholder codes — actual terminology lookups via:
- **UMLS API** — SNOMED CT and ICD-10 codes
- **RxNav API** — RxNorm medication codes
- **LOINC** — Lab observation codes
- **Local fallback mappings** — Graceful degradation when APIs are unavailable

### Synthetic Data Watermarking
Multi-layer approach so synthetic data can never be confused with real PHI:
- FHIR: `meta.tag` with synthetic markers
- C-CDA: Document title suffix, custom templateId
- HL7v2: MSH-11 processing mode, synthetic assigning authority

### AI-Powered Clinical Coherence
GPT-4 generates personas with internally consistent:
- Demographics and social context
- Diagnoses appropriate for age/history
- Medications matching conditions
- Lab values in clinically plausible ranges

---

## Key Product Decisions

| Decision | Rationale |
|----------|-----------|
| **Story-first input** | Developers think "45-year-old with diabetes" not "ICD-10 E11.9". Meet users where they are. |
| **Multiple output formats** | FHIR is modern, but 90% of US hospitals still use HL7v2. Support the real world. |
| **Real terminology codes** | Placeholder codes don't test real integrations. Valid codes are essential. |
| **Mandatory watermarking** | Synthetic data mistaken for PHI is a compliance nightmare. Build safety in. |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Next.js Frontend                        │
│  Landing Page → Persona Builder → Output Preview & Export   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     Server Actions                           │
│  Persona Generation → Terminology Enrichment → Generators   │
│      (OpenAI)              (UMLS API)         (FHIR/CDA/HL7)│
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      External APIs                           │
│     OpenAI GPT-4  ←→  UMLS/NLM  ←→  RxNav (RxNorm)         │
└─────────────────────────────────────────────────────────────┘
```

---

## Demo Video

<!-- TODO: Embed 2-minute walkthrough video -->
*Coming soon*

---

## What I Learned

**On AI product development:**
- Structured output from LLMs requires careful prompt engineering and validation layers
- "AI-generated" doesn't mean "no rules" — constraints improve quality
- Graceful degradation (local fallbacks when APIs fail) is essential for reliability

**On healthcare interoperability:**
- Standards exist but implementations vary wildly
- Terminology mapping is harder than it looks (SNOMED ↔ ICD-10 isn't 1:1)
- Watermarking synthetic data is a solved problem with clear best practices

---

## Links

- **Live Demo:** [v0-tabula-health.vercel.app](https://v0-tabula-health.vercel.app)
- **Product Plan:** [Detailed decisions and roadmap](docs/PRODUCT_PLAN.md)
- **Original PRD:** [MVP requirements document](docs/PRD.md)

---

## Tech Stack

- **Frontend:** Next.js 14, React, Tailwind CSS, shadcn/ui
- **AI:** OpenAI GPT-4 for persona generation
- **Terminology:** UMLS API (SNOMED CT, ICD-10), RxNav (RxNorm), LOINC
- **Deployment:** Vercel

---

*Built by [Scott Tse](https://github.com/sky-t) as an AI product management portfolio piece.*
