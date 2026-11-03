---
name: flow-verify
description: 驗收已完成實作的 openspec change。當用戶說「驗收」、「開始驗收」、「verify」或使用 /flow-verify 時觸發。開場一次問完 3 個項目（單元測試、文件註解、code-review 皆預設勾選）並標注將使用的工具，依序只執行已勾選項目，每項各自派獨立 subagent 並帶入對應 effort 以隔離 context；單元測試/文件註解依修改檔案的平台動態委派對應工具，code-review 依設定列出可用工具供選擇並自動先跑 lint/format，findings 列成編號清單附建議（改 code／改 spec+code／改 spec／不處理），由用戶逐條決定後才修正；review 修正後會重新驗證受影響項目；中斷後若程式碼未變可接續剩下的項目；最後產出驗收摘要供人工驗收參考。
---

# flow-verify：驗收

flow-apply 完成後的自動化驗收編排。人工驗收仍由用戶進行，
本 skill 負責跑完自動化項目並給出摘要。

開始前先載入 `flow-conventions` skill（以 plugin 安裝時名稱帶前綴，例如 `<plugin>:flow-conventions`）取得共用規範；環境沒有 Skill 工具時，改讀同層目錄的
`../flow-conventions/SKILL.md`，且必須讀完整份。
另外必須讀完整份本 skill 的 `references/verify-tools.md`（以下簡稱 verify-tools）：

- flow-conventions 的「提問協定」：提問協定（所有提問一律依此呈現與解析）
- flow-conventions 的「`.flow.yaml` 結構」：`.flow.yaml` 結構、代號相容、`verified` 本輪邊界、`verify_progress` 中斷接續
- verify-tools 的「驗收清單」：驗收清單與順序
- verify-tools 的「驗收相關設定」：`verify_tools`／`verify_commands`／`verify_notes` 設定結構
- verify-tools 的「平台偵測與工具委派」：平台偵測、工具委派、多工具執行、呼叫契約

## 流程

開始前先做 flow-conventions 的「執行前檢查」（本 skill 不 commit，略過第 3 項）；
未通過就停下並建議 `/flow-init`。

### 1. 選定 change 與開場詢問

- change 選擇邏輯同 flow-apply（指定 > 推斷 > 列出讓用戶挑）。
- **接續檢查**：`.flow.yaml` 的 `verified`（normalize 後）非空、且是 `verify_selected`
  的真子集時，代表上次那輪中斷了。依 flow-conventions 的「`verify_progress` 的中斷接續」
  計算目前快照：
  - 與 `verify_progress` 完全相同 → 先問是否接續，不問項目勾選：

    ```
    ❓ 上次驗收中斷，程式碼與當時相同。
       已完成：單元測試、文件註解　剩下：code-review
      1) 接續，只跑剩下的項目 ←預設
      2) 重新開始，重新勾選項目
    ```

    選接續 → 不改寫 `verify_selected`、不清除 `verified`，照下一點做平台偵測與
    工具解析（只解析剩下的項目），接著從第一個未完成的項目進入第 2 步。
    選重新開始 → 照一般流程。
  - 不同或缺 `verify_progress` → 回報一行「上次進度已失效（程式碼有變動），重新開始」，
    照一般流程。
- 先做平台偵測（verify-tools 的「平台偵測」）與工具解析（verify-tools 的「委派工具」），
  **在提問前就知道每個項目會用哪些工具**，才能在選項後透明標注。
- 依提問協定（多選）一次問完 3 個項目，選項後標注將使用的工具：

```
❓ 這次驗收要執行哪些項目？
  1) 單元測試    ←預設  [android-unit-testing]
  2) 文件註解    ←預設  [kotlin-kdoc]
  3) code-review ←預設  [team-code-review, /code-review]
                        （候選：codex:rescue）
回覆：數字多選 `1,3`／`0` 套預設／或描述要改的部分（如「3 只跑 team-code-review」）
```

- 用戶要調整某項目用哪些工具時，**在同一則回覆用自然語言表達即可**，
  不另開一輪提問。**這類覆寫僅適用本次執行**，不得寫回 `config.yaml` 的
  `default` 欄位——那是跟著 repo 走的共用設定，一次性的口頭調整不該悄悄
  變成永久預設。要調整長期預設請直接編輯 `config.yaml`（見 flow-help 速查）。
