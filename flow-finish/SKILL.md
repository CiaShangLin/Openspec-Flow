---
name: flow-finish
description: 使用者手動完成 OpenSpec change、歸檔，並在最後決定本地合併、建立 PR 或保留分支時使用。觸發語句包含「完成這個 change」、「收尾」、「finish」、「準備合併」、「歸檔」或 /flow-finish。
effort: medium
---

# flow-finish：完成 change

手動觸發的最後收尾。提交尚未提交的驗收修正、歸檔並同步 spec、推送分支，最後每次
都由使用者選擇整合方式（本地合併／建立 PR／保留分支）。

開始前先載入 `flow-conventions` skill（以 plugin 安裝時名稱帶前綴，例如 `<plugin>:flow-conventions`）取得共用規範；環境沒有 Skill 工具時，改讀同層目錄的
`../flow-conventions/SKILL.md`，且必須讀完整份。所有決策點依其「提問協定」呈現與解析；
取消、保留與不清理一律是預設。commit 訊息依其「commit 規則」產生，與遠端主機互動依本 skill 的
「遠端主機判斷」一節。

開始前先做 flow-conventions 的「執行前檢查」；未通過就停下並建議 `/flow-init`。

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
`kdoc`→`doc-comment`；見 flow-conventions 的「舊代號相容」）。

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

檢查 `git status --porcelain`。`openspec/changes/<name>/.flow.yaml` 會隨 archive 一起
提交，不算未提交變更。

其餘有 staged、unstaged 或 untracked 檔案時（常見為 flow-verify 修正後尚未提交）：

1. 依 flow-apply「建立實作 checkpoint」的方式，把檔案分為「明確屬於本 change」與「歸屬不明」；
   歸屬不明的檔案逐一依提問協定詢問「納入／排除並保持原狀（預設）」。
2. 有任何檔案被排除 → 列出並停止。帶著未提交檔案切換分支或合併會把它們帶進 base
   branch；使用者處理後重新執行 flow-finish。
3. 全部納入時，列出檔案後依提問協定單選：

   > 還有未提交的變更，要提交後繼續收尾嗎？
   > 1. 提交後繼續  ←預設
   > 2. 停止，我自己處理

4. 選 1 → 精確 stage 這些檔案，依 flow-conventions 的「commit 規則」 commit（意圖：提交 `<name>`
   的驗收修正）。本步不 push，push 在第 5 步。commit 失敗就停止。

不得 stash 或丟棄任何檔案。

### 4. 自動同步與 archive（僅 active change）

先取得 `openspec status --change <name> --json`，並讀 tasks 檔案。從 status 的
`artifactPaths.specs.existingOutputPaths` 取得本 change 的 delta spec，依 capability 映射出
主 spec 路徑 `openspec/specs/<capability>/spec.md`，記為 `SYNCED_SPECS`；任何路徑無法安全
映射時停止，不猜測。

- 任一 artifact 未完成、或 tasks 有 `- [ ]` → 依提問協定單選顯示未完成項目（tasks 逐條列出）：
  1. 仍然同步並歸檔
  2. 剩餘 tasks 已完成（例如已實機測試），幫我勾選並 commit 後繼續收尾
  3. 停止收尾（預設）
  只有明確選 1 或 2 才繼續。
  - 選項 2 只在**未完成項目全是 tasks** 時列出；有 artifact 未完成時不提供（artifact
    缺漏無法靠勾選補上）。
  - 選 2 → 只把上方列出的 `- [ ]` 改成 `- [x]`，不改動其他內容；精確 `git add`
    該 tasks 檔案，依 flow-conventions 的「commit 規則」 commit（意圖：標記 `<name>`
    剩餘 tasks 已完成）。本步不 push，push 在第 5 步。commit 失敗就停止。
    完成後重新讀 tasks 確認已無 `- [ ]`，再往下執行 archive。
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
  不得再次合併。
- 依 「遠端主機判斷」一節判斷遠端主機：
  - **可使用主機 CLI**（GitHub＋`gh`、GitLab＋`glab`）→ 查詢以 `<branch>` 為來源的開啟中
    PR／MR（例如 `gh pr list --head <branch> --state open --json number,state,baseRefName,url`）。
    恰有一個就保存 `PR_NUMBER` 與 `PR_URL` 後進入第 7 步；多個時列出讓使用者選。沒有開啟
    中的才查已合併的；找到已合併者，保存 `PR_NUMBER` 後進入第 7 步同步 base。已關閉但
    未合併的不得阻止建立新的 PR。
  - **其他主機**（Bitbucket、自架等）→ 不呼叫任何主機 CLI。改以
    `git fetch origin <base_branch>` 後 `git merge-base --is-ancestor <branch> origin/<base_branch>`
    判斷：為真代表已在遠端合併，進入第 7 步的「已合併」處理。
