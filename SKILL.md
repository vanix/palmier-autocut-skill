---
name: palmier-autocut
description: >
  Use ONLY when the user wants to edit a video using Palmier Pro MCP.
  Trigger keywords: 幫我剪片, 剪片, edit video, palmier pro, video editing,
  後製, 剪輯, 做影片, 影片編輯, video post-production.
  Starts an interactive questionnaire to collect requirements, then
  executes the full Palmier Pro MCP editing workflow: analyze, organize,
  timeline, silence removal, multi-track music, external whisper subtitle
  transcription, typo fix, export. Also supports searching and downloading
  copyright-free music from the web.
---

# Palmier Pro 自動剪片流程

## 鐵則（違反會產出爛字幕，使用者會生氣）

| # | 規則 | 說明 |
|:-:|:----|:-----|
| 1 | ❌ 禁止使用 `add_captions()` | 這是 Palmier 內建字幕引擎，斷句亂、時間軸不準 |
| 2 | ❌ 禁止使用 `get_transcript()` 拼湊字幕 | transcript 的 frame 資料不可直接拿來組字幕 clip |
| 3 | ❌ 禁止用 Python 手動組字串取代 Whisper | 不管理由是什麼，不執行走捷徑就是錯的 |
| 4 | ✅ **唯一合法流程** | 抽音訊 → Whisper CLI 轉錄 SRT → 修正錯字 → `add_texts` 匯入 |
| 5 | ⏰ 跳過 Whisper = 直接跳過 Phase 7 | 若時間不夠寧可先輸出無字幕版，也不准用捷徑 |
| 6 | ⚠️ **BGM 必須在 `remove_silence` 之後加入** | 若先放 BGM，silence 的 ripple 會把 BGM 切裂成幾十段碎片，聽感斷斷續續 |
| 7 | ⚠️ **任何步驟失敗 → 先 retry 一次，再 fail 則告知使用者並詢問下一步** | 不要靜靜卡住或假設成功 |

## 預設值

| 項目 | 預設值 |
|:----:|:-------:|
| 解析度 | 1920×1080, 30fps, 16:9 |
| 輸出格式 | MP4 H.264 1080p |
| BGM 音量 | -20dB（volume: 0.1） |
| 淡入淡出 | 2 秒 |
| 強制靜音刪除 | `remove_silence()` |
| 字幕語言 | zh-TW |
| 字幕模型 | `mlx_whisper` large-v2-mlx（GPU 加速） |
| 字幕格式 | 白字 `#FFFFFF` + 黑底 `#000000`，無邊框，底部置中（centerY: 0.85） |
| B-roll | 不上時間軸（可選） |

## 第一步：互動問卷

依序詢問使用者以下問題，**全部問完再開始執行**：

1. **影片主題是什麼？**
   （e.g., 太宰府旅遊 vlog、東京美食、婚禮紀錄…）

2. **素材在哪個資料夾？**
   （e.g., `~/Desktop/太宰府影片/`）
   ⚠️ 後續路徑自動由此衍生：
   - 輸出影片 → `{素材資料夾}/{主題名稱}_精華版.mp4`
   - 暫存檔（音訊 .wav、Whisper SRT、JSON 等）→ `{素材資料夾}/temp/`
   - Agent **必須**使用素材路徑，不可自行丟到 `~/Downloads/`

3. **剪輯風格？**（預設流水帳）
   - A：流水帳 — 素材按時間順序全上，只清理靜音與明顯廢片段落
   - B：AI挑重點 — 逐支分析素材，只保留精彩片段（有對話、有情緒、有重點），剪成精華版
     若選 B，Phase 3 會先用 inspect_media 逐支分析，再決定保留哪些段落

4. **主要語言？**（預設 zh-TW）

5. **影片比例與解析度？**（預設 16:9 1080p）
   - 可選：16:9, 9:16, 1:1, 4:3
   - 可選：720p, 1080p, 2K, 4K

6. **音樂需求？**（僅選 B「AI挑重點」時詢問；A「流水帳」跳過）
   - 使用者提供現有音樂檔案，或要求從網路抓無版權音樂
   - 若要上網抓：詢問風格偏好（日系清新、輕快旅行、Lo-fi chill、傳統和風等）
   - BGM 音量預設 -20dB，可在此調整
   - BGM 預設 2 秒淡入淡出

