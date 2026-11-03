# Flow Skills 共用規範

flow-init / flow-discuss / flow-apply / flow-verify / flow-finish（及 flow-note）的單一規範來源。
規範調整時只改此檔，所有 flow skill 同步生效。

## 0. 提問協定（所有 flow skill 適用）

flow 系列會在多種 AI 環境執行（Claude Code、Codex 等），各環境的提問工具能力不同。
本節規範**提問的語意**，各環境依自身能力決定呈現方式，兩者分開。

### 0.1 語意規範（所有環境一致）

任何決策點提問都必須滿足四條：

1. **選項必須列全**——不得丟開放式問題讓用戶自己猜要回什麼
2. **必須標出預設值**——哪些選項是預設，一眼可見
3. **必須聲明單選或多選**
4. **必須允許自然語言覆寫**——用戶可以不照選項回答，直接描述要改的部分

### 0.2 呈現方式（依當前環境的能力判斷）

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

### 0.3 回覆語法

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

### 0.4 安全預設

涉及**破壞性操作**的問題（丟棄分支、覆蓋既有檔案、刪除 worktree 等），
預設值只能是保留／取消／停止這類安全選項，不得把破壞性操作納入 `0` 的範圍。

### 0.5 邊界情況

回覆為非法編號、超出範圍、或數字與自然語言混雜且語意衝突時，
**重問一次並指出問題所在**，不得自行猜測用戶意圖。

### 0.6 適用邊界

本協定**只適用於決策點**——選項可窮舉、且答案會改變接下來行為的提問。

| 適用（套協定） | 不適用（維持自由對話） |
|---|---|
| flow-discuss：建議路線是否採用、type 三選一、artifact review loop（定稿／釐清／壓力測試／重新評估）、文件 commit 後是否 push | 「這個功能要解決什麼問題？」等需求釐清、explore／架構評估／grill-me 內部的追問 |
| flow-apply：branch／worktree、worktree 建立後如何進入（同 session 切過去／新開 session）、分支已存在怎麼辦、是否同步文件、tasks 完成後要調整／建立 checkpoint／結束、commit 後是否 push、無 upstream 時是否設定並 push | 人工 review 後要調整的具體內容 |
| flow-verify：開場項目、找不到工具時的二選一 | 需求細節的來回釐清 |
| flow-finish：驗收缺項是否繼續、未提交變更是否提交後繼續、未完成 tasks/artifacts 是否歸檔、合併／PR／保留、是否已合併、是否清理 | |
| flow-init：是否執行 `openspec init`、要設定的 AI 工具、設定差異套用方式、commit 格式確認、選用 setup 是否執行、是否提交 | `context` 內容的來回補充 |

**NEVER 把自由討論硬套成編號選單**——那會毀掉 flow-discuss 的探索深度。

## 1. 分支命名規範

依 change 類型決定前綴，後半段使用 openspec change 的 kebab-case 名稱：

| 類型 | 分支名 | 範例 |
|---|---|---|
| feature（新需求） | `feature/<change-name>` | `feature/cloud-record-page` |
| bug-fix（bug 修復） | `bug-fix/<change-name>` | `bug-fix/m3u8-parse-crash` |
| refactor（重構） | `refactor/<change-name>` | `refactor/extract-download-state-machine` |

## 2. 驗收清單

flow-verify 開場一次問完，3 個項目（多選，依第 0 節提問協定呈現），
預設勾選狀態如下：

| 項目 | 代號 | 預設 | 說明 |
|---|---|---|---|
| 單元測試 | `unit-test` | 勾選 | 依 change 修改的檔案副檔名判斷平台，分別 delegate 給對應平台的工具執行，結果合併呈現，詳見第 5 節。專案慣例預設不補測試的層級（例如部分專案的 UI 層，由單元測試工具或專案 `context` 定義）需要時先詢問——此為單一 class 類型層級的規則，跟本項是否整個跳過是不同層級，兩者並存。 |
| 文件註解 | `doc-comment` | 勾選 | 為本次新增/修改的 public class 與 function 補上文件型註解（Kotlin 的 KDoc、Swift 的 DocC 等）。依平台委派，規則同第 5.2 節。 |
| code-review | `code-review` | 勾選 | 依第 4 節設定取得可用工具供選擇，不寫死清單；執行前自動先跑 lint/format（不進勾選，詳見第 5.3 節）。放最後執行，才能連測試與文件註解一起審。 |

