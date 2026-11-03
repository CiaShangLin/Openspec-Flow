---
name: flow-discuss
description: Use when 使用者要求開始或繼續 OpenSpec 的需求討論、規劃新功能、bug 修復或重構，例如「開始需求討論」、「討論一個需求」、「我有個想法」或使用 /flow-discuss。
---

# flow-discuss：需求討論與規劃

以 OpenSpec 為基底的使用者入口與編排層。Flow 負責理解意圖、建議路線、維持討論與 artifacts 一致；實際探索、提案與文件更新委派給 OpenSpec skills。

開始前先載入 `flow-conventions` skill 取得共用規範；環境沒有 Skill 工具時，改讀同層目錄的
`../flow-conventions/SKILL.md`，且必須讀完整份。所有決策點提問依其第 1 節；需求釐清本身一律維持自由對話。

開始前先做 flow-conventions 的「執行前檢查」；未通過就停下並建議 `/flow-init`。

## 核心原則

- **使用者說意圖，不必選 skill。** Flow 依上下文提出路線建議與理由；使用者可確認或覆寫。
- **會建立或修改 artifacts 的動作，先取得明確同意。**
- **一般釐清不等於壓力測試。** `grill-me` 只挑戰已成形的方案，不得作為「繼續討論」的預設行為。
- **已有 change 時，每輪有明確結論的討論，都要同步 artifacts。** 尚未建立 change 時，已確認結論在使用者同意 proposal 後一併納入。
- **本 skill 不實作 application code。** 要實作時引導 `/flow-apply <change-name>`。

## 1. 理解上下文並建議路線

先以自由對話理解問題、目標、範圍、限制與成功條件，不要一開始要求使用者選 skill 或固定流程。

使用者指明既有 active change，或敘述明確是在調整既有規劃時，先執行：

```bash
openspec status --change "<name>" --json
```

使用回傳的 `changeRoot`、`artifactPaths` 與 `actionContext` 取得實際路徑；不得假設固定在 repo-local `openspec/` 目錄。

### 1.1 路由判斷

| 訊號 | 建議路線 | 使用 skill |
|---|---|---|
| 需求模糊、方案未定、可行性未知、多個取捨未收斂 | 探索 | `openspec-explore` |
| 跨模組耦合、責任邊界不清、概念混亂、難測試、改一處壞多處 | 架構評估 | `improve-codebase-architecture` |
| 目標、範圍與成功條件已足夠明確，準備建立規劃 | 建立 change | `openspec-propose` |
| 既有 active change 的決策、設計或 scope 需要調整 | 探索後更新 | `openspec-explore` → `openspec-update-change` |

### 1.2 說明建議並取得確認

Flow 必須說明判斷理由，並讓使用者採用或覆寫；不得靜默進入架構評估或建立 artifacts。依提問協定（單選）呈現：

```text
❓ 我建議先走「<路線>」，原因：
- <觀察到的上下文訊號>
- <為何此路線比替代方案合適>

  1) 採用建議路線  ←預設
  2) 改為探索討論
  3) 直接建立 OpenSpec change
  4) 先說明其他方向
回覆：`0` 套用預設／數字／或自然語言覆寫
```

可省略不適用選項，但必須保留「採用建議」與自然語言覆寫。

### 1.3 Change 類型

建立新 change 時，type 限定 `feature`、`bug-fix`、`refactor`。若可從上下文明確推斷，Flow 要說明推斷與理由，並讓使用者覆寫；不明確才依提問協定詢問三選一。使用者輸入 migrate、chore 等其他分類時，先說明最接近的三種分類並確認後才寫入。

在 change 建立後，將確認的 type 寫入 `.flow.yaml`：

```yaml
type: feature # 或 bug-fix / refactor
```

## 2. 執行選定路線

### 2.1 探索路線

呼叫 `Skill(openspec-explore)`：可讀程式碼、既有 artifacts 與相關文件，並以自由對話釐清需求、比較方案、揭露風險與未知項；不得寫 application code。需求足以正式規劃時，詢問是否進入 proposal 路線。

### 2.2 架構評估路線

第 1.2 節確認後呼叫 `Skill(improve-codebase-architecture)`。委派契約：

- 只分析現況、模組責任、依賴方向、概念邊界與遷移選項。
- 輸出現況問題、2–3 個可行方向、取捨與建議遷移策略。
- 不改 application code、不 commit、不建立獨立流程文件。
- 結論一律使用繁體中文（zh-TW）。
- 評估結果在使用者同意建立 proposal 前只存在於 Flow 對話脈絡，不得另行落地。

評估完成後，自由討論確認採用方向；使用者同意建立 change 時，進入 proposal 路線。

### 2.3 Proposal 路線

