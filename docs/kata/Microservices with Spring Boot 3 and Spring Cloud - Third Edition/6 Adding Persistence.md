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

## Product Service

### 功能: 新增 Product

#### 測試: Reopsitory 新增 Product

```kotlin
@DataMongoTest
@Import(MongoDbConfiguration::class)
@TestMethodOrder(OrderAnnotation::class)
class ProductRepositoryTest {
    @Autowired
    private lateinit var productRepository: ProductRepository

    @Test
    @Order(1)
    fun `add product`() {
        val entity = productRepository.save(ProductEntity(productId = 1, name = "product-1", weight = 100))
        productRepository.existsById(entity.id!!) shouldBe true
    }
}
```

- 新增 String Data MongoDB 相依

    ???tip

        在 build.gradle.kts 的 dependencies 區塊中 ++cmd+n++ > Add Starters... > String Data MongoDB > 勾選 > OK > ++cmd+i++

- 新增 String testcontainers 相依

    ???tip

        在 build.gradle.kts 的 dependencies 區塊中 ++cmd+n++ > Add Starters... > Testcontainers > 勾選 > OK > ++cmd+i++

- 實作 `MongoDbConfiguration`

    ???tip

        ```kotlin title="MongoDbConfiguration.kt"
        @TestConfiguration
        class MongoDbConfiguration {
            @Bean
            @ServiceConnection
            fun container(): MongoDBContainer {
                return MongoDBContainer("mongo:6.0.4")
            }
        }
        ```

- 實作 `ProductRepository`, `ProductEntity`

    ???tip

        ```kotlin title="ProductRepository.kt"
        interface ProductRepository : MongoRepository<ProductEntity, String>
        ```

        ```kotlin title="ProductEntity.kt"
        import org.springframework.data.annotation.Id
        import org.springframework.data.annotation.Version
        import org.springframework.data.mongodb.core.mapping.Document

        @Document("product")
        data class ProductEntity(
            @Id
            val id: String? = null,
            @Version
            val version: Long = 0,
            @Indexed(unique = true)
            val productId: Int,
            val name: String,
            val weight: Int,
        )
        ```

        ```properties title="application.properties"
        spring.data.mongodb.auto-index-creation=true
        ```
#### 測試: Repository 新增重複的 Product

```kotlin
@Test
@Order(2)
fun `add duplicate product`() {
    shouldThrow<DuplicateKeyException> {
        productRepository.save(ProductEntity(productId = 1, name = "product-1", weight = 100))
    }
}
```

#### 測試: Mapper 將 CreateProductrequest 轉換成 ProductEntity

```kotlin
@SpringBootTest
class ProductMapperTest {
    @Autowired
    lateinit var productMapper: ProductMapper

    @Test
    fun toEntity() {
        productMapper.toEntity(CreateProductRequest(productId = 1, name = "product-1", weight = 100)) should {
            it.id shouldBe null
            it.version shouldBe 0
            it.productId shouldBe 1
            it.name shouldBe "product-1"
            it.weight shouldBe 100
        }
    }
}
```

- 新增 mapstruct dependencies

    ???tip

        ```kotin title="build.gradle.kts"
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
            }
        }
        ```

- 實作 Mapper

    ???tip

        ```kotlin title="ProductMapper.kt"
        import org.example.productservice.persistence.ProductEntity
        import org.mapstruct.Mapper

        @Mapper
        interface ProductMapper {
            fun toEntity(createProductRequest: CreateProductRequest): ProductEntity
        }
        ```

#### 測試: Mapper 將 ProductEntity 轉換成 Product

```kotlin
@Test
fun toProduct() {
    productMapper.toProduct(ProductEntity(id = "1", version = 1, productId = 1, name = "product-1", weight = 100)) should {
        it.productId shouldBe 1
        it.name shouldBe "product-1"
        it.weight shouldBe 100
    }
}
```

- 實作 toProduct

    ??? tip

        ```kotlin
        @Mappings(
            Mapping(target = "serviceAddress", constant = "")
        )
        fun toProduct(productEntity: ProductEntity): Product
        ```

#### 測試: API 實作新增 Product

```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestMethodOrder(OrderAnnotation::class)
@Import(MongoDbConfiguration::class)
class ProductServiceImplApplicationTests {

    @Test
    @Order(2) // (1)!
    fun `create product`() {
        client.post()
            .uri("/product")
            .accept(APPLICATION_JSON)
            .bodyValue(CreateProductRequest(productId = 1, name = "product-1", weight = 100))
            .exchange()
            .expectStatus().isOk()
            .expectHeader().contentType(APPLICATION_JSON)
            .expectBody()
            .jsonPath("$.productId").isEqualTo(1)
            .jsonPath("$.name").isEqualTo("product-1")
            .jsonPath("$.weight").isEqualTo(100)
    }

    // ...
}
```

