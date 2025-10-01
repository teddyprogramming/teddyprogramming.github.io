# Commit Message Instructions

本專案採用 **Conventional Commits** 規範來標準化 commit 訊息，讓 commit 歷史更清楚且便於自動化工具處理。

## Conventional Commits 規範

### 基本格式

```
<type>: <description>

[optional body]

[optional footer(s)]
```

### 範例

```
feat: add effective java item 5 notes
docs: update documentation instructions
fix: correct mobile responsive layout
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

## 實際範例

### 新增內容

```bash
feat: add effective java item 1-5 notes
feat: add post about dependency injection
feat: add clean architecture chapter 7
```

### 修正錯誤

```bash
fix: correct code example in singleton pattern
fix: resolve mobile layout issues
fix: fix broken links in navigation
```

### 文檔更新

```bash
docs: update project README
docs: add commit message guidelines
docs: reorganize folder structure explanation
```

### 重構和整理

```bash
refactor: simplify dependency injection examples
style: improve markdown formatting
chore: organize file structure
```

## 撰寫原則

### 1. 動詞使用現在式

- ✅ `add`、`update`、`fix`、`remove`
- ❌ `added`、`updated`、`fixed`、`removed`

### 2. 首字母小寫

- ✅ `feat: add new feature`
- ❌ `feat: Add new feature`

### 3. 簡潔明確

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
feat: add effective java item 6 notes

# 整理既有內容
refactor: reorganize dependency injection explanation

# 修正內容錯誤
fix: correct builder pattern code example

# 更新文檔結構
docs: update documentation guidelines

# 樣式和格式調整
style: standardize code block formatting
```

這樣的規範讓 commit 歷史更清楚，也便於後續維護和回顧變更。