使用者明確同意建立 change 後，呼叫 `Skill(openspec-propose)`，並傳達所有 artifacts 一律以繁體中文（zh-TW）撰寫，以及已確認的需求、範圍、限制、驗收條件與架構評估結論。確認 artifacts 位於 CLI 回傳的 `changeRoot` 後，進入第 3 節。

### 2.4 既有 Change 的更新路線

先以 `openspec-explore` 釐清改動方向，再整理本輪已確認的決策差異，最後呼叫 `Skill(openspec-update-change)`。委派時必須要求：只更新 planning artifacts、不改 code、讀取所有既有 artifacts 檢查雙向一致性，且每份 artifact 寫入前展示修改內容與理由，取得使用者確認；修改內容一律用繁體中文。

若改變的是 change 的根本意圖，而非原規劃修訂，建議建立新的 change，不得混入原 change。

## 3. Artifact Review Loop

初版 artifacts 建立完成，或既有 artifacts 完成本輪更新後，回報：本輪已確認決策、已更新 artifacts 與摘要、尚未確認問題或風險、以及與上一輪相比的主要差異。

再依提問協定（單選）詢問：

```text
❓ 目前文件已更新，下一步要做什麼？

  1) 文件定稿，進入提交
  2) 繼續釐清／補充需求  ←預設
  3) 壓力測試目前方案
  4) 重新評估範圍或架構
回覆：`0` 套用預設／數字／或自然語言覆寫
```

### 3.1 繼續釐清／補充需求

使用自由對話，呼叫或持續使用 `openspec-explore`。有明確結論時整理「本輪決策差異」，經使用者確認後委派 `openspec-update-change` 回填 artifacts，接著回到本節選單。

### 3.2 壓力測試目前方案

只有已有可描述的方案、design 或決策時才可呼叫 `Skill(grill-me)`。執行前限定本輪主題，例如快取失效與資料一致性、多來源解析的擴充性、錯誤處理與重試策略，或遷移策略與相容性。

輸出必須整理為：接受的 finding 與採用決策、不採用的 finding 與理由、尚待確認的風險、需要更新的 artifacts。不得形成無限追問；一輪結束後回到 Flow，由使用者決定是否繼續。有採用的 finding 時，經確認後委派 `openspec-update-change` 同步 artifacts，再回到本節選單。

### 3.3 重新評估範圍或架構

原目標或成功條件已改變、問題超出原 change、發現跨模組架構問題、或原方案無法成立時，建議此路線。先說明訊號並詢問是否：回到 `openspec-explore`、呼叫 `improve-codebase-architecture`、建立新的 change，或保留原 change 僅記錄待辦。選擇待辦時可建議 `/flow-note`，不得自行建立 backlog 項目。

## 4. 定稿、提交與交接

使用者選擇「文件定稿」時：

1. 顯示最終問題與目標、已確認設計決策、產出 artifacts、尚存風險或刻意不處理範圍。
2. 確認 artifacts 位於選定 change 的 `changeRoot`。
3. 精確加入該 change 目錄並依 flow-conventions 的「commit 規則」提交（commit 意圖：完成
   `<change-name>` 的需求討論文件）：

   ```bash
   git add -- "<changeRoot>"
   ```

4. commit 成功後依提問協定（單選）詢問：

   ```text
   ❓ 討論文件已提交，要推送到遠端嗎？
     1) 推送
     2) 暫時不推送  ←預設
   ```

5. 使用者同意才執行 `git push`。結尾提示：

   ```text
   準備實作時，執行 `/flow-apply <change-name>`。
   ```

commit 或 push 失敗時，保留現場、附原始錯誤並停止；不得回退已成功建立的 commit。

## NEVER

- **NEVER** 在討論階段寫 application code；要實作時引導 `/flow-apply`。
- **NEVER** 呼叫 `superpowers:brainstorming`；需求探索由 `openspec-explore` 處理。
- **NEVER** 將「繼續討論」自動等同於 `grill-me`。
- **NEVER** 未經確認就呼叫 `improve-codebase-architecture` 或 `openspec-propose`。
- **NEVER** 未經確認就修改 OpenSpec artifacts；更新一律委派 `openspec-update-change`，逐 artifact 確認。
- **NEVER** 在已有 change 時將已確認結論只留在對話中；每輪結論都必須回填 artifacts。尚未建立 change 時，結論在使用者同意 proposal 後一併納入。
- **NEVER** 把自由討論硬套成編號選單；選單只用於會改變流程的決策點。
- **NEVER** 把自由文字分類直接寫入 `.flow.yaml`；type 只能是 feature、bug-fix、refactor。
- **NEVER** 以繁體中文以外的語言撰寫 OpenSpec artifacts。
- **NEVER** 未選擇「文件定稿」就 commit，或未同意就 push。
- **NEVER** 在提交時加入 change 目錄以外的檔案。
- **NEVER** 靜默忽略 OpenSpec CLI 或委派 skill 的錯誤；明確回報後停止。