**「文件註解」不是平台專有名詞**：舊版稱為「KDoc」，但 KDoc 是 Kotlin 專有、
JUnit 是 JVM 專有，與 flow 多語言共用的目標相衝。改名後涵蓋 KDoc／DocC／
docstring／TSDoc，且不會與一般 inline 註解或 `docs/` 產出文件混淆。

執行順序固定為上表由上到下，只執行被勾選的項目。開場詢問完成後，勾選結果立刻
寫入 `.flow.yaml` 的 `verify_selected`（見第 3 節），供 flow-finish 判斷驗收完整性。

## 3. `.flow.yaml` 結構

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
verified_at:            # flow-verify 全部完成時寫入，保留為驗收紀錄
  head: <git rev-parse HEAD 的值，作為 diff 的比較基準>
  diff_hash: <git diff <head> | git hash-object --stdin 的結果，涵蓋當時未提交的變更內容>
finish:                 # flow-finish archive 後寫入，支援 commit/push/整合續跑
  archive_path: openspec/changes/archive/YYYY-MM-DD-<change-name>
  synced_specs:         # 本次 archive 實際同步的主 spec 路徑
    - openspec/specs/<capability>/spec.md
```

### 3.1 舊代號相容

`junit` → `unit-test`、`kdoc` → `doc-comment` 為代號改名。舊 change 檔案不主動改寫、
不要求遷移，依位置分兩類處理：

- **值 alias**（`verify_selected`／`verified`）：讀取時接受新舊兩種值，
  寫回時一律只寫 canonical（新）值。
- **key alias**（`openspec/config.yaml` 的 `verify_tools.kdoc` → `verify_tools.doc_comment`）：
  見第 4 節的碰撞規則。

**比較前必須先 normalize**——先把讀到的值轉成 canonical 代號，再與需求集合比較：

```
normalized = normalize_aliases(verified)      # junit→unit-test, kdoc→doc-comment
required   = {unit-test, doc-comment, code-review}
missing    = required - normalized
```

否則會發生「先轉成新代號、卻拿舊代號集合去比對」導致三項永遠被判定為缺少。

### 3.2 缺欄位的處理

- 缺 `base_branch`／`verified_at` → 略過對應檢查，不報錯。
- 缺 `verify_selected`（優化前建立的舊 change）→ fallback 為檢查 `verified`
  （normalize 後）是否缺 `unit-test`／`doc-comment`／`code-review` 任一項，
  缺任一項時由 flow-finish 問是否繼續，預設停止、可明確覆寫。

### 3.3 `verified` 的本輪邊界

`verified` **每輪重建、不累積**：flow-verify 開場寫入 `verify_selected` 時，
一併清除整個 `verified`。

否則會出現：第一次三項全過 → 程式碼又改了 → 第二次只勾 code-review 但中途中斷 →
舊的 `verified` 仍在 → flow-finish 判定驗收完成而放行一個已失效的 change。

若「單元測試」因環境限制（如無 Xcode）只完成部分平台，仍整項記入 `verified`，
flow-verify 的收尾摘要須附註警告說明未執行的平台與原因（見第 5.4 節），不得
只寫「完成」誤導人工驗收。

### 3.4 `verified_at` 的驗收快照

flow-verify 完成時保存 `head` 與相對該 commit 的 `diff_hash`，讓人工需要時可以
追查驗收當下的內容。**flow-finish 不比較這兩個值**：手動呼叫 finish 即表示使用者
接受驗收結果，不能因內容雜湊不同額外打斷收尾。

## 4. 專案設定（`openspec/config.yaml` 的 `flow:` 區塊）

跟著專案走、不隨 skill 更新。所有 flow 設定只放這一個檔案，不得複製到各 change
的 `.flow.yaml` 或 artifacts。寫入時機分兩種：

| 欄位 | 誰寫入、何時 |
|---|---|
| `commit`（第 4.5 節） | flow-init。discuss 是第一個需要 commit 的步驟，早於其他會詢問的流程 |
| `worktree_base` | flow-apply 首次選 worktree 時詢問後寫入 |
| `verify_tools`／`verify_commands`／`verify_notes` | flow-verify／flow-finish 首次用到時詢問後寫入；flow-init 不預先設定 |

整合方式（本地合併／PR／保留）**不寫入設定**，每次由 flow-finish 詢問。

`worktree_base` 沒有固定值，預設建議值須由 flow-apply 依當前專案目錄名稱
（`<repo名>-worktrees`）現場推斷，不可寫死任何專案名稱：

```yaml
flow:
  worktree_base: ../<repo名>-worktrees   # 例如專案目錄為 my-app，推斷為 ../my-app-worktrees
