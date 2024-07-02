# Quartz: Hello World

## 每 5 秒輸出 "Hello World!" 到畫面上

```plantuml
hide circle

class Scheduler

interface Job {
    execute(context): void
}

class HelloWorldJob implements Job

class JobDetail {
}

interface Trigger {
}


JobDetail -> Job
Trigger -> JobDetail
Scheduler -> "*" Trigger

object helloWorldTrigger
helloWorldTrigger .u.|> Trigger

object helloWorldJobDetail
helloWorldJobDetail .u.|> JobDetail

helloWorldTrigger -> helloWorldJobDetail
helloWorldJobDetail -> HelloWorldJob
```

當程式啟動時， Quartz Scheduler 會在背景執行，並在 `helloWorldTrigger` 定義的每 5 秒執行一次 `HelloWorldJob` 實作的 `execute` 輸出 "Hello World!" 到畫面上。

在 `build.gradle.kts` 中新增 `spring-boot-starter-quartz` 相依套件。如此，程式啟動後就會有一個 Quartz Scheduler 在背景執行。

```kotlin title="build.gradle.kts"
implementation("org.springframework.boot:spring-boot-starter-quartz")
```

接著，我們需要定義 `HelloWorldJob` 實作 `Job` 介面，實作輸出 "Hello World!" 的邏輯。

```java title="HelloWorldJob.java"
public class HelloWorldJob implements Job {
    @Override
    public void execute(JobExecutionContext jobExecutionContext) {
        System.out.println("Hello World!");
    }
}
```

然後，我們需要再新增兩個 Bean 定義 `JobDetail` 和 `Trigger`。其中，`JobDetail` 定義了 `HelloWorldJob` 的 metadata，而 `Trigger` 定義了 `JobDetail` 的觸發條件。

``` java title="QuartzConfig.java"
@Component
public class QuartzConfig {

    @Bean
    public JobDetail helloWorldJobDetail() {
        return JobBuilder.newJob(HelloWorldJob.class)
                         .withIdentity("helloWorldJob")
                         .storeDurably() // (1)
                         .build();
    }

    @Bean
    public Trigger helloWorldTrigger(JobDetail printHelloWorldJobDetail) {
        SimpleScheduleBuilder scheduleBuilder = SimpleScheduleBuilder
                .simpleSchedule()
                .withIntervalInSeconds(5)
                .repeatForever();
        return TriggerBuilder.newTrigger()
                             .forJob(printHelloWorldJobDetail)
                             .withIdentity("helloWorldTrigger")
                             .withSchedule(scheduleBuilder)
                             .build();
    }
}
```

1. `Trigger` 要執行的 `JobDetail` 必須要在 `JobStore` 中，設定 `storeDurably()` 來保證 `JobDetail` 在 `JobStore` 中。否則，`JobDetail` 會在沒有與 `Trigger` 關聯的情況下被移除。

最後，把程式跑起來後，每 5 秒就會在畫面上輸出 "Hello World!"。

## 每 5 秒輸出 "Hello {name}!" 到畫面上

這裡展示 `Job` 可以存取到 bean 的方式。為了方便展示，我們新增一個 `name` bean，並在 `HelloWorldJob` 中取得 `name` bean 的值。

```java title="HelloWorldJob.java"
@Bean
public String name() {
    return "Teddy";
}
```

修改 `HelloWorldJob` 的建構式接收 `name` 字串，這個變數的實體，會透過 DI 注入。

```java title="HelloWorldJob.java" hl_lines="2 4 5 6 10"
public class HelloWorldJob implements Job {
    private final String name;

    public HelloWorldJob(String name) {
        this.name = name;
    }

    @Override
    public void execute(JobExecutionContext jobExecutionContext) {
        System.out.println("Hello " + name + "!");
    }
}
```

最後，把程式跑起來後，每 5 秒就會在畫面上輸出 "Hello Teddy!"。

## 自動化測試

使用 [awaitility](https://github.com/awaitility/awaitility) 來實作需要等待時間的測試。首先，需要增加相依套件。

```kotlin title="build.gradle.kts"
testImplementation("org.awaitility:awaitility")
```

!!! tip "awaitility 在 Spring Boot 3.2 已受到相依控管，所以使用 3.2 以上的版本這裡不需要加版本號碼。"

使用 JUnit 5 進行測試。測試驗證 15 秒內，必須要有 "Hello World!" 字串輸出到 standard output 上。

```java
@SpringBootTest
class HelloWorldJobTest {
    private final PrintStream standardOut = System.out; // (1)!

    private final ByteArrayOutputStream outputStreamCaptor = new ByteArrayOutputStream(); // (2)!

    @BeforeEach
    public void setUp() {
        System.setOut(new PrintStream(outputStreamCaptor)); // (3)!
    }

    @AfterEach
    public void tearDown() {
        System.setOut(standardOut); // (4)!
    }

    @Test
    void helloWorld() {
    await().atMost(Duration.ofSeconds(15)) // (5)!
           .untilAsserted(() -> assertThat(outputStreamCaptor.toString()).contains("Hello World!")); // (6)!
    }
}
```

1. 先記下預設的 `System.out`，在測試結束後還原使用。
2. 建立一個 `ByteArrayOutputStream` 來取代 `System.out`，這樣可以把 `System.out` 的輸出捕捉到 `ByteArrayOutputStream` 中。
3. 在每一個測試開始前，把 `System.out` 設定為 `ByteArrayOutputStream`。
4. 在每一個測試結束後，把 `System.out` 還原為預設的 `System.out`。
5. 等待最多 15 秒。
6. 確認 standard output 輸出必須包含 "Hello World!"。

### 參考

- [Scheduling Tasks](https://spring.io/guides/gs/scheduling-tasks)
- [Unit Testing of `System.out.println()` with JUnit](https://www.baeldung.com/java-testing-system-out-println)