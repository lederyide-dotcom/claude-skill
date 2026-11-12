---
name: "pc-dev-env-win"
description: "替 使用者 的 Windows 開發機寫程式、腳本、部署或整合軟體時使用：列出已安裝的工具、版本、路徑與整合方式，避免建議重裝或用錯版本。"
---

# 開發機環境清單（desktop-54b809h）

硬體：Windows x64、RTX 4060 8GB。寫指令時以 PowerShell 為主，Linux 工具走 WSL2 或 Docker。已裝的工具直接用，不要建議重裝；缺少的才建議安裝，並優先用 scoop 或 winget。

## 語言與建置
- Python：3.10（`C:\Program Files\Python310`）＋ chocolatey 的 python3 / python313 並存 → 指令一律用 `py -3.10` 或專案 venv，不要假設 `python` 指向哪一版
- CUDA Toolkit 11.8 與 12.1 並存；PyTorch 安裝時明確指定 cu118 或 cu121；Nsight Systems / Compute 可用於效能分析
- Node.js、Go、Java、.NET、Visual Studio 2022 Build Tools / Windows SDK、Android Studio
- Git、GitHub CLI（gh）、Git Credential Manager、OpenSSH

## 容器與虛擬化
- Docker Desktop、WSL、Hyper-V、VcXsrv（WSL GUI）
- 本機服務資料：`D:\local-ai-agent\{n8n_data,open-webui,qdrant_storage}`

## 本機 AI
- Ollama（預設埠 11434）、LM Studio（OpenAI 相容 API，預設埠 1234）、AnythingLLM、ComfyUI Desktop、SD WebUI（D 槽）、vLLM（走 WSL/Docker）
- HuggingFace 快取：`C:\huggingface_cache`
- 編輯器／代理：VS Code、Cursor、Antigravity，以及 Cline、Aider、Continue、Codex、Copilot、Gemini CLI 設定

## 3D、遊戲與 CAD
- Blender 5.0（`C:\Program Files\Blender Foundation\Blender 5.0`）：批次用 `blender -b <file.blend> -P script.py`，腳本用 bpy
- Unity Hub（D 槽）＋ Unity 6000.3.15f1：C# 腳本，專案放 `D:\Unity` 或使用者目錄
- SolidWorks / AutoCAD 經驗（作品集有檔案）

## GIS 與水利
- QGIS 3.34.15 與 3.40.9、OSGeo4W Shell、GRASS GIS 8.4、SAGA GIS 9.x、QField、Google Earth Pro
- 自動化用 PyQGIS 或 `qgis_process`（在 OSGeo4W Shell 內執行）
- `C:\SWAT`、`C:\SewerAI`：水文／下水道相關工作目錄

## 資料庫
- PostgreSQL 17 ＋ PostGIS 3、pgRouting、h3-pg、pgPointcloud、MobilityDB、ogr_fdw；pgAdmin 4、psql
- SQL Server 2014 ＋ SQL Express（舊版，新專案優先用 PostgreSQL）
- MongoDB、SQLite（DB Browser）

## 工程實務
- PCCES Win 4.3（公共工程經費估價）：BOM／預算資料可從這裡匯出再結構化
- GanttProject、Edraw、Miro（RealtimeBoard）

## 影音
- ffmpeg 有多份（scoop、`C:\ffmpeg-master-latest-win64-gpl-shared`、Audacity 內建）→ 腳本寫明完整路徑或先 `where ffmpeg`
- Audacity、VidCoder、PotPlayer、Clipchamp、Vibe_Voice2Text（`D:\Tools`）

## 文件與雲端
- Microsoft Office、LibreOffice（可用 `soffice --headless --convert-to pdf` 批次轉檔）
- Google Drive、OneDrive、MEGAsync、pCloud
- Obsidian、AFFiNE

## 網路與遠端
- Bot webhook 測試優先用 ngrok 或 Cloudflare tunnel；內網機器互連用 Tailscale

## API 與通訊
- Postman、Telegram、Discord、LINE（Bot 開發）