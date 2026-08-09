---
name: flow-finish
description: 完成 openspec change 的收尾——歸檔 spec 並決定如何整合（合併/建 PR/保留/丟棄）。當用戶說「完成這個 change」、「收尾」、「finish」、「準備合併」、「歸檔」或使用 /flow-finish 時觸發。先確認驗收已完成，執行 openspec archive 將 change 歸檔並同步 delta specs，再委派 finishing-a-development-branch 呈現合併/PR/保留/丟棄選項，選擇推送建 PR 時 PR 標題與內文一律以繁體中文（zh-TW）撰寫；若本次是用 flow-apply 建立的外部 worktree（位於專案目錄外層的 worktrees 資料夾），該 skill 認不出這個路徑不會自動清理，flow-finish 於合併或丟棄後補做 worktree 與分支清理。建 PR 後追問是否已合併，驗證 MERGED 後可切回原分支 pull 並選擇性清理。
---

# flow-finish：完成 change

flow-verify 之後的收尾編排：先歸檔 spec，再委派既有 skill 決定整合方式，
並補齊它認不出的 worktree 清理。

共用規範見同層目錄下的 `../flow-conventions/conventions.md`（本 skill 與
flow-conventions 須維持同一層級，不論安裝在 Claude Code 的 `.claude/skills/`
或其他環境的對應 skills 目錄）。**本 skill 的所有決策點提問一律依第 0 節
提問協定呈現與解析**；且本 skill 的選項多屬破壞性操作（丟棄、清理分支與
worktree），依第 0.4 節，預設值一律是保留／取消這類安全選項。

## 流程

### 1. 選定 change

選擇邏輯同 flow-apply/flow-verify（指定 > 推斷 > 列出讓用戶挑）。

### 2. 驗收完整性檢查

讀 `openspec/changes/<name>/.flow.yaml`。

**比較前一律先 normalize 代號**（`junit`→`unit-test`、`kdoc`→`doc-comment`，
見 conventions 第 3.1 節），否則舊 change 會被誤判為三項全缺：

```
normalized = normalize_aliases(verified)
```

- 若有 `verify_selected` 欄位 → 檢查其中每一項（同樣 normalize 後）是否都出現在
  `normalized` 裡，缺任一項 → 警告「驗收尚未完成（缺 X），仍要繼續歸檔嗎？」，
  經用戶確認才繼續（不強制擋，因為可能是刻意跳過某項）。
- 若沒有 `verify_selected`（優化前建立的舊 change）→ fallback 為檢查
  `normalized` 是否缺 `unit-test`／`doc-comment`／`code-review` 任一項，行為同上。

**驗收是否過期**：若 `.flow.yaml` 有 `verified_at`，以其 `head` 為基準重算：

```
git diff <verified_at.head> | git hash-object --stdin
```

- 與 `verified_at.diff_hash` 不同 → 警告「驗收結果可能已過期（程式碼在驗收後有
  變動），建議重跑 `/flow-verify <name>`，仍要繼續歸檔嗎？」，**提示但不阻擋**。
- 相同 → 未過期，不警告（不論這段內容此刻是否已被 commit）。
- 沒有 `verified_at` 欄位（舊 change）、或 `head` 對應的 commit 已不存在
  （如被 rebase／squash 掉）→ 略過此檢查，不報錯。

### 3. 執行 openspec archive

呼叫 `Skill(openspec-archive-change)`，**在目前分支、委派整合之前**執行：
- 同步 delta specs 進 `openspec/specs/<capability>/spec.md`
- 將 change 目錄搬到 `openspec/changes/archive/YYYY-MM-DD-<name>/`
  （`.flow.yaml` 隨目錄一起搬移）

archive 過程中若 openspec-archive-change 自己提示 artifacts／tasks 未完成，
依它原生的確認流程處理，不跳過確認直接硬做。

archive 必須在委派整合（第 4 步）之前完成，讓 spec 更新跟程式碼變更
落在同一個 PR／commit 裡一起走，不留下不同步的空窗期。

### 4. 委派完成整合

呼叫 `Skill(superpowers:finishing-a-development-branch)`，讓它：
- 驗證測試通過
- 呈現「合併本地 ／推送建 PR／保留／丟棄」選項
- 執行用戶選擇

記下用戶選了哪個選項，供第 5 步判斷是否要補清理。

**若選擇「推送建 PR」，PR 標題與內文一律以繁體中文（zh-TW）撰寫**——
覆寫該 skill 預設的英文慣例；若 repo 有既有 PR 樣板，內容仍需套用樣板結構，
只是文字改以 zh-TW 撰寫。

**若選擇「推送建 PR」，額外校正 PR 的 Destination Branch**：

讀 `openspec/changes/<name>/.flow.yaml` 的 `base_branch`：

- 沒有這個欄位（優化前建立的舊 change）→ 略過本項校正，不報錯。
- 有這個欄位 → 執行（不帶參數，`gh` 會依目前所在分支自動找到剛建立的 PR，
  不需要額外查 PR number）：
  ```bash
  gh pr view --json number,baseRefName
  ```
  取得 PR 編號與實際 base：
  - 與 `.flow.yaml` 的 `base_branch` 不同 → 執行：
    ```bash
    gh pr edit <number> --base <base_branch>
    ```
    修正，並在第 6 步收尾回報中註明「PR base branch 已從 X 修正為 Y」。
  - 相同 → 不需動作。

