# 提升 Temporal 應用程式碼品質

## Backward Compatibility (向後相容性設計)

### 使用類別封裝輸入輸出參數

```java
// ✅ 建議：使用類別封裝，便於未來擴充
class GreetingInput {
    String name;
    String languageCode;
}

class GreetingOutput {
    String greeting;
}

GreetingOutput greetSomeone(GreetingInput input);

// ❌ 不建議：直接使用基本型別，修改會破壞相容性
// String greetInSpanish(String name)
```

**優點**：可在不改變方法簽名的情況下新增欄位，完全不影響既有 Workflow Execution。

## Task Queue 最佳實務

### 使用常數定義 Task Queue 名稱

```java
// 定義常數
public class Constants {
    public static final String TASK_QUEUE_NAME = "my-task-queue-name";
}

// Client 端
WorkflowOptions options = WorkflowOptions.newBuilder()
    .setWorkflowId("my-workflow")
    .setTaskQueue(Constants.TASK_QUEUE_NAME)
    .build();

// Worker 端
Worker worker = factory.newWorker(Constants.TASK_QUEUE_NAME);
```

**避免問題**：名稱不一致會導致 Worker 永遠收不到任務，Workflow 卡住。

### 生產環境部署策略

- 每個 Task Queue 至少運行**兩個 Worker 程序**
- 部署在**不同主機**上避免單點故障
- 自動故障轉移：當一個 Worker 當機時，其他 Worker 自動接管

## Workflow ID 設計

### 命名原則

- 使用具有**業務意義**的 ID：
    - 訂單處理：`process-order-90743812`
    - 貸款管理：`manage-loan-28430614`
- 確保在應用場景中具有**唯一性**
- 考慮加入時間戳或唯一標識符避免衝突

### 四種重複使用策略

**WORKFLOW_ID_REUSE_POLICY_ALLOW_DUPLICATE (預設)**
- 允許在前一個 Workflow 完成後重複使用 ID，不論成功或失敗

**WORKFLOW_ID_REUSE_POLICY_ALLOW_DUPLICATE_FAILED_ONLY**
- 僅在前一個 Workflow Execution 失敗時才允許重複使用

**WORKFLOW_ID_REUSE_POLICY_REJECT_DUPLICATE**
- 完全不允許重複使用 Workflow ID

**WORKFLOW_ID_REUSE_POLICY_TERMINATE_IF_RUNNING**
- 若有執行中相同 ID Workflow，終止該 Workflow 並啟動新的

```java
// 設定重複使用策略範例
WorkflowOptions options = WorkflowOptions.newBuilder()
    .setWorkflowId("example-workflow-id")
    .setTaskQueue(Constants.TASK_QUEUE_NAME)
    .setWorkflowIdReusePolicy(
        WorkflowIdReusePolicy.WORKFLOW_ID_REUSE_POLICY_ALLOW_DUPLICATE_FAILED_ONLY)
    .build();
```

## 資料保留與合規性

### 保留期間 (Retention Period)

- 定義：Workflow Execution 完成後到資料被自動刪除的期間
- 典型範圍：1-30 天
- 僅對**已完成**的 Workflow 計算，不影響執行中的 Workflow

### 長期資料保存方案

- **Self-hosted Temporal**: 使用 Archival 功能將完成的 Workflow 歷史記錄自動歸檔至外部持久儲存系統 (如檔案系統、Amazon S3)
- **Temporal Cloud**: 使用 Export 功能每小時自動將歷史資料匯出到 S3
- 適用於需要數月或數年後查詢 Workflow Execution 記錄的合規要求

## 日誌處理

### 使用 Temporal 專用日誌 API

Workflow 日誌設定：

```java
// Workflow 中使用專用 Logger
public static final Logger logger = Workflow.getLogger(MyWorkflowImpl.class);

logger.info("處理訂單 {}, 金額: {}", orderId, amount);
```

Activity 日誌設定：

```java
// Activity 可使用標準 Logger
private static final org.slf4j.Logger logger =
    org.slf4j.LoggerFactory.getLogger(MyActivityImpl.class);
```

**優點**：

- **重播安全機制**：在 History Replay 過程中自動抑制重複日誌輸出，確保相同日誌不會在原始執行和重播階段重複出現

## 存取執行結果

### 非同步執行概念

Temporal 支援長期運行的 Workflow (數月或數年)，透過非同步設計實現：

- **非阻塞呼叫**：方法呼叫立即返回，不等待實際執行完成
- **持久性執行**：即使啟動 Workflow 的應用程式已關閉，Workflow 仍會由 Temporal 系統中的 Worker 持續執行到完成

### 同步呼叫 (阻塞模式)

**Workflow Execution**

```java
GreetingWorkflow workflow = client.newWorkflowStub(GreetingWorkflow.class, options);
String greeting = workflow.greetSomeone(name); // 阻塞等待結果
```

**Activity Execution**

```java
String spanishGreeting = activities.greetInSpanish(name); // 阻塞等待結果
```

### 非同步呼叫 (非阻塞模式)

**非同步 Workflow 呼叫**

```java
import java.util.concurrent.CompletableFuture;

GreetingWorkflow workflow = client.newWorkflowStub(GreetingWorkflow.class, options);

// 立即啟動但不阻塞
CompletableFuture<String> greeting = WorkflowClient.execute(workflow::greetSomeone, "World");

// 需要結果時才阻塞等待
String result = greeting.get();
```

**非同步 Activity 呼叫**

```java
import io.temporal.workflow.Async;
import io.temporal.workflow.Promise;

// 立即啟動兩個 Activity，不等待完成
Promise<String> hello = Async.function(activities::greetInSpanish, name);
Promise<String> bye = Async.function(activities::farewellInSpanish, name);

// 需要結果時才阻塞等待
String helloResult = hello.get();
String byeResult = bye.get();
```

### 平行執行優化

**循序執行 vs 平行執行**

**循序執行 (較慢)**

```java
String hello = activities.greetInSpanish(name);      // 等待完成
String goodbye = activities.farewellInSpanish(name); // 等待完成
String thanks = activities.thankInSpanish(name);     // 等待完成
```

**平行執行 (更快)**

```java
// 同時啟動三個 Activity
Promise<String> hello = Async.function(activities::greetInSpanish, name);
Promise<String> goodbye = Async.function(activities::farewellInSpanish, name);
Promise<String> thanks = Async.function(activities::thankInSpanish, name);

// 分別等待各自完成
String helloResult = hello.get();
String goodbyeResult = goodbye.get();
String thanksResult = thanks.get();
```

### 效能提升

- 🚀 **速度提升**：不相關的 Activity 平行執行可提升約 3 倍速度
- 📊 **適用場景**：多個獨立 Activity 且有足夠系統容量時
- ⚡ **關鍵概念**：分離執行請求與結果取得

### 最佳實務

- 對於獨立的 Activity 使用非同步執行
- 充分利用系統平行處理能力
- 在需要結果時才呼叫 `get()` 方法

## Workflow 確定性保障

Workflow 程式碼必須保持確定性，避免以下行為：

- **時間相關函數**：使用 `Workflow.currentTimeMillis()` 而非 `System.currentTimeMillis()`
- **隨機數生成**：使用 `Workflow.newRandom()` 或 `Workflow.randomUUID()`
- **外部系統呼叫**：所有外部互動應通過 Activity 完成
- **共享可變狀態**：跨 Workflow 的狀態應通過 Activity 或訊號傳遞
- **多執行緒操作**：避免在 Workflow 中建立新的執行緒
- **非確定性函式庫**：避免使用可能產生不同結果的第三方函式庫
