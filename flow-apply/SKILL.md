---
name: flow-apply
description: 依 openspec change 開始實作。當用戶說「開始實作」、「實作這個 change」、「apply」或使用 /flow-apply 時觸發。先依 change 類型建立符合命名規範的分支（feature/、bug-fix/、refactor/ 前綴），詢問使用 branch 或 worktree，選 worktree 會再問要同 session 切過去（預設）還是新開 session 切過去，再委派 openspec-apply-change 執行 tasks；實作中若發現 bug 或跟原規劃不符並已與用戶討論調整，會詢問是否委派 openspec-update-change 同步文件。
---

# flow-apply：建立工作區並實作

在 openspec-apply-change 之前加上工作區準備：依類型命名分支、
選擇 branch 或 worktree、決定如何進入 worktree。

共用規範（提問協定、分支命名、專案設定、工具對照表）見同層目錄下的
`../flow-conventions/conventions.md`（本 skill 與 flow-conventions
須維持同一層級，不論安裝在 Claude Code 的 `.claude/skills/` 或
其他環境的對應 skills 目錄）。**本 skill 的所有決策點提問一律依第 0 節
提問協定呈現與解析**：選項列全、標出預設、聲明單／多選、允許自然語言覆寫。

## 流程

### 1. 選定 change 並取得類型

- 有指定 change 名稱就用它；否則：單一 active change 自動選、
  多個則 `openspec list --json` 列出讓用戶挑。
- 宣告：「Using change: <name>」。
- 讀 `openspec/changes/<name>/.flow.yaml` 的 `type` 決定分支前綴
  （對照 conventions.md 第 1 節）。
- `.flow.yaml` 不存在或缺 `type` → 依提問協定問一次「這個 change 是
  feature / bug-fix / refactor？」並回填。`type` 只能是這三者之一，
  用戶若用自然語言輸入其他分類（如「migrate」、「chore」），須先歸類確認到
  最貼近的一種才能寫入，不得原封不動採用自由文字。

### 2. 工作區選擇

前置檢查（任一不通過就停下詢問，不自行處理）：

1. 目前已在該 change 的分支上？→ 是則跳過本節其餘步驟，直接到步驟 3。
2. 分支名已存在？→ 問要沿用該分支還是換名。
3. 工作目錄有未提交變更？→ 停下詢問（commit / stash / 放棄由用戶決定）。
   這項檢查在下面的 metadata commit 步驟前必須通過，工作目錄要保持乾淨。

冪等性檢查（處理「上次 push 失敗中止後重跑」的情況，僅在前置檢查都通過後執行）：

- 讀 `openspec/changes/<name>/.flow.yaml`，若已經同時存在 `workspace`、
  `branch`、`base_branch` 三個欄位 → 代表 metadata 已寫入過，跳過下方
  「決定工作區」與「寫入 metadata 並 commit + push」兩步，直接檢查是否已 push：
  ```bash
  git log origin/<base_branch>..<base_branch>
  ```
  - 輸出為空（已 push）→ 直接跳到「建立分支/worktree」步驟。
  - 輸出不為空（尚未 push）→ 重新執行 `git push`，失敗一樣視為錯誤中止
    （處理方式同下方「寫入 metadata 並 commit + push」步驟 4）。

決定工作區（僅首次執行，跳過冪等性檢查判定為已完成的情況）：

依提問協定（單選）問：

> 「要用 branch 還是 worktree？」（預設：branch）

- **branch**：記下目標分支名 `<前綴>/<change-name>`（先不執行 `git checkout -b`，
  留到下方「建立分支/worktree」步驟）。
- **worktree**：
  1. 讀 `openspec/config.yaml` 的 `flow.worktree_base`。
  2. 未設定 → 依當前專案目錄名稱推斷預設建議值
     （`../<當前專案目錄名稱>-worktrees/`，不可寫死任何專案名稱），
     問一次是否採用此建議或改用其他路徑，答案寫入 `openspec/config.yaml`
     （先不 commit，留到下方跟 `.flow.yaml` 一起 commit）：
     ```yaml
     flow:
       worktree_base: ../<當前專案目錄名稱>-worktrees
     ```
     之後不再詢問。
  3. 記下目標路徑 `<worktree_base>/<change-name>` 與分支名
     `<前綴>/<change-name>`（先不執行 `git worktree add`）。

寫入 metadata 並 commit + push（僅首次執行）：

1. 取得 `base_branch`＝目前所在分支名稱：
   ```bash
   git branch --show-current
   ```
2. 把以下欄位寫入 `openspec/changes/<name>/.flow.yaml`：
   ```yaml
   workspace: branch   # 或 worktree
   branch: feature/cloud-record-page
   base_branch: main
   ```
3. Commit（若上一步驟有異動 `openspec/config.yaml` 一併加入；若沒有異動，
   只 add `.flow.yaml` 即可）：
   ```bash
   git add openspec/changes/<name>/.flow.yaml openspec/config.yaml
   git commit -m "chore: record flow workspace metadata for <change-name>"
   ```
