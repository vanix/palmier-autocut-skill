# Palmier AutoCut Skill

AI Agent 技能（Skill）——透過 Palmier Pro MCP 自動化影片剪輯流程。
我只有搭配 OpenCode 使用，可自行測試能否與其他 AI Agent 搭配。

## 概述

當使用者說出「幫我剪片」、「剪片」、「edit video」等關鍵字時自動觸發，涵蓋完整後製流程：

- 素材匯入與分類（A-roll / B-roll）
- 粗剪（trim、split、silence removal）
- 外部 Whisper 語音轉錄（`mlx_whisper` GPU 加速）
- 錯字修正與字幕定位
- 多章節背景音樂配樂（可上網搜尋 CC0 音樂，可能會不小心抓到版權音樂）
- 最終輸出（H.264 或 FCPXML）

## 安裝

```bash
git clone https://github.com/vanix/palmier-autocut-skill.git
cp -r palmier-autocut-skill/palmier-autocut ~/.config/opencode/skills/palmier-autocut
```

依賴套件：

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

系統會先以互動問卷收集需求（主題、素材路徑、剪輯風格、解析度、語言、章節音樂、輸出格式等），全部回答完後自動執行完整流程。

## 工作流程

```
Phase 1  初始化          → new_project + import_media
Phase 2  素材分析         → get_media + inspect_media 分類 A/B-roll
Phase 3  精選素材         → 流水帳：全上只清明顯廢段
                          AI挑重點：逐支分析只留精彩片段
Phase 4  時間軸建構       → add_clips（注意 startFrame 須累積）+ 粗剪 + remove_silence()
Phase 5  B-roll 覆疊      → 可選，預設跳過
Phase 6  背景音樂         → 逐章節分軌 + 音量 0.1 + 淡入淡出（注意裁切 BGM tail）
Phase 7  字幕轉錄         → 外部 Whisper → 除幻覺 → 錯字修正 → add_texts（一次匯入勿分批）
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

## 錯字修正

Whisper 轉錄後常有同音字、近似音錯誤，可用 Python regex 批次取代修正（SKILL 內附範例），也可自行請 AI Agent 幫你修正錯字。

## 授權

MIT License
