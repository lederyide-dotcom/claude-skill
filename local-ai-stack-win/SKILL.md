---
name: "local-ai-stack-win"
description: "在 Windows + RTX 4060 8GB 上部署或除錯本機 AI 服務時使用：Ollama、vLLM、SD WebUI、n8n、Qdrant、Open WebUI，以及 LINE／Telegram Bot webhook 串接。"
---

# 本機 AI 服務部署（Windows / 8GB VRAM）

## 硬體前提
- RTX 4060 8GB VRAM；同一時間只讓一個重 GPU 服務常駐（LLM 與 SD 同時跑會爆顯存）
- 給模型建議時先估 VRAM：權重大小 + KV cache + 系統保留約 1GB
  - 7B–8B Q4 量化：約 5GB，可順跑
  - 14B Q4：約 9GB，會溢出到系統記憶體，速度明顯下降
  - 更大模型：建議改用雲端 API 或多人共用 GPU 主機
- 每次建議都說明：模型、量化等級、context 長度、預估 VRAM

## 服務組合慣例
- Ollama：Windows 原生安裝，預設埠 11434
- vLLM：走 WSL2 或 Docker（需 GPU 支援），注意 `--gpu-memory-utilization` 與 `--max-model-len` 要壓低才放得進 8GB
- SD WebUI / ComfyUI：與 LLM 錯開使用，或啟用低顯存參數
- n8n + Qdrant + Open WebUI：用一份 `docker-compose.yml` 管理，資料掛載到 `D:\<專案>\<服務>_data`
- 每個服務寫清楚：埠號、資料目錄、啟停指令、健康檢查網址

## Bot 串接模式
- LINE：Flask 或 FastAPI 的 `/callback`，驗證 `X-Line-Signature`；Token 從 `.env` 讀
- Telegram：優先用 polling 開發、webhook 上線
- 對外測試用 ngrok 或 cloudflared tunnel；正式環境改固定網域
- 生圖、推論等長任務：先回覆「處理中」，完成後再推播，避免 webhook 逾時

## 除錯順序
1. `nvidia-smi` 看顯存與佔用程序
2. 服務本身 log（Docker：`docker compose logs -f <服務>`）
3. 埠衝突：`netstat -ano | findstr <埠號>`
4. WSL 與 Windows 之間的網路：容器內連主機用 `host.docker.internal`

## 輸出要求
- 指令分 PowerShell／WSL 兩欄標示清楚
- 設定檔給完整內容，不給片段
- 涉及機密時用 `.env` 變數名稱代替實際值