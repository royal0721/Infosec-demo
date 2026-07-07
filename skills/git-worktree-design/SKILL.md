---
name: git-worktree-design
description: 當使用者需要一次新增多個功能，或其需求適合拆分為多個 feature branch 並行開發時，或訊息中出現「worktree」、「git worktree」、「多分支開發」、「parallel branches」等關鍵字，自動觸發本 Skill。流程為：先分析需求並提出 worktree 拆分方案，經使用者確認後再建立。
---

# Git Worktree Design — 聰明的平行開發

分析使用者需求，判斷是否適合以 `git worktree` 拆分為多個 feature branch 平行開發；若適合，提出完整方案並在確認後執行建立。

---

## 流程總覽

分析當前狀態 → 全局設計（拆分方案 + Feature Spec）→ 建立 worktree → 安裝依賴 → 寫入 Spec 檔案 → 確認結果。整體採「先設計、後執行」原則：所有規劃在建立任何 worktree 之前一次完成並交付使用者審閱。

---

## 1. 分析當前狀態

在提出任何方案前，先掌握 repo 的現況：

```bash
# 確認當前分支
git branch --show-current

# 列出既有 worktree
git worktree list

# 取得 remote 資訊
git remote -v

# 確認工作目錄狀態
git status --short
```

若存在未提交的變更，先提醒使用者處理（`commit` 或 `stash`）後再繼續，以免影響後續分支建立。

---

## 2. 全局設計（拆分方案 + Feature Spec）

在建立任何 worktree 之前，先從全局視角完成所有 feature 的設計。本步驟同時產出**拆分方案**與各 feature 的 **Spec 草案**，讓使用者一次審閱整體規劃，而非逐一補件。

### 2a. 需求拆分

依據使用者需求，拆分為多個彼此獨立的 feature branch。拆分時遵循下列原則：

| 原則           | 說明                                                     |
| -------------- | -------------------------------------------------------- |
| **功能獨立性** | 每個 worktree 只負責一項獨立功能，降低跨分支衝突         |
| **最小相依**   | 盡量避免分支間相互依賴，確保能獨立開發與測試             |
| **合理粒度**   | 不宜過細（徒增管理負擔），亦不宜過粗（失去平行開發效益） |
| **命名語意**   | 分支名稱應清楚描述功能，統一採 `feature/<功能名>` 格式   |

### 2b. 設計各 Feature Spec

針對每個拆分出的 feature，依使用者的原始需求與專案現況**主動推導**出完整的 Spec 內容，而非提供空白範本。每份 Spec 應涵蓋下列區塊：

| 區塊           | 說明                                                                                       |
| -------------- | ------------------------------------------------------------------------------------------ |
| **目標**       | 用一兩句點出此分支要解決的問題與預期成果——聚焦「為什麼做」與「做完長什麼樣」               |
| **實作範圍**   | 拆成可逐項勾選的任務清單，粒度細到 AI 讀了即可動手；並界定「不做什麼」以防範圍蔓延         |
| **驗收標準**   | 以可驗證的方式描述「怎樣算完成」：具體行為、輸入/輸出、UI 狀態或可執行的測試，避免主觀形容 |
| **技術約束**   | 必須遵守的邊界：專案慣例與架構、禁止引入的依賴、須相容的既有介面與資料結構                 |
| **跨分支備註** | 與其他 feature 的相依關係、共用檔案/元件的潛在衝突、建議的合併先後順序                     |

> **重點**：全局設計階段即需納入跨分支相依的考量。例如：feature A 產出的共用元件是否影響 feature B、各分支之間的合併是否有先後順序。

### 2c. 向使用者呈現完整方案

以「總表 + 各 feature Spec 摘要」的形式，向使用者呈現**完整方案**：

```
📋 Worktree 拆分方案（共 N 個分支）

| # | 分支名稱 | Worktree 目錄 | 負責功能 |
|---|----------|---------------|----------|
| 1 | feature/auth-refactor | ../project-auth | 統一認證與 session 管理（共用基礎） |
| 2 | feature/billing-page | ../project-billing | 訂閱計費頁面 |
| 3 | feature/audit-log | ../project-audit | 操作稽核紀錄 |

---

📝 Feature Spec 摘要：

### 1. feature/auth-refactor
- 目標：抽出統一的認證與 session 管理，供其他功能共用
- 驗收：登入/登出流程正常，token 過期可自動更新，既有頁面不受影響
- 相依：無，但為其他分支的基礎，建議最先合併

### 2. feature/billing-page
- 目標：新增月/年切換的訂閱計費頁面
- 驗收：切換月/年時價格正確更新，未登入自動導向登入頁，手機版排版正常
- 相依：需 auth-refactor 提供的使用者狀態，建議於 1 之後合併

### 3. feature/audit-log
- 目標：新增使用者操作的稽核紀錄與查詢介面
- 驗收：關鍵操作皆留下紀錄，查詢可依時間與使用者篩選
- 相依：無，可獨立開發與合併

---

建議合併順序：1 →（2、3）；其中 2 依賴 1，3 可於任意時間合併

確認執行？(Y/n)
```

