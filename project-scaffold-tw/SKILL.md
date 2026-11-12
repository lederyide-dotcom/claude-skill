---
name: "project-scaffold-tw"
description: "開新專案、PoC 或作品集條目，或整理既有專案資料夾時使用：套用 YYYYMM_主題 命名、00/01/99 編號資料夾、README、.env 機密管理與 Windows 腳本慣例。"
---

# 專案骨架與命名規範

## 何時使用
- 開新專案、新 PoC、新作品集條目
- 整理既有專案資料夾、補 README、補 .gitignore

## 命名規則
- 專案資料夾：`YYYYMM_主題`（例：`202607_n8n_Crawler_Docker_Telegram`）；單日活動／影片用 `YYYYMMDD_主題`
- 子資料夾用數字前綴排序：`00_素材`、`01_src`、`02_data`、`03_docs`、`99_成果`、`99_bk`
- 檔案版本：`_v1`、`_v2`…；不要用「(1)」「- 複製」這類系統自動命名，發現時建議改名
- 正式文件加文件編號：`DEV-YYYYMM-主題-NNN` 或 `DOC-YYYY-<代碼>-NNN`
- 程式、腳本、Docker 掛載路徑用英文＋底線，避免空白與全形符號（ffmpeg、.bat、WSL 掛載才不會出錯）；中文名只用在成果檔與文件

## 標準結構
```
YYYYMM_主題/
├─ 00_素材/      # 截圖、logo、原始影片、參考資料
├─ 01_src/       # 程式碼
├─ 02_data/      # dataset、模型輸出、runs（大檔不進 git）
├─ 03_docs/      # 規格、實測報告、會議紀錄
├─ 99_成果/      # pptx、pdf、demo.mp4、gif
├─ 99_bk/        # 舊版備份
├─ .env.example
├─ .gitignore
└─ README.md
```

## README 必備段落
1. 一句話目的（解決誰的什麼問題）
2. 架構圖（Mermaid）
3. 環境需求：GPU/VRAM、Python 版本、Docker/WSL、外部服務（LINE、Telegram、Ollama…）
4. 快速啟動（Windows 指令優先）
5. 成果連結（99_成果 內的影片、簡報）
6. 版本紀錄（日期＋變更）

## 機密處理
- Token、Channel Secret、API Key 一律放 `.env`，程式用環境變數讀取，並提供 `.env.example`
- recovery codes、token 文字檔不放在專案或作品集資料夾內；發現時提醒使用者移出
- `.gitignore` 預設排除：`.env`、`*.pt`、`*.safetensors`、`runs/`、`dataset/`、`*.mp4`、`__pycache__/`、`node_modules/`

## 環境慣例
- 主機為 Windows：啟動腳本提供 `.bat` 或 `.ps1`；需要 Linux 的部分註明走 WSL2 或 Docker
- 本機 GPU 為 RTX 4060 8GB：預設選擇可放進 8GB VRAM 的模型、量化等級與 batch size，超出時明講並提供替代方案