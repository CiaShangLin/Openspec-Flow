---
name: flow-conventions
description: flow 系列內部共用規範：提問協定、.flow.yaml 狀態格式、config.yaml 的 flow 設定、執行前檢查、commit 規則與錯誤處理。只在 flow-init、flow-discuss、flow-apply、flow-verify、flow-finish、flow-note 指示時載入，不直接回應使用者請求。
user-invocable: false
---

# Flow Skills 共用規範

flow 系列多個 skill 共用的規則。只屬於單一 skill 的規則放在該 skill 內：
分支命名在 flow-apply，遠端主機與 PR 在 flow-finish，驗收清單、工具委派與
`verify_*` 設定在 flow-verify 的 `references/verify-tools.md`。

## 1. 提問協定（所有 flow skill 適用）

flow 系列會在多種 AI 環境執行（Claude Code、Codex 等），各環境的提問工具能力不同。
本節規範**提問的語意**，各環境依自身能力決定呈現方式，兩者分開。

### 1.1 語意規範（所有環境一致）

任何決策點提問都必須滿足四條：

1. **選項必須列全**——不得丟開放式問題讓用戶自己猜要回什麼
2. **必須標出預設值**——哪些選項是預設，一眼可見
3. **必須聲明單選或多選**
4. **必須允許自然語言覆寫**——用戶可以不照選項回答，直接描述要改的部分

### 1.2 呈現方式（依當前環境的能力判斷）

```
單選，且選項數 ≤ 當前結構化提問工具的實際上限 → 用該工具
其餘情況（多選、超過上限、無此類工具）        → 編號清單
```

**上限不得寫死成數字**：Claude Code 的 AskUserQuestion 上限為 4 個選項，
其他環境的結構化提問工具上限可能更低。一律以「當前工具的實際上限」判斷，
不得把某一環境的限制寫進規範。

編號清單格式：

```
❓ 這次驗收要執行哪些項目？
  1) 單元測試    ←預設
  2) 文件註解    ←預設
  3) code-review ←預設
回覆：數字多選 `1,3`／`0` 套預設／或描述要改的部分
```

### 1.3 回覆語法

| 回覆 | 意義 |
|---|---|
| `2` | 單選，選第 2 項 |
| `1,3` | 多選（全形逗號、頓號、空白分隔皆須接受） |
| `0` | 套用全部預設（別名：`預設`／`default`／`d`） |
| 自然語言 | 描述要改的部分，例如「3 只跑 team-code-review」 |

**為何用 `0` 而非字母**：注音／倉頡等中文輸入法下打英文字母需切換輸入法，
違反「少操作」的目的；`0` 與選項編號同一輸入系統，也不佔用正常選項編號。
多數聊天介面無法送出空訊息，所以不採用「直接 Enter」。

**`0` 的精確語意**：選取所有標記為預設的項目，**不是全選**。

### 1.4 安全預設

涉及**破壞性操作**的問題（丟棄分支、覆蓋既有檔案、刪除 worktree 等），
預設值只能是保留／取消／停止這類安全選項，不得把破壞性操作納入 `0` 的範圍。

### 1.5 邊界情況

回覆為非法編號、超出範圍、或數字與自然語言混雜且語意衝突時，
**重問一次並指出問題所在**，不得自行猜測用戶意圖。

### 1.6 適用邊界

本協定**只適用於決策點**——選項可窮舉、且答案會改變接下來行為的提問。

| 適用（套協定） | 不適用（維持自由對話） |
|---|---|
| flow-discuss：建議路線是否採用、type 三選一、artifact review loop（定稿／釐清／壓力測試／重新評估）、文件 commit 後是否 push | 「這個功能要解決什麼問題？」等需求釐清、explore／架構評估／grill-me 內部的追問 |
| flow-apply：branch／worktree、worktree 建立後如何進入（同 session 切過去／新開 session）、分支已存在怎麼辦、是否同步文件、tasks 完成後要調整／建立 checkpoint／結束、commit 後是否 push、無 upstream 時是否設定並 push | 人工 review 後要調整的具體內容 |
| flow-verify：開場項目、找不到工具時的二選一 | 需求細節的來回釐清 |
| flow-finish：驗收缺項是否繼續、未提交變更是否提交後繼續、未完成 tasks/artifacts 是否歸檔、合併／PR／保留、是否已合併、是否清理 | |
| flow-init：是否執行 `openspec init`、要設定的 AI 工具、設定差異套用方式、commit 格式確認、選用 setup 是否執行、是否提交 | `context` 內容的來回補充 |

**NEVER 把自由討論硬套成編號選單**——那會毀掉 flow-discuss 的探索深度。

## 2. `.flow.yaml` 結構

每個 change 目錄一份（`openspec/changes/<name>/.flow.yaml`），跨階段傳遞狀態：