7. **輸出編碼？**（預設 H.264，可選 H.265 / ProRes 給本地保留，或 fcpxml 給外部剪輯軟體）

## 第二步：無版權音樂搜尋流程

若使用者要求上網抓音樂：

1. 用 `websearch` 搜尋無版權音樂：
   - 搜尋關鍵字：`"copyright free" "travel vlog" music download` + 風格關鍵字
   - 推薦來源：Pixabay Music（CC0）、Uppbeat、YouTube Audio Library

2. 用 `webfetch` 確認授權條款（CC0 或 free to use）

3. 用 `curl -L` 或 `python3 -c "import urllib.request; urllib.request.urlretrieve(...)"` 下載 MP3 到素材目錄

4. `import_media(source={path: "下載路徑/"})` 匯入專案

5. 告知使用者已加入哪些音樂及來源

## 第三步：完整執行 SOP

### ⚠️ 通用錯誤處理規則

- 每個步驟若失敗，**先 retry 一次**（如網路抖動、檔案未就緒）
- retry 仍失敗 → 明確告知使用者錯誤訊息，並詢問：略過此步驟／改用替代方案／終止
- 例如：`mlx_whisper` 安裝失敗 → 提示改 `openai-whisper`；`import_media` 單檔失敗 → 跳過該檔繼續
- `get_timeline()` 是每次操作前的 sanity check — 拿到最新的 clip id / frame 再動作

---

### Phase 1：初始化（Steps 1–3）

```
Step 1. new_project(name="主題名稱", aspectRatio="16:9", quality="1080p", fps=30)
        ⚠️ 若專案已存在 → 問使用者是否 open_project 取代
Step 2. import_media(source={path: "素材目錄/"})
        → 一次匯入整個資料夾及子目錄
        ⚠️ 若部分檔案格式不支援 → import_media 會跳過，不影響已匯入的素材
Step 3. get_media() → 確認素材已到位（必要時 retry）
```

### Phase 2：素材分析（Steps 4–6）

```
Step 4. get_media() → 取得所有素材清單（含 hasAudio, durationSeconds）
Step 5. 判斷分類方式（可選，依需求執行）：
        - 使用者知道哪些是 A-roll → 直接指定
        - 需要系統判斷 → 對每支影片 call inspect_media(mediaRef=ID, maxFrames=3)
          回傳：畫面抽幀 + 語音轉錄 segments
          分類標準：轉錄 >15 字且有對話 → A-roll；無對話/極少 → B-roll
Step 6. organize_media(moves=[{items: [ids], into: "A-roll"}, ...])
        - B-roll 分類後不上時間軸，僅保留在媒體庫備用
```

### Phase 3：精選素材 — 決定要留哪些、剪多長（Step 7）

根據第 3 題的剪輯風格，執行不同策略：

**A：流水帳模式**
```
Step 7a. 所有 A-roll 按時間順序全上時間軸
         - 只做最小程度清理：
           a) 移除明顯廢段（鏡頭蓋、嚴重晃動、拍到地面）
           b) 不做內容篩選，保留完整敘事流
         - 保留每支素材的入點調整（去開頭 1-2 秒調整鏡頭）
```

**B：AI挑重點模式**
```
Step 7b. 逐段檢視素材，決定保留範圍：
         a) 看檔名時間戳 → 知道拍攝順序
         b) 對每支素材 call inspect_media(mediaRef=ID, maxFrames=5)
            → 看抽幀 + ASR 摘要，快速判斷這段在拍什麼
         c) 保留條件：
            - 有對話、有重點（導覽、美食、互動）
            - 風景過場（走路、搭車）若太長可只留 2-3 秒
            - 拍糊、鏡頭蓋、晃到無法看的 → 直接跳過
         d) 決定每支素材的入點（trim in）：
            e.g., VID_143000.mp4 前 5 秒都在調整鏡頭
            → 上片時用 source=[5, endSec] 跳過前 5 秒（素材時間軸）
            ⚠️ 不是 timeline 的 startFrame（那是放的位置，不是來源裁切）
         - 可能只保留 30-60% 的素材，剪成精華版
```

### Phase 4：時間軸建構 — 上片 + 粗剪（Steps 8–12）

