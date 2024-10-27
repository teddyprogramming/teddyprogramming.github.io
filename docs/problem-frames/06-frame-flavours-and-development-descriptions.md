![](img/06-01.svg)
圖: 抽象的 problem diagram

- 需求的描述會落在 c, d 的描述上。
- 部分的需求描述落在 c 的屬性。(c 需要滿足什麼屬性才能使需求滿足)
- 規格描述 machine 對於 a, b 應有的行為。
- 部分的規格描述落在 d 的屬性。(machine 對於 d 的屬性應做何者相應的動作)
- 最後，需要描述該如何利用 a, c 來實現需求。

Domain flavours:

- static flavours
- dynamic flavours
- control flavours
- informal flavours
- conceptual flavours

## Static flavours

### Physical static (物理的靜態) domain

=== "原文"

    > **Package router control**
    >
    > A package router is a large mechanical device used by postal and delivery organisations to sort packages into bins according to their destinations.
    >
    > The packages carry bar-coded labels. They move along a conveyor to a reading station where their package-ids and destinations are read. They then slide by gravity down pipes fitted with sensors at top and bottom. The pipes are connected by two-position switches that the computer can flip (when no package is present between the incoming and outgoing pipes). At the leaves of the tree of pipes are destination bins, corresponding to the bar-coded destinations.
    >
    > A package cannot overtake another either in a pipe or in a switch. Also, the pipes are bent near the sensors so that the sensors are guaranteed to detect each package separately. However, packages slide at unpredicatable speeds, and may get too close together to allow a switch to be set correctly. A misrouted package may be routed to any bin, an apporpriate message being displayed. There are control buttons by which an operator can command the controlling computer to stop and start the conveyor.
    >
    > The problem is to build the controlling computer to obey the operator's commands, to route packages to their destination bins by setting the switches appropriately, and to report misrouted packages.

=== "翻譯"

    > **包裹路由控制**
    >
    > 包裹路由器(package router)是郵政和快遞組織使用的大型機械設備，根據目的地將包裹(package)分類到不同的箱子(bin)中。
    >
    > 包裹上帶有條碼標籤。它們沿著輸送帶(conveyor)移動到讀取站(reading station)，那裡讀取它們的包裹 ID 和目的地。然後它們沿著安裝有頂部和底部感應器(sensor)的管道(pipe)滑動。管道由兩個位置開關連接，計算機可以翻轉(當進出管道之間沒有包裹時)。在管道樹的末端是目的地箱子，對應於條碼目的地。
    >
    > 包裹在管道或開關中不能超越另一個包裹。此外，管道在感應器附近彎曲，以便感應器保證分別檢測每個包裹。然而，包裹以不可預測的速度滑動，可能會彼此靠得太近，以致無法正確設置開關。錯誤路由的包裹可能被路由到任何箱子，並顯示相應的訊息。作業員可以通過控制按鈕命令控制電腦停止和啟動輸送帶。
    >
    > 問題是建置控制電腦，以遵循作業員的命令，通過適當設定開關將包裹路由到目的地箱子，並報告錯誤路由的包裹。

![](img/06-02.svg)

假設現在包裹在 sw3，然後目的地是 bin 4，那麼該往左還是往右？

![](img/06-03.svg)

要回答這個問題，我們需要有 pipes, switches, bins 的配置圖，他們形成 static domain。

### Structural flavours

package router layout 是一個樹狀結構。reading station 是 root note。switches 是 interior nodes (內部節點)。bins 是 leaves。pipe 是 arc (弧) 連接兩個非 root 的 node。switch 出口只有兩個方向，因此結構為二元樹狀結構。
