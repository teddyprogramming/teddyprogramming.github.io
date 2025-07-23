# SRP (Single Responsibility Principle)

> A module should be responsible to one, and only one, actor.

## 症狀: Accidental duplication

![](srp-1.drawio)

`Employee` 分別服務三個角色:

- `calculatePay` 服務 CFO (Chief Financial Officer)
- `reportHours` 服務 COO (Chief Operation Officer)
- `save` 服務 CTO (Chief Technology Officer)

三個角色相依同一個模組將導致，不同角色相依的行為相互影響。

舉例來說，`calculatePay` 與 `reportHours` 共用非加班的工時計算演算法，命名為 `regularHours`。

![](srp-2.drawio)

假定，CFO 需要對非加班的工時計算進行調整，開發人員對 `regularHours` 進行調整，不過他沒留意 COO 相依的 `reportHours` 也使用到這個方法。最終，CFO 在確認功能修改符合預期後上線了，然而 COO 的功能卻被破壞了。

COO 與 CFO 需要計算非加班的工時演算法，它們要的計算邏輯類似但又不完全一樣，通常就會複製演算法然後個別修改，造成 accidental duplicate 的結果。

## 參考

- Clean Architecture Chapter 7
