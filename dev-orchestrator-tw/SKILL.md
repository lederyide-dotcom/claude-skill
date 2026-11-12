---
name: "dev-orchestrator-tw"
description: "由 Cowork 對話當總指揮，把開發目標拆成任務、並行派給子代理／雲端排程／Windows Claude Code，依任務選模型、在隔離環境測試、驗收證據、持續滾動到完成時使用。"
---

# 開發總指揮（Cowork → 子代理）

原則（來自 Felix Rieseberg 的做法與 使用者 的授權）：
- 指揮者不寫大段程式：只做判斷、拆解、派工、驗收、整合、回報。
- 任務切得清楚比模型強弱重要；切不清楚的先拿去規劃，不要直接派實作。
- 以結果驗收：只信驗收指令的實際輸出與 `02_data/` 證據，不信子代理自評。安全相關變更（簽章、白名單、權限、機密）例外：一定要另派審查者看 diff。
- 先在隔離環境試，再碰 使用者 的主機環境；只要安全規則守住，隔離環境內可以大量試。
- 狀態存在檔案與追蹤板，不靠對話記憶；任何一輪中斷，下一輪都能接手。
- 對 使用者 的回覆一律繁體中文、簡短、依分工（雲端／Windows／使用者）分組，不敘述過程。

## 0. 開工前確認（每個專案一次）
1. repo 根目錄要有：`CLAUDE.md`、`03_docs/spec.md`（沒有 → 先用 spec-handoff-tw 產出）、`03_docs/progress.md`（沒有就建）
2. 授權範圍寫在 spec 的「可以自行決定」與「必須停下來問我」；指揮者與所有子代理都只在這個範圍內決定
3. 追蹤板（例：Q4 目標 artifact 的 `tasks` collection）是唯一任務清單；新任務先寫進去再派工
4. 確認這個專案資料夾已連接到本對話（或排程的 folders）

## 1. 三條執行路線（先選路線，再選模型）

| 路線 | 怎麼啟動 | 做得到 | 做不到 |
|---|---|---|---|
| **A 雲端子代理** | 本對話的 Agent 工具（可並行、背景執行，可指定 model） | 研究、規格、文件、程式審查；不需 GPU 的程式與測試；在雲端容器用 uv 建拋棄式 venv（Python 3.10–3.13）做套件矩陣測試；可用 device_bash 讀寫已連接資料夾 | 連不到 Windows 的 GPU、Ollama 11434、Forge 7860；雲端容器沒有 Docker daemon；資料夾預設不能刪檔 → 不做 git 寫入 |
| **B 雲端輪替排程** | create_trigger（預設每 2 小時、台北 08–22）或 fire_trigger 立即跑 | 跨 session 持續滾動；每輪從檔案與追蹤板接手 | 同 A；且每輪是新 session，無法回訊息給其他 session |
| **C Windows Claude Code** | 使用者 開著的 Remote Control session；ListAgents 找標示 `Remote Control` 的列，SendMessage 派工 | GPU 訓練／推論、本機服務聯調、LINE 實測、Docker Desktop、WSL 測試 distro、Windows 原生驗證、git commit／push、在 worktree 內再開自己的子代理 | 使用者 不在線就不能用；訊息不能代替核准，對方的權限提示照常跳出 |

路線規則：
- 使用者 認為 Windows 無人值守運行有風險：**預設走 A＋B**，C 只在 使用者 開著 Remote Control 時用；不要用 Windows 工作排程器跑 `claude -p` 無人任務，除非 使用者 明確要求
- ListAgents 列不到 Windows session，或標示 `offline` → 需要 C 的任務在追蹤板 note 標「等 Windows」，推播 使用者 一次即可，不要重複催
- 派給 C 的訊息可能沒有回覆地址：結果一律要求寫進 `02_data/` 與 `03_docs/progress.md`，指揮者用 device_bash 讀檔驗收，不等回訊
- git commit／push 只由 C 或 使用者 做；A、B 只改檔、寫證據、在 progress.md 列「待 commit 的檔案」。commit 作者用 使用者 的 GitHub 帳號與 noreply email

## 2. 模型分派

