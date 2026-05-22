---
name: toxicology-assessment
description: Perform systematic hazard identification, dose-response analysis, point-of-departure derivation, and risk characterization following EPA, REACH, and OECD frameworks
license: MIT
compatibility: opencode
metadata:
  domain: toxicology
  frameworks: EPA IRIS, REACH, OECD TG, GHS
---

## What I do

Guide structured toxicological assessments — from hazard identification through risk characterization — following current regulatory frameworks (EPA, REACH/ECHA, OECD) and best practices for chemical safety evaluation.

## When to use me

Load this skill when:
- Characterizing the hazard profile of a chemical from database or literature data
- Deriving a point of departure (PoD) or reference value (RfD, RfC, NOAEL, BMDL)
- Interpreting ToxCast/Tox21 high-throughput screening results
- Performing read-across or chemical category analysis
- Writing or reviewing a hazard assessment, risk characterization, or regulatory dossier section

---

## Hazard identification workflow

Work through the following endpoints in order. For each, note: data available (Y/N), study type, species, duration, and key finding.

### 1. Acute toxicity
- LD50 / LC50 (oral, dermal, inhalation) → GHS acute toxicity category (I–V)
- Primary sources: `comptox_get_hazard_data`, PubChem GHS annotations (`pubchem_get_safety_info`)
- GHS oral LD50 cut-offs (rat): Cat I ≤ 5, Cat II ≤ 50, Cat III ≤ 300, Cat IV ≤ 2000, Cat V ≤ 5000 mg/kg

### 2. Repeat-dose / systemic toxicity
- Sub-acute (28-day), sub-chronic (90-day), chronic (≥ 12 months) NOAEL/LOAEL from animal studies
- Critical endpoint: the most sensitive adverse effect at the lowest dose
- Record: species, strain, sex, route, dose units, duration, NOAEL, LOAEL, and the critical effect

### 3. Genotoxicity battery (OECD TG 471–490)
Minimum battery for regulatory acceptance:
- Ames test (OECD 471) — bacterial gene mutation
- In vitro micronucleus (OECD 487) or chromosomal aberration (OECD 473)
- If in vitro positive: in vivo follow-up required (erythrocyte MN, OECD 474)
- Report: positive/negative/equivocal for each test; note metabolic activation (±S9)

### 4. Carcinogenicity
- IARC classification (Group 1/2A/2B/3): search `pubmed_search` + `pubchem_get_description`
- EPA carcinogen classification (A/B1/B2/C/D/E): `comptox_get_hazard_data`
- Mechanism: mutagenic (DNA-reactive) vs. non-mutagenic (epigenetic, cytotoxic, hormonal)
- Note: mutagenic carcinogens require linear low-dose extrapolation; non-mutagenic allow threshold approaches

### 5. Reproductive and developmental toxicity (RepDT)
- OECD TG 414 (prenatal developmental), 416 (two-generation reproductive), 443 (extended EOGRTS)
- Key endpoints: fertility, implantation, fetal body weight, malformations (structural/functional), NOAEL for maternal vs. developmental toxicity
- EU classification: Repr. 1A/1B/2 under CLP; SVHC candidate list under REACH

### 6. Endocrine disruption
- Query `comptox_get_bioactivity_summary` for ToxCast ER/AR/TH/StAR pathway flags
- EDSP Tier 1 assay flags: ERα binding, uterotrophic, Hershberger, fish short-term reproduction
- WHO/IPCS ED criteria: demonstrate (1) adverse effect, (2) endocrine MoA, (3) plausible link between MoA and effect

### 7. Neurotoxicity and immunotoxicity
- OECD TG 424 (neurotoxicity study in rodents), 426 (developmental neurotoxicity)
- Note AChE inhibition data (relevant for organophosphates/carbamates)
- Immunotoxicity: OECD TG 407 extended, TDAR assay