- 其他情況 → 目前 `git branch --show-current` 必須等於 `branch`；不相同就停止並顯示應回到的
  branch／worktree，不得 push 目前錯誤的 branch。

只有第三種情況才執行下列 commit/push：

1. 用 `finish.archive_path`、`finish.synced_specs` 與原 active path（若仍在 index）組成唯一可
   stage 的路徑集合。`git status --porcelain` 若出現集合外的檔案就停止。使用精確路徑的
   `git add -A -- <paths>`；不得用 `git add .` 或納入其他變更。
2. 若有 staged 變更，依 flow-conventions 的「commit 規則」 commit（意圖：歸檔 `<name>` 並同步 spec）。
   若沒有 archive／spec 變更可提交，清楚回報，但仍繼續檢查 push。
3. 取得目前分支與 upstream。已有 upstream → `git push`；沒有 upstream 但有 `origin` →
   `git push -u origin <branch>`。缺少 `origin` 或目前 detached HEAD 時停止並回報，不猜測
   遠端或目標分支。**保留 push 輸出**：其他主機建立 PR 時需要其中的網址（第 6.2 節）。
4. commit 或 push 失敗時停止，不回退 archive 或已建立的 commit，不 force-push。下次呼叫
   flow-finish 時第 1 步會選到同名 archive，從本步重試。

### 6. 選擇整合方式

push 成功後，每次都依提問協定單選詢問，不寫入設定：

> 要如何整合 `<branch>`？
> 1. 本地合併回 `<base_branch>`
> 2. 建立 PR
> 3. 保留分支，之後自己處理（預設）

「丟棄」不列入選單；只有使用者明確要求時才依第 6.3 節處理。選 3 → 回報分支與
worktree 位置後進入第 9 步。

#### 6.1 本地合併

1. `base_branch` 缺少（舊 change）→ 依分支來源推測並請使用者確認；不得猜測後直接合併。
2. worktree workspace 時，先回到主 repo 目錄：
   `git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel`。
3. 合併前記錄 base 是否有本分支沒有的提交：
   `git merge-base --is-ancestor <base_branch> <branch>`（為真 ＝ base 沒有新提交）。
4. `git checkout <base_branch>`；有 upstream 時 `git pull --ff-only`。失敗就停止。
5. `git merge <branch>`。發生衝突 → 停止並列出衝突檔案，保留合併中的狀態交由使用者
   處理；不得自動 abort，也不得自行選邊解決。
6. 合併後驗證：
   - 第 3 步為真 → 合併結果的程式碼與分支相同，沿用 flow-verify 的結果，回報
     「base 無新提交，沿用驗收結果」，不重跑測試。
   - 第 3 步為假 → 依 `flow.verify_commands.test`（flow-verify 的 `references/verify-tools.md` 的「`verify_commands`」）執行測試。
     未快取時依同檔的「內省查詢」內省取得指令，依提問協定詢問「執行 `<指令>`／跳過合併後
     測試」；選執行後寫回快取。
   - 測試失敗 → 停止並回報；合併只在本地、尚未推送，保留分支與 worktree，不清理。
7. `git push origin <base_branch>`。失敗時保留本地合併、分支與 worktree，停止並回報；
   不得 force-push 或清理。成功後進入第 8 步。

#### 6.2 建立 PR

分支已在第 5 步推送。依 「遠端主機判斷」一節：

- **可使用主機 CLI** → 以 CLI 建立 PR／MR，目標為 `<base_branch>`。標題與內文一律用
  繁體中文（zh-TW），沿用 repo PR 樣板的結構。保存 `PR_NUMBER` 與 `PR_URL`。
- **其他主機** → 不呼叫任何 CLI。顯示第 5 步 push 輸出中的建立 PR 網址；沒有網址
  （例如本次沒有新提交需要推送）時，請使用者到遠端網頁建立。

接著進入第 7 步。

#### 6.3 丟棄（僅在使用者明確要求時）

列出將被刪除的本地分支、`git log <base_branch>..<branch> --oneline` 的提交與 worktree
路徑，並說明遠端分支不會被刪除。要求使用者輸入 `discard` 確認；收到確認後依第 8 步的
方式移除 worktree，再以 `git branch -D <branch>` 刪除分支。

