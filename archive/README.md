# Archive

This directory contains older notebooks and logs preserved for reference and historical tracking.

## Contents

- **`flux_audio_mini_test.ipynb`** — Smoke test notebook used to validate the full pipeline on a small dataset (1000 iterations). Kept as a reference for debugging and quick validation runs.

- **`TRAINING_LOG.md`** — Detailed training progression log documenting the stages from 50k to 150k iterations, learning rate adjustments, and known issues encountered.

- **`SYMLINKS.md`** — Documentation of symlink setup used to connect Google Drive directories to the Colab runtime environment for checkpoint and dataset access.

## Why Archived?

These files remain in the repository for historical and reference purposes, but are not part of the main workflow. The formal training notebook is now in [../notebooks/](../notebooks/).

## Using These Files

- To run a quick test: Use `flux_audio_mini_test.ipynb` on a small subset to verify the pipeline works before committing to the full 150k training.
- For troubleshooting symlink issues: Refer to `SYMLINKS.md` and [../docs/troubleshooting.md](../docs/troubleshooting.md).
- For historical context: Review `TRAINING_LOG.md` to understand the progression of the 150k training.
