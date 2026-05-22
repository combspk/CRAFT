---
name: scientific-writing
description: Draft, structure, and review scientific text for toxicology and cheminformatics outputs including hazard assessments, read-across summaries, methods sections, and regulatory dossier narratives
license: MIT
compatibility: opencode
metadata:
  domain: scientific-writing
  audience: toxicologists, regulatory scientists, cheminformaticians
---

## What I do

Guide the drafting of clear, accurate, and appropriately hedged scientific text for toxicology and cheminformatics work — including structured hazard summaries, methods sections, results narratives, regulatory dossier language, and technical reports.

## When to use me

Load this skill when:
- Drafting a hazard assessment narrative, read-across justification, or risk characterization summary
- Writing a methods section describing how data were retrieved or how a model was built
- Describing computational results (QSAR predictions, EpiSuite output, ToxCast results) in prose
- Reviewing or editing scientific text for accuracy, clarity, and appropriate uncertainty language
- Preparing any text intended for regulatory submission (EPA, ECHA, FDA)

---

## Core writing principles

### Precision over fluency
Scientific text must be precise first, readable second. Never sacrifice accuracy for a cleaner sentence. When in doubt, add a qualifier rather than simplify.

### Claim–evidence–caveat structure
Every scientific claim should be accompanied by:
1. **The claim**: a specific, falsifiable statement
2. **The evidence**: the data source, study, or prediction that supports it
3. **The caveat**: the uncertainty, limitation, or assumption that qualifies it

Bad: "Bisphenol A is an endocrine disruptor."
Good: "Bisphenol A exhibits estrogenic activity in multiple in vitro assays (EC50 ~5 nM in the ERα binding assay; ToxCast ER pathway flag active), though in vivo potency is substantially lower due to rapid conjugation and excretion."

### Distinguish prediction from measurement
Always be explicit about data provenance:
- **Predicted/estimated**: EpiSuite, QSAR model, read-across value — use "predicted", "estimated", "modeled"
- **Measured/experimental**: GLP study, peer-reviewed measurement — use "measured", "observed", "reported"
- **Derived/calculated**: RfD, CSF, HQ — use "derived", "calculated"

Never use "the LogKow is 3.2" when the value is a model prediction. Use "the predicted LogKow is 3.2 (EpiSuite KowWIN)".

---

## Document structures

### Hazard assessment summary (single chemical)

```
1. Chemical Identity
   - IUPAC name, CAS, DTXSID, SMILES / InChIKey
   - Molecular formula, MW, structural class

2. Physicochemical Properties
   - Table: MW, LogKow, water solubility, vapor pressure, Henry's Law constant
   - Source and method (measured vs. predicted) for each value
   - Environmental fate flags: persistence, bioaccumulation potential, mobility

3. Toxicokinetics (ADME)
   - Absorption route(s) of concern
   - Key metabolic pathways and reactive intermediates (if known)
   - Protein binding, plasma half-life, route of elimination

4. Hazard Identification
   - Subsections by endpoint: acute, repeat-dose, genotoxicity,
     carcinogenicity, reproductive/developmental, endocrine
   - For each endpoint: data available (Y/N/limited), summary of
     critical study, key finding, reliability score (Klimisch)

5. Dose-Response and Point of Departure
   - Critical endpoint and study
   - NOAEL/LOAEL or BMDL (with CI)
   - Applied uncertainty factors and rationale
   - Derived RfD/RfC or slope factor

6. Risk Characterization (if exposure data available)
   - Exposure scenario(s) and CDI/intake estimate with source
   - HQ (non-cancer) and/or excess cancer risk
   - Uncertainty and sensitivity discussion

7. Data Gaps and Recommendations
   - Missing studies needed to reduce uncertainty
   - Priority testing recommendations
```

### Read-across justification

```
1. Problem formulation
   - Target chemical and the endpoint for which data are lacking

2. Source chemical(s) identification
   - Identity, structural similarity (Tanimoto, shared functional groups)
   - Method used to identify analogs (PubChem similarity search, ChEMBL, expert judgment)

3. Category hypothesis
   - The biological/chemical rationale linking source and target
   - Shared structural alert, metabolic pathway, or receptor interaction

4. Uncertainty analysis
   - Differences between source and target (metabolic, physicochemical)
   - Adjustment applied (if any) and its justification

5. Conclusion and read-across value
   - The adopted value, its source, and the confidence level
```

### Methods section (computational)

Structure for any computational analysis:

```
Data retrieval
  Source database(s), version or access date, API endpoints used,
  any filters applied (organism, assay type, confidence score cutoff).

Data curation
  Standardization steps: SMILES canonicalization, duplicate removal,
  unit conversion, outlier definition and handling.

Descriptor calculation / model
  Software name and version, descriptor set, any preprocessing
  (variance filter, correlation filter, scaling).

Validation
  Train/test split method (scaffold-based, temporal, random),
  metrics reported, cross-validation scheme.

Applicability domain
  Definition method and boundary; proportion of query compounds
  within AD.
```

