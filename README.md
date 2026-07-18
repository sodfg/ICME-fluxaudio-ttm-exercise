# ICME26 FluxAudio — Text-to-Music (ATTM)

NTU 專題／畢業論文練習專案，參加 ICME 2026 Grand Challenge 的 Text-to-Music（ATTM）題目，
在 [FluxAudio](https://github.com/ntu-musicailab/ICME26-ATTM-GC-FluxAudio)（MeanAudio 架構）上做資料前處理、訓練、生成與評估，
整個流程在 Google Colab 上執行。

## 檔案

- `flux_audio_formal.ipynb` — 目前唯一維護的主線 notebook，包含完整 8 階段 pipeline。

## 圖解說明

- [三個雲的儲存空間地圖](https://claude.ai/code/artifact/6fc6c7ae-b229-42cd-8079-660d0d822fd0)
  — 視覺化說明 symlink、npz、av_bench，以及資料／checkpoint／生成音檔分別存在哪個空間。

## Pipeline（8 個階段）

1. 掛載 Google Drive、快取檢查 + 環境安裝：clone `ICME26-ATTM-GC-FluxAudio`、安裝套件
2. 下載輔助組件權重（MeanAudio 預訓練權重）
3. 雲端快取流：從 Google Drive 下載/解壓已預處理好的 Jamendo 音檔
4. 資料前處理：切分 train / val / test
5. 特徵萃取：VAE latent + T5/CLAP text embedding
6. 模型訓練（FluxAudio-S），checkpoint 透過 symlink 自動同步到 Google Drive
7. 推論：用訓練好的模型生成音樂並試聽/下載
8. 評估：計算 FAD（Fréchet Audio Distance）與 CLAP score

## 資料與 checkpoint

模型權重、資料集與訓練 checkpoint **不放在這個 repo**。目前這份主線 notebook 的實際存取方式：

- **資料**：透過 `gdown` 從學長分享的 Google Drive 檔案 ID 下載預處理好的 Jamendo zip（第三階段），單向下載，不是掛載自己的 Drive
- **checkpoint**：訓練前會掛載 Google Drive，並把 `exps/{EXP_ID}` symlink 到 `MyDrive/FluxAudio_checkpoints/{EXP_ID}/`，訓練中每次存檔都直接寫進 Drive，不用等訓練結束才手動下載，Colab 斷線也不會遺失
- **生成音檔**：目前仍是訓練/生成都在 Colab 本機 `/content` 進行，最後用 `files.download()` 手動下載到自己電腦（第七階段）

notebook 開頭的設定 cell（`PROJECT_DIR` / `EVAL_DIR` / `EXP_ID` / `DRIVE_CHECKPOINT_DIR`）統一管理路徑與實驗名稱，重新命名實驗時只需要改這一格，checkpoint 在 Drive 上的路徑會自動跟著換。

## 評估工具

使用 [ICME26-ATTM-GC-Evaluation](https://github.com/ntu-musicailab/ICME26-ATTM-GC-Evaluation) 計算 FAD 與 CLAP score，notebook 第 8 階段會自動 clone 並安裝。

## 執行方式

1. 在 Google Colab 開啟 `flux_audio_formal.ipynb`
2. 從上到下依序執行；若 Runtime 還留有先前解壓的資料，第一格的快取檢查會自動跳過重複下載
3. 訓練/生成完成後記得手動跑第七、八階段的下載 cell，把音檔和 checkpoint 存到本機，避免 Colab 斷線後遺失成果
