# 了解 Event History

## 概述

Event History 是 Temporal 的核心機制，記錄 Workflow 執行過程中的所有狀態變化。

## Event History 結構

### Event 基本組成

每個 Event 都包含：

1. **Event ID**：歷史中的唯一識別符
2. **Event Type**：事件類型
3. **Event Time**：發生時間戳記
4. **Event Data**：特定於事件類型的資料

### 常見 Event 類型

- **WorkflowExecutionStarted**：Workflow 開始執行
- **ActivityTaskScheduled**：Activity 任務排程
- **ActivityTaskStarted**：Activity 開始執行
- **ActivityTaskCompleted**：Activity 執行完成
- **TimerStarted**：計時器啟動
- **TimerFired**：計時器觸發
- **WorkflowExecutionCompleted**：Workflow 執行完成

## Command 與 Event 對應

### Activity 執行流程

```
Workflow Code          Command              Event
─────────────────     ─────────────        ──────────────────────
Activity Method   →   ScheduleActivity  →  ActivityTaskScheduled
                                        →  ActivityTaskStarted
                                        →  ActivityTaskCompleted
```

### Timer 執行流程

```
Workflow Code          Command              Event
─────────────────     ─────────────        ──────────────────────
Workflow.sleep()  →   StartTimer        →  TimerStarted
                                        →  TimerFired
```

## Task 狀態模式

### Activity Task 狀態

1. **Scheduled**：Task 加入 Queue (Cluster 執行)
2. **Started**：Worker 開始處理 (Worker 執行)
3. **Closed 狀態**：
    - **Completed**：成功完成
    - **Failed**：執行失敗
    - **Timed Out**：超時 (Cluster 判定)

### Workflow Task 狀態

遵循相同的 Scheduled → Started → Closed 模式。

## Sticky Execution

### 概念

為提升效能，Temporal 會將同一個 Workflow 的後續 Task 導向同一個 Worker。

### 運作機制

1. **初始 Task**：放入指定的 Task Queue，任何 Worker 都能處理
2. **Sticky Queue**：Worker 開始輪詢專屬的 Sticky Queue
3. **後續 Task**：優先放入 Sticky Queue 給同一個 Worker
4. **容錯機制**：5 秒內 Worker 無回應則回到原始 Queue

!!! info "重要提醒"
    Sticky Execution 只適用於 Workflow Task，不適用於 Activity Task。

## Event History 限制

### Event 數量限制

- **警告門檻**：10K (10,240) Events
- **終止門檻**：50K (51,200) Events
- **建議**：單一 Workflow 不超過數千個 Events

!!! note "解決方案"
    使用 Continue-As-New 機制延續執行，建立新的 Workflow 與新的 Event History。

## 資料安全

### 機密資料保護

- **問題**：Event History 包含輸入/輸出資料，可能含機密資訊
- **解決方案**：使用 Custom Codec 進行端到端加密
- **傳輸保護**：TLS 加密保護資料傳輸

!!! tip "最佳實務"
    建立自訂 Codec 對敏感資料進行加密，確保只有應用程式本身能讀取。

## 效能考量

### Event History 查詢

- Event History 持久化儲存在資料庫中
- Worker 會快取 Workflow 執行狀態以提升效能
- 大量 Events 會影響 Workflow 重建效能

### 最佳實務

1. **控制 Event 數量**：避免過度使用 Activity 或 Timer
2. **使用 Continue-As-New**：定期重置 Event History
3. **資料加密**：保護敏感資訊
4. **監控限制**：追蹤 Event 數量避免達到限制