```
Step 8. 依檔名時間戳排序已選素材

Step 9. 上片 — 用 add_clips 一次多支（強烈推薦）
        ⚠️ 不可用 insert_clips（會 ripple 位移後續所有片段，導致 BGM 碎裂、字幕偏移）
        ⚠️ 不可加 trackIndex，否則音軌無法自動連結，影片會變成無聲
        - 必須計算累積 startFrame：每支的 startFrame = 前一支 endFrame
        - e.g., clip1(0→302), clip2(302→3649), clip3(3649→5149) ...
        - 長度指定二選一（互斥）：
          a) source=[startSec, endSec] — 素材時間軸，適合選段
          b) endFrame — 時間軸 frame，適合精確控制

Step 10. 粗剪 — 移除頭尾多餘段落：
         # 先從指定 frame 切開（二選一）
         # 方式 A：已知 clip ID
         split_clips(splits=[{clipId: "<clipId>", atFrame: <frame>}])
         # 方式 B：只知道 track 和 frame
         split_clips(trackIndex=<trackIndex>, frames=[<frame1>, <frame2>])

         # 刪除不要的範圍（多段可一次傳）
         ripple_delete_ranges(
           trackIndex=<trackIndex>,
           ranges=[[<startFrame>, <endFrame>], ...],
           units="frames"
         )
         # 若 sync-locked track 擋住 → 用 ignoreSyncLockedTracks 放行

         常見動作：
         - 每支素材開頭：移除調整鏡頭、等人的空白
         - 每支素材結尾：移除講完話後的尷尬停頓
         - 中間廢話：離題或無意義的段落直接剪掉

Step 11.  get_timeline() → 確認 clip 位置正確

Step 12. remove_silence()  ← 強制執行
         → 刪除句間 >1s 靜音，ripple close gaps
         → 粗剪後再做 silence removal，可避免剪到不該剪的對話間隙
         → 必須在字幕轉錄之前執行，因為它會改變 timeline frame 位置
         → 字幕的音訊必須從 silence-removed 後的 timeline 提取
         → ⚠️ BGM 此時必須「尚未加入」。若 BGM 已存在，silence 的 ripple
            會把 BGM clip 在每個靜音區間處切裂，BGM 變成幾十段碎片
```

### Phase 5：B-roll 覆疊（Steps 13–16，預設跳過）

```
Step 13. 詢問使用者是否需要加 B-roll
         - 不需要 → 跳過此 Phase
         - 需要 → 繼續以下步驟

Step 14. 從媒體庫選擇 B-roll 素材（已經分類到 B-roll 資料夾的素材）
         inspect_media(mediaRef=ID, maxFrames=3) 快速預覽內容

Step 15. 插入 B-roll（兩條路徑）：
         路徑 A — 覆疊在現有影片上方（新 Video Track）：
           add_clips(entries=[{mediaRef: "<id>", startFrame: <timeline_pos>, ...}])
           # 不加 trackIndex 會自動建新軌道疊在上面
           
         路徑 B — 取代主軌道片段（換鏡頭）：
           先 split_clips 切開主軌道 → remove_clips 移除該段 → add_clips 補上 B-roll
           
Step 16. B-roll 音軌靜音 + 透明度調整：
         set_clip_properties(clipIds=[], volume=0)
         set_clip_properties(clipIds=[], opacity=0.85)
```

### Phase 6：背景音樂（Steps 17–24）

⚠️ **此時 `remove_silence()` 已完成，時間軸不再變動，才加入 BGM。**

若先放 BGM 再做 remove_silence，ripple 會把 BGM clip 在每個靜音區間處
切裂成碎片（一段 122 秒的 BGM_coast 可能被切成 40+ 段），聽感上 BGM
會不斷被中斷跳接。故嚴格遵守：**先 silence → 後 BGM**。

