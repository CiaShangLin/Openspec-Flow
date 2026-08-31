---
name: flow-apply
description: Use when 使用者要求開始或繼續實作 OpenSpec change，例如「開始實作」、「實作這個 change」、「apply」或使用 /flow-apply。
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

若 task 不清楚、設計與實作衝突、或遇到未知／不穩定的失敗，先暫停目前 task，
依第 4 節處理；不得跳過 task、猜測修法或提前勾選完成。

### 4. 處理設計偏差與未知異常

#### 4.1 設計偏差：explore → update → 續跑

在**寫 code 前或實作中**發現原本的 design.md／specs／tasks 無法正確指引實作時：

1. 暫停目前 task；不得以繞過規格的方式繼續實作、勾選 task，或只在對話中記住差異。
2. 說明衝突與影響，依提問協定詢問是否先探索並同步規劃（預設：探索並同步）。
3. 使用者同意後呼叫 `Skill(openspec-explore)`，讀取現有 artifacts 與相關程式碼，
   以自由對話釐清可行方案；不得在此階段寫 application code。
4. 使用者確認方向後，詢問是否呼叫 `Skill(openspec-update-change)`。同意時傳入已確認的
   決策差異，讓它逐 artifact 提案並經使用者確認後更新 planning artifacts。
5. 更新完成後重新執行：

   ```bash
   openspec status --change "<name>" --json
   ```

   重新讀取 apply instructions 與 artifacts，再回到 `openspec-apply-change`；tasks
   已調整、缺 artifact、或 change 根本意圖已改變時，不得沿用舊 task 假設。

若使用者不想同步文件，僅可在差異不影響目前 task 正確性時繼續；否則停止，
由使用者選擇同步、另開 change，或使用 `/flow-note` 記錄後結束本次實作。

#### 4.2 未知異常：diagnosing-bugs

下列情況暫停目前 task，呼叫 `Skill(diagnosing-bugs)`：

- 測試或執行結果不穩定（flake）。
- 錯誤原因未知，或已嘗試一次合理修正仍失敗。
- 使用者回報的 bug 無法直接對應到已知 task。
- 效能明顯退化。

先說明觸發訊號，依提問協定詢問是否進行診斷（預設：進行診斷）；使用者同意後才
呼叫 `Skill(diagnosing-bugs)`。

單純語法錯誤、明確的編譯錯誤，或已知且局限於目前 task 的失敗，可直接在 task
內修正，不必啟動完整診斷流程。

委派 `diagnosing-bugs` 時，要求它建立可重現的 feedback loop、確認實際症狀、
提出可證偽假設、驗證根因，並在適用的 seam 補上 regression test；所有指令輸出
與 captured artifacts 都必須先遮蔽敏感資訊。

若無法建立 red-capable feedback loop，或診斷需要使用者提供環境、遮蔽後的 trace／log
或 production instrumentation 授權，明確回報所需項目後暫停本次實作。不得修改產品 code、
勾選 task，或在沒有 feedback loop 的情況下猜測根因。

診斷完成後依根因分流：

- **程式碼仍符合既有規劃** → 修正 code、通過 regression test 與原始 feedback loop，
  再回到 `openspec-apply-change`。
- **規劃錯誤或缺漏** → 走第 4.1 節，探索並更新 artifacts 後續跑。
- **超出目前 change 範圍的架構債或後續工作** → 建議 `/flow-note` 或另開 change；
  不得靜默混入目前 task。

根因確認前不得做投機性的產品 code 修改或勾選 task；可加入有唯一標籤的暫時診斷
instrumentation，但修正後必須清除。必要修正完成且原始 feedback loop 驗證前，不得將
目前 task 標記完成，也不得以偶然一次通過的 flaky 結果宣稱問題已解決。

### 5. 收尾

tasks 全部完成或用戶要求的調整完成、必要檢查通過後，依提問協定（單選）詢問：

> 「所有 tasks 已完成。接下來要怎麼處理？」（預設：本次實作已完成，等待人工 review 後結束）

1. **我有需要調整的部分**
2. **將目前變更依語意分批 commit**
3. **本次實作已完成，等待人工 review 後結束**（預設）

