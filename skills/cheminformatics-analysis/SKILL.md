---
name: cheminformatics-analysis
description: Analyze chemical structures and datasets using SMILES, molecular descriptors, fingerprints, QSAR, drug-likeness rules, and structure-activity relationship methods
license: MIT
compatibility: opencode
metadata:
  domain: cheminformatics
  tools: RDKit, pandas, scikit-learn, mordred
---

## What I do

Guide structured cheminformatics analysis of chemical structures and datasets — from single-molecule property profiling through multi-compound SAR, scaffold analysis, chemical space visualization, and QSAR model building.

## When to use me

Load this skill when:
- Profiling a single compound or a set of compounds for drug-likeness or environmental fate relevance
- Analyzing bioactivity data retrieved from ChEMBL, PubChem, or CompTox
- Building or evaluating a QSAR/QSPR model
- Performing scaffold or chemical space analysis on a compound library
- Writing Python code that uses RDKit, mordred, or chemical descriptor libraries

---

## Identifier handling

Always resolve ambiguous input to a canonical form before analysis:

1. **Name or CAS → SMILES**: use `pubchem_find_cid` + `pubchem_get_properties` or `episuite_search`.
2. **SMILES canonicalization**: use RDKit `Chem.MolToSmiles(Chem.MolFromSmiles(smi))`. Never trust user-supplied SMILES as canonical without sanitizing first.
3. **Validity check**: call `Chem.MolFromSmiles(smi)` and verify the result is not `None` before any downstream computation. Log and skip invalid structures rather than raising.
4. **InChIKey** is the preferred stable cross-database identifier. Generate with `Chem.MolToInchiKey(Chem.MolFromSmiles(smi))`.

---

## Single-compound property profiling

For each compound, compute and report the following descriptor categories:

### Constitutional descriptors
- Molecular formula and exact monoisotopic mass
- Molecular weight (from RDKit or `pubchem_get_properties`)
- Heavy atom count, ring count, aromatic ring count
- Rotatable bond count

### Physicochemical properties
| Property | Source priority |
|----------|----------------|
| LogP (AlogP / XLogP3) | EpiSuite `logKow` > PubChem XLogP > RDKit Crippen |
| Water solubility (log S) | EpiSuite `waterSolubilityFromLogKow` or `waterSolubilityFromWaterNt` |
| Topological polar surface area (TPSA) | RDKit or PubChem |
| H-bond donors / acceptors | RDKit `Lipinski` module |
| Vapor pressure, Henry's Law constant | EpiSuite `mpbpvp`, `henrysLawConstant` |

### Environmental fate descriptors (use EpiSuite)
- `logKoc` (soil/sediment sorption)
- `biodegradationRate` (BIOWIN ready/ultimate)
- `atmosphericHalfLife` (OH radical rate, direct photolysis)
- `bioconcentration` (BCF, BAF)
- `fugacityModel` (Level III multimedia partitioning)

---

## Drug-likeness and lead-likeness filters

Apply in order; report which rules pass/fail for each compound.

### Lipinski Ro5 (oral drug candidates)
- MW ≤ 500 Da
- LogP ≤ 5
- H-bond donors ≤ 5
- H-bond acceptors ≤ 10
- Violations > 1 → likely poor oral absorption (note: natural products and biologics are exempt)

### Veber rules (oral bioavailability)
- Rotatable bonds ≤ 10
- TPSA ≤ 140 Å²

### Lead-like (for optimization campaigns)
- MW 250–350 Da
- LogP –1 to 3
- H-bond donors ≤ 3, acceptors ≤ 6
- Rotatable bonds ≤ 7

### PAINS (Pan-Assay Interference Compounds)
- Screen using RDKit `FilterCatalog` with `PAINS` filter set.
- Flag but do not automatically discard — PAINS hits warrant experimental follow-up.

### Environmental relevance filters (for toxicology, not drug discovery)
- Persistence: BIOWIN score > 0.5 (likely not readily biodegradable) is a concern
- Bioaccumulation: log BCF > 3.3 triggers PBT screening under REACH and TSCA
- Mobility: log Koc < 1.5 → high groundwater leaching potential

---

## Molecular fingerprints and similarity

### Fingerprint types and their best uses
| Fingerprint | RDKit class | Best for |
|-------------|-------------|---------|
| Morgan (ECFP4) | `AllChem.GetMorganFingerprintAsBitVect(mol, 2, 2048)` | Bioactivity similarity, scaffold hopping |
| RDKit topological | `Chem.RDKFingerprint(mol)` | General similarity, known-actives retrieval |
| MACCS keys | `MACCSkeys.GenMACCSKeys(mol)` | Quick diversity / clustering |
| AtomPairs | `AllChem.GetAtomPairFingerprintAsBitVect(mol)` | 3D-pharmacophore-like similarity |

