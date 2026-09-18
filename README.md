# Codex Task Handoff

`task-handoff` 是一套 Codex Skill，可讓同一項程式開發任務在不同的 Codex 對話之間延續。它把最小且可驗證的續作狀態保存在確認過的 project root 內，支援 Git repository，也支援沒有 Git 的明確 Codex workspace／project，不需要 conversation ID、背景服務或額外 registry。

最新已發布版本：**v0.1**

目前開發版本：**v0.2（Unreleased）**

## 主要功能

- **Create**：為新的獨立開發目標建立續作狀態。
- **Checkpoint**：在達到重要里程碑、取得驗證結果、遇到阻礙或準備暫停時更新狀態。
- **Handoff**：在切換對話前更新 canonical handoff。
- **Resume**：在新的對話中找出 active task，重新確認 project root、handoff、active task 與相關來源，再接著處理 Remaining Work。
- **Recover**：handoff 遺失或過期時，根據目前可用的 project evidence，重建最低限度且可靠的狀態；Git 模式可以使用 Git history 與 working-tree evidence，Non-Git 模式則不能。

## 什麼是 Task

Task 是需要跨對話延續的開發目標，不等於 Codex conversation／thread、Git branch、clone 或 commit。

如果一般的 debug 子問題仍是為了完成同一個交付目標，就沿用原本的 Task，不必另建 Task。只有形成獨立的交付目標時，才建立新的 Task。

## 設計範圍

續作狀態保存在確認過的 project root 內：

```text
<project-root>/.codex/
├── active-task
└── handoffs/
    └── <task-id>/
        └── current.md
```

- 每個 Git working tree 或 Non-Git project root 同一時間只處理一個進行中的 Task。
- 每個 task 只保留一份 canonical `current.md`，不另外建立歷史版本。
- 不儲存或映射 Codex thread ID。
- 不提供 registry、history、daemon、自動切換對話或 token monitoring。
- 執行 Resume 時，以目前可驗證的 Git／project evidence 與檔案內容為準，不會直接照著過期的 Remaining Work 執行。

## Git 與 Non-Git 模式

Skill 每次進行 Create、Checkpoint、Handoff、Resume 或 Recover 都會重新判斷模式與 project root：

1. 先執行 `git rev-parse --show-toplevel`。成功時一律使用 canonical Git root 與 **Git** 模式。
2. 只有當失敗原因明確表示目前位置不是 Git repository 時，才考慮 **Non-Git** fallback。若是 Git 未安裝、權限、corrupt repository、unsafe directory 或其他不明錯誤，會停止而不會靜默降級。
3. Non-Git root 必須來自目前 session 明確且唯一的 Codex workspace／project root，或使用者明示的絕對路徑。若 workspace 過寬且包含多個 plausible projects，除非 session context 或使用者明確指定該 root，否則仍視為不明確。不能只把 current directory、向上找到的 manifest 或常見目錄名稱當作 root。
4. 若沒有唯一可驗證的 root，Skill 會詢問使用者，不會自行猜測，也不會建立或更新 handoff。

| 驗證能力 | Git 模式 | Non-Git 模式 |
| --- | --- | --- |
| Project root | `git rev-parse --show-toplevel` 的 canonical root | 明確的 Codex workspace／project root 或使用者確認的絕對路徑 |
| Branch、commit、working tree | 驗證 branch、HEAD、`git status`；尚無 commit 時標示 partial | **Unavailable (Non-Git)**，不會假裝已驗證 |
| Stale-state 判斷 | Git state、handoff narrative、相關檔案 | root、handoff、active task、Relevant Files 的存在與目前內容 |
| Resume 的基準 | 目前 Git 與來源狀態 | 目前檔案內容；會明示 Git stale-state 驗證不可用 |
| Recover 的證據 | 可使用 bounded log、targeted diff、status 與來源 | 只能使用 `PROJECT_CONTEXT.md` 與目前相關檔案，信心較低 |

兩種模式都會驗證 task ID，並確認 `.codex/active-task` 與 `.codex/handoffs/<task-id>/current.md` 的正規化路徑留在 project root 內。既有 symlink 或 Windows reparse point 若會把路徑導向界線外，也會被拒絕。

Handoff 的既有 `Repository State` 區段仍保留，並新增下列欄位：

- `Mode`：`Git` 或 `Non-Git`。
- `Project Root Source`：Git top-level、Codex workspace／project context，或 user-confirmed absolute path；不保存可能包含個人資訊的絕對路徑。
- `Git Validation`：`Verified`、`Partial (No commits yet)` 或 `Unavailable (Non-Git)`。

Non-Git handoff 的 `Branch` 與 `Last Known Commit` 都必須寫成 `Unavailable (Non-Git)`。舊 handoff 缺少新增欄位時仍可 Resume；Skill 會先從目前環境重新判斷模式，並在下一次獲授權更新時補齊欄位。

### 從 Non-Git 轉成 Git

如果後來加入 Git，下一次操作會重新判斷模式：

