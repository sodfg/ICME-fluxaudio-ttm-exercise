# ICME26 FluxAudio — Text-to-Music (ATTM)

NTU 專題／畢業論文練習專案，參加 ICME 2026 Grand Challenge 的 Text-to-Music（ATTM）題目，
在 [FluxAudio](https://github.com/ntu-musicailab/ICME26-ATTM-GC-FluxAudio)（MeanAudio 架構）上做資料前處理、訓練、生成與評估，
整個流程在 Google Colab 上執行。

## 檔案

- `flux_audio_formal.ipynb` — 目前唯一維護的主線 notebook，包含完整 8 階段 pipeline。

## Pipeline（8 個階段）

1. 快取檢查 + 環境安裝：clone `ICME26-ATTM-GC-FluxAudio`、安裝套件
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
- **checkpoint / 生成音檔**：訓練與生成都在 Colab 本機 `/content` 進行，最後用 `files.download()` 手動下載到自己電腦（第七、八階段）

⚠️ **這代表 checkpoint 沒有中途自動備份** — 如果 Colab runtime 在你下載 checkpoint 之前斷線，訓練成果會直接遺失。目前 `num_iterations=1000` 只是小規模測試，之後要跑正式訓練時，建議比照舊版 notebook 的做法加回 Google Drive 自動同步（例如訓練中定期把 `exps/{EXP_ID}` symlink 到 Drive），不要只靠最後手動下載一次。

notebook 開頭的設定 cell（`PROJECT_DIR` / `EVAL_DIR` / `EXP_ID`）統一管理路徑與實驗名稱，重新命名實驗時只需要改這一格。

## 評估工具

使用 [ICME26-ATTM-GC-Evaluation](https://github.com/ntu-musicailab/ICME26-ATTM-GC-Evaluation) 計算 FAD 與 CLAP score，notebook 第 8 階段會自動 clone 並安裝。

## 執行方式

1. 在 Google Colab 開啟 `flux_audio_formal.ipynb`
2. 從上到下依序執行；若 Runtime 還留有先前解壓的資料，第一格的快取檢查會自動跳過重複下載
3. 訓練/生成完成後記得手動跑第七、八階段的下載 cell，把音檔和 checkpoint 存到本機，避免 Colab 斷線後遺失成果
