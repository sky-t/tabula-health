<p align="center">
  <img src="images/tabula_transparent.png" alt="Tabula Health" width="500" />
</p>

# Tabula Health Generator

> Every health record begins with a human story.

**AI-powered synthetic healthcare data generator** that transforms natural language patient descriptions into standards-compliant FHIR, C-CDA, and HL7v2 messages.

*Tabula*, Latin for "tablet" or "slate", evokes the blank canvas on which compelling patient stories are crafted. Like a tabula rasa, each generation starts fresh, shaped entirely by your narrative.

[![Live Demo](https://img.shields.io/badge/Live%20Demo-Visit%20App-blue?style=for-the-badge)](https://v0-tabula-health.vercel.app)
[![Built with Next.js](https://img.shields.io/badge/Built%20with-Next.js%2014-black?style=for-the-badge&logo=next.js)](https://nextjs.org)
[![Demo Video](https://img.shields.io/badge/Demo%20Video-Watch%20on%20Loom-purple?style=for-the-badge&logo=loom)](https://www.loom.com/share/259ab536660d4301a020bb15d12ca46a)

---

## Table of Contents

- [Why Not Just Use Synthea?](#why-not-just-use-synthea)
- [The Problem](#the-problem)
- [The Solution](#the-solution)
- [Why I Built This](#why-i-built-this)
- [Technical Highlights](#technical-highlights)
- [Key Product Decisions](#key-product-decisions)
- [Architecture](#architecture)
- [AI Evaluation & Quality](#ai-evaluation--quality)
- [Building in Public](#building-in-public)
- [Demo Video](#demo-video)
- [What I Learned](#what-i-learned)
- [Links](#links)
- [Tech Stack](#tech-stack)

---

## Why Not Just Use Synthea?

While [Synthea](https://github.com/synthetichealth/synthea) is the gold standard for bulk synthetic patient generation, it simulates entire populations from birth to death using rule-based modules. It's excellent for creating population scale synthetic data.

But there's a gap:

| Need | Synthea | Tabula Health Generator |
|------|---------|---------------------|
| Generate 10,000 patients for load testing | ✅ Perfect | ❌ Overkill |
| "I need a 45-year-old diabetic with recent chest pain" | ❌ Configure modules, filter output, hope for match | ✅ Describe it, get tailored records on demand |
| Test a specific edge case scenario | ❌ May require custom module development | ✅ Describe the scenario in plain English |
| Explore "what if" clinical variations | ❌ Requires code changes | ✅ Adjust your prompt |
| Time to first useful record | Minutes to hours (setup, run, filter) | Seconds |

**Synthea is a population simulator. Tabula Health Generator is a scenario fulfillment tool for medical records.**

They're complementary: use Synthea when you need volume and statistical realism across a population. Use Tabula Health when you need a specific patient story on demand, without handcrafting interop files, manually creating test patients in an EHR, or stitching together a cohesive set of conditions yourself.

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

All with real terminology codes (ICD-10, SNOMED CT, LOINC, RxNorm) and synthetic data watermarks for safe testing, aligned with the patient story.

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
- **FHIR R4** — Modern REST-based standard increasingly adopted across the digital health ecosystem 
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

## AI Evaluation & Quality

Shipping the generator was step one. The harder question: **how do you know the output is actually good?**

The generation pipeline has one non-deterministic step (LLM persona generation) and two deterministic steps (terminology enrichment, format generation). These require different evaluation strategies:

| Layer | Nature | Eval method |
|-------|--------|-------------|
| **Persona generation** (LLM) | Non-deterministic, subjective quality | Manual trace annotation → scoped binary judges |
| **Terminology enrichment** | Deterministic lookup | Automated tests with fixed inputs and expected outputs |
| **Format generators** (FHIR/C-CDA/HL7v2) | Deterministic transformation | Automated tests — known-good persona in, valid output asserted |

Format validity and terminology accuracy are deterministic and testable with traditional automated tests. Clinical coherence — does this patient record make medical sense as a whole? — is the hard problem, and the one that requires human judgment to define before it can be automated.

### Approach

Following the methodology in Husain & Shankar's [AI Evals FAQ](https://hamel.dev/blog/posts/evals/) (2025): **start with error analysis, not infrastructure.** Build evaluation criteria from observed failures, not hypothesized ones. The manual annotation effort is scoped to the LLM-generated persona — that's where non-determinism lives and where human judgment is actually needed.

A key principle from this methodology: classify every failure as either a **specification failure** (the prompt doesn't define correct behavior → fix the generator) or a **generalization failure** (the model has correct instructions but fails to apply them → build an automated evaluator). This distinction determines the right fix and prevents building evaluators for problems that a prompt change would eliminate — a common mistake in AI eval work.

Quality controls operate at two layers:
- **Deterministic validation** — A clinical plausibility validator catches physiologically impossible values (lab ranges, vital signs) with rule-based checks. No LLM judge needed for what is essentially a lookup problem.
- **LLM-as-judge** — Reserved for failures that require clinical reasoning to evaluate: incomplete lab panels, diagnosis specificity, medication appropriateness. These can't be reduced to rules.

### Evaluation Roadmap

| Step | Activity | Output |
|------|----------|--------|
| ✅ 1 | Generate persona traces across clinical scenarios | Trace dataset for annotation |
| ✅ 2 | Build trace annotation viewer | Structured review tool with pass/fail verdicts and clinical notes |
| ✅ 3 | Manual error analysis, round 1 — open coding (12 traces) | Free-text annotations on each persona |
| ✅ 4 | Failure taxonomy, round 1 — axial coding | [14 failure modes, each classified as spec vs. generalization →](docs/ERROR_ANALYSIS.md) |
| ✅ 5 | Scale annotation to 55 traces; rebuild taxonomy | [10 stable failure codes with frequency data →](docs/FAILURE_TAXONOMY.md) |
| ✅ 6 | Implement spec fixes based on taxonomy | Prompt engineering (7 failure modes); deterministic plausibility validator (range errors) |
| 7 | Re-annotate with updated prompt; measure improvement | Validate which failures were truly spec vs. generalization |
| 8 | Build binary evaluators for residual generalization failures | Scoped LLM judges for failures that persist after spec fixes |
| 9 | Validate judges against human labels | True positive / true negative rates per evaluator |

With the eval framework in place, I can also bring in early beta testers and capture their real-world prompts as test cases — moving beyond curated scenarios to the messy inputs the tool will actually see.

**Eval framework spec:** [docs/EVAL_FRAMEWORK.md](docs/EVAL_FRAMEWORK.md)

---

## Building in Public

I'm sharing the evaluation and quality improvement process openly as I work through it. Not just polished results, but the annotated traces, error taxonomies, and iteration cycles where I'm building intuition about what "clinically coherent synthetic data" actually means in practice.

**Published so far:**

- **[Failure taxonomy, round 1](docs/ERROR_ANALYSIS.md)** — 14 failure modes from an initial 12-trace annotation pass, each classified as specification failure (fix the prompt) or generalization failure (build an automated evaluator).

- **[Failure taxonomy, round 2](docs/FAILURE_TAXONOMY.md)** — Scaled to 55 annotated traces. 10 stable failure codes with frequency data (e.g., `incomplete_labs` 58%, `diagnosis_error` 44%). Derived from empirical annotation, not hypothesized failure modes. Includes a saturation check: the taxonomy is considered stable when 20 consecutive new annotations produce no new category.

- **Spec fixes implemented** — 7 of the 10 failure modes were specification failures addressable by prompt engineering: expanded condition→lab mapping table (8 new conditions including ED baseline, STI panels, liver disease); diagnosis accuracy rules (specificity modifiers, anti-premature-diagnosis, terminology substitutions); condition→procedure table; controlled substance medication guidance; classification rules (urinalysis ≠ procedure; PHQ-9 ≠ procedure). Deterministic plausibility validator extended to cover CBC components, iron studies, and B12 — the most frequently flagged `range_error` sub-patterns.

**Coming next:**

- **Re-annotation cycle** — Generate new traces with the updated prompt, annotate against the same 10 failure codes, measure whether spec fix failure rates dropped. This closes the loop and reveals which remaining failures are true generalization failures requiring automated evaluators.
- **Binary evaluator development** — Scoped LLM judges for the residual generalization failures, validated against human-labeled ground truth.


---

## Demo Video

[![Watch the demo](https://img.shields.io/badge/Watch%20Demo-Loom-purple?style=for-the-badge&logo=loom)](https://www.loom.com/share/259ab536660d4301a020bb15d12ca46a)

---

## What I Learned

**On AI product development:**
- Structured output from LLMs requires careful prompt engineering and validation layers
- "AI-generated" doesn't mean "no rules" — constraints improve quality
- Graceful degradation (local fallbacks when APIs fail) is essential for reliability
- Evaluation is a product problem, not just an engineering one — defining "good" requires domain judgment before you can automate measurement

**On healthcare interoperability:**
- Standards exist but implementations vary wildly
- Terminology mapping is harder than it looks (SNOMED ↔ ICD-10 isn't 1:1)
- Watermarking synthetic data is a solved problem with clear best practices

**On AI evaluation:**
- Automated format/code validation is the easy part — clinical coherence judgment is the hard part
- Manual error analysis on real traces builds intuition that no amount of prompt engineering replaces
- The right eval criteria come from observed failures, not from guessing what might go wrong

---

## Links

- **Live Demo:** [v0-tabula-health.vercel.app](https://v0-tabula-health.vercel.app)
- **Product Plan:** [Detailed decisions and roadmap](docs/PRODUCT_PLAN.md)
- **Original PRD:** [MVP requirements document](docs/PRD.md)
- **Eval Framework:** [AI evaluation design and roadmap](docs/EVAL_FRAMEWORK.md)
- **Error Analysis:** [Eval cycle 1 — failure taxonomy and spec vs. generalization classification](docs/ERROR_ANALYSIS.md)

---

## Tech Stack

- **Frontend:** Next.js 14, React, Tailwind CSS, shadcn/ui
- **AI:** OpenAI GPT-4 for persona generation
- **Terminology:** UMLS API (SNOMED CT, ICD-10), RxNav (RxNorm), LOINC
- **Deployment:** Vercel

---

*Built by [Scott Tse](https://github.com/sky-t) as an AI product management portfolio piece.*
