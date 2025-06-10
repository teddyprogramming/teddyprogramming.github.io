# 3 Creating a Set of Cooperating Microservices

```plantuml
hide circle

class "Product Composite" as pc {
    Product ID
    Name
    Weight
    Recommendations
    Reviews
}

class "Product" as p {
    Product ID
    Name
    Weight
}

class "Review" as r {
    Product ID
    Review ID
    Author
    Subject
    Content
}

class "Recommendation" as rcd {
    Product ID
    Recommendation ID
    Author
    Rate
    Content
}

class RecommendationSummary {
    Recommendation ID
    Author
    Rate
}

class ReviewSummary {
    Review ID
    Author
    Subject
}

pc ..> p
pc ..> r
pc ..> rcd
pc ->"*" RecommendationSummary
pc ->"*" ReviewSummary
```

對於每個微服務，我們會建立一個 Spring Boot 專案，並有以下設定:

- 使用 Gradle 作為建構工具
- 產生符合 Java 21 的程式碼
- 專案會被打包成一個 fat JAR 檔案
- 引入 `Spring Actuator` 和 `WebFlux` 模組的相依套件
- 基於 Spring Boot v3.4.5（其依賴 Spring Framework v6.2.6）

由於這個階段我們尚未建立任何服務註冊機制，我們會將所有微服務都執行在 `localhost` 上，並為每個微服務使用固定的埠號。我們將使用以下埠號：

- Product Composite Service: `8000`
- Product Service: `8001`
- Recommendation Service: `8002`
- Review Service: `8003`

??? tip "設定 port"

    ```yaml title="application.yml"
    server.port=8000
    ```

