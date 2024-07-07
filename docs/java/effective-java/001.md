# Item 1: Consider static factory methods instead of constructors

## 優點

偏好 static factory method 的優點：

1. 可以提供有意義的名稱，幫助閱讀理解的名稱

    - `BigInteger.probablePrime` 比 `BigInteger(int, int, Random)` 更容易理解。
    - constructor 之間只能夠透過參數型別不同來加以區隔，開發人員需要知道這些不同 constructor 對應的實作。但是，他們也可能會記錯而誤用。例如，我們有辦法區分 `BigInteger(int[])`, `BigInteger(int[], int)`, `BigInteger(int, int[])` 他們的不同嗎？

2. 不必每次都建立新物件

    - 套用 Flyweight pattern, Singleton pattern
    - 可以避免建立不必要重複的物件
    - Instance-controlled class

3. 可以回傳 Subclass

    - interface-based frameworks
    - [conceptual weight](https://en.wikiversity.org/wiki/Software_Design/Interface_size): 開發人員需要掌握多少概念才能夠使用 API

    !!! info
        Java 8 以前，interface 不能有 static method。在實作上，需要 static factory methods 的 class 會命名成 Type + s，並將 constructor 宣告成 private，例如 `Collections`, `Arrays`。

4. 可以依據情況回傳不同的 Subclass

    - 允許不同的 release 更換 return 的 subclass。
    - `EnumSet` 的實作，根據 enum 數量的不同，回傳不同的 subclass。回傳的 subclass 與條件是可以改變而不影響呼叫端的。

        ```java
        // 示意程式碼
        class EnumSet {
           static <E extends Enum<E>> EnumSet<E> noneOf(Class<E> elementType) {
                if (elementType.getEnumCount() <= 64)
                    return new RegularEnumSet(..)
                else
                    return new JumboEnumSet(..)
           }
        }
        ```
      
        - 條件 (`<= 64`) 與 subclass 的實作是可以改變的，而不影響呼叫端的程式碼。

5. 在撰寫程式碼時，類別不必存在

    - 利於 service provider frameworks
        - 框架開發商不必提前知道所有 interface 可能的實作
        - 讓 interface 實作延遲到 development time 決定
        - 例如: JDBC API 的 Connection interface，開發人員可以在開發時在決定具體的 Connection 實作，像是透過 property 設定 Connection 是 MySql `spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver`
    - Bridge pattern
    - 可以配合 Dependency injection frameworks 使用
    - `ServiceLoader`

## 限制

偏好 static factory method 的限制：

1. **拋棄繼承**:

    - constructor 被宣告成 `private` / `protected`，使得 subclass 無法初始化 superclass，造成無法繼承。
    - 無法繼承意味著放棄 polymorphism 的特性。

    !!! question
        如果同時讓 class 保有 static factory methods 並且把 constructor 宣告成 `public` 呢?

2. **增加 debug 難度**:

    - 因為建立物件的動作隱藏在 static factory methods 裡，所以 debug 時無法直接看到建立物件的地方，以及怎麼被建立的。
    - 以下幾種命名規則，可以幫助 static factory methods 的理解: (此為列舉而非窮舉)
         - `from`
              - type-conversion method: 從 **一個** type 的物件轉換成另一個 type 的物件
              - 範例:
                  - `Date.from(instant)`
         - `of`
              - aggregation method: 從 **多個** 參數的物件合併成一個 type 的物件
              - 範例:
                  - `EnumSet.of(JACK, QUEEN, KING)`
                  - `LocalDate.of(2024, 7, 4)`
         - `valueOf`
              - 比 `from` 與 `of` 更囉唆一點的版本
              - 範例:
                  - `Integer.valueOf("100")` 
                  - `Integer.valueOf("100", 2)`
         - `instance` / `getInstance`
              - 從 **0** 至 **多個** 參數建立物件
              - 雖然說是 `getInstance`，但是不保證回傳的是同一個物件
              - 範例:
                  - `Calendar.getInstance()` 
                  - `Calendar.getInstance(TimeZone.getDefault())` 
         - `create` / `newInstance`
             - 從 **0** 至 **多個** 參數建立物件
             - 每次都會建立新的物件
             - 範例:
                 - `Array.newInstance(Integer.class, 10)`
         - `getType`
             - 回傳的物件是 `Type` 的 instance
             - 雖然為 get，但是不保證回傳的是同一個物件
             - 範例:
                 - `Files.getFileStore(path)`
         - `newType`
             - 回傳的物件是 `Type` 的 instance
             - 每次都會建立新的物件
             - 範例:
                 - `BufferedReader br = Buffer.newBufferedReader(path)`
         - `type`
             - `getType` 與 `newType` 的簡化版
             - 範例:
                 - `Collections.singletonList("apple")`
 