- Git root 與原 project root 相同，而且 canonical handoff 仍位於該 root 下時，沿用原 task，將 mode change 視為 potentially stale，完成 Git 驗證後在下一次獲授權更新時改寫 metadata。
- 只有 `git init`、尚未建立 first commit 時，仍屬於 Git 模式。Metadata 使用 `Git Validation: Partial (No commits yet)`、已驗證的 branch（否則 `Unknown`），以及 `Last Known Commit: Unavailable (No commits yet)`；不會降回或標示成 Non-Git。
- 如果新的 Git root 是原 root 的父層或子層，導致 canonical `.codex` 位置改變，Skill 不會搜尋、讀取、搬移或複製舊 handoff。確認 canonical root 後，只能根據該 root 內的目前證據與使用者提供的 task facts 進行 Recover；canonical path 外的舊 handoff 仍不可讀取。

## `.codex` 與版本控制

`.codex/active-task` 和 `.codex/handoffs/` 通常只用來保存本機的續作狀態。在 Git 模式下，如果不希望把這些狀態提交，可以考慮在 repository 的 `.gitignore` 加入：

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

Skill 會先重新確認 project root、handoff、active task 與 Relevant Files。Git 模式還會驗證 branch、HEAD 與 working tree；Non-Git 模式會明示 Git 驗證不可用，並以目前檔案內容為準。

## 安全原則

- 不把 secrets、credentials、personal data、完整 log 或大型 diff 寫進 handoff。
- 不會只因 handoff 有記錄，就自動執行未經授權的產品程式碼變更、Git 操作、安裝流程或外部系統變更。
- 不會自動切換 branch，也不會執行 reset、restore、clean、stash 或 commit。
- task ID 必須符合小寫 kebab-case 格式，`.codex` 與 handoff 路徑也必須位於確認過的 project root 之下。
- handoff 只是可能過期的任務筆記，不能取代目前可取得的 Git／project evidence 與來源內容。

## 測試環境

v0.1 已在以下環境完成實際驗證：

- Windows。
- Codex Desktop 隨附的 USER Codex runtime。
- 使用 `<CODEX_HOME>/skills/task-handoff` 作為實測相容安裝位置。
- Codex 能在 USER scope 找到這個 Skill（discovery）。
- 可以明確呼叫 `$task-handoff`。
- 只輸入「繼續」即可觸發 implicit Resume。
- 已使用兩個不同的 thread 完成 Conversation A／B 的 Create、Checkpoint、Handoff 與 Resume 流程。Resume 會重新驗證 Git 與來源，再接著處理 Remaining Work。
- Failure Tests A～F 已完成；其中 v0.1 的非 Git 目錄案例預期停止，其他情境包括沒有 task、多個 plausible tasks、非法 task ID、stale handoff，以及 missing handoff recovery。

這次尚未發布的 Non-Git 調整另完成下列開發驗證：

- Git 模式仍能取得相同 root、branch、HEAD 與 working-tree status。
- 在隔離且明示絕對路徑的 Non-Git project 中完成 Create、Checkpoint、Handoff 與唯讀 Resume。
- Resume 重新確認 active task、handoff 的 `Task` 欄位、Relevant Files 的 containment／存在／內容，以及完整 schema；Non-Git metadata 沒有宣稱 Git 已驗證。
- 非法 task ID（包括 slash、backslash、`.`、`..`、traversal、drive prefix 與 absolute path）全數拒絕。
- 正規化 traversal 與 sibling-prefix trap 都無法通過 separator-aware containment。
- Root ambiguity 已完成停止且不寫入的 decision-branch 測試；尚未在沒有任何 workspace candidate 的 Desktop session 做完整互動測試。
- Non-Git→Git 同 root、unborn repository 與 root-change 分支已完成靜態決策測試；依限制未執行 `git init` 建立實際 unborn test repository。

## 已知限制

- 尚未用 GUI 視覺確認 Codex Desktop 側欄是否會立即顯示這個 Skill。
- 尚未用 GUI 視覺確認 `$task-handoff` 的 autocomplete／點選介面。
- Conversation A／B 並非透過 Codex Desktop UI 手動建立。
- 尚未在 Codex Desktop 以實際 Non-Git workspace 完成跨 thread 的 Create、Checkpoint、Handoff 與 Resume E2E。
- 尚未在實際 unborn Git repository 完成 Resume；目前驗證限於明確的決策與 metadata invariant。
- 尚未建立惡意 symlink／Windows reparse point，實際驗證 filesystem escape 會被拒絕；目前已驗證正規化 traversal 與 sibling-prefix trap。
- Windows sandbox helper 偶爾會出現 `helper_unknown_error`。現有證據指向執行環境問題，尚無證據顯示與 Skill 缺陷有關。

## 開發與驗證

修改 `SKILL.md` 或 `agents/openai.yaml` 後，請重新執行 Codex Skill validator，並分別在隔離的 Git test repository 與明確的 Non-Git test project 中驗證 discovery、handoff 與 resume。另需測試 root ambiguity、非法 task ID、path traversal 與 symlink／reparse-point containment。不能只靠文字比對判斷 Skill 是否正確。

## 授權

本專案採用 [MIT License](LICENSE)。
