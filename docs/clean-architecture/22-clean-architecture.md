# Clean Architecture

## Context

在現代軟體開發中，應用程式通常需要整合多個外部系統: 資料庫、Web框架、第三方服務、UI框架等。這些外部依賴經常變化，而核心業務邏輯相對穩定。在傳統分層架構中，業務邏輯層往往直接依賴具體的技術實作 (如特定的資料庫存取類別、外部服務客戶端)，而不是依賴抽象介面。這種直接依賴會導致業務邏輯與技術細節緊密耦合，使系統難以測試、難以維護，且容易受到外部變化的影響。

## Problem

**如何設計一個既能整合必要外部系統，又能保護核心業務邏輯不受外部變化影響的架構？**

### 具體挑戰

1. **外部依賴的不穩定性**

    - 資料庫技術可能從關聯式改為 NoSQL
    - Web 框架版本升級可能破壞 API
    - 第三方服務可能更改介面或停止服務

2. **測試困難**

    - 單元測試需要真實資料庫連線
    - 整合測試需要外部服務正常運作
    - 測試環境設定複雜且脆弱

3. **業務邏輯散落**

    - 業務規則分散在資料存取層中

        ```java
        // DAO 層包含業務邏輯
        public class OrderDAO {
            public void save(Order order) {
                // 業務規則混在 DAO 中
                if (order.getTotal() > 10000) {
                    order.setStatus("REQUIRES_APPROVAL");
                }
                jdbcTemplate.update("INSERT INTO orders...", order);
            }
        }
        ```

    - 用戶介面包含業務決策邏輯

        ```javascript
        // Controller 包含業務邏輯
        function createOrder() {
            const order = getOrderFromForm();
            // 業務規則混在 UI 層
            if (order.customerType === 'VIP' && order.total > 5000) {
                order.discount = 0.1;
            }
            saveOrder(order);
        }
        ```

    - 很難找到「業務邏輯在哪裡」

        - 折扣計算在前端
        - 訂單驗證在 DAO
        - 狀態判斷在 Service
        - 相同邏輯重複出現在多個地方

## Forces

### 穩定性需求

- 業務邏輯是系統的核心價值，應該最穩定
- 外部技術細節變化頻繁，不應影響核心邏輯

### 可測試性需求

- 業務邏輯必須能夠獨立測試
- 不應依賴外部系統來驗證核心邏輯

### 可維護性需求

- 修改外部整合不應影響業務邏輯
    - 例如: 從 MySQL 改為 PostgreSQL，不需要修改訂單計算邏輯
- 業務邏輯變更不應影響技術實作細節
    - 例如: 修改折扣計算規則，不需要修改資料庫 schema 或 API 格式
    - **前提**: 初始設計時的抽象分層要適當
        - 資料庫 schema 設計要足夠通用 (如: discount_amount 而非 vip_discount)
        - API 介面要足夠彈性 (如: 使用通用的計算結果欄位)
        - 領域模型要穩定且表達力強

### 可替換性需求

- 應該能夠替換資料庫而不修改業務邏輯
- 應該能夠替換 UI 框架而不修改業務邏輯

## Solution

Clean Architecture 透過 **依賴反轉** 和 **明確分層** 來解決這些問題:

### Dependency Rule (依賴規則)

**<span style="color: #0099FF;">Source code</span>[^1] dependencies must point only inward, toward higher-level policies. (<span style="color: #0099FF;">程式碼</span>依賴必須只指向內層，朝向更高層級的政策。)**

[^1]: 這裡特別強調「程式碼」依賴，是因為在最外層基本上是非程式碼 (例如資料庫、檔案系統等)，這些外部系統不受此依賴規則約束。

!!!note "什麼是「政策」(Policy)？"

    在軟體架構中，「政策」指的是系統的核心業務邏輯和規則，例如:

    - 業務規則: 「訂單金額超過 10000 元需要主管核准」
    - 計算邏輯: 「VIP 客戶享有 10% 折扣」
    - 工作流程: 「訂單建立 → 庫存檢查 → 付款處理 → 出貨」

    這些政策代表系統的核心價值，相對穩定且不應該依賴於技術實作細節 (如資料庫類型、UI 框架等)。

![](22-1.png)

### 四個同心圓分層

Clean Architecture 將系統分為四個同心圓分層，每層都有明確的職責:

