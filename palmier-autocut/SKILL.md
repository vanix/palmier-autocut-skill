---
name: palmier-剪片
description: >
  Use ONLY when the user wants to edit a video using Palmier Pro MCP.
  Trigger keywords: 幫我剪片, 剪片, edit video, palmier pro, video editing.
  Starts an interactive questionnaire to collect requirements, then
  executes the full Palmier Pro MCP editing workflow: analyze, organize,
  timeline, silence removal, multi-track music, external whisper subtitle
  transcription, typo fix, export. Also supports searching and downloading
  copyright-free music from the web.
---

# Palmier Pro 自動剪片流程

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
| MCP 端點 | http://127.0.0.1:19789/mcp |

## 第一步：互動問卷

依序詢問使用者以下問題，**全部問完再開始執行**：

1. **影片主題是什麼？**
   （e.g., 太宰府旅遊 vlog、東京美食、婚禮紀錄…）

2. **素材在哪個資料夾？**
   （e.g., ~/Desktop/太宰府影片/）

3. **剪輯風格？**（預設流水帳）
   - A：流水帳 — 素材按時間順序全上，只清理靜音與明顯廢片段落
   - B：AI挑重點 — 逐支分析素材，只保留精彩片段（有對話、有情緒、有重點），剪成精華版
   若選 B，Phase 3 會先用 inspect_media 逐支分析，再決定保留哪些段落

4. **輸出影片要存在哪？**（預設：素材資料夾，即 Q2 的路徑）
   （e.g., ~/Desktop/太宰府影片/最終輸出.mp4）
   ⚠️ 若使用者接受預設，Agent 必須實際使用 Q2 的素材路徑，不可自行丟到 ~/Downloads/

5. **過程中處理的暫存檔要存在哪？**（預設：素材資料夾，即 Q2 的路徑）
   （e.g., ~/Desktop/太宰府影片/temp/）
   包含：提取的音訊 .wav、Whisper SRT、修正後的 SRT、frame JSON 等
   ⚠️ 同上，若接受預設則用 Q2 路徑下的 temp/ 子目錄，不可用 /tmp/

6. **影片比例與解析度？**（預設 16:9 1080p）
   - 可選：16:9, 9:16, 1:1, 4:3
   - 可選：720p, 1080p, 2K, 4K

7. **主要語言？**（預設 zh-TW）

8. **每個章節的音樂風格？**
   - 使用者提供現有音樂檔案，或要求從網路抓無版權音樂
   - 若要上網抓：詢問風格偏好（日系清新、輕快旅行、Lo-fi chill、傳統和風等）

9. **BGM 音量？**（預設 -20dB）

10. **輸出格式？**（預設 H.264 1080p，或 fcpxml 給外部剪輯軟體）

## 第二步：無版權音樂搜尋流程

若使用者要求上網抓音樂：

1. 用 `websearch` 搜尋無版權音樂：
   - 搜尋關鍵字：`"copyright free" "travel vlog" music download` + 風格關鍵字
   - 推薦來源：Pixabay Music（CC0）、Uppbeat、YouTube Audio Library

2. 用 `webfetch` 確認授權條款（CC0 或 free to use）

3. 用 `curl` 或 `python` 下載 MP3 到素材目錄

4. `import_media(source={path: "下載路徑/"})` 匯入專案

5. 告知使用者已加入哪些音樂及來源

## 第三步：完整執行 SOP

### Phase 1：初始化

```
1. 連線 MCP（claude mcp add --transport http palmier-pro http://127.0.0.1:19789/mcp）
2. new_project(name="主題名稱", aspectRatio="16:9", quality="1080p", fps=30)
3. import_media(source={path: "素材目錄/"})
   → 一次匯入整個資料夾及子目錄
```

### Phase 2：素材分析

```
4. get_media() → 取得所有素材清單（含 hasAudio, durationSeconds）
5. 判斷分類方式（可選，依需求執行）：
   - 使用者知道哪些是 A-roll → 直接指定
   - 需要系統判斷 → 對每支影片 call inspect_media(mediaRef=ID, maxFrames=3)
     回傳：畫面抽幀 + 語音轉錄 segments
     分類標準：轉錄 >15 字且有對話 → A-roll；無對話/極少 → B-roll
6. organize_media(moves=[{items: [ids], into: "A-roll"}, ...])
   - B-roll 分類後不上時間軸，僅保留在媒體庫備用
```

### Phase 3：精選素材（決定要留哪些、剪多長）

根據第 3 題的剪輯風格，執行不同策略：

**A：流水帳模式**
```
- 所有 A-roll 按時間順序全上時間軸
- 只做最小程度清理：
  a) 移除明顯廢段（鏡頭蓋、嚴重晃動、拍到地面）
  b) 不做內容篩選，保留完整敘事流
- 保留每支素材的入點調整（去開頭 1-2 秒調整鏡頭）
```