### 7. PR 後處理

從續跑進入時，使用第 5 步保存的 `PR_NUMBER`。

- **可使用主機 CLI** 且 `FLOW_PATH` 有 `base_branch` → 確認 PR 的目標分支，不同就以 CLI
  校正（例如 `gh pr edit <PR_NUMBER> --base <base_branch>`）。其他主機略過校正。

依提問協定單選：

> PR 已合併了嗎？要切回 <base_branch> 並 pull 嗎？
> 1. 已合併，切回並 pull
> 2. 尚未合併，留在目前分支（預設）

- 選尚未合併 → 保留 branch/worktree，進入第 9 步。
- 選已合併 → 先驗證：可使用主機 CLI 時確認 PR 狀態為已合併（例如
  `gh pr view <PR_NUMBER> --json state` 為 `MERGED`）；其他主機以 「遠端主機判斷」一節的
  `git fetch` ＋ `merge-base --is-ancestor` 驗證。驗證不通過時警告，再讓使用者確認是否
  繼續；預設停止。
- 驗證後更新 base branch：branch workspace 在目前目錄 `git checkout <base_branch>` 後
  `git pull`；worktree workspace 則在主 repo 目錄 `git pull`，並提示使用者回到主 repo。
  再進入第 8 步。

### 8. 清理已整合的工作區

適用於「本地合併且 base branch 已 push」或「PR 已驗證合併並已更新 base branch」。
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

flow-apply 建立的 worktree 一律由本步清理。若 Git 拒絕移除（有未提交或 untracked
檔案），列出 `git -C <WORKTREE_PATH> status --porcelain -uall`，保留現場並停止；不得
自行 `--force`。

### 9. 收尾回報

回報 archive 路徑、commit/push 結果、整合方式、合併後是否重跑測試（或沿用驗收結果）、
PR URL（如有），以及 branch/worktree 是否已清理或保留。

## 遠端主機判斷

flow 不假設遠端是 GitHub。需要和遠端主機互動時，先以
`git remote get-url origin` 判斷主機：

| 主機 | 條件 | PR 相關操作 |
|---|---|---|
| GitHub | URL 含 `github.com` 且 `gh` 可用 | 可用 `gh pr list/create/view/edit` |
| GitLab | URL 含 `gitlab` 且 `glab` 可用 | 可用 `glab mr list/create/view` |
| 其他（Bitbucket、自架等），或上述 CLI 不可用 | — | **不呼叫任何主機 CLI**。建立 PR ＝ push 分支後顯示 `git push` 輸出中的建立 PR 網址；沒有網址時請使用者到遠端網頁建立 |

判斷分支是否已合併時，不依賴主機 CLI：先 `git fetch <remote> <base_branch>`，
再以 `git merge-base --is-ancestor <branch> <remote>/<base_branch>` 判斷。

## NEVER

- **NEVER** 以 `verified_at` hash 差異阻擋、警告或要求重跑驗收。
- **NEVER** 未經確認就把工作區變更納入 commit，或納入歸屬不明的檔案。
- **NEVER** 在 tasks/artifacts 未完成時靜默 archive；預設必須停止。
- **NEVER** 未經用戶選擇「勾選並 commit」就自行把 `- [ ]` 改成 `- [x]`，也不得勾選清單以外的項目。
- **NEVER** 在 archive、commit、push、合併後測試或 base branch push 失敗後繼續整合或清理。
- **NEVER** 使用 `git add .`、force-push 或為了清理而自行 `git worktree remove --force`。
- **NEVER** 寫死 commit 訊息；一律依 flow-conventions 的「commit 規則」。
- **NEVER** 在 GitHub／GitLab 以外的主機，或主機 CLI 不可用時呼叫 `gh`／`glab`。
- **NEVER** 把整合方式寫入設定或替使用者預選；每次都詢問，預設保留分支。
- **NEVER** 自動 abort 合併或自行解決合併衝突。
- **NEVER** 在 PR／保留情境清理 worktree；唯一例外是 PR 已驗證合併、base 已更新且
  使用者明確選擇清理。
- **NEVER** 只憑口頭說「已合併」就切回或清理；必須先依第 7 步驗證。
- **NEVER** 在 `.flow.yaml` 缺 `base_branch` 時報錯；舊 change 缺欄位時略過 PR base
  校正，並依第 6.1 節請使用者確認 base branch。
