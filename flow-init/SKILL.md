---
name: flow-init
description: 在專案導入或重新檢查 flow 系列時使用。當用戶說「初始化 flow」、「導入 flow」、「設定 openspec」、「新專案要用 flow」、「檢查 flow 依賴」或使用 /flow-init 時觸發。依序檢查 openspec CLI 與必要 skills、必要時執行 openspec init，草擬 config.yaml 的 context／rules／operations 與 flow.commit，列出現況與建議的差異後才寫入；可重複執行，不覆蓋使用者手改的設定。
effort: medium
---

# flow-init：導入與重新檢查

flow 系列的初始化入口。先確認依賴，再讓 `openspec/config.yaml` 具備 flow 需要的
專案設定。新專案從頭建立；既有專案只補缺的部分。

共用規範見同層目錄下的 `../flow-conventions/conventions.md`：

- 第 0 節：提問協定（所有決策點依此呈現與解析）
- 第 4 節：`flow:` 區塊的欄位與寫入時機（本 skill 只寫 `flow.commit`）
- 第 8 節：必要／選用依賴與安裝方式
- 第 9 節：commit 規則

## 核心原則

- **先看現況，再提案，確認後才寫。** 從 repo 能觀察到的不問使用者；只問觀察不到的。
- **可重複執行。** 重跑時只列出差異；使用者手改過的內容一律保留，不整段覆寫。
- **只寫該寫的。** `worktree_base`、`verify_tools`、`verify_commands` 由其他 flow 在
  首次用到時詢問，本 skill 不預先設定；整合方式（本地合併／PR／保留）不寫入設定。

## 流程

### 1. 依賴檢查（遇到第一個缺少的必要項目就停下）

依序檢查，缺少時顯示安裝方式並停止，使用者處理後重新執行 `/flow-init`：

1. `git rev-parse --show-toplevel` 成功——flow 依賴 git 分支與 commit；不在 git repo
   時停下，不自行 `git init`。
2. `openspec --version` 可執行。缺少時提示
   `npm install -g @fission-ai/openspec@latest`。
3. 其餘必要 skill（openspec-* 與 Matt 的 skills）在第 2 步完成後才檢查，因為
   `openspec init` 會安裝 openspec-* skills。

### 2. OpenSpec 初始化

執行 `openspec list --json` 讀取 `root`：

- `root` 不為 `null` → 已初始化，跳到第 3 步。
- `root` 為 `null`，且錯誤訊息以 `Declared in` 或 `Invalid store declaration in` 開頭
  → 專案透過 store 使用 OpenSpec：顯示 `message` 與 `fix` 後停止，不得執行 init。
- `root` 為 `null` → 依提問協定確認是否初始化，並一併確認要設定的 AI 工具
  （依 repo 內既有的 `.claude/`、`.codex/`、`.cursor/` 等目錄與目前環境推斷預設，
  多選）。確認後執行：

  ```bash
  openspec init --tools <工具清單> --language zh-TW
  ```

  `--language` 只在新建 `config.yaml` 時生效，會在 `context` 開頭寫入語言要求。
  init 失敗時保留現場並停止。

初始化後檢查必要 skills（conventions 第 8 節）：

- openspec-explore／propose／apply-change／update-change／archive-change 缺少 →
  多半是 profile 不是 core：提示 `openspec config profile` 選 core 後執行
  `openspec update`，停止。
- Matt 的 `grill-me`／`improve-codebase-architecture`／`diagnosing-bugs`／`code-review`
  缺少 → 顯示 conventions 第 8 節的安裝指令，停止。

以「目前環境可用的 skill 清單」判斷是否存在，不依賴固定安裝路徑。

### 3. 草擬 `config.yaml` 的建議內容

讀取現有的 `openspec/config.yaml`，對以下每一塊判斷「已有／缺少／可補強」。

**`context`**（OpenSpec 建立 artifacts 時都會帶入，上限 50KB）。從 repo 觀察草擬，
依序取材：README、CLAUDE.md／AGENTS.md、建置設定（Gradle、package.json、
Cargo.toml 等）、模組或套件結構、測試設定、CI 設定。建議章節：

```
## 專案概覽      名稱、用途
## 技術棧        語言與版本、建置工具、架構、網路、資料庫、非同步
## 模組結構      各模組用途，以及各自的編譯／測試指令
## 架構慣例      新程式碼的寫法慣例
## 測試規範      測試框架、哪些要測、哪些預設不測
## 環境限制      例如需要內網才能建置
## 團隊慣例      語言、commit 規範、文件放置位置
```

- 既有 `context` 已有內容 → 不重寫，只列出缺少的章節供補充。
- 沒有語言要求（新專案由 `--language` 寫入；既有專案可能沒有）→ 建議補上
  `All artifacts must be written in zh-TW.`。
- 觀察不到的資訊（例如環境限制）列為待補問題，以自由對話詢問，不硬套選單。

