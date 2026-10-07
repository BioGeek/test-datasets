# nf-core/denovoproteomics test data

Test data for the [nf-core/denovoproteomics](https://github.com/nf-core/denovoproteomics) pipeline.

## Contents

| File | Description | Size |
|------|-------------|------|
| `samplesheet.csv` | Samplesheet for standard mode (2 samples) | <1 KB |
| `samplesheet_mapping.csv` | Samplesheet for mapping mode (2 samples) | <1 KB |
| `samplesheet_hepg2.csv` | Samplesheet for the de novo, rescoring, assembly and mapping fixture (1 sample) | <1 KB |
| `vendor/bruker_timstof_dia.d/` | Bruker timsTOF diaPASEF acquisition (TDF) | 11 MB |
| `vendor/bruker_timstof_dda.d/` | Bruker timsTOF ddaPASEF acquisition (TDF) | 59 MB |
| `vendor/sciex_qtrap.wiff` + `.wiff.scan` | Sciex QTRAP acquisition | 3.3 MB |
| `winnow/winnow_psms.csv` | 500 winnow-scored PSMs, the input to protein assembly | 57 KB |
| `clustalo/cluster_multi_4seq.fasta` | A 4-sequence scaffold cluster, input to CLUSTALO_ALIGN | <1 KB |
| `clustalo/cluster_multi_2seq.fasta` | A 2-sequence scaffold cluster (the minimum alignable) | <1 KB |
| `msspectra/OVEMB150205_12_60.mzML` | First 60 spectra of OVEMB150205_12, for the CI smoke test | 880 KB |
| `msspectra/HepG2_rep1_120.mzML` | First 120 spectra of a HepG2 immunopeptidome run, for de novo, rescoring, assembly and mapping | 728 KB |
| `database/human_HepG2_mini.fasta` | 1,000-protein human slice, the mapping reference for `HepG2_rep1_120.mzML` | 638 KB |

## Vendor spectra

These exercise the conversion modules, which are the only part of the pipeline
that touches vendor binary formats. Conversion is independent of acquisition
mode, so a DIA acquisition tests a converter just as well as a DDA one.

### `vendor/bruker_timstof_dia.d`

Drives `TDF2MZML`. 500 frames, TDF schema 3; converts to an mzML carrying
30 MS1 and 940 MS2 spectra.

**Source:** [tacular-omics/tdfextractor](https://github.com/tacular-omics/tdfextractor),
`tests/data/example_dia.d`, MIT licence, Copyright (c) 2023 Patrick Garrett.

A `.d` only converts if it holds TDF data with a populated `GlobalMetadata`
table. The MannLabs/timsrust fixtures are far smaller but are simulated and lack
that table, so tdf2mzml rejects them; the other small `.d` archives in
circulation are BAF (QTOF), which is a different format entirely.

### `vendor/bruker_timstof_dda.d`

The same acquisition as above but ddaPASEF, kept for end-to-end runs. 59 MB,
converting to 65 MS1 and 2519 MS2 spectra, all of them carrying a precursor
charge state (1922 at 2+, 510 at 3+, 52 at 1+, 25 at 4+, 10 at 5+).

That is the difference that matters: the DIA fixture above converts correctly
but its spectra carry **no** precursor charge, so InstaNovo discards every one
of them and de novo sequencing cannot run on it. The DIA file is the cheap
fixture for testing conversion; this one is the fixture for testing the pipeline.

**Source:** [tacular-omics/tdfextractor](https://github.com/tacular-omics/tdfextractor),
`tests/data/200ngHeLaPASEF_1min.d`, MIT licence, Copyright (c) 2023 Patrick Garrett.

### `vendor/sciex_qtrap.wiff` + `vendor/sciex_qtrap.wiff.scan`

Drives `MSCONVERT`. Converts to a 9.9 MB mzML carrying 2127 MS1 and 108 MS2
spectra. A `.wiff` is only readable alongside its `.wiff.scan` companion, so
both files are needed and the module stages them together.

**Source:** [ProteoWizard/pwiz](https://github.com/ProteoWizard/pwiz),
`pwiz_tools/BiblioSpec/tests/inputs/201208-378803.wiff`, Apache-2.0.

Note that msconvert names its output after the sample recorded inside the
bundle (`sciex_qtrap-ABRR-AUG-1.mzML` here), not after the input file, and a
`.wiff` holding several samples yields one mzML per sample.

The corresponding test is excluded from CI: msconvert runs the closed-source
Sciex reader under wine in a 6.7 GB image carrying a vendor licence agreement
that has to be accepted by hand.

## Assembly input

### `winnow/winnow_psms.csv`

500 winnow-scored PSMs with the columns protein assembly consumes:
`spectrum_id`, `prediction`, `calibrated_confidence`, `psm_fdr`, `psm_q_value`,
`psm_pep`. Lets the assembly subworkflow be tested without first running
prediction and rescoring.

### `clustalo/cluster_multi_*seq.fasta`

Single clusters in the shape `SPLIT_MMSEQS_CLUSTERS` scatters to
`CLUSTALO_ALIGN`: a FASTA of de novo scaffolds that MMseqs2 grouped together.

Taken from a real `greedy` assembly run over `winnow/winnow_psms.csv`.
`cluster_multi_4seq` holds four scaffolds sharing the core
`AGATVGGEGQASQLGGGGGGGGG` with ragged ends, so the alignment has gaps on both
sides. `cluster_multi_2seq` is the two-sequence boundary case — the smallest
input the subworkflow routes to alignment rather than passing through.

These exist because alignment coverage was otherwise incidental. The subworkflow
only invokes `CLUSTALO_ALIGN` when MMseqs2 happens to emit a cluster with more
than one sequence, and whether that happens varies by assembly mode and by tool
version: over the same PSM fixture, `greedy` produces 3 such clusters and `dbg`
158, while `dbg_weighted` produces none.

## CI smoke-test spectra

### `msspectra/OVEMB150205_12_60.mzML`

The first 60 spectra of the canonical Thermo test file, carrying 26 MS2 spectra all of which
have a precursor charge state. Enough to drive the whole pipeline; small enough that CI does
not spend three quarters of an hour on it.

Why it exists: de novo prediction runs on CPU in CI and its cost is linear in spectrum count
— roughly 38 of the 42 minutes of a conda job went to predicting the full file's 1,193
spectra. This subset runs the same chain, conversion through assembly and quantification, in
about 200 seconds.

Measured end to end with `--mode all --assembly_mode all --quantify`: 21 PSMs, 1 mapped
protein, greedy 17 scaffolds, dbg 18, dbg_weighted 0 — the last exercising the pipeline's
empty-scaffold guard, which skips clustering for an unproductive assembler rather than
failing on MMseqs2's `query createdb died`.

Two smaller options were tried and rejected. A single-spectrum MGF starves Winnow's
calibrator (`All spectra were removed during feature computation … insufficient RT spread`).
The HUPO-PSI example mzMLs are MS1-only, so they contain no de novo input at all.

**Derived from:** `nf-core/test-datasets@modules:data/proteomics/msspectra/OVEMB150205_12.mzML`,
first 60 spectra by index.

```bash
msconvert OVEMB150205_12.mzML --mzML --zlib --64 \
    --filter "index [0,59]" --outfile OVEMB150205_12_60.mzML
```

```
sha256: bceb49c29d327279abdc7a92f25c42f4f701df0171d848fc5b9e8b06f9081293
```

### `msspectra/HepG2_rep1_120.mzML` and `database/human_HepG2_mini.fasta`

The first 120 spectra of a HepG2 immunopeptidome run, all of them MS2 with a
precursor charge state, paired with a 1,000-protein slice of the human
proteome.

This is the fixture for everything downstream of conversion: de novo
prediction, Winnow rescoring, assembly and mapping. `OVEMB150205_12_60.mzML`
drives the same chain but its predictions are too poor to exercise it
meaningfully, which is the reason this file exists. Measured over the first 600
spectra of each file, with InstaNovo 1.2.2 and the `winnow-general-model`
calibrator at a 5% FDR threshold:

| | OVEMB150205_12 | HepG2_rep1 |
|---|---|---|
| InstaNovo confidence, median | 0.34 | 0.95 |
| predictions matching the precursor mass within 20 ppm | 34% | 83% |
| PSMs kept by Winnow at 5% FDR | 40 / 591 (7%) | 213 / 589 (36%) |
| filtered peptides found in the human proteome | 2 / 40 (5%) | 96 / 121 (79%) |

The difference is the acquisition. `OVEMB150205_12` is an LTQ Orbitrap Velos
run whose MS2 scans are recorded in the ion trap at unit resolution
(`ITMS + c NSI d Full ms2 …@cid30.00`); `HepG2_rep1` is an Orbitrap run with
high-resolution beam-type CID MS2. De novo models are trained on the latter,
and the fragment mass accuracy they rely on is simply absent from the former.

At 120 spectra the fixture yields 27 PSMs at 5% FDR, 19 unique peptides, 17 of
which map to 29 proteins — abundant human proteins such as hnRNP A1, serum
albumin, haemoglobin and alpha-1-antitrypsin. That is enough for Winnow's FDR
filter to do real work and for a non-trivial assembly threshold to be chosen,
which a fixture retaining a handful of noise PSMs cannot support.

**Spectra derived from:**
`nf-core/test-datasets@modules:data/proteomics/msspectra/HepG2_rep1_small.mzML`,
first 120 spectra by index. That file is itself a subset of an nf-core/mhcquant
test run (`nf-core/test-datasets@mhcquant:testdata/HepG2_rep1_small.mzML`,
"subsets of cell line HepG2 immunopeptidome runs"), searched with Comet under
unspecific cleavage — an HLA ligandome, so the peptides are not tryptic.

```python
from pyopenms import MzMLFile, MSExperiment

exp = MSExperiment()
MzMLFile().load("HepG2_rep1_small.mzML", exp)
exp.setSpectra(list(exp.getSpectra())[:120])
exp.setChromatograms([])
MzMLFile().store("HepG2_rep1_120.mzML", exp)
```

```
sha256: b0e5c6d7c0d1573e8393bf8a3911a0074599c2251fcaee5b80a82706e86e5a2d
```

**FASTA derived from:**
`nf-core/test-datasets@modules:data/proteomics/database/UP000005640_9606.fasta`
(human SwissProt, 20,610 entries). The slice holds the 29 proteins the fixture's
peptides map to plus 971 entries taken at a fixed stride through the rest, so
the reference stays a realistic search space rather than a list of guaranteed
hits. Mapping against the slice reproduces the full-proteome result exactly:
17 of 19 peptides, 29 proteins.

```
sha256: d168f75c80fd7373e0e7a7fc7719761da68c19bf54b2f1a4447030bc8ff97de8
```

## Cross-branch references

Spectra and FASTA references not listed above are reused from the `modules`
branch to avoid data duplication:

- **Spectra**: `data/proteomics/msspectra/OVEMB150205_12.raw` (22.5 MB) and
  `OVEMB150205_14.raw` (26.5 MB)
- **FASTA reference** for mapping mode: `data/proteomics/database/yeast_UPS_mini.fasta`
  (4.2 KB, 10 proteins). Note that this file holds 10 human UPS proteins and no
  yeast; `database/human_HepG2_mini.fasta` above is the reference that pairs with
  spectra in this branch.

## Usage

```bash
# Stub test (CI, no real tools)
nextflow run nf-core/denovoproteomics -profile test -stub --outdir results

# Full test (real tools, small data)
nextflow run nf-core/denovoproteomics -profile test_full --outdir results
```