```
Step 17. get_transcript() → 取得逐字稿
         ⚠️ 此處僅用於分析章節轉折，不可用來拼湊字幕
  
Step 18. AI 分析逐字稿，自動找出話題轉折點，定義章節 frame 範圍

Step 19. 每章節放對應音樂到獨立 Audio Track
         add_clips(entries=[{mediaRef: "<BGM_id>", startFrame: <frame>}])

Step 20. 音量 -20dB：
         set_clip_properties(clipIds=[], volume=0.1)

Step 21. 2 秒淡入淡出（以 30fps 為例，60 frames = 2 秒）：
         set_keyframes(clipId="<BGM_clipId>", property="volume",
           keyframes=[[0, 0], [60, 0.1], [end-60, 0.1], [end, 0]])

Step 22. 音樂太短 → 將同一首歌重複加多次（loop）：
         用同一個 mediaRef 連續 add_clips，每段 startFrame 接續前一段的 endFrame
         e.g., 歌曲 30 秒（900 frames），章節要 75 秒（2250 frames）
         → add_clips 三筆：0→900, 900→1800, 1800→2250

Step 23. BGM tail 裁切 — 避免尾段黑畫面+音樂：
         # get_timeline() 的 totalFrames = 影片實際長度
         # BGM clip 的 frames 常超出此值，須手動裁切
         # clip 格式：{start, end, id, ...}，end 為 exclusive
         tl = get_timeline()
         total = tl["totalFrames"]
         for t in tl["tracks"]:
           for c in t.get("clips", []):
             if c["end"] > total:
               set_clip_properties(
                 clipIds=[c["id"]],
                 durationFrames=total - c["start"]
               )

Step 24. BGM 一旦定位後，不可再對 timeline 執行任何 ripple 操作
         （remove_silence、ripple_delete_ranges、insert_clips），
         否則 BGM 必定碎裂。若要修改已含 BGM 的時間軸：
         a) 先 remove_clips(clipIds=["<BGM_clipId>"]) 移除 BGM
         b) 執行修改
         c) 重新加入 BGM（回到 Step 19）
```

### Phase 7：字幕（Steps 25–32，強制外部 Whisper — 禁止使用 Palmier 內建工具）

⚠️ **鐵則：字幕來源只能是 `mlx_whisper` 或 `whisper` CLI 的 SRT 輸出。**
- ❌ 禁止使用 `add_captions()` — 內建字幕引擎
- ❌ 禁止使用 `get_transcript()` — 它的 frame 資料不可拿來組字幕
- ❌ 禁止用 Python 從 transcript 資料拼湊字幕取代 Whisper
- ✅ 唯一合法流程：抽音訊 → Whisper CLI 轉錄 → 修正錯字 → SRT→JSON → `add_texts` 匯入

若不遵守此規則，產出的字幕會：句子過長、斷點不自然、時間軸錯位。

⏰ **執行時機**：Phase 4 `remove_silence()` 之後執行。因為 silence 刪除會改變
timeline frame 位置，必須從 silence-removed 後的 timeline 提取音訊來轉錄，
否則字幕 frame 會對不上影片。

使用 `mlx_whisper`（Apple Silicon GPU 加速）或 `openai-whisper`（CPU 備用）產出
word-level 時間戳 SRT。

