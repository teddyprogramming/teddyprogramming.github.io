
## 打開 Springdoc 的功能

https://springdoc.org/

幫 product-composite-service 新增 opanapi dependencies

```gradle
implementation("org.springdoc:springdoc-openapi-starter-webflux-api:2.8.4")
implementation("org.springdoc:springdoc-openapi-starter-webflux-ui:2.8.4")
```

啟動 service 後，透過 http://localhost:8080/swagger-ui.html 存取 Swagger UI。

## 自定義 Swagger UI 內容

使用 `OpenAPI` bean 定義 Swagger UI 內容:

- title: `Product Composite Service API`
- contact:
    - name: YOUR NAME
    - url: YOUR WEBSITE
    - email: YOUR EMAIL

## 自定義 Swagger UI 的路徑

- `springdoc.swagger-ui.path=/openapi/swagger-ui.html`
- `springdoc.api-doc.path=/openapi/v3/api-docs`

## 設定 Swagger UI 掃描的位址

- `springdoc.packageToScan=se.magnus.microservices.composite.product`
- `springdoc.pathToMatch=/**`

## Adding API-specific documentation to the ProductCompositeService interface

- `@Tag` 標記在 Controller class 提供 `name` 與 `description`
- `@Operation` 標記在 API method 上，提供以下欄位:
    - `summary`
    - `description`

        ??? tip

            ```kotlin
            @Operation(
                summary = "Returns a composite view of the specified product id",
                description = """
            If the requested product id is found the method will return information regarding:
            1. Base product information
            1. Reviews
            1. Recommendation
            1. Service Addresses<br>(technical information regarding the addresses of the microservices that created the response)
            # Expected partial and error responses
            In the following cases, only a partial response be created (used to simplify testing of error conditions)
            ## Product id 113
            200 - Ok, but no recommendations will be returned
            ## Product id 213
            200 - Ok, but no reviews will be returned
            ## Non-numerical product id
            400 - A **Bad Request** error will be returned
            ## Product id 13
            404 - A **Not Found** error will be returned
            ## Negative product ids
            422 - An **Unprocessable Entity** error will be returned
                """
            )
            ```

- `@ApiResponses` 描述 http code 的 response

    ??? tip "程式碼"

        ```kotlin
        @ApiResponses(
            ApiResponse(responseCode = "200", description = "OK"),
            ApiResponse(
                responseCode = "400",
                description = "Bad Request, invalid format of the request. See response message for more information"
            ),
            ApiResponse(
                responseCode = "404",
                description = "Not found, the specified id does not exist"
            ),
            ApiResponse(
                responseCode = "422",
                description = "Unprocessable entity, input parameters caused the processing to fail. See response message for more information"
            ),
        )
        ```

    ??? note "`@ControllerAdvice` 中處理的例外必須繼承 `Exception`，才會被 render 到 api response。"

        - 標記 `@ResponseStatus`
        - 標記 `@ExceptionHandler` 例外必須繼承 `Exception`，繼承 `Throwable` 會無法呈現。
            -  例外需要滿足條件才會被列入考慮
                - [GenericResponseService#L721](https://github.com/springdoc/springdoc-openapi/blob/a9f338c2740cb32f9372d2e3f3ca6f97f6492549/springdoc-openapi-starter-common/src/main/java/org/springdoc/core/service/GenericResponseService.java#L721)
                - [GenericResponseService#L827](https://github.com/springdoc/springdoc-openapi/blob/a9f338c2740cb32f9372d2e3f3ca6f97f6492549/springdoc-openapi-starter-common/src/main/java/org/springdoc/core/service/GenericResponseService.java#L829)

## Try it out

- 使用 Swagger 打 API 並取得結果。
    - `200`, `400`, `404`, `422` 的結果