### Similarity metrics
- **Tanimoto** (Jaccard): standard for binary fingerprints. `DataStructs.TanimotoSimilarity(fp1, fp2)`
- Threshold guidance: > 0.85 = very similar, 0.6–0.85 = analog, < 0.4 = structurally distinct
- For activity cliffs: identify pairs with Tanimoto > 0.85 but >10-fold potency difference

### Clustering compound libraries
1. Compute pairwise Tanimoto matrix (use `DataStructs.BulkTanimotoSimilarity` for efficiency).
2. Cluster with Butina algorithm (`Chem.rdMolDescriptors.CalcNumAtomPairFingerprints` or sklearn AgglomerativeClustering with precomputed distance matrix).
3. Select one representative per cluster (the medoid) for diversity subset selection.

---

## Scaffold analysis

### Murcko scaffolds
```python
from rdkit.Chem.Scaffolds import MurckoScaffold
scaffold = MurckoScaffold.GetScaffoldForMol(mol)
generic_scaffold = MurckoScaffold.MakeScaffoldGeneric(scaffold)
```

### Scaffold frequency analysis
1. Compute Murcko scaffold SMILES for each compound in the dataset.
2. Count scaffold frequency; scaffolds appearing in ≥ 3 compounds are "privileged".
3. Report top-N scaffolds with representative compound and activity range.
4. Flag singleton scaffolds as low-confidence in SAR terms.

---

## SAR analysis procedure

When working with bioactivity data (e.g., from `chembl_get_target_bioactivities`):

1. **Standardize activity values**: convert all measurements to a common unit (nM); log-transform IC50/Ki values (`pIC50 = -log10(IC50_M)`).
2. **Remove outliers**: flag and investigate compounds with > 2 log units spread across replicate measurements.
3. **Identify activity cliffs**: pairs with high structural similarity (Tanimoto > 0.8) but large potency difference (ΔpIC50 > 2). These reveal key pharmacophore elements.
4. **Match Molecular Pairs (MMP)**: use RDKit `rdMMPA` to enumerate single-atom/group substitutions and their effect on activity.
5. **Scaffold SAR table**: for each privileged scaffold, tabulate substitution patterns vs. pIC50 range. Highlight positions where small changes drive large activity shifts.

---

## QSAR/QSPR modeling

### Descriptor selection
- Start with 2D RDKit descriptors (`Chem.rdMolDescriptors`, `Chem.Descriptors`) for interpretability.
- For more coverage, use `mordred` (1,826 descriptors) but apply variance threshold and correlation filter first:
  - Drop descriptors with variance < 0.01.
  - Drop one from any pair with Pearson |r| > 0.95.
- Always compute descriptors with RDKit-sanitized molecules, not raw user SMILES.

### Modeling workflow
1. **Split**: use scaffold-based split (not random) to avoid data leakage. Libraries: `astartes` or manual Butina clustering.
2. **Baseline model**: Random Forest or Gradient Boosting (scikit-learn). These handle descriptor collinearity well.
3. **Evaluation metrics**:
   - Regression: R² (train/test), RMSE, Q²LOO (leave-one-out cross-validation)
   - Classification: AUROC, MCC (Matthews Correlation Coefficient), balanced accuracy
4. **Applicability domain (AD)**: define using leverage (Williams plot) or k-NN distance in descriptor space. Predictions outside AD must be flagged with a warning.
5. **Feature importance**: use permutation importance (not MDI for RF) to identify key descriptors. Map top descriptors back to structural features.

### Minimum reporting requirements (OECD QSAR principles)
- Defined endpoint with units and assay conditions
- Unambiguous algorithm description
- Defined applicability domain
- Goodness-of-fit and cross-validation statistics
- Mechanistic interpretation where possible

---

## Chemical space analysis

### Dimensionality reduction
- Compute Morgan fingerprints for all compounds.
- Apply PCA (fast, interpretable) or UMAP (better local structure) to reduce to 2D.
- Color points by: activity class, scaffold cluster, source dataset, or a continuous property (LogP, pIC50).

### Diversity metrics
- **Scaffold diversity**: unique Murcko scaffolds / total compounds (higher = more diverse)
- **Fingerprint diversity**: mean pairwise Tanimoto distance (target > 0.6 for a diverse library)

---

## Common pitfalls

- **Aromatic perception failures**: always call `Chem.SanitizeMol(mol)` before any descriptor computation.
- **Stereo-blind analysis**: Morgan fingerprints with `useChirality=False` ignore stereochemistry. Set `useChirality=True` when stereochemistry is pharmacologically relevant.
- **Unit confusion**: ChEMBL stores activity values in various units. Always check `standard_units` and convert before comparing.
- **Cliff artifacts**: a "cliff" with Tanimoto = 1.0 often means duplicate entries with different assay conditions, not a true SAR cliff.
- **Descriptor scaling**: tree-based models (RF, XGBoost) do not need feature scaling; SVR and kNN do. Always scale before PCA.
- **Data imbalance**: if active/inactive ratio is > 10:1, use stratified splitting and report MCC not accuracy.
