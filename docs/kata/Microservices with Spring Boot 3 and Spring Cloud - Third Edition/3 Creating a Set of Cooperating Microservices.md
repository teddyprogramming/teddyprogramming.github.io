# 3. Creating a Set of Cooperating Microservices

```plantuml
component "Product Composite" as pc
component "Product" as p
component "Review" as r
component "Recommendation" as rd

pc --> p
pc --> r
pc --> rd
```

Ports:

- Product Composite: `7000`
- Product: `7001`
- Recommendation: `7002`
- Review: `7003`

# Kata 1: Using Spring Initializr to generate skeleton code

```text
/
├── product-composite-service
├── product-service
├── recommendation-service
└── review-service
```

- 使用 Spring Initializr 建立專案
- 專案相依: `actuator`, `webflux`
- 使用 gradle 編譯專案

    ??? tip

        ```shell
        cd product-service; ./gradlew build; cd -; cd recommendation-service; ./gradlew build; cd -; cd review-service; ./gradlew build; cd -; cd product-composite service; ./gradlew build; cd -;
        ```

# Kata 2: Setting up multi-project builds in Gradle

調整專案結構，使可以用單一指令 `./gradlew build` 編譯所有專案。

# Kata 3: Adding RESTful APIs

## Kata 3.1: Product API

將 product-service 的 port 設定成 `7001`。

```plantuml
hide circle

package api {
    interface ProductService {
        getProduct(productId: Int): Product
    }
    note right of ProductService::getProduct
    GET /product/{productId}
    end note

    class Product {
        productId: Int
        name: String
        weight: Int
        serviceAddress: String
    }

    ProductService ..> Product
}

package product-service {
    class ProductServiceImpl
}

ProductService <|.. ProductServiceImpl
```

### Test 3.1.1: Normal case for product = 1

在 `@SpringBootTest` 增加 `webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT` 參數，並新增以下測試，然後實作讓測試通過。

```kotlin hl_lines="6-15"
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ProductServiceApplicationTests {

    @Autowired lateinit var client: WebTestClient

    @Test
    fun getProductById() {
        client.get()
            .uri("/product/1")
            .accept(MediaType.APPLICATION_JSON)
            .exchange()
            .expectStatus().isOk()
            .expectHeader().contentType(MediaType.APPLICATION_JSON)
            .expectBody().jsonPath("$.productId").isEqualTo(1)
    }
}
```

- 使用 mock data，主要提供一個 normal case，在 product id = 1 的情況下，可以取得資料。

    !!! note "書中 Controller 視為 Service，因此命名為 `ProductService` 且放在 `service` 套件下"

- 實作 `ServiceUtil` 取得 host name 與 IP。

    ??? tip "使用 `InetAddress`"
        取得 host name: `InetAddress.getLocalHost().getHostName()`
        取得 host IP: `InetAddress.getLocalHost().getHostAddress()`

### Test 3.1.2: Product not found for product = 13

新增以下測試，並實作讓測試通過。

```kotlin
@Test
fun getProductNotFound() {
    client.get()
        .uri("/product/13")
        .accept(MediaType.APPLICATION_JSON)
        .exchange()
        .expectStatus().isNotFound
        .expectHeader().contentType(MediaType.APPLICATION_JSON)
        .expectBody()
        .jsonPath("$.path").isEqualTo("/product/13")
        .jsonPath("$.message").isEqualTo("No product found for productId: 13")
}
```

- `NotFoundException`

```plantuml
hide circle

class HttpErrorInfo {
    timestamp: ZonedDateTime
    path: String
    httpStatus: HttpStatus
    message: String
}
```

??? tip "使用 `@RestControllerAdvice` 處理 product not found 的 response"

    ```kotlin
    @RestControllerAdvice
    class GlobalControllerExceptionHandler {

        @ResponseStatus(HttpStatus.NOT_FOUND)
        @ExceptionHandler(NotFoundException::class)
        @ResponseBody
        fun handleNotFoundException(
            request: ServerHttpRequest,
            e: NotFoundException
        ): HttpErrorInfo {
            return HttpErrorInfo(
                ZonedDateTime.now(),
                request.path.pathWithinApplication().value(),
                HttpStatus.NOT_FOUND,
                requireNotNull(e.message)
            )
        }

        data class HttpErrorInfo(
            val timestamp: ZonedDateTime,
            val path: String,
            val httpStatus: HttpStatus,
            val message: String,
        )
    }
    ```

