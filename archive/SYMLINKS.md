# Symlink 使用地圖

這個專案在好幾個地方都用到 symlink（捷徑），但目的不只一種。整理成三類，方便之後跟別人解釋。

**symlink 的本質**：一個「指標檔案」，本身幾乎不佔空間，打開它就等於直接打開它指向的真正目的地。跟 Windows 的捷徑、Google Drive 的「加入我的雲端硬碟」概念一樣。

---

## 目的一：省硬碟空間，不重複複製巨大的音檔

### 第四階段｜`prepare_jamendo_for_meanaudio.py`（baseline repo 提供）

把完整資料集切成 train/val/test 三組時，**不會真的把音檔複製三份**，而是在每組的 `audios/` 資料夾裡建立指回原始音檔的 symlink：

```python
symlink_path = output_audio_dir / f"{sample_id}{source_audio.suffix}"
if not symlink_path.exists():
    symlink_path.symlink_to(source_audio.resolve())
```

**為什麼**：原始資料集有 55701 首歌、上百 GB。如果 train/val/test 三組各自複製一份完整音檔，硬碟空間會直接乘三。用 symlink，三組各自的 `audios/` 資料夾看起來都有完整音檔可以讀，但實際上都指向同一份原始檔案。

---

## 目的二：讓 Colab 重開機後，本機路徑還能接回 Google Drive 上的資料

Colab 虛擬機是用完即丟的——`/content` 下的檔案，虛擬機一斷線重置就全部消失。這幾處都是「把本機的某個資料夾路徑，直接連到 Google Drive」，這樣不管虛擬機重開幾次，程式碼裡寫的本機路徑都能繼續接上 Drive 裡保存下來的東西。

### 第二階段末｜checkpoint 提前連結（`flux_audio_formal.ipynb` cell 11）

```python
link_path = '/content/ICME26-ATTM-GC-FluxAudio/exps/fluxaudio_s_50k'
target_path = '/content/drive/MyDrive/FluxAudio_checkpoints/fluxaudio_s_50k'
if not os.path.exists(link_path):
    os.symlink(target_path, link_path)
```

### 第六階段｜訓練前的 checkpoint 自動同步（`flux_audio_formal.ipynb` cell 30、`flux_audio_mini_test.ipynb` cell 26）

跟上面目的一樣，但版本更完整——多處理了「本地資料夾已經存在、但不是 symlink」的情況（例如上一次沒斷線、是本機真的存了資料），會先把內容搬進 Drive 再補建 symlink，不會覆蓋掉：

```python
if os.path.exists(local_dir) and not os.path.islink(local_dir):
    for f in os.listdir(local_dir):
        shutil.move(os.path.join(local_dir, f), os.path.join(drive_dir, f))
    os.rmdir(local_dir)
if not os.path.exists(local_dir):
    os.symlink(drive_dir, local_dir)
```

**為什麼這個最關鍵**：`train.py` 存 checkpoint 時，寫的路徑是 `exps/{EXP_ID}/xxx.pth`，程式自己完全不知道 Drive 的存在。有了這個 symlink，`exps/{EXP_ID}` 這個「看起來是本機的路徑」，寫進去的每一個位元組其實都直接落在 Drive 上。斷線重開，只要重新跑一次這個 symlink cell，訓練就能從中斷的地方接續，而不用每次都重頭訓練。

---

## 目的三：讓專案程式碼找得到外部套件

### 第六階段｜安裝 av-benchmark 評估工具（`flux_audio_mini_test.ipynb` cell 25）

```bash
!git clone https://github.com/hkchengrex/av-benchmark.git
%cd {PROJECT_DIR}
!ln -sf /content/av-benchmark/av_bench ./av_bench
```

`av-benchmark` 是另外從 GitHub clone 下來、裝在 `/content/av-benchmark` 的獨立套件，但 FluxAudio 專案的程式碼是用 `from av_bench.extract import extract` 這種方式匯入，預期 `av_bench` 就在專案資料夾底下。這個 symlink 負責把兩個各自獨立的資料夾「接」在一起。

> ⚠️ **這格也是我們之前踩到「資料夾無限巢狀」那個 bug 的源頭**——如果執行當下工作目錄不小心停在 `/content/av-benchmark` 而不是專案資料夾，`ln -sf` 會把捷徑建在 `av_bench` 自己裡面，變成自己連自己的迴圈。細節見這份筆記對應的 [debug 筆記](https://claude.ai/code/artifact/3afed384-5ca4-404c-ad6d-564e0a2974df)。

---

## 特殊情況：symlink 會「失效」，要留意的地方

### 清理階段（`flux_audio_formal.ipynb` cell 25）

第四階段建立的 `test/audios/` 裡放的是 symlink，指向原始音檔。但清理階段為了省空間會把原始音檔整個刪掉——**這樣一來，test 的 symlink 全部會變成失效的死連結**。所以清理之前，程式會先把 test 的音檔複製成「真正的檔案」備份到 `audios_real/`（評估階段要真的讀到音訊內容，死掉的 symlink 沒用）：

```python
# test 音檔目前只是 symlink，刪掉原始音檔前要先備份成真正的檔案
shutil.copy2(os.path.realpath(os.path.join(test_audio_dir, f)), dst)
```

train/val 的 `audios/` 就不特別處理，因為訓練跟驗證只在特徵萃取（第五階段）時讀過一次音檔，那時已經轉換成 npz 了，之後音檔本身失效也不影響。

### 第八階段（cell 43）

讀取 test 音檔時，優先找清理階段備份的 `audios_real/`，找不到才 fallback 回原本的 symlink 路徑（`audios/`）——對應到上面清理階段做的事。

---

## 小結：三種目的，一次看懂

| 目的 | 連的是什麼 → 什麼 | 為什麼要用 symlink 而不是複製 |
|---|---|---|
| 省空間 | split 的 `audios/` → 原始 Jamendo 音檔 | 避免三組資料各複製一份，省下數倍硬碟空間 |
| 跨 session 保存 | 本機 `exps/{EXP_ID}` → Google Drive | Colab 虛擬機重置後，本機路徑仍能接回 Drive 上留存的 checkpoint |
| 接外部套件 | 專案資料夾裡的 `av_bench` → clone 下來的 av-benchmark repo | 讓程式碼裡寫死的 import 路徑找得到實際檔案 |
