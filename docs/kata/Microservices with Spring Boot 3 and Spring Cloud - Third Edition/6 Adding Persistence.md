`product` 和 `recommendation` 使用 MongoDB，`review` 使用 MySQL。

```plantuml
component "Product Composite" as pc
component "Product" as p
component "Recommendation" as rd
component "Review" as r
database "MongoDB" as pm
database "MongoDB" as rdm
database "MySQL" as rm

pc --> p
pc --> rd
pc --> r
p --> pm
rd --> rdm
r --> rm
```

## 實作 Product service 的 persistence layer

```plantuml
hide circle
interface ProductRepository implements PagingAndSortingRepository, CrudRepository {
    findByProductId(productId: int): Optional<ProductEntity>
}
class "<font color=gray>@Document(collection=products)</font>\nProductEntity" as entity {
    <font color=gray>@Id</font> id: String
    <font color=gray>@Version</font> version: Integer
    <font color=gray>@Indexed(unique=true)</font> productId: int
    name: String
    weight: int
}
ProductRepository ..> entity
```

### 測試: 新增 Product

實作測試，並讓測試通過。

```kotlin hl_lines="1-2 7-21"
@DataMongoTest
@Import(MongoDbContainerConfig::class)
class ProductRepositoryTest {
    @Autowired
    lateinit var productRepository: ProductRepository

    @Test
    fun create() {
        val product = ProductEntity(
            productId = "1",
            name = "Product 1",
            weight = 100
        )
        val savedEntity = productRepository.save(product)

        savedEntity.id shouldNotBe null
        savedEntity.version shouldBe 1
        savedEntity.productId shouldBe "1"
        savedEntity.name shouldBe "Product 1"
        savedEntity.weight shouldBe 100
    }
}
```

- Spring data (MongoDb)

    ??? info "gradle dependencies"

        ```gradle
        implementation("org.springframework.boot:spring-boot-starter-data-mongodb")
        ```

- MongoDb testcontainer

    ??? info "gradle dependencies"

        ```gradle
        testImplementation("org.testcontainers:junit-jupiter")
        testImplementation("org.springframework.boot:spring-boot-testcontainers")
        testImplementation("org.testcontainers:mongodb")
        ```

    ??? info "MongoDbContainerConfig"

        ```gradle title="MongoDbConfiguration.kt"
        import org.springframework.boot.test.context.TestConfiguration
        import org.springframework.boot.testcontainers.service.connection.ServiceConnection
        import org.springframework.context.annotation.Bean
        import org.testcontainers.containers.MongoDBContainer

        @TestConfiguration
        class MongoDbContainerConfig {
            @Bean
            @ServiceConnection
            fun container(): MongoDBContainer {
                return MongoDBContainer("mongo:6.0.4")
            }
        }
        ```

### 測試: 更新 Product

```kotlin
class ProductRepositoryTest {
    // ...

    companion object {
        private lateinit var savedEntity: ProductEntity
    }

    @Test
    @Order(2)
    fun update() {
        savedEntity = productRepository.save(savedEntity.copy(name = "Product 2"))

        savedEntity.version shouldBe 2
        savedEntity.name shouldBe "Product 2"
    }
}
```

- 將 `create` 測試的 `savedEntity` 變數 extract 成 static variable。
- 將 `create` 標上 `@Order(1)`。
- 在 class 標 `@TestMethodOrder(MethodOrderer.OrderAnnotation::class)` 讓 `@Order` 作用。

### 測試: 取得 Product

```kotlin
@Test
@Order(3)
fun get() {
    val result = productRepository.findById(savedEntity.id!!)

    result.isPresent shouldBe true
    result.get() shouldBe savedEntity
}
```

### 測試: 重複的 key

```kotlin
import org.springframework.dao.DuplicateKeyException

@Test
@Order(4)
fun `duplicate id`() {
    shouldThrow<DuplicateKeyException> {
        productRepository.save(savedEntity.copy(version = 0))
    }
}
```

### 測試: 樂觀鎖(optimistic lock)

```kotlin
@Test
@Order(5)
fun `optimistic lock`() {
    val entity1 = productRepository.findById(savedEntity.id!!).get()
    val entity2 = productRepository.findById(savedEntity.id!!).get()

    savedEntity = productRepository.save(entity1) // version + 1
    shouldThrow<OptimisticLockingFailureException> {
        productRepository.save(entity2) // use old version to save
    }
}
```

### 測試: 刪除 Product

```kotlin
@Test
@Order(6)
fun delete() {
    productRepository.delete(savedEntity)

    productRepository.existsById(savedEntity.id!!) shouldBe false
}
```

### 測試: pagination

