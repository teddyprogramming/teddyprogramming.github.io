```java
@SpringBootApplication
public class MyApplication {
  public static void main(String[] args) {
    SpringApplication.run(MyApplication.class, args);
  }
}
```

`@SpringBootApplication` 的功能：

1. Component Scanning
   自動掃描標註類別所在套件及其子套件中的 Spring components（例如 `@Component`, `@Service,` `@Repository`, `@Controller`）與 configuration 類別（例如 `@Configuration`）。
2. Configuration Class
   將標註的類別本身視為一個 Spring 的 configuration 類別（等同於標註 `@Configuration`）。
3. 啟用 Auto-configuration
   啟用 Spring Boot 的自動配置（Auto-Configuration），自動載入依賴關聯的預設配置，簡化手動設定。

`@SpringBootApplication` 其實是 `@Configuration`, `@EnableAutoConfiguration` 和 `@ComponentScan` 的組合註解，方便開發者一次性啟用相關功能。

## Component Scanning

```java hl_lines="1"
@Component
public class MyComponentImpl implements MyComponent {
  // ...
}
```

- 使用 `@Component` 標註的類別會被 Spring 容器管理，成為一個 Spring Component。
- 該類別可以透過自動注入（automatically injected，或稱 auto-wired)，供其他需要的類別使用。

```java hl_lines="4"
public class AnotherComponent {
  private final MyComponent myComponent;

  public AnotherComponent(MyComponent myComponent) {
    this.myComponent = myComponent;
  }

  // ...
}
```

!!! info "優先考慮 constructor injection 比起 field 或 setter injection 可以確保 component 的 immutable。"

可以使用 `@ComponentScan` 指定額外要額外掃瞄的套件:

```java hl_lines="4"
package se.magnus.myapp;

@SpringBootApplication
@ComponentScan({"se.magnus.myapp", "se.magnus.util" })
public class MyApplication {
  // ..
}
```

- 如此設定後，Spring 容器會自動掃描並管理 `se.magnus.myapp` 與 `se.magnus.util` 套件下的所有 Spring Components（如 `@Component`, `@Service`, `@Repository` 等）。
- 這些被掃描到的 components 就可以透過 `@Autowired` 自動注入到其他類別中使用。

## Configuration

可以在類別上使用 `@Configuration` 註解，以定義或覆寫 Spring Boot 的預設設定。

```java hl_lines="1 4-8"
@Configuration
public class MyCustomConfig {

  @Bean
  public MyService myService() {
    // 自訂一個 MyService Bean，覆寫預設設定
    return new MyCustomServiceImpl();
  }
}
```

- `@Configuration` 標註的類別表示這是一個 Spring 的設定類別。
- Bean 方法會定義並註冊一個 Bean（例如 `MyService`）到 Spring 容器中。
- 如果有同名的 Bean 已存在（例如由 Auto-Configuration 提供的），這個自訂的設定將會覆寫預設的定義。
