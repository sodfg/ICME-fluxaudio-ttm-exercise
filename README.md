# ICME26 FluxAudio — Text-to-Music (ATTM)

在 [FluxAudio](https://github.com/ntu-musicailab/ICME26-ATTM-GC-FluxAudio)（MeanAudio 架構）上做資料前處理、訓練、生成與評估，整個流程在 Google Colab 上執行。

---

## 🚀 快速開始

### 我想要...

| 目標 | 選擇 notebook | 所需時間 | 難度 |
|------|-------------|--------|------|
| **驗證 pipeline 能跑通** | [📝 煙霧測試](archive/flux_audio_mini_test.ipynb) | 30-45 分鐘 | ⭐ 簡單 |
| **訓練到 150k 迭代** | [🔥 完整訓練](notebooks/fluxaudio_train_colab.ipynb) | 7-8 小時 | ⭐⭐ 中等 |
| **評估已訓練的模型** | [📊 150k 評估](notebooks/fluxaudio_eval_150k_colab.ipynb) | 1-2 小時 | ⭐⭐ 中等 |

### 基本步驟

1. **準備**：Google Colab + Google Drive（存放 checkpoint 和資料）
2. **選擇**：上表挑一個 notebook
3. **執行**：在 Colab 打開後，按 ▶️ 從上到下執行每一格
4. **結果**：訓練完成後，checkpoint 存在 Drive；評估結果存在 notebook cell 輸出

> 💡 **第一次推薦**：先跑煙霧測試驗證環境，再進行完整訓練

---

## 📁 檔案結構與說明

## 📁 檔案結構與說明

### Notebooks

**主要訓練/評估 Notebooks：**

- **[notebooks/fluxaudio_train_colab.ipynb](notebooks/fluxaudio_train_colab.ipynb)** — 完整訓練 pipeline  
  包含 8 個階段：環境設置、資料加載、特徵萃取、模型訓練、推論、評估。支持從 50k 恢復訓練到 150k 迭代。
  
- **[notebooks/fluxaudio_eval_150k_colab.ipynb](notebooks/fluxaudio_eval_150k_colab.ipynb)** — 150k 評估專用  
  使用 100 首測試歌曲（300 個 10 秒片段）計算 FAD 和 CLAP 分數。

### 文檔

- **[docs/RESULTS.md](docs/RESULTS.md)** — 150k 訓練結果  
  驗證的評估指標：FAD=0.413、CLAP Matched=0.204、CLAP Shuffled=0.071
  
- **[docs/troubleshooting.md](docs/troubleshooting.md)** — 故障排除指南  
  常見 Colab 問題和解決方案

### 歷史檔案（archive/）

- [archive/flux_audio_mini_test.ipynb](archive/flux_audio_mini_test.ipynb) — 煙霧測試版本（1000 iterations）  
  用於快速驗證 pipeline 完整性
  
- [archive/TRAINING_LOG.md](archive/TRAINING_LOG.md) — 訓練進度歷史紀錄
  
- [archive/SYMLINKS.md](archive/SYMLINKS.md) — Google Drive symlink 配置說明

---

## 更多資訊

完整的執行步驟、參數說明、故障排除等詳細信息，請見 notebook 內的 cell 註釋：

- **[notebooks/fluxaudio_train_colab.ipynb](notebooks/fluxaudio_train_colab.ipynb)** — 訓練流程、checkpoint 管理、分段執行指南
- **[notebooks/fluxaudio_eval_150k_colab.ipynb](notebooks/fluxaudio_eval_150k_colab.ipynb)** — 150k 評估流程、CLAP 與 FAD 計算
- **[docs/troubleshooting.md](docs/troubleshooting.md)** — 常見 Colab 問題與解決方案
- **[docs/RESULTS.md](docs/RESULTS.md)** — 150k 訓練的驗證結果

## 注意事項

- 模型權重、資料集與訓練 checkpoint **不放在此 repo**，存放在 Google Drive
- 每個 notebook 第一格都有詳細的前置條件檢查（GPU、環境變數、路徑設定）
- 推薦先跑煙霧測試驗證環境，再進行完整訓練