選擇第 3 項時結束 flow-apply，並回報：change 名稱、OpenSpec tasks 已全部完成、
checkpoint 的 commit／push 狀態，以及 branch 或 worktree 位置。接著提示：
「需要驗收時，請另外手動執行 `/flow-verify <name>`。」
`flow-verify` 一律由用戶另外手動觸發；本 skill 不自動呼叫，也不把它列成會直接
進入驗收的選項。`openspec-apply-change` 若回傳 archive 建議，以本節為準，
不得在 apply 收尾引導 archive。用戶在本次 flow-apply 對話中說「可以驗收了」等
自然語言時，只提示 `/flow-verify <name>` 並結束；必須由用戶另行明確呼叫
`/flow-verify <name>` 才開始驗收。

#### 5.1 調整後回到收尾

- 用戶選擇第 1 項或直接用自然語言描述調整 → 完成調整與必要檢查後，重新顯示
  同一個收尾選單。
- 可以反覆「調整 → 人工 review → 再調整」；每次調整或 checkpoint 處理完成後
  都回到收尾選單，直到用戶選擇結束。
- 已建立 checkpoint 後又有修改，後續變更建立新的 commit；不 amend、不 rebase、
  不重寫既有 checkpoint 歷史。

#### 5.2 建立實作 checkpoint

用戶選擇第 2 項即代表授權執行下列 commit 流程，不再逐批等待確認：

1. 先檢查 staged、unstaged、untracked 變更，判斷哪些檔案明確屬於目前 change。
   只納入可確認的檔案；有歸屬不明的變更就停止，逐一依提問協定（單選）詢問
   「納入目前 change／排除並保持原狀（預設）」，不自行處理。
2. 依「可獨立理解與還原的語意變更」分組，OpenSpec task 與模組邊界僅作為
   輔助判斷。執行前先展示每批的檔案、commit message 與分組理由，供用戶掌握，
   但不要求再次確認。
3. 逐批精確 `git add <files>` 並 commit，禁止 `git add .`。任一批失敗就停止，
   保留先前已成功建立的 commits，回報完成到哪一批與原始錯誤；不得宣稱
   checkpoint 完成。
4. commits 全部成功後，依 5.3 詢問是否 push；不因 commit 成功就自動 push。

若沒有新的變更可提交，先明確回報。若目前 branch 有尚未 push 的 commits，仍依
5.3 詢問是否 push；若沒有則回到收尾選單。

#### 5.3 Push checkpoint

詢問 push 前先取得目前 branch、remote/upstream 與將推送的 commit 範圍，並把它們
顯示給用戶：

- 已有 upstream → 以 `<upstream>..HEAD` 取得待推送範圍。
- 沒有 upstream → 以 `.flow.yaml` 的 `base_branch` 為基準取得
  `<base_branch>..HEAD`；缺少 `base_branch` 或該 ref 不存在時停止詢問，不猜測範圍。
- 將推送的範圍混有無法確認歸屬的舊 commits → 停止並詢問，不自行 push。
- 已有 upstream → 依提問協定（單選）詢問「push／暫時不 push」（預設：暫時不 push）。
- 沒有 upstream → 顯示 remote 與 branch，依提問協定（單選）詢問是否執行
  `git push -u <remote> <branch>`（預設：暫時不 push）；remote 或目標 branch
  不明時先停下詢問，不得猜測。
- 用戶同意才執行 push。push 失敗時保留本地 commits，附上原始錯誤並停止；
  不得宣稱遠端 checkpoint 已建立。
- 用戶選擇暫時不 push → 保留本地 commits 並回到同一個收尾選單。
- push 成功後說明「checkpoint 只代表保存目前實作，不代表已通過驗收」，再回到
  同一個收尾選單。

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
- **NEVER 在用戶未選擇第 5.2 節的實作 checkpoint 時自動提交或推送實作變更**——完成 tasks 不等於取得實作 checkpoint 的 git 操作授權；第 2 節首次建立工作區所需的 metadata commit/push 不受此限。
- **NEVER 用 `git add .` 或把歸屬不明的變更納入 checkpoint**——只提交明確屬於目前 change 的檔案。
- **NEVER 在 commit／push 失敗或待推送範圍不明時繼續後續動作**——保留已完成的本地 commits，回報狀態後停止。
- **NEVER amend、rebase 或重寫既有 checkpoint commits**——後續調整一律建立新的 commit。
- **NEVER 在 flow-apply 收尾自動呼叫 flow-verify 或引導 archive**——驗收一律由用戶另外手動觸發 `/flow-verify <name>`。