```

### 4.1 `verify_tools`：工具物件清單

三個 step（`unit_test`／`doc_comment`／`code_review`）共用同一種結構，
由 flow-verify 首次使用時掃描並詢問後寫入：

```yaml
flow:
  verify_tools:
    unit_test:
      - name: android-unit-testing
        kind: skill
        platforms: [android]
        default: true
    doc_comment:
      - name: kotlin-kdoc
        kind: skill
        platforms: [android]
        default: true
    code_review:
      - name: team-code-review
        kind: skill
        platforms: [android]
        default: true
      - name: /code-review
        kind: command             # 呼叫方式與 skill 不同
        hosts: [claude-code]      # 只在此環境可用
        internal_parallelism: true  # 自己會 spawn 子代理，見第 5.5 節
        default: true
```

欄位語意：

| 欄位 | 必填 | 語意 |
|---|---|---|
| `name` | 是 | 工具名稱 |
| `kind` | 是 | `skill` \| `command`，決定呼叫方式 |
| `platforms` | 否 | **省略或 null ＝ 適用所有偵測到的平台**；非空清單 ＝ 僅適用列出的平台；空清單 ＝ 設定錯誤，提示使用者 |
| `hosts` | 否 | 省略 ＝ 任何環境可用；有值 ＝ 只在列出的環境可用 |
| `default` | 否 | 省略視為 `false`；`true` 表示開場提問時預先勾選 |
| `internal_parallelism` | 否 | `true` 表示該工具內部自己會平行委派，見第 5.5 節 |

**為何需要 `kind` 與 `hosts`**：`openspec/config.yaml` 跟著 repo 進版控，但
`/code-review`（command）、`team-code-review`（skill）呼叫方式不同，且不是每個
環境都有這些工具。同事拉下 repo、或換到另一個 agent 環境時，沒有這兩個欄位
就會直接失效。

**向後相容（舊版扁平字串格式）**：優化前的設定是 `<step>.<platform>: <單一工具名稱>`
的扁平字串（例如 `unit_test: {android: android-unit-testing}`）。讀到這種格式時，
視為 `[{name: <工具名稱>, kind: skill, platforms: [<platform>], default: true}]`；
不主動改寫既有檔案，下次該設定被更新寫回快取時才轉成新格式。

**不做**：schema version、未知欄位處理、完整 migration 規格。這是人類會手改的
設定檔，不是 API contract。

### 4.2 新舊 key 碰撞規則

canonical key（`doc_comment`）存在時一律優先；只有它不存在時才讀 legacy key
（`kdoc`）。更新該設定並寫回 canonical 時，順手移除 legacy key。

（「不主動遷移 ＋ 寫回 canonical」必然會產生兩個 key 並存的狀態，沒有優先序
就會出現一邊讀舊值、一邊讀新值。）

### 4.3 `verify_commands`：lint／test 指令快取

lint/format 與 flow-finish 合併後的測試都走內省路線，不走工具委派，見第 5.3 節：

```yaml
flow:
  verify_commands:
    lint:
      android: <實際指令>
      ios: <實際指令>
    test:                 # flow-finish 本地合併後驗證用；首次需要時詢問後寫入
      android: <實際指令>