- 詢問完成後，執行任何項目之前：
  1. 把勾選結果（canonical 代號陣列）寫入 `.flow.yaml` 的 `verify_selected`
  2. **清除整個 `verified`**（每輪重建、不累積，見 flow-conventions 的「`verified` 的本輪邊界」），
     並更新 `verify_progress`

```yaml
verify_selected:
  - unit-test
  - doc-comment
  - code-review
verified: []
```

### 2. 依序執行已勾選的項目（完成一項就把代號寫入 `.flow.yaml` 的 `verified`）

依 `verify_selected` 決定要跑哪些，未勾選的項目直接跳過，不執行也不記錄。

**每次寫入或清除 `verified`（包含第 4 節的清除），都要同時更新 `verify_progress`**，
指令見 flow-conventions 的「`verify_progress` 的中斷接續」。寫入順序是先完成檔案
修改、再寫 `verified` 與 `verify_progress`，中斷時才不會留下「已標記完成、但快照
不含該項修改」的狀態。

#### 2.0 派工方式：每項獨立 subagent、嚴格依序

每個項目在獨立 context 執行，讓各項 effort 各自生效，前一項的建置 log 與測試
輸出也不會污染後一項。本 skill 明確要求派工時在 Agent 工具帶入下表的 `effort`
（subagent 類型用 general-purpose，不另建 agent 定義檔）：

| 工作 | 執行者 | effort |
|---|---|---|
| 1. 單元測試 | subagent，載入 `verify_tools.unit_test` 解析出的工具 | medium |
| 2. 文件註解 | subagent，載入 `verify_tools.doc_comment` 解析出的工具 | medium |
| 3. lint/format | 主 session | — |
| 3. 一般 review 工具（`verify_tools.code_review` 中未標 `internal_parallelism` 者） | subagent，唯讀 | high |
| 3. `internal_parallelism: true` 的工具（如 `/code-review`） | 主 session 直接執行（subagent 無法再派 subagent，見 verify-tools 的「多工具執行策略」） | — |
| 開場詢問、finding 修正、第 4 節重新驗證、寫入 `.flow.yaml`、摘要 | 主 session | — |

**順序**：等前一個 subagent 回報、主 session 確認結果後，才派下一個；項目 1、2
會寫檔，不得同時派出。同一項目解析出多個工具時（如 Android／iOS 各一個），
寫檔項目也逐一派工，不平行。只有唯讀的 review 工具依 verify-tools 的「多工具執行策略」平行。

**派工 prompt 必須包含**：

- change 名稱、工作目錄絕對路徑（worktree 時為 worktree 路徑）
- fixed point（`.flow.yaml` 的 `base_branch`）與本次修改的檔案清單
- 要載入的工具名稱，以及下方該項目的規則（原文轉述，不得省略）
- verify-tools 的「工具呼叫契約」的呼叫契約：不得向用戶提問、不得執行 setup、
  不得寫入 `.flow.yaml`
- 要求依下方格式回報

**subagent 回報格式**：

```
狀態：done／needs_decision／failed／skipped_env
修改的檔案：<清單，唯讀工具填「無」>
執行的指令與結果：<實際指令、通過／失敗數或原始錯誤摘錄>
待決事項：<needs_decision 時列出要問用戶的問題與選項>
findings：<僅 review 工具>
```

**主 session 依狀態處理**：

- `done` → 確認回報附有實際指令與結果（缺漏視為未完成，重新派工），寫入
  `verified`，回報一行進度後**自動接續**下一項。
- `needs_decision`（如本次 diff 含 UI 層類別、要不要補測試）→ 依提問協定詢問，
  把答案帶入 prompt 重新派同一項目，只處理剩下的部分。
- `failed` → 停下回報用戶，不得標記 `verified`、不得接續下一項。
- `skipped_env` → 依 verify-tools 的「環境能力限制」記錄並接續。

**唯讀保護**：派出 review subagent 前記下 `git status --porcelain` 與
`git diff HEAD | git hash-object --stdin`，回報後再比對一次；有變化時停下
回報用戶，不自行還原。

**退化行為**：環境沒有 subagent 能力時，改由主 session 依同樣順序直接執行
（對照 verify-tools 的「環境能力對照」），其餘規則不變。

#### 2.1 各項目規則

固定執行順序（由上到下）：

