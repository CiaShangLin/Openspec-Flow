# flow-verify：驗收清單與工具委派

flow-verify 專用的規則。開場前讀取本檔；共用的提問協定、`.flow.yaml` 與錯誤處理
見 flow-conventions skill。

## 1. 驗收清單

flow-verify 開場一次問完，3 個項目（多選，依 flow-conventions 的「提問協定」提問協定呈現），
預設勾選狀態如下：

| 項目 | 代號 | 預設 | 說明 |
|---|---|---|---|
| 單元測試 | `unit-test` | 勾選 | 依 change 修改的檔案副檔名判斷平台，分別 delegate 給對應平台的工具執行，結果合併呈現，詳見「平台偵測與工具委派」。專案慣例預設不補測試的層級（例如部分專案的 UI 層，由單元測試工具或專案 `context` 定義）需要時先詢問——此為單一 class 類型層級的規則，跟本項是否整個跳過是不同層級，兩者並存。 |
| 文件註解 | `doc-comment` | 勾選 | 為本次新增/修改的 public class 與 function 補上文件型註解（Kotlin 的 KDoc、Swift 的 DocC 等）。依平台委派，規則同「委派工具」。 |
| code-review | `code-review` | 勾選 | 依「驗收相關設定」設定取得可用工具供選擇，不寫死清單；執行前自動先跑 lint/format（不進勾選，詳見「內省查詢」）。放最後執行，才能連測試與文件註解一起審。 |

**「文件註解」不是平台專有名詞**：舊版稱為「KDoc」，但 KDoc 是 Kotlin 專有、
JUnit 是 JVM 專有，與 flow 多語言共用的目標相衝。改名後涵蓋 KDoc／DocC／
docstring／TSDoc，且不會與一般 inline 註解或 `docs/` 產出文件混淆。

執行順序固定為上表由上到下，只執行被勾選的項目。開場詢問完成後，勾選結果立刻
寫入 `.flow.yaml` 的 `verify_selected`（見 flow-conventions 的「`.flow.yaml` 結構」），供 flow-finish 判斷驗收完整性。

## 2. 驗收相關設定（`openspec/config.yaml` 的 `flow:` 區塊）

### 2.1 `verify_tools`：工具物件清單

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
        internal_parallelism: true  # 自己會 spawn 子代理，見「多工具執行策略」
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
| `internal_parallelism` | 否 | `true` 表示該工具內部自己會平行委派，見「多工具執行策略」 |

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

### 2.2 新舊 key 碰撞規則

canonical key（`doc_comment`）存在時一律優先；只有它不存在時才讀 legacy key
（`kdoc`）。更新該設定並寫回 canonical 時，順手移除 legacy key。

（「不主動遷移 ＋ 寫回 canonical」必然會產生兩個 key 並存的狀態，沒有優先序
就會出現一邊讀舊值、一邊讀新值。）

### 2.3 `verify_commands`：lint／test 指令快取

lint/format 與 flow-finish 合併後的測試都走內省路線，不走工具委派，見「內省查詢」：

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

### 2.4 `verify_notes`：找不到工具時的應急說明

```yaml
flow:
  verify_notes:
    unit_test:
      ios: <使用者輸入的說明文字>
```

## 3. 平台偵測與工具委派

flow-verify 執行「單元測試」「文件註解」「code-review」時，不假設專案只有
Android／Kotlin。判斷依據是 change 實際修改的檔案副檔名，而非在 config.yaml
或 `.flow.yaml` 宣告固定的 repo 平台——這樣純 Android repo（無 `.swift` 檔案）
永遠不會觸發以下任何平台分支，行為與導入本節前完全一致。

### 3.1 平台偵測

- 列出這次 change 修改的檔案（diff against `base_branch`），依副檔名分類：
  - `.kt`／`.kts` → Kotlin（Android／commonMain 等）
  - `.swift` → iOS
- 同一個 change 可能同時有兩種檔案，「單元測試」等項目需分別處理兩側，結果
  合併呈現在同一份驗收摘要裡，不拆成兩個勾選項。

這套映射刻意保持寬鬆，實際歸屬由 AI 依專案脈絡判斷；未來使用時若出現不順暢
再回頭調整，現階段不設計更精細的 language／target 兩層模型。

### 3.2 委派工具（單元測試／文件註解／code-review）

