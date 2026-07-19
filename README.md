# ICME26 FluxAudio — Text-to-Music (ATTM)

NTU 專題／畢業論文練習專案，參加 ICME 2026 Grand Challenge 的 Text-to-Music（ATTM）題目，
在 [FluxAudio](https://github.com/ntu-musicailab/ICME26-ATTM-GC-FluxAudio)（MeanAudio 架構）上做資料前處理、訓練、生成與評估，
整個流程在 Google Colab 上執行。

## 檔案

- `flux_audio_formal.ipynb` — **正式訓練**主線 notebook，`fluxaudio_s_50k` 實驗，完整 50000 iterations，包含完整 8 階段 pipeline。
- `flux_audio_mini_test.ipynb` — **煙霧測試**版，`mini_test_1k` 實驗，只跑 1000 iterations，用來快速驗證整條 pipeline 能不能跑通，不求生成品質。

兩份檔案用不同的 `EXP_ID` / `EXP_NAME`，checkpoint 各自存在 Google Drive 底下不同的資料夾，互不影響，可以放心各自執行。

## 圖解說明

- [三個雲的儲存空間地圖](https://claude.ai/code/artifact/6fc6c7ae-b229-42cd-8079-660d0d822fd0)
  — 視覺化說明 symlink、npz、av_bench，以及資料／checkpoint／生成音檔分別存在哪個空間。

## Pipeline（8 個階段，兩份 notebook 共通架構）

1. GPU 檢查、掛載 Google Drive、環境安裝：clone `ICME26-ATTM-GC-FluxAudio`、安裝套件
2. 下載輔助組件權重（MeanAudio 預訓練權重）
3. 載入訓練資料（兩份 notebook 在這裡的實作不同，見下方「資料與 checkpoint」）
4. 資料前處理：切分 train / val / test
5. 特徵萃取：VAE latent + T5/CLAP text embedding
6. 模型訓練（FluxAudio-S），checkpoint 透過 symlink 自動同步到 Google Drive，支援斷線後自動接續訓練
7. 推論：用訓練好的模型生成音樂並試聽/下載
8. 評估：計算 FAD（Fréchet Audio Distance）與 CLAP score

## 資料與 checkpoint

模型權重、資料集與訓練 checkpoint **不放在這個 repo**。兩份 notebook 第三階段的資料來源不一樣：

- **`flux_audio_mini_test.ipynb`**：透過 `gdown` 從學長分享的 Google Drive 檔案 ID 下載**原始**、尚未特徵萃取的 Jamendo zip，單向下載，不掛載自己的 Drive，每次都要重新切分＋萃取特徵。
- **`flux_audio_formal.ipynb`**：直接從**自己的** Drive 讀取已經特徵萃取好的 npz 快取（`FluxAudio_data/jamendo_meanaudio_ready_cache.zip`），一次性還原即可直接進入訓練，省掉重複做第三、四、五階段；跑完第一次後，第五階段後面會自動把處理好的結果打包回 Drive，之後開新 Runtime 都能直接沿用。

checkpoint 與生成音檔的存取方式兩份共通：

- **checkpoint**：訓練前會掛載 Google Drive，並把 `exps/{EXP_ID}` symlink 到 `MyDrive/FluxAudio_checkpoints/{EXP_ID}/`，訓練中每次存檔都直接寫進 Drive，不用等訓練結束才手動下載，Colab 斷線也不會遺失，重新連線會自動偵測舊 checkpoint 接續訓練。
- **生成音檔**：訓練/生成都在 Colab 本機 `/content` 進行，沒有自動同步回 Drive，最後要手動跑 `files.download()` 的 cell 下載到自己電腦（第七、八階段）。

notebook 開頭的設定 cell 統一管理路徑與實驗名稱（`flux_audio_formal.ipynb` 是 `EXP_NAME`／`flux_audio_mini_test.ipynb` 是 `EXP_ID`），重新命名實驗時只需要改這一格，checkpoint 在 Drive 上的路徑會自動跟著換。

## 評估工具

使用 [ICME26-ATTM-GC-Evaluation](https://github.com/ntu-musicailab/ICME26-ATTM-GC-Evaluation) 計算 FAD 與 CLAP score，notebook 第 8 階段會自動 clone 並安裝。

## 執行方式

1. 想快速驗證 pipeline 能不能跑通，先跑 `flux_audio_mini_test.ipynb`；要正式訓練/產出結果才跑 `flux_audio_formal.ipynb`
2. 在 Google Colab 開啟對應的 notebook，從上到下依序執行
3. 訓練/生成完成後記得手動跑第七、八階段的下載 cell，把音檔和 checkpoint 存到本機，避免 Colab 斷線後遺失成果
