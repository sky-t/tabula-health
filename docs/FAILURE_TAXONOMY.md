# Failure Taxonomy

Empirically derived from axial coding of **55 clinical reviewer annotations** (open coding phase).

Generated: 2026-05-01
Methodology: [Hamel Husain's eval approach](https://hamel.dev/blog/posts/evals-faq/) — open coding → axial coding → binary evaluators

> **This taxonomy was discovered from the data, not pre-defined.** Categories should be revisited after each annotation round to check for new patterns. The taxonomy is considered stable at saturation (20 consecutive new annotations produce no new category).

---

## Summary Table

| # | Category | Code | Severity | Frequency |
|---|----------|------|----------|-----------|
| 1 | Non-unique patient names | `non_unique_names` | 🟡 Major | 38/55 (69%) |
| 2 | Lab / vital synthetic range wrong | `range_error` | 🟡 Major | 34/55 (62%) |
| 3 | Incomplete lab panel for clinical context | `incomplete_labs` | 🟡 Major | 32/55 (58%) |
| 4 | Diagnosis coding or specificity error | `diagnosis_error` | 🔴 Critical | 24/55 (44%) |
| 5 | Missing expected procedure | `missing_procedure` | 🟡 Major | 9/55 (16%) |
| 6 | Item in wrong clinical category | `item_misclassified` | 🟢 Minor | 9/55 (16%) |
| 7 | Labs ordered but not clinically indicated | `over_testing` | 🟢 Minor | 8/55 (15%) |
| 8 | Missing expected medication for condition | `missing_medication` | 🟡 Major | 8/55 (15%) |
| 9 | Missing specialist referral | `missing_referral` | 🟢 Minor | 5/55 (9%) |
| 10 | Medication dose or frequency wrong | `medication_dosing` | 🟡 Major | 4/55 (7%) |

---

## Category Definitions

### 1. Non-unique patient names `non_unique_names`

**Severity:** 🟡 Major
**Frequency:** 38/55 traces (69%)

The same patient name (first name, last name, or both) appears across multiple generated personas. The reviewer noticed specific patterns: the male name "Alex" recurs frequently; "Patricia Webb" appeared on multiple traces. Cross-trace name reuse was also observed (a different trace had the same name as the PTSD veteran; the lupus patient shared a name with the migraine patient).

**Representative notes:**
> "Repeated name. Might add diagnoses of anaphylaxis and scombroid."

> "Note that many of the male names are Alex, seems repetitive."

> "Repeated name and age from the migraine patient."

> "Repeated name from the PTSD veteran persona."

**Note:** This is likely a systematic defect in the name generation step, not in the clinical reasoning. It is the highest-frequency finding but does not break clinical logic — it would break the illusion of realism if multiple personas are viewed side by side.

---

### 2. Lab / vital synthetic range wrong `range_error`

**Severity:** 🟡 Major
**Frequency:** 34/55 traces (62%)

Synthetic ranges for labs or vital signs are physiologically implausible, wrong for the condition, or internally inconsistent. Includes multiple sub-patterns:

**Sub-patterns observed:**
- **Erythrocyte / RBC component ranges off** — erythrocytes, neutrophils, and lymphocytes are the most frequently flagged. The reviewer repeatedly notes these ranges are wrong across unrelated case types.
- **Alkaline phosphatase "1–12" pattern** — a 1–12 range appears on alk phos across multiple traces; the correct range is approximately 44–147 U/L. This appears to be a recurring generation artifact.
- **HDL impossibly high** — "up to 1000" appeared twice; realistic range for a dyslipidemic patient is 30–70 mg/dL.
- **Reference and synthetic ranges swapped** — on some traces, the reference range contains the synthetic values and vice versa.
- **Ranges unnecessarily narrowed** — reviewer notes vital sign synth ranges are needlessly tight (e.g., weight 60–70 instead of 50–100 for a healthy adult), without any clinical justification from the HPI.
- **Ranges inconsistent with the stated diagnosis** — e.g., H&H synth ranges suggesting anemia in a patient whose diagnosis is "without bleeding"; COPD with H&H trending low (should trend high in chronic hypoxia).

**Representative notes:**
> "Erythrocyte, neutrophil, and lymphocyte synth ranges are wrong."

> "Synth range on HDL wrong (up to 1000 should be like 30–50 for this pt)."

> "For SBP, looks like synth and ref ranges have been switched."

> "Vital sign synths are reasonable but don't need to be narrowed like this."

> "Lab synths are shifting him to bleeding gastritis with the anemia in the H&H synth ranges, but the dx is 'without bleeding' — that's mildly inconsistent."

---

### 3. Incomplete lab panel for clinical context `incomplete_labs`

**Severity:** 🟡 Major
**Frequency:** 32/55 traces (58%)

The generated persona includes some labs but omits the standard-of-care panel a clinician would actually order for the presenting condition. The most common gap is the absence of a full Basic Metabolic Panel (BMP) or Comprehensive Metabolic Panel (CMP) when a partial chemistry result is present. CBC differentials are also frequently incomplete.

**Condition-specific patterns:**
- ER presentations (anaphylaxis, pesticide exposure, severe headache, traveler's diarrhea): reviewer expects CBC + CMP at minimum; often only a subset is present
- Chronic disease follow-up (DM, CKD, dementia): full BMP/CMP is standard; partial panels appear
- Pediatric and adolescent cases: lab panels sometimes over-complete or under-complete relative to the clinical picture
- STI screening: full STI panel (GC/CT, HIV, syphilis, hep B, hep C) expected; only partial present
- HRT monitoring: CMP, estradiol, testosterone, CBC expected; lipids only
- Preeclampsia: urine protein:creatinine ratio and CMP expected

**Representative notes:**
> "Would likely get a whole BMP, not just a calcium and D."

> "In the ER she certainly would get labs (CBC, CMP), though not directly useful really."

> "CBC very incomplete (only WBC present)."

> "For this person would likely do full STI labs, including GC/CT urine (and throat/rectum if exposed in those areas); HIV; syphilis; hep B; hep C."

> "Would also normally have full BMP, not just these few labs, as well as a urine albumin:creatinine."

---

### 4. Diagnosis coding or specificity error `diagnosis_error`

**Severity:** 🔴 Critical
**Frequency:** 24/55 traces (44%)

The diagnosis is wrong, uses the wrong ICD-10 specificity modifier, uses informal language instead of clinical terminology, is premature given the available evidence, or is internally inconsistent with other persona elements.

**Sub-patterns observed:**
- **Wrong specificity modifier** — e.g., "cirrhosis without ascites" when the patient has ascites; "T2DM without complications" when the patient has a diabetic foot ulcer; "CKD stage 1" when labs show stage 3–4 severity; "behavioral disturbance absent" when the patient is wandering
- **Informal / non-clinical terminology** — e.g., "road rash" instead of abrasion/laceration; "breast hypertrophy" instead of gynecomastia
- **Diagnosis too specific for the clinical evidence** — e.g., "suspected ALS" when the evidence supports only "neuromuscular disorder, NOS"; "asthma" when reactive airway disease is more appropriate at first presentation
- **Diagnosis wrong for the clinical presentation** — e.g., IBS diagnosed in a patient with multiple alarm symptoms (suggesting IBD or celiac); stomach irritation listed for a likely inhaled pesticide exposure
- **Duplicate or contradictory diagnoses** — same condition listed twice with different modifiers; diabetes listed as "with complications" and "without complications" in the same persona
- **Missing expected diagnosis** — e.g., no "dehydration" diagnosis when vitals suggest it; no "cellulitis" when a wound is red and streaking; no "anaphylaxis" in an anaphylactic presentation

**Representative notes:**
> "For dx, probably want to say cirrhosis WITH ascites, not without."

> "Dx should probably be reactive airway dz rather than actual asthma, since prompt says no PMH."

> "The dx of IBS is wrong, because of all the alarm sx — more likely IBD or celiac dz."

> "Not sure dx would be 'suspected ALS' just bc patient is worried about it."

> "Duplicative diagnoses for diabetes, would remove the one that says 'without complications.'"

---

### 5. Missing expected procedure `missing_procedure`

**Severity:** 🟡 Major
**Frequency:** 9/55 traces (16%)

A procedure that a clinician would routinely perform for the presenting condition is absent from the persona. Most commonly: imaging not included when indicated; wound care procedures (tetanus, Tdap) not included; emergency procedures missing.

**Representative notes:**
> "Typically would not do a density scan, but rather start with X-ray, then MRI."

> "Another procedure would be tetanus (Tdap) shot."

> "Would be getting... EKG, urine or serum tox screen, even arterial blood gas for any respiratory distress."

> "Could possibly give a nebulizer treatment under procedures."

---

### 6. Item in wrong clinical category `item_misclassified`

**Severity:** 🟢 Minor
**Frequency:** 9/55 traces (16%)

A clinical item is placed in the wrong section of the persona. Common patterns: lab tests listed as procedures; assessments (PHQ-9, GAD-7, Columbia Scale) listed as procedures rather than as clinical decision support tools; encounter codes using vague administrative language instead of clinical codes.

**Representative notes:**
> "Urinalysis should be in labs, not procedures."

> "HIV test isn't a procedure, would remove and put it in labs."

> "'Psych eval' not a real 'procedure,' might be PHQ-9, GAD-7, or Columbia suicide scale."

> "'Behavioral assessment' doesn't really exist as a procedure."

> "'Kidney function eval' is not a typical code." / "Encounters: 'mgt' and 'eval' are redundant and not usually used."

---

### 7. Labs ordered but not clinically indicated `over_testing`

**Severity:** 🟢 Minor
**Frequency:** 8/55 traces (15%)

The persona includes labs that a clinician would not typically order for the presenting condition. This reduces realism and can signal that the model is applying a generic "order some labs" heuristic rather than reasoning about what is actually indicated.

**Representative notes:**
> "Probably wouldn't draw a CBC for this (if any labs, could be a TSH)."

> "Wouldn't necessarily get a CBC for obvious shingles, nor a varicella virus PCR (only if dx is uncertain)."

> "Typically wouldn't check labs in an infant with diaper rash, without any other concerning sx."

> "This is a very full lab panel compared to some of the more medically complicated patients."

---

### 8. Missing expected medication for condition `missing_medication`

**Severity:** 🟡 Major
**Frequency:** 8/55 traces (15%)

A medication that a clinician would routinely prescribe for the stated condition is absent. The reviewer also notes a systematic gap: controlled substances (benzodiazepines, opioids, stimulants) are rarely or never present, and medication allergies are absent from all reviewed traces.

**Representative notes:**
> "Meds should include epipen 0.3 mg."

> "Usually also give magnesium along with labetalol, for sz prevention."

> "He would be on some type of meds, likely an antipsychotic like seroquel or zyprexa."

> "In general I haven't seen much controlled substances like benzo, narcotics, or ADD meds in these lists. Also haven't seen any med allergies."

---

### 9. Missing specialist referral `missing_referral`

**Severity:** 🟢 Minor
**Frequency:** 5/55 traces (9%)

The clinical scenario would typically prompt a referral to a specialist, but none is present in the persona. Often described by the reviewer as "might" or "could" — this is generally a realism gap rather than a hard error.

**Representative notes:**
> "Maybe lactation consultant referral? Otherwise seems good."

> "Possible referral to Rheumatology."

> "Might refer to Immunology (or not, on the first visit), and possibly ENT for ear abnormalities."

---

### 10. Medication dose or frequency wrong `medication_dosing`

**Severity:** 🟡 Major
**Frequency:** 4/55 traces (7%)

The medication is clinically appropriate but the dose or dosing frequency is incorrect or incomplete. Includes missing frequency (PRN without time frame), wrong dose for renal function, and missing maximum dose constraints.

**Representative notes:**
> "Meds are missing doses (not a common error)."

> "Ibuprofen sig should be 400 mg q 6 hrs as needed."

> "For sumatriptan, a typical dose would be 50 mg every 2 hours prn for a max of 200 mg in 24 hours (just says 'as needed' now, without a time frame)."

---

## Patterns Not Yet Reaching Category Status

The following were observed in only 1–2 notes and did not form a stable category. Monitor in the next annotation round:

- **Lab cadence metadata confusing** — "every 365 days" appearing on labs; unclear if this represents order cadence or result date. (notes 17, 55)
- **Duplicate data elements** — the same lab appearing twice in the panel. (notes 46, 50)
- **Race/ethnicity terminology** — "race: Latino" vs. "ethnicity: Hispanic" conflation. (note 52)

---

## Implications for Next Phases

### Annotation tool update (Phase 2)

Replace the pre-defined `commonIssues` checkboxes in the trace viewer with these 10 codes. Reviewers will:
1. Give a binary **Pass / Fail** verdict (the primary signal)
2. Check applicable failure codes (any that apply)
3. Add free-text notes (optional — open coding continues)

### Binary evaluator targets (Phase 4)

Priority order for building automated evaluators (highest frequency + clearest signal first):

| Priority | Category | Evaluator type |
|----------|----------|----------------|
| 1 | `range_error` | Rule-based: flag alk phos 1–12, HDL > 200, O2 < 88 at rest |
| 2 | `non_unique_names` | Rule-based: check name uniqueness across generated batch |
| 3 | `incomplete_labs` | LLM binary judge: "Given this condition, is the lab panel complete?" |
| 4 | `diagnosis_error` | LLM binary judge: "Is the primary diagnosis appropriate and coded correctly for the clinical picture?" |
| 5 | `over_testing` | LLM binary judge: "Are any ordered tests not indicated by the clinical presentation?" |
| 6 | `missing_medication` | LLM binary judge: "Is there a clearly expected medication absent for a stated condition?" |

### Saturation check

After the next 20–45 annotation traces, re-run coding over new notes only. If no new category emerges, the taxonomy is stable and Phase 4 can begin.
