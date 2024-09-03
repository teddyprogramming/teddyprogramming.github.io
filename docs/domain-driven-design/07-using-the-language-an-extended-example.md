# Ch 7: Using the Language: An Extended Example

## Cargo Shipping System (貨物運送系統)

我們要替一家貨運公司開發軟體，最初的需求如下:

1. 追蹤貨物的處理狀態
2. 預約寄送貨物
3. 當貨物抵達某個地方時，自動寄送發票給客戶

用來表示 Domain Model 的 class diagram 如下:

![](07/01.png)

透過 Domain Model 我們可能使用以下的描述:

- `Cargo` (貨物) 涉及多個 `Customer` (客戶)，每一個 `Customer` 扮演著不同的 `Role` (角色)。
- `Cargo` (貨物) 的 `Goal` (目標) 已指定。
- 透過一組滿足 `Specification` (規格) 的 `Carrier Movement` (運輸動作) 將達成 `Delivery Goal` (運送目標)。

圖中的每個物件的意義:

- `Handling Event` (處理事件): 描述對 `Cargo` 採取的處理，像是
    - loading it onto a ship (將貨物裝上船)
    - clearing it through customs (將貨物通過海關檢查並獲得許可)
    - loading (裝貨)
    - unloading (卸貨)
    - being claimed by the receiver (被收貨人提走)
- `Delivery Specification` (運送規格): 描述 `delivery goal` (運送目標)，包含了 `destination` (目的地) 與 `arrival time` (抵達時間)。
- `Customer`: `Role` (角色) 區隔了 `Customer` (客戶) 在運送扮演的身份。
    - `Role` 可以是 shipper (託運人), receiver (收貨人), payer (付款人) 等。
    - `Customer` 與 `Cargo` 的關係是「qualified (限定的) many-to-one」而非「many-to-many」。
- `Carrier Movement` (運輸行動): 描述 `Carrier` (如，卡車或船) 從一 `Location` (地點) 到另一 `Location` (地點) 的旅程。
    - ??? tip "看圖說明"
          ![](07/01-carrier-movement.png)
          `Cargo` 經過多個 `Handling Event` 處理，透過 `Carrier` 的 `Carrier Movement` 在 `Location` 之間移動。
- `Delivery History` (運送歷史紀錄): 描述 `Cargo` 在運送過程的。

Model 已經涵蓋實作需要的概念。假定我們有適當的機制保存物件與搜尋物件。

Model 的 refinement, design, implementation 是在迭代開發的過程中互相配合、同步進行的。也就是說，Model 不會是完全設計好，然後再交由下一個階段進行實作，而是應隨著開發不斷地發展、改進與調整。

這個範例從一個相對比較成熟的 Model 開始。並且，為了聚焦本章的重點，範例限制 Model 的修改動機必須是「為了使 Model 能與具體的實作相互關聯」，然後使用 building block patterns 進行修改 (即 Entity, Value Object, Aggregation, Repository 等)。

## Isolating the Domain: Introducing the Applications

使用 Layered Architecture 將 domain 與其他部分區隔。

以下是三個 application layer 的 class:

1. **Tracking Query**: 查詢 `Cargo` 的處理情況。
2. **Booking Application**: 註冊新的 `Cargo` 讓系統處理。
3. **Activity Logging Application**: 紀錄 `Cargo` 處理的事件。

Application layer 負責向 domain layer 問問題，domain layer 負責回答問題。

## 區分 Entity 與 Value Object

檢視每一個物件: (判斷方法: 可以共用的物件是 Value Object，不行的是 Entity)

- `Customer`
    - Entity
    - 唯一識別碼: customer ID
- `Cargo`
    - Entity
    - 唯一識別碼: tracking ID
- `Handling Event` 與 `Carrier Movement`
    - Entity
    - 唯一識別碼
        - `Carrier Movement`: schedule ID (從 shipping schedule 中的 code)
        - `Handling Event`: Cargo tracking ID + completion time + type
- `Location`
    - Entity
- `Delivery History`
    - Entity
    - 唯一識別碼: Cargo tracking ID (`Delivery History` 與 `Cargo` 一對一關聯，沒有自己的唯一識別碼。`Delivery History` 的識別碼來自 `Cargo`)
- `Delivery Specification`
    - Value Object (可以有兩個 `Cargo` 送往相同的地點，因此共用同一個 Specification)
- `Role` 與其他屬性
    - Value Object
    - 其他屬性，包含像是 time, name

## 關聯

指定關聯方向。圖中說明每個關聯方向的原因。

![](07/02.png)

- `Carrier Movement - Handling Event`
    - 如果要從 `Carrier` 追蹤貨物，就需要 `Carrier Movement -> Handling Event`，但是我們的業務不需要。
    - 業務需要追蹤 `Cargo` 的狀態，需要透過 `Cargo -> Delivery History -> Handling Event -> Carrier Movement -> Location` 知道目前 `Cargo` 的位址。

## Aggregate Boundaries

Aggregate root 是 Entity 且有自己的唯一識別碼: `Customer`, `Cargo`, `Carrier Movement`, `Location`

`Cargo` 的 aggregate 可以把所有因為 `Cargo` 而存在的事物劃入邊界中，包含 `Delivery History`, `Delivery Specification`, `Handling Event`。

`Handling Event` 最後被設計成自己為 aggregate，因為業務上有需要查詢 `Cargo` 處理狀態，需要透過 `Handling Event`。

![](07/03.png)

## Repository

只有 Aggregate root 會有 Repository。

![](07/04.png)

沙盤推演，這些 Repository 是否能滿足需求。

- **Booking Application**
    - 客戶 `Customer` (`Customer Repository`)
    - 預定貨物要運送到 `Location` (`Location Repository`)
- **Activity Logging Application**
    - 使用 `Carrier Movement Repository` 查詢要裝貨的 `Carrier Movement`。
    - 使用 `Cargo Repository` 紀錄已完成裝貨。

沒有 `Handling Event Repository`，在這次迭代中 `Handling Event` 是與 `Delivery History` 關聯產生集合 且 沒有查詢 `Handling Event` 的需求。