**`rules`**（依 artifact 分開）。缺少時建議：

```yaml
rules:
  proposal:
    - 必須包含「非目標 (Non-goals)」章節，明確界定範圍
    - 說明受影響的模組與範圍
  design:
    - 涉及資料結構變更時，須說明遷移策略
    - 引入新依賴時，須說明選型理由
  specs:
    - 需求採 SHALL/MUST 語氣，並附至少一個 Scenario
  tasks:
    - 每個 task 控制在約 2 小時內可完成的粒度
    - 邏輯層變更須包含對應的測試 task
    - 最後一個 task 包含 lint 與編譯驗證
```

依第 3 步觀察到的技術棧調整措辭（例如把「lint」寫成實際工具名）。

**`operations`**（OpenSpec 只接受 `apply` 與 `archive`，欄位只有 `guidance`）。缺少時建議：

```yaml
operations:
  apply:
    guidance:
      - 先跑受影響範圍的測試，再跑完整測試
      - 除非 task 或使用者要求，不主動安裝或啟動應用程式
  archive:
    guidance:
      - 完成摘要保持簡短
```

### 4. 偵測 commit 格式（`flow.commit`）

已有 `flow.commit` 就沿用，只在偵測結果明顯不同時列為差異。缺少時依序偵測：

1. 專案說明指定的 commit 方式：CLAUDE.md／AGENTS.md，或 `config.yaml` 的
   `context` 中提到的 commit skill 或規範 → 取為 `skill` 或 `format`。
2. 目前環境中名稱或描述與 commit 相關的 skill → 列出讓使用者選，選定後寫入 `skill`。
3. 都沒有 → 讀 `.git/hooks/commit-msg`（及 husky、commitlint 等設定）與
   `git log -20 --format=%s`，歸納出格式規則與一個範例，寫入 `format`。

偵測結果須讓使用者確認；推斷不出格式時，以自由對話請使用者描述。

### 5. 呈現差異並寫入

以一張表列出每一項的「現況 → 建議」，未變更的項目只列一行 ✅：

```
❓ flow-init 檢查結果（<repo 名稱>）
  ✅ openspec 已初始化（root: openspec/）
  ✅ 必要 skills 齊全
  ➕ context：補上「測試規範」「環境限制」兩節
  ➕ operations.apply.guidance：新增 2 條
  ➕ flow.commit.skill：<偵測到的 skill>（來源：context 第 N 行）
  1) 全部套用  ←預設
  2) 逐項選擇
  3) 都不套用
回覆：`0` 套用預設／數字／或自然語言覆寫
```

- 寫入時只改動被選中的區塊，保留檔案中其他 key、註解與順序；`flow:` 區塊內
  只新增或修改 `commit`，不動既有的 `worktree_base`、`verify_*`。
- 寫入後以 `openspec list --json` 確認設定仍可被正確解析；失敗時回報錯誤並還原
  本次寫入。

### 6. 選用項目（各詢問一次）

- 已安裝 Matt 的 `setup-matt-pocock-skills`，且專案沒有 `docs/agents/` →
  詢問是否現在執行（`/code-review` 會讀取它產生的 issue tracker 設定）。預設：不執行。
- 環境中沒有可建立 skill 的工具 → 只告知一行：之後 flow-verify 找不到驗收工具時，
  只能以文字說明應急。不詢問、不阻擋。

### 7. 提交與回報

有寫入檔案時，依提問協定詢問是否提交（預設：暫不提交，讓使用者先檢視 diff）。
同意時依 conventions 第 9 節，只 stage `openspec/` 下本次新增或修改的檔案與 init
產生的工具設定檔。**不得 push。**

最後回報：依賴檢查結果、寫入了哪些區塊、尚待使用者補充的 `context` 問題，以及
下一步提示：「準備規劃需求時，執行 `/flow-discuss`」。

## NEVER

- **NEVER 在必要依賴缺少時繼續後續步驟**——停下並顯示安裝方式。
- **NEVER 未經確認就執行 `openspec init`**，或手動建立 `openspec/` 目錄與設定檔。
- **NEVER 對透過 store 使用 OpenSpec 的專案執行 init**——顯示錯誤的 `fix` 後停止。
- **NEVER 整段覆寫既有的 `context`、`rules` 或 `flow:` 區塊**——只補差異，保留使用者手改的內容。
- **NEVER 預先寫入 `worktree_base`、`verify_tools`、`verify_commands` 或整合方式**——由各 flow 在用到時詢問。
- **NEVER 在 `operations` 下寫入 `apply`、`archive` 以外的 key，或 `guidance` 以外的欄位**——OpenSpec 會拒絕解析。
- **NEVER 寫死任何 commit 訊息或 commit skill 名稱**——一律現場偵測並經使用者確認。
- **NEVER push**；提交也預設不做，交由使用者決定。