---

## Language patterns for common situations

### Reporting predicted values
> "The predicted log octanol–water partition coefficient (log Kow) for [compound] is [X] (EpiSuite KowWIN v1.68), suggesting [interpretation]. No measured value was identified in the literature."

### Reporting a NOAEL from an animal study
> "In a 90-day gavage study in Sprague-Dawley rats (OECD TG 408, GLP), the NOAEL for [compound] was [X] mg/kg-day based on [critical effect] in [sex]. The LOAEL was [Y] mg/kg-day. This study was assigned Klimisch reliability score 1."

### Reporting an RfD derivation
> "A chronic oral RfD of [X] mg/kg-day was derived from the NOAEL of [Y] mg/kg-day from [study citation], applying a composite uncertainty factor of [Z] (UFH = 10 for intraspecies variability; UFA = 10 for interspecies extrapolation; UFS = 10 for sub-chronic-to-chronic extrapolation)."

### Reporting a hazard quotient
> "At the estimated chronic daily intake of [E] mg/kg-day, the non-cancer hazard quotient (HQ = CDI / RfD) is [HQ]. [An HQ < 1 indicates that the estimated exposure is below the level of concern based on current toxicity data / An HQ > 1 indicates that the estimated exposure exceeds the reference dose and warrants further evaluation]."

### Describing ToxCast results
> "[Compound] was active in [N] of [total] ToxCast assays tested (hit rate: [%]). Notable activity was observed in estrogen receptor alpha (ERα) binding and transcriptional activation assays (AC50 = [X] µM), nuclear receptor pathway assays ([list]), and DNA damage response assays ([list]). These in vitro activities suggest [biological interpretation]; however, correlation with in vivo apical endpoints has not been established for this compound."

### Acknowledging data gaps
> "No chronic toxicity, reproductive toxicity, or carcinogenicity data were identified for [compound] in the available databases (PubChem, ChEMBL, CompTox, as of [date]). The hazard assessment is therefore based on [basis: read-across / acute data / in vitro screening] and carries substantial uncertainty."

### Expressing uncertainty appropriately
Avoid: "the compound is safe at this exposure level"
Use: "the estimated HQ is below 1, indicating that the exposure is within the range considered acceptable based on current toxicity data and default uncertainty assumptions"

Avoid: "this compound is toxic"
Use: "this compound produced [specific effect] at [dose] in [species/test system]"

---

## Formatting conventions

### Numbers and units
- Report values with appropriate significant figures (3 sig figs for most toxicology values).
- Always include units: mg/kg-day, µM, mg/L, mg/m³, µg/kg-day.
- Use scientific notation for very small or large numbers: 2.4 × 10⁻⁶ (not 0.0000024).
- pH-adjusted values (pKa) and log values: report to 2 decimal places.

### Chemical identifiers in text
- First mention: full IUPAC name, then "([common name], CAS [number])". Subsequent mentions: common name.
- Always include CAS number in the Chemical Identity section.
- Include DTXSID when referencing CompTox data.

### Tables
- Every table needs: title above, source/notes below, units in column headers.
- Predicted values: superscript "a" with footnote "ᵃ Predicted by [model], [version]."
- Missing data: use "ND" (not determined) or "NA" (not available), not blank cells.

### Citations
- Primary literature (peer-reviewed study): Author(s) (Year). Preferred source for PoD.
- Database entry: Database name, accession/identifier, access date.
- Model prediction: model name, version, URL, access date.
- Use consistent citation format throughout a document (ACS, APA, or Vancouver — follow project convention).

---

## Review checklist

Before finalizing any scientific text, verify:

- [ ] Every numerical claim has a source (study, database, model)
- [ ] Predicted values are labeled as predicted
- [ ] Units are present on all numerical values
- [ ] Uncertainty and data limitations are stated
- [ ] Species, route, and duration are specified for all animal study data
- [ ] No absolute claims of safety or toxicity — only relative to specific conditions
- [ ] Chemical is identified by name + CAS on first mention
- [ ] Abbreviations are defined on first use
- [ ] Regulatory thresholds cited refer to the correct jurisdiction and current version
- [ ] All conclusions follow from the evidence presented — no unsupported leaps

---

## Common pitfalls

- **Overconfident language**: regulatory science deals in probabilities and weight-of-evidence, not certainties. Hedge claims appropriately.
- **Endpoint conflation**: "toxic" is meaningless without specifying the endpoint, dose, route, species, and duration.
- **Passive-voice ambiguity**: "the sample was treated" — by whom, with what, when? Specify the subject in methods sections.
- **Mixing measured and predicted values in the same table row without flagging**: always distinguish sources in every row.
- **Reporting the mean without the N and variability**: always state N, SD or SE, and range for any reported mean value.
- **Copying database text verbatim**: always paraphrase and cite. Database descriptions may themselves contain errors or be out of date.