使用 `notify_user` 工具展示完整方案（拆分 + Spec 摘要），並等待使用者確認後再進入下一步。

---

## 3. 建立 Worktree

使用者確認後，依序為每個 feature 建立 worktree 與新分支：

```bash
# 建立各 worktree（新分支）
git worktree add -b <branch_name> <worktree_path>
```

### Worktree 目錄命名規則

- 目錄置於當前 repo 的**同層級**（`../`）。
- 格式：`../<project-name>-<feature-short-name>`。
- 以 repo 目錄名作為 `<project-name>` 前綴，避免與其他專案混淆。

---

## 4. 安裝依賴

偵測專案使用的套件管理器，並為每個 worktree 各自安裝依賴：

```bash
# 依 lock file 判斷套件管理器
# pnpm-lock.yaml    → pnpm install
# yarn.lock         → yarn install
# package-lock.json → npm install
# bun.lockb         → bun install
```

對每個 worktree 執行：

```bash
cd <worktree_path> && <package_manager> install
```

> **注意**：每個 worktree 擁有獨立的工作目錄，`node_modules` 不會共享，必須各自安裝。

---

## 5. 寫入 Feature Spec 檔案

將步驟 2 設計好的 Spec 內容，透過 `write_to_file` 工具寫入各 worktree 根目錄下的 `<worktree_path>/git-worktree-spec.md`。

### Spec 檔案格式

```markdown
# Feature Spec: <功能名稱>

> 本文件由 Git Worktree Design Skill 自動產生，供 AI Agent 作為開發指引。

## 分支資訊

| 項目          | 值                |
| ------------- | ----------------- |
| 分支名稱      | `feature/<name>`  |
| 基於分支      | `<base_branch>`   |
| Worktree 路徑 | `<absolute_path>` |
| 建立時間      | `<timestamp>`     |

## 目標

（填入步驟 2 的設計內容）

## 實作範圍

- [ ] 具體任務 1
- [ ] 具體任務 2
- [ ] 具體任務 3

## 驗收標準

- 條件 A 成立時，應有行為 X
- 條件 B 成立時，應有行為 Y

## 技術約束

- 不得引入新的 npm 依賴（視情況填寫）
- 須相容既有的設計系統 / API 介面

## 跨分支備註

與其他 worktree 分支的相依關係、合併順序建議等。
```

---

## 6. 確認結果

所有 worktree 建立完成後執行：

```bash
git worktree list
```

並以表格向使用者呈現最終結果：

```
✅ Worktree 建立完成！

| Worktree 目錄 | 分支 | 狀態 | Spec |
|---------------|------|------|------|
| /path/to/project-auth | feature/auth-refactor | ✅ 就緒 | ✅ 已寫入 |
| /path/to/project-billing | feature/billing-page | ✅ 就緒 | ✅ 已寫入 |
| /path/to/project-audit | feature/audit-log | ✅ 就緒 | ✅ 已寫入 |

💡 提示：
- 切換工作目錄至對應 worktree 即可開始開發
- 所有 worktree 共享同一個 .git，commit 歷史互通
```

---

## 邊界情況處理

| 情況              | 處理方式                                                                        |
| ----------------- | ------------------------------------------------------------------------------- |
| **分支已存在**    | 改用不帶 `-b` 的指令（`git worktree add <path> <existing-branch>`），並提示確認 |
| **目錄已存在**    | 提示衝突，並建議替代目錄名                                                      |
| **有未提交變更**  | 提醒先 `commit` 或 `stash`                                                      |
| **遠端分支同步**  | 建議先 `git fetch` 取得最新遠端狀態                                             |
| **Worktree 清理** | 提醒開發完成後以 `git worktree remove` 與 `git branch -d` 清理                  |

---

## 常用維護指令

```bash
# 列出所有 worktree
git worktree list

# 移除 worktree（保留分支）
git worktree remove <path>

# 強制移除（有未提交變更時）
git worktree remove --force <path>

# 清理失效的 worktree 參照
git worktree prune

# 刪除分支（合併後）
git branch -d <branch_name>
```
