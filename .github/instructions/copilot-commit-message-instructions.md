# Commit Message Instructions

本專案採用 **Conventional Commits** 規範來標準化 commit 訊息，讓 commit 歷史更清楚且便於自動化工具處理。

## Conventional Commits 規範

### 基本格式

```
<type>([optional scope]): <description>

[optional body]

[optional footer(s)]
```

### 範例

```
feat(java): add effective java item 5 notes
docs: update documentation instructions
fix(css): correct mobile responsive layout
```

## Type 類型說明

| Type | 說明 | 範例 |
|------|------|------|
| `feat` | 新功能 | `feat: add new blog post` |
| `fix` | 錯誤修正 | `fix: correct typo in java notes` |
| `docs` | 文檔變更 | `docs: update README` |
| `style` | 格式調整（不影響程式邏輯） | `style: format code indentation` |
| `refactor` | 重構程式碼 | `refactor: reorganize folder structure` |
| `chore` | 雜項工作（不影響主要功能） | `chore: update dependencies` |
| `build` | 建構系統相關 | `build: update mkdocs configuration` |
| `ci` | CI/CD 相關 | `ci: add github actions workflow` |
| `test` | 測試相關 | `test: add unit tests` |
| `perf` | 效能優化 | `perf: optimize image loading` |

## Scope 範圍（可選）

用來指定變更的範圍或模組：

- `java` - Java 相關筆記
- `python` - Python 相關筆記
- `docs` - 文檔相關
- `css` - 樣式相關
- `config` - 配置檔案
- `blog` - 部落格文章

## 實際範例

### 新增內容

```bash
feat(java): add effective java item 1-5 notes
feat(blog): add post about dependency injection
feat(docs): add clean architecture chapter 7
```

### 修正錯誤

```bash
fix(java): correct code example in singleton pattern
fix(css): resolve mobile layout issues
fix(docs): fix broken links in navigation
```

### 文檔更新

```bash
docs: update project README
docs(instructions): add commit message guidelines
docs: reorganize folder structure explanation
```

### 重構和整理

```bash
refactor(java): simplify dependency injection examples
style(docs): improve markdown formatting
chore: organize file structure
```

## 撰寫原則

### 1. 使用繁體中文

- **Description 使用繁體中文**：`feat: 新增 Java 筆記`
- **Body 詳細說明可用中文**

### 2. 動詞使用現在式

- ✅ `add`、`update`、`fix`、`remove`
- ❌ `added`、`updated`、`fixed`、`removed`

### 3. 首字母小寫

- ✅ `feat: add new feature`
- ❌ `feat: Add new feature`

### 4. 簡潔明確

- 描述要具體說明做了什麼
- 避免過於籠統的描述
- 一個 commit 專注於一個主要變更

## 實務建議

### 適合的 Commit 大小

- **小且專注**：一個 commit 處理一個邏輯變更
- **原子性**：可以獨立回滾的變更單位
- **有意義**：每個 commit 都有明確目的

### 常見模式

```bash
# 新增文章或筆記
feat(java): 新增 effective java item 6 筆記

# 整理既有內容
refactor(docs): 重新整理 dependency injection 說明

# 修正內容錯誤
fix(java): 修正 builder pattern 程式碼範例

# 更新文檔結構
docs: 更新文檔撰寫指引

# 樣式和格式調整
style: 統一程式碼區塊格式
```

這樣的規範讓 commit 歷史更清楚，也便於後續維護和回顧變更。
