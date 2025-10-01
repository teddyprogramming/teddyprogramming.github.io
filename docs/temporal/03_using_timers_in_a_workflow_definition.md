# Workflow 中的 Timer 使用

## Timer 概念

**持久性計時器**：Temporal 的延遲機制

- **阻塞執行**：Workflow 暫停直到計時器觸發
- **資源效率**：由 Temporal Cluster 維護，Worker 等待時不消耗資源
- **時間範圍**：從一秒到數年

⚠️ **注意**：避免設定少於一秒的計時器，網路延遲會影響精確度

## 使用場景

- **固定間隔**：註冊後 1 天發送歡迎信、1 週發送指南
- **動態間隔**：根據前次結果調整下次執行時間
- **等待期間**：業務流程中的審核期、冷卻期

## Timer API

| 類型 | 特性 | 適用場景 |
|------|------|----------|
| `Workflow.sleep()` | 同步，立即阻塞 | 簡單循序操作 |
| `Workflow.newTimer()` | 非同步，設定時不阻塞 | 平行流程控制 |

⚠️ **重要**：切勿使用 `Thread.sleep()` 或 `java.util.Timer`，會導致 `WorkflowTaskTimedOut`

### 同步計時器

```java
// 基本用法
Workflow.sleep(Duration.ofSeconds(10));
Workflow.sleep(Duration.ofMinutes(1));
Workflow.sleep(Duration.ofHours(2).plusMinutes(30));
```

### 非同步計時器

```java
// 設定計時器但不阻塞
Promise<Void> timer = Workflow.newTimer(Duration.ofSeconds(30));

// 執行其他操作
performOtherTasks();

// 需要時才等待
timer.get();
```

## Timer 持久性

**由 Temporal Cluster 維護**：不依賴 Worker 運行狀態

**Worker 當機恢復情境**：

- **快速重啟**：繼續等待剩餘時間
- **延遲重啟**：計時器已觸發，立即繼續執行

**設計建議**：考慮 Worker 延遲恢復的情況，確保業務邏輯能適應執行時間超過預期
