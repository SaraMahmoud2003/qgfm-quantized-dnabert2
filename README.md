# Reproducibility package: quantized DNABERT-2 promoter study (ENGR701 Assignment 3)

This folder holds the code, records and exported results behind the Assignment 3 manuscript, "A Controlled Framework for Evaluating Accuracy and Deployment Cost of Quantized Genomic Foundation Models". Repository: [TO SPECIFY: GitHub URL once created]. Code licence: [TO SPECIFY: author's choice, e.g. MIT; no LICENSE file is included yet]. The repository holds code, run records and exported results only; it contains no model weights and no DNA sequences.

## 1. What was run

All stages ran in Google Colab on a Tesla T4 between 5 and 8 November 2026. Each notebook reads and writes `MyDrive/qgfm/` (set as `ROOT` in `notebooks/qgfm_utils.py`).

| Order | Notebook | Runtime | Does | Writes (under `MyDrive/qgfm/`) |
|---|---|---|---|---|
| 0 | `00_setup_and_env_check.ipynb` | T4 | Installs and records versions; pins model and dataset revisions; LLM.int8() probe | `env/versions.json`, `run_log.md`, `qgfm_utils.py` |
| 1 | `01_data_audit.ipynb` | CPU | Loads the pinned dataset; chromosome partition; duplicate and conflict checks; GC, TATA, tokens; proximity clusters; MMseqs2 near-duplicate screen | `data/partitions.parquet`, `data/data_report.md` |
| 2 | `02_pilot_audit.ipynb` | T4 | Short C1 pilot, resume test, LLM.int8() audit (48 modules), NF4 check, C3 pilot (validation only) | log entries |
| 3 | `03_train_c1_v2.ipynb` | T4 | C1: full fine-tuning of DNABERT-2, seeds 0 to 4; best checkpoint by validation loss | `runs/c1/seed*/` |
| 4 | `04_quantize_and_sensitivity.ipynb` | T4 | C2 (LLM.int8(), alpha 6) and variants; per-module validation sensitivity; selective and random module sets | `quant/` |
| 5 | `05_train_c3.ipynb` | T4 | C3: CNN on one-hot input, recipe of Tang et al., seeds 0 to 4 | `runs/c3/seed*/` |
| 6 | `06_predict.ipynb` | T4 | Per-window predictions for every configuration and seed; first test-set read recorded in `preds/LOCK.json` | `preds/` |
| 7 | `07_benchmark.ipynb` | T4 | Size, memory, time, energy (one process per checkpoint); batch-size sweep (seed 0) | `bench/` |
| 8 | `08_analysis.ipynb` | CPU | H1 to H6, variants, sensitivity analyses, calibration, tables and figures | `analysis/tables/`, `analysis/figures/`, `analysis/results_summary.json` |
| 9 | `09_exhibits.ipynb` | CPU | Manuscript exhibits (Figs. 1, 3, 4, 10; Tables VI to VIII, X) | `analysis/exhibits/` |
| 10 | `10_posthoc_reviewer.ipynb` | CPU | Post hoc (8 Nov): errors by GC quartile and subtype; per-chromosome counts | `analysis/posthoc/` |
| 11 | `11_collect_for_package.ipynb` | CPU | Added for packaging, not part of the study: copies existing records read-only (no weights, no sequences), reports missing files. Mock-tested only; not yet run on the real Drive | `MyDrive/qgfm_collect_<time>/` |

`notebooks/qgfm_utils.py` (paths, loading, quantization helpers) and `notebooks/qgfm_stats.py` (metrics, ECE, cluster bootstrap, paired and unpaired deltas, Holm, non-inferiority decision, H5 model) are imported by the notebooks. `docs/HOW_TO_RUN.md` is the original stage-by-stage guide. It covers stages 0 to 8 and names `03_train_c1`; the version used for the reported runs is `03_train_c1_v2`. Earlier versions are in `notebooks/superseded/`, with the generator scripts (`build*.py`) and the synthetic-data test of notebook 08 (`_test_08_make_fake.py`).

## 2. Environment (from `env/versions.json`, logged in `docs/RESULTS_LOG.md`, Stage 0)

- GPU: Tesla T4 (compute capability 7.5), driver 580.82.07, CUDA runtime 13.0; Python 3.13.16, cuDNN 92700 (Stage 7 benchmark environment line)
- torch 2.11.0+cu130, transformers 4.57.6, bitsandbytes 0.50.2
- accelerate 1.15.0, datasets 4.8.5, safetensors 0.8.0, einops 0.8.2
- scikit-learn 1.6.1, pandas 2.2.3, pyarrow 23.0.1, tabulate 0.9.0
- MMseqs2 build 2a3dafc (from `mmseqs version`)

Colab images change. Notebook 00 reinstalls the recorded versions; a future image may not support them.

## 3. Data and model

- Dataset: `InstaDeepAI/nucleotide_transformer_downstream_tasks_revised`, task `promoter_all`, Hugging Face revision `851f9946252e90c665cdb3cc3eedb78f1f26197c`. Retrieved by `datasets.load_dataset(..., revision=...)` in notebook 01. Licence status (checked 10 Nov 2026 on the Hugging Face API and the dataset README at the current revision 851f994): the card metadata is empty and has no license field, and the README contains no licence or terms-of-use statement. The README states that the promoters were downloaded from the Eukaryotic Promoter Database (EPD). No permission to redistribute the sequences has been established, so they are not included; users retrieve them from the source with notebook 01, and records are mapped by `window_id` and the source record `name` (chromosome, coordinates, label).
- Model: `zhihan1996/DNABERT-2-117M`, revision `7bce263b15377fc15361f52cfab88f8b586abda0`, loaded with `trust_remote_code`. The model card has no licence tag; the authors' GitHub repository (MAGICS-LAB/DNABERT_2) is Apache-2.0 and asks that the DNABERT-2 paper be cited. Not redistributed here.
- Partitions: test chr20 and chr21 (1,584 windows); validation chr17 to chr19 (3,428); training all other chromosomes (26,572). No record was removed. Notebook 01 rebuilds `partitions.parquet` deterministically from the pinned revision. A sequence-free copy, `data/partition_ids.csv` (window id, record name, partition, coordinates, label, strata, cluster, near-duplicate flag), is written by notebook 11.

## 4. Contents of this folder

```
README.md
docs/      RESULTS_LOG.md (run accounting, deviations X1 to X6), analysis_plan_v2.md,
           HOW_TO_RUN.md, coverage_check.md, posthoc_printout_2026-11-08.md
notebooks/ 00 to 11, qgfm_utils.py, qgfm_stats.py, superseded/
scripts/   figure_label_edits.py, redraw_fig6_tradeoff.py, collect_from_drive.py,
           run_notebook_local.py (runs 08 to 10 outside Colab on a local copy; tested on synthetic records only)
results/notebook_exports/   tables/*.csv, figures/, exhibits/, results_summary.json (notebook exports, 6 Nov)
results/manuscript_figures/ the 10 PDFs used in the manuscript
```

## 5. Figures that differ from the notebook exports

Seven manuscript figures are the notebook exports unchanged. Three were edited afterwards. No data value changed in any of them.

| Figure | File | Edit | Reproduce |
|---|---|---|---|
| Fig. 4 | `fig4_batch_sweep.pdf` | Page box enlarged 28 pt at the top so a clipped label shows (6 Nov); y label reworded to "median; quartile band" (9 Nov) | `python scripts/figure_label_edits.py IN OUT` on the unedited exports |
| Fig. 5 | `fig_instability.pdf` | Page box enlarged 62 pt on the right (6 Nov); legend "C1 seed pairs (floor)" renamed "C1 between-run reference" (9 Nov) | same script |
| Fig. 6 | `fig_tradeoff.pdf` | Redrawn outside Colab (7 Nov) with leader lines and separate markers for measured and estimated sizes; values typed in from the exports | `python scripts/redraw_fig6_tradeoff.py OUT` |

Running these scripts on the files in `results/notebook_exports/` gave PDFs whose 150 dpi renderings are pixel-identical to `results/manuscript_figures/` (checked 9 Nov). Notebooks 08 and 09 were updated afterwards to print the new labels, so a fresh notebook run produces the new wording with notebook typography.

## 6. What is not included

- Model checkpoints (`runs/`, `quant/` weights): derived from DNABERT-2 weights; the Hugging Face model card has no license tag (the authors' GitHub repository is Apache-2.0), and redistribution of fine-tuned weights has not been checked further. [TO SPECIFY: not redistributed / available on request]
- Raw dataset sequences: see section 3.
- Per-window predictions, benchmark records, the test-set lock file and run log: in Drive. Notebook 11 copies them; [TO SPECIFY: whether they are shared].
- Executed notebooks with cell outputs: the copies here have no outputs. Execution evidence is `docs/RESULTS_LOG.md` (filled from the printed stage summaries) and the exports.

## 7. Verification status

- Done: every number in Sections VII to IX of the manuscript was checked against `results/notebook_exports/` and the log (9 Nov). The figure edit scripts were checked as described in section 5.
- Not done: regeneration of tables from saved predictions, any rerun of training or benchmarking, and independent re-execution by another person or machine.
- `analysis_plan_v2.md` was amended once (X3, 5 Nov) before any test-set prediction. It was not hashed or registered. The `MANIFEST.sha256` written by notebook 11, and any Git commit or release of this repository, identify the current package version only. They are not evidence of when files were created or of preregistration.

## 8. Known limitations (as in Section IX of the manuscript)

- One task, dataset, model, library (bitsandbytes 0.50.2) and GPU type. Seeds vary only the training run; no other data partition or hyperparameter setting was tested.
- Shared Colab GPU; clocks read before and after each timed block ranged 585 to 1,410 MHz and were not monitored during it.
- Conversion time was not measured separately from loading, and parameter sharing (tied weights) was not checked. Both remain partly addressed.
- The batch-size sweep used seed 0 only.

## 9. Citation and attribution

Cite the DNABERT-2 paper (Zhou et al., ICLR 2024), the Nucleotide Transformer paper (Dalla-Torre et al., Nature Methods 2025), LLM.int8() (Dettmers et al., NeurIPS 2022), MMseqs2 (Steinegger and Söding, Nat. Biotechnol. 2017) and Tang et al. (Genome Biol., 2025) for the CNN recipe. Full entries are in the manuscript's `references.bib`.
