# Palmier AutoCut Skill

AI Agent 技能（Skill）——透過 [Palmier Pro MCP](https://palmier.pro) 自動化影片剪輯流程。
專為 OpenCode 設計。

## 功能

當使用者說出「幫我剪片、剪片、edit video、後製、剪輯、做影片」等關鍵字時自動觸發，執行完整後製流程：

| Phase | 內容 |
|:-----:|:------|
| 1. 初始化 | `new_project` + `import_media`（整資料夾） |
| 2. 素材分析 | `get_media` + `inspect_media` 自動分類 A-roll / B-roll |
| 3. 精選素材 | 流水帳全上／AI 挑重點，逐支分析決定保留段落 |
| 4. 時間軸建構 | `add_clips` 累積 startFrame → 粗剪（split + ripple）→ `remove_silence()` |
| 5. B-roll 覆疊 | 可選，疊新軌道或取代主軌 |
| 6. 背景音樂 | 逐章節分軌 → 音量 0.1 → 2s 淡入淡出 → 裁切 BGM tail |
| 7. 字幕轉錄 | 外部 `mlx_whisper` / `whisper` CLI → 除幻覺 → 錯字修正 → SRT→JSON → `add_texts` |
| 8. 輸出 | H.264 / H.265 / ProRes 或 FCPXML |

## 安裝

```bash
git clone https://github.com/vanix/palmier-autocut-skill.git ~/.config/opencode/skills/palmier-autocut
```

依賴套件（字幕轉錄用）：

```bash
pip install mlx-whisper    # Apple Silicon GPU，推薦
pip install openai-whisper # CPU 備用
```

## 使用方式

```
幫我剪片
```

系統會依序詢問 7 個問題（主題、素材路徑、剪輯風格、語言、解析度、音樂需求、輸出格式），全部回答完後自動執行。

## 互動問卷

1. **影片主題？** （e.g. 太宰府旅遊 vlog）
2. **素材資料夾？** （路徑衍生：輸出 + 暫存檔）
3. **剪輯風格？** A: 流水帳 / B: AI 挑重點
4. **主要語言？** （預設 zh-TW）
5. **解析度？** （16:9 / 9:16 / 1:1 / 4:3，720p–4K）
6. **音樂需求？** （自備或抓 CC0，風格偏好，音量 -20dB）
7. **輸出編碼？** （H.264 / H.265 / ProRes / fcpxml）

## 關鍵原則

- **先 silence → 後 BGM**：BGM 必須在 `remove_silence()` 完成後才加入，否則 silence ripple 會把 BGM 切裂成幾十段
- **BGM 定位後不可 ripple**：修改請先移除 BGM → 修改 → 重加 BGM
- **字幕只能用外部 Whisper**：禁止 `add_captions()` 和 `get_transcript()` 拼湊
- **音訊必須從 silence-removed 後的 timeline 提取**：從原始素材抽會 frame 偏移
- **每步操作前先 `get_timeline()`**：確保 clip id / frame 都是最新的

## 字幕規格

| 項目 | 值 |
|:----:|:---:|
| 轉錄引擎 | `mlx_whisper` large-v2-mlx（GPU）／ `whisper` large-v2（CPU） |
| 字體顏色 | `#FFFFFF` |
| 背景顏色 | `#000000` |
| 字體大小 | 48 pt |
| 位置 | 底部置中（centerY: 0.85） |
| 語言 | zh-TW |

## 無版權音樂

搜尋 CC0 音樂：Pixabay Music、Uppbeat、YouTube Audio Library。用 `websearch` + `webfetch` 確認授權後下載 MP3，`import_media` 匯入專案。

## 錯字修正

Whisper 同音字誤判（在→再、因該→應該等），SKILL 內附 Python regex 批次修正腳本 + SRT→JSON 轉換腳本。


## 經驗教訓

10 條實戰歸納，含 BGM 碎裂、字幕軌清除、Whisper 幻覺、BGM tail 裁切等 — 詳見 SKILL.md。

## 授權

MIT License