| 任務性質 | 模型 | 例子 |
|---|---|---|
| 盤點、搜尋、讀 log、格式轉換、匯總數字、跑套件矩陣的單格 | `haiku` | 列出 runs/ 新結果、從 CSV 算指標、整理 import 清單、在一個 venv 裝套件並跑測試 |
| 範圍清楚、驗收可自動跑的實作／文件／測試／Dockerfile | `sonnet` | 依 spec 改一個模組、寫 README、補單元測試、寫測試用 Dockerfile |
| 範圍不清、架構取捨、跨模組、同一錯誤失敗 2 次、相依衝突判斷、最終審查 | `opus` | 拆解新目標、除錯卡關、安全審查、驗收判定 |
| 極難問題 | `fable` | 只在 使用者 明說可以時使用（可能耗用額外額度） |

- 實作者與審查者不可是同一個子代理；審查用新的子代理，只給 spec、起點 commit 與 diff，不給實作過程
- 升級規則：haiku 結果不完整 → sonnet；sonnet 同一項失敗 2 次 → opus 先重新切任務，再派 sonnet
- 不確定就選 sonnet；不要預設全用 opus

## 3. 並行規則
- 同時最多 3 個「會寫檔」的子代理；只讀的盤點／審查不限，但一輪合計不超過 6 個
- 套件矩陣的每一格各自獨立環境，可以並行；但只寫各自的結果檔，最後由一個子代理合併成 `02_data/compat_matrix.csv`
- 同一 repo 的寫入者：每個子代理在派工單寫死「只能改的檔案／資料夾」，互不重疊；會重疊就改成依序執行。C 路線在 Windows 用 git worktree（server mode `--spawn worktree` 或子代理 `isolation: worktree`）
- 同一個檔案不可同時被雲端（A／B）與 Windows（C）修改
- 鎖檔：排程每輪開工先檢查 `03_docs/.orchestrator.lock`；存在且未滿 90 分鐘 → 本輪直接結束；超過 → 視為中斷，接手。結束前一定移除或標記結束（不能刪檔時寫入 `released`）
- GPU 任務一次只派一個（RTX 4060 8GB），且要求先跑 `nvidia-smi`；GPU 容器也算在內

## 4. 隔離測試環境

目的：在乾淨環境驗證「一鍵安裝可用」、找出套件能支援的最大範圍、重現問題，且不污染 使用者 的主機。

| 層級（由輕到重） | 在哪裡 | 路線 | 適合 |
|---|---|---|---|
| 雲端拋棄式 venv | 雲端容器，`uv venv -p 3.10` 等；放 scratchpad 下的獨立資料夾 | A／B | 純 Python 相依解析、requirements 能否裝起來、CPU 單元測試、版本矩陣 |
| Docker 容器 | Windows Docker Desktop（WSL2 後端） | C | 系統套件、乾淨 Linux 安裝、CUDA 映像、多服務組合（compose） |
| WSL 測試 distro | 專用 distro `claude-sandbox` | C | 需要完整 Linux、反覆安裝序、長時間實驗 |
| Windows 原生 | 專案 venv，`py -3.10` | C | 正式環境的最後確認 |

選層規則：能在較輕的層證明的就不往上開。Linux 結果過不代表 Windows 過；任何要交付的結論，最後一格必須在 Windows 原生跑過（沒跑就標「等 Windows 驗證」）。

### 4.1 套件相容矩陣（找出可支援的最大範圍）
- 維度：Python 版本（預設 3.10、3.11、3.12）× 相依策略（鎖定版／最低版 `uv pip install --resolution lowest-direct`／最新版 `--upgrade`）× 平台（雲端 Linux CPU／Docker／WSL／Windows）× 計算（CPU／CUDA 11.8／CUDA 12.1，GPU 格只在 C 跑）
- 每一格：建乾淨環境 → 安裝 → `pip check` → 跑測試 → 記錄；不要在同一個環境裡換版本重試
- 結果寫 `02_data/compat_matrix.csv`：日期、commit、python、策略、平台、計算、安裝結果、測試結果、耗時、備註（失敗原因取錯誤訊息第一行）
- 產出建議：`requires-python`、每個主要套件的下限／上限、鎖檔（`requirements.lock` 或 `uv.lock`），寫進 progress.md
- 可自行決定：放寬版本範圍、補上遺漏的套件、小版本升級。必須問：大版本升級（例如 torch、numpy 2、CUDA 版本）、移除套件、替換主要框架
- 雲端裝不到的套件（網路白名單、Windows 專用、需 GPU）標「雲端無法測」，改派 C