!!!note "關於圓圈數量的說明"

    Robert Martin 在書中提到：

    - **圖中的四個圓圈只是示意圖**：並非固定限制
    - **可以有更多圓圈**：根據應用程式的複雜度，可能需要更多層次
    - **依賴規則仍然適用**：無論有多少層，依賴關係都必須指向內層

    重點是**依賴方向**和**關注點分離**的原則。

```kroki-plantuml
hide circle

package "Use Cases" {
    interface CreateOrderInputPort {
        execute(requestModel: CreateOrderRequestModel)
    }

    interface CreateOrderOutputPort {
        presentSuccess(responseModel: CreateOrderResponseModel)
    }

    interface OrderRepository {
        save(order: Order): Order
        findById(id: OrderId): Optional<Order>
    }

    class CreateOrderUseCase implements CreateOrderInputPort {
    }

    CreateOrderUseCase -u-> OrderRepository
}

package "Interface Adapters" {
    class OrderController {
        createOrder(httpRequest: CreateOrderHttpRequest)
    }

    class OrderPresenter {
        presentSuccess(responseModel: CreateOrderResponseModel)
        getViewModel(): CreateOrderViewModel
    }

    class JdbcOrderRepository {
        jdbcTemplate: JdbcTemplate
    }
}

package "Frameworks & Drivers" {
    class JdbcTemplate {
    }

    class Database {
    }
}

OrderController --> CreateOrderInputPort
OrderPresenter ..|> CreateOrderOutputPort
CreateOrderUseCase -u-> CreateOrderOutputPort
JdbcOrderRepository ..|> OrderRepository
JdbcOrderRepository -u-> JdbcTemplate
JdbcTemplate -u-> Database
```

#### 1. Entities (實體層) - 企業業務規則

- **職責**: 封裝企業級業務邏輯和規則
- **特徵**:
    - 最不可能因外部變化而改變
    - 可被多個應用程式重用
    - 不依賴任何外層

```java
// 範例: 訂單實體
public class Order {
    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderItem> items;
    private OrderStatus status;

    // 業務規則: 訂單總額計算
    public Money calculateTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }

    // 業務規則: 訂單確認條件
    public void confirm() {
        if (items.isEmpty()) {
            throw new IllegalStateException("Cannot confirm empty order");
        }
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("Order already processed");
        }
        this.status = OrderStatus.CONFIRMED;
    }
}
```

#### 2. Use Cases (使用案例層) - 應用業務規則

- **職責**: 協調實體的互動以完成特定應用功能
- **特徵**:
    - 包含應用特定的業務規則
    - 定義資料進出的介面
    - 不關心資料如何呈現或持久化

```java
// Use Cases 層定義的 Repository 介面
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(OrderId id);
}

// 範例: 建立訂單使用案例
public class CreateOrderUseCase implements CreateOrderInputPort {
    private final OrderRepository orderRepository;
    private final CreateOrderOutputPort outputPort;

    public CreateOrderUseCase(OrderRepository orderRepository,
                              CreateOrderOutputPort outputPort) {
        this.orderRepository = orderRepository;
        this.outputPort = outputPort;
    }

    @Override
    public void execute(CreateOrderRequestModel requestModel) {
        // 建立訂單實體
        Order order = new Order(
            OrderId.generate(),
            requestModel.getCustomerId(),
            buildOrderItems(requestModel.getItems())
        );

        // 應用業務規則
        order.confirm();

        // 持久化
        Order savedOrder = orderRepository.save(order);

        // 透過 OutputPort 回傳結果
        CreateOrderResponseModel response = new CreateOrderResponseModel(
            savedOrder.getId().getValue(),
            savedOrder.calculateTotal().getValue()
        );
        outputPort.presentSuccess(response);
    }

    private List<OrderItem> buildOrderItems(List<OrderItemData> items) {
        // 建構 OrderItem 的邏輯
        return items.stream()
            .map(item -> new OrderItem(item.getProductId(), item.getQuantity()))
            .collect(Collectors.toList());
    }
}
```

#### 3. Interface Adapters (介面 Adapter 層)

- **職責**: 在內外層之間轉換資料格式，讓不同的介面能夠溝通
- **包含**:
    - Controllers
    - Gateways
    - Presenters

