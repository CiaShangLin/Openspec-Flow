---
name: flow-finish
description: 使用者手動完成 OpenSpec change、歸檔，並在最後決定本地合併、建立 PR 或保留分支時使用。觸發語句包含「完成這個 change」、「收尾」、「finish」、「準備合併」、「歸檔」或 /flow-finish。
---

# flow-finish：完成 change

手動觸發的最後收尾。實作與驗收已完成、實作檔已先提交；本 skill 只提交 archive／同步 spec
產生的變更、推送分支，然後委派 `superpowers:finishing-a-development-branch` 決定整合。

共用規範見同層目錄下的 `../flow-conventions/conventions.md`。所有決策點依其第 0 節
提問協定呈現與解析；取消、保留與不清理一律是預設。

## 流程

### 1. 選定或續跑 change

選擇邏輯同 flow-apply/flow-verify（指定 > 推斷 > 列出讓用戶挑）。先在
`openspec/changes/<name>/` 找 active change：

- 找到 active change → 執行第 2–4 步。
- 找不到 active change，但在 `openspec/changes/archive/` 找到同名 archive → 這是前次
  archive 已完成、commit、push 或整合中斷的續跑。若有多個同名 archive，列出完整路徑讓
  用戶選擇；選定後直接執行第 5 步，不得再次 archive。
- 兩者都找不到 → 明確報錯並停止，不猜測 archive 路徑。

選定後把 `FLOW_PATH` 固定為 active change 的 `.flow.yaml`，或已選 archive 的
`.flow.yaml`。之後所有 `workspace`、`branch`、`base_branch` 都從 `FLOW_PATH` 讀取；
archive 後必須更新 `FLOW_PATH` 到 archive 內的新路徑，不能再讀舊 active 路徑。

archive 完成後，`FLOW_PATH` 必須寫入可續跑的精確路徑：

```yaml
finish:
  archive_path: openspec/changes/archive/YYYY-MM-DD-<name>
  synced_specs:
    - openspec/specs/<capability>/spec.md
```

`synced_specs` 是本次 archive 實際更新的主 spec 路徑。舊 archive 缺少此欄位時，不得猜測
要 stage 哪些 spec；列出 archive 路徑並停止，交由使用者處理。

### 2. 驗收完整性檢查（僅 active change）

讀 `FLOW_PATH`，比較前一律 normalize 代號（`junit`→`unit-test`、
`kdoc`→`doc-comment`；見 conventions 第 3.1 節）。

- 有 `verify_selected` → 檢查每一項是否都在 normalized `verified`。
- 沒有 `verify_selected` 的舊 change → fallback 檢查 `unit-test`、`doc-comment`、
  `code-review` 是否都在 normalized `verified`。

缺項時依提問協定單選：

> 驗收尚未完成（缺 X），要繼續收尾嗎？
> 1. 繼續收尾
> 2. 停止，先補完驗收（預設）

選停止就結束；明確選繼續才可往下。

**不比較 `verified_at`**：手動呼叫 flow-finish 已代表使用者接受當前驗收結果。
`verified_at` 仍由 flow-verify 保存作為驗收紀錄，但不得以 hash 差異警告、阻擋或要求
重跑驗收。

### 3. 收尾前工作區檢查（僅 active change）

實作檔必須已先 commit。檢查 `git status --porcelain`：唯一可接受的未提交檔案是
`openspec/changes/<name>/.flow.yaml`，因為它會隨 archive 一起成為本次收尾 commit 的
一部分。

若還有其他 staged、unstaged 或 untracked 檔案，列出檔案並停止；不得 stash、丟棄、
或把它們納入收尾 commit。使用者處理並提交後，再重新執行 flow-finish。

### 4. 自動同步與 archive（僅 active change）

先取得 `openspec status --change <name> --json`，並讀 tasks 檔案。從 status 的
`artifactPaths.specs.existingOutputPaths` 取得本 change 的 delta spec，依 capability 映射出
主 spec 路徑 `openspec/specs/<capability>/spec.md`，記為 `SYNCED_SPECS`；任何路徑無法安全
映射時停止，不猜測。

- 任一 artifact 未完成、或 tasks 有 `- [ ]` → 依提問協定單選顯示未完成項目：
  1. 仍然同步並歸檔
  2. 停止收尾（預設）
  只有明確選 1 才繼續。
- 全部完成 → 不提問。

沒有停止時，直接執行：

```bash
openspec archive <name> --yes --json
```

此指令會同步 delta specs 並移動 change；不得再為一般 sync／archive 詢問使用者。
從 JSON 的 `archive.path` 取得 archive 路徑，並將預先映射的 `SYNCED_SPECS` 寫入上方
`finish` 欄位；再將 `FLOW_PATH` 更新為 `<archive_path>/.flow.yaml`，供後續步驟讀取。

`openspec archive` 失敗時，保留現場並停止；不得進入 commit、push 或整合。

### 5. 續跑判定、精確 commit 並 push

本步適用於剛 archive 的 change 和第 1 步辨識出的 archive 續跑。

先從 `FLOW_PATH` 讀取 `branch`、`base_branch` 與 `finish`。先檢查既有整合狀態：

- `git merge-base --is-ancestor <branch> <base_branch>` 為真 → source branch 已在本地 base
  branch。直接 `git push origin <base_branch>`；成功後進入第 8 步，失敗則保留現場並停止。
  不得再次委派 merge。
- 先以 `gh pr list --head <branch> --state open --json number,state,baseRefName,url` 尋找開啟中的
  PR；找到時若恰有一個就保存其 `PR_NUMBER` 與 `PR_URL` 後直接進入第 7 步，多個時列出讓
  使用者選一個。沒有開啟中的 PR 才再查 `--state merged`；若找到已合併 PR，保存其
  `PR_NUMBER` 後進入第 7 步同步 base。已關閉但未合併的 PR 不得阻止建立新的 PR。
