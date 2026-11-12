---
name: "agent-env-setup-win"
description: "設定、檢查或修正 Claude Code 環境時使用（預設 Windows 原生）：主機診斷、%USERPROFILE%\\.claude 的 CLAUDE.md 與 skill 路由表、settings.json 備份合併、防護 Hook、code-reviewer subagent、從 WSL 遷移設定、repo 上線。"
---

# Claude Code 環境設定（Windows 原生優先）

原則：先診斷再設定；只合併、不覆蓋；每一步都能還原。少而精：1 個 Hook、1 個 subagent、1 個 MCP（GitHub），跑順再加。

## 目前已完成的安裝（2026-10-26）
- Claude Code 2.1.310 裝在 Windows（`C:\Users\User\.local\bin\claude.exe`），WSL 未安裝
- 已寫入 `%USERPROFILE%\.claude\`：CLAUDE.md、hooks\guard.py、agents\code-reviewer.md、7 個自訂 skill、settings.json（已合併）
- 安裝包在 `C:\Users\User\Downloads\claude-setup\`：雙擊 `run_setup.bat` 可重跑（冪等，有差異才備份覆蓋），紀錄在 `report\setup_log.txt`
- Hook 使用 `C:\Program Files\Python310\python.exe`；8 項實測全數通過

## 0. 執行端：預設 Windows 原生
使用者 的主力環境在 Windows（PowerShell、`py -3.10`、Docker Desktop、Windows 原生 Ollama、專案在 D:\／F:\），所以 Claude Code 設定放 `%USERPROFILE%\.claude\`。
- WSL2 與 Windows 的 `~/.claude` 是兩份獨立設定；**WSL2 沒有安裝 Claude Code CLI，就不要設定 WSL 端**
- 確認安裝位置：PowerShell `Get-Command claude; claude --version`；WSL `command -v claude`（注意拼字：`claude` 不是 `cluade`）
- 不要假設，先跑第 1 步診斷
- 從雲端 session 操作時：電腦操作對檔案總管只有「單擊」權限（雙擊會變成重新命名），終端機不能打字；做法是把安裝包寫到 Downloads，請 使用者 雙擊 .bat，再讀 report 驗證

## 1. 診斷（唯讀，PowerShell）

```powershell
[Console]::OutputEncoding = [Text.Encoding]::UTF8
$out = Join-Path ([Environment]::GetFolderPath('Desktop')) 'claude_diag.txt'
& {
  "== 工具 =="
  foreach ($c in 'claude','git','bash','py','python','node','docker','ollama','codex') {
    if (Get-Command $c -EA SilentlyContinue) { "{0,-8} {1}  [{2}]" -f $c, ((& $c --version 2>&1 | Select-Object -First 1)), (Get-Command $c).Source } else { "{0,-8} (無)" -f $c }
  }
  "== API 金鑰（只看有無）=="
  foreach ($n in 'ANTHROPIC_API_KEY','ANTHROPIC_BASE_URL') { $v = @('Process','User','Machine' | % { [Environment]::GetEnvironmentVariable($n,$_) }) | ? { $_ }; "{0} {1}" -f $n, $(if ($v) {'已設定 ⚠️'} else {'未設定'}) }
  "== GPU =="; nvidia-smi --query-gpu=name,memory.used,memory.total --format=csv,noheader
  "== Windows .claude =="; $cd = "$env:USERPROFILE\.claude"; if (Test-Path $cd) { Get-ChildItem $cd -Force -Name } else { '(無)' }
  "== WSL =="
  wsl -e sh -c 'command -v claude >/dev/null 2>&1 && echo WSL_claude=installed || echo WSL_claude=not_installed; ls -A ~/.claude 2>/dev/null || echo WSL_dotclaude=none'
  "== 專案 git / CLAUDE.md / spec =="
  foreach ($r in 'D:\','D:\作品集','F:\') { if (Test-Path $r) { Get-ChildItem $r -Directory -EA SilentlyContinue | ? { $_.Name -notmatch '^\$|System Volume|Recovery' } | % { "{0,-40} git:{1} CLAUDE:{2} spec:{3}" -f $_.FullName, [int](Test-Path "$($_.FullName)\.git"), [int](Test-Path "$($_.FullName)\CLAUDE.md"), [int](Test-Path "$($_.FullName)\03_docs\spec.md") } } }
} | Out-File $out -Encoding utf8
"已存到 $out"
```

### 依診斷結果決定
| 發現 | 處置 |
|---|---|
| Windows 無 claude | 先安裝 Claude Code（Windows 原生），再繼續 |
| `ANTHROPIC_API_KEY` 已設定 | 提醒會改走 API 計費（不吃 Pro 額度）；使用者 確認後才移除 |
| WSL 有 ~/.claude 但沒裝 CLI | 走第 3 步遷移到 Windows |
| 已有 settings.json | 走第 6 步合併，不覆蓋 |
| 專案無 git | 列為第一優先（第 9 步） |
| VRAM 被常駐服務占用 | 寫進全域 CLAUDE.md 的環境段 |

## 2. 備份
修改任何 `.claude` 檔案前，先複製到 `.claude\backup_YYYYMMDD_HHMMSS\` 或 `<檔名>.bak.YYYYMMDD_HHMMSS`。

## 3. 從 WSL 遷移到 Windows（WSL 端曾誤設時）

```powershell
$src = (wsl -e sh -c 'wslpath -w ~/.claude' | Out-String).Trim()
$dst = "$env:USERPROFILE\.claude"
$ts  = Get-Date -Format 'yyyyMMdd_HHmmss'
New-Item -ItemType Directory -Force $dst, "$dst\agents", "$dst\hooks" | Out-Null
foreach ($f in 'CLAUDE.md') {
  if (Test-Path "$dst\$f") { Copy-Item "$dst\$f" "$dst\$f.bak.$ts" }
  if (Test-Path "$src\$f") { Copy-Item "$src\$f" "$dst\$f" }
}
Copy-Item "$src\agents\*" "$dst\agents\" -Force -EA SilentlyContinue
Copy-Item "$src\hooks\*"  "$dst\hooks\"  -Force -EA SilentlyContinue
Get-ChildItem $dst -Recurse -Name
```
- **不要複製 settings.json**：WSL 版的 Hook 指令是 `python3 ~/...`，Windows 要用第 6 步重新合併產生
- 遷移後把 CLAUDE.md 的環境段改成 Windows 版（第 4 步範本）
- WSL 端舊檔留著無害；使用者 要清掉時再刪

## 4. 全域 CLAUDE.md（`%USERPROFILE%\.claude\CLAUDE.md`）
已存在則合併差異並先給 使用者 看；不存在則建立：

```markdown
# 全域規則（使用者）

## Skill 路由（依情境自動組合，回覆開頭一行標註本次套用哪些）
| 情境 | Skill |
|---|---|
| 規劃新任務、寫 spec、交接 | spec-handoff-tw |
| repo 內實作、修 bug、訓練、實驗 | autonomous-dev-loop |
| 開新專案、整理資料夾、README | project-scaffold-tw |
| 寫指令、用到本機工具或路徑 | pc-dev-env-win |
| 本機 AI 服務、GPU、Docker 服務、Bot webhook | local-ai-stack-win |
| 成果整理成文件、簡報、效能表、demo | demo-deliverable-pack |
| Claude Code 設定、Hook、權限、subagent | agent-env-setup-win |
常見組合：新 PoC = scaffold + spec；實作本機 AI 服務 = dev-loop + local-ai-stack + pc-dev-env；收尾 = dev-loop 驗證 → deliverable-pack。

## 環境
- Claude Code 在 Windows 原生執行；需要 Linux 的部分走 WSL2 或 Docker Desktop
- Python 用 `py -3.10` 或專案 venv，不假設 `python` 指向哪版
- RTX 4060 8GB VRAM，同時只跑一個重 GPU 服務；Ollama 為 Windows 原生（埠 11434）
- SD 模型目錄 F:\SD\models：唯讀
- 啟動腳本提供 .ps1 或 .bat

## 工作方式
- 開工先讀專案 CLAUDE.md 與 03_docs/spec.md
- 在 spec「可以自行決定」範圍內放手做，碰到「必須停下來問我」才停
- 完成 = 驗收標準全部親自驗證、證據寫進 02_data/
- 不確定就說不確定，絕不編造執行結果或數字
- 交付前呼叫 code-reviewer

## 程式碼
- 完整型別標註、Pydantic、I/O 用 async、明確例外處理、ruff
- 機密只放 .env 並提供 .env.example；新增套件寫進 requirements.txt

## Git
- 每完成一個驗收項目 commit，訊息繁體中文
- 不 force push、不 reset --hard、不改寫 main；大改動開分支或 worktree
```

## 5. 防護 Hook（`%USERPROFILE%\.claude\hooks\guard.py`）

```python
#!/usr/bin/env python3
"""PreToolUse 防護：攔截機密檔與破壞性操作。exit 2 = 阻擋並回饋給 Claude。"""
from __future__ import annotations

import json
import re
import sys
from typing import Any

RULES: list[tuple[str, str, bool]] = [
    # (pattern, 原因, 是否允許唯讀工具)
    (r"(^|[\s/\\'\"])\.env(?![\w.])", "禁止存取 .env（請改用 .env.example）", False),
    (r"\.env\.(?!example\b)\w+", "禁止存取 .env.* 機密檔", False),
    (r"rm\s+-[a-zA-Z]*r[a-zA-Z]*\s+(/|~|\*|\.\.?/?(\s|$)|\$HOME)", "禁止大範圍 rm -r", False),
    (r"Remove-Item\b.*-Recurse", "禁止 Remove-Item -Recurse，請先問 使用者", False),
    (r"git\s+push\s+.*(--force|-f\b)", "禁止 force push", False),
    (r"git\s+reset\s+--hard", "禁止 git reset --hard，請先問 使用者", False),
    (r"(/mnt/f/SD/models|F:[\\/]+SD[\\/]+models|/f/SD/models)", "SD 模型目錄只允許讀取", True),
]
READ_ONLY_TOOLS = {"Read", "Grep", "Glob"}


def main() -> int:
    # Windows 下 stdout/stderr 被導向時預設 cp950，含中文或符號會 UnicodeEncodeError → exit 1 → 擋不住
    for stream in (sys.stdout, sys.stderr):
        try:
            stream.reconfigure(encoding="utf-8", errors="replace")  # type: ignore[union-attr]
        except (AttributeError, ValueError):
            pass
    try:
        raw: bytes = sys.stdin.buffer.read()
        payload: dict[str, Any] = json.loads(raw.decode("utf-8-sig"))  # utf-8-sig：容忍 BOM
    except (json.JSONDecodeError, UnicodeDecodeError, ValueError) as exc:
        print(f"guard.py 無法解析輸入：{exc}", file=sys.stderr)
        return 0  # 解析失敗不阻擋，避免卡死所有工具

    tool: str = str(payload.get("tool_name", ""))
    ti: dict[str, Any] = payload.get("tool_input") or {}
    target = " ".join(str(ti[k]) for k in ("command", "file_path", "path", "pattern") if ti.get(k))

    for pattern, reason, read_ok in RULES:
        if read_ok and tool in READ_ONLY_TOOLS:
            continue
        if re.search(pattern, target, flags=re.IGNORECASE):
            print(f"[BLOCKED] {reason}｜工具：{tool}｜目標：{target[:200]}", file=sys.stderr)
            return 2
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

## 6. settings.json 合併（`merge_settings.py`）

```python
"""合併 ~/.claude/settings.json：保留既有設定，只加入權限規則與防護 Hook。

用法：python merge_settings.py [--report <遮蔽後副本輸出路徑>]
環境變數 HOOK_PYTHON：Hook 使用的 python.exe 絕對路徑（未設定則用目前直譯器）。
"""
from __future__ import annotations

import argparse
import json
import os
import re
import shutil
import sys
import tempfile
from datetime import datetime
from pathlib import Path
from typing import Any

HOME = Path.home()
PATH = HOME / ".claude" / "settings.json"
IS_WIN = os.name == "nt"
GUARD = (HOME / ".claude" / "hooks" / "guard.py").as_posix()
PY = Path(os.environ.get("HOOK_PYTHON") or sys.executable).as_posix()
HOOK_CMD = f'"{PY}" "{GUARD}"'

ALLOW: list[str] = [
    "Edit", "Bash(git:*)", "Bash(pytest:*)", "Bash(ruff:*)", "Bash(pip:*)",
    "Bash(docker compose:*)", "Bash(docker ps:*)", "Bash(docker logs:*)",
    "Bash(nvidia-smi:*)", "Bash(ls:*)", "Bash(mkdir:*)", "Bash(curl http://localhost:*)",
] + (["Bash(py:*)"] if IS_WIN else ["Bash(python:*)", "Bash(python3:*)"])
DENY: list[str] = [
    "Read(**/.env)", "Edit(**/.env)", "Bash(rm -rf:*)", "Bash(git push --force:*)",
    "Bash(git push -f:*)", "Bash(git reset --hard:*)",
] + ([] if IS_WIN else ["Edit(//mnt/f/SD/models/**)"])
MATCHER = "Bash|Edit|Write|MultiEdit|Read|Grep|Glob"
SECRET_KEY = re.compile(r"key|token|secret|password", re.IGNORECASE)


def union(old: list[str], new: list[str]) -> list[str]:
    return old + [x for x in new if x not in old]


def mask(obj: Any) -> Any:
    if isinstance(obj, dict):
        return {k: ("***" if SECRET_KEY.search(k) and isinstance(v, str) else mask(v)) for k, v in obj.items()}
    if isinstance(obj, list):
        return [mask(x) for x in obj]
    return obj


def load() -> dict[str, Any]:
    if not PATH.exists():
        return {}
    try:
        data = json.loads(PATH.read_text(encoding="utf-8-sig") or "{}")
    except json.JSONDecodeError as exc:
        sys.exit(f"[FAIL] 現有 settings.json 格式錯誤，未修改：{exc}")
    if not isinstance(data, dict):
        sys.exit("[FAIL] settings.json 不是 JSON 物件，未修改")
    bak = PATH.with_name(f"settings.json.bak.{datetime.now():%Y%m%d_%H%M%S}")
    shutil.copy2(PATH, bak)
    print(f"[BACKUP] {bak}")
    return data


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--report", type=Path, default=None)
    args = ap.parse_args()

    cfg = load()
    perms: dict[str, Any] = cfg.setdefault("permissions", {})
    a0, d0 = list(perms.get("allow", [])), list(perms.get("deny", []))
    perms["allow"], perms["deny"] = union(a0, ALLOW), union(d0, DENY)

    pre: list[dict[str, Any]] = cfg.setdefault("hooks", {}).setdefault("PreToolUse", [])
    replaced = False
    for entry in pre:
        for h in entry.get("hooks", []):
            if "guard.py" in str(h.get("command", "")):
                h["command"] = HOOK_CMD  # 統一成目前路徑
                entry["matcher"] = MATCHER
                replaced = True
    if not replaced:
        pre.append({"matcher": MATCHER, "hooks": [{"type": "command", "command": HOOK_CMD}]})

    PATH.parent.mkdir(parents=True, exist_ok=True)
    fd, tmp = tempfile.mkstemp(dir=PATH.parent, suffix=".tmp")
    try:
        with os.fdopen(fd, "w", encoding="utf-8") as f:
            json.dump(cfg, f, ensure_ascii=False, indent=2)
            f.write("\n")
        os.replace(tmp, PATH)
    except OSError as exc:
        Path(tmp).unlink(missing_ok=True)
        sys.exit(f"[FAIL] 寫入失敗：{exc}")

    json.loads(PATH.read_text(encoding="utf-8"))  # 再讀一次確認合法
    print(f"[OK] {PATH}｜allow +{len(perms['allow']) - len(a0)}｜deny +{len(perms['deny']) - len(d0)}｜hook {'已更新' if replaced else '已加入'}")
    print(f"[OK] Hook 指令：{HOOK_CMD}")
    kept = [k for k in cfg if k not in ("permissions", "hooks")]
    if kept:
        print(f"[OK] 保留原有設定：{', '.join(kept)}")
    risky = [x for x in perms["allow"] if x in ("Bash", "Bash(*)", "Bash(rm:*)", "Bash(sudo:*)")]
    if risky:
        print(f"[WARN] 既有 allow 過寬，建議檢查：{risky}")

    if args.report:
        args.report.parent.mkdir(parents=True, exist_ok=True)
        args.report.write_text(json.dumps(mask(cfg), ensure_ascii=False, indent=2) + "\n", encoding="utf-8")


if __name__ == "__main__":
    main()
```

執行（PowerShell）：`$env:HOOK_PYTHON = 'C:\Program Files\Python310\python.exe'; py -3.10 merge_settings.py --report report\settings.masked.json`
- Hook 指令用 python.exe **絕對路徑**＋ guard.py 絕對路徑（已實測：`"C:/Program Files/Python310/python.exe" "C:/Users/User/.claude/hooks/guard.py"`）
- 重跑安全：重複規則不會重加，既有 guard Hook 會更新成目前路徑
- 還原：把最新的 `settings.json.bak.*` 複製回 `settings.json`

## 已知坑（Windows PowerShell 5.1，已實際踩過）
- 呼叫原生程式時不要接 `| Select-Object -First 1`：會提前終止程式，`$LASTEXITCODE` 變成非 0。先 `$lines = @(& exe ...)` 再取第一行
- 傳給原生程式的參數不要含雙引號，5.1 會吃掉引號導致語法錯誤
- `$OutputEncoding = [Text.Encoding]::UTF8` 會在管線輸入前加 BOM；要用 `New-Object System.Text.UTF8Encoding $false`
- 含中文的 .ps1 要存成 UTF-8 **含 BOM**，否則 5.1 以 cp950 讀取會亂碼
- SKILL.md、CLAUDE.md、settings.json 要 **不含 BOM**（用 Copy-Item 或 Python 寫入，不用 5.1 的 `Out-File -Encoding utf8`）
- PATH 上的 `python.exe`／`python3.exe` 可能是 `WindowsApps` 的 Store 假捷徑，要排除
- Python 讀 stdin 用 `sys.stdin.buffer.read().decode("utf-8-sig")`；stderr 要 reconfigure 成 utf-8，否則中文訊息在 cp950 下會讓 Hook exit 1 而擋不住
- 原生程式輸出不會進 Start-Transcript，要 `2>&1 | ForEach-Object { Write-Host $_ }` 才會記錄

## 7. code-reviewer subagent（`%USERPROFILE%\.claude\agents\code-reviewer.md`）

```markdown
---
name: code-reviewer
description: 實作完成、交付或 commit 前主動使用。以全新視角審查變更，對照 spec.md 驗收標準，找出錯誤、安全問題與未達標之處。
tools: Read, Grep, Glob, Bash
---
你是獨立審查者，沒看過實作過程。只根據 repo 現況與 03_docs/spec.md 判斷，不相信先前任何回報。

步驟：
1. `git log --oneline -10`、`git diff <起點commit>` 列出變更
2. 逐條對照 spec 驗收標準，自己重跑驗證指令
3. 檢查 02_data/ 證據檔是否存在、數字是否與回報一致
4. 檢查型別標註、例外處理、機密外洩、寫死路徑、缺漏相依套件、train/val 資料洩漏

輸出：
- 驗收表（項目｜✅ 通過 / ❌ 未達標 / ⚠️ 無法驗證｜證據）
- 問題：🔴 必修（檔名:行號）、🟡 建議、🟢 小事
- 結論：可交付／需修正（一句話理由）

不要修改任何檔案。
```

## 8. Skill 安裝到本機 Claude Code
`%USERPROFILE%\.claude\skills\` 已有 `synced\` 資料夾，帳號 skill 很可能會自動同步；另外也放了一份本機副本。在 Claude Code 問「列出你目前可用的 skills」：同名 skill 出現兩次時，刪掉 `skills\<名稱>\` 本機副本，保留 synced 版。

## 9. Repo 上線（每個專案一次）
1. 無 git：建立 .gitignore（Python 慣例、.env 保留 .env.example、venv、__pycache__、`*.pt *.pth *.safetensors *.ckpt *.onnx`、dataset、runs、log、`*.mp4`），列出將追蹤檔案與總大小，單檔超過 50MB 先問，使用者 確認後 `git init` 並 commit「初始化版本控制」
2. 專案 CLAUDE.md（60 行內）：一句話目的、ASCII 架構、實際跑過的啟動／測試指令、資料夾用途、禁區、已知問題；不確定標 TODO；先給 使用者 看再 commit
3. 建立 `03_docs/`，有任務時依 spec-handoff-tw 產出 spec.md

## 10. 驗證設定（在任一 repo 啟動 claude 後）
1. 要求讀取 `.env` → 預期出現 `[BLOCKED]` 訊息
2. 要求執行 `git reset --hard` → 預期被擋
3. 要求列出 subagent → 預期有 code-reviewer
4. 要求列出 skills → 預期 7 個自訂 skill 都在，且沒有重複
任何一項不符，回報實際輸出並修正，不要宣稱完成。