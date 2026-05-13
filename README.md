# 🧬 Functional Profiling Pipeline

Two complementary approaches for functional profiling of metagenomic unmapped reads: **HUMAnN3** for pathway-level annotation and **eggNOG-mapper** for orthology-based functional classification.

---

## Overview

After host genome alignment, unmapped reads are subjected to functional profiling to characterize the metabolic potential of the microbial community. This pipeline supports two methods:

| Method | Input | Database | Output |
|--------|-------|----------|--------|
| HUMAnN3 | Paired-end FASTQ (unmapped reads) | UniRef protein DB | Pathway & gene family abundance |
| eggNOG-mapper | Protein FASTA (`.faa`) | eggNOG OGs | COG, KEGG, GO, EC annotations |

---

## Method 1: HUMAnN3

### Requirements

| Tool | Source |
|------|--------|
| `humann` ≥3.0 | [biobakery](https://github.com/biobakery/humann) |
| `gnu parallel` | system / conda |
| `awk` | system (coreutils) |

### Input

| Item | Description |
|------|-------------|
| Paired-end FASTQ | Unmapped reads from STAR genome alignment |
| File naming pattern | `<sample>_genome_Unmapped.out.mate1` / `...mate2` |
| Kraken2 report *(optional)* | `<sample>_report.txt` — used to skip nucleotide search via taxonomic profile |

Samples are **auto-detected** from `INPUT_DIR` based on the file naming pattern. No hardcoded sample list is needed.

### Configuration

```bash
INPUT_DIR="/data/work/alignment_results/genome"   # Unmapped reads
PROT_DB="/data/work/ref_human"                    # UniRef protein database
KRAKEN_DIR="/data/work/kraken_results/report"     # Kraken2 reports (optional)
OUTDIR="/data/work/funct"                         # Output directory
THREADS=8                                         # Threads per sample
PARALLEL_JOBS=3                                   # Samples processed simultaneously
```

> **Note on parallelism:** `PARALLEL_JOBS=3` with `THREADS=8` means up to 24 CPU cores may be used simultaneously. Adjust based on your server capacity.

### Usage

```bash
chmod +x humann3_pipeline.sh
bash humann3_pipeline.sh
```

### How It Works

1. **Auto-detect samples** — scans `INPUT_DIR` for all files matching `*_genome_Unmapped.out.mate1` and derives sample names automatically.
2. **Taxonomic profile from Kraken2** — if a Kraken2 report exists for the sample, it is converted to a HUMAnN3-compatible taxonomic profile, allowing HUMAnN3 to skip the nucleotide search step entirely and go straight to translated search.
3. **HUMAnN3 translated search** — runs DIAMOND BLASTX against the UniRef protein database using the paired-end reads.
4. **Parallel execution** — uses GNU Parallel to process multiple samples simultaneously. Falls back to Bash background processes (`&` + `wait`) if GNU Parallel is not available.

### Key Flags

| Flag | Purpose |
|------|---------|
| `--bypass-nucleotide-index` | Skip ChocoPhlAn nucleotide DB build |
| `--bypass-nucleotide-search` | Skip nucleotide search entirely |
| `--taxonomic-profile` | Provide pre-computed Kraken2 taxonomy to guide search |
| `--log-level ERROR` | Suppress verbose logs during parallel runs |

### Output

| File | Description |
|------|-------------|
| `<sample>_genefamilies.tsv` | Gene family abundance (UniRef90/UniRef50) |
| `<sample>_pathabundance.tsv` | Metabolic pathway abundance (MetaCyc) |
| `<sample>_pathcoverage.tsv` | Pathway coverage (presence/absence) |
| `<sample>_kraken_for_humann.tsv` | Converted Kraken2 profile (if used) |

---

## Method 2: eggNOG-mapper

### Requirements

| Tool | Source |
|------|--------|
| `eggnog-mapper` ≥2.1 | [eggNOG-mapper](https://github.com/eggnogdb/eggnog-mapper) |
| `gnu parallel` | system / conda |

### Input

| Item | Description |
|------|-------------|
| Protein FASTA | `.faa` files per sample |
| eggNOG database | Pre-downloaded to `DB_DIR` |

All `.faa` files in `INPUT_DIR` are **auto-detected** — no hardcoded sample list needed.

### Configuration

```bash
INPUT_DIR="/data/work/proteins"        # Directory containing .faa files
OUT_DIR="/data/work/eggnog_output"     # Output directory
DB_DIR="/data/work/eggnog"             # eggNOG database directory
THREADS=8                              # Threads per job
PARALLEL_JOBS=4                        # Jobs processed simultaneously
```

> **Note on parallelism:** `PARALLEL_JOBS=4` with `THREADS=8` means up to 32 CPU cores may be used simultaneously. Adjust based on your server capacity.

### Usage

```bash
chmod +x eggnog_pipeline.sh
bash eggnog_pipeline.sh
```

### How It Works

1. **Auto-detect `.faa` files** — scans `INPUT_DIR` for all protein FASTA files.
2. **Skip completed samples** — if a valid, non-empty `.emapper.annotations` file already exists for a sample, it is skipped. Empty or corrupt outputs are automatically re-run.
3. **Run eggNOG-mapper** — runs `emapper.py` per sample using a unique temp directory to avoid cross-job conflicts during parallel execution.
4. **Fallback annotation** — if `emapper.py` exits successfully but the annotations file is empty, the pipeline generates annotations directly from the `.emapper.hits` file as a fallback.
5. **Progress monitoring** — a background monitor reports completed/remaining samples every 30 seconds.
6. **Cleanup** — per-job temp directories are removed after each sample completes.

### Output

| File | Description |
|------|-------------|
| `<sample>_eggnog.emapper.annotations` | Full functional annotations (COG, KEGG, GO, EC, etc.) |
| `<sample>_eggnog.emapper.hits` | Raw DIAMOND hits against eggNOG database |
| `<sample>_eggnog.emapper.seed_orthologs` | Best-hit seed orthologs per query |

### Annotation Columns (`.emapper.annotations`)

```
query  seed_ortholog  evalue  score  eggNOG_OGs  max_annot_lvl  COG_category
Description  Preferred_name  GOs  EC  KEGG_ko  KEGG_Pathway  KEGG_Module
KEGG_Reaction  KEGG_rclass  BRITE  KEGG_TC  CAZy  BiGG_Reaction  PFAMs
```

### Database Setup

```bash
# Download eggNOG database (~50 GB disk space required)
download_eggnog_data.py -y --data_dir /data/work/eggnog
```

---

## Citation

If you use this pipeline, please cite:

- **HUMAnN3**: Beghini et al. *Integrating taxonomic, functional, and strain-level profiling of diverse microbial communities with bioBakery 3.* eLife, 2021. https://doi.org/10.7554/eLife.65088
- **eggNOG-mapper**: Cantalapiedra et al. *eggNOG-mapper v2: Functional Annotation, Orthology Assignments, and Domain Prediction at the Metagenomic Scale.* Molecular Biology and Evolution, 2021. https://doi.org/10.1093/molbev/msab293
- **eggNOG database**: Hernández-Plaza et al. *eggNOG 6.0: enabling comparative genomics across 12535 organisms.* Nucleic Acids Research, 2023. https://doi.org/10.1093/nar/gkac1022
- **GNU Parallel**: Tange O. *GNU Parallel - The Command-Line Power Tool.* USENIX Magazine, 2011. https://doi.org/10.5281/zenodo.16303

---

## License

MIT License. See `LICENSE` for details.