```yaml
type: feature          # feature | bug-fix | refactor（flow-discuss 寫入）
workspace: branch      # branch | worktree（flow-apply 寫入）
branch: feature/cloud-record-page   # flow-apply 寫入
base_branch: main      # flow-apply 寫入，這個 change 實際分支出去的來源分支
verify_selected:        # flow-verify 開場詢問後寫入，記錄本次勾選了哪些項目
  - unit-test
  - doc-comment
  - code-review
verified:               # flow-verify 逐項完成後寫入
  - unit-test
  - doc-comment
  - code-review
verify_progress:        # flow-verify 每次寫入或清除 verified 時更新，供中斷後接續
  head: <git rev-parse HEAD 的值>
  diff_hash: <「verify_progress 的中斷接續」定義的快照雜湊>
verified_at:            # flow-verify 全部完成時寫入，保留為驗收紀錄
  head: <git rev-parse HEAD 的值，作為 diff 的比較基準>
  diff_hash: <git diff <head> | git hash-object --stdin 的結果，涵蓋當時未提交的變更內容>
finish:                 # flow-finish archive 後寫入，支援 commit/push/整合續跑
  archive_path: openspec/changes/archive/YYYY-MM-DD-<change-name>
  synced_specs:         # 本次 archive 實際同步的主 spec 路徑
    - openspec/specs/<capability>/spec.md
```

### 2.1 舊代號相容

`junit` → `unit-test`、`kdoc` → `doc-comment` 為代號改名。舊 change 檔案不主動改寫、
不要求遷移，依位置分兩類處理：

- **值 alias**（`verify_selected`／`verified`）：讀取時接受新舊兩種值，
  寫回時一律只寫 canonical（新）值。
- **key alias**（`openspec/config.yaml` 的 `verify_tools.kdoc` → `verify_tools.doc_comment`）：
  見 flow-verify 的 `references/verify-tools.md` 的「新舊 key 碰撞規則」的碰撞規則。

**比較前必須先 normalize**——先把讀到的值轉成 canonical 代號，再與需求集合比較：

```
normalized = normalize_aliases(verified)      # junit→unit-test, kdoc→doc-comment
required   = {unit-test, doc-comment, code-review}
missing    = required - normalized
```

否則會發生「先轉成新代號、卻拿舊代號集合去比對」導致三項永遠被判定為缺少。

### 2.2 缺欄位的處理

- 缺 `base_branch`／`verified_at` → 略過對應檢查，不報錯。
- 缺 `verify_selected`（優化前建立的舊 change）→ fallback 為檢查 `verified`
  （normalize 後）是否缺 `unit-test`／`doc-comment`／`code-review` 任一項，
  缺任一項時由 flow-finish 問是否繼續，預設停止、可明確覆寫。

### 2.3 `verified` 的本輪邊界

`verified` **每輪重建、不累積**：flow-verify 開場寫入 `verify_selected` 時，
一併清除整個 `verified`。唯一例外是「`verify_progress` 的中斷接續」：上次中斷後
程式碼完全沒變，且使用者選擇接續時，沿用原本的 `verify_selected` 與 `verified`。

否則會出現：第一次三項全過 → 程式碼又改了 → 第二次只勾 code-review 但中途中斷 →
舊的 `verified` 仍在 → flow-finish 判定驗收完成而放行一個已失效的 change。

若「單元測試」因環境限制（如無 Xcode）只完成部分平台，仍整項記入 `verified`，
flow-verify 的收尾摘要須附註警告說明未執行的平台與原因（見 flow-verify 的 `references/verify-tools.md` 的「環境能力限制」），不得
只寫「完成」誤導人工驗收。

### 2.4 `verified_at` 的驗收快照

flow-verify 完成時保存 `head` 與相對該 commit 的 `diff_hash`，讓人工需要時可以
追查驗收當下的內容。**flow-finish 不比較這兩個值**：手動呼叫 finish 即表示使用者
接受驗收結果，不能因內容雜湊不同額外打斷收尾。

### 2.5 `verify_progress` 的中斷接續

flow-verify 每次寫入或清除 `verified` 時，同時更新 `verify_progress`，記下當下的
程式碼快照。下次開場時，若上次那輪沒做完，且目前快照與 `verify_progress` 完全相同，
代表已完成的項目仍然有效，可以只跑剩下的項目。

快照雜湊涵蓋已追蹤檔案的變更，以及未追蹤的新檔案（例如剛補上的測試檔），
但排除 `.flow.yaml` 本身，否則寫入進度就會讓快照失效：

```bash
F=<FLOW_PATH，即 .flow.yaml 相對 repo 根目錄的路徑>
{ git diff HEAD -- . ":(exclude)$F"
  git ls-files -o --exclude-standard -- . ":(exclude)$F"
  git ls-files -o --exclude-standard -z -- . ":(exclude)$F" | xargs -0 git hash-object
} | git hash-object --stdin
```

`head` 或雜湊任一不同、或缺 `verify_progress`，一律視為進度失效，照一般流程重新開始。
flow-finish 不讀這個欄位。

## 3. 專案設定（`openspec/config.yaml` 的 `flow:` 區塊）

跟著專案走、不隨 skill 更新。所有 flow 設定只放這一個檔案，不得複製到各 change
的 `.flow.yaml` 或 artifacts。寫入時機分兩種：

