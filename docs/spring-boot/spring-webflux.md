gradle dependency

```gradle
implementation('org.springframework.boot:spring-boot-starter-webflux')
```

- Spring WebFlux 預設使用 Netty 作為 Web 伺服器

??? info "使用 tomcat 作為預設 Web 伺服器"

    ```gradle
    implementation('org.springframework.boot:spring-boot-starter-webflux')
    {
        exclude group: 'org.springframework.boot', module: 'spring-boot-starter-reactor-netty'
    }
    implementation('org.springframework.boot:spring-boot-starter-tomcat')
    ```

    啟動 Server 應會看到 log

    ```log
    2023-03-09 18:23:44.182 INFO 17648 --- [ main] o.s.b.w.embedded.tomcat.TomcatWebServer : Tomcat initialized with port(s): 8080 (http)
    ```

預設的 port 為 8080，以下範例更改 port 為 7001:

```yaml title="application.yaml"
server.port: 7001
```

## Rest Controller

```java hl_lines="1 4"
@RestController
public class MyRestService {

  @GetMapping(value = "/my-resource", produces = "application/json")
  List<Resource> listResources() {
    // ...
  }

  // ...
 }
```

- `@RestController`
    - 標註類別為一個 RESTful Web Service 的 Controller，負責處理 HTTP 請求並返回資料（通常為 JSON 或 XML）。
    - 等同於同時使用 `@Controller` 和 `@ResponseBody`。
- `@GetMapping`
    - 定義一個 GET 方法對應到 **/my-resource** 的路徑。
    - `produces = "application/json"` 指定回傳內容類型為 JSON。

在 Server 啟動後，連接到網址可以取得 `List<Resource>` 的 JSON 資料:
**http://localhost:8080/my-resource**