### Test 3.1.3: Negative product id

新增以下測試，並實作讓測試通過。

```kotlin
@Test
fun getProductInvalidParameterNegativeValue() {
    client.get()
        .uri("/product/-1")
        .accept(MediaType.APPLICATION_JSON)
        .exchange()
        .expectStatus().isEqualTo(HttpStatus.UNPROCESSABLE_ENTITY) // 422
        .expectHeader().contentType(MediaType.APPLICATION_JSON)
        .expectBody()
        .jsonPath("$.path").isEqualTo("/product/-1")
        .jsonPath("$.message").isEqualTo("Invalid productId: -1")
}
```

- `InvalidInputException`

### Test 3.1.4: Product Id type mismatch

新增以下測試，並實作讓測試通過。

```kotlin
@Test
fun getProductInvalidParameterString() {
    client.get()
        .uri("/product/not-integer")
        .accept(MediaType.APPLICATION_JSON)
        .exchange()
        .expectStatus().isEqualTo(HttpStatus.BAD_REQUEST)
        .expectHeader().contentType(MediaType.APPLICATION_JSON)
        .expectBody()
        .jsonPath("$.path").isEqualTo("/product/not-integer")
        .jsonPath("$.message").isEqualTo("Type mismatch.")
}
```

!!! tip "server.error.include-message=always"

### 3.1.5: 手動測試

```shell
curl http://localhost:7001/product/1
curl http://localhost:7001/product/13
curl http://localhost:7001/product/-1
curl http://localhost:7001/product/not-integer
```

## Kata 3.2: Review API

```plantuml
hide circle

package api {

    interface ReviewService {
        getReviews(productId: Int): List<Review>
    }
    note right of ReviewService::getReviews
    GET /review?productId={productId}
    end note

    class Review {
        productId: Int
        reviewId: Int
        author: String
        subject: String
        content: String
        serviceAddress: String
    }

    ReviewService ..> Review
}

package review-service {
    class ReviewServiceImpl
}

ReviewService <|.. ReviewServiceImpl
```

### Test 3.2.1: Normal case for product id = 1

```kotlin
@Test
fun getReviewByProductId() {
    client.get()
        .uri("/review?productId=1")
        .accept(MediaType.APPLICATION_JSON)
        .exchange()
        .expectStatus().isOk
        .expectHeader().contentType(MediaType.APPLICATION_JSON)
        .expectBody()
        .jsonPath("$.length()").isEqualTo(3)
        .jsonPath("$[0].productId").isEqualTo(1)
}
```

- 將 `ServiceUtil` 抽到 `util` 模組，讓不同的模組可以共用。

    ??? tip "手動新增 `util` module"

        需要增加下面設定

        ```gradle title="build.gradle.kts"
        plugins {
            id("io.spring.dependency-management") version "1.1.7"
        }

        dependencies {
            implementation(platform("org.springframework.boot:spring-boot-dependencies:3.4.2"))
            implementation("org.springframework:spring-contextt")
        }
        ```

        留意 `ServiceUtil` 是否在 sacn package 下。

### Test 3.2.2: product id not found

```kotlin
@Test
fun `get review not found`() {
    client.get()
        .uri("/review?productId=213")
        .accept(MediaType.APPLICATION_JSON)
        .exchange()
        .expectStatus().isOk()
        .expectHeader().contentType(MediaType.APPLICATION_JSON)
        .expectBody()
        .jsonPath("$.length()").isEqualTo(0)
}
```

### Test 3.2.3: product id is negative

```kotlin
@Test
fun `get review invalid parameter negative value`() {
    client.get()
        .uri("/review?productId=-1")
        .accept(MediaType.APPLICATION_JSON)
        .exchange()
        .expectStatus().isEqualTo(HttpStatus.UNPROCESSABLE_ENTITY)
        .expectHeader().contentType(MediaType.APPLICATION_JSON)
        .expectBody()
        .jsonPath("$.path").isEqualTo("/review")
        .jsonPath("$.message").isEqualTo("Invalid productId: -1")
}
```