**PR 建立回傳後，追問合併狀態**：

依提問協定（單選）問：

> 「PR 已合併了嗎？要切回 <base_branch> 並 pull 嗎？」
> - **已合併，切回並 pull**
> - **尚未合併，留在目前分支**（預設；之後可自行切回，或重新執行 /flow-finish）

base 分支取 `.flow.yaml` 的 `base_branch`；缺欄位（舊 change）→ 改用上面
`gh pr view` 查到的 `baseRefName`。

- 選**尚未合併** → 到此結束，進入第 6 步收尾回報。
- 選**已合併** → 先驗證再動作：
  1. `gh pr view <number> --json state` 確認 state 為 `MERGED`；
     不是 → 警告「PR 狀態仍是 X，確定要切回嗎？」，經用戶再次確認才繼續。
  2. 切回並更新，依 `.flow.yaml` 的 `workspace` 區分：
     - **branch** → 在目前目錄執行：
       ```bash
       git checkout <base_branch>
       git pull
       ```
     - **worktree** → base 分支通常 checked out 在主 repo 目錄，不可在
       worktree 內 checkout；改為對主 repo 目錄執行 pull：
       ```bash
       MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
       git -C "$MAIN_ROOT" pull
       ```
       並提示用戶後續請回主 repo 目錄工作。
  3. 追問：「要順便清理本地分支／worktree 嗎？」→ 是：依第 5 步的清理
     指令執行（worktree remove → branch -d）；否：保留，提醒之後可手動清理。

### 5. 補齊 worktree 清理（僅當 `.flow.yaml` 的 `workspace: worktree`）

`finishing-a-development-branch` 只會自動清理 repo 內 `.worktrees/`／
`worktrees/` 底下的 worktree；flow-apply 建立的路徑是 repo 外部的
`<worktree_base>/<change-name>`，不符合它的清理條件。

**預期中的過渡性錯誤**：委派它執行「合併」或「丟棄」時，它會判定外部
worktree 不歸自己管而跳過移除，但接著仍會執行 `git branch -d/-D <branch>`
刪分支——此時分支還被 worktree 佔用，這個指令**會失敗並顯示
`cannot delete branch ... checked out at ...` 錯誤**。這是預期行為，
不是流程出錯，不要因此中斷；繼續往下執行本步驟即可解決（先移除
worktree，分支就能正常刪除）：

- 第 4 步選擇**合併本地**或**丟棄**時，flow-finish 自己補做：
  ```bash
  MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
  cd "$MAIN_ROOT"
  git worktree remove <worktree_path>
  git worktree prune
  git branch -d <branch>   # 丟棄情境改用 -D 強制刪除
  ```
- **移除前先讓當前 session 離開該 worktree**：若這次是 flow-apply 的「同
  session 切過去」（工作目錄就在 `<worktree_path>` 內），必須先用當前環境的
  離開機制退回主 repo 目錄（Claude Code：`ExitWorktree` 帶 `action: keep`，
  由本 skill 自行執行實際移除；無此機制的環境用上面的 `cd "$MAIN_ROOT"`），
  否則目錄被刪掉後 session 會停在不存在的路徑，後續指令全數失敗。
- 第 4 步選擇**推送建 PR**或**保留**時，**不清理**——worktree 還要留給
  用戶在 PR 上迭代修改。例外：第 4 步的合併追問已驗證 PR 為 MERGED
  且用戶同意清理時，已在該步驟處理完畢，本步驟不再重複。

若原本是用 **branch**（非 worktree）建立的 change，跳過本步驟，
分支清理完全交給 `finishing-a-development-branch` 處理。

### 6. 收尾

回報最終狀態：archive 後的路徑、選擇的整合方式、worktree／分支是否已清理。

## NEVER

- **NEVER 在驗收未完成時靜默略過警告直接歸檔**——必須先問過用戶再繼續。
- **NEVER 在 push/PR／保留情境下清理 worktree**——用戶還需要它繼續迭代 PR。唯一例外：`gh pr view` 已驗證 PR 狀態為 MERGED 且用戶明確同意清理。
- **NEVER 只憑用戶口頭說「已合併」就切回或清理**——必須先用 `gh pr view --json state` 驗證，狀態不是 MERGED 時警告並再次確認。
- **NEVER 假設 finishing-a-development-branch 會清理外部 worktree**——它只認
  `.worktrees/`／`worktrees/` 路徑，flow-apply 的慣例路徑不在其中，
  必須由本 skill 自己補做。
- **NEVER 把 archive 排在合併／建 PR 之後才做**——archive 必須發生在
  委派整合之前，確保 spec 更新跟著同一個 PR/commit 一起走。
- **NEVER 在 `.flow.yaml` 缺 `base_branch` 時報錯**——優化前建立的舊 change
  缺欄位屬正常，直接略過 PR base 校正即可，不得因此中斷收尾流程。
- **NEVER 用英文撰寫 PR 標題或內文**——一律以繁體中文（zh-TW）撰寫，
  技術名詞（程式碼識別符、指令、檔名）維持原文即可。
