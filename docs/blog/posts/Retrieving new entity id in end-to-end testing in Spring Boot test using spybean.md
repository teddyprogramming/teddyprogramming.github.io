---
categories:
    - Spring Boot
    - Testing
date: 2024-07-15
---

# Retrieving new entity id in integration testing in Spring Boot test using `@SpyBean`

在 integration test 中，常會涉及將資料寫入資料庫，然後再透過其他 API 將資料撈出來以確認資料確實寫入資料庫。

```java
@SpringBootTest
@AutoConfigureMockMvc
public class IntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    public void test() throws Exception {
        mockMvc.perform(post("/data").content("Hello world!")) // (1)!
               .andExpect(status().isOk());

        // 透過另外一個 API 撈資料，確認
    }
}
```

1. 假設會在 `data` 資料庫新增一筆資料 "Hello world!"

但是，有時候可能因為種種原因，可能是沒有這種撈資料的 API 可用，亦或者測試設計，不摻雜其他動作刻意為之等等。

總之，沒有撈資料 API 的情況下，要確認資料是否正確新增到資料庫，有一種做法是直接確認資料確實寫進資料庫。不過，直接撈資料庫最後一筆資料，然後進行驗證的做法會有缺陷，我們如何確定最後一筆資料一定是剛才 API 寫入的資料呢？如果我們在不同測試共用相同的測試資料，就更難以確定了。我們可以使用 `@SpyBean` 來解決這個問題，確認 repository 寫入資料庫的 method 確實有被呼叫，並取得方才新增資料的 id。

```java hl_lines="8-9 16-20"
@SpringBootTest
@AutoConfigureMockMvc
public class IntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @SpyBean // (1)!
    private DataRepository dataRepository;

    @Test
    public void test() throws Exception {
        mockMvc.perform(post("/data").content("Hello world!"))
               .andExpect(status().isOk());

        ArgumentCaptor<Data> dataCaptor = ArgumentCaptor.forClass(Data.class);
        verify(dataRepository).save(dataCaptor.capture()); // (2)!
        Integer caseId = savedEntity.getValue().getId(); // (3)!
        Data actual = dataRepository.findById(caseId); // (4)!
        assertThat(actual).isEqualTo(new Data(caseId, "Hello world!")); // (5)!
    }
}
```

1. 標記 `@SpyBean`
2. 確認 `dataRepository.save` method 有被呼叫
3. 取得新增資料的 id
4. 透過 id 取得新增資料
5. 驗證新增資料是否正確

Mockito 的 `spy` 允許我們使用實際 object 的呼叫，並且在需要的地方做部分的 mock。`@SpyBean` 就是幫我們基於實際的 bean 產生 spy bean，讓我們可以在測試中使用實際的 repository，並且在需要的地方做部分的 mock。
