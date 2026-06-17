# clinvar-variant-annotation

ClinVar/gnomAD variant annotation and pathogenicity EDA for **SCN1A** (GRCh38).

## Problem

`SCN1A` encodes the Nav1.1 sodium channel; loss-of-function variants cause Dravet
syndrome and a broader epilepsy spectrum. This project ingests ClinVar variant
records for SCN1A, annotates each with gnomAD population allele frequency, and
produces a clean annotated table plus an honest exploratory analysis of the
clinical-significance distribution.

The point of the project is the engineering and the discipline around it: correct
VCF `INFO` parsing and multiallelic handling, a streaming parse of the (multi-GB)
ClinVar XML, strict genome-build hygiene, and a normalized variant join. Because
SCN1A is loss-of-function driven, the analysis includes a biological sanity check:
truncating variants should enrich for pathogenic classifications.

**Scope:** current target is SCN1A. Generalization to SCN2A and a contrastive
analysis of APOE (where the pathogenic/benign frame breaks down) are planned
extensions, not yet built.

## Data sources

All data is **gitignored** and downloaded locally; nothing here is committed.
Genome build is **GRCh38** throughout.

| Source | File | Version / release | Build | URL |
|---|---|---|---|---|
| ClinVar (VCF) | `clinvar_*.vcf.gz` | `<release date>` | GRCh38 | NCBI ClinVar FTP, `vcf_GRCh38/` |
| ClinVar (XML) | `ClinVarVCVRelease_*.xml.gz` | `<release>` | n/a | NCBI ClinVar FTP |
| gnomAD | `gnomad.v4.sites.chr2.vcf.bgz` (+ `.tbi`) | `<v4.x>` | GRCh38 | gnomAD downloads |
| Reference | GRCh38 primary assembly FASTA | `<source / version>` | GRCh38 | `<Ensembl / UCSC / NCBI>` |

_Access date: `<YYYY-MM-DD>`_

### Obtaining the data

```bash
# ClinVar VCF + XML: download from the NCBI ClinVar FTP into data/

# gnomAD: tabix-slice the SCN1A region from the chr2 sites VCF
#   (the .tbi index must sit alongside the .bgz)
tabix gnomad.v4.sites.chr2.vcf.bgz chr2:<start>-<end> > data/gnomad_scn1a.vcf

# Reference FASTA: GRCh38 primary assembly, for bcftools norm
```

SCN1A is on chr2 (2q24.3). `TODO:` record the exact GRCh38 coordinates and their
source (GENCODE/RefSeq). The same chr2 slice also covers SCN2A for the extension.

## Environment & setup

Runs under **WSL2 (Ubuntu)**, Python **3.11**, managed with **uv**.

```bash
git clone <repo-url>
cd clinvar-variant-annotation
uv sync                          # builds the env from pyproject.toml + uv.lock
sudo apt install bcftools tabix  # system tools, not Python packages
```

## How to run

The intended pipeline order (flag names may firm up during the build):

```bash
# 1. ingest + filter ClinVar VCF to SCN1A
python -m src.ingest_clinvar_vcf --vcf data/clinvar_GRCh38.vcf.gz --gene SCN1A \
    --out data/scn1a_clinvar.parquet

# 2. bounded streaming extract from the ClinVar XML (SCN1A subset)
python -m src.parse_clinvar_xml --xml data/ClinVarVCVRelease.xml.gz --gene SCN1A \
    --out data/scn1a_xml.parquet

# 3. normalize against the GRCh38 reference (left-align, decompose multiallelics)
python -m src.normalize --in data/scn1a_clinvar.parquet --ref data/GRCh38.fa \
    --out data/scn1a_norm.parquet

# 4. annotate with gnomAD allele frequency on the normalized key
python -m src.annotate_gnomad --in data/scn1a_norm.parquet \
    --gnomad data/gnomad_scn1a.vcf --out data/scn1a_annotated.parquet

# 5. assemble the annotated table
python -m src.build_table --in data/scn1a_annotated.parquet \
    --out results/scn1a_table.csv

# 6. exploratory analysis
jupyter lab notebooks/eda.ipynb

# stretch: pathogenic-vs-benign classifier
python -m src.classify --in data/scn1a_annotated.parquet \
    --report results/classifier_report.md
```

Tests:

```bash
uv run pytest
```

## Results / EDA summary

`TODO:` fill after running. Cover:

- Clinical-significance (CLNSIG) distribution for SCN1A.
- gnomAD allele-frequency distribution by significance (pathogenic variants
  should skew ultra-rare or absent).
- Truncating-vs-pathogenic enrichment (the LoF sanity check).
- If the classifier was built: metrics reported both with and without the
  allele-frequency feature, on all reviewed variants and on the CLNREVSTAT
  >= 2-star subset.

## Methodology

See [`docs/methodology.md`](docs/methodology.md) for the approach, the
build-hygiene rationale, the normalization-before-join reasoning, and the
classifier leakage discussion.

## Limitations & honest scope

- **SNV/indel only.** Repeat expansions and large CNVs are outside what this
  pipeline parses.
- **Single gene.** Results are SCN1A-specific; the pipeline's generality is only
  demonstrated by the SCN2A extension.
- **Classifier is not a clinical tool.** gnomAD allele frequency is partly
  circular as a feature, because rarity is itself an ACMG pathogenicity criterion
  (PM2). Performance is reported with and without AF and discussed as label
  leakage, not as independent predictive power.
- **ClinVar significance is an assertion, not ground truth.** Classifications
  reflect submitter assertions and review status; VUS and conflicting calls are
  excluded from the classifier.
- **gnomAD AF is population frequency**, not functional evidence.

## AI assistance disclosure

Built with Claude Code under the constraints in [`CLAUDE.md`](CLAUDE.md). The
hand-rolled `INFO` parser and multiallelic split (`src/parse_info.py`,
`src/split_multiallelic.py`) are written from scratch and are defensible without
assistance. Library-based ingestion (`cyvcf2`), normalization (`bcftools norm`),
and the classifier baselines (`scikit-learn`) use standard tools. `TODO:` refine
this list once the repo is built to reflect exactly which modules were
AI-assisted versus owned-and-rederived.

## License

MIT. See [`LICENSE`](LICENSE).