[Spring Initializr - Product Service](https://start.spring.io/#!type=gradle-project-kotlin&language=kotlin&platformVersion=3.4.5&packaging=jar&jvmVersion=21&groupId=com.example&artifactId=product-service&name=product-service&description=Demo%20project%20for%20Spring%20Boot&packageName=com.example.product-service&dependencies=webflux,actuator)
當我們建立好這四個專案後，檔案結構會如下所示:

```
└ microservices/
    ├── product-composite-service
    ├── product-service
    ├── recommendation-service
    └── review-service
```

為了讓我們能夠用一個指令就建構所有微服務，我們可以在 Gradle 中設定一個 multi-project build。步驟如下:

1. 首先，我們在根目錄建立 `settings.gradle` 檔案，用來描述 Gradle 應該建構哪些專案：

    ```kotlin
    include(":microservices:product-composite-service")
    include(":microservices:product-service")
    include(":microservices:recommendation-service")
    include(":microservices:review-service")
    ```

2. 接著，我們會將其中一個專案中產生的 Gradle 執行檔複製過來，這樣就可以在多專案建構中重複使用它們:

    - `gradle/`
    - `gradlew`
    - `gradlew.bat`
    - `.gitignore`
    - `.gitattributes`

3. 我們不再需要在每個專案中產生的上述檔案，所以可以將它們移除。

4. 現在，我們可以用一個指令來建構所有微服務:

    ```shell
    ./gradlew build
    ```

!!! note "使用 `git add -A` 將沒有忽略的檔案加到版控系統"

從 Spring Boot v2.5.0 開始，當執行 `./gradlew build` 指令時，會產生兩個 JAR 檔案：一個是一般的 JAR 檔案，另一個是只包含編譯後 Java 類別檔的 plain JAR。由於我們不需要這個新的 plain JAR，因此停用了它的建立，這樣就可以使用萬用字元來引用一般的 JAR 來啟動 Spring Boot 應用，例如：

`java -jar microservices/product-service/build/libs/*.jar`

為了停用 plain JAR 的建立，我們在每個 microservice 的 `build.gradle.kts` 檔案中加上以下設定：

```gradle title="build.gradle.kts"
tasks.named<Jar>("jar") {
    enabled = false
}
```

Product Service cases

| Request                 | Response                                         |
| ----------------------- | ------------------------------------------------ |
| GET /product/1          | OK                                               |
| GET /product/no-integer | BAD_REQUEST (`Type mismatch.`) [^1]              |
| GET /product/13         | NOT_FOUND (`Product (id: 13) not found.`)        |
| GET /product/-1         | UNPROCESSABLE_ENTITY (`Invalid product id: -1.`) |

[^1]: 在 properties 檔案新增 `server.error.include-message=always` 將會在 response 中提供 `message`

!!! note
    錯誤的訊息，必須包含兩個欄位 `path`, `message`

??? tip "撰寫 HTTP request 測試"

    - `@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)`
    - `@Autowired private lateinit var webTestClient: WebTestClient`
    - test code
        ```kotlin
        webTestClient.get().uri("/product/1")
            .exchange()
            .expectStatus().isOk
            .expectBody()
            .jsonPath("$.id").isEqualTo("1")
            .jsonPath("$.name").isEqualTo("Product 1")
            .jsonPath("$.weight").isEqualTo(123)
        ```

??? tip "Global Exception Handler"

    1. 新增 Global Exception Handler 的類別

    2. 在類別標上 `@RestControllerAdvice`

    3. 在類別中新增處理例外的 function

        - 假設處理 `NotFoundException`，則在 function 標上 `@ExceptionHandler(NotFoundException::class)`
        - 假設回傳的 http status 是 not found，則在 function 標上 `@ResponseStatus(HttpStatus.NOT_FOUND)`

    4. function 接受處理的例外當作參數。
        - 例如 `fun handleNotFound(e: NotFoundException)`

    5. function 接受 `ServerHttpRequest` 當作參數，用以取得 path
        - 例如 `fun handleNotFound(.., request: ServerHttpRequest)`
        - path 為 `request.path.pathWithinApplication().value()`

    6. 宣告類別用以表達回傳的資料。

        ```kotlin
        data class Response(
            val path: String,
            val message: String,
        )
        ```

Recommendation service cases

| Request                                  | Response                                                                  |
| ---------------------------------------- | ------------------------------------------------------------------------- |
| GET /recommendation?productId=1          | OK, 3 筆 recommendations                                                  |
| GET /recommendation                      | BAD_REQUEST (`Required query parameter 'productId' is not present.`) [^1] |
| GET /recommendation?productId=no-integer | BAD_REQUEST (`Type mismatch.`)                                            |
| GET /recommendation?productId=113        | OK, empty                                                                 |
| GET /recommendation?productId=-1         | UNPROCESSABLE_ENTITY (`Invalid productId: -1.`)                           |

??? tip "實作提示"

    - 驗證 json 內容

        ```kotlin
        @Test
        fun `get recommendations`() {
            webTestClient
                .get()
                .uri("/recommendation?productId=1")
                .exchange()
                .expectStatus().isOk
                .expectHeader().contentType(MediaType.APPLICATION_JSON)
                .expectBody()
                .json(
                    """[
                    { "id": 1, "productId": 1, "content": "recommendation 1", "author": "author 1", "rate": 4.5 },
                    { "id": 2, "productId": 1, "content": "recommendation 2", "author": "author 2", "rate": 3.0 },
                    { "id": 3, "productId": 1, "content": "recommendation 3", "author": "author 3", "rate": 5.0 }
                ]"""
            )
        }
        ```

        Json 預設比對是寬鬆模式 (LENIENT)，field 順序不重要，實際上有多的 field 沒有 assert 也不重要。

    - Query parameter: `@RequestParam`

    - 使用 jsonPath 驗證 list 的長度

        ```kotlin
        @Test
        fun `empty recommendation`() {
            webTestClient
                .get()
                .uri("/recommendation?productId=113")
                .exchange()
                .expectStatus().isOk
                .expectHeader().contentType(MediaType.APPLICATION_JSON)
                .expectBody()
                .jsonPath("$.length()").isEqualTo(0)
        }
        ```

!!! note "重構"
    - 新增 `util` module
        - 新增純 gradle 專案，不必是 spring boot
        - 設定 `build.gradle.kts`

            ```kotlin title="build.gradle.kts"
            plugins {
                // ...
                // 從既有的 service 專案，複製 plugins 的內容
                kotlin("plugin.spring") version "1.9.25"
                id("org.springframework.boot") version "3.4.5"
                id("io.spring.dependency-management") version "1.1.7"
            }

            dependencies {
                implementation("org.springframework:spring-context") // 我們不需要 spring boot starter，這裡使用 spring-context
                implementation("org.springframework:spring-webflux") // 我們後面會需要 webflux 的類別，這需要 spring-webflux
                // ...
            }
            tasks.named<BootJar>("bootJar") {
                enabled = false
            }
            ```
    - 設定 Auto-Configuration，讓相依此 module 的 project，可以自動掃描 `util` 定義的 beans

        ```kotlin title="UtilConfiguration.kt"
        @Configuration
        @ComponentScan("org.example.util") // 這裡填入 util 的 base package，以掃描在此 package 下的所有 bean
        class UtilConfiguration {
        }
        ```

        ```imports title="resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports"
        org.example.exception.UtilConfiguration
        ```

    - 將例外移到 `util` module 共用
    - 將 Global Exception Handler 移到 `util` module 共用

Review service cases

| Request                          | Response                                                                  |
| -------------------------------- | ------------------------------------------------------------------------- |
| GET /review?productId=1          | OK, 3 筆 reviews                                                          |
| GET /review                      | BAD_REQUEST (`Required query parameter 'productId' is not present.`) [^1] |
| GET /review?productId=no-integer | BAD_REQUEST (`Type mismatch.`) [^1]                                       |
| GET /review?productId=213        | OK, empty                                                                 |
| GET /review?productId=-1         | UNPROCESSABLE_ENTITY (`Invalid productId: -1.`)                           |

Product Composite service

| Request | Response |
| - | - |
| GET /product-composite/1 | OK, length(reviews) = 1, length(recommendations) = 1 |
| GET /product-composite/2 | NOT_FOUND (`Not found productId: 2.`) |
| GET /product-composite/-1 | UNPROCESSABLE_ENTITY (`Invalid productId: -1.`) |

- 使用假資料，`productId=1` 時，reviews, recommendations 各一筆

!!! note "重構"

    - 提取 `ProductService`, `RecommendationService`, `ReviewService` 成 interface
    - 新增 `api` module，將 service interface 放在這個 module 中

??? tip "測試"

    - 使用 [WireMock](https://wiremock.org/docs/spring-boot/)
        - `testImplementation("org.wiremock.integrations:wiremock-spring-boot:3.6.0")`
        - 在測試標上 `@EnableWireMock`
            ```kotlin
            // ...
            @EnableWireMock(ConfigureWireMock(baseUrlProperties = ["product-service.host", "review-service.host", "recommendation-service.host"]))
            class ProductCompositeServiceApplicationTests {
                // ...
            }
            ```
        - 用 `subFor` 來 mock api 回傳的資料
            ```kotlin
            stubFor(get("/product/1").willReturn(okJson("{\"id\": 1, \"name\": \"Product 1\", \"weight\": 123}")))
            stubFor(get("/review?productId=1").willReturn(okJson("[{\"id\": 1, \"productId\": 1, \"author\": \"Author 1\", \"subject\": \"Subject 1\", \"content\": \"Content 1\"}]")))
            stubFor(get("/recommendation?productId=1").willReturn(okJson("[{\"id\": 1, \"productId\": 1, \"author\": \"Author 1\", \"rate\": 4.2, \"content\": \"Content 1\"}]")))
            ```

??? tip "實作 service client"

    ```kotlin
    interface ProductServiceClient : ProductService {
        @GetExchange("/product/{productId}")
        override suspend fun getProduct(@PathVariable productId: Int): Product
    }

    @Bean
    fun productServiceClient(@Value("\${product-service.host}") host: String): ProductServiceClient {
        val webClient = WebClient.builder().baseUrl(host).build()
        val adapter = WebClientAdapter.create(webClient)
        val factory = HttpServiceProxyFactory.builderFor(adapter).build()
        return factory.createClient(ProductServiceClient::class.java)
    }
    ```

    - `WebClient` 才有支援 suspend，`RestTemplate`, `RestClient` 如果要產生 suspend method 的 API，在呼叫時，會拋出錯誤。

    參考: https://docs.spring.io/spring-framework/reference/integration/rest-clients.html#rest-http-interface

半自動 Integration tests

| Request | Response |
| - | - |
| GET /product-composite/1 | len(reviews) = 3 && len(recommendations) = 3 |
| GET /product-composite/13 | 404 (`Product (id: 13) not found.`) |
| GET /product-composite/113 | len(reviews) = 3 && len(recommendations) = 0 |
| GET /product-composite/213 | len(reviews) = 0 && len(recommendations) = 3 |

??? tip

    - 建立新的專案 integration-tests

    - ` webTestClient = WebTestClient.bindToServer().baseUrl("http://localhost:8000").build()`

    - 手動啟動所有 microservices 後再執行測試

## 參考

<https://github.com/PacktPublishing/Microservices-with-Spring-Boot-and-Spring-Cloud-Third-Edition/tree/main/Chapter03>