### 8. Persistence, Bioaccumulation, Toxicity (PBT) / vPvB screening
| Criterion | P threshold | B threshold | T threshold |
|-----------|------------|------------|------------|
| REACH Annex XIII | DT50 water > 40d, soil > 120d | log BCF > 3.3 or log Kow > 5 | chronic NOEC < 0.01 mg/L aquatic, or CMR, ED |
| Use EpiSuite | `biodegradationRate` | `bioconcentration` | ECOSAR chronic values |

---

## Point of departure (PoD) derivation

### NOAEL/LOAEL from animal studies
1. Identify the critical study: lowest NOAEL for the most sensitive relevant endpoint.
2. Convert to human-equivalent dose (HED) if using animal data: HED = animal dose × (BW_animal / BW_human)^0.25 (EPA default body-weight scaling).
3. Apply uncertainty factors (UFs) to derive the Reference Dose (RfD) or Reference Concentration (RfC):

| UF code | Reason | Default value |
|---------|--------|--------------|
| UFH | Human intraspecies variability | 10 |
| UFA | Animal-to-human extrapolation | 10 |
| UFS | Sub-chronic to chronic extrapolation | 10 |
| UFL | LOAEL to NOAEL | 10 |
| UFD | Database deficiency | 1–10 |

**RfD = NOAEL (HED) / (UFH × UFA × UFS × UFL × UFD)**

UFs can be replaced by chemical-specific adjustment factors (CSAFs) when PBPK or toxicokinetic data are available.

