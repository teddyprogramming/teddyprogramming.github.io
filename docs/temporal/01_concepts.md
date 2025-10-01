# Temporal 核心概念

## 核心元件

**Workflow**：業務邏輯的執行流程

- 必須是**確定性的 (deterministic)**
- 定義整個業務流程的步驟

**Activity**：處理不可靠或非確定性的操作

- 封裝 I/O、外部 API 呼叫等
- 自動重試機制

**Worker**：執行 Workflow 和 Activity 的程式

- 輪詢 Temporal Cluster 的 Task Queue


## 錯誤處理機制

### Activity 重試

**自動重試**：

- 預設指數 backoff：1秒、2秒、4秒...（最多 100 秒）
- 持續重試直到成功、取消或逾時

**自訂重試策略**：
```java
RetryOptions.newBuilder()
    .setMaximumAttempts(10)
    .setDoNotRetry("com.example.DatabaseCorruptionException")
    .build();
```

**重要原則**：

- Activity 必須設計為 **idempotent**
- 相同輸入多次執行應產生相同結果

### Workflow 錯誤處理

| 類型 | Activity | Workflow |
|------|----------|----------|
| 重試機制 | 預設指數 backoff 重試 | 無預設重試 |
| 一般 Exception | 自動重試 | 重試整個 Task |
| TemporalFailure | - | 直接標記失敗 |

**明確終止 Workflow**：
```java
if (distance.getKilometers() > 25) {
  throw ApplicationFailure.newFailure(
    "Customer lives outside the service area",
    OutOfServiceAreaException.class.getName()
  );
}
```

### 跨語言支援

**例外轉換**：自動將語言特定例外轉換為通用的 `ApplicationFailure`

**簡化處理**：

```java
try {
  // 可能拋出 IOException 的程式碼
} catch (IOException e) {
  throw Activity.wrap(e);  // 轉為 ActivityFailure
}
```
