# Palmier AutoCut Skill

OpenCode 專用技能（Skill）——透過 Palmier Pro MCP 自動化影片剪輯流程。

## 概述

這是一個 OpenCode Agent Skill，當使用者說出「幫我剪片」、「剪片」、「edit video」等關鍵字時自動觸發。它將完整的影片後製流程整合為一套互動問卷 + 自動執行 SOP，涵蓋：

- 素材匯入與分類（A-roll / B-roll）
- 粗剪（trim、split、silence removal）
- 外部 Whisper 語音轉錄（`mlx_whisper` GPU 加速）
- 錯字修正與字幕定位
- 多章節背景音樂配樂（可上網搜尋 CC0 音樂）
- 最終輸出（H.264 或 FCPXML）

## 前置需求

- macOS（Apple Silicon 為佳，支援 GPU 加速）
- [Palmier Pro](https://palmier.pro)（影片剪輯軟體，需啟用 MCP Server）
- [OpenCode](https://opencode.ai)（CLI 工具）
- Python 3.10+
- [mlx-whisper](https://github.com/ml-explore/mlx-whisper)（選擇性，GPU 轉錄用）
- FFmpeg

## 安裝

將 `SKILL.md` 放入 OpenCode 的 skills 目錄：

```bash
mkdir -p ~/.config/opencode/skills/palmier-剪片
cp SKILL.md ~/.config/opencode/skills/palmier-剪片/
```

若使用 GPU 轉錄，安裝相依套件：

```bash
pip install mlx-whisper openai-whisper
```

## 使用方式

在 OpenCode CLI 中輸入：

```
幫我剪片
```

或直接描述需求：

```
剪片，素材在 ~/Desktop/我的影片/，輸出 1080p mp4
```

系統會先以互動問卷收集以下資訊：

1. 影片主題
2. 素材資料夾路徑
3. 輸出影片路徑（預設：素材資料夾）
4. 暫存檔路徑（預設：素材資料夾）
5. 解析度與比例（預設 16:9 1080p）
6. 語言（預設 zh-TW）
7. 章節規劃
8. 每章節的音樂風格
9. BGM 音量（預設 -20dB）
10. 輸出格式（預設 H.264）

全部回答完後，自動執行完整剪輯流程。

## 工作流程

```
Phase 1  初始化          → new_project + import_media
Phase 2  素材分析         → get_media + inspect_media 分類 A/B-roll
Phase 3  精選素材         → 逐段檢視，決定保留範圍與入點
Phase 4  時間軸建構       → add_clips + 粗剪 + remove_silence()
Phase 5  B-roll 覆疊      → 可選，預設跳過
Phase 6  背景音樂         → 逐章節分軌 + 音量 0.1 + 淡入淡出
Phase 7  字幕轉錄         → 外部 Whisper → 除幻覺 → 錯字修正 → add_texts
Phase 8  輸出             → H.264 或 fcpxml
```

## 字幕規格

| 項目 | 值 |
|:----:|:---:|
| 轉錄引擎 | `mlx_whisper` large-v2-mlx（GPU）或 `whisper` large-v2（CPU） |
| 字體顏色 | `#FFFFFF`（白色） |
| 背景顏色 | `#000000`（黑色） |
| 字體大小 | 48 pt |
| 位置 | 底部置中（centerY: 0.85） |
| 語言 | zh-TW |

> 不建議使用 large-v3-mlx，該模型在中文語音上有嚴重複讀幻覺（走走走、來來來等）。

## 錯字修正

Whisper 對中文專有名詞常有同音／近似音錯誤，內建常見修正規則：

| 原始 | 修正 |
|:----:|:----:|
| 太宅府 | 太宰府 |
| 梅汁餅 | 梅枝餅 |
| 遊步院 | 由布院 |
| 老貓 | 龍貓 |
| 神文時代 | 繩文時代 |

## 音樂

支援每章節配不同背景音樂。可：
- 使用本機音樂檔案
- 自動搜尋 CC0 無版權音樂（Pixabay、Uppbeat、YouTube Audio Library）

## 授權

本技能文件採用 MIT License。