```java
// 範例: 訂單控制器
@RestController
public class OrderController {
    private final CreateOrderInputPort createOrderInputPort;

    public OrderController(CreateOrderInputPort createOrderInputPort) {
        this.createOrderInputPort = createOrderInputPort;
    }

    @PostMapping("/orders")
    public ResponseEntity<?> createOrder(@RequestBody CreateOrderHttpRequest httpRequest) {
        // 將 HTTP 請求模型轉換為 Use Case 請求模型
        CreateOrderRequestModel requestModel = new CreateOrderRequestModel(
            httpRequest.getCustomerId(),
            httpRequest.getItems()
        );

        // 透過 InputPort 執行使用案例
        createOrderInputPort.execute(requestModel);

        return ResponseEntity.ok().build();
    }
}

// 範例: Repository 實作 (屬於 Interface Adapters 層)
@Repository
public class JdbcOrderRepository implements OrderRepository {
    private final JdbcTemplate jdbcTemplate;

    public JdbcOrderRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public Order save(Order order) {
        // 將領域物件轉換為資料庫格式
        String sql = "INSERT INTO orders (id, customer_id, total, status) VALUES (?, ?, ?, ?)";
        jdbcTemplate.update(sql,
            order.getId().getValue(),
            order.getCustomerId().getValue(),
            order.calculateTotal().getValue(),
            order.getStatus().name()
        );
        return order;
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        String sql = "SELECT id, customer_id, total, status FROM orders WHERE id = ?";
        try {
            Order order = jdbcTemplate.queryForObject(sql,
                (rs, rowNum) -> OrderMapper.fromResultSet(rs),
                id.getValue()
            );
            return Optional.of(order);
        } catch (EmptyResultDataAccessException e) {
            return Optional.empty();
        }
    }
}
```

#### 4. Frameworks & Drivers (框架和驅動程式層)

- **職責**: 提供實際的技術基礎設施
- **包含**:
    - **Frameworks (框架)**: Spring Boot, Express.js, Django 等
    - **Drivers (驅動程式)**: 資料庫驅動、檔案系統、網路協議等
    - **External Systems (外部系統)**: 第三方 API、訊息佇列、快取系統等
    - **Infrastructure (基礎設施)**: Web Server、Database、File System 等

```java
// 範例: 資料庫配置 (Frameworks & Drivers 層)
@Configuration
public class DatabaseConfig {

    @Bean
    public DataSource dataSource() {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl("jdbc:postgresql://localhost:5432/orders");
        dataSource.setUsername("user");
        dataSource.setPassword("password");
        return dataSource;
    }

    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

**重要概念澄清**：
- **Repository 介面**：定義在 Use Cases 層 (內層定義抽象)
- **Repository 實作**：位於 Interface Adapters 層 (外層實作抽象)
- **資料庫驅動/框架**：位於 Frameworks & Drivers 層 (最外層基礎設施)

### 依賴注入和介面

透過依賴注入實現依賴反轉:

```java
// 在組裝層進行依賴注入
@Configuration
public class ApplicationConfig {
    @Bean
    public CreateOrderUseCase createOrderUseCase(OrderRepository orderRepository) {
        return new CreateOrderUseCase(orderRepository);
    }