1. 這裡 `@Order` 給 2，後面會需要在前面測試不存在的 case

- 在 `ProductService` 新增 `createProduct` 方法
- 如果需要，調整 `CreateProductRequest` 所放的 module
- 在 `ProductServiceImpl` 中實作 `createProduct` 的 API
- 搭配 `ProductMapper`, `ProductRepository` 實作 `createProduct`

    ???tip

        ```kotlin title="ProductServiceImpl.kt"
        @PostMapping("/product")
        override fun createProduct(@RequestBody request: CreateProductRequest): Product {
            return productRepository.save(request.toEntity()).toProduct()
        }

        private fun CreateProductRequest.toEntity(): ProductEntity {
            return productMapper.toEntity(this)
        }

        private fun ProductEntity.toProduct(): Product {
            return productMapper.toProduct(this).copy(serviceAddress = serviceUtil.getServiceAddress())
        }
        ```

#### 測試: API 實作新增重複 Product ID

```kotlin
@Test
@Order(3)
fun `duplicate error`() {
    client.post()
        .uri("/product")
        .accept(APPLICATION_JSON)
        .bodyValue(CreateProductRequest(productId = 1, name = "product-1", weight = 100))
        .exchange()
        .expectStatus().isEqualTo(HttpStatus.UNPROCESSABLE_ENTITY)
        .expectHeader().contentType(APPLICATION_JSON)
        .expectBody()
        .jsonPath("$.path").isEqualTo("/product")
        .jsonPath("$.message").isEqualTo("Duplicate key, Product Id: 1")
}
```

- 捕捉 `DuplicateKeyException` 拋出 `InvalidInputException`

### 功能: 查詢 Product

#### 測試: Repository 查詢 Product by productId

```kotlin title="ProdcutRepository.kt"
@Test
@Order(3)
fun `get product`() {
    productRepository.findByProductId(savedEntity.productId).get() should {
        it.productId shouldBe 1
        it.name shouldBe "product-1"
        it.weight shouldBe 100
    }
}
```

- 在 `ProductRepository` 新增 `findByProductId` 的方法

#### 測試: API 實作查詢 Product

```kotlin title="ProductServiceImplApplicationTests.kt"
class ProductServiceImplApplicationTests {
    @Test
    @Order(1) // (1)!
    fun `get product not exist`() {
        client.get()
            .uri("/product/1")
            .accept(APPLICATION_JSON)
            .exchange()
            .expectStatus().isNotFound()
            .expectHeader().contentType(APPLICATION_JSON)
            .expectBody()
            .jsonPath("$.path").isEqualTo("/product/1")
            .jsonPath("$.message").isEqualTo("Product with productId=1 not found")
    }

    // ...

    @Test
    @Order(3) // (2)!
    fun `get product by productId`() {
        // ...
    }
}
```

1. 在新增 Product 前，測試 Product 不存在
2. 原本查詢的測試，放在新增 Product 後執行

<!-- 避免下面受上面的影響，將下面的條列式內容被當作程式碼的註解，所以這裡做一個隔開的作用 -->

- 調整 `getProduct` 的實作
- 刪除 `productId == 13` 回傳找不到 Product 的判斷。

    ???tip

        ```kotlin hl_lines="6-8" title="ProductServiceImpl.kt"
        @GetMapping("/product/{productId}")
        override fun getProduct(@PathVariable productId: Int): Product {
            if (productId < 1) {
                throw InvalidInputException("Invalid productId: $productId")
            }
            return productRepository.findByProductId(productId).orElseThrow {
                NotFoundException("Product with productId=$productId not found")
            }.toProduct()
        }
        ```

### 功能: 刪除 Product

#### 測試: Repository 刪除 Product

```kotlin
@Test
@Order(3)
fun `delete product`() {
    productRepository.deleteByProductId(savedEntity.productId)
    productRepository.existsById(savedEntity.id!!) shouldBe false
}
```

#### 測試: API 刪除 Product

```kotlin
@Test
@Order(4)
fun `delete product`() {
    client.delete()
        .uri("/product/1")
        .exchange()
        .expectStatus().isOk()
}

@Test
@Order(5)
fun `deleted product is not exist`() {
    client.get()
        .uri("/product/1")
        .accept(APPLICATION_JSON)
        .exchange()
        .expectStatus().isNotFound()
}
```
