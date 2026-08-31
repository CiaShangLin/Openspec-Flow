---
name: flow-verify
description: 驗收已完成實作的 openspec change。當用戶說「驗收」、「開始驗收」、「verify」或使用 /flow-verify 時觸發。開場一次問完 5 個項目（單元測試、文件註解、code-review 預設勾選，mermaid 架構圖與 usecase 文件預設不勾選）並標注將使用的工具，依序只執行已勾選項目；單元測試/文件註解依修改檔案的平台動態委派對應工具，code-review 依設定列出可用工具供選擇並自動先跑 lint/format；review 修正後會重新驗證受影響項目，最後產出驗收摘要供人工驗收參考。
---

# flow-verify：驗收

flow-apply 完成後的自動化驗收編排。人工驗收仍由用戶進行，
本 skill 負責跑完自動化項目並給出摘要。

共用規範見同層目錄下的 `../flow-conventions/conventions.md`
（本 skill 與 flow-conventions 須維持同一層級，不論安裝在 Claude Code 的
`.claude/skills/` 或其他環境的對應 skills 目錄）：

- 第 0 節：提問協定（所有提問一律依此呈現與解析）
- 第 2 節：驗收清單與順序、usecase 格式
- 第 3 節：`.flow.yaml` 結構、代號相容、`verified` 本輪邊界
- 第 4 節：`verify_tools` 設定結構
- 第 5 節：平台偵測、工具委派、多工具執行、呼叫契約

## 流程

### 1. 選定 change 與開場詢問

- change 選擇邏輯同 flow-apply（指定 > 推斷 > 列出讓用戶挑）。
- 先做平台偵測（conventions 第 5.1 節）與工具解析（第 5.2 節），
  **在提問前就知道每個項目會用哪些工具**，才能在選項後透明標注。
- 依提問協定（多選）一次問完 5 個項目，選項後標注將使用的工具：

```
❓ 這次驗收要執行哪些項目？
  1) 單元測試    ←預設  [ngs-android-unit-testing]
  2) 文件註解    ←預設  [ngs-kdoc]
  3) code-review ←預設  [ngs-code-review, /code-review]
                        （候選：codex:rescue）
  4) mermaid 架構圖
  5) usecase 文件
回覆：數字多選 `1,3`／`0` 套預設／或描述要改的部分（如「3 只跑 ngs-code-review」）
```

- 用戶要調整某項目用哪些工具時，**在同一則回覆用自然語言表達即可**，
  不另開一輪提問。**這類覆寫僅適用本次執行**，不得寫回 `config.yaml` 的
  `default` 欄位——那是跟著 repo 走的共用設定，一次性的口頭調整不該悄悄
  變成永久預設。要調整長期預設請直接編輯 `config.yaml`（見 flow-help 速查）。
- 詢問完成後，執行任何項目之前：
  1. 把勾選結果（canonical 代號陣列）寫入 `.flow.yaml` 的 `verify_selected`
  2. **清除整個 `verified`**（每輪重建、不累積，見 conventions 第 3.3 節）

```yaml
verify_selected:
  - unit-test
  - doc-comment
  - code-review
verified: []
```

### 2. 依序執行已勾選的項目（完成一項就把代號寫入 `.flow.yaml` 的 `verified`）

依 `verify_selected` 決定要跑哪些，未勾選的項目直接跳過，不執行也不記錄。
固定執行順序（由上到下）：

1. **單元測試**（含 `unit-test` 才執行）
   - 依 change 修改的檔案判斷平台，依 `verify_tools.unit_test` 解析出的工具
     執行（見 conventions 第 5.1、5.2 節）；有覆蓋缺口時依第 5.2 節掃描或
     走二選一流程；環境無對應工具鏈時依第 5.4 節報告並跳過，不視為失敗。
   - **委派給 `ngs-android-unit-testing` 時**：找出本次修改的邏輯層類別
     （ViewModel/Repository/UseCase 等）補測試並執行對應建置任務驗證通過。
     UI 層（Adapter/ViewHolder/DiffUtil）預設不寫，需要時先詢問——此為單一
     class 類型層級的既有規則，跟整體是否勾選本項是不同層級，兩者並存。
   - 各平台結果合併呈現在同一份驗收摘要裡。任一側測試失敗 → 修到通過才算
     完成，不得跳過（環境限制導致的跳過不算失敗）。
