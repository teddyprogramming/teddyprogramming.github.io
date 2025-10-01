# 提升 Temporal 應用程式碼品質

## Backward Compatibility 設計

**使用類別封裝參數**：
```java
// ✅ 建議：便於擴充，不破壞 backward compatibility
class GreetingInput {
    String name;
    String languageCode;
}

GreetingOutput greetSomeone(GreetingInput input);

// ❌ 避免：直接使用基本型別
// String greetInSpanish(String name)
```

## Task Queue 最佳實務

**使用常數管理名稱**：

```java
public class Constants {
    public static final String TASK_QUEUE_NAME = "my-task-queue";
}
```

**生產環境部署**：

- 每個 Task Queue 至少兩個 Worker 程序
- 部署在不同主機避免單點故障

## Workflow ID 設計

**命名原則**：

- 使用具業務意義的 ID：`process-order-90743812`
- 確保唯一性，考慮加入時間戳

**重複使用策略**：

- `WORKFLOW_ID_REUSE_POLICY_ALLOW_DUPLICATE` (預設)：允許在前一個 Workflow 完成後重複使用，不論成功或失敗
- `WORKFLOW_ID_REUSE_POLICY_ALLOW_DUPLICATE_FAILED_ONLY`：只在失敗時重複使用
- `WORKFLOW_ID_REUSE_POLICY_REJECT_DUPLICATE`：不允許重複
- `WORKFLOW_ID_REUSE_POLICY_TERMINATE_IF_RUNNING`：終止執行中的相同 ID Workflow

```java
WorkflowOptions options = WorkflowOptions.newBuilder()
    .setWorkflowIdReusePolicy(
        WorkflowIdReusePolicy.WORKFLOW_ID_REUSE_POLICY_ALLOW_DUPLICATE_FAILED_ONLY)
    .build();
```

## 資料保留與日誌

**保留期間**：

- Workflow 完成後到資料刪除的期間（1-30 天）
- 長期保存：使用 Archival（Self-hosted）或 Export（Cloud）

**日誌處理**：

```java
// Workflow 使用專用 Logger，支援重播安全
public static final Logger logger = Workflow.getLogger(MyWorkflowImpl.class);

// Activity 使用標準 Logger
private static final org.slf4j.Logger logger =
    org.slf4j.LoggerFactory.getLogger(MyActivityImpl.class);
```

## 非同步執行與平行處理

**同步 vs 非同步**：

```java
// 同步：阻塞等待結果
String greeting = workflow.greetSomeone(name);

// 非同步：立即返回 Future
CompletableFuture<String> greeting = WorkflowClient.execute(workflow::greetSomeone, "World");
String result = greeting.get(); // 需要時才等待
```

**平行執行 Activity**：

```java
// 循序執行（慢）
String hello = activities.greetInSpanish(name);
String goodbye = activities.farewellInSpanish(name);

// 平行執行（快 3 倍）
Promise<String> hello = Async.function(activities::greetInSpanish, name);
Promise<String> goodbye = Async.function(activities::farewellInSpanish, name);
String helloResult = hello.get();
String goodbyeResult = goodbye.get();
```

## Workflow 確定性要求

Workflow 必須保持確定性，避免：

- **時間函數**：使用 `Workflow.currentTimeMillis()` 而非 `System.currentTimeMillis()`
- **隨機數**：使用 `Workflow.newRandom()` 或 `Workflow.randomUUID()`
- **外部呼叫**：所有外部互動需透過 Activity
- **多執行緒**：避免在 Workflow 中建立新執行緒
- **非確定性函式庫**：避免可能產生不同結果的第三方函式庫