- 其他情況 → 目前 `git branch --show-current` 必須等於 `branch`；不相同就停止並顯示應回到的
  branch／worktree，不得 push 目前錯誤的 branch。

只有第三種情況才執行下列 commit/push：

1. 用 `finish.archive_path`、`finish.synced_specs` 與原 active path（若仍在 index）組成唯一可
   stage 的路徑集合。`git status --porcelain` 若出現集合外的檔案就停止。使用精確路徑的
   `git add -A -- <paths>`；不得用 `git add .` 或納入其他變更。
2. 若有 staged 變更，先尋找專案指示或設定指定的 commit skill；有可用且適用的專屬 skill
   就委派它產生／執行 commit。沒有時執行：
   ```bash
   git commit -m "chore: archive OpenSpec change <name>"
   ```
   若沒有 archive／spec 變更可提交，清楚回報，但仍繼續檢查 push。
3. 取得目前分支與 upstream。已有 upstream → `git push`；沒有 upstream 但有 `origin` →
   `git push -u origin <branch>`。缺少 `origin` 或目前 detached HEAD 時停止並回報，不猜測
   遠端或目標分支。
4. commit 或 push 失敗時停止，不回退 archive 或已建立的 commit，不 force-push。下次呼叫
   flow-finish 時第 1 步會選到同名 archive，從本步重試。

### 6. 委派完成整合

push 成功後呼叫 `Skill(superpowers:finishing-a-development-branch)`。它必須重跑完整測試，
再呈現其原生的三個選項：

1. 合併回 `<base_branch>` 本地
2. 推送並建立 PR
3. 保留分支不處理

不得把「丟棄」加回常規選單；只有使用者明確要求丟棄時，才依委派 skill 的 `discard`
確認流程處理。**委派契約**：本地 merge 的 merge 後測試照原 skill 執行，但 branch/worktree
cleanup 必須延後，交由本 skill 在 base branch push 成功後的第 8 步處理。

選本地合併時，委派 skill 在 merge 後測試成功才可繼續。接著 flow-finish 必須：

```bash
git push origin <base_branch>
```

base branch push 失敗時保留本地 merge、分支與 worktree，停止並回報；不得 force-push 或
清理。成功後進入第 8 步。

### 7. PR 後處理

選建立 PR 時，PR 標題與內文一律用繁體中文（zh-TW），但沿用 repo PR 樣板的結構。建立
回傳後保存其 `PR_NUMBER` 與 `PR_URL`；從續跑進入時，使用第 5 步保存的 `PR_NUMBER`。
從 `FLOW_PATH` 取得 `base_branch`；有值時用
`gh pr view <PR_NUMBER> --json number,baseRefName` 確認，不同就以
`gh pr edit <PR_NUMBER> --base <base_branch>` 校正。

PR 建立後依提問協定單選：

> PR 已合併了嗎？要切回 <base_branch> 並 pull 嗎？
> 1. 已合併，切回並 pull
> 2. 尚未合併，留在目前分支（預設）

- 選尚未合併 → 保留 branch/worktree，進入第 9 步。
- 選已合併 → 先以 `gh pr view <PR_NUMBER> --json state` 驗證為 `MERGED`。不是 MERGED 時，
  警告後再讓使用者確認是否繼續；預設停止。
- 驗證後更新 base branch：branch workspace 在目前目錄 `git checkout <base_branch>` 後
  `git pull`；worktree workspace 則在主 repo 目錄 `git pull`，並提示使用者回到主 repo。
  再進入第 8 步。

### 8. 清理已整合的工作區

適用於「本地合併且 base branch 已 push」或「PR 已驗證 MERGED 並已更新 base branch」。
依提問協定單選：

> 要清理本地分支／worktree 嗎？
> 1. 清理
> 2. 保留（預設）

選保留就回報位置。選清理時：

- `workspace: branch` → `git branch -d <branch>`。
- `workspace: worktree` → 先以 `git worktree list --porcelain` 尋找
  `branch refs/heads/<branch>` 對應的唯一 `WORKTREE_PATH`；找不到或有多個時停止，不猜測。
  離開該 worktree 後，在主 repo 依序執行
  `git worktree remove <WORKTREE_PATH>`、`git worktree prune`、`git branch -d <branch>`。

外部 flow-apply worktree 不會被 superpower 自動移除，必須由本步處理。若 Git 拒絕移除
（有未提交或 untracked 檔案），列出 `git -C <WORKTREE_PATH> status --porcelain -uall`，保留
現場並停止；不得自行 `--force`。

### 9. 收尾回報

回報 archive 路徑、commit/push 結果、整合方式、PR URL（如有），以及 branch/worktree
是否已清理或保留。

## NEVER

- **NEVER** 以 `verified_at` hash 差異阻擋、警告或要求重跑驗收。
- **NEVER** 在收尾前把實作或歸屬不明的工作區變更納入 commit。
- **NEVER** 在 tasks/artifacts 未完成時靜默 archive；預設必須停止。
- **NEVER** 在 archive、commit、push 或 base branch push 失敗後繼續整合或清理。
- **NEVER** 使用 `git add .`、force-push 或為了清理而自行 `git worktree remove --force`。
- **NEVER** 在 PR／保留情境清理 worktree；唯一例外是 PR 已驗證 MERGED、base 已更新且
  使用者明確選擇清理。
- **NEVER** 只憑口頭說「已合併」就切回或清理；必須先驗證 PR state 為 `MERGED`。
- **NEVER** 在 `.flow.yaml` 缺 `base_branch` 時報錯；舊 change 缺欄位時略過 PR base
  校正，並由委派 skill 依分支來源確認 base branch。
