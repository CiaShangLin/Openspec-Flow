---
name: flow-note
description: 開發備忘錄。當用戶說「記一下」、「這個之後再做」、「先記到備忘錄」、「等 XX 做完再改這裡」、「看一下備忘錄」、「備忘錄有哪些」或使用 /flow-note 時觸發。把開發途中發現的缺陷或被其他功能卡住的待辦記錄到 openspec/backlog.md，供之後建立新 change 時翻閱；也支援查看清單與更新條目狀態。
effort: low
---

# flow-note：開發備忘錄

純備忘錄，給用戶自己翻閱提醒用。開發途中發現超出當前 Task 範圍的缺陷、
或需等其他功能完成才能動工的修改，先記下來，之後由用戶自行決定
何時走 `/flow-discuss` 建立新 change 處理。

**不做跨 skill 自動整合**：flow-discuss / flow-finish 不會主動讀取
backlog，記錄與消化的節奏完全由用戶掌控。

開始前依 `../flow-conventions/conventions.md` 第 8 節的第 1、2 項確認專案已
初始化 OpenSpec；未初始化就停下並建議 `/flow-init`，不自行建立 `openspec/`。

## 記錄檔

- 位置固定為目標專案的 `openspec/backlog.md`。
- 檔案不存在時自動建立，表頭如下：

```markdown
# 開發備忘錄（Backlog）

開發途中發現、留待之後建立新 change 處理的項目，由 /flow-note 維護。
```

## 三種模式（依用戶輸入判斷）

### 1. 記錄（輸入帶有要記的內容）

1. 從用戶敘述萃取**標題**（簡短動詞句）與**描述**（一到數行，保留關鍵情境）。
2. 判斷**類型**（feature / bug-fix / refactor）；不明確時依
   `../flow-conventions/conventions.md` 第 0 節提問協定問一次。
3. **來源**自動偵測，不要問用戶：
   - `git branch --show-current` 取得當前分支，對照
     `openspec/changes/*/.flow.yaml` 的 `branch` 欄位，找到就填
     `<change-name>（實作中發現）`。
   - 對不到 change 就填分支名；不在 git repo 或偵測失敗填「無」。
4. **前置依賴**：用戶敘述有「等 XX 做完」「要先有 XX」才填，否則填「無」。
5. 依下方格式 append 到 `openspec/backlog.md` 末尾。
6. 回覆一句「已記錄到 openspec/backlog.md」即結束，讓用戶回到原本的工作。

條目格式：

```markdown
## 補上 Ktlint 自動修復功能
- 類型: bug-fix
- 記錄日期: 2026-07-17
- 來源: cloud-record-page（實作中發現）
- 前置依賴: 無
- 狀態: 待處理

Ktlint 在 XXX 情境下會誤報，需要補上自動修復邏輯。
```

狀態值固定三種：`待處理`、`已轉 change`、`已完成`。

### 2. 查看（無內容，或說「看一下備忘錄」）

- 讀 `openspec/backlog.md`，以精簡列表列出**待處理**項目
  （標題、類型、記錄日期、前置依賴）；用戶指明要看全部時才連同
  已轉 change／已完成一起列。
- 檔案不存在或沒有待處理項目 → 告知備忘錄是空的。
- 只列出，不主動建議建立 change——要不要處理由用戶決定。

### 3. 更新狀態（用戶說某條做完了／已經開 change 了）

- 依用戶描述比對條目標題，把該條「狀態」改為 `已完成` 或 `已轉 change`；
  已轉 change 時若知道 change 名稱，順手在狀態後標注
  （例：`已轉 change（fix-ktlint-autofix）`）。
- 比對不到或有多條相似 → 列出候選讓用戶挑，不猜。

## NEVER

- **NEVER 自動 commit backlog.md**——只寫檔。實作途中工作區常是髒的，
  插入 commit 會打亂節奏；讓它跟著用戶之後的 commit 自然進版。
- **NEVER 記錄完延伸討論或動手修那個問題**——這個 skill 的價值就是
  「記下來、不中斷」，記完立刻讓用戶回到原本的 Task。
- **NEVER 猜測狀態更新的對象**——比對不到條目就列候選詢問。
- **NEVER 未經用戶明確要求刪除條目**——完成用狀態標記，保留紀錄。