### 4.2 Docker 安全規則
- 只用官方或 Docker Hub 認證發行者的映像，tag 固定版本，不用 `latest`
- 命名與標籤：`--name claude-test-<任務編號>`、`--label owner=claude-test`；建的映像也加同一個 label
- 掛載：repo 唯讀 `-v <repo>:/src:ro`；只有輸出資料夾可寫 `-v <repo>\02_data\docker:/out`。不掛 C:\、整個磁碟、使用者家目錄、`.env`、`docker.sock`
- 禁止 `--privileged`、`--network host`、`--cap-add`；需要對外埠只綁 `127.0.0.1`，避開主機已占用的埠（先查再綁）
- 資源上限：`--memory 8g --cpus 4`；GPU 容器 `--gpus all` 一次一個，且不與 Forge／Ollama 重載同時跑
- 機密不進映像：`.dockerignore` 排除 `.env`、`*.pt`、`*.safetensors`、資料集；需要環境變數時用測試用的假值檔
- 清理只清自己的：`docker ps -a --filter label=owner=claude-test`、`docker image prune -a --filter label=owner=claude-test`。禁止 `docker system prune`、`docker volume prune`，以及動到別人的容器（hermes、n8n、qdrant、open-webui 等）
- 磁碟：開始前看剩餘空間；本任務的映像加快取預估超過 20GB 先問

### 4.3 WSL 安全規則
- 只在專用測試 distro `claude-sandbox` 操作；第一次建立（`wsl --import`）先問 使用者，之後可自由使用
- distro 內 `/etc/wsl.conf` 設 `[automount] enabled=false` 與 `[interop] enabled=false`，讓它碰不到 Windows 磁碟與 exe；原始碼用 `git clone` 或複製進去，不直接掛 `/mnt/d`
- 可以重置自己建的 `claude-sandbox`（`wsl --terminate`、`wsl --unregister claude-sandbox`）；不動 使用者 既有的 distro，不改 `.wslconfig`
- 不執行 `wsl --shutdown`：會中斷 Docker Desktop 與 Hermes，需要時先問

### 4.4 雲端 venv 規則
- 一律建在 scratchpad 下的獨立資料夾，不在 使用者 的專案資料夾內建 venv
- 測試時複製一份原始碼到 scratchpad 再裝；不在連接資料夾裡寫快取或產物，只把結果 CSV 寫回 `02_data/`
- 網路受白名單限制；裝不到就記錄原因，不要改用其他來源繞過

## 5. 派工單（給每個子代理的固定格式）

```
任務：<追蹤板編號> <一句話>（回覆用繁體中文）
repo：<路徑>；先讀 CLAUDE.md 與 03_docs/spec.md，依 autonomous-dev-loop 執行。
測試環境：<雲端 venv／Docker／WSL claude-sandbox／Windows 原生>，遵守 dev-orchestrator-tw 第 4 節的安全規則。
只能改：<檔案或資料夾清單>
不可動：.env 與所有機密檔（不讀不印）、原始資料、spec「必須停下來問我」列出的事項
驗收（自己跑）：
- <指令> → 結果寫入 02_data/<檔名>
完成後：在 03_docs/progress.md 加一段（日期、任務編號、結果、證據路徑、待 commit 檔案、下一步）；清掉自己建的容器與暫存環境。
卡關：同一錯誤 3 次、碰到必須問的事、需要超過可改範圍 → 停下，寫明原因與需要什麼，不要繞過。
回報：結果（✅／⚠️／❌）、每條驗收的證據路徑、改了哪些檔、5 行內。
```

## 6. 每一輪的迴圈
1. **讀狀態**：追蹤板 tasks、spec、progress.md、`02_data/` 最新證據、git log（只讀）、CI 結果（有的話）。CI 紅燈 → 本輪只修紅燈
2. **選下一批**：未完成、依賴已滿足、路線可用的任務；依到期日排序。`owner = david` 的不派工，只列在回報
3. **切任務**：單一派工預估超過 60 分鐘或驗收不能自動化 → 先派 opus 拆成更小的任務並寫回追蹤板
4. **先隔離、後主機**：會改相依、安裝流程或系統設定的任務，先在第 4 節的隔離環境跑過，再派 Windows 原生驗證
5. **派工**：依第 1–5 節選路線、模型、環境、範圍；互不依賴的一次並行派出；追蹤板改成 `doing`，note 寫路線與模型
6. **驗收**：另派審查者重跑驗收指令、看 diff。沒有實際輸出的項目一律算未完成
7. **整合**：更新追蹤板（有證據才設 `done`；等 commit 或等 使用者 實測的設 `doing` 並寫明等什麼）、progress.md、spec 版本紀錄
8. **決定**：還有可派的且未達本輪上限 → 回到 2；否則回報並結束本輪