```
Step 25. 確認 ffmpeg 可用：
         which ffmpeg || which ffprobe || (brew install ffmpeg)

Step 26. 從 silence-removed 後的 timeline 提取音訊：
         # export 影片是非同步的（背景跑），需要輪詢等待完成
         temp_video = "<TEMP_DIR>/_temp_audio_source.mp4"
         export_project(
           mode="video", codec="H.264", resolution="720p",
           outputPath=temp_video
         )
         # 輪詢等待 export 完成（檔案出現且大小不再變動）
         import time, os
         while not os.path.exists(temp_video) or os.path.getsize(temp_video) < 1024:
             time.sleep(2)
         prev_size = -1
         while prev_size != os.path.getsize(temp_video):
             prev_size = os.path.getsize(temp_video)
             time.sleep(3)
         
         # 從完成的臨時檔抽音訊
         ffmpeg -y -i "<TEMP_DIR>/_temp_audio_source.mp4" \
           -vn -ar 16000 -ac 1 "<TEMP_DIR>/audio.wav"
         
         # 清理臨時影片
         os.remove(temp_video)
         
         ⚠️ <TEMP_DIR> = Q2 的素材路徑 + /temp/
         ⚠️ 不可從原始素材抽音訊（未反映 silence removal，frame 會偏移）

Step 27. 安裝 Whisper（首次使用，檢查 Python 環境）：
         # 先確認 pip 在哪個環境
         which python3 && python3 -m pip --version
         
         # 路徑 A：mlx_whisper（Apple Silicon GPU 加速，推薦）
         python3 -m pip install mlx-whisper
         # 若失敗 → 改用路徑 B

         # 路徑 B：openai-whisper（CPU 備用）
         python3 -m pip install openai-whisper
         # 若仍失敗 → 告知使用者安裝失敗，問是否跳過字幕

Step 28. 轉錄（二選一）：
         # 路徑 A：mlx_whisper（GPU 加速，推薦）
         mlx_whisper "<TEMP_DIR>/audio.wav" \
           --model mlx-community/whisper-large-v2-mlx \
           --language zh \
           --word-timestamps True \
           --output-format srt \
           --output-dir "<TEMP_DIR>"

         # 路徑 B：openai-whisper（CPU 備用）
         whisper "<TEMP_DIR>/audio.wav" \
           --model large-v2 \
           --language zh \
           --word_timestamps True \
           --output_format srt \
           --output_dir "<TEMP_DIR>"

         ⚠️ large-v3-mlx 有嚴重重複幻覺（走走走、來來來），
            強烈建議使用 large-v2-mlx。若仍有少量幻覺，需手動清除。
         ⚠️ 若轉錄失敗 → 問使用者是否降級模型（small/tiny）或跳過字幕

Step 29. 清除幻覺段落（若有）：
         以 python 掃描 SRT，移除：
         - 重複單字（「出出出」、「走走走」等）
         - 同一詞句連續重複 >3 次（如「要起來了」×28）— 整段移除
         幻覺特徵：相同文字以固定間隔（如 2 秒）重複出現，且次數異常

Step 30. 修正錯字 — 使用 Python 批次取代（替換 SRT 文字內容）：
         依專案實際錯字定義修正表（ASR 同音字、近似音、專有名詞等）
         （錯字表見下方章節）

Step 31. SRT → JSON（frame 換算）— 使用下方提供的 Python 腳本：
         python3 "<TEMP_DIR>/srt_to_palmier.py" \
           "<TEMP_DIR>/audio.srt" \
           --fps 30 \
           --output "<TEMP_DIR>/subtitles.json"

Step 32. 匯入 timeline（一次匯入所有字幕，勿分批）：
         ⚠️ add_texts 若分批匯入，每批會自動建新軌道，導致字幕散落各軌。
            必須全部 entries 在單一次 add_texts 中處理。
         ⚠️ 若要重新上字幕（如追加片段後），先 get_timeline() 找出舊字幕軌道，
            再用 manage_tracks(remove=[舊字幕軌道索引]) 移除，避免疊出新軌道。
            範例：
            tl = get_timeline()
            subtitle_track_idx = None
            for t in tl["tracks"]:
                if t.get("type") == "video" and any(
                    c.get("mediaType") == "text" for c in t.get("clips", [])
                ):
                    subtitle_track_idx = t["index"]
                    break
            if subtitle_track_idx is not None:
                manage_tracks(remove=[subtitle_track_idx])
         
         # 載入 JSON 後一次匯入
         add_texts(entries=json.load(open("<TEMP_DIR>/subtitles.json")))
```

### Phase 8：輸出（Step 33）

```
Step 33. export_project(
           mode="video", codec="H.264", resolution="1080p",
           outputPath="<素材目錄>/<主題名稱>_精華版.mp4"
         )
         或
         export_project(
           mode="fcpxml",
           outputPath="<素材目錄>/<主題名稱>_精華版.fcpxml"
         )
         → fcpxml 可匯入 DaVinci Resolve / Final Cut Pro
         ⚠️ outputPath 必填素材目錄路徑，不可省略（預設會跑到 ~/Downloads/）
         ⚠️ 若 export 失敗 → 檢查磁碟空間、outputPath 是否可寫、codec 是否支援
```

## SRT→JSON 轉換腳本

建立 `srt_to_palmier.py` 於暫存目錄（`{素材資料夾}/temp/`）：

