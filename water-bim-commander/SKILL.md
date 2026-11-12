---
name: "water-bim-commander"
description: "在 water-treatment-bim-demo 專案當總指揮時使用：派子代理開發、合併推送等 CI、同步 F: 磁碟、請 BIM-Windows 跑 Unity 驗證、重跑一鍵驗證並更新驗證紀錄網頁。"
---

# 水處理儀控 BIM DEMO：總指揮流程

適用：使用者 說「你負責指揮」「持續目標」「依建議進行」時，在雲端 clone `/home/claude/water-treatment-bim-demo` 指揮。回覆一律繁體中文（台灣用語），依分工簡短回報。專案內同名檔案版在 `.claude/skills/water-bim-commander/SKILL.md`。

## 0. 開工
1. 讀 `CLAUDE.md`、`docs/GOALS.md`（§2 授權、§3 必停、§4 推送、§10 分工）、`docs/agent-inbox.md`、`docs/_exchange/README.md` 最新三筆交接。
2. `git fetch && git merge --ff-only origin/main`；看 `gh run list --branch main --limit 3` 確認 CI 綠。
3. 查 F: 鎖：`device_bash` 看 `data/.goal-run.lock`；有鎖且非本 session，不動 F:。

## 1. 派工（一項一個 worktree）
- `git worktree add -b <m#-name> /home/claude/wt/<name> origin/main`；需要的 gitignored 輸入（`data/imports/`、`data/review-events.jsonl`）先複製進去。
- 模型選擇：要看圖、設計判斷、跨模組 → Opus；查資料、規則明確的程式＋測試、文件 → Sonnet；機械小改 → 自己做。同時並行最多 2 個，省額度。
- 提示必含：要讀的檔、使用者 原話與日期、誠實標示規則（模擬值、assumed、AI 判斷不冒充人工）、不得碰 evidence／主 checkout／裝置、測試指令、交接檔格式（附署名）、README 表頂加一列、commit 結尾只加 Claude-Session 行（D-018：不加 Co-Authored-By）、不推送、最終回報格式。
- 子代理在背景跑；不輪詢、不讀其 transcript，等通知。

## 2. 審查與合併
- 讀回報，挑出與規則衝突的判斷並修正（例：OCR 結果不進 repo、類比出處只能算「推算」）。
- `git merge --no-edit <branch>`；`docs/_exchange/README.md` 衝突一律兩邊都留、新的在上。
- 合併 commit 訊息改成英文，結尾補 Claude-Session 行（不加 Co-Authored-By）。
- 必跑：`python3 tests/test_invariants.py`、`python3 -m unittest discover -s tests`，全過才推。
- 推送後**一定等 CI**（D-033 教訓）：`until gh run list ... | grep -q completed; do sleep 20; done`（背景執行或 timeout 夠長）。

## 3. 和 Windows 交換成果（2026-10-29 起：GOALS §4、`docs/agent-inbox.md`）
- 派工：用裝置橋接把任務檔寫進 F: 的 `data/inbox/<id>.md`（格式見 `docs/agent-inbox.md` §2）。
- 收件：`git fetch origin '+refs/heads/win/*:refs/remotes/origin/win/*'`，讀 F: 的 `data/outbox/`；審查 → 測試 → 合併 main → 推 → 等 CI → 刪已合併的 win/* 分支。
- F: 跟上 main：由 BIM-Windows 在下個任務前 `git merge --ff-only origin/main`；不要從 VM 改 F: 的工作樹。
- Windows 的 token（限本 repo、Contents 讀寫）有到期日；距到期不到兩週時提醒 使用者 更新。

### 備案：bundle 同步（Windows 推不出去時；VM 沒有刪除權限）
1. 雲端：`git bundle create /mnt/user-data/outputs/down-<sha>.bundle <F的HEAD>..main`；**每次用新檔名**（同檔名會傳到舊檔）。
2. `device_commit_files` 寫到 `F:\water-treatment-bim-demo\data\output\`。
3. VM：確認 HEAD 是預期值且工作樹乾淨 → `git fetch <bundle> main:refs/sync/<n>` → 對 diff 中每個檔 `git cat-file --filters <new>:<path> > <path>`（保留 .cmd 的 CRLF）→ `git update-ref HEAD <new> <old>` → `git update-ref refs/remotes/origin/main <new>` → `git read-tree HEAD`。
4. 每步後把殘留鎖改名：`.git/*.lock`、`.git/objects/*.lock`、`.git/refs/**/*.lock` 移到 `.git/stale-<name>-<時間>`（不可放在 refs 底下）。
5. 反向（Windows commit 上推）：VM `git bundle create data/output/up-<sha>.bundle <base>..main` → `device_stage_files` → 雲端 fetch＋ff → 跑測試 → 推 → 等 CI → VM 更新 origin/main。

## 4. Unity 實機驗證
- 雲端無法跑 Unity。首選：寫一個收件匣任務（`data/inbox/`），完成標準寫 log 字串：`WATER_BIM_RUNTIME_OK metadata=<N>`、`WATER_BIM_SIM_DONE failures=0`、`PARITY_OK differences=0`；BIM-Windows 的 `/loop` 會接手，證據 commit 在 `win/<主題>` 並推上來。
- 收件匣沒在跑時，備案是請 使用者 在 BIM-Windows（auto mode）貼一行有終點的 goal：`/goal 依 GOALS §2 跑 scripts\unity_verify.cmd …；git status 乾淨後停止，或 15 回合後停止`。沒有終點的 goal 會空轉，請 使用者 `/goal clear`（行首不能有空格）。
- BIM-Windows 跳出的確認：測試、git（推 main 以外）、改行尾、Unity batchmode、`git push origin win/*` → 允許；推 main、`gh`、刪檔、`.env`、`git push * --force`／`--no-verify` → 拒絕；「don't ask again for: git *」之類的寬放行一律不選。
- Windows 上 IfcOpenShell 可能被智慧型應用程式控制封鎖：算跳過，不建議關閉該保護（D-035）。

## 5. 驗收與發布
- `python3 scripts/demo_verify.py --out data/output/demo-verify/v<N>`（約 7 分鐘，背景跑），要 `整體 PASS`、UI 斷言全過。
- 看 1 張關鍵截圖確認畫面正確，再用 Artifact 發布到既有的驗證紀錄網頁（label 寫版次與主題）。
- 最後回報：結果表（項目／結果）、GitHub 與 F: 版本、等 使用者 的事（只列真的需要人決定的）。

## 必停（GOALS §3）
改資料定義或指標定義、改對外介面、刪原始資料、付費、碰 `F:\SD*` 與其他專案、碰憑證或 `.env`、連真實 PLC／SCADA。先準備好選項再問 使用者（一題、附建議）。