- 把 exception 及其 handler 移到 util 下共用

### Test 3.2.4: prodict id is not integer

```kotlin
@Test
fun `get review invalid parameter string`() {
    client.get()
        .uri("/review?productId=no-integer")
        .accept(MediaType.APPLICATION_JSON)
        .exchange()
        .expectStatus().isBadRequest()
        .expectHeader().contentType(MediaType.APPLICATION_JSON)
        .expectBody()
        .jsonPath("$.path").isEqualTo("/review")
        .jsonPath("$.message").isEqualTo("Type mismatch.")
}
```

### Test 3.2.5: missig product id

```kotlin
@Test
fun `get review missing parameter`() {
    client.get()
        .uri("/review")
        .accept(MediaType.APPLICATION_JSON)
        .exchange()
        .expectStatus().isBadRequest()
        .expectHeader().contentType(MediaType.APPLICATION_JSON)
        .expectBody()
        .jsonPath("$.path").isEqualTo("/review")
        .jsonPath("$.message").isEqualTo("Required query parameter 'productId' is not present.")
}
```

- 不需要實作

## Kata 3.3: Recommendation API

```plantuml
hide circle

package api {
    interface RecommendationService {
        getRecommendations(productId: Int): List<Recommendation>
    }
    note right of RecommendationService::getRecommendations
    GET /recommendation?productId={productId}
    end note

    class Recommendation {
        productId: Int
        recommendationId: Int
        author: String
        rate: Int
        content: String
        serviceAddress: String
    }

    RecommendationService ..> Recommendation
}

package recommendation-service {
    class RecommendationServiceImpl
}

RecommendationService <|.. RecommendationServiceImpl
```

### Kata 3.3.1: get recommendation by product id

```kotlin
@Test
fun `get recommendation by product id`() {
    client.get()
        .uri("/recommendation?productId=1")
        .accept(MediaType.APPLICATION_JSON)
        .exchange()
        .expectStatus().isOk()
        .expectHeader().contentType(MediaType.APPLICATION_JSON)
        .expectBody()
        .jsonPath("$.length()").isEqualTo(3)
        .jsonPath("$[0].productId").isEqualTo(1)
}
```

### Kata 3.3.2: get recommentation not found

```kotlin
@Test
fun `get recommendation not found`() {
    client.get()
        .uri("/recommendation?productId=113")
        .accept(MediaType.APPLICATION_JSON)
        .exchange()
        .expectStatus().isOk()
        .expectHeader().contentType(MediaType.APPLICATION_JSON)
        .expectBody()
        .jsonPath("$.length()").isEqualTo(0)
}
```

### Kata 3.3.3: get recommentation invalid parameter negative product id

```kotlin
@Test
fun `get recommendation invalid parameter negative value`() {
    client.get()
        .uri("/recommendation?productId=-1")
        .accept(MediaType.APPLICATION_JSON)
        .exchange()
        .expectStatus().isEqualTo(HttpStatus.UNPROCESSABLE_ENTITY)
        .expectHeader().contentType(MediaType.APPLICATION_JSON)
        .expectBody()
        .jsonPath("$.path").isEqualTo("/recommendation")
        .jsonPath("$.message").isEqualTo("Invalid productId: -1")
}
```

### Kata 3.3.4: get recommentation invalid parameter string

```kotlin
@Test
fun `get recommendation invalid parameter string`() {
    client.get()
        .uri("/recommendation?productId=no-integer")
        .accept(MediaType.APPLICATION_JSON)
        .exchange()
        .expectStatus().isBadRequest()
        .expectHeader().contentType(MediaType.APPLICATION_JSON)
        .expectBody()
        .jsonPath("$.path").isEqualTo("/recommendation")
        .jsonPath("$.message").isEqualTo("Type mismatch.")
}
```

### Kata 3.3.5: get recommendation missing parameter