```python
#!/usr/bin/env python3
"""Convert Whisper SRT (word-level timestamps) to Palmier add_texts JSON.

Usage:
    python3 srt_to_palmier.py <input.srt> --fps 30 --output subtitles.json
"""
import argparse
import json
import re
import sys
from pathlib import Path


SRT_TIME_RE = re.compile(
    r"(\d{2}):(\d{2}):(\d{2})[,.](\d{3})"
)


def srt_time_to_seconds(tc: str) -> float:
    m = SRT_TIME_RE.match(tc)
    if not m:
        return 0.0
    h, mi, s, ms = int(m[1]), int(m[2]), int(m[3]), int(m[4])
    return h * 3600 + mi * 60 + s + ms / 1000


def parse_srt(path: str) -> list[dict]:
    """Parse SRT into list of {index, start, end, text}."""
    blocks = []
    current = {}
    text_lines = []

    for line in Path(path).read_text(encoding="utf-8").splitlines():
        line = line.strip()
        if not line:
            if current:
                current["text"] = " ".join(text_lines).strip()
                if current["text"]:
                    blocks.append(current)
                current = {}
                text_lines = []
            continue
        if " --> " in line:
            parts = line.split(" --> ")
            current["start"] = srt_time_to_seconds(parts[0])
            current["end"] = srt_time_to_seconds(parts[1])
        elif line.isdigit() and not current:
            current["index"] = int(line)
        else:
            text_lines.append(line)

    if current and text_lines:
        current["text"] = " ".join(text_lines).strip()
        if current["text"]:
            blocks.append(current)

    return blocks


def blocks_to_entries(blocks: list[dict], fps: int) -> list[dict]:
    """Convert SRT blocks to Palmier add_texts entries."""
    entries = []
    for b in blocks:
        start_frame = round(b["start"] * fps)
        end_frame = round(b["end"] * fps)
        if end_frame <= start_frame:
            end_frame = start_frame + 1
        entries.append({
            "startFrame": start_frame,
            "endFrame": end_frame,
            "content": b["text"],
            "color": "#FFFFFF",
            "backgroundColor": "#000000",
            "fontSize": 48,
            "alignment": "center",
            "transform": {"centerX": 0.5, "centerY": 0.85},
        })
    return entries


def main():
    parser = argparse.ArgumentParser(description="SRT to Palmier JSON")
    parser.add_argument("input", help="Path to input SRT file")
    parser.add_argument("--fps", type=int, default=30, help="Timeline FPS")
    parser.add_argument("--output", "-o", default="subtitles.json",
                        help="Output JSON path")
    args = parser.parse_args()

    if not Path(args.input).exists():
        print(f"Error: {args.input} not found", file=sys.stderr)
        sys.exit(1)

    blocks = parse_srt(args.input)
    if not blocks:
        print("Warning: no subtitle blocks found", file=sys.stderr)
        sys.exit(0)

    entries = blocks_to_entries(blocks, args.fps)
    Path(args.output).write_text(
        json.dumps(entries, ensure_ascii=False, indent=2),
        encoding="utf-8",
    )
    print(f"Converted {len(entries)} entries → {args.output}")
    print(f"  FPS: {args.fps}")
    print(f"  First frame: {entries[0]['startFrame']}")
    print(f"  Last frame:  {entries[-1]['endFrame']}")


if __name__ == "__main__":
    main()
```

## 章節與音樂對應範例

| 章節 | 音樂風格 | 典型內容 |
|:----:|:--------:|:---------|
| Ch1 | 輕快旅行（Upbeat） | 到達車站、第一印象 |
| Ch2 | 日式傳統（Japanese Instrumental） | 老街散步、寺廟參拜 |
| Ch3 | 現代放鬆（Chill EDM） | 博物館、現代景點 |
| Ch4 | 溫馨收尾（Easy Travel） | 美食、購物、回程 |

## 錯字修正表（SRT 轉 JSON 前執行）

外部 whisper SRT 產出後，用 Python 批次取代修正，再匯入 timeline。常見錯誤類型：

| 類型 | 範例 | 說明 |
|:----:|:----:|:----:|
| 同音字 | 在→再、因該→應該 | ASR 選錯同音字 |
| 近似音 | 馬雞→麻糬、皮孔→鼻孔 | 中文 ASR 極限 |
| 贅詞 | 出出出、走走走（刪除） | 幻覺重複，整段移除 |
| 簡繁 | 迴家→回家、擡頭→抬頭 | 模型偏好問題 |
| 專有名詞 | 人名、地名、品牌（需逐案修正） | ASR 不認識專有名詞 |

修正方式（匯入前，直接在 SRT 檔案上操作）：

