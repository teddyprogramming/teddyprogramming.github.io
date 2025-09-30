# Workflow 中的計時器 (Timer) 使用

## Timer 基礎概念

**持久性計時器** 是 Temporal 提供的延遲機制，具有以下特性：

- **阻塞執行：** Workflow 暫停直到計時器觸發
- **資源效率：** 由 Temporal Cluster 維護，Worker 等待時不消耗資源
- **時間彈性：** 可設定從一秒到數年的延遲

⚠️ **注意事項：** 避免設定少於一秒的計時器，因為網路延遲會影響其精確度

## 常見使用場景

計時器可用於以下幾種情境：

### 1. 固定間隔執行活動

**實際應用：** 電子郵件提醒排程

- 註冊後 1 天：發送歡迎郵件
- 註冊後 1 週：發送使用指南
- 註冊後 1 個月：發送滿意度調查

### 2. 動態間隔執行

**實際應用：** 根據前次活動結果調整下次執行時間，實現自適應策略

### 3. 等待外部流程完成

**實際應用：** 在業務流程中插入必要的等待期（如審核期、冷卻期）

## 計時器 API 概覽

Temporal Java SDK 提供兩種計時器 API：

### API 類型對比

| 類型 | 阻塞行為 | 適用場景 | 主要方法 |
|------|----------|----------|---------|
| 同步計時器 | 立即阻塞執行 | 簡單循序操作 | `Workflow.sleep()` |
| 非同步計時器 | 設定時不阻塞 | 複雜平行流程 | `Workflow.newTimer()` |

⚠️ **安全警告：** 切勿在 Workflow 中使用 `Thread.sleep()` 或 `java.util.Timer`，會導致 `WorkflowTaskTimedOut` 錯誤

## 同步計時器使用方法

### 使用 Workflow.sleep

```java
import java.time.Duration;
import io.temporal.workflow.Workflow;

// 暫停 Workflow Execution 10 秒
Workflow.sleep(Duration.ofSeconds(10));
```

### 常見用法範例

```java
// 暫停各種時間長度
Workflow.sleep(Duration.ofMinutes(1));  // 1 分鐘
Workflow.sleep(Duration.ofHours(1));    // 1 小時
Workflow.sleep(Duration.ofDays(1));     // 1 天

// 複合時間
Workflow.sleep(Duration.ofHours(2).plusMinutes(30));  // 2小時30分鐘
```

## 非同步計時器使用方法

### 使用 Workflow.newTimer

```java
import java.time.Duration;
import io.temporal.workflow.Workflow;
import io.temporal.workflow.Promise;

// 建立非同步計時器
Promise<Void> timerPromise = Workflow.newTimer(Duration.ofSeconds(30));
logger.info("計時器已設定");

// 在需要等待時呼叫 get()
timerPromise.get();
logger.info("計時器已觸發");
```

### 同時執行多個計時器

```java
// 同時設定多個計時器
Promise<Void> timer1 = Workflow.newTimer(Duration.ofMinutes(5));
Promise<Void> timer2 = Workflow.newTimer(Duration.ofMinutes(10));

// 執行其他操作
performOtherTasks();

// 依序等待計時器完成
timer1.get();
logger.info("5 分鐘計時器完成");

timer2.get();
logger.info("10 分鐘計時器完成");
```

## 計時器的持久性與可靠性

### Worker 當機時的行為

計時器由 Temporal Cluster 維護，不依賴於 Worker 的運行狀態。當 Worker 呼叫 `Workflow.sleep` 或 `Workflow.newTimer` 時，Cluster 負責計時並在適當時機觸發。

### 兩種恢復情境

假設設定了 10 秒計時器，但 Worker 在 3 秒後當機：

**情境 1：快速重啟 (2 秒後)**

- Workflow 會繼續等待剩餘的 5 秒
- 使用體驗平滑，就像從未當機

**情境 2：延遲重啟 (20 分鐘後)**

- 計時器早已觸發，Worker 重啟後立即繼續執行
- 不會額外延遲，但總執行時間延長

### 設計建議

開發 Workflow 時應考慮 Worker 可能延遲恢復的情況，確保業務邏輯能夠適應實際執行時間超過預期的情況。
