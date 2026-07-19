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

## 建議與待辦

- **開新的正式實驗記得改 `EXP_NAME`**：如果要用 `flux_audio_formal.ipynb` 跑一個新設定（換資料量、換超參數），先把設定區的 `EXP_NAME` 改掉，不要沿用 `fluxaudio_s_50k`，否則會直接接續／覆蓋掉已經訓練好的 50000 步 checkpoint。
- **notebook 裡看不出目前訓練進度**：commit 前 cell output 會被清空，所以 GitHub 上完全看不到目前實際跑到第幾步、FAD/CLAP 多少分。已經另外開了 [`TRAINING_LOG.md`](./TRAINING_LOG.md) 這張純文字進度表，每次分段訓練告一段落，補一行「日期 / 實驗名稱 / 進度 / 備註」，之後回頭看或找人幫忙都不用重新解析整份 notebook。
- **生成音檔／checkpoint 目前完全依賴瀏覽器下載**：第七、八階段最後是用 `files.download()` 觸發瀏覽器下載視窗，沒有自動存回 Drive。建議產出真的要交、要留存的版本時，順手把 checkpoint 和音檔也複製一份到 Drive 的固定資料夾，不要只靠瀏覽器下載紀錄，比較不會因為找不到本機檔案而要重新生成一次。
- **如果常態需要一次訓練 7 小時以上**：免費版 Colab 閒置斷線／連線時數上限會是長期困擾，若這個專案還會持續訓練更大的模型或更多 iterations，值得評估升級 Colab Pro（背景執行、更長連線時數），會比一直手動分段省心。

## 評估工具

使用 [ICME26-ATTM-GC-Evaluation](https://github.com/ntu-musicailab/ICME26-ATTM-GC-Evaluation) 計算 FAD 與 CLAP score，notebook 第 8 階段會自動 clone 並安裝。

## 執行方式

先決定要跑哪一份：想快速驗證 pipeline 能不能跑通、改完 code 想確認沒壞掉，就跑 `flux_audio_mini_test.ipynb`；要正式訓練、產出可以拿去交的結果，才跑 `flux_audio_formal.ipynb`。

### `flux_audio_mini_test.ipynb`（煙霧測試，1000 步）

單次執行時間短（約 30-45 分鐘），不需要分段：

1. 在 Google Colab 開啟，Runtime 選 GPU（L4）
2. 從上到下依序執行到底，中途不用中斷
3. 跑完想看結果的話，第七、八階段有試聽/下載 cell，但這只是煙霧測試，生成品質差是正常的，重點是確認每個階段都沒有噴錯

### `flux_audio_formal.ipynb`（正式訓練，50000 步）

**完整跑一次大約要 7～7.5 小時**（實測數據：每 1 萬步約 1.4～1.5 小時），對免費 Colab 或沒辦法整台電腦開機一整天的情況並不友善。這份 notebook 有針對「分段執行」設計過，可以放心中斷、之後接著跑：

**原理**：第六階段訓練時，checkpoint 每 5000 步（`SAVE_CHECKPOINT_INTERVAL`）就會透過 symlink 直接寫進 Google Drive；重新執行同一段訓練指令時，程式會自動偵測 Drive 上有沒有舊 checkpoint，有的話直接從那個 iteration 接著訓練，不會從頭開始。**斷線最多損失的是最後一次存檔之後的進度，最多 5000 步（約 40 分鐘），不是全部重來。**

每次重開一個新的 Colab session，因為 `/content` 會被清空，實際要跑的步驟：

| 步驟 | 要不要跑 | 說明 |
|---|---|---|
| 掛載 Drive、GPU 檢查、第一階段環境安裝 | ✅ 每次都要 | `/content` 重置了，repo 要重新 clone + pip install，約 2-3 分鐘 |
| 第二階段：權重下載＋checkpoint symlink（cell 9-11） | ✅ 每次都要 | 權重檔也在 `/content` 會消失；symlink 建好才讀得到 Drive 上的舊 checkpoint |
| 快取檢查 cell（cell 12） | ✅ 每次都要 | 已經處理好的特徵資料存在 Drive 快取裡，這格會自動偵測並還原 |
| 第三、四、五階段（資料下載/切分/特徵萃取） | ❌ 可以跳過 | 快取還原後不需要重跑，除非快取遺失 |
| 第六階段訓練（cell 30-31） | ✅ 想繼續訓練就跑 | 直接執行即可，會自動接著上次的 iteration，不用改參數 |
| 第七、八階段（推論／評估） | 訓練全部跑完才需要 | 中途不用跑 |

換句話說，每次真正要顧著電腦的，只有「開頭安裝流程（幾分鐘）+ 你這次想跑的訓練時間」，跑一跑想關掉隨時可以關，下次照這個順序重開就好。

**額外提醒**：Colab 免費版分頁閒置太久（約 90 分鐘沒有互動）或連續開太久（約 12 小時上限）都可能自動斷線，規劃分段時間時可以抓保守一點；如果你發現常常斷在很尷尬的地方、覺得 5000 步損失太多，可以把設定區的 `SAVE_CHECKPOINT_INTERVAL` 調小（例如 1000 或 2000），存檔更頻繁、損失更少，代價是存檔本身跟讀寫 Drive 會稍微拖慢一點點整體速度。