### Benchmark dose (BMD) modeling (preferred over NOAEL)
- Use when dose-response data are available (≥ 3 dose groups + control)
- Software: EPA BMDS (https://www.epa.gov/bmds) or R package `bmds`
- Benchmark response (BMR): 10% extra risk for quantal data; 1 SD change for continuous
- Report: BMD10, BMDL10 (95% lower confidence bound), best-fitting model and AIC
- Use BMDL as PoD in place of NOAEL/LOAEL when available

### Cancer slope factor (linear low-dose, mutagenic carcinogens)
- PoD: BMDL10 (preferred) or LED10 from animal bioassay
- Cancer slope factor (CSF) = BMR / BMDL10 (in (mg/kg-day)⁻¹)
- Risk at exposure E: risk = CSF × E
- Report 10⁻⁶, 10⁻⁵, and 10⁻⁴ risk-specific doses

---

## High-throughput toxicology (ToxCast/Tox21) interpretation

### Retrieving and reading results
Use `comptox_get_bioactivity_summary` then interpret:
- **AC50 (µM)**: concentration producing 50% of maximal response — lower = more potent
- **Hit call**: active/inactive/inconclusive
- **Assay endpoint category**: nuclear receptor (NR), stress response (SR), cell viability (CVTOX), epigenetic

### Prioritization approach (National Toxicology Program / EPA)
1. Calculate the **Toxicity Quotient** = median AC50 across active assays (lower = higher concern)
2. Flag activity in:
   - Nuclear receptor pathways (ER, AR, AhR, PPAR, thyroid — endocrine relevance)
   - DNA damage response (p53, ATM/ATR — genotoxicity concern)
   - Mitochondrial function (OXPHOS — general cytotoxicity flag)
3. Compounds active in > 50% of assays at < 1 µM: flag for priority testing
4. Compare AC50 to estimated human exposure (from `comptox_get_exposure_data`) → risk quotient = exposure / AC50 (converted to same units)

### Limitations to note in every report
- In vitro activity ≠ in vivo hazard (bioavailability, metabolism not captured)
- Cell-line specific activity may not translate to primary cell or in vivo responses
- Some assay positives are assay artifacts (cytotoxicity, fluorescence interference) — check CVTOX flags

---

## Read-across and chemical category analysis

### When read-across is applicable
- Target chemical has insufficient toxicity data
- Structural analogs with adequate data exist
- Shared MoA or metabolic pathway can be argued

### Procedure
1. **Define the category**: use `pubchem_similarity_search` or `chembl_get_compound_bioactivities` to identify structural analogs; confirm shared functional groups relevant to toxicity (e.g., all are α,β-unsaturated carbonyls → Michael acceptors → genotoxicity concern).
2. **Gap analysis**: map available vs. needed data for source analogs.
3. **Trend analysis**: confirm monotonic SAR trend (e.g., increasing chain length → increasing bioaccumulation) or justify "worst-case" analog selection.
4. **Uncertainty characterization**: document structural differences that could affect metabolism, biokinetics, or MoA.
5. **ECHA Read-Across Assessment Framework (RAAF)**: score each scenario against ECHA's 16 assessment elements.

---

## Risk characterization

### Non-cancer (threshold) risk
- **Hazard Quotient (HQ)** = chronic daily intake (CDI) / RfD
  - HQ < 1: acceptable risk
  - HQ > 1: potential concern — does not mean harm is certain
- **Hazard Index (HI)**: sum of HQs across chemicals sharing a common endpoint

### Cancer (linear) risk
- **Excess lifetime cancer risk** = CDI × CSF
  - Acceptable range: 10⁻⁶ to 10⁻⁴ (risk management decision)

### Exposure estimates (use CompTox data)
- Retrieve from `comptox_get_exposure_data`
- Default EPA body weight: 70 kg adult, 10 kg child (age 1–6)
- Default inhalation rate: 20 m³/day adult
- Exposure duration: 70 yr (lifetime average), 350 days/yr, 24 hr/day for chronic

### Uncertainty and sensitivity
- State all default assumptions explicitly
- Perform one-way sensitivity analysis on the top 3 uncertain parameters
- Report the hazard/risk estimate as a range, not a point value, when data are limited

---

## GHS classification reference

| Endpoint | GHS class | Criteria summary |
|----------|-----------|-----------------|
| Acute oral tox | Cat 1–5 | LD50 ≤ 5 → ≤ 5000 mg/kg |
| Skin corrosion/irritation | Cat 1A/1B/1C, 2 | Draize/in vitro (EpiDerm, EpiSkin) |
| Serious eye damage | Cat 1, 2A/2B | Draize/BCOP/ICE |
| Skin sensitization | Cat 1A/1B | LLNA EC3, DPRA, KeratinoSens |
| CMR | Carc 1A/1B/2; Repr 1A/1B/2; Muta 1A/1B/2 | IARC + EPA classification |
| STOT-RE | Cat 1/2 | NOAEL: ≤ 10 mg/kg-d (Cat 1) or ≤ 100 mg/kg-d (Cat 2) |
| Aquatic acute/chronic | Cat 1/2/3 | LC50/NOEC; ECOSAR predictions valid for classification |

---

## Data quality and confidence scoring

For each study used, record a reliability score following Klimisch criteria:
| Score | Meaning |
|-------|---------|
| 1 | Reliable without restriction (GLP or equivalent) |
| 2 | Reliable with restrictions (non-GLP but adequate) |
| 3 | Not reliable (significant methodological deficiencies) |
| 4 | Not assignable (insufficient detail) |

Only Klimisch 1 and 2 studies should be used as primary evidence for PoD derivation. Klimisch 3 studies may be used for weight-of-evidence and trend analysis.

---

## Common pitfalls

- **Species extrapolation without scaling**: never apply animal NOAELs directly to humans without body-weight or surface-area scaling.
- **Confusing NOAEL with a safe dose**: NOAEL is the highest tested dose showing no adverse effect — it is bounded by study design, not biology.
- **Ignoring metabolic activation**: a compound may be non-toxic but produce a toxic metabolite (e.g., benzo[a]pyrene → BPDE). Always check CYP metabolism data in `uniprot_get_protein` (CYP1A1, CYP1B1, etc.) and ChEMBL.
- **Over-relying on Tox21 AC50 without cytotoxicity correction**: activity at concentrations causing cytotoxicity is an artifact. Check AC50 relative to cytotoxicity AC50.
- **Missing the critical window**: developmental toxicants may only act in narrow gestational windows — a study that misses the window gives a false-negative NOAEL.
- **Conflating hazard and risk**: a highly hazardous chemical at negligible exposure may pose lower risk than a moderately hazardous chemical at high exposure.
