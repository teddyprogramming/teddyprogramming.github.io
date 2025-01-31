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

- Product Composite: `6000`
- Product: `6001`
- Recommendation: `6002`
- Review: `6003`

!!! note "書本使用 port 7000, 7001, 7002, 7003"

    因為 macos 中，port 7000 被 airplay 佔用了，所以改用 `6000`, `6001`, `6002`, `6003`

# 建立 microservice 專案

按下面目錄結構新增 4 個 microservices 專案。

```text
/
├── product-composite-service
├── product-service
├── recommendation-service
└── review-service
```

- 使用 Spring Initializr 建立專案
- 專案相依 `actuator`, `webflux`
- 使用 gradle 編譯專案

    ??? tip "使用 `./gradlew build`"

        ```shell
        cd product-service; ./gradlew build; cd -; cd recommendation-service; ./gradlew build; cd -; cd review-service; ./gradlew build; cd -; cd product-composite service; ./gradlew build; cd -;
        ```

# 單一指令編譯專案

調整專案結構，使可以用單一指令 `./gradlew build` 編譯所有專案。

!!! note "IntelliJ 可能需要將 gradle 設定清除後重新開啟 IDE。"

# Adding RESTful APIs

## Product API

將 product-service 的 port 設定成 `6001`。

```plantuml
hide circle

class ProductService {
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
```

!!! note "書中作者將 Controller 視作 Service，因此命名成 `ProductService`。"

### Test: Get product

讓測試通過。

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

- 設定 `@SpringBootTest` 參數 `webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT`。
- Autowire `WebTestClient`。
- 目前沒有 persistence layer，用假資料即可。
- 實作 `ServiceUtil` 封裝 `serviceAddress` 的細節。

    - 格式為 `${hostName}/${ipAddress}:${port}`

        ??? tip "使用 `InetAddress` 取得 `hostName` 與 `ipAddress`"
            取得 host name: `InetAddress.getLocalHost().getHostName()`
            取得 host IP: `InetAddress.getLocalHost().getHostAddress()`

### Test: Get product not found

讓測試通過。

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

- 宣告 `NotFoundException`，在 product 找不到時拋出。
- 使用 `@RestControllerAdvice` 處理例外狀況

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

- 例外狀況回傳 `HttpErrorInfo`

    ```plantuml
    hide circle

    class HttpErrorInfo {
        timestamp: ZonedDateTime
        path: String
        httpStatus: HttpStatus
        message: String
    }
    ```


### Test: Get product negative product id

讓測試通過。

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

- 宣告 `InvalidInputException`，在 `productId` 為負數時拋出。
- 處理例外拋出時回傳的資料。

### Test: Get product invalid parameter productId type mismatch

讓測試通過。

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

- 資料型態不吻合會被 Spring 擋下，預設錯誤訊息沒有 `message` 欄位，通過設定可以加入。
    - `server.error.include-message=always`

## Refactor: 新增程式碼共用的 module

### `util` module

- 新增 `util` module。

    - 注意 `plugins` 與 `dependencies` 的內容與下面一致，可以減少很多詭異事件..

        ```gradle title="build.gradle.kts" hl_lines="3 7 9"
        plugins {
            kotlin("jvm") version "1.9.25"
            id("io.spring.dependency-management") version "1.1.7"
        }

        dependencies {
            implementation(platform("org.springframework.boot:spring-boot-dependencies:3.4.2"))
            implementation("org.jetbrains.kotlin:kotlin-reflect")
            implementation("org.springframework:spring-context")
            implementation("org.springframework:spring-web")
            testImplementation("org.jetbrains.kotlin:kotlin-test-junit5")
            testImplementation("io.projectreactor:reactor-test")
            testRuntimeOnly("org.junit.platform:junit-platform-launcher")
        }
        ```

- 將 `ServiceUtil` 移到 `util` 下
    - IntelliJ 熱鍵: ++f6++
- 執行測試確保 Refactor 結果沒有破壞功能。
    - 如果測試失敗，應該是需要設定 product-service 的 scan package，包含 `ServiceUtil` 所在的 package。

### `api` module

- 新增 `api` module
- 將 Exception Handler, `NotFoundException`, `InvalidInputException` 移到 `api` 下。
- 執行測試

## Review API

```plantuml
hide circle

class ReviewService {
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
```

### Test: Get review

讓測試通過。

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

### Test: Get review not found

讓測試通過。

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

### Test: Product id is negative

讓測試通過。

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

### Test: Prodict id is not integer

讓測試通過。

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