1. **單元測試**（含 `unit-test` 才執行）
   - 依 change 修改的檔案判斷平台，依 `verify_tools.unit_test` 解析出的工具
     執行（見 verify-tools 的「平台偵測」「委派工具」）；有覆蓋缺口時依 verify-tools 的「委派工具」掃描或
     走二選一流程；環境無對應工具鏈時依 verify-tools 的「環境能力限制」報告並跳過，不視為失敗。
   - **委派單元測試工具時**：傳入本次修改的類別清單，並註明「由 flow-verify
     委派」。測試範圍、執行驗證與「未覆蓋」清單皆依該工具自身的執行流程；
     該工具判定需要人工決定的範圍（例如專案慣例預設不補測試的層級）以
     `needs_decision` 回報，不自行提問。主 session 收到 `needs_decision` 後
     詢問再派工，「未覆蓋」清單併入驗收摘要。
   - 各平台結果合併呈現在同一份驗收摘要裡。任一側測試失敗 → 修到通過才算
     完成，不得跳過（環境限制導致的跳過不算失敗）。
2. **文件註解**（含 `doc-comment` 才執行）
   - 為本次新增/修改的 public class 與 function 補上該語言的文件型註解，
     依 `verify_tools.doc_comment` 解析出的工具執行（見 verify-tools 的「平台偵測」「委派工具」）。
   - 內容一律以繁體中文（zh-TW）撰寫。
   - 測試程式碼不補文件註解：單元測試與 UI 測試的類別與函式一律跳過，
     不處理也不列入摘要。
3. **code-review**（含 `code-review` 才執行）
   - 執行前自動先跑 lint/format，依 verify-tools 的「內省查詢」內省取得指令並快取，
     不需用戶勾選；repo 未設定對應工具則靜默略過並在摘要註明。
   - 依第 1 步已解析好的工具清單執行，多工具時依 verify-tools 的「多工具執行策略」決定平行／分批／循序。
   - **委派時遵守 verify-tools 的「工具呼叫契約」呼叫契約**：主動傳入 fixed point（`.flow.yaml` 的
     `base_branch`）與 spec（該 change 的 openspec artifacts）；被委派的工具
     不得自行提問、不得執行 setup 或寫入非 finding 修正的檔案。
   - 結果呈現見下方第 3 節。
   - findings 逐項驗證是否成立，依第 3.1 節分類並列成待決清單，**等用戶逐條決定後才動手**。
   - 有修改 code 時，修正後依第 4 節重新驗證。

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

### 合併清單（team-code-review, codex:rescue）
【高】DownloadRepo.kt:88 未處理的 !!    ← 2 個來源都提到
【低】命名：mDataList → mVideoList      ← team-code-review

### /code-review（獨立呈現，不併入上表）
#### Standards
…
#### Spec
…

lint：ktlint 0 error；SwiftLint 未設定（略過）
```

#### 3.1 待決清單：每條 finding 由用戶決定

各工具的原始輸出呈現完之後，把所有 finding（不分來源，含判定不成立的）整理成
一份連續編號的清單。每條先驗證是否成立，再依「這條 finding 是否改變行為」給出建議：

| 建議 | 適用情況 | 處理方式 |
|---|---|---|
| 改 code | bug 或品質問題，行為不變（spec 已正確，或屬於 spec 不描述的實作細節，例如命名、`!!`） | 只改 code |
| 改 spec+code | 行為要改 | 先委派 `openspec-update-change` 更新 artifacts，再改 code |
| 改 spec | code 是對的，spec 寫錯或漏寫 | 只委派 `openspec-update-change`，不改 code |
| 不處理 | 判定不成立，或刻意保留（例如沿用既有模組慣例） | 不改，理由記入摘要 |

```
## 待決 findings（請逐條決定）
1.【高】DownloadRepo.kt:88 未處理的 !!          ← team-code-review, codex:rescue
   建議：改 code — 失敗時會 crash，spec 已要求顯示錯誤訊息
2.【中】重試次數寫死 3 次，spec 未定義上限       ← /code-review（Spec）
   建議：改 spec — code 的行為合理，是 spec 漏寫
3.【低】命名 mDataList → mVideoList              ← team-code-review
   建議：不處理 — 沿用該模組既有命名
回覆：逐條決定（如「1 改code 2 改spec 3 保留」）／`0` 全部照建議／或用自然語言說明
```

- 沒有任何 finding 時不提問，直接標記 code-review 完成。
- 用戶回覆後依序處理：先把所有「改 spec」「改 spec+code」的條目一次委派給
  `Skill(openspec-update-change)`（附上 finding、用戶的決定，並註明以 zh-TW 撰寫），
  再修改 code。
- 只有「改 spec」或「不處理」、沒有任何 code 修改時，不觸發第 4 節，直接標記 code-review 完成。

### 4. review 修正後的重新驗證

code-review 排在最後，若它的 findings 導致修改程式碼，**先前已標記 verified 的
項目就失效了**，而且**最後那批修正本身還沒被 review 過**。

```
1. 用戶決定「改 code」或「改 spec+code」的 finding，動手修改 code 前
     → 清除 code-review 的 verified
     → 清除「本輪已勾選且會被此次修改影響」的項目的 verified
2. 完成修改
     → 只重跑「本輪已勾選」的受影響項目 ＋ lint
       （依第 2.0 節重新派工，effort 與順序不變）
       （未勾選的項目不跑——尊重「只執行勾選項目」的契約）
3. 對最終 diff 再跑一次 code-review
4. 不再產生需要修改的有效 finding → 才重新標記 verified
```

第 3 步若又產生 finding，依第 3.1 節再列一次待決清單由用戶決定；有修改 code 時回到第 1 步。**第三輪仍有新的需修改 finding
時停下來回報用戶決定**，避免無限迴圈。

### 5. 收尾

全部項目完成後，寫入驗收快照供人工追查驗收當下的內容（flow-finish 不會比較此快照
來阻擋手動收尾；見 flow-conventions 的「`verified_at` 的驗收快照」）：

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
  各項的決定（改 code／改 spec+code／改 spec／不處理及理由）；若有重新驗證，
  註明重跑了哪些項目，或「未勾選，略過」>
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
- **NEVER 盲目接受 review 建議**——逐項驗證是否成立，給出建議後由用戶逐條決定。
- **NEVER 在用戶回覆待決清單前修改任何 code 或 artifacts**——即使是判定成立的高嚴重度 finding；用戶常選擇改 spec 而不是改 code。
- **NEVER 只改 code 而略過行為改變的 spec**——finding 會改變行為時，先用 `openspec-update-change` 更新 artifacts，再改 code。
- **NEVER 在 review 修正後沿用舊的 verified 標記**——受影響項目必須清除並重跑，
  且最終 diff 要再過一次 code-review，否則會留下「看似通過、實際已被後續修改破壞」的狀態。
- **NEVER 累加 `verified`**——每輪驗收開場就清空重建，否則舊結果會讓 flow-finish 誤判。唯一例外是快照完全相同、且用戶選擇接續時。
- **NEVER 在快照不同時接續上次進度**——`head` 或雜湊任一不同，已完成的項目就可能失效，必須重新開始。
- **NEVER 為了重新驗證而執行本輪未勾選的項目**——尊重用戶的勾選範圍。
- **NEVER 未詢問就為專案慣例預設不補測試的層級補單元測試**——例如部分專案的 UI 層；以單元測試工具或專案 `context` 的規定為準。
- **NEVER 為測試程式碼補文件註解**——只針對正式程式碼，且一律以繁體中文撰寫。
- **NEVER 忽略 `verify_selected` 直接跑滿 3 項或全部跳過**——只執行用戶勾選的項目。
- **NEVER 讓多個 review 子代理同時修改檔案**——子代理一律唯讀、只回報 findings，修正統一由主 agent 執行。
- **NEVER 同時派出會寫檔的項目，或前一項尚未確認完成就派下一項**——單元測試可能改到正式程式碼，文件註解會改正式檔案，code-review 必須看到兩者完成後的最終 diff。
- **NEVER 把 `internal_parallelism: true` 的工具包進 subagent**——subagent 無法再派 subagent，該工具一律由主 session 直接執行。
- **NEVER 讓 subagent 寫入 `.flow.yaml`**——`verify_selected`、`verified`、`verify_progress`、`verified_at` 一律由主 session 依回報結果寫入。
- **NEVER 讓被委派的工具自行向用戶提問或執行 setup／寫檔**——缺輸入時回報給本 skill 統一處理（verify-tools 的「工具呼叫契約」）。
- **NEVER 因平台偵測邏輯改變單一平台 repo 的既有驗收行為**——change 未修改其他平台檔案時，相關分支必須完全不觸發。
- **NEVER 在因環境限制跳過部分平台時，遺漏在驗收摘要附註警告**——必須明確標註哪個平台未執行、原因為何、建議如何補跑。
