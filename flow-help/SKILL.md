---
name: flow-help
description: flow 系列與 openspec 系列 skill 的導覽說明。當用戶說「flow 怎麼用」、「該用哪個 flow」、「flow 系列有哪些」、「openspec 跟 flow 差在哪」、「我現在該做什麼」、「驗收工具怎麼設定」、「怎麼多加一個 code review 工具」或使用 /flow-help 時觸發。簡短說明每個 skill 的用途與使用時機，搭配情境案例協助用戶挑對 skill，並附驗收工具設定（verify_tools）速查。
effort: low
---

# flow-help：flow／openspec 系列導覽

不知道該用哪個 skill 時的查詢入口。被觸發時，依用戶的問題挑選下方
相關段落回答；用戶沒有具體問題時，呈現「全貌」與「情境對照」兩節即可，
不用整份倒出來。

## 全貌

flow 系列是 **編排層**，包住 openspec 系列（**文件與任務引擎**），
再加上分支管理、驗收清單、worktree 清理等專案慣例。
一個 change 的完整生命週期：

```
/flow-discuss ──→ /flow-apply ──→ /flow-verify ──→ /flow-finish
   需求討論         建分支實作        驗收檢查         歸檔+合併
      │                │                                │
  openspec-explore  openspec-        （單元測試、文件    openspec-
  openspec-propose  apply-change      註解、review…）    archive-change
```

日常只需要記四個 `/flow-*` 指令，openspec skills 由 flow 自動委派。
另有隨時可用的 `/flow-note`（開發備忘錄），不屬於生命週期的任一階段。
專案第一次使用 flow 前先執行 `/flow-init`；之後各 flow 檢查到缺少設定或依賴時，
也會提示重新執行它。

## flow 系列（主要入口）

| Skill | 做什麼 | 什麼時候用 |
|---|---|---|
| `flow-init` | 導入與重新檢查：確認 openspec 與 Matt skills 已安裝、必要時執行 `openspec init`，補齊 `config.yaml` 的 context／rules／operations 與 commit 格式；可重複執行，只補差異 | 新專案第一次用 flow，或 flow 提示缺少設定／依賴時 |
| `flow-discuss` | 需求討論：分流模糊想法（explore）與明確需求（propose），產出 openspec 文件；收尾確認後 commit + push 討論產物 | 有想法、需求、bug、重構要討論時，**一切的起點** |
| `flow-apply` | 依 change 類型建分支（feature/、bug-fix/、refactor/）或 worktree（選 worktree 會再問要同 session 切過去或新開 session；可帶 `-b`／`-w` 直接指定免問），委派實作 tasks | 討論完、要開始寫 code 時 |
| `flow-verify` | 開場勾選驗收項目（單元測試／文件註解／code-review）後依序執行；review findings 逐條由你決定改 code、改 spec 或不處理；中斷後程式碼沒變可接續 | tasks 全部完成、要驗收時 |
| `flow-finish` | 提交剩餘的驗收修正、tasks 未全勾時可選擇代為勾選並 commit 後繼續、歸檔 spec，每次選擇本地合併／建 PR／保留分支，清理分支與 worktree | 驗收通過、要收尾整合時 |
| `flow-note` | 開發備忘錄：把途中發現的缺陷、被依賴卡住的待辦記到 `openspec/backlog.md`，可查看與更新狀態 | 實作中發現超出範圍的事想先記下來時，**任何階段都可用** |
| `flow-conventions` | 共用規範（提問協定、`.flow.yaml`、commit 規則、執行前檢查等），`user-invocable: false`，由其他 flow skill 載入；驗收專屬規則在 flow-verify 的 `references/verify-tools.md` | 想查規範細節時直接閱讀 |

## openspec 系列（引擎層，通常不直接呼叫）

| Skill | 做什麼 | 被誰委派 |
|---|---|---|
| `openspec-explore` | 思考夥伴模式：探索想法、釐清問題，不產 code | flow-discuss（模糊路線） |
| `openspec-propose` | 一次產出 proposal.md／design.md／specs／tasks.md | flow-discuss（明確路線） |
| `openspec-apply-change` | 照 tasks.md 逐項實作並勾選進度 | flow-apply |
| `openspec-update-change` | 修訂既有 design.md／specs／tasks.md 並確保彼此一致，不動 code | flow-apply（實作中發現偏差、經詢問同意後） |
| `openspec-archive-change` | 同步 delta specs 並把 change 搬進 archive | flow-finish |

單獨呼叫的合理時機：只想跟 AI 對談釐清想法、完全不打算進開發流程
→ 直接用 `openspec-explore`。其餘情況一律走 flow 入口，
否則會漏掉分支命名、`.flow.yaml`、驗收紀錄等編排步驟。

