# Workflow 確定性原理與實務

!!! note "筆記重點"
    Temporal Workflow 的確定性是核心要求 - 相同輸入必須產生相同的 Commands 序列。涵蓋確定性原理、常見陷阱、部署注意事項和錯誤恢復方法。

## History Replay 機制

### 持久執行保證

Temporal 確保程式碼可靠執行，即使面臨：

- Worker 當掉
- 伺服器重啟
- 網路中斷

### Repl// Reset 策略：選擇 validateOrder 之前的安全點
// 這樣新執行就不會期望找到被移除的 validateOrder Activity
```
Worker 當掉 → 新 Worker 接管 → 載入 Event History → 重新執行程式碼 → 恢復狀態 → 從中斷點繼續
```

### Replay 特性

| 組件 | 原始執行 | Replay 執行 |
|------|----------|-------------|
| **日誌** | 正常輸出 | 抑制重複輸出 |
| **Activity** | 實際執行 | 使用歷史結果 |
| **Timer** | 設定計時器 | 基於歷史狀態 |
| **變數** | 計算產生 | 重建相同狀態 |

## 確定性要求

!!! important "確定性定義"
    **相同輸入 → 相同 Commands → 相同順序**

    給定相同輸入，Workflow 每次執行都必須產生相同的 Commands 序列。

### 確定性驗證流程

```
原始執行：Input → [Command1, Command2, Command3] → Events
Replay 執行：Input → [Command1, Command2, Command3] → ✅ 狀態恢復成功

非確定性：Input → [Command1, Command3] → ❌ Commands 不匹配
```

### 驗證機制

1. **Event History 分析**：Worker 從 Events 推導期望的 Commands
2. **程式碼執行**：重新執行程式碼產生實際 Commands
3. **Commands 比對**：驗證期望與實際是否一致
4. **錯誤處理**：不一致時拋出非確定性錯誤

## 非確定性來源與解決方案

### 🚫 常見問題清單

| 問題類型 | ⚠️ 需注意 | ✅ 確定性做法 |
|----------|-------------|-------------|
| **隨機數** | `random.nextInt()` | Activity 中產生 |
| **系統時間** | `System.currentTimeMillis()` | `Workflow.currentTimeMillis()` |
| **執行緒** | `Thread.sleep()` | `Workflow.sleep()` |
| **外部狀態** | 直接存取資料庫 | Activity 中存取 |
| **遍歷順序** | `HashMap.keySet()` | `LinkedHashMap.keySet()` 或排序 |
| **Run ID** | 條件判斷中使用 | 避免使用 |

### 詳細範例

#### 1. 隨機數處理
```java
// ❌ 錯誤：每次 Replay 產生不同值
@WorkflowMethod
public String processOrder() {
    int randomId = new Random().nextInt(1000);
    if (randomId > 500) {
        return callExpressDelivery(); // 非確定性分支
    }
    return callStandardDelivery();
}

// ✅ 正確：透過 Activity 取得確定性值
@WorkflowMethod
public String processOrder() {
    int randomId = generateRandomId(); // Activity call
    if (randomId > 500) {
        return callExpressDelivery();
    }
    return callStandardDelivery();
}
```

#### 2. 時間處理
```java
// ❌ 錯誤：Replay 時時間不同
long deadline = System.currentTimeMillis() + Duration.ofHours(1).toMillis();

// ✅ 正確：使用 Workflow 時間
long deadline = Workflow.currentTimeMillis() + Duration.ofHours(1).toMillis();
```

#### 3. 集合遍歷
```java
// ⚠️ 需注意：HashMap 遍歷順序不確定
Map<String, Integer> items = new HashMap<>();
for (String item : items.keySet()) {
    processItem(item); // 在 Workflow 中順序可能影響確定性
}

// ✅ 確定性做法：使用有序集合或排序
Map<String, Integer> items = new LinkedHashMap<>();
// 或者排序後遍歷
List<String> sortedKeys = items.keySet().stream()
    .sorted().collect(Collectors.toList());
```

## 部署與相容性

### 🔍 相容性評估標準

!!! question "關鍵問題"
    新版本程式碼是否會產生與舊版本不同的 Commands 序列？

### 典型問題場景

```java
// 原始版本
@WorkflowMethod
public void processOrder() {
    validateOrder();     // Activity 1
    checkInventory();    // Activity 2
    processPayment();    // Activity 3
}

// 🔴 危險變更：移除 Activity
@WorkflowMethod
public void processOrder() {
    // validateOrder();  // 刪除此行
    checkInventory();
    processPayment();
}

// ✅ 安全變更：修改 Activity 內部
@ActivityMethod
public void validateOrder() {
    // 修改驗證邏輯（不影響 Commands）
    log.info("Enhanced validation");
    // ... 新的驗證邏輯
}
```

### 錯誤症狀

當部署不相容變更時會出現：

- Workflow 執行卡住或失敗
- Worker 日誌出現 NonDeterministicError
- Web UI 中 Workflow 狀態顯示異常
- 無法繼續處理後續任務

## Workflow Reset 機制

### 概念與用途

Workflow Reset 是從錯誤部署中恢復的重要工具，它會：

1. **終止當前執行**：停止有問題的 Workflow
2. **建立新執行**：從指定的 Event History 點重新開始
3. **保留部分歷史**：Reset 點之前的 Events 會保留

### 🔧 Reset 操作步驟

#### Web UI 操作
```
1. 開啟 Workflow 詳情頁面
2. 點選 "Reset" 按鈕
3. 選擇適當的 Event ID
   ├─ 通常選擇錯誤發生前的最後一個安全點
   ├─ 如：WorkflowTaskCompleted Event
   └─ 避免選擇 ActivityTaskScheduled 等中間狀態
4. 輸入 Reset 原因
5. 確認執行
```

#### 命令列操作
```bash
temporal workflow reset \
  --workflow-id "pizza-workflow-order-XD001" \
  --event-id 4 \
  --reason "Deployed incompatible change (deleted Activity)" \
  --namespace default
```

### ⚠️ Reset 注意事項

| 項目 | 說明 | 影響 |
|------|------|------|
| **Event 保留** | 只保留 Reset 點之前的 Events | 之後的進度會遺失 |
| **重新執行** | Reset 點之後的所有邏輯重新執行 | 可能產生重複的副作用 |
| **新執行** | Reset 會建立全新的執行，原執行被終止 | 原執行和新執行分開記錄 |
| **Activity 重複** | 已完成的 Activity 可能重新執行 | 需要考慮冪等性 |

### 實際案例

```java
// 問題場景：移除了 validateOrder Activity
// 原始版本
@WorkflowMethod
public void processOrder() {
    validateOrder();     // Event ID 3: ActivityTaskScheduled
                        // Event ID 4: ActivityTaskCompleted
    processPayment();    // Event ID 5: ActivityTaskScheduled
}

// 新版本（有問題）
@WorkflowMethod
public void processOrder() {
    // validateOrder(); // 被移除！
    processPayment();    // 現在是第一個 Activity
}

// Reset 策略：選擇 validateOrder 之前的安全點
// 這樣新執行就不會期望找到被移除的 validateOrder Activity
```

###  監控重點

| 監控項目 | 指標 | 處理方式 |
|----------|------|----------|
| **非確定性錯誤** | 錯誤日誌 | 立即 Reset |
| **執行中 Workflow** | 數量統計 | 部署前確認 |
| **Reset 操作** | 頻率追蹤 | 分析根本原因 |
| **部署影響** | 成功率 | 回滾準備 |