2. **文件註解**（含 `doc-comment` 才執行）
   - 為本次新增/修改的 public class 與 function 補上該語言的文件型註解，
     依 `verify_tools.doc_comment` 解析出的工具執行（見 conventions 第 5.1、
     5.2 節）。
   - 內容一律以繁體中文（zh-TW）撰寫。
   - 測試程式碼不補文件註解：單元測試與 UI 測試的類別與函式一律跳過，
     不處理也不列入摘要。
3. **code-review**（含 `code-review` 才執行）
   - 執行前自動先跑 lint/format，依 conventions 第 5.3 節內省取得指令並快取，
     不需用戶勾選；repo 未設定對應工具則靜默略過並在摘要註明。
   - 依第 1 步已解析好的工具清單執行，多工具時依第 5.5 節決定平行／分批／循序。
   - **委派時遵守第 5.6 節呼叫契約**：主動傳入 fixed point（`.flow.yaml` 的
     `base_branch`）與 spec（該 change 的 openspec artifacts）；被委派的工具
     不得自行提問、不得執行 setup 或寫入非 finding 修正的檔案。
   - 結果呈現見下方第 3 節。
   - findings 逐項驗證是否成立再修正（不盲改），重大問題先回報用戶再動手。
   - **修正後依第 4 節重新驗證。**
4. **mermaid 架構圖**（含 `mermaid` 才執行）
   - 呼叫 `Skill(design-doc-mermaid)`，產出至專案慣例位置
     （如 core 模組 `docs/diagrams/`）。
5. **usecase 文件**（含 `usecase` 才執行）
   - 依 conventions 的 Given-When-Then 格式，
     寫入 `docs/usecases/<change-name>.md`。

### 3. code-review 結果呈現

**不是所有工具的輸出都能合併**。標記 `internal_parallelism: true` 的工具
（例如自帶 Standards／Spec 雙軸子代理的 `/code-review`）通常有自己的呈現規則，
強行合併會違反其設計。

- `internal_parallelism: true` 的工具 → **輸出原樣獨立成段**，不併入去重清單
- 其餘工具 → 合併去重，同一問題被多個工具指出就合併並**保留全部來源**

**命中數只是「多來源支持度」，不等於嚴重度**——排序依各工具給的 severity，
命中數作為輔助標示，不得用來提升優先序。

```
## code-review 結果

### 合併清單（ngs-code-review, codex:rescue）
【高】DownloadRepo.kt:88 未處理的 !!    ← 2 個來源都提到
【低】命名：mDataList → mVideoList      ← ngs-code-review

### /code-review（獨立呈現，不併入上表）
#### Standards
…
#### Spec
…

lint：ktlint 0 error；SwiftLint 未設定（略過）
```

### 4. review 修正後的重新驗證

code-review 排在最後，若它的 findings 導致修改程式碼，**先前已標記 verified 的
項目就失效了**，而且**最後那批修正本身還沒被 review 過**。

```
1. 接受 finding、動手修改前
     → 清除 code-review 的 verified
     → 清除「本輪已勾選且會被此次修改影響」的項目的 verified
2. 完成修改
     → 只重跑「本輪已勾選」的受影響項目 ＋ lint
       （未勾選的項目不跑——尊重「只執行勾選項目」的契約）
3. 對最終 diff 再跑一次 code-review
4. 不再產生需要修改的有效 finding → 才重新標記 verified
```

第 3 步若又產生需修改的 finding，回到第 1 步。**第三輪仍有新的需修改 finding
時停下來回報用戶決定**，避免無限迴圈。