    @Bean
    public OrderRepository orderRepository(JdbcTemplate jdbcTemplate) {
        return new JdbcOrderRepository(jdbcTemplate);
    }
}
```

### Controller 和 Presenter 的依賴反轉

Controller 和 Presenter 透過 DIP 與 Use Cases 互動：


**依賴方向說明**：

- **Controller 層 (Interface Adapters)**：

    - Controller 依賴 InputPort interface (向內依賴)
    - Controller 使用 HttpRequest model (同層依賴)
    - Controller 建立 RequestModel 傳遞給 Use Case

- **Use Case 層**：

    - UseCase 實作 InputPort interface
    - UseCase 依賴 OutputPort interface (同層依賴)
    - UseCase 依賴 Repository interface (同層依賴)
    - 使用 Request 和 Response Models 進行資料傳遞

- **Interface Adapters 層**：

    - JdbcOrderRepository 實作 OrderRepository interface (外層實作內層介面)
    - Repository 實作負責將領域物件轉換為資料庫格式

- **Frameworks & Drivers 層**：

    - 提供 JdbcTemplate、資料庫驅動等基礎設施
    - Repository 實作會使用這些基礎設施來完成實際的資料存取

!!!note "Clean Architecture 中的資料傳遞設計原則"

    基於 Clean Architecture 的核心原則，在層與層之間傳遞資料時應該遵循以下設計考量：

    **核心原則**：

    - **依賴規則**：內層不應該知道外層的存在
    - **框架獨立性**：業務邏輯不應依賴特定的技術框架
    - **介面隔離**：使用介面定義邊界，而非具體實作

    **實作建議**：

    - Use Cases 透過 Input Port 接收資料
    - Use Cases 透過 Output Port 傳送資料
    - 資料結構應該是 **不依賴外部框架的純粹資料結構**
    - 通常實作為 POJO (Plain Old Java Objects) 或 DTO (Data Transfer Objects)

    **設計考量**：

    - **框架獨立性**：避免在核心業務邏輯中使用框架特定的註解或依賴
    - **職責分離**：每層有自己的資料模型，透過轉換來隔離變化
    - **測試性**：純粹的資料結構易於建立和測試

    **對比範例**：

    ```java
    // ❌ 包含框架依賴
    public class CreateOrderHttpRequest {
        @JsonProperty("customer_id")
        @NotBlank
        private String customerId;

        @Valid
        private List<OrderItemRequest> items;
    }

    // ✅ 框架獨立的資料結構
    public class CreateOrderRequestModel {
        private final String customerId;
        private final List<OrderItemData> items;

        // 建構子和 getter...
    }
    ```

    這種設計確保 Use Case 層符合 Clean Architecture 的依賴規則。

- **Presenter 層**：

    - Presenter 實作 OutputPort interface
    - Presenter 接收 ResponseModel 並轉換為 ViewModel
    - ViewModel 包含 UI 特定的展示邏輯

**關鍵設計原則**：

- 每一層都有自己的資料模型，避免跨層污染
- 依賴方向始終指向內層(業務核心)
- 外層負責資料轉換和格式 adaptation

#### Controller 透過 DIP 呼叫 Use Cases

```java
public interface CreateOrderInputPort {
    void execute(CreateOrderRequestModel requestModel);
}

@RestController
public class OrderController {
    private final CreateOrderInputPort createOrderUseCase;

    public OrderController(CreateOrderInputPort createOrderUseCase) {
        this.createOrderUseCase = createOrderUseCase;
    }

    @PostMapping("/orders")
    public ResponseEntity<?> createOrder(@RequestBody CreateOrderHttpRequest httpRequest) {
        // 將 HTTP 請求模型轉換為 Use Case 請求模型
        CreateOrderRequestModel requestModel = new CreateOrderRequestModel(
            httpRequest.getCustomerId(),
            convertToOrderItemData(httpRequest.getItems())
        );

        createOrderUseCase.execute(requestModel);
        return ResponseEntity.ok().build();
    }

    private List<OrderItemData> convertToOrderItemData(List<OrderItemRequest> items) {
        // 轉換邏輯：處理 HTTP 特定的資料格式
        return items.stream()
            .map(item -> new OrderItemData(item.getProductId(), item.getQuantity()))
            .collect(Collectors.toList());
    }
}
```

**為什麼 Controller 也需要分別定義 HttpRequest 和 RequestModel？**

1. **HTTP 層的職責**：

    - `CreateOrderHttpRequest` 處理 HTTP 特定的需求 (如 JSON 序列化、驗證註解)
    - `CreateOrderRequestModel` 專注於業務邏輯所需的純淨資料

2. **框架獨立性與資料轉換控制**：

    - Use Case 不依賴 Spring Boot 或 JSON 等框架的註解和依賴
    - 業務邏輯可以在不同的應用程式類型中重用 (Web API、CLI 工具、批次處理等)
    - Controller 可以在轉換過程中處理 HTTP 特定的邏輯 (如資料驗證、格式轉換、預設值設定)

**Controller 使用 DIP 與 Use Case 互動的好處**：

1. **鬆耦合設計**

    - Controller 不知道具體的 Use Case 實作細節
    - 可以替換不同的 Use Case 實作而不修改 Controller

2. **可測試性**

    Controller 透過 DIP 的主要測試價值在於能夠隔離測試各層的職責：

    ```java
    @Test
    void shouldReturn200WhenOrderCreatedSuccessfully() {
        // 模擬成功的 Use Case
        CreateOrderInputPort mockUseCase = Mockito.mock(CreateOrderInputPort.class);
        OrderController controller = new OrderController(mockUseCase);

        CreateOrderHttpRequest request = new CreateOrderHttpRequest("CUST001", List.of());

        // 測試 HTTP 層的行為
        ResponseEntity<?> response = controller.createOrder(request);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    }

    @Test
    void shouldReturn500WhenUseCaseThrowsException() {
        // 模擬 Use Case 拋出例外
        CreateOrderInputPort mockUseCase = Mockito.mock(CreateOrderInputPort.class);
        doThrow(new RuntimeException("Database error"))
            .when(mockUseCase).execute(any());

        OrderController controller = new OrderController(mockUseCase);
        CreateOrderHttpRequest request = new CreateOrderHttpRequest("CUST001", List.of());

        // 驗證 Controller 如何處理例外
        assertThrows(RuntimeException.class, () ->
            controller.createOrder(request));
    }
    ```

    重點是測試 **Controller 的職責**（HTTP 處理、錯誤處理），而非驗證是否有呼叫其他組件。

3. **框架獨立性**

    - Controller 的核心邏輯不依賴特定的業務實作
    - 可以從 Spring MVC 切換到其他 Web 框架，Use Case 不受影響

4. **職責分離**

    - Controller 專注於 HTTP 請求處理和資料轉換
    - Use Case 專注於業務邏輯執行
    - 各自可以獨立演化和測試


#### Presenter 透過 DIP 接收 Use Cases 結果

```java
public interface CreateOrderOutputPort {
    void presentSuccess(CreateOrderResponseModel responseModel);
}

