# CLAUDE.md — ClinVar Variant Annotation

## What this is
Parse ClinVar variant data for SCN1A, annotate with gnomAD population allele
frequency, and produce a clean annotated table + honest EDA on clinical
significance. Stretch: a pathogenic-vs-benign classifier; optional SCN2A and
APOE extensions.

## Standing principles (do not violate)
- Defensible without AI assistance. The core logic must be something I can rebuild from scratch and explain at a whiteboard. The hand-rolled INFO parser and multiallelic split in particular must stay readable and owned. You may write AI-assisted code, but every non-trivial piece comes with a plain-language explanation of what it does and why, in comments or docs/methodology.md.
- Explain as you build. For each parser, transform, and metric, record the
  alternatives considered and why this choice won.
- Honest framing. Never claim more than the code supports. Separate what I
  implemented from what a library does, flag AI-assisted sections, state
  limitations plainly. For the classifier, surface the allele-frequency leakage issue rather than hiding it (see below).
- Plans before execution. Propose a plan and wait for approval before large
  changes, restructures, or deleting work. I decide.
- Tests where logic is testable. The parsers, multiallelic split, build
  detection, and XML field extraction get pytest tests against committed
  synthetic fixtures. Test the seams I own, not library internals.

## Repo conventions
- Keep raw and derived data out of git. Commit code, configs, synthetic
  fixtures, and docs only.
- README carries exact data-source URLs, versions/release dates, and genome
  build (GRCh38), with the access date.
- docs/methodology.md explains approach, choices, and evaluation.

## Genome-build hygiene
- Everything is GRCh38. Detect and assert the build from the ClinVar VCF header (##reference / ##contig). gnomAD must be v4 (GRCh38), never v2.1.1 (GRCh37).
- Never join ClinVar to gnomAD on raw CHROM:POS. Normalize (left-align, trim, decompose multiallelics) against the GRCh38 reference first, then join on the normalized (CHROM, POS, REF, ALT) key.

## Environment
Run under WSL2 (Ubuntu). uv-managed env (pyproject.toml + uv.lock), Python 3.11.
Python deps: cyvcf2, pysam, pandas, scikit-learn, pytest (stdlib xml.etree for XML).
System tools (NOT Python packages — install via apt, not uv): bcftools, tabix.

## Project-specific instructions
- Filter to SCN1A via the GENEINFO INFO field; if filtering by region, record the exact GRCh38 coordinates and their source. Do not hardcode coordinates silently.
- Classifier labels: collapse CLNSIG to pathogenic (Pathogenic/Likely_pathogenic) vs benign (Benign/Likely_benign); drop VUS and conflicting. Consider filtering to CLNREVSTAT >= 2 stars for a higher-confidence subset and report both.
- Classifier honesty: gnomAD AF is partly circular as a feature, because rarity is itself an ACMG pathogenicity criterion (PM2). Report performance with and without AF, and discuss this as label leakage, not as predictive power.