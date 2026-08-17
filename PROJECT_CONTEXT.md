# FluxAudio TTM Project Handoff

Last updated: 2026-08-17 (Asia/Taipei)

## Purpose

This file transfers the working context of the FluxAudio text-to-music project to a new Codex task, especially one running inside VS Code + WSL/Ubuntu.

Before changing the repository, read this file completely. Preserve existing user work, inspect the repository first, and make cleanup changes on a separate branch with a draft pull request.

## Repository

- User repository: <https://github.com/sodfg/ICME-fluxaudio-ttm-exercise>
- Upstream code used in Colab: <https://github.com/ntu-musicailab/ICME26-ATTM-GC-FluxAudio>
- Main model: FluxAudio-S
- Primary environment: Google Colab
- Training/evaluation artifacts: Google Drive (do not commit large artifacts to GitHub)

## Current Objective

Clean up the user's GitHub repository and publish a clear, reproducible version of the project that contains:

1. A polished formal Colab training notebook.
2. A separate 150k evaluation notebook.
3. A concise README explaining the workflow.
4. A results document recording the verified 150k metrics.
5. A `.gitignore` that excludes datasets, model weights, generated audio, caches, and secrets.

Do not upload model checkpoints, the 53.56 GB dataset archive, extracted NPZ data, 300 generated clips, Hugging Face tokens, or other credentials.

## Training Summary

### Original problem

The original FluxAudio-S checkpoint at 50k iterations produced audio that was low-pitched, monotonous, and structurally weak. Inference itself was eventually confirmed to work; the poor output was mainly a model/training-state problem rather than an audio-player or file-decoding problem.

The earlier learning-rate schedule had effectively decayed too far by 50k. Resume training was changed to a constant learning rate with a short warm-up.

### Effective resume configuration

- Model: `fluxaudio_s`
- Batch size: `32`
- Evaluation batch size: `32`
- Text encoder: `t5_clap`
- `data_dim.text_c_dim=512`
- RoPE enabled
- MeanFlow disabled
- CFG strength: `4.5`
- Learning rate: `3e-5`
- Schedule: constant
- Warm-up: 100 steps when scheduler was reset at the initial resume
- Workers: `8`
- `ac_oversample_rate=5`
- Compilation disabled
- Final sampling skipped during training to reduce failure risk
- Checkpoint and model weights saved periodically to Google Drive

### Training stages

Training resumed successfully through these stages:

- 50,000 -> 50,100: safety/debug run
- 50,100 -> 55,000
- 55,000 -> 60,000
- 60,000 -> 70,000
- 70,000 -> 80,000
- 80,000 -> 100,000
- 100,000 -> 150,000

The user reported clear audible improvement by 70k and considered the 150k output good.

### Important Drive checkpoints

```text
/content/drive/MyDrive/FluxAudio_checkpoints/fluxaudio_s_100k_stage5/
/content/drive/MyDrive/FluxAudio_checkpoints/fluxaudio_s_150k_stage6/
```

Each full resume checkpoint is approximately 2.24 GB. Each inference-only `*_last.pth` is approximately 0.45 GB.

Checkpoint files stay in Drive and must not be committed.

## Dataset

Prepared dataset archive:

```text
/content/drive/MyDrive/FluxAudio_data/jamendo_meanaudio_ready_cache.zip
```

Audit results:

- Compressed size: approximately 53.56 GB
- Train NPZ: 166,496
- Validation NPZ: 299
- Test NPZ/caption rows: 300
- Test songs: 100
- Each test song is divided into three approximately 10-second clips
- Test reference audio in the archive: 100 MP3 files under `test/audios_real`

The qualitative samples used while training were only checkpoint comparisons. Training itself used the full prepared training dataset.

## Verified 150k Evaluation

Evaluation root in Drive:

```text
/content/drive/MyDrive/FluxAudio_evaluation/150k/
```

Evaluation protocol:

- 100 held-out Jamendo test songs
- Three 10-second reference clips per song
- 300 reference clips total
- 300 generated clips total
- Deterministic seeds based on clip suffix (`42`, `43`, `44`)
- Reference and generated files stored in separate directories
- SHA-256 overlap check used to ensure generated audio was not compared with itself

### FAD

- Metric: Frechet Audio Distance using `clap-laion-music`
- FAD: **0.41323621016018786**
- Lower is better
- Result file:

```text
/content/drive/MyDrive/FluxAudio_evaluation/150k/results/fad.csv
```

The value is plausible and encouraging, but it must be described as a local held-out Jamendo evaluation. It must not be presented as directly equivalent to an official hidden-test leaderboard result.

### CLAP

- Evaluated examples: `300`
- Checkpoint: `music_speech_audioset_epoch_15_esc_89.98.pt`
- Matched mean: **0.20371972024440765**
- Matched standard deviation: `0.08168002218008041`
- Matched median: `0.209974467754364`
- Matched minimum: `-0.018051641061902046`
- Matched maximum: `0.4611210823059082`
- Shuffled-caption mean: **0.07118639349937439**
- Matched-minus-shuffled gap: approximately `0.13253`
- Matched score is approximately `2.86x` the shuffled score
- Result file:

```text
/content/drive/MyDrive/FluxAudio_evaluation/150k/results/clap_results.json
```

Interpretation: the model has meaningful text-audio alignment, although some samples remain weak. This CLAP checkpoint differs from the official music-only checkpoint, so the score should not be directly compared with an official challenge table.

### Earlier 50k CLAP reference

- Matched mean: approximately `0.18598`
- Shuffled mean: approximately `0.03118`