**B：AI挑重點模式**
```
- 逐段檢視素材，決定保留範圍：
  a) 看檔名時間戳 → 知道拍攝順序
  b) 對每支素材 call inspect_media(mediaRef=ID, maxFrames=5)
     → 看抽幀 + ASR 摘要，快速判斷這段在拍什麼
  c) 保留條件：
     - 有對話、有重點（導覽、美食、互動）
     - 風景過場（走路、搭車）若太長可只留 2-3 秒
     - 拍糊、鏡頭蓋、晃到無法看的 → 直接跳過
  d) 決定每支素材的入點（trim in）：
     e.g., VID_143000.mp4 前 5 秒都在調整鏡頭 → startFrame=150
- 可能只保留 30-60% 的素材，剪成精華版
```

### Phase 4：時間軸建構 — 上片 + 粗剪

```
8. 依檔名時間戳排序已選素材
9. 上片（二選一）：
   方法 A：add_clips（一次多支）
     - 必須計算累積 startFrame：每支的 startFrame = 前一支 endFrame
     - e.g., clip1(0→302), clip2(302→3649), clip3(3649→5149) ...
     - 若全部都設 startFrame=0 會全部重疊在 frame 0
     ⚠️ 不可加 `trackIndex`，否則音軌無法自動連結，影片會變成無聲
   方法 B：insert_clips（單支插入，推薦）
     - 先 add_clips 一支影片建立 track 0
     - 其餘用 insert_clips(atFrame=累積位置, trackIndex=0, entries=[{mediaRef, source=[startSec, endSec]}])
   兩種方法皆可，方法 B 更精確控制每支長度
10. 粗剪 — 移除頭尾多餘段落：
    split_clip(clipId, atFrame=frame)     # 從指定 frame 切開
    ripple_delete_ranges(ranges=[{         # 刪除不要的範圍
      trackIndex, startFrame, endFrame
    }])
    常見動作：
    - 每支素材開頭：移除調整鏡頭、等人的空白
    - 每支素材結尾：移除講完話後的尷尬停頓
    - 中間廢話：離題或無意義的段落直接剪掉
11. remove_silence()  ← 強制執行
    → 刪除句間 >1s 靜音，ripple close gaps
    → 粗剪後再做 silence removal，可避免剪到不該剪的對話間隙
    → 必須在字幕轉錄之前執行，因為它會改變 timeline frame 位置
    → 字幕的音訊必須從 silence-removed 後的 timeline 提取
```

### Phase 5：B-roll 覆疊（可選，預設跳過）

```
12. B-roll 不上時間軸，保留在媒體庫備用
13. 若使用者要求，再手動插入 B-roll：
    insert_clips(atFrame, entries=[{mediaRef}])
    set_clip_properties(clipIds=[], volume=0)    # 音軌靜音
    set_clip_properties(clipIds=[], opacity=0.85)
```

### Phase 6：背景音樂

```
14. get_transcript() → 取得逐字稿
15. AI 分析逐字稿，自動找出話題轉折點，定義章節 frame 範圍
16. 每章節放對應音樂到獨立 Audio Track
17. 音量 -20dB：set_clip_properties(clipIds=[], volume=0.1)
18. 2 秒淡入淡出：
    set_keyframes(clipId, property="volume",
      keyframes=[[0,0], [60,0.1], [end-60,0.1], [end,0]])
19. 音樂太短可追加 loop
20. ⚠️ BGM tail 裁切：
    - BGM clip 的 frames 常會超出影片總長，導致尾段黑畫面+音樂
    - 用 set_clip_properties(clipIds=[], durationFrames=章節長度)
      將每段 BGM 結尾裁齊對應章節的 endFrame
```

### Phase 7：字幕（外部 Whisper 轉錄 — 捨棄內建 add_captions）

⏰ **執行時機**：Phase 4 `remove_silence()` 之後執行。因為 silence 刪除會改變
timeline frame 位置，必須從 silence-removed 後的 timeline 提取音訊來轉錄，
否則字幕 frame 會對不上影片。

使用 `mlx_whisper`（Apple Silicon GPU 加速）或 `whisper`（CPU 備用）產出
word-level 時間戳 SRT。

