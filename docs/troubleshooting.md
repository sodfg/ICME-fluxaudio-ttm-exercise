# Troubleshooting Guide

This document describes known issues and their solutions encountered during FluxAudio training and evaluation on Google Colab.

---

## Colab Runtime Issues

### Ephemeral `/content` Directory

**Problem**: After a Colab runtime resets, `/content` directory and all local files disappear.

**Solution**:
- Store all checkpoints, datasets, and results in **Google Drive** (persistent).
- Use symlinks to link Drive directories into the Colab runtime.
- Re-clone the repository and restore symlinks after reconnection.
- See [SYMLINKS.md](../archive/SYMLINKS.md) for symlink setup details.

---

## Missing Dependencies

### ImportError: No module named `hydra`

**Problem**: Early imports fail with missing `hydra`.

**Solution**: Ensure all dependencies are installed in the setup stage:
```bash
pip install hydra-core
```

### av_bench Dependency

**Problem**: Evaluation modules import `av_bench` even when evaluation is disabled, causing `ImportError`.

**Solution**: 
- Install `av_bench` if doing full evaluation:
  ```bash
  pip install av-bench
  ```
- Or create a lightweight stub module for optional imports during training.

---

## Distributed Training

### KeyError: 'LOCAL_RANK'

**Problem**: Importing training/sample modules directly without `torchrun` raises:
```
KeyError: 'LOCAL_RANK'
```

**Cause**: These modules expect to run under `torchrun` distributed launch, which sets `LOCAL_RANK` environment variable.

**Solution**: Always launch training with:
```bash
torchrun --standalone --nproc_per_node=1 train_fluxaudio_s.sh
```

This is a diagnostic issue during manual imports; distributed launch handles it correctly.

---

## Training Checkpoints

### Scheduler Resume Support

**Problem**: Resuming from a checkpoint may not properly restore learning rate scheduler state.

**Solution**:
- The repository includes a patch supporting scheduler reset on checkpoint load.
- When resuming, the scheduler is reset with warm-up (100 steps) to a constant learning rate (`3e-5`).
- Subsequent resume stages do not reset the scheduler again.

### Skip Final Sampling

**Problem**: At the end of a training stage, EMA synthesis and test sampling may fail due to unrelated dependencies or missing evaluation data, causing the entire training run to fail despite successful model updates.

**Solution**:
- Use `--skip_final_sample` flag (or equivalent configuration) to skip final sampling and evaluation during training.
- Perform evaluation separately after training completes.

---

## Inference and Batch Generation

### Filename Too Long

**Problem**: `infer.py` uses the complete prompt text as the output filename, which fails on long captions:
```
OSError: [Errno 36] File name too long
```

**Solution**:
- Truncate the output filename before saving:
  ```python
  save_path = save_path.with_name(save_path.stem[:180] + save_path.suffix)
  ```
- After batch inference, copy and rename generated files to deterministic clip IDs.

---

## Evaluation

### LAION-CLAP Logging Bug (Python 3.12)

**Problem**: FAD evaluation fails with:
```
TypeError: not all arguments converted during string formatting
```

**Cause**: A `logging.info()` call in LAION-CLAP passes multiple unformatted arguments on Python 3.12.

**Solution**:
- Patch the logging call to concatenate a single formatted message:
  ```python
  logging.info(str(n) + "\t" + ("Loaded" if n in ckpt else "Unloaded"))
  ```
- FAD evaluation then completes successfully with return code 0.

---

## Google Drive Integration

### Mount Error: Mountpoint Already Contains Files

**Problem**: Mounting Google Drive fails with:
```
ValueError: Mountpoint must not already contain files
```

**Cause**: The target mountpoint directory is not empty.

**Solution**:
1. Check if `/content/drive/MyDrive` already exists and contains files.
2. If it does, move it aside:
   ```bash
   mv /content/drive/MyDrive /content/drive/MyDrive.bak
   ```
3. Then mount Google Drive.

---

## Environment Setup

### Python Version

- Primary development: **Python 3.10+**
- Known issue on 3.12: LAION-CLAP logging bug (see above)

### PyTorch and CUDA

- Install compatible versions for your Colab GPU:
  ```bash
  pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
  ```

### Requirements File

- If adding dependencies, update or create a `requirements.txt` in the root directory.
- Use `pip freeze > requirements.txt` to snapshot a working environment.

---

## Reproducing Results

If you need to reproduce the 150k evaluation results:

1. Obtain the 150k checkpoint from Google Drive.
2. Prepare the test dataset (100 Jamendo songs, 300 clips).
3. Use deterministic seeds (42, 43, 44) per clip.
4. Run inference with the configuration documented in [RESULTS.md](../RESULTS.md).
5. Evaluate FAD and CLAP metrics using the same checkpoints and protocols.

See [RESULTS.md](../RESULTS.md) for full metrics and interpretation.

---

## Further Help

Refer to:
- [README.md](../README.md) — Project overview and architecture.
- [RESULTS.md](../RESULTS.md) — Evaluation results and training progression.
- [PROJECT_CONTEXT.md](../PROJECT_CONTEXT.md) — Complete project context and handoff notes.
- [notebooks/](../notebooks/) — Training and evaluation notebooks.