```kotlin
@Test
fun `get recommendation missing parameter`() {
    client.get()
        .uri("/recommendation")
        .accept(MediaType.APPLICATION_JSON)
        .exchange()
        .expectStatus().isBadRequest()
        .expectHeader().contentType(MediaType.APPLICATION_JSON)
        .expectBody()
        .jsonPath("$.path").isEqualTo("/recommendation")
        .jsonPath("$.message").isEqualTo("Required query parameter 'productId' is not present.")
}
```

## Kata 3.4: Product Composite API

```plantuml
hide circle

package api {
    interface ProductCompositeService {
        getProduct(productId: Int): ProductAggregate
    }
    note right of ProductCompositeService::ProductAggregate
    GET /product-composite/{productId}
    end note

    class ProductAggregate {
        productId: Int
        name: String
        weight: Int
        recommendations: List<RecommendationSummary>
        reviews: List<ReviewSummary>
        serviceAddresses: ServiceAddresses
    }

    class RecommendationSummary {
        recommendationId: Int
        author: String
        rate: Int
    }

    class ReviewSummary {
        reviewId: Int
        author: String
        subject: String
    }

    class ServiceAddresses {
        cmp: String <font color=green># product composite service</font>
        pro: String <font color=green># product service</font>
        rev: String <font color=green># review service</font>
        rec: String <font color=green># recommendation service</font>
    }

    ProductCompositeService ..> ProductAggregate
    ProductAggregate --> RecommendationSummary
    ProductAggregate --> ReviewSummary
    ProductAggregate --> ServiceAddresses
}

package product-composite {
    class ProductCompositeServiceImpl
}

package product {
    class ProductService
}

package review {
    class ReviewService
}

package recommendation {
    class RecommendationService
}

ProductCompositeService <|.. ProductCompositeServiceImpl
ProductCompositeServiceImpl ..> ProductService
ProductCompositeServiceImpl ..> ReviewService
ProductCompositeServiceImpl ..> RecommendationService
```

### Refactor: 抽 Service interfaces

```plantuml
hide circle

package api {
    interface ProductService #red {
        getProduct(productId: Int): Product
    }
    interface RecommendationService #red {
        getRecommendations(productId: Int): List<Recommendation>
    }
    interface ReviewService #red {
        getReviews(productId: Int): List<Review>
    }
}

package product-service {
    class "ProductService<font color=red>Impl</font>" as ps
}

package recommendation-service {
    class "RecommendationService<font color=red>Impl</font>" as rec
}

package review-service {
    class "ReviewService<font color=red>Impl</font>" as rev
}

ProductService <|.. ps
RecommendationService <|.. rec
ReviewService <|.. rev
```

- `Product`, `Recommendation`, `Review` 搬到 `api` 下
- 抽 interface: `ProductService`, `RecommendationService`, `ReviewService`
    - 先將原本的物件改名增加後綴 `Impl`
    - 抽 interface 到 `api` 套件
- 執行測試，確定 Refactor 後測試依然通過 `./gradlew test`

??? tip "IntelliJ 熱鍵"

    Move class: ++f6++
    Rename: ++shift+f6++ (vim ++"\\rn"++)
    抽 Interface: ++ctrl+t++ > Extract Interface...

### Test 3.4.1: Get product by productId