```

快取的指令必須是單一、可重跑的指令，**不得串接 `rm -rf` 等清除動作**——
這類組合會觸發權限確認，且清除失敗時難以判斷測試結果。

### 4.4 `verify_notes`：找不到工具時的應急說明

```yaml
flow:
  verify_notes:
    unit_test:
      ios: <使用者輸入的說明文字>
```

### 4.5 `commit`：commit 訊息格式

由 flow-init 偵測後寫入，所有 flow 的 commit 依第 9 節使用：

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

## 5. 平台偵測與工具委派

flow-verify 執行「單元測試」「文件註解」「code-review」時，不假設專案只有
Android／Kotlin。判斷依據是 change 實際修改的檔案副檔名，而非在 config.yaml
或 `.flow.yaml` 宣告固定的 repo 平台——這樣純 Android repo（無 `.swift` 檔案）
永遠不會觸發以下任何平台分支，行為與導入本節前完全一致。

### 5.1 平台偵測

- 列出這次 change 修改的檔案（diff against `base_branch`），依副檔名分類：
  - `.kt`／`.kts` → Kotlin（Android／commonMain 等）
  - `.swift` → iOS
- 同一個 change 可能同時有兩種檔案，「單元測試」等項目需分別處理兩側，結果
  合併呈現在同一份驗收摘要裡，不拆成兩個勾選項。

這套映射刻意保持寬鬆，實際歸屬由 AI 依專案脈絡判斷；未來使用時若出現不順暢
再回頭調整，現階段不設計更精細的 language／target 兩層模型。

### 5.2 委派工具（單元測試／文件註解／code-review）

這三項需要語意理解，delegate 給對應工具執行，flow-verify 本身不直接下建置/測試指令：

1. 讀 `openspec/config.yaml` 的 `flow.verify_tools.<step>`（見第 4.1 節），
   過濾掉當前環境不可用的工具（依 `hosts`）與不適用於偵測到的平台的工具（依 `platforms`）。
2. **依「覆蓋缺口」判斷是否需要掃描**——不是「全部不可用才掃描」：

   ```
   對每個 (step, 偵測到的平台) 檢查過濾後是否仍有至少一個工具覆蓋
     有覆蓋 → 照跑
     沒有   → 該組合單獨進入下方第 3 步的掃描流程
   省略 platforms 的通用工具可一次覆蓋所有偵測到的平台
   ```

   若用「全部不可用才掃描」會漏掉這種情況：混合 Android／iOS 的 change 中，
   Android 側工具仍可用、iOS 側完全沒有工具——因為「不是全部不可用」所以不觸發
   掃描，**iOS 側的驗收就靜默消失了**。

3. 有缺口 → 動態掃描目前環境可用的 skill/command，依第 0 節提問協定列出讓用戶選。
   - 找不到任何對應工具 → 依提問協定問用戶：「現在建一個正式 skill」或
     「先在 config.yaml 寫一段內嵌文字說明應急」。只有環境中有可建立 skill 的
     工具時才列出前者，並標注將使用的工具名稱（現場掃描，不寫死名稱）。
     - 選建 skill → 呼叫該建立 skill 的工具，建立完成後把工具寫入
       `flow.verify_tools.<step>` 快取。
     - 選內嵌文字 → 把用戶輸入的說明寫入 `flow.verify_notes.<step>.<platform>`
       （自由文字欄位），flow-verify 依這段文字說明執行；不寫入 `verify_tools`
       快取（沒有工具可快取），下次一樣會走這條路徑，除非之後升級成正式 skill。
   - 選定後寫入 `flow.verify_tools.<step>`，含 `kind`／`platforms`／`hosts`。

所有平台一視同仁，包含 Android／Kotlin：`config.yaml` 尚未設定
`verify_tools.unit_test`／`verify_tools.doc_comment` 時，一樣走上面第 3 步的
掃描／詢問流程（`android-unit-testing`、`kotlin-kdoc` 會在掃描時被列為候選
工具），選定後寫回快取，不特別硬編碼任何平台的預設對應。

### 5.3 內省查詢（lint/format）

lint/format 是確定性 CLI 規則檢查，沒有 AI 工具的概念，改用內省查詢，不走
5.2 的委派路線：

1. 讀 `openspec/config.yaml` 的 `flow.verify_commands.lint.<platform>`，
   有值就直接執行。
2. 沒有快取值 → 對 repo 做內省查詢取得實際指令（掃描建置設定、可用 task 清單等），
   查到就執行並寫入快取；查不到任何對應工具設定 → 視為「此 repo 未設定」，
   靜默略過該平台的 lint，摘要中註明「lint：`<platform>` 未設定，略過」，非錯誤。
3. 快取指令執行失敗（例如 task not found）→ 自動重新執行一次第 2 步的內省
   查詢並更新快取，再試一次；仍失敗才依第 7 節錯誤處理原則報錯中止。
4. lint/format 不是獨立勾選項，併入 code-review 步驟，在其他 review 工具
   動態掃描與執行之前自動先跑，不需要用戶勾選。

### 5.4 環境能力限制

- 若 change 修改了某平台的檔案，但目前環境沒有對應工具鏈（例如非 macOS 或
  未安裝 Xcode）→ 明確報告「<平台> 部分因環境限制未執行，需在 <環境> 上補跑」
  並跳過該平台部分；其他平台照常執行完成並標記完成。
- 這與「指令快取失效」不同，不套用 5.3 的自動重查邏輯——工具鏈本來就不
  存在，重查沒有意義。
- 對應 `.flow.yaml` 的 `verified` 記錄方式見第 3 節。

### 5.5 多工具執行策略

同一個 step 選了多個工具時：

```
可用平行子代理      → 各工具分別平行執行
超過可用 concurrency → 分批執行（不因 slot 不足拒絕）
無平行能力          → 循序執行
```

依**能力**判斷，不依環境名稱硬編碼——各環境是否支援平行子代理會隨版本改變。

- **子代理一律唯讀、只回報 findings**：多個 review 工具若都能改檔，會併發修改
  同一批檔案。
- **`internal_parallelism: true` 的工具由主 agent 直接執行**，不再往外包一層，
  且**排成獨立批次**、不與其他工具同時搶 concurrency（布林值無法得知它內部佔
  幾個 slot，獨立批次是最省事又安全的處理）。

### 5.6 工具呼叫契約

flow-verify 委派工具時：

- **主動把必要輸入傳進去**：fixed point ＝ `.flow.yaml` 的 `base_branch`；
  spec ＝ 該 change 的 openspec artifacts。
- **任何被委派的工具不得自行向使用者提問**；缺必要輸入時回報給 flow-verify，
  由它依第 0 節提問協定統一處理，避免破壞「開場一次問完」。
- **禁止被委派的工具執行 setup、建立輔助文件、或任何非 finding 修正的寫檔行為。**
- **委派 `/code-review` 時，prompt 須明確告知**：fixed point 與 spec 已由
  flow-verify 提供，`docs/agents/issue-tracker.md` 缺失視為非必要，**略過 issue
  查詢，不得提示或要求執行 `/setup-matt-pocock-skills`**。否則該工具預設遇到
  缺檔會停下來叫人跑 setup，中斷「開場一次問完」的流程。

### 5.7 指令歸屬

- **flow 不硬編碼任何建置／測試指令**。需要時依序解析：專案的指令設定
  （CLAUDE.md／AGENTS.md 等）→ `config.yaml` 快取 → 委派對應 skill。
- **NEVER 主動安裝／啟動 APP**——由 tasks 內容或用戶指示決定。
- 指令失敗的處理順序：先套 5.3 的快取失效重查 → 再套 5.4 的環境限制跳過 →
  兩者都不適用才依第 7 節明確報錯停下。

### 5.8 NEVER

- **NEVER 因為平台偵測邏輯而改變單一平台 repo 的既有驗收行為**——偵測不到
  其他平台的檔案時，所有相關分支必須完全不觸發，執行內容與偵測邏輯導入前一致。

## 6. 工具對照表（可携性）

| 步驟 | 有對應機制的環境（如 Claude Code） | 無對應機制時的退化行為 |
|---|---|---|
| 選項詢問 | 結構化提問工具（見第 0.2 節） | 編號清單（見第 0.2 節） |
| 多工具執行 | 平行子代理 | 循序執行（見第 5.5 節） |
| 驗收項目派工 | 每項依序派 subagent，Agent 工具帶 `effort`（見 flow-verify 第 2.0 節） | 主 session 依同樣順序直接執行 |
| 進入 worktree | 環境的工作目錄切換機制（Claude Code：`EnterWorktree` 帶 `path`） | `cd <worktree 路徑>`，再以 `pwd` 驗證；切不過去就改請用戶新開 session |

openspec-* 與 Matt 的 skill 是**必要依賴**（見第 8 節），由 flow-init 確認已安裝，
因此不提供退化行為；執行中發現缺少時依第 7 節停下。

## 7. 錯誤處理原則

- openspec CLI 不存在或指令失敗 → 明確報錯並停下，不靜默跳過。
- 必要依賴的 skill 不存在 → 停下，說明缺少哪個 skill 與安裝方式，建議執行
  `/flow-init` 重新檢查；不得自行以退化行為代替。
- flow-apply 的 metadata push 失敗（權限不足、分支保護規則等）→ 視為錯誤並中止，
  不建立分支/worktree、不靜默略過、不自動改用其他方式繞過。

## 8. 執行前檢查（preflight）

flow-discuss／flow-apply／flow-verify／flow-finish／flow-note 開始時先做以下檢查。
**只檢查、不修復**：任一項不通過就停下，說明原因並建議執行 `/flow-init`，
不得自行執行 `openspec init`、手動建立 `openspec/` 或寫入設定。

1. `openspec --version` 可執行。
2. `openspec list --json` 的 `root` 不為 `null`。
   - `root` 為 `null` 且錯誤訊息以 `Declared in` 或 `Invalid store declaration in`
     開頭 → 專案透過 store 使用 OpenSpec 但此機器無法解析：顯示該錯誤的
     `message` 與 `fix` 後停下，**不要**建議 `/flow-init`。
3. `openspec/config.yaml` 有 `flow.commit`（第 4.5 節）。flow-verify 不會 commit，
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
讀取的 issue tracker 設定）、任何可建立 skill 的工具（見第 5.2 節）。

flow 不依賴 superpowers。

## 9. commit 規則

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

## 10. 遠端主機與 PR

flow 不假設遠端是 GitHub。需要和遠端主機互動時，先以
`git remote get-url origin` 判斷主機：

| 主機 | 條件 | PR 相關操作 |
|---|---|---|
| GitHub | URL 含 `github.com` 且 `gh` 可用 | 可用 `gh pr list/create/view/edit` |
| GitLab | URL 含 `gitlab` 且 `glab` 可用 | 可用 `glab mr list/create/view` |
| 其他（Bitbucket、自架等），或上述 CLI 不可用 | — | **不呼叫任何主機 CLI**。建立 PR ＝ push 分支後顯示 `git push` 輸出中的建立 PR 網址；沒有網址時請使用者到遠端網頁建立 |

判斷分支是否已合併時，不依賴主機 CLI：先 `git fetch <remote> <base_branch>`，
再以 `git merge-base --is-ancestor <branch> <remote>/<base_branch>` 判斷。
