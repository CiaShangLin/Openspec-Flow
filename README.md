# OpenSpec Flow

以 [OpenSpec](https://github.com/Fission-AI/OpenSpec) 為文件與任務引擎的流程編排 skills。它將需求討論、實作、驗收與收尾串為一致的 change 生命週期，並補足分支、worktree、checkpoint、驗收紀錄及收尾整合等專案慣例。

![OpenSpec Flow 總覽：flow-init 導入後，依序經過 discuss、apply、verify、finish 四個階段，各自委派 openspec 與其他 skills](assets/medium/openspec-flow-overview.png)

這是個人 workflow 擴充，不是 OpenSpec 的官方建議流程。各 skills 會共用
[`flow-conventions`](flow-conventions/SKILL.md) skill（`user-invocable: false`）的提問協定、
`.flow.yaml` 狀態格式、commit 規則與錯誤處理；驗收專屬規則在 `flow-verify/references/`。

## 內容

![flow 各階段步驟：橘色為需要使用者決定的關卡](assets/medium/openspec-flow-skill-map.png)

| 目錄 | 用途 |
| --- | --- |
| `flow-init/` | 導入與重新檢查：確認依賴、必要時執行 `openspec init`，補齊 `config.yaml` 的 context／rules／operations 與 commit 格式；可重複執行，只補差異。 |
| `flow-discuss/` | 以自由對話理解需求，建議 explore、架構評估、proposal 或既有 change 更新路線；確認後才建立或修改 OpenSpec artifacts。 |
| `flow-apply/` | 依 change 類型建立分支或 worktree（可帶 `-b`／`-w` 直接指定，免問），委派 `openspec-apply-change` 實作 tasks；遇到規格偏差或未知異常時先探索／診斷，完成後可建立語意化 checkpoint。 |
| `flow-verify/` | 依使用者選定項目執行單元測試、文件註解與 code review，每項派獨立 subagent 並帶入對應 effort，並記錄本輪驗收結果。review findings 列成編號清單附建議（改 code／改 spec+code／改 spec／不處理），由使用者逐條決定後才修正；中斷後若程式碼未變，可只跑剩下的項目。 |
| `flow-finish/` | 檢查驗收、提交剩餘的驗收修正、tasks 未全勾時可選擇代為勾選並 commit 後繼續、同步 delta specs、archive change 並推送，最後每次由使用者決定本地合併、建立 PR 或保留分支；不限 GitHub，中斷後可續跑。 |
| `flow-note/` | 開發途中的備忘錄，記到 `openspec/backlog.md`，不自動整合進其他 skill。 |
| `flow-help/` | 導覽入口，說明各 skill 用途、觸發時機與驗收工具設定速查。 |
| `flow-conventions/` | 所有 flow skill 共用的規範（`user-invocable: false`，由其他 flow skill 載入）：提問協定、`.flow.yaml` 結構、`config.yaml` 的 `flow:` 區塊、執行前檢查、commit 規則、錯誤處理原則。分支命名在 `flow-apply`，平台偵測與驗收工具委派在 `flow-verify/references/verify-tools.md`。 |

## 使用方式

日常只需記住 discuss、apply、verify、finish 四個主要入口，其餘按需使用：

| 情境 | 指令 |
| --- | --- |
| 專案第一次使用 flow，或 flow 提示缺少設定 | `/flow-init` |
| 有想法、需求、bug 或重構要規劃 | `/flow-discuss` |
| 已有 change，要開始或繼續實作 | `/flow-apply <change-name> [-b \| -w]` |
| tasks 完成，要跑驗收 | `/flow-verify <change-name>` |
| 驗收後要 archive、合併或建立 PR | `/flow-finish <change-name>` |
| 想記下之後再處理的事項 | `/flow-note` |
| 不確定該走哪個流程 | `/flow-help` |

`flow-discuss` 中的需求釐清維持自由對話；建立 artifacts、切換路線、提交或推送等決策則會明確詢問。所有 OpenSpec artifacts 一律以繁體中文（zh-TW）撰寫。

![flow-discuss 路線：依上下文建議探索、架構評估、建立或更新 change，經 Review loop 定稿後交給 flow-apply](assets/medium/openspec-flow-routing.png)

## 安裝

把上述 `flow-*` 目錄整組複製進你環境對應的 skills 目錄（例如 Claude Code 的
`.claude/skills/`），並保持 `flow-conventions/` 與各 `flow-*` 目錄同一層級——
各 skill 會載入 `flow-conventions` skill 取得共用規範。也可以包進自己的 plugin，
此時 skill 名稱會帶 plugin 前綴（例如 `<plugin>:flow-conventions`），不影響使用。

必要依賴：

| 依賴 | 安裝 |
| --- | --- |
| [OpenSpec](https://github.com/Fission-AI/OpenSpec) CLI | `npm install -g @fission-ai/openspec@latest` |
| [Matt Pocock skills](https://github.com/mattpocock/skills)（`grill-me`、`improve-codebase-architecture`、`diagnosing-bugs`、`code-review`） | Claude Code：`claude plugin install mattpocock-skills@claude-plugins-official`；其他環境：`npx skills@latest add mattpocock/skills` |

選用依賴（缺少不阻擋）：

| 依賴 | 用途 |
| --- | --- |
| Matt 的 `setup-matt-pocock-skills` | 產生 `/code-review` 會讀取的 issue tracker 設定；`/flow-init` 偵測到專案沒有 `docs/agents/` 時會提議一次 |
| 任何可建立 skill 的工具 | 驗收缺少對應平台的工具時，可當場建立；沒有時改為文字提示 |

flow 不依賴 superpowers。

接著在專案目錄執行 `/flow-init`。它會檢查依賴、在需要時執行
`openspec init --language zh-TW`（同時安裝 openspec-* skills），並補齊 flow 需要的
專案設定。之後可隨時重跑，只會列出差異。

## 設計原則

- **使用者表達意圖，不必先選 skill。** Flow 依目前上下文提出建議路線與理由，使用者始終可以覆寫。
- **討論自由，交付物一致。** 探索、架構評估與壓力測試的結論，都在確認後回填 OpenSpec artifacts。
- **實作與規劃保持同步。** 發現規格偏差時先暫停、探索並更新規劃；未知或不穩定的失敗先建立可重現的診斷回路。
- **驗收與整合分離。** `flow-apply` 的 checkpoint 只保存實作；`flow-verify` 與 `flow-finish` 皆由使用者另行明確觸發。
- **設定集中在專案。** 所有 flow 設定只放 `openspec/config.yaml`；commit 訊息依 `flow.commit`（專案的 commit skill 或格式）產生，不寫死在 skill 裡，被 hook 擋下時不得繞過。
- **push 與整合由使用者決定。** apply 與 verify 不主動 push，留給使用者先看 diff；finish 每次詢問要本地合併、建立 PR 或保留分支，不綁定 GitHub。
- **review 結論由使用者決定。** 每條 finding 附建議（改 code、改 spec、兩者或不處理），使用者回覆前不修改任何內容。

## 更新日誌

### 2026-11-03

**新增**
- `flow-apply`：支援 `-b`（branch）／`-w`（worktree）參數，帶參數時跳過工作區類型提問；其餘決策點照常詢問，與 `.flow.yaml` 已記錄的工作區不符時先停下確認。
- `flow-finish`：tasks 未全勾時新增「剩餘 tasks 已完成，幫我勾選並 commit 後繼續收尾」選項，適用已實機測試但未回報的情況；有 artifact 未完成時不提供。
- `flow-init`：檢查 OpenSpec CLI 與 Matt Pocock skills、需要時執行 `openspec init --language zh-TW`，草擬 `config.yaml` 的 context／rules／operations 與 `flow.commit`，列出「現況 → 建議」差異後才寫入；可重複執行。
- 所有 flow skill 開場加上執行前檢查：專案未初始化或缺少 `flow.commit` 時停下並建議 `/flow-init`，不自行建立 `openspec/`。
- `flow-verify`：code-review findings 列成編號清單附建議，由使用者逐條決定；改變行為的 finding 先用 `openspec-update-change` 更新 artifacts 再改 code。
- `flow-verify`：新增 `.flow.yaml` 的 `verify_progress` 快照，中斷後若程式碼未變可只跑剩下的項目。
- `flow-finish`：發現未提交的驗收修正時，提議依 commit 規則提交後繼續（預設），不再直接停止。

**變更**
- commit 訊息改依 `openspec/config.yaml` 的 `flow.commit` 產生，移除寫死的訊息；被 commit-msg hook 擋下時依提示修正重試一次，不使用 `--no-verify`。
- `flow-finish` 的整合步驟改為自有選單（本地合併／建立 PR／保留分支，預設保留），不再依賴 superpowers；只在 GitHub／GitLab 且有 CLI 時呼叫 `gh`／`glab`，其他主機（如 Bitbucket）改用 git 判斷是否已合併，PR 則提供 push 輸出的網址。
- `flow-finish` 本地合併後，只有 base 分支在驗收後有新 commit 時才重跑測試；測試指令快取在 `flow.verify_commands.test`，不得串接 `rm -rf`。
- `flow-conventions` 由 `conventions.md` 改為 `user-invocable: false` 的 skill；驗收專屬規則移到 `flow-verify/references/verify-tools.md`，分支命名移到 `flow-apply`。
- 跨檔案引用改用章節名稱，不再使用章節編號。
- 範例改用通用名稱（如 `android-unit-testing`、`team-code-review`）。
- 重畫 README 的三張流程圖（總覽、`flow-discuss` 路線、各階段步驟），補上 `flow-init`，並對齊目前 `flow-apply`、`flow-verify`、`flow-finish` 的步驟與決策點。

### 2026-08-31

- 同步最新的 workflow skills，並更新流程圖。

### 2026-08-09

- 首次發布 OpenSpec Flow skills。
