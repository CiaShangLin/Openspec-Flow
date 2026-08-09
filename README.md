# OpenSpec Flow

以 [OpenSpec](https://github.com/Fission-AI/OpenSpec) 為底層文件與任務引擎，
在上面包一層「編排層」：把需求討論、建立工作區、實作、自動化驗收、收尾歸檔
串成四個階段，並補上分支管理、worktree 清理、驗收清單等專案慣例。

```text
discuss → apply → verify → finish
```

![OpenSpec Flow 總覽](assets/medium/openspec-flow-overview.png)

這是個人 workflow 擴充，不是 OpenSpec 的官方建議流程。使用前建議先讀過
`flow-conventions/conventions.md`，理解各 skill 共用的提問協定與設定結構。

## 目錄結構

![Flow Skills 與委派關係](assets/medium/openspec-flow-skill-map.png)

| 目錄 | 用途 |
| --- | --- |
| `flow-discuss/` | 需求討論分流：模糊需求走 `openspec-explore`，明確需求走 `openspec-propose`；偵測到方向發散或假設未被挑戰時，經同意引入 `brainstorming` 或 `grill-me`。 |
| `flow-apply/` | 建立分支或 worktree，委派 `openspec-apply-change` 依 `tasks.md` 實作；規格與現況不符時經確認後同步。 |
| `flow-verify/` | 自動化驗收編排：單元測試、文件註解、code review 等項目依平台委派對應工具，結果寫回 `.flow.yaml`。 |
| `flow-finish/` | 歸檔 change、同步 delta specs，交給既有 skill 決定合併／PR／保留／丟棄，並補做 worktree 清理。 |
| `flow-note/` | 開發途中的備忘錄，記到 `openspec/backlog.md`，不自動整合進其他 skill。 |
| `flow-help/` | 導覽入口，說明各 skill 用途、觸發時機與驗收工具設定速查。 |
| `flow-conventions/` | 所有 flow skill 共用的規範來源：提問協定、分支命名、`.flow.yaml` 結構、`config.yaml` 的 `flow:` 區塊、平台偵測與工具委派、錯誤處理原則。 |
| `assets/` | 說明文件用的圖片素材。 |

## 安裝

把上述 `flow-*` 目錄整組複製進你環境對應的 skills 目錄（例如 Claude Code 的
`.claude/skills/`），並保持 `flow-conventions/` 與各 `flow-*` 目錄同一層級——
skill 內部以相對路徑 `../flow-conventions/conventions.md` 讀取共用規範。

底層仍需先安裝好 OpenSpec：

```bash
npm install -g @fission-ai/openspec@latest
cd your-project
openspec init
```

## 設計原則

![討論分流規則](assets/medium/openspec-flow-routing.png)

- **平台差異寫進設定，不寫死在流程。** `flow-verify` 不需要知道每個工具的
  細節，只依 `openspec/config.yaml` 的 `verify_tools` 選對工具，讓同一套流程
  同時照顧 Android／iOS 或其他平台組合。
- **skill 不因存在就被強制觸發。** `brainstorming` 適合選項還沒收斂的時候，
  `grill-me` 適合方案成形但假設未被挑戰的時候；是否引入一律先詢問。
- **討論可以自由，交付物保持一致。** 不論前面用了哪些輔助 skill，最終都收斂
  回 OpenSpec 的 proposal、design、spec 與 tasks。
