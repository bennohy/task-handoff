# Codex Task Handoff

`task-handoff` 是一套 Codex Skill，可讓同一項程式開發任務在不同的 Codex 對話之間延續。它把最小且可驗證的續作狀態保存在 Git repository 內，不需要 conversation ID、背景服務或額外 registry。

目前版本：**v0.1**

## 主要功能

- **Create**：為新的獨立開發目標建立續作狀態。
- **Checkpoint**：在達到重要里程碑、取得驗證結果、遇到阻礙或準備暫停時更新狀態。
- **Handoff**：在切換對話前更新 canonical handoff。
- **Resume**：在新的對話中找出 active task，重新驗證 Git 與相關來源，再接著處理 Remaining Work。
- **Recover**：handoff 遺失或過期時，根據 `PROJECT_CONTEXT.md`、Git 與相關來源，重建最低限度且可靠的狀態。

## 什麼是 Task

Task 是需要跨對話延續的開發目標，不等於 Codex conversation／thread、Git branch、clone 或 commit。

如果一般的 debug 子問題仍是為了完成同一個交付目標，就沿用原本的 Task，不必另建 Task。只有形成獨立的交付目標時，才建立新的 Task。

## 設計範圍

v0.1 把續作狀態保存在 repository 內：

```text
<repo-root>/.codex/
├── active-task
└── handoffs/
    └── <task-id>/
        └── current.md
```

- 每個 Git working tree（目前用來實際編輯檔案的工作目錄）同一時間只處理一個進行中的 Task。
- 每個 task 只保留一份 canonical `current.md`，不另外建立歷史版本。
- 不儲存或映射 Codex thread ID。
- 不提供 registry、history、daemon、自動切換對話或 token monitoring。
- 執行 Resume 時，以目前的 Git 與來源狀態為準，不會直接照著過期的 Remaining Work 執行。

## `.codex` 與 Git 版本控制

`.codex/active-task` 和 `.codex/handoffs/` 通常只用來保存本機的續作狀態。如果不希望把這些狀態提交到 Git，可以考慮在 repository 的 `.gitignore` 加入：

```gitignore
.codex/active-task
.codex/handoffs/
```

加入規則前，請先確認 repository 原本是否已使用 `.codex` 存放其他內容。不要直接忽略整個 `.codex/`，以免連同其他需要版本控制的設定一起排除。

## Skill 結構

```text
task-handoff/
├── SKILL.md
├── README.md
└── agents/
    └── openai.yaml
```

- `SKILL.md`：Codex 使用的正式操作規範。
- `agents/openai.yaml`：定義 UI metadata、預設 prompt，以及 implicit invocation policy。
- `README.md`：供 GitHub 使用者閱讀；Codex 仍以 `SKILL.md` 作為正式操作指令。

## 安裝

### 官方建議位置