### Test: Missig product id

讓測試通過。(不需改程式碼)

```kotlin
@Test
fun `get reiew missing parameter`() {
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

## Recommendation API

```plantuml
hide circle

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
```

### Test: Get recommendations

讓測試通過。

```kotlin
@Test
fun `get recommendations by product id`() {
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

### Test: Get recommentations not found

讓測試通過。

```kotlin
@Test
fun `get recommendations not found`() {
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

### Test: Negative product id

讓測試通過。

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

### Test: Product id is not integer

讓測試通過。

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

### Test: Missing prodict id

讓測試通過。

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

## Refactor: 抽 Service interface

```plantuml
hide circle

package api {
    interface ProductService #orange {
        getProduct(productId: Int): Product
    }
    interface RecommendationService #orange {
        getRecommendations(productId: Int): List<Recommendation>
    }
    interface ReviewService #orange {
        getReviews(productId: Int): List<Review>
    }

    class Product #orange
    class Recommendation #orange
    class Review #orange

    ProductService .> Product
    RecommendationService .> Recommendation
    ReviewService .> Review
}

package product-service {
    class "ProductService<font color=orange>Impl</font>" as ProductServiceImpl
}
package recommendation-service {
    class "RecommendationService<font color=orange>Impl</font>" as RecommendationServiceImpl
}
package review-service {
    class "ReviewService<font color=orange>Impl</font>" as ReviewServiceImpl
}

ProductService <|.. ProductServiceImpl
RecommendationService <|.. RecommendationServiceImpl
ReviewService <|.. ReviewServiceImpl
```

- 將 `ProductService`, `RecommendationService`, `ReviewService` 抽成 interface 放到 `api` 下。
    - 先重新命名既有的 class，加上後墜 `Impl`。(使用 ++shift+f6++ 或 vim ++"\\rn"++ )
    - 抽 interface (使用 ++ctrl+t++ > Extract Interface...)
- 將 `Product`, `Recomendation`, `Review` 移動到 `api` 下。
    - 使用 ++f6++

## Product Composite API

```plantuml
hide circle

package api {
    interface ProductService
    interface RecommendationService
    interface ReviewService
}

package product-composite {
    class ProductCompositeService {
        getProduct(productId: Int): ProductAggregate
    }
    note right of ProductCompositeService::ProductAggregate
    GET /product-composite/{productId}
    end note

    class ProductCompositeIntegration

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
    ProductCompositeService --> ProductCompositeIntegration
    ProductAggregate --> RecommendationSummary
    ProductAggregate --> ReviewSummary
    ProductAggregate --> ServiceAddresses
}

package product {
    class ProductServiceImpl
}

package review {
    class ReviewServiceImpl
}

package recommendation {
    class RecommendationServiceImpl
}

ProductService <|.. ProductServiceImpl
RecommendationService <|.. RecommendationServiceImpl
ReviewService <|.. ReviewServiceImpl
ProductService <|.. ProductCompositeIntegration
RecommendationService <|.. ProductCompositeIntegration
ReviewService <|.. ProductCompositeIntegration
```

### Test: Get product

讓測試通過。

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
    - [WireMockk](https://docs.spring.io/spring-cloud-contract/docs/current/reference/html/project-features.html#features-wiremock) 這裡的測試使用這個
    - [MockWebServer](https://docs.spring.io/spring-framework/reference/web/webflux-webclient/client-testing.html)

### Test: Get product not found

讓測試通過。

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

### Test: Get product invalid parameter

讓測試通過。

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

## Disable gradle build to generate plain jar file

預設 `gradle build` 會產生兩個 jar 檔案:

1. ordinary JAR file
2. plain JAR file containing only the class files resulting from compiling the Java files in the Spring Boot application

我們不需要第 2 個 jar 檔。設定 `build.gradle.kts` 讓 `gradle build` 僅產生第一個 jar 檔案。

??? tip

    在跟目錄新增檔案，加入以下設定。

    ```gradle title="build.gradle.kts"
    subprojects {
        afterEvaluate {
            if (plugins.hasPlugin("org.springframework.boot")) {
                tasks.getByName("jar") {
                    enabled = false
                }
            }
        }
    }
    ```

## 手動測試

- 啟動 microservices

    ```shell
    $ for jar in $(find . -name "*-service-*.jar"); do
        java -jar $jar &
    done
    ```

- 測試 product-service, recommendation-service, review-service, product-composite-service 的 API。

- 停掉所有 microservices

    ```shell
    pkill -P $$
    ```

## 半自動化測試

TODO()
