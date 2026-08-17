# FluxAudio-S Training Results — 150k Iterations

## Summary

This document records the verified evaluation metrics for the **FluxAudio-S** model trained to **150,000 iterations** on the Jamendo dataset using the ICME26 ATTM workflow.

**Important**: These results represent a held-out local Jamendo evaluation and should not be directly compared with official hidden-test leaderboard results without noting this distinction.

---

## Evaluation Protocol

- **Test Dataset**: 100 held-out Jamendo songs  
- **Clips per Song**: 3 clips of ~10 seconds each  
- **Total Generated Clips**: 300  
- **Deterministic Seeds**: Based on clip suffix (`42`, `43`, `44`)  
- **Reference Audio**: 300 MP3 files from `test/audios_real`  
- **Validation**: SHA-256 overlap check to ensure generated audio is not compared with reference  

---

## Metrics

### Frechet Audio Distance (FAD)

| Metric | Value |
|--------|-------|
| **FAD Score** | **0.41323621016018786** |
| **Model** | `clap-laion-music` |
| **Note** | Lower is better; represents audio quality and diversity |

**Interpretation**: The FAD score of ~0.413 is plausible and encouraging for a locally held-out evaluation. The model generates audio with reasonable quality relative to reference samples.

**Result File**: `/content/drive/MyDrive/FluxAudio_evaluation/150k/results/fad.csv`

---

### CLAP Score (Text-Audio Alignment)

| Metric | Value |
|--------|-------|
| **Matched Mean** | **0.20371972024440765** |
| **Matched Median** | 0.209974467754364 |
| **Matched Std Dev** | 0.08168002218008041 |
| **Matched Min** | -0.018051641061902046 |
| **Matched Max** | 0.4611210823059082 |
| **Shuffled Mean** | 0.07118639349937439 |
| **Matched vs. Shuffled Gap** | ~0.132530 |
| **Advantage Ratio** | Matched is **~2.86×** shuffled score |

**Checkpoint**: `music_speech_audioset_epoch_15_esc_89.98.pt`

**Interpretation**: 
- The model demonstrates meaningful text-audio alignment, with matched captions scoring significantly higher than shuffled ones.
- However, some samples remain weak (minimum score ≈ -0.018).
- The CLAP checkpoint used differs from the official music-only checkpoint, so direct comparison with challenge leaderboards is not appropriate.

**Historical Comparison**: 
- At 50k iterations, matched mean was ~0.186 and shuffled mean was ~0.031.
- At 150k, the gap has widened, indicating improved alignment, though a formal checkpoint comparison across 50k → 100k → 150k would require identical evaluation protocol and fixed shuffle permutation.

**Result File**: `/content/drive/MyDrive/FluxAudio_evaluation/150k/results/clap_results.json`

---

## Training Progression

The model was trained in stages with resume at checkpoint:

| Stage | Iterations | Notes |
|-------|-----------|-------|
| 1 | 50,000 → 50,100 | Safety/debug run after resume |
| 2 | 50,100 → 55,000 | Continued with constant LR schedule |
| 3 | 55,000 → 60,000 | — |
| 4 | 60,000 → 70,000 | User reported clear audible improvement |
| 5 | 70,000 → 80,000 | — |
| 6 | 80,000 → 100,000 | — |
| 7 | 100,000 → 150,000 | Final stage; output rated as good quality |

---

## Training Configuration

| Parameter | Value |
|-----------|-------|
| Model | `fluxaudio_s` |
| Batch Size | 32 |
| Eval Batch Size | 32 |
| Text Encoder | `t5_clap` |
| `text_c_dim` | 512 |
| RoPE | Enabled |
| MeanFlow | Disabled |
| CFG Strength | 4.5 |
| Learning Rate | 3e-5 (constant schedule) |
| Warm-up Steps | 100 (reset on initial resume) |
| Workers | 8 |
| `ac_oversample_rate` | 5 |
| Compilation | Disabled |
| Final Sampling | Skipped during training |

---

## Related Artifacts

All model checkpoints, datasets, and generated audio are stored in Google Drive and **are not committed to this repository**:

- **Training Checkpoints**: `/content/drive/MyDrive/FluxAudio_checkpoints/fluxaudio_s_150k_stage6/`  
- **Dataset (NPZ Cache)**: `/content/drive/MyDrive/FluxAudio_data/jamendo_meanaudio_ready_cache.zip` (~53.56 GB)  
- **Evaluation Results**: `/content/drive/MyDrive/FluxAudio_evaluation/150k/`  

---

## Notes on Reproducibility

To reproduce this evaluation:

1. Obtain the trained 150k checkpoint from the Drive path above.
2. Prepare the held-out test dataset (100 songs, 300 clips total).
3. Use deterministic seeds (42, 43, 44) per clip.
4. Run FAD using `clap-laion-music` model.
5. Run CLAP using the same checkpoint version as noted above.
6. Document any differences in environment, dependencies, or setup.

See [troubleshooting.md](docs/troubleshooting.md) for known Colab issues and workarounds.