## 7. 必須停下來問 使用者
- 改動資料定義或類別、刪除原始資料、新增付費服務、改變對外介面
- 需要帳號、金鑰、網域、超過 8GB VRAM
- 放寬安全檢查、白名單、簽章驗證或內容過濾
- 大版本升級（torch、numpy 2、CUDA）、移除或替換主要套件
- 第一次建立 WSL 測試 distro、修改 Docker Desktop 或 `.wslconfig` 設定、`wsl --shutdown`、映像加快取預估超過 20GB
- 同一任務連續兩輪沒有進展
- 遇到停止條件時：把決定點寫進 progress.md（選項與建議）、追蹤板設 `blocked`、推播 使用者；若是排程，把該排程 enabled 設 false，等 使用者 決定後再開

## 8. 用量預算（Pro 額度與 Claude Code 共用）
- 一輪派工上限 6 個子代理；排程預設每 2 小時、台北 08–22
- 套件矩陣一輪最多 9 格；要更多就分幾輪跑
- 可用 haiku 就不用 sonnet；可用 sonnet 就不用 opus
- 出現用量上限或 rate limit → 本輪停止，回報剩餘任務，不重試迴圈

## 9. 持續滾動（建立指揮排程）
用 create_trigger 建一個排程，requires_local_device=true，folders 列出專案資料夾，prompt 範本：

```
你是 <專案> 的開發總指揮，依 dev-orchestrator-tw 執行一輪（回覆用繁體中文）。使用者 不在線上，不要等他。
追蹤板：<artifact URL>（tasks collection，只處理 goal = <目標編號>）
專案資料夾：<路徑>
本輪上限：6 個子代理、90 分鐘。
先拿鎖、讀狀態、派工、驗收、整合、釋放鎖。
會改相依或安裝流程的任務：先在雲端 venv 跑矩陣，需要 Docker／WSL／Windows 的格標「等 Windows」。
需要 Windows 的任務：ListAgents 找 Remote Control session；不在線就標「等 Windows」。
全部完成或碰到必須問的事 → 停用本排程（update_trigger enabled=false）並推播。
結束前用 SendUserMessage 回報：雲端／Windows／使用者 三組，各 1–2 行。
```

- 一個目標一個排程；不要讓兩個排程改同一個 repo
- 目標完成後停用排程，不要刪除（保留執行紀錄）

## 10. Windows 端準備（第一次使用 C 路線時告訴 使用者）
- Claude Code 需 v2.1.234 以上（Windows 原生的跨 session 訊息）
- 在 repo 目錄啟動：`claude remote-control --name <repo>-worker --spawn worktree --permission-mode acceptEdits`
  - `acceptEdits`：改檔不問、其他指令（含 docker、wsl）會推播到手機請 使用者 核准（預設，較安全）
  - 使用者 明說可以才用 `--permission-mode auto`
- 在 `/config` 開啟「Push when actions required」，核准請求會推到手機
- 建議放在 `~/.claude/agents/` 的子代理（可用 agent-env-setup-win 建立）：
  - `scout`（model: haiku，tools: Read, Grep, Glob, Bash）：盤點與讀 log
  - `implementer`（model: sonnet，isolation: worktree）：依派工單實作
  - `env-tester`（model: haiku）：依第 4 節在 Docker／WSL claude-sandbox 跑矩陣格
  - `code-reviewer`（model: opus，tools: Read, Grep, Glob, Bash）：只看 spec、起點 commit 與 diff

## 11. 回報格式（給 使用者）

```
本輪：<目標編號>｜完成 n／進行 n／卡住 n
雲端：<做完什麼，證據路徑>
Windows：<做完什麼／等 Windows 的任務>
使用者：<需要你決定或做的事，沒有就寫「無」>
```

## 搭配的 skill
- 規劃與 spec：spec-handoff-tw
- 子代理執行：autonomous-dev-loop
- 指令、路徑、版本：pc-dev-env-win；本機 AI 服務與 Docker：local-ai-stack-win
- 成果整理：demo-deliverable-pack