The 150k matched score is higher, but the shuffled baseline also changed. A formal checkpoint comparison should regenerate/evaluate 50k, 100k, and 150k under one identical fixed protocol and fixed shuffle permutation.

## Known Colab and Repository Issues

These failures have already been diagnosed. Avoid rediscovering them from scratch.

### Ephemeral Colab runtime

`/content` disappears after a runtime reset. Google Drive persists. Therefore the repository, packages, extracted dataset, symlinks, and code patches may need to be restored after reconnection.

### Missing dependencies

Observed missing imports included:

- `hydra`
- `av_bench`

`av_bench` was used by evaluation imports even when evaluation was disabled. The code was patched so it could be optional for training, or an import-compatible link/module was created.

### `LOCAL_RANK`

Importing certain training/sample modules directly without `torchrun` caused:

```text
KeyError: 'LOCAL_RANK'
```

This was a diagnostic-import issue, not a training failure. Actual distributed launch should use `torchrun --standalone --nproc_per_node=1`.

### Scheduler resume support

The repository needed a patch supporting scheduler reset on checkpoint load. The initial 50k resume reset the scheduler and warmed up to `3e-5`; later stages continued without resetting it again.

### Skip final sample

A patch/option was added to skip final EMA synthesis and test sampling so a successful training stage would not fail at the end due to unrelated sampling/evaluation dependencies.

### Inference checkpoint weights

Required auxiliary files included:

```text
v1-16.pth
best_netG.pt
empty_string_t5.pth
empty_string_clap_c.pth
music_speech_audioset_epoch_15_esc_89.98.pt
```

These are cached in Google Drive and copied into the runtime repository's `weights/` directory when needed. Do not commit them.

### Filename too long during batch generation

`infer.py` used the complete prompt as the output filename and failed on long captions. A practical patch truncated the output stem before `torchaudio.save`, for example:

```python
save_path = save_path.with_name(save_path.stem[:180] + save_path.suffix)
```

Batch evaluation then copied/renamed the generated file to a short deterministic clip ID.

### LAION-CLAP logging bug on Python 3.12

FAD initially failed because a `logging.info` call passed multiple unformatted arguments. The relevant call was patched to concatenate/format a single message, conceptually:

```python
logging.info(str(n) + "\t" + ("Loaded" if n in ckpt else "Unloaded"))
```

After this patch, FAD completed with return code 0.

### Drive mount edge case

Google Drive mounting can fail with:

```text
ValueError: Mountpoint must not already contain files
```

Do not force-mount over a non-empty local `/content/drive`. Check whether `/content/drive/MyDrive` already exists; otherwise move the local conflicting directory aside before mounting.

## Existing Local Artifacts from the Previous Task

The previous Windows Codex workspace contained these generated notebooks/scripts:

```text
flux_audio_formal_v2.ipynb
flux_audio_formal_100k.ipynb
flux_audio_eval_100k_full.ipynb
flux_audio_eval_100k_audit.ipynb
flux_audio_eval_150k.ipynb
build_formal_v2.py
update_formal_100k.py
build_eval_100k_full.py
build_eval_100k_audit.py
build_eval_150k.py
```

Treat them as source material. Inspect the actual GitHub repository and select the newest clean notebooks rather than uploading every intermediate file.

## Recommended Repository Layout

Do not apply this layout blindly. Inspect the current repository first and preserve useful history.

```text
ICME-fluxaudio-ttm-exercise/
├── README.md
├── RESULTS.md
├── .gitignore
├── notebooks/
│   ├── fluxaudio_train.ipynb
│   └── fluxaudio_eval_150k.ipynb
├── docs/
│   └── troubleshooting.md
└── scripts/
    └── README.md
```

If older notebooks are still useful but too numerous, move them to `archive/` with a short explanation instead of deleting them immediately.

## GitHub Publishing Rules

1. Inspect `git status`, the current branch, repository files, history, and remote before editing.
2. Do not stage unrelated user changes.
3. Work on a branch such as `agent/organize-fluxaudio-project`.
4. Keep large and generated artifacts out of Git.
5. Validate notebook JSON and scan notebooks for embedded tokens/secrets.
6. Use a concise commit message such as `organize training and evaluation workflow`.
7. Push the branch and open a **draft pull request**.
8. In the PR body, explain what moved, what was documented, which results were added, and what was deliberately excluded.
9. Do not merge without the user's confirmation.

## Suggested `.gitignore` Coverage

Ensure equivalent patterns exist for:

```gitignore
*.pth
*.pt
*.ckpt
*.npz
*.wav
*.flac
*.mp3
*.zip
__pycache__/
.ipynb_checkpoints/
wandb/
outputs/
data/
weights/
exps/
.env
```

Review the repository before adding broad patterns if it intentionally tracks any small fixtures under these names.

## Instructions for the New Codex Task

Use this prompt after opening the repository inside VS Code + WSL:

> Read `PROJECT_CONTEXT.md` completely. Inspect the current repository and git history. Organize the FluxAudio project into a clear, reproducible training and evaluation repository. Preserve useful existing content, archive rather than delete ambiguous notebooks, add the verified 150k FAD and CLAP results with the stated limitations, prevent large artifacts and secrets from being committed, validate the notebooks, and publish the changes on a new branch as a draft PR. Do not merge the PR.

## Immediate Next Step

Inside WSL/Ubuntu:

```bash
git --version
gh --version
gh auth status
```

Then clone/open the repository and copy this file to its root before starting the new Codex task.