```python
import re
from pathlib import Path

srt_path = "<TEMP_DIR>/audio.srt"
content = Path(srt_path).read_text(encoding="utf-8")

fixes = [
    (r'\b在(?=次|度|見|說|這裡|那裡)', '再'),  # 前後文判斷
    (r'因該', '應該'),
    (r'馬雞', '麻糬'),
    (r'皮孔', '鼻孔'),
    # 依專案實際錯字增加
]

for pattern, replacement in fixes:
    content = re.sub(pattern, replacement, content)

Path(srt_path).write_text(content, encoding="utf-8")
print("Typo fixes applied to", srt_path)
```

## 經驗教訓（實戰累積）

> ⚠️ 以下教訓已整合至上方各 Phase 的 SOP 中，此處為快速回顧。

以下為實際剪片專案中歸納的注意事項：

| # | 教訓 | 說明 |
|:-:|:----|:-----|
| 1 | `add_clips` 比 `insert_clips` 穩定 | insert 會 ripple 位移後續所有片段，造成 BGM 碎裂、字幕偏移。一律用 add_clips 計算 startFrame 累積位置 |
| 2 | Whisper 幻覺必清除 | large-v2-mlx 在音樂/環境音段落會重複同一詞句（如「要起來了」×28），需以 Python 掃描移除連續重複 >3 次的區段 |
| 3 | 追加字幕前先清舊軌道 | 第二次 add_texts 若沒先 `manage_tracks(remove=[舊字幕軌道索引])`，會疊出 V3/V4 多軌。正確做法：`get_timeline()` 找出 mediaType=text 的 clip 所在 track → `manage_tracks(remove=[index])` |
| 4 | BGM 尾段必裁齊 | `get_timeline()` 的 totalFrames 是影片實際長度。BGM clip 常超出總長，需用 `set_clip_properties(durationFrames=...)` 裁切，否則尾段黑畫面 |
| 5 | remove_silence 會改 timeline frame | silence removal 會刪除靜音並 ripple 關閉間隙，所有後續 frame 位置都會位移。務必在字幕轉錄「前」執行，否則字幕時間軸全錯 |
| 6 | source vs endFrame 精度 | `source=[startSec, endSec]` 適合素材時間軸選段；`endFrame` 適合 timeline frame 精確控制。兩者互斥 |
| 7 | ⭐ BGM 必須在 silence 之後加入 | `remove_silence` 的 ripple 會把所有音軌一起位移切裂。若 BGM 已存在，會變成幾十段碎片。**先 silence → 後 BGM 才是正確順序** |
| 8 | BGM 定位後不可再 ripple | 一旦 BGM 定位完成，任何會 ripple 時間軸的操作（remove_silence、ripple_delete_ranges、insert_clips）都會碎裂 BGM。修改請先移除 BGM → 修改 → 重加 BGM |
| 9 | 音訊必須從 silence-removed 後的 timeline 提取 | 從原始素材抽音訊做 Whisper 轉錄，frame 對不上已 ripple 的時間軸。務必 export 臨時影片再抽音訊 |
| 10 | 每一步操作前先 get_timeline() | 確保 clip id、frame 位置、track 索引都是最新的，避免對到舊 id 造成操作失敗 |

## MCP 工具速查

| 工具 | 用途 |
|:----:|:----:|
| get_projects / new_project / open_project | 專案管理 |
| get_media / import_media / organize_media | 媒體管理 |
| inspect_media / search_media | 素材分析 |
| get_timeline / inspect_timeline | 時間軸讀取 |
| add_clips / insert_clips / remove_clips / move_clips | 片段操作 |
| split_clips / ripple_delete_ranges | 剪輯 |
| manage_tracks | 軌道管理 |
| set_clip_properties / set_keyframes | 屬性/動畫 |
| detect_beats / denoise_audio | 音訊處理 |
| remove_silence | 靜音刪除 |
| get_transcript / update_text | 逐字稿 / 字幕修正 |
| add_texts | 字幕／標題文字匯入 |
| `mlx_whisper`（GPU） / `whisper`（CPU 備用） | 外部轉錄（取代 add_captions） |
| apply_color / apply_effect / inspect_color | 調色/特效 |
| generate_video / generate_image / generate_audio | AI 生成 |
| export_project | 輸出 |
| undo | 復原 |
