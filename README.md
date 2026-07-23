# ICME26 FluxAudio — Text-to-Music (ATTM)

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

## 模型選擇：為什麼用 FluxAudio-S，不是 FluxAudio-L

baseline repo 提供兩種模型大小，兩份 notebook 從頭到尾都只用過 **FluxAudio-S**：

| | FluxAudio-S | FluxAudio-L |
|---|---|---|
| 參數量 | 120M | 480M（4 倍大） |
| 訓練腳本 | `train_fluxaudio_s.sh` | `train_fluxaudio_l.sh` |

**為什麼選 S：**

- **記憶體限制**：官方訓練 S 版（`batch_size=128`）在 A6000 上就要吃到 45GB VRAM；Colab 這邊用的是 22.5GB 記憶體的顯卡，訓練 S 版已經得把 `batch_size` 降到 32 才塞得下。L 版參數量是 4 倍，很可能連塞進去都有困難，得再大幅壓低 batch size，訓練也會更慢。
- **跟官方 baseline 比較時是公平的**：查過官方公布的 baseline 分數（FAD=0.6073、CLAP=0.2015），[明確寫著是針對 `fluxaudio_s` 模型算的](https://puzzle-ferry-27e.notion.site/MeanAudio-Baseline-Demo-3192b9caac2680658c1ed7e3081729a2)，不是 L 版。這個專案訓練出來的 `fluxaudio_s_50k` 拿去跟這組數字對比，模型大小是對齊的、比較是有意義的。
- **符合這個專案的目標**：這個專案的重點是「把 baseline pipeline 完整跑通一次、自己真的搞懂每個環節」，不是要衝出最高分數，S 版已經足夠達成這個目標，暫不考慮換 L 版。

## 資料與 checkpoint

模型權重、資料集與訓練 checkpoint **不放在這個 repo**。兩份 notebook 第三階段的資料來源不一樣：

- **`flux_audio_mini_test.ipynb`**：預設透過 `gdown` 從學長分享的 Google Drive 檔案 ID 下載**原始**、尚未特徵萃取的 Jamendo zip，單向下載，不掛載自己的 Drive，每次都要重新切分＋萃取特徵。
- **`flux_audio_formal.ipynb`**：直接從**自己的** Drive 讀取已經特徵萃取好的 npz 快取（`FluxAudio_data/jamendo_meanaudio_ready_cache.zip`），一次性還原即可直接進入訓練，省掉重複做第三、四、五階段；跑完第一次後，第五階段後面會自動把處理好的結果打包回 Drive，之後開新 Runtime 都能直接沿用。

**兩份notebook 現在可以共用同一份快取**：因為兩份處理的是同一套完整資料集、同樣的切分參數（`val_samples=100`、`test_samples=100`、預設 `seed=42`），`flux_audio_mini_test.ipynb` 在第二階段後面多了一格「【可選】快取捷徑」，可以直接讀取 `flux_audio_formal.ipynb` 存在 Drive 的那份快取，跳過第三、四、五階段。**這格預設是不執行的**——因為 mini_test 存在的意義就是驗證整條 pipeline（尤其是三、四、五階段的程式碼）能不能跑通，如果每次都直接抓快取跳過，就測不到那幾階段有沒有被改壞了。什麼時候該用哪種：

| 情境 | 要不要跑那格可選 cache cell |
|---|---|
| 剛改了第三、四、五階段的程式碼（或第一次不確定資料處理邏輯還通不通），想確認真的沒壞 | ❌ 不要跑，讓它照原本流程走一次完整版 |
| 只是想快速驗證第六、七、八階段（訓練/推論/評估）的邏輯，資料處理那段這幾天沒動過 | ✅ 直接跑，省下大半時間 |

checkpoint 與生成音檔的存取方式兩份共通：

- **checkpoint**：訓練前會掛載 Google Drive，並把 `exps/{EXP_ID}` symlink 到 `MyDrive/FluxAudio_checkpoints/{EXP_ID}/`，訓練中每次存檔都直接寫進 Drive，不用等訓練結束才手動下載，Colab 斷線也不會遺失，重新連線會自動偵測舊 checkpoint 接續訓練。
- **生成音檔**：訓練/生成都在 Colab 本機 `/content` 進行，沒有自動同步回 Drive，最後要手動跑 `files.download()` 的 cell 下載到自己電腦（第七、八階段）。

notebook 開頭的設定 cell 統一管理路徑與實驗名稱（`flux_audio_formal.ipynb` 是 `EXP_NAME`／`flux_audio_mini_test.ipynb` 是 `EXP_ID`），重新命名實驗時只需要改這一格，checkpoint 在 Drive 上的路徑會自動跟著換。

### `flux_audio_formal.ipynb` 設定區參數說明

| 參數 | 意思 | 目前的值 | 要不要調整 |
|---|---|---|---|
| `EXP_NAME` | 實驗名稱，決定 checkpoint 存在 Drive 哪個資料夾，也決定要不要接續舊 checkpoint | `fluxaudio_s_50k` | 開新實驗一定要改，見上方提醒 |
| `NUM_ITERATIONS` | 總共要訓練幾步 | `50000` | 依實驗需求，通常步數越多模型學得越完整，但也越花時間 |
| `BATCH_SIZE` | 每一步同時丟幾筆資料進去，算出平均誤差後才調整模型一次 | `32` | 不是越高越好，受 GPU 記憶體限制（L4 太高會爆記憶體），且通常要搭配調整學習率一起改，不建議單獨亂調 |
| `EVAL_BATCH_SIZE` | 驗證階段一次處理幾筆資料，跟訓練用的 `BATCH_SIZE` 是分開的設定 | `32` | 一般不用特別調 |
| `VAL_INTERVAL` | 每隔幾步暫停一下，用驗證集檢查目前訓練得怎麼樣（不會影響模型本身） | `10000` | 想更常看到進度可以調小，但每次驗證都要花額外時間，調太小會拖慢整體訓練速度 |
| `SAVE_CHECKPOINT_INTERVAL` | 每隔幾步存一次檔到 Drive | `5000` | 前面「分段執行」討論過，想降低斷線損失可以調小，代價是存檔本身也會佔一點時間 |
| `NUM_WORKERS` | 用幾個平行的小幫手去讀取硬碟上的資料 | `8` | 通常不用動，除非明顯感覺讀資料是瓶頸 |

## 建議與待辦

- **開新的正式實驗記得改 `EXP_NAME`**：`EXP_NAME` 決定了 checkpoint 在 Drive 上存在哪個資料夾（`FluxAudio_checkpoints/{EXP_NAME}/`），訓練開始時程式會自動去該資料夾找有沒有舊 checkpoint、有的話就接著練下去。如果要跑一個新設定（換資料量、換超參數）卻沒改 `EXP_NAME`、還是沿用 `fluxaudio_s_50k`，就會載入舊的、已經練完 50000 步的 checkpoint 當起點，輕則什麼都不會訓練（因為已經達到 `NUM_ITERATIONS`），重則新舊資料/設定混在一起練出意義不明的模型。開新實驗前，先把設定區的 `EXP_NAME` 改成沒用過的新名字（例如 `fluxaudio_s_v2`）。
- **notebook 裡看不出目前訓練進度**：commit 前 cell output 會被清空，所以 GitHub 上完全看不到目前實際跑到第幾步、FAD/CLAP 多少分。已經另外開了 [`TRAINING_LOG.md`](./TRAINING_LOG.md) 這張純文字進度表，每次分段訓練告一段落，補一行「日期 / 實驗名稱 / 進度 / 備註」，之後回頭看或找人幫忙都不用重新解析整份 notebook。
- **生成音檔／checkpoint 目前完全依賴瀏覽器下載**：第七、八階段最後是用 `files.download()` 觸發瀏覽器下載視窗，沒有自動存回 Drive。建議產出真的要交、要留存的版本時，順手把 checkpoint 和音檔也複製一份到 Drive 的固定資料夾，不要只靠瀏覽器下載紀錄，比較不會因為找不到本機檔案而要重新生成一次。
- **如果常態需要一次訓練 7 小時以上**：免費版 Colab 閒置斷線／連線時數上限會是長期困擾，若這個專案還會持續訓練更大的模型或更多 iterations，值得評估升級 Colab Pro（背景執行、更長連線時數），會比一直手動分段省心。
- **改一份 notebook 的共用邏輯，記得檢查另一份要不要跟著改**：兩份 notebook 有幾格幾乎是複製貼上的（環境安裝、checkpoint symlink、評估工具安裝）。2026-07-19 抓到兩個 `flux_audio_mini_test.ipynb` 的真 bug——第八階段複製 CLAP checkpoint 前少了 `os.makedirs(exist_ok=True)`、修 `laion_clap` logging bug 的字串少寫一個反斜線（`"\t"` 被 Python 解讀成真的 tab，永遠比對不到檔案內容）——兩個都是 `flux_audio_formal.ipynb` 早就修過、但改良過程沒回頭同步到測試版留下的坑。之後修好一份，記得檢查另一份是不是也有同樣的舊寫法。
- **第八階段算 FAD 的 cell，偶爾會出現「跑完但沒印出分數」的假失敗**：現象是 cell 顯示執行完畢（有打勾、有花費時間），但畫面停在 `[Frechet Audio Distance] Loading 100 audio` 就沒有下文，後面查詢結果的 cell 也讀不到 `fad.csv`；到終端機直接重跑同一行 `python src/fad.py ...` 卻能正常印出分數。目前判斷是 Colab 顯示這個工具（`fadtk`）大量 `tqdm` 進度條輸出時的顯示問題，不是分數算錯。2026-07-23 已經把 `flux_audio_formal.ipynb` 這格改成用 Python 的 `subprocess.run(..., capture_output=True)` 明確接住輸出（跟之前修 mini_test 同一招），初步判斷應該能解決，但還沒有多次重現驗證，如果之後又遇到同樣的「找不到結果」，記得先確認是不是這個老問題，直接去終端機重跑同一個指令通常就能拿到正確分數。

## 評估工具

使用 [ICME26-ATTM-GC-Evaluation](https://github.com/ntu-musicailab/ICME26-ATTM-GC-Evaluation) 計算 FAD 與 CLAP score，notebook 第 8 階段會自動 clone 並安裝。

## 執行方式

先決定要跑哪一份：想快速驗證 pipeline 能不能跑通、改完 code 想確認沒壞掉，就跑 `flux_audio_mini_test.ipynb`；要正式訓練、產出可以拿去交的結果，才跑 `flux_audio_formal.ipynb`。

### `flux_audio_mini_test.ipynb`（煙霧測試，1000 步）

單次執行時間短（約 30-45 分鐘，若跳過資料處理改用共用快取則幾分鐘內就能到訓練），不需要分段：

1. 在 Google Colab 開啟，Runtime 選 GPU（L4）
2. 第二階段後面那格「【可選】快取捷徑」：想測完整 pipeline 就跳過不執行；只想快速驗證訓練/推論/評估就直接執行，會自動跳過第三、四、五階段（詳見上方「資料與 checkpoint」的說明）
3. 從上到下依序執行到底，中途不用中斷
4. 跑完想看結果的話，第七、八階段有試聽/下載 cell，但這只是煙霧測試，生成品質差是正常的，重點是確認每個階段都沒有噴錯

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