OpenAI 官方文件目前將 USER scope Skill 的本機位置列為 `$HOME/.agents/skills`，因此對應位置是 `$HOME/.agents/skills/task-handoff`。官方也建議透過 `$skill-installer` 安裝指定的 Skill。發布到 GitHub 後，請以當時的 [官方 Skill 文件](https://developers.openai.com/docs/build-skills) 為準。

### v0.1 實測相容的手動安裝位置

本版本已在 Codex Desktop 隨附的 USER Codex runtime 使用下列位置完成 E2E 驗證：

```text
<CODEX_HOME>/skills/task-handoff
```

`<CODEX_HOME>` 是 Codex 狀態資料根目錄的路徑占位符，不能直接當成終端機命令。未設定 `CODEX_HOME` 時，預設值是 `~/.codex`。在 Windows 上，預設實際位置是 `%USERPROFILE%\.codex\skills\task-handoff`。詳情可參考 [OpenAI 的環境變數文件](https://developers.openai.com/docs/config-file/environment-variables)。

安裝前請先確認目標資料夾是否已存在；如果存在，先比較內容，不要直接覆寫不明檔案。

Windows PowerShell 範例（請在 repository 上一層執行）：

```powershell
$source = Resolve-Path .\task-handoff
$codexHome = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $HOME ".codex" }
$skillsRoot = Join-Path $codexHome "skills"
$destination = Join-Path $skillsRoot "task-handoff"

if (Test-Path -LiteralPath $destination) {
    throw "Target already exists: $destination"
}

New-Item -ItemType Directory -Path $skillsRoot -Force | Out-Null
Copy-Item -LiteralPath $source -Destination $destination -Recurse
```

macOS／Linux shell 範例（請在 repository 上一層執行）：

```sh
source_dir="./task-handoff"
codex_home="${CODEX_HOME:-$HOME/.codex}"
destination="$codex_home/skills/task-handoff"

if [ -e "$destination" ]; then
    printf 'Target already exists: %s\n' "$destination" >&2
    exit 1
fi

mkdir -p "$codex_home/skills"
cp -R "$source_dir" "$destination"
```

安裝後，請新開一個 Codex 對話，確認系統能找到 `$task-handoff`。如果 Codex 找不到這個 Skill，請重新啟動 Codex，再檢查實際安裝位置。

## 使用方式

明確建立或保存 task 狀態：

```text
Use $task-handoff to create continuity state for this coding task.
```

準備切換對話：

```text
Use $task-handoff to checkpoint the current milestone and prepare a handoff.
```

新開對話後，如果 repository 內只有一個能明確識別的 active task，可以直接輸入：

```text
繼續
```

Skill 會先讀取 handoff，重新確認 repository root、branch、HEAD、working tree 和相關來源，再判斷哪些工作可以安全接續。

## 安全原則

- 不把 secrets、credentials、personal data、完整 log 或大型 diff 寫進 handoff。
- 不會只因 handoff 有記錄，就自動執行未經授權的產品程式碼變更、Git 操作、安裝流程或外部系統變更。
- 不會自動切換 branch，也不會執行 reset、restore、clean、stash 或 commit。
- task ID 必須符合小寫 kebab-case 格式，寫入路徑也必須位於 `.codex/handoffs/` 之下。
- handoff 只是可能過期的任務筆記，不能取代目前的 Git 與來源證據。

## 測試環境

v0.1 已在以下環境完成實際驗證：

- Windows。
- Codex Desktop 隨附的 USER Codex runtime。
- 使用 `<CODEX_HOME>/skills/task-handoff` 作為實測相容安裝位置。
- Codex 能在 USER scope 找到這個 Skill（discovery）。
- 可以明確呼叫 `$task-handoff`。
- 只輸入「繼續」即可觸發 implicit Resume。
- 已使用兩個不同的 thread 完成 Conversation A／B 的 Create、Checkpoint、Handoff 與 Resume 流程。Resume 會重新驗證 Git 與來源，再接著處理 Remaining Work。
- Failure Tests A～F 已完成，測試情境包括非 Git 目錄、沒有 task、多個 plausible tasks、非法 task ID、stale handoff，以及 missing handoff recovery。

## 已知限制

- 尚未用 GUI 視覺確認 Codex Desktop 側欄是否會立即顯示這個 Skill。
- 尚未用 GUI 視覺確認 `$task-handoff` 的 autocomplete／點選介面。
- Conversation A／B 並非透過 Codex Desktop UI 手動建立。
- Windows sandbox helper 偶爾會出現 `helper_unknown_error`。現有證據指向執行環境問題，尚無證據顯示與 Skill 缺陷有關。

## 開發與驗證

修改 `SKILL.md` 或 `agents/openai.yaml` 後，請重新執行 Codex Skill validator，並在隔離的 Git test repository 中實際驗證 discovery、handoff 與 resume。不能只靠文字比對判斷 Skill 是否正確。

## 授權

本專案採用 [MIT License](LICENSE)。