### 5. 收尾

全部項目完成後，寫入驗收快照供人工追查驗收當下的內容（flow-finish 不會比較此快照
來阻擋手動收尾；見 conventions 第 3.4 節）：

```yaml
verified_at:
  head: <git rev-parse HEAD 的值>
  diff_hash: <git diff HEAD | git hash-object --stdin 的結果>
```

以回覆訊息產出驗收摘要（不落地檔案），內容須反映**最後一次**的執行結果：

```
## 驗收摘要：<change-name>
- 單元測試：<各平台新增 N 個測試、執行結果；或因環境限制未執行、需於 X 環境補跑；
  或「未勾選，略過」>
- 文件註解：<補充了哪些類別/函式，或「未勾選，略過」>
- code-review：<lint/format 結果（含未設定而略過的平台）；使用工具、發現 N 項、
  處理結果；若有重新驗證，註明重跑了哪些項目，或「未勾選，略過」>
- 產出文件：<mermaid / usecase 路徑，或「無」>
- 略過項目：<本次未勾選的項目清單，或「無」>
```

提示：「人工驗收通過後，執行 `/flow-finish <name>` 進行歸檔與收尾」。

**為何 archive 必須在合併／建 PR 之前**：`openspec archive` 不只是搬移 change
目錄，還會把 delta specs 同步進 `openspec/specs/<capability>/spec.md`——這是
一次真正的內容變更。若等合併後才做，等於要在 main 上再補一個獨立 commit/PR
才能更新 spec，多一道流程且容易被忘記，且多數團隊的 main 分支保護規則也不允許
直接 push。在分支上先歸檔，讓 spec 更新跟程式碼變更落在**同一個 PR**裡一起
被 review、一起合併，避免 spec 與程式碼出現不同步的空窗期。

正確順序：verify（本 skill）→ `/flow-finish`（archive → archive/spec commit → push →
合併/PR/保留 → worktree 清理）。archive、commit、push／建 PR／合併、worktree 清理都不屬於
flow-verify 的職責，交給 `flow-finish` 處理。

## NEVER

- **NEVER 測試未通過就標記 verified**——修到通過才算完成，不得跳過或降級為警告。
- **NEVER 憑印象填寫驗收摘要**——每一項必須基於實際指令輸出，且反映最後一次執行結果。
- **NEVER 盲目接受 review 建議**——逐項驗證是否成立再修正，重大問題先回報用戶。
- **NEVER 在 review 修正後沿用舊的 verified 標記**——受影響項目必須清除並重跑，
  且最終 diff 要再過一次 code-review，否則會留下「看似通過、實際已被後續修改破壞」的狀態。
- **NEVER 累加 `verified`**——每輪驗收開場就清空重建，否則舊結果會讓 flow-finish 誤判。
- **NEVER 為了重新驗證而執行本輪未勾選的項目**——尊重用戶的勾選範圍。
- **NEVER 未詢問就為 UI 層（Adapter/ViewHolder/DiffUtil）補單元測試**——專案慣例為預設不寫。
- **NEVER 為測試程式碼補文件註解**——只針對正式程式碼，且一律以繁體中文撰寫。
- **NEVER 忽略 `verify_selected` 直接跑滿 5 項或全部跳過**——只執行用戶勾選的項目。
- **NEVER 讓多個 review 子代理同時修改檔案**——子代理一律唯讀、只回報 findings，修正統一由主 agent 執行。
- **NEVER 讓被委派的工具自行向用戶提問或執行 setup／寫檔**——缺輸入時回報給本 skill 統一處理（conventions 第 5.6 節）。
- **NEVER 因平台偵測邏輯改變單一平台 repo 的既有驗收行為**——change 未修改其他平台檔案時，相關分支必須完全不觸發。
- **NEVER 在因環境限制跳過部分平台時，遺漏在驗收摘要附註警告**——必須明確標註哪個平台未執行、原因為何、建議如何補跑。