```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@EnableWireMock(ConfigureWireMock(portProperties = ["app.product-service.port", "app.recommendation-service.port", "app.review-service.port"]))
class ProductCompositeServiceApplicationTests {
    @Test
    fun `get product by product id`() {
        stubFor(get("/product/1").willReturn(okJson(""" { "productId": 1, "name": "product 1", "weight": 123, "serviceAddress": "localhost/127.0.0.1:7001" } """)))
        stubFor(
            get("/recommendation?productId=1").willReturn(
                okJson(
                    """ [
                        { "recommendationId": 1, "productId": 1, "author": "Teddy", "rate": 1, "content": "content-1", "serviceAddress": "localhost/127.0.0.1:7002" },
                        { "recommendationId": 2, "productId": 1, "author": "Teddy", "rate": 2, "content": "content-2", "serviceAddress": "localhost/127.0.0.1:7002" },
                        { "recommendationId": 3, "productId": 1, "author": "Teddy", "rate": 3, "content": "content-3", "serviceAddress": "localhost/127.0.0.1:7002" }
                    ] """
                )
            )
        )
        stubFor(
            get("/review?productId=1").willReturn(
                okJson(
                    """ [
                        { "reviewId": 1, "productId": 1, "author": "Teddy", "subject": "subject-1", "content": "content-1", "serviceAddress": "localhost/127.0.0.1:7003" },
                        { "reviewId": 2, "productId": 1, "author": "Teddy", "subject": "subject-2", "content": "content-2", "serviceAddress": "localhost/127.0.0.1:7003" },
                        { "reviewId": 3, "productId": 1, "author": "Teddy", "subject": "subject-3", "content": "content-3", "serviceAddress": "localhost/127.0.0.1:7003" }
                        ] """
                )
            )
        )

        client.get()
            .uri("/product-composite/1")
            .accept(APPLICATION_JSON)
            .exchange()
            .expectStatus().isOk()
            .expectHeader().contentType(APPLICATION_JSON)
            .expectBody()
            .jsonPath("$.productId").isEqualTo(1)
            .jsonPath("$.recommendations.length()").isEqualTo(3)
            .jsonPath("$.reviews.length()").isEqualTo(3)
    }
}
```

- 使用 `RestTemplate` 串接 API

    ??? tip "使用 `RestTemplate` 配合 `ParameterizedTypeReference` 接收 List 處理 generic type 的問題"

        ```kotlin title="Example"
        override fun getRecommendations(productId: Int): List<Recommendation> {
            return checkNotNull(
                restTemplate.exchange(
                    "http://$recommendationServiceHost:$recommendationServicePort/recommendation?productId=$productId",
                    HttpMethod.GET,
                    null,
                    object : ParameterizedTypeReference<List<Recommendation>>() {}
                ).body)
        }
        ```

- 逐步實作，分三個 subtask: `$.productId`, `$.recommendations.length()`, `$.reviews.length()`，一次做一個，還沒有做的 assertion 先註解起來。
- 定義 properties: microservices 的 host 與 port
- Spring 建議的 Mock Server
    - [WireMockk](https://docs.spring.io/spring-cloud-contract/docs/current/reference/html/project-features.html#features-wiremock) 測試使用這個
    - [MockWebServer](https://docs.spring.io/spring-framework/reference/web/webflux-webclient/client-testing.html)

### Test 3.4.2: Get product not found

```kotlin
@Test
fun `get product not found`() {
    stubFor(
        get("/product/13").willReturn(
            jsonResponse(
                """{"timestamp":  ${ZonedDateTime.now()}, "path": "/product/13", "httpStatus": ${HttpStatus.NOT_FOUND.value()}, "message": "Product with productId=13 not found"}""",
                HttpStatus.NOT_FOUND.value()
            )
        )
    )

    client.get()
        .uri("/product-composite/13")
        .accept(APPLICATION_JSON)
        .exchange()
        .expectStatus().isNotFound()
        .expectHeader().contentType(APPLICATION_JSON)
        .expectBody()
        .jsonPath("$.path").isEqualTo("/product-composite/13")
        .jsonPath("$.message").isEqualTo("Not found: 13")
}
```

### Test 3.4.3: Get product invalid parameter

```kotlin
@Test
fun `get product invalid parameter`() {
    stubFor(
        get("/product/-1").willReturn(
            jsonResponse(
                """{"timestamp":  ${ZonedDateTime.now()}, "path": "/product/-1", "httpStatus": ${HttpStatus.UNPROCESSABLE_ENTITY.value()}, "message": "Invalid productId: -1"}""",
                HttpStatus.UNPROCESSABLE_ENTITY.value()
            )
        )
    )

    client.get()
        .uri("/product-composite/-1")
        .accept(APPLICATION_JSON)
        .exchange()
        .expectStatus().isEqualTo(HttpStatus.UNPROCESSABLE_ENTITY)
        .expectHeader().contentType(APPLICATION_JSON)
        .expectBody()
        .jsonPath("$.path").isEqualTo("/product-composite/-1")
        .jsonPath("$.message").isEqualTo("Invalid: -1")
}
```