這三項需要語意理解，delegate 給對應工具執行，flow-verify 本身不直接下建置/測試指令：

1. 讀 `openspec/config.yaml` 的 `flow.verify_tools.<step>`（見「`verify_tools`」），
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

3. 有缺口 → 動態掃描目前環境可用的 skill/command，依 flow-conventions 的「提問協定」提問協定列出讓用戶選。
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

### 3.3 內省查詢（lint/format）

lint/format 是確定性 CLI 規則檢查，沒有 AI 工具的概念，改用內省查詢，不走
5.2 的委派路線：

1. 讀 `openspec/config.yaml` 的 `flow.verify_commands.lint.<platform>`，
   有值就直接執行。
2. 沒有快取值 → 對 repo 做內省查詢取得實際指令（掃描建置設定、可用 task 清單等），
   查到就執行並寫入快取；查不到任何對應工具設定 → 視為「此 repo 未設定」，
   靜默略過該平台的 lint，摘要中註明「lint：`<platform>` 未設定，略過」，非錯誤。
3. 快取指令執行失敗（例如 task not found）→ 自動重新執行一次第 2 步的內省
   查詢並更新快取，再試一次；仍失敗才依 flow-conventions 的「錯誤處理原則」錯誤處理原則報錯中止。
4. lint/format 不是獨立勾選項，併入 code-review 步驟，在其他 review 工具
   動態掃描與執行之前自動先跑，不需要用戶勾選。

### 3.4 環境能力限制

- 若 change 修改了某平台的檔案，但目前環境沒有對應工具鏈（例如非 macOS 或
  未安裝 Xcode）→ 明確報告「<平台> 部分因環境限制未執行，需在 <環境> 上補跑」
  並跳過該平台部分；其他平台照常執行完成並標記完成。
- 這與「指令快取失效」不同，不套用 5.3 的自動重查邏輯——工具鏈本來就不
  存在，重查沒有意義。
- 對應 `.flow.yaml` 的 `verified` 記錄方式見 flow-conventions 的「`.flow.yaml` 結構」。

### 3.5 多工具執行策略

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

### 3.6 工具呼叫契約

flow-verify 委派工具時：

- **主動把必要輸入傳進去**：fixed point ＝ `.flow.yaml` 的 `base_branch`；
  spec ＝ 該 change 的 openspec artifacts。
- **任何被委派的工具不得自行向使用者提問**；缺必要輸入時回報給 flow-verify，
  由它依 flow-conventions 的「提問協定」提問協定統一處理，避免破壞「開場一次問完」。
- **禁止被委派的工具執行 setup、建立輔助文件、或任何非 finding 修正的寫檔行為。**
- **委派 `/code-review` 時，prompt 須明確告知**：fixed point 與 spec 已由
  flow-verify 提供，`docs/agents/issue-tracker.md` 缺失視為非必要，**略過 issue
  查詢，不得提示或要求執行 `/setup-matt-pocock-skills`**。否則該工具預設遇到
  缺檔會停下來叫人跑 setup，中斷「開場一次問完」的流程。

### 3.7 指令歸屬

- **flow 不硬編碼任何建置／測試指令**。需要時依序解析：專案的指令設定
  （CLAUDE.md／AGENTS.md 等）→ `config.yaml` 快取 → 委派對應 skill。
- **NEVER 主動安裝／啟動 APP**——由 tasks 內容或用戶指示決定。
- 指令失敗的處理順序：先套 5.3 的快取失效重查 → 再套 5.4 的環境限制跳過 →
  兩者都不適用才依 flow-conventions 的「錯誤處理原則」明確報錯停下。

### 3.8 NEVER

- **NEVER 因為平台偵測邏輯而改變單一平台 repo 的既有驗收行為**——偵測不到
  其他平台的檔案時，所有相關分支必須完全不觸發，執行內容與偵測邏輯導入前一致。

## 4. 環境能力對照

| 步驟 | 有對應機制的環境（如 Claude Code） | 無對應機制時的退化行為 |
|---|---|---|
| 多工具執行 | 平行子代理 | 循序執行（見「多工具執行策略」） |
| 驗收項目派工 | 每項依序派 subagent，Agent 工具帶 `effort`（見 flow-verify 第 2.0 節） | 主 session 依同樣順序直接執行 |