```kotlin
@Test
@Order(7)
fun pagination() {
    productRepository.saveAll(
        (1001..1007)
            .shuffled()
            .map { i -> ProductEntity("$i", "Product $i", i) }
            .toList()
    )

    var nextPageRequest: Pageable = PageRequest.of(0, 3, Sort.by(Sort.Order.asc("productId")))
    productRepository.findAll(nextPageRequest) should {
        it.totalPages shouldBe 3
        it.content shouldHaveSize 3
        it.content.map { it.productId } shouldBe listOf("1001", "1002", "1003")
        it.hasNext() shouldBe true
        nextPageRequest = it.nextPageable()
    }

    productRepository.findAll(nextPageRequest) should {
        it.totalPages shouldBe 3
        it.content shouldHaveSize 3
        it.content.map { it.productId } shouldBe listOf("1004", "1005", "1006")
        it.hasNext() shouldBe true
        nextPageRequest = it.nextPageable()
    }

    productRepository.findAll(nextPageRequest) should {
        it.totalPages shouldBe 3
        it.content shouldHaveSize 1
        it.content.map { it.productId } shouldBe listOf("1007")
        it.hasNext() shouldBe false
        nextPageRequest = it.nextPageable()
    }
}
```

### 測試: 查詢不存在的 Proudct

```kotlin
// ...
@TestMethodOrder(OrderAnnotation::class)
@Import(MongoDbContainerConfig::class)
class ProductServiceImplApplicationTests {

    // ...

    @Test
    @Order(1)
    fun `get not-existing product`() {
        client.get()
            .uri("/product/1")
            .accept(APPLICATION_JSON)
            .exchange()
            .expectStatus().isNotFound()
    }

    // ...
}
```

- 在測試執行，需搭配 MongoDB test container，故標上 `@Import(MongoDbContainerConfig::class)`。

    ???tip "`MongoDbContainerConfig` 相關程式碼"

        ```gradle title="build.gradle.kts"
        testImplementation("org.testcontainers:junit-jupiter")
        testImplementation("org.springframework.boot:spring-boot-testcontainers")
        testImplementation("org.testcontainers:mongodb")
        ```

        ```kotlin title="MongoDbContainerConfig.kt"
        import org.springframework.boot.test.context.TestConfiguration
        import org.springframework.boot.testcontainers.service.connection.ServiceConnection
        import org.springframework.context.annotation.Bean
        import org.testcontainers.containers.MongoDBContainer

        @TestConfiguration
        class MongoDbContainerConfig {
            @Bean
            @ServiceConnection
            fun container(): MongoDBContainer {
                return MongoDBContainer("mongo:6.0.4")
            }
        }
        ```
- 原本既有的查詢 productId = 1 的測試，先標上 `@Disabled` 讓測試不要執行。
- mapstruct

    ???tip

        ```gradle title="build.gradle.kts"
        plugins {
            kotlin("kapt") version "1.9.25"
        }

        dependencies {
            implementation("org.mapstruct:mapstruct:1.6.3")
            kapt("org.mapstruct:mapstruct-processor:1.6.3")
        }

        kapt {
            arguments {
                arg("mapstruct.defaultComponentModel", "spring")
                arg("mapstruct.defaultInjectionStrategy", "field")
            }
        }
        ```

        - [mapstruct 官方 kotlin gradle 範例](https://github.com/mapstruct/mapstruct-examples/blob/main/mapstruct-kotlin-gradle/build.gradle.kts)

### 測試: 新增 Product API

```kotlin
@Test
fun `create product`() {
    client.post()
        .uri("/product")
        .bodyValue(Product(1, "Product 1", 100, "SA"))
        .accept(APPLICATION_JSON)
        .exchange()
        .expectStatus().isOk()
        .expectHeader().contentType(APPLICATION_JSON)

    productRepository.findById("1").isPresent shouldBe true
}
```

## 實作 Recommendation service 的 persistence layer

```plantuml
hide circle
class RecommendationRepository implements PagingAndSortingRepository, CrudRepository {
    findByProductId(productId: int): List<RecommendationEntity>
}
class "<font color=gray>@Document(collection=recommendations)</font>\nRecommendationEntity" as entity {
    <font color=gray>@Id</font> id: String
    <font color=gray>@Version</font> version: Integer
    productId: int
    recommendationId: int
    author: String
    rating: int
    content: String
}
note bottom of entity
index:
@CompoundIndex(name = "prod-rec-id", unique = true, def = "{'productId': 1, 'recommendationId' : 1}")
end note
RecommendationRepository ..> entity
```

- `{productId: 1, recommendationId: 1}` 代表 `productId` 與 `recommendationId` 皆遞增

## 實作 Review service 的 persistence layer

```plantuml
hide circle
class ReviewRepository implements CrudRepository {
    <font color=gray>@Transactional(readOnly=true)</font> findByProductId(productId: int): List<ReviewEntity>
}
class "<font color=gray>@Entity</font>\n<font color=gray>@Table(name=reviews)</font>\nReviewEntity" as entity {
    <font color=gray>@Id</font> <font color=gray>@GeneratedValue</font> id: int
    <font color=gray>@Version</font> version: int
    productId: int
    reviewId: int
    author: String
    subject: String
    content: String
}
note bottom of entity
index:
@Table(name = "reviews", <font color=greenyellow>indexes = { @Index(name = "reviews_unique_idx", unique = true, columnList = "productId,reviewId") }</font>)
end note

ReviewRepository ..> entity
```

## 透過 docker 對 MongoDB 查詢

```shell
$ docker-compose exec mongodb mongosh product-db --quiet --eval "db.products.find()"
```

## 透過 docker 對 MySQL 查詢

```shell
$ docker-compose exec mysql mysql -uuser -p review-db -e "select * from reviews"
```