4. Push：
   ```bash
   git push
   ```
   - **push 失敗（權限不足、分支保護規則禁止直接 push 等）→ 視為錯誤，明確
     報錯並中止整個 flow-apply**，不建立分支/worktree。錯誤訊息附上 git 的
     原始輸出，並提示：「請解決來源分支的 push 權限問題後重新執行
     `/flow-apply <name>`」。本地 commit 不回退——這是無害的紀錄性 commit，
     下次重跑會被上方冪等性檢查偵測到並跳過重複 commit。

建立分支/worktree（push 成功後才執行，或冪等性檢查判定已 push 後執行）：

- **branch**：
  ```bash
  git checkout -b <前綴>/<change-name>
  ```
- **worktree**：先確認是否已存在（處理「選了新開 session、卻又在主 repo 重跑」
  的情況）：
  ```bash
  git worktree list
  ```
  - 已列出 `<worktree_base>/<change-name>` 且掛在目標分支上 → **跳過建立**，
    直接進入下方「進入 worktree」。
  - 尚未存在 → 建立：
    ```bash
    git worktree add <worktree_base>/<change-name> -b <前綴>/<change-name>
    ```
  - 路徑已存在但未註冊在 `git worktree list`（殘留目錄）→ 依 NEVER 停下詢問，
    不自行刪除。

進入 worktree（僅 worktree，建立成功或確認已存在後執行）：

依提問協定（單選）問：

> 「要怎麼進入這個 worktree？」（預設：同 session 切過去）

- **同 session 切過去**（預設）：用當前環境的 worktree 切換機制把工作目錄切到
  `<worktree_base>/<change-name>`（對照 conventions.md 第 6 節；無此機制的環境
  退化為 `cd <路徑>`），切換後必須以 `pwd` 確認確實在該 worktree 目錄，
  確認通過才進入步驟 3。
  - 切換失敗、或當前環境的工作目錄被鎖在原專案目錄（例如已固定 cwd 的子代理）
    → 不硬闖、不在原目錄開始實作，改依下方「新開 session」輸出接續指示並結束。
- **新開 session**：**flow-apply 到此結束**，不進入步驟 3。輸出接續指示：
  ```
  worktree 已建立：<worktree_base>/<change-name>
  分支：<前綴>/<change-name>
  請在該目錄開新 session，執行 `/flow-apply <change-name>` 繼續實作
  ```
  新 session 重跑時，前置檢查第 1 項會偵測到已在該 change 的分支上，直接跳到
  步驟 3，不會重複建立分支或 worktree。

### 3. 委派實作

呼叫 `Skill(openspec-apply-change)` 執行 tasks，沿用其任務勾選與進度追蹤
（退化行為：照 `openspec/changes/<name>/tasks.md` 逐項實作並勾選）。

### 4. 實作中發現偏差時同步文件

實作過程中若發現 bug、或做法跟原本 design.md／tasks.md 不符，與用戶討論
並確定調整方向、且已實際修改 code 之後，主動詢問用戶是否要委派
`Skill(openspec-update-change)` 同步更新該 change 的 openspec 文件
（design.md／specs／tasks.md），避免文件與實作結果脫節。

- 用戶同意 → 呼叫 `Skill(openspec-update-change)`，帶上剛才討論確認的異動
  內容，讓它逐一提案修改並經確認後寫入。
- 用戶婉拒 → 記下差異但不強制阻擋，繼續原本任務即可；差異若複雜或用戶
  想留到之後處理，可改用 `/flow-note` 記錄。

### 5. 收尾

tasks 全部完成時提示：「執行 `/flow-verify <name>` 進行驗收」。

## NEVER

- **NEVER 在有未提交變更的工作目錄直接建分支**——先停下詢問（commit / stash / 放棄由用戶決定），否則變更會被帶到新分支造成混淆。
- **NEVER 自動刪除或覆蓋既有分支與 worktree 目錄**——名稱衝突一律先問。
- **NEVER 猜測 change 類型**——`.flow.yaml` 缺 `type` 就問，猜錯前綴會污染分支歷史。
- **NEVER 丟開放式問題取代決策點提問**——選項必須列全並標出預設，見 conventions 第 0 節。
- **NEVER 靜默跳過 git 指令失敗**（分支衝突、worktree 目錄已存在等）——明確報錯並停下。
- **NEVER 在 metadata push 失敗時仍繼續建立分支/worktree**——push 失敗一律視為錯誤中止，避免 metadata 只存在本地、跟遠端脫節。
- **NEVER 在 worktree 建立後直接假設進入方式**——一律問「同 session 切過去／新開 session」，不自行 cd 也不只丟一句提醒了事。
- **NEVER 在工作目錄尚未實際切到 worktree 時就開始實作**——切換後要用 `pwd` 驗證；切不過去就改走「新開 session」，否則會在原專案目錄改檔案，污染來源分支。
- **NEVER 未經詢問就自行呼叫 openspec-update-change 修改文件**——即使發現實作跟原規劃有落差，也要先讓用戶決定是否同步，不能自作主張改文件。