```
20. extract audio（存放於 Q5 暫存目錄）：
    # 從 silence-removed 後的 timeline 輸出完整音訊
    ffmpeg -i <project_video> -vn -ar 16000 -ac 1 <TEMP_DIR>/audio.wav

    或直接從原始素材提取（若尚未 export 影片）：
    ffmpeg -i <原始影片檔案> -vn -ar 16000 -ac 1 <TEMP_DIR>/audio.wav
    ⚠️ <TEMP_DIR> = Q5 的路徑，不可寫死 /tmp/

21. 轉錄（二選一）：
    # 路徑 A：mlx_whisper（GPU 加速，推薦）
    mlx_whisper /tmp/audio.wav \
      --model mlx-community/whisper-large-v2-mlx \
      --language zh \
      --word-timestamps True \
      --output-format srt \
      --output-dir /tmp

    # 路徑 B：原始 openai-whisper（CPU 備用，無幻覺）
    whisper /tmp/audio.wav \
      --model large-v2 \
      --language zh \
      --word_timestamps True \
      --output_format srt \
      --output_dir /tmp

    ⚠️ 注意：large-v3-mlx 有嚴重重複幻覺（走走走、來來來），
       強烈建議使用 large-v2-mlx。若仍有少量幻覺，需手動清除。

22. 安裝（首次使用）：
    pip install mlx-whisper          # 路徑 A
    pip install openai-whisper       # 路徑 B

23. 清除幻覺段落（若有）：
    以 python 掃描 SRT，移除重複單字（「出出出」等）的區段

24. 修正錯字 — 使用 Python 批次取代（替換 SRT 文字內容）：
     依專案實際錯字定義修正表（ASR 同音字、近似音、專有名詞等）

25. SRT → JSON（frame 換算）：python3 腳本
    frame = round(seconds × fps)
    注意 endFrame 不可 <= startFrame，若相等則 +1

26. 匯入 timeline（一次匯入所有字幕，勿分批）：
    ⚠️ add_texts 若分批匯入，每批會自動建新軌道，導致字幕散落各軌。
       必須全部 entries 在單一次 add_texts 中處理。
    # 先檢查時間軸現有軌道內容，決定移除哪條舊字幕軌道
    manage_tracks(remove=[舊字幕軌道索引])
    # 每筆 entry 都帶完整樣式
    entries = [{
        "startFrame": ...,
        "endFrame": ...,
        "content": "...",
        "color": "#FFFFFF",           # 白字
        "backgroundColor": "#000000", # 黑底
        "fontSize": 48,
        "alignment": "center",
        "transform": {"centerX": 0.5, "centerY": 0.85}
    }]
    add_texts(entries=entries)        # 匯入（自動建 V2）
```

#### 字幕預設樣式與位置

匯入後的字幕 clip 屬性（預設樣式）：

| 屬性 | 值 | 說明 |
|:----:|:---:|:-----|
| `mediaType` | `text` | 純文字 clip |
| `color` | `#FFFFFF` | 白色字體 |
| `backgroundColor` | `#000000` | 黑色底色 |
| `fontSize` | `48` | 字體大小（canvas points） |
| `borderColor` | 無 | 不使用文字框線 |
| `transform.centerX` | `0.5` | 水平居中 |
| `transform.centerY` | `0.85` | 底部 15% （傳統字幕位置） |
| `transform.width` | 自動依文字長度計算 | 0-1 正規化寬度 |
| `transform.height` | 自動依文字長度計算（約 0.048） | 0-1 正規化高度 |

若要修改已匯入的字幕：
- 內容：`update_text(clipIds=[...], content="新文字")`
- 位置/大小：`set_clip_properties(clipIds=[...], transform={...})`
- 樣式：`add_texts` 時在 entry 中指定 `color`, `backgroundColor`, `fontSize`

### Phase 8：輸出

```
27. export_project(mode="video", codec="H.264", resolution="1080p", outputPath=<Q4路徑>)
     或 export_project(mode="fcpxml", outputPath=<Q4路徑>)
     → 匯出給 DaVinci Resolve / Final Cut Pro
     ⚠️ outputPath 必填 Q4 路徑，不可省略（預設會跑到 ~/Downloads/）
```

## 章節與音樂對應範例

| 章節 | 音樂風格 | 典型內容 |
|:----:|:--------:|:---------|
| Ch1 | 輕快旅行（Upbeat） | 到達車站、第一印象 |
| Ch2 | 日式傳統（Japanese Instrumental） | 老街散步、寺廟參拜 |
| Ch3 | 現代放鬆（Chill EDM） | 博物館、現代景點 |
| Ch4 | 溫馨收尾（Easy Travel） | 美食、購物、回程 |

## 錯字修正

外部 whisper SRT 產出後，用 Python 批次取代修正，再匯入 timeline。常見錯誤類型：

| 類型 | 範例 | 說明 |
|:----:|:----:|:----:|
| 同音字 | 在→再、因該→應該 | ASR 選錯同音字 |
| 近似音 | 馬雞→麻糬、皮孔→鼻孔 | 中文 ASR 極限 |
| 贅詞 | 出出出、走走走（刪除） | 幻覺重複，整段移除 |
| 簡繁 | 迴家→回家、擡頭→抬頭 | 模型偏好問題 |
| 專有名詞 | 人名、地名、品牌（需逐案修正） | ASR 不認識專有名詞 |

修正方式（匯入前）：
```python
# SRT 文字批次修正 — 依專案實際錯字定義
fixes = [
    (r'在', '再'),       # 同音字
    (r'因該', '應該'),
    (r'馬雞', '麻糬'),   # 近似音
    # 依專案實際錯字增加
]
for pattern, replacement in fixes:
    srt_content = re.sub(pattern, replacement, srt_content)
```
然後將修正後的 SRT 轉為 JSON（frame 換算），透過 `add_texts` 匯入 timeline。

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
| mlx_whisper / whisper CLI | 外部轉錄（取代 add_captions） |
| apply_color / apply_effect / inspect_color | 調色/特效 |
| generate_video / generate_image / generate_audio | AI 生成 |
| export_project | 輸出 |
| undo | 復原 |
