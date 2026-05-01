# mmpR5 Pipeline v3 — Modular Google Colab Notebooks

End-to-end pipeline for *Mycobacterium tuberculosis* **mmpR5 (Rv0678)** variant discovery,
structural annotation, and bedaquiline (BDQ) resistance classification.

---

## Overview

The pipeline is split into **6 module notebooks** orchestrated by a **controller notebook**.
Each module is independently runnable (standalone) or can be driven via
[papermill](https://papermill.readthedocs.io/) from the controller.

```
mmpR5_pipeline_v3/
├── build_notebooks.py              ← regenerate all notebooks from source
├── mmpR5_pipeline_controller.ipynb ← orchestrator (run this in Colab)
├── m0_setup_and_discovery.ipynb    ← MANUAL: ColabFold install + SRA discovery
├── m1_data_acquisition.ipynb       ← SRA download + fastp QC
├── m2_assembly_and_extraction.ipynb← alignment + variant calling + CDS annotation
├── m3_structural_features.ipynb    ← ColabFold + pLDDT + mCSM-PDB2 + MAESTRO
├── m4_ml_classification.ipynb      ← ML resistance classification
└── m5_reporting.ipynb              ← summary table + pLDDT plots + run log
```

---

## Reference genome

| Field | Value |
|-------|-------|
| File | `H37Rv_NC000962_MTBKO_20180411.fasta` |
| Accession | NC_000962 |
| mmpR5 coordinates | `complement(762572..763123)` — 552 bp |
| Chromosome name in FASTA | `H37Rv_NC000962_MTBKO_20180411` |

> **Note:** m2 auto-detects the chromosome name from the FASTA header via `SeqIO.parse()`,
> so any H37Rv FASTA works regardless of the header line.

---

## Quick start (Google Colab)

### Step 1 — Run m0 manually (once per session)

Open `m0_setup_and_discovery.ipynb` in a **separate Colab tab**.

- Installs `bwa`, `samtools`, `bcftools`, `fastp`
- Installs ColabFold + CUDA JAX (**runtime restart required**)
- After restart: re-run the post-install cell to write `colabfold_ready.flag` to Drive
- Optionally run the SRA discovery cell (Module 0b) to pull SRR accessions from a BioProject

> m0 is **never run via the controller** — ColabFold's required runtime restart would
> terminate any running session.

### Step 2 — Run the controller

Open `mmpR5_pipeline_controller.ipynb`. Edit the single config cell:

```python
# ── Edit here ──────────────────────────────────────────────────────────────
SAMPLE_CSV    = "mmpR5_pipeline/input/my_samples.csv"  # CSV: srr, sample_label, phenotype
SINGLE_SRR    = ""            # or: one accession directly
SAMPLE_LABEL  = ""

DRIVE_OUTPUT  = "mmpR5_pipeline/output"
NB_DIR        = "mmpR5_pipeline/notebooks"  # where the m*.ipynb files live on Drive

RUN_ML        = True          # False if <20 labeled samples
# ── End edit ───────────────────────────────────────────────────────────────
```

Then **Run All**. The controller:
1. Writes `pipeline_config.json` to Drive
2. Checks for `colabfold_ready.flag` (halts with a clear error if m0 was not run)
3. Calls m1 → m5 sequentially via papermill
4. Prints a status summary

### Step 3 — Run modules standalone (optional)

Each module has a `# parameters` tagged cell at the top. Set values there and
**Run All** — the module loads `pipeline_config.json` from Drive as a fallback
for any parameter not explicitly set.

---

## Sample CSV format

```csv
srr,sample_label,phenotype
SRR33383911,isolate_001,R
SRR33383902,isolate_002,S
SRR33384000,isolate_003,R
SRR33383963,isolate_004,U
SRR33383960,isolate_005,S
```

`phenotype` values: `R` (resistant), `S` (susceptible), `U` (unknown / unlabeled).
ML training (m4) requires ≥ 20 labeled (R or S) rows.

---

## Output directory layout

```
mmpR5_pipeline/output/
├── pipeline_config.json
├── sample_manifest.csv
├── feature_table_all_samples.csv         ← flat per-isolate feature table (m3)
├── feature_table_all_samples_with_ml.csv ← + OOF ML probabilities (m4)
├── samples/
│   └── <label>/
│       ├── 01_qc/          fastp JSON + HTML
│       ├── 02_alignment/   sorted BAM + coverage
│       ├── 03_variants/    VCF (whole-genome + mmpR5 region)
│       ├── 04_annotation/  aa_annotation.tsv, mCSM/MAESTRO CSVs
│       ├── 05_structure/   ColabFold PDB + scores JSON
│       └── <label>_full_summary.csv
├── 06_reports/
│   ├── completion_table.csv
│   ├── plddt_combined_fig01.png  (max 12 panels/figure)
│   └── pipeline_run_log.txt
└── 07_ml_results/
    ├── ml_cv_metrics.csv
    ├── roc_curves.png
    └── feature_importance.png
```

---

## Module reference

### m1 — Data acquisition
- Parallel SRA download via `ThreadPoolExecutor` + `fasterq-dump`
- fastp QC: Q20 base quality, 50 bp minimum length, adapter auto-detection, deduplication
- Writes `sample_manifest.csv` for downstream modules
- **Checkpointing:** skips any sample whose FASTQ already exists

### m2 — Assembly and extraction
- BWA-MEM alignment (Illumina) or Minimap2 (Nanopore / PacBio)
- Coverage gate: ≥ 10× mean depth over ≥ 90% of mmpR5 (552 bp)
- bcftools mpileup → call (haploid, `--ploidy 1`), filtered to mmpR5 ± 200 bp
- CDS annotation using complement-strand position math:
  ```
  cds_pos_0  = RV0678_END − genomic_pos      (0-based)
  codon_num  = cds_pos_0 // 3
  codon_pos  = cds_pos_0 %  3
  aa_pos     = codon_num + 1                 (1-based)
  ```
- HGVS-style output: `M1V`, `A152V`, `syn_A31A`, `132insG`, `stop_Q51*`, etc.
- **Checkpointing:** skips samples with existing `aa_annotation.tsv`

### m3 — Structural features
> Requires `colabfold_ready.flag` on Drive (written by m0).

- ColabFold AlphaFold2 structure prediction per unique variant sequence
- pLDDT and pTM extraction from `*_scores_rank_001_*.json`
- WT structure: 4NB8 chain A (downloaded from RCSB)
- mCSM-PDB2 API: `ΔΔG_protein`, `ΔΔG_DNA`, `ΔΔG_ligand`
- MAESTRO API: `ΔΔG_MAESTRO`
- Physicochemical features: KD hydrophobicity, Grantham distance (full 190-pair matrix), BLOSUM62, charge, polarity, size
- All API calls wrapped in `try/except` with manual fallback instructions
- Writes `feature_table_all_samples.csv` with `has_structural_data` flag

### m4 — ML classification
> Skipped automatically if fewer than 20 labeled (R/S) samples.

Mirrors the reference R analysis (`main_bothRandSremoved_Gscore_03022026.html`):

| Model | Features |
|-------|----------|
| Baseline LR | `is_frameshift` + `dna_bind_enc` |
| Enhanced LR | + structural + physicochemical |
| Enhanced RF | + structural + physicochemical |
| Enhanced XGBoost | + structural + physicochemical |

- DNA-binding mutation catalogue (50+ variants, Lu et al. 2026) encoded as binary feature
- Stratified 5-fold cross-validation; **G-score** (geometric mean of sensitivity and specificity) as primary metric
- Outputs: `ml_cv_metrics.csv`, ROC curves, permutation feature importance

### m5 — Reporting
- Per-sample completion table (8 pipeline stages)
- Combined pLDDT plots (max `MAX_PER_FIGURE = 12` panels per figure)
- ML results summary
- `pipeline_run_log.txt` with run timestamp, platform info, and stage-by-stage status

---

## Example results — 5-sample local test run

Five SRR accessions from NCBI SRA, processed with the m1 + m2 logic
(ColabFold / API steps excluded from local test):

| Sample | SRR | Reads (post-QC) | Q30 | Mapped | mmpR5 depth | Variant | Effect |
|--------|-----|-----------------|-----|--------|-------------|---------|--------|
| sample_A | SRR33383911 | 3,404,014 | 87.1% | 97.03% | 104× | `syn_A31A` | Synonymous (GCG→GCT, Ala31) |
| sample_B | SRR33383902 | 2,971,654 | 87.8% | 99.56% | 110× | — | No high-confidence call |
| sample_C | SRR33384000 | 3,480,146 | 87.9% | 99.81% | 110× | `M1V` | **Start codon loss** (ATG→GTG) |
| sample_D | SRR33383963 | 2,733,194 | 84.0% | 99.63% |  81× | `syn_A31A` | Synonymous (GCG→GCT, Ala31) |
| sample_E | SRR33383960 | 4,684,782 | 84.2% | 99.86% | 174× | — | No high-confidence call |

**Annotation math for sample_C M1V (pos 763123, T→C on + strand):**
```
cds_pos_0  = 763123 − 763123 = 0
codon_num  = 0 // 3 = 0  →  aa_pos = 1
codon_pos  = 0 %  3 = 0  (first base of codon)
complement of T→C  =  A→G on CDS strand
codon_ref (ATG) → codon_mut (GTG)  →  Met → Val  →  M1V
```

---

## Rebuilding notebooks from source

If you need to regenerate all notebooks (e.g. after editing cell code):

```bash
cd mmpR5_pipeline_v3/
python3 build_notebooks.py
```

The builder reads cell content from an embedded JSON blob and writes all 7 `.ipynb` files
with both bug fixes applied:
1. m2 chromosome auto-detection from FASTA header
2. Controller module filenames corrected to match actual `.ipynb` names

---

## Dependencies (Colab install — handled by m0 / each notebook)

| Tool | Purpose |
|------|---------|
| `bwa` / `minimap2` | Read alignment |
| `samtools` | BAM processing, coverage |
| `bcftools` | Variant calling |
| `fastp` | Read QC and trimming |
| `fasterq-dump` (sra-tools) | SRA download |
| `colabfold` | AlphaFold2 structure prediction |
| `scikit-learn` | LR, RF, cross-validation |
| `xgboost` | Gradient boosting classifier |
| `imbalanced-learn` | Class imbalance handling |
| `papermill` | Notebook orchestration (controller only) |
| `biopython` | SeqIO, Entrez, sequence manipulation |