| 欄位 | 誰寫入、何時 |
|---|---|
| `commit`（「commit 訊息格式」） | flow-init。discuss 是第一個需要 commit 的步驟，早於其他會詢問的流程 |
| `worktree_base` | flow-apply 首次選 worktree 時詢問後寫入 |
| `verify_tools`／`verify_commands`／`verify_notes` | flow-verify／flow-finish 首次用到時詢問後寫入；flow-init 不預先設定 |

整合方式（本地合併／PR／保留）**不寫入設定**，每次由 flow-finish 詢問。

`verify_tools`／`verify_commands`／`verify_notes` 的欄位定義見 flow-verify 的 `references/verify-tools.md` 的「驗收相關設定」；
`worktree_base` 的推斷方式見 flow-apply。

### 3.1 `commit`：commit 訊息格式

由 flow-init 偵測後寫入，所有 flow 的 commit 依「commit 規則」使用：

```yaml
flow:
  commit:
    skill: <commit skill 名稱>   # 有值：委派該 skill 產生並執行 commit；沒有則省略
    format: |                   # 沒有 skill 時使用：格式規則與至少一個範例
      第一行：<type>(<scope>): <摘要>
      範例：docs(openspec): complete discussion for add-login
```

`skill` 與 `format` 至少要有一個；兩者都有時以 `skill` 為主，`format` 作為該
skill 不可用時的備援。

## 4. 執行前檢查（preflight）

flow-discuss／flow-apply／flow-verify／flow-finish／flow-note 開始時先做以下檢查。
**只檢查、不修復**：任一項不通過就停下，說明原因並建議執行 `/flow-init`，
不得自行執行 `openspec init`、手動建立 `openspec/` 或寫入設定。

1. `openspec --version` 可執行。
2. `openspec list --json` 的 `root` 不為 `null`。
   - `root` 為 `null` 且錯誤訊息以 `Declared in` 或 `Invalid store declaration in`
     開頭 → 專案透過 store 使用 OpenSpec 但此機器無法解析：顯示該錯誤的
     `message` 與 `fix` 後停下，**不要**建議 `/flow-init`。
3. `openspec/config.yaml` 有 `flow.commit`（「commit 訊息格式」）。flow-verify 不會 commit，
   可略過本項。
4. 依賴的 skill 只在**即將使用時**才檢查（例如 discuss 走架構評估前才確認
   `improve-codebase-architecture` 存在），不在開場一次掃完。

必要依賴：

| 依賴 | 用途 | 安裝 |
|---|---|---|
| `openspec` CLI | 全部 | `npm install -g @fission-ai/openspec@latest` |
| `openspec-explore`／`openspec-propose`／`openspec-apply-change`／`openspec-update-change`／`openspec-archive-change` | discuss、apply、finish | `openspec init`（core profile）會安裝；缺少時 `openspec update` |
| `grill-me`／`improve-codebase-architecture`／`diagnosing-bugs`／`code-review` | discuss、apply、verify | Matt Pocock skills。Claude Code：`claude plugin install mattpocock-skills@claude-plugins-official`；其他環境：`npx skills@latest add mattpocock/skills` |

選用依賴（缺少不阻擋）：Matt 的 `setup-matt-pocock-skills`（提供 `/code-review`
讀取的 issue tracker 設定）、任何可建立 skill 的工具（見 flow-verify 的 `references/verify-tools.md` 的「委派工具」）。

flow 不依賴 superpowers。

openspec-* 與 Matt 的 skill 是**必要依賴**，由 flow-init 確認已安裝，
因此不提供退化行為；執行中發現缺少時依「錯誤處理原則」停下。

## 5. commit 規則

所有 flow 產生的 commit（discuss 的文件提交、apply 的 metadata 與 checkpoint、
finish 的收尾提交、flow-init 的初始化提交）一律依本節決定訊息，**不得寫死任何
commit 訊息字串**。

1. **先精確 stage**：只 `git add -- <明確路徑>`，禁止 `git add .`。
2. **依 `flow.commit` 產生訊息**：
   - 有 `skill` 且環境可用 → 委派該 skill，並告知：已 stage 的路徑、本次 commit 的
     意圖摘要（例如「記錄 <change> 的工作區 metadata」）、不得再 stage 其他檔案、
     不得 push。
   - 否則依 `format` 自行組出訊息。
   - 兩者都沒有 → 停下並建議 `/flow-init`，不得自行猜格式。
3. **hook 拒絕時**：顯示 hook 的原始輸出，依其指出的問題修正訊息**重試一次**；
   仍失敗就停下回報。不得使用 `--no-verify` 或其他繞過 hook 的方式。
4. **commit 不等於 push**：是否 push 由各 skill 的決策點決定。

## 6. 錯誤處理原則

- openspec CLI 不存在或指令失敗 → 明確報錯並停下，不靜默跳過。
- 必要依賴的 skill 不存在 → 停下，說明缺少哪個 skill 與安裝方式，建議執行
  `/flow-init` 重新檢查；不得自行以退化行為代替。
- flow-apply 的 metadata push 失敗（權限不足、分支保護規則等）→ 視為錯誤並中止，
  不建立分支/worktree、不靜默略過、不自動改用其他方式繞過。
