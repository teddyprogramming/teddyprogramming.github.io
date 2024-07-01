# Quartz: JDBC JobStore

## 簡單的 Job 範例

以下範例展示，在 Spring Boot 中如何設定 Quartz 使用 JDBC JobStore。如此一來，我們把 Job 放到 cluster 上執行。

在 `build.gradle.kts` 中新增 `spring-boot-starter-quartz` 相依套件。如此，程式啟動後就會有一個 Quartz Scheduler 在背景執行。

```kotlin title="build.gradle.kts"
implementation("org.springframework.boot:spring-boot-starter-quartz")
```

使用 docker compose 執行 mysql。

```yaml title="compose.yml"
version: '3.1'

services:
  database:
    image: mysql:8.4.0
    ports:
      - "3306:3306"
    environment:
      MYSQL_ALLOW_EMPTY_PASSWORD: true
      MYSQL_DATABASE: quartz
```

設定 spring boot 的 properties。

```yaml title="src/main/resources/application.yml"
spring:
  quartz:
    job-store-type: jdbc #(1)!
    jdbc:
      initialize-schema: always #(2)!
    properties:
      org:
        quartz:
          scheduler:
            instanceName: scheduler
            instanceId: AUTO #(3)!
          dataSource:
            quartzDataSource: #(4)!
              driver: com.mysql.cj.jdbc.Driver
              URL: jdbc:mysql://localhost:3306/quartz
              user: root
              password:
              provider: hikaricp
          jobStore:
            class: org.quartz.impl.jdbcjobstore.JobStoreTX
            dataSource: quartzDataSource #(5)!
            isClustered: true #(6)!
```

1. 使用 JDBC JobStore。
2. 每次啟動都會初始化 schema。換句話說，我們可以不必手動新增 quartz 相關的 tables。([SQL 參考](https://github.com/quartz-scheduler/quartz/tree/main/quartz/src/main/resources/org/quartz/impl/jdbcjobstore))
3. 在 cluster 上的 pod 需要有唯一的 instance id。這裡設定 `AUTO` 即可達到需求。
4. data source 的設定。這裡的名稱會在下面設定 job store 的 dataSource 參考。
5. job store 使用的 dataSource。
6. 啟用 cluster。

新增 `Job` 實作，簡單的在畫面輸出現在的時間以及 "Hello World!"。印出時間是為了幫助我們識別這是哪個時間點所觸發的 Job。

```java title="HelloWorldJob.java"
public class HelloWorldJob implements Job {
    @Override
    public void execute(JobExecutionContext jobExecutionContext) {
        String time = LocalDateTime.now().format(DateTimeFormatter.ofPattern("hh:mm:ss"));
        System.out.println(time + " Hello World!");
    }
}
```

新增 `JobDetail` 和 `Trigger` 的 Bean 定義。這裡設定每 10 秒觸發一次 Job 在畫面上輸出 "Hello World!"。

```java
@Component
public class QuartzConfig {

    @Bean
    public JobDetail helloWorldJobDetail() {
        return JobBuilder.newJob(HelloWorldJob.class)
                         .withIdentity("helloWorldJob")
                         .storeDurably()
                         .build();
    }

    @Bean
    public Trigger helloWorldTrigger(JobDetail printHelloWorldJobDetail) {
        return TriggerBuilder.newTrigger()
                             .forJob(printHelloWorldJobDetail)
                             .withIdentity("helloWorldTrigger")
                             .withSchedule(CronScheduleBuilder.cronSchedule("*/10 * * * * ?"))
                             .build();
    }
}
```

為了可以同時執行多個 instance，我們將程式打包成 tar 檔。專案的目錄 /build/libs 應該會產生 xxx.jar 檔。

```shell
./gradlew bootJar
```

打開 3 個 terminal，分別執行以下指令執行程式: (xxx 替換成實際檔案名稱)

```shell
java -jar build/libs/xxx.jar
```

觀察每 10 秒只會有一個 terminal 輸出 "Hello World!"。這表示 Job 是在 cluster 上執行的。將輸出訊息的 terminal 關閉，可以觀察到其他 terminal 會接續執行輸出 "Hello World!" 的 Job。

## 參考

- [Sprint Boot Quartz Scheduler](https://docs.spring.io/spring-boot/reference/io/quartz.html)
- [Quartz Configuration - Configure Clustering with JDBC-JobStore](https://www.quartz-scheduler.org/documentation/quartz-2.2.2/configuration/ConfigJDBCJobStoreClustering.html) 
 
 