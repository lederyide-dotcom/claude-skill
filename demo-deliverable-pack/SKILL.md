---
name: "demo-deliverable-pack"
description: "專案做出成果、要整理成交付或展示素材時使用：產出技術文件、CPU/GPU/人工效能實測表、提案簡報、Demo 影片分鏡與 ffmpeg GIF 腳本。"
---

# 成果交付包

每個專案收尾時固定產出下列五件，放進 `99_成果/`。先問清楚對象（主管、客戶、評審、課程）再決定語氣與深度。

## 1. 技術文件
- 文件編號：`DEV-YYYYMM-主題-NNN`
- 章節：目的與背景 → 系統架構（Mermaid）→ 方法與模型 → 實測結果 → 問題與解決 → 結論與下一步
- 數據一律附來源（哪個資料集、幾筆、哪台機器）

## 2. 效能實測表（試算表）
- 固定比較欄位：方法（人工／CPU／GPU）、處理量、總耗時、單位耗時、準確率或錯誤率、硬體規格、成本估算
- 用公式計算節省時間與倍數，不手打結果
- 附一張比較長條圖

## 3. 提案簡報（10–15 頁）
頁序：痛點 → 現況流程 → AI 解法 → 架構圖 → Demo 截圖 → 實測數據 → 效益（時間／人力／成本）→ 風險與限制 → 導入階段 → 下一步
- 每頁一個主張，標題寫結論不寫主題

## 4. Demo 影片分鏡
- 長度 60–120 秒，分鏡表欄位：秒數、畫面、旁白／字幕、素材檔名
- 開頭 5 秒先給結果畫面，再講流程

## 5. GIF 與縮圖（Windows .bat）
```bat
:: 截圖序列轉 GIF（1 fps）
(for %%F in ("*.png") do @echo file '%%~fF') > file_list.txt
ffmpeg -r 1 -f concat -safe 0 -i file_list.txt -vf "scale=960:-1" output.gif

:: 影片轉高品質 GIF（調色盤兩段式）
ffmpeg -i demo.mp4 -vf "fps=10,scale=960:-1:flags=lanczos,palettegen" palette.png
ffmpeg -i demo.mp4 -i palette.png -lavfi "fps=10,scale=960:-1:flags=lanczos[x];[x][1:v]paletteuse" demo.gif

:: 擷取封面圖（第 3 秒）
ffmpeg -ss 3 -i demo.mp4 -frames:v 1 cover.jpg
```

## 檢查清單
- 數字在文件、試算表、簡報三處一致
- 檔名帶版本號與日期
- 沒有出現 token、內部 IP、客戶個資