@Component
public class OrderPresenter implements CreateOrderOutputPort {
    private CreateOrderViewModel viewModel;

    @Override
    public void presentSuccess(CreateOrderResponseModel responseModel) {
        this.viewModel = new CreateOrderViewModel(
            "訂單建立成功",  // UI 訊息
            responseModel.getOrderId(),
            formatCurrency(responseModel.getTotal()) // 格式化顯示
        );
    }

    private String formatCurrency(BigDecimal amount) {
        return NumberFormat.getCurrencyInstance(Locale.TAIWAN).format(amount);
    }

    public CreateOrderViewModel getViewModel() {
        return viewModel;
    }
}

public class CreateOrderUseCase implements CreateOrderInputPort {
    private final CreateOrderOutputPort outputPort;

    public CreateOrderUseCase(CreateOrderOutputPort outputPort) {
        this.outputPort = outputPort;
    }

    @Override
    public void execute(CreateOrderRequestModel requestModel) {
        Order order = new Order(
            OrderId.generate(),
            requestModel.getCustomerId(),
            requestModel.getItems()
        );

        order.confirm();

        Order savedOrder = orderRepository.save(order);

        CreateOrderResponseModel response = new CreateOrderResponseModel(
            savedOrder.getId().getValue(),
            savedOrder.calculateTotal().getValue()
        );
        outputPort.presentSuccess(response);
    }
}
```

#### 關鍵特點

- **Controller 不直接依賴 Use Case 實作**：透過 InputPort 介面
- **Use Case 不直接依賴 Presenter 實作**：透過 OutputPort 介面
- **依賴方向正確**：外層依賴內層的抽象介面
- **各層職責清晰**：Controller 處理 HTTP，Presenter 處理格式化，Use Case 處理業務邏輯

## Results

### 正面效果

1. **獨立性 (Independence)**

    - 業務邏輯完全獨立於外部技術
    - 可以單獨測試核心邏輯
    - 外部系統變更不影響業務邏輯

2. **可測試性 (Testability)**

   ```java
   @Test
   void shouldCreateOrderWhenCustomerExistsAndItemsValid() {
       // 使用模擬 Repository，無需真實資料庫
       OrderRepository mockRepository = Mockito.mock(OrderRepository.class);
       CreateOrderUseCase useCase = new CreateOrderUseCase(mockRepository);

       // 測試業務邏輯
       OrderId result = useCase.execute(validRequest);

       assertThat(result).isNotNull();
   }
   ```

3. **可替換性 (Replaceability)**

    - 可以替換資料庫 (MySQL → MongoDB) 而不影響業務邏輯
    - 可以替換 Web 框架 (Spring → Vert.x) 而不修改 Use Cases
    - 可以替換 UI (Web → Mobile) 而不改變核心功能

4. **規模化發展 (Scalability)**

    - 團隊可以平行開發不同架構層
    - 新功能不會污染現有核心邏輯
    - 複雜度被明確分離和管理

### 潛在缺點

1. **複雜度增加**

    - 需要更多的介面和 Adapter
    - 程式碼行數可能增加
    - 學習曲線較陡峭

2. **過度設計風險**

    - 簡單專案可能不需要如此複雜的結構
    - 需要根據專案規模衡量是否適用

3. **團隊學習成本**

    - 需要團隊對架構有共同理解
    - 需要良好的文件和培訓
