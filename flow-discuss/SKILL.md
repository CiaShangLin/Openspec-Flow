---
name: flow-discuss
description: 以 openspec 為基底的需求討論流程。當用戶說「開始需求討論」、「討論一個需求」、「我有個想法」、「討論一個 bug 修復」、「討論重構」或使用 /flow-discuss 時觸發。先分流：模糊需求走 openspec-explore、明確需求走 openspec-propose；討論中偵測到方向發散或方案未被挑戰時，經用戶同意引入 brainstorming 或 grill-me；最終一律產出 openspec 格式文件，且所有文件以繁體中文（zh-TW）撰寫。收尾時詢問用戶「討論完畢或繼續討論」，選擇完畢即 commit + push 討論產物。
---

# flow-discuss：需求討論

以 openspec 為基底的需求討論編排流程。本 skill 只負責分流與編排，
實際討論與文件產出委派給既有 openspec skills。

共用規範（提問協定、工具對照表、錯誤處理原則）見同層目錄下的
`../flow-conventions/conventions.md`（本 skill 與 flow-conventions
須維持同一層級，不論安裝在 Claude Code 的 `.claude/skills/` 或
其他環境的對應 skills 目錄）。

**提問協定的適用邊界（本 skill 特別重要）**：只有**決策點**才套第 0 節的
編號選單格式——開場分流、type 三選一、是否引入 brainstorming／grill-me、
討論是否完畢。**討論本身的來回釐清一律維持自由對話**，不得硬套成選擇題，
否則會毀掉探索深度。

## 流程

### 1. 開場分流

依提問協定（單選）問：

> 「這次要討論的是？」（預設：明確需求開發）
> - **模糊想法／還在探索** → explore 路線
> - **明確需求開發** → propose 路線，type = feature
> - **bug 修復** → propose 路線，type = bug-fix
> - **重構** → propose 路線，type = refactor

propose 路線選定後，將 type 記下，待 change 目錄建立後寫入
`openspec/changes/<name>/.flow.yaml`：

```yaml
type: feature   # 或 bug-fix / refactor
```

**type 限定三選一**：`type` 只能是 `feature`／`bug-fix`／`refactor` 三者之一，
不存在其他分類。提問協定允許用戶用自然語言覆寫，若用戶輸入其他分類
（例如「migrate」、「chore」），**不得**把自由文字原封不動當作 type 寫入
`.flow.yaml`——必須先判斷最貼近上述三者的哪一類，向用戶說明歸類理由並確認後，
才寫入對應的 `feature`／`bug-fix`／`refactor`。

### 2. 委派執行

- **explore 路線**：呼叫 `Skill(openspec-explore)` 進入思考夥伴模式。
- **propose 路線**：呼叫 `Skill(openspec-propose)` 產出
  proposal.md / design.md / tasks.md。

**文件語言規範**：所有討論產物（proposal.md、design.md、tasks.md、
specs 等 openspec artifacts）一律以**繁體中文（zh-TW）**撰寫；
程式碼識別符、指令、檔名等技術名詞維持原文即可。
委派 openspec skills 時須明確傳達此要求。

### 3. 討論中的動態偵測（貫穿全程）

偵測到以下訊號時，先說明理由並建議、**經用戶同意才呼叫**，不自行介入：

| 訊號 | 建議 skill |
|---|---|
| 方向發散、多個方案未收斂、用戶說「不知道怎麼做」 | superpowers:brainstorming |
| 方案已成形但未被挑戰、關鍵決策沒討論過取捨、存在未驗證假設 | grill-me |

建議句式範例：
> 「目前有三個方向還沒收斂，建議用 brainstorming 系統性探索一輪，要嗎？」
> 「方案看起來定了，但快取失效策略還沒被挑戰過，要不要用 grill-me 壓力測試？」

補充討論的結論**一律回填 openspec artifacts**（proposal/design/specs/tasks），
不另立文件——最終輸出格式必須是 openspec 的。

### 4. 收尾

- explore 路線收斂後，主動問：「要轉成正式 proposal 嗎？」
  → 是：接 `Skill(openspec-propose)` 帶入討論結論，並補問 type 寫入 `.flow.yaml`。
- 確認最終產物位於 `openspec/changes/<name>/`。
- 依提問協定（單選）問：

  > 「討論完畢了嗎？」（預設：繼續討論）
  > - **討論完畢** → 進入 commit + push
  > - **繼續討論** → 回到第 3 步

- 選擇**繼續討論** → 回到第 3 步的動態偵測繼續討論，結論一樣回填
  openspec artifacts；之後再次回到本步驟確認，直到用戶選擇討論完畢。
- 選擇**討論完畢** → commit + push 討論產物（只加入該 change 目錄，
  不得使用 `git add -A` 或夾帶其他檔案）：

  ```bash
  git add openspec/changes/<name>/
  git commit -m "docs(openspec): complete discussion for <change-name>"
  git push
  ```

  - explore 路線最終沒有轉成 proposal（未建立 change 目錄）→ 沒有產物
    可 commit，說明後直接結束，不執行 git 指令。
  - push 失敗（權限不足、分支保護規則等）→ 明確報錯並中止，本地 commit
    不回退，提示用戶解決權限問題後自行 `git push`。
- 結尾提示：「準備實作時執行 `/flow-apply <name>`」。

## NEVER

- **NEVER 在討論階段寫實作程式碼**——沿用 explore mode 的限制；用戶要實作時提醒先走 `/flow-apply`。
- **NEVER 未經用戶同意就呼叫 brainstorming 或 grill-me**——先說明理由並取得同意，否則會在用戶不想發散時拖長對話。
- **NEVER 把自由文字直接當作 type 寫入 `.flow.yaml`**——`type` 只能是 `feature`／`bug-fix`／`refactor`，自由文字須先歸類確認，否則會污染分支前綴與後續文件分類。
- **NEVER 把討論本身硬套成編號選單**——提問協定只適用於決策點，需求釐清一律維持自由對話（conventions 第 0.6 節）。
- **NEVER 把補充討論的結論寫成 openspec 以外的獨立文件**——最終輸出格式必須是 openspec artifacts。
- **NEVER 以繁體中文以外的語言撰寫討論產物**——所有 openspec artifacts 一律用 zh-TW 撰寫，技術名詞（程式碼識別符、指令、檔名）維持原文。
- **NEVER 未經用戶確認「討論完畢」就 commit + push**——用戶選「繼續討論」時不得提前 commit，必須等收尾確認通過。
- **NEVER 在收尾 commit 時加入 change 目錄以外的檔案**——只 commit `openspec/changes/<name>/`，避免夾帶工作目錄中不相關的變更。
- **NEVER 靜默跳過 openspec CLI 錯誤**——明確報錯並停下；委派的 skill 不存在時告知後改用退化行為。