## 情境對照

| 你的狀況 | 用這個 |
|---|---|
| 「新專案要開始用 flow」／flow 提示要先初始化 | `/flow-init` |
| 「我有個模糊的想法，還不確定要做什麼」 | `/flow-discuss` → 選「模糊想法／還在探索」 |
| 「需求很明確，幫我規劃怎麼做」 | `/flow-discuss` → 選「明確需求開發」 |
| 「有個 bug 要修」／「這段想重構」 | `/flow-discuss` → 選 bug 修復／重構 |
| 「討論完了，開工吧」 | `/flow-apply <change-name>`（加 `-b` 用 branch、`-w` 用 worktree，免問工作區） |
| 「上次做到一半，繼續實作」 | `/flow-apply <change-name>`（會接續未勾選的 tasks） |
| 「code 寫完了，幫我驗收」 | `/flow-verify <change-name>` |
| 「驗收做到一半中斷了」 | `/flow-verify <change-name>`（程式碼沒變會提議只跑剩下的項目） |
| 「驗收過了，要合併／發 PR」 | `/flow-finish <change-name>` |
| 「現在有哪些 change？做到哪了？」 | `openspec list`；細節看 `openspec/changes/<name>/.flow.yaml` |
| 「實作到一半發現別的缺陷，之後再修」 | `/flow-note` 記下來，繼續原本的 Task |
| 「這裡要等另一個功能做完才能改」 | `/flow-note` 記下來並註明前置依賴 |
| 「討論到一半想寫 code」 | 先完成 flow-discuss 收尾，再 `/flow-apply`——討論階段不寫實作碼 |
| 「實作中發現 bug 或跟原規劃不符，討論調整後改了 code，文件要跟著改嗎」 | `/flow-apply` 會主動詢問是否委派 `Skill(openspec-update-change)` 同步文件 |
| 「只想聊聊想法，不打算開發」 | 直接 `Skill(openspec-explore)` |

## 驗收工具設定（速查）

**大多數時候你不用碰這個檔**——flow-verify 在已有有效快取時直接跑；首次使用
或快取失效時會掃描環境、依提問協定問你選，然後自動寫回 `openspec/config.yaml`。

要手改時的結構：

```yaml
flow:
  verify_tools:
    code_review:
      - name: team-code-review
        kind: skill             # skill | command，決定怎麼呼叫
        platforms: [android]    # 省略 = 適用所有偵測到的平台
        default: true           # 開場提問時是否預先勾選
      - name: /code-review
        kind: command
        hosts: [claude-code]    # 省略 = 任何環境可用
        internal_parallelism: true   # 自己會 spawn 子代理，由主 agent 直接執行
        default: true
      - name: codex:rescue
        kind: skill
        hosts: [claude-code]
        default: false          # 是候選，但不預勾
```

`unit_test`、`doc_comment` 用同一種結構，只是通常每個平台各一筆。

| 問題 | 做法 |
|---|---|
| 想多加一個 review 工具 | `code_review` 底下多一筆，填 `name` 與 `kind` |
| 這次只想跑其中一個 | **不用改檔**——開場提問時直接說「3 只跑 team-code-review」 |
| 某個工具只有我的環境有 | 加 `hosts:`，其他環境會自動跳過並重新掃描補位 |
| 完整欄位定義 | `flow-verify/references/verify-tools.md` 的「驗收相關設定」 |
| 掃描／快取失效／找不到工具時的行為 | 同上的「委派工具」 |

## 常見疑問

- **flow 和 openspec 差在哪？** openspec 負責「文件跟任務」；flow 負責
  「流程跟環境」（分支、worktree、驗收清單、commit/push、清理）。
  用 flow 就同時拿到兩者。
- **可以跳過某個階段嗎？** 可以但不建議跳過 flow-discuss——沒有
  openspec change 目錄，後面三個階段都無從接手。flow-verify 的項目
  可在開場勾選時取消，flow-finish 會警告但不強制擋。
- **狀態存在哪？** 每個 change 目錄的 `.flow.yaml`（type、branch、
  驗收紀錄），跨階段、跨 session 傳遞，詳見 flow-conventions 的「`.flow.yaml` 結構」。

## NEVER

- **NEVER 在本 skill 內執行任何流程**——flow-help 只做說明與導引，
  用戶決定後改觸發對應的 flow skill，不代替它們動手。
- **NEVER 猜測用戶所處階段**——不確定時列出「情境對照」讓用戶自己對號，
  或查 `openspec list` 確認現況後再建議。
