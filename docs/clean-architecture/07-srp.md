# SRP (Single Responsibility Principle)

> A module should be responsible to one, and only one, actor.

## 情境

在軟體開發過程中，我們經常會建立一些核心的業務物件或模組來處理多種相關的功能。這些物件通常代表重要的業務概念(如員工、訂單、產品等)，而不同的 actors 會對同一個物件有不同的需求和關注點。

例如，在一個企業系統中：

- 人力資源部門關心員工的基本資料和績效管理
- 財務部門關心員工的薪資計算和成本分析
- 資訊技術部門關心資料的儲存和系統整合
- 營運部門關心員工的工時統計和生產力分析

當我們嘗試用單一的類別或模組來滿足所有這些不同部門的需求時，就容易產生職責混雜的問題。表面上看起來這樣的設計很合理，因為所有功能都圍繞著同一個核心概念，但實際上卻會帶來許多意想不到的困難。

## 問題

**核心衝突**: 當一個模組需要服務多個不同的 actors 時，如何在滿足所有需求的同時，避免不同需求間的相互干擾？

在企業系統中，核心業務物件(如 `Employee`)往往需要支援多個部門的不同需求。表面上看，將所有相關功能集中在一個類別中似乎很合理，但這會帶來以下困境：

**相互衝突的力量(Forces)**:

- **內聚性(Cohesion)** vs **多重職責**: 我們希望相關功能聚集在一起，但多個職責會降低內聚性
- **重用性** vs **獨立性**: 我們想要重用共同的邏輯，但不同 actors 的需求可能需要獨立演化
- **簡單性** vs **彈性**: 單一類別看似簡單，但會犧牲對變化的回應彈性
    - *簡單*: 所有員工相關功能都在一個 `Employee` 類別中，開發者只需要知道一個地方就能處理所有員工相關的需求
    - *彈性*: 當 CFO 要求改變薪資計算時，必須修改 `Employee` 類別，但這個修改可能意外影響 COO 的工時報表或 CTO 的資料儲存；任何部門的需求變更都需要重新測試整個類別的所有功能；多個開發者無法同時修改不同部門的需求(檔案衝突)
- **一致性** vs **客製化**: 我們希望維持一致的介面，但不同 actors 需要客製化的行為

```kroki-plantuml
allowmixing
hide circle

class Employee {
    calculatePay
    reportHours
    save
}

actor CTO
actor COO
actor CFO

CFO ..> Employee
COO ..> Employee
CTO ..> Employee
```

**問題的表現**:

以下的 `Employee` 類別同時承擔三個不同角色的需求:

- `calculatePay` 方法負責支援 CFO(財務長)的薪資計算
- `reportHours` 方法負責支援 COO(營運長)的工時報表
- `save` 方法負責支援 CTO(技術長)的資料儲存

這種設計讓 `Employee` 模組的職責混雜，當不同角色的需求變動時，彼此之間容易產生不預期的影響，降低了系統的可維護性與穩定性。

### 症狀

當一個模組承擔多重職責時，會出現以下症狀：

#### 1. Accidental duplication

舉例來說，`calculatePay` 與 `reportHours` 這兩個方法都依賴同一個名為 `regularHours` 的方法來計算非加班工時。假設 CFO(財務長)提出需求，要求調整非加班工時的計算方式，開發人員便修改了 `regularHours`。但如果他沒有注意到 COO(營運長)所依賴的 `reportHours` 也會用到這個方法，這次修改就可能在不經意間影響到 COO 的功能。結果，雖然 CFO 的需求被滿足並順利上線，COO 的功能卻因此出現錯誤。

```kroki-plantuml
hide circle
hide members

object reportHours
object calculatePay
object regularHours

reportHours --> regularHours
calculatePay --> regularHours
```

這種情況會產生典型的風險：當一個模組同時承擔多個角色的需求時，對其中一個職責的修改可能會意外影響到其他職責，導致 bug 發生。

雖然 CFO 和 COO 都需要計算非加班工時，但這僅僅是表面上的相似。實際上，兩個角色對於工時計算的需求和定義可能存在差異。若強行讓他們共用同一個 `regularHours` 方法，便可能產生「意外的重複」(accidental duplication): 將不同職責下的需求誤認為可共用的邏輯，導致未來當其中一方需求變動時，另一方也受到影響。

!!! note "意外的重複 一詞"

    **意外**: 開發者可能誤以為這是合理的程式碼重用，實際上卻因為忽略了不同職責間需求的差異，導致錯誤的耦合與潛在風險，因此稱為「意外」。這種情況下，原本應該獨立演化的邏輯被不當共用，當某一方需求變動時，另一方也會受到影響。

    **重複**: 這裡的重複並不是指單純的程式碼重複，而是指不同職責下，原本應該獨立演化的概念或邏輯被誤認為可以共用，導致它們在實作上被重複利用。這種重複會讓後續的需求變更產生不必要的牽連與風險，降低系統的彈性與可維護性。

    補充說明: 解決這類問題不一定要產生大量相似但略有差異的程式碼，更重要的是思考如何用更貼近業務語意的 domain model 來表達需求。例如在本例中，單純用一個整數來表示工時可能不足以覆蓋不同角色的需求，這時可以考慮設計一個如 `WorkRecord` 的模型，讓各角色根據自身需求定義專屬的行為與屬性，從而降低耦合、提升可維護性。

#### 2. Merges

當兩個不同的開發者因為不同的原因修改同一個 `Employee` 類別時，很可能會產生合併衝突(merge conflicts)。

舉例來說:
- 一位開發者為了 CFO，修改了 `calculatePay` 方法
- 另一位開發者為了 COO，修改了 `reportHours` 方法

即使這兩個修改從業務角度來看是完全無關的，但因為它們位於同一個原始檔案中，在版本控制系統中合併時就可能發生衝突。更糟糕的是，即使合併在機械層面上成功進行，兩個變更也可能在語意上不相容。

比如說，兩個修改都需要變更某個共用的演算法或變數，這時候就會產生問題。

舉個具體例子：

假設 `Employee` 類別中有一個共用的 `workingDays` 變數：

```java
public class Employee {
    private int workingDays = 22; // 每月工作天數

    // CFO 關心的薪資計算
    public Money calculatePay() {
        return baseSalary.divide(workingDays);
    }

    // COO 關心的工時報表
    public String reportHours() {
        return "Average hours per day: " + (totalHours / workingDays);
    }
}
```

現在兩個開發者同時進行修改:

- **開發者 A**: 為了 CFO 的需求，將 `workingDays` 改為 20(扣除國定假日)
- **開發者 B**: 為了 COO 的需求，在 `reportHours` 中將 `workingDays` 改為 `actualWorkingDays`(扣除請假天數)

當這兩個修改合併時，雖然程式碼可以成功合併且沒有編譯錯誤，但現在 CFO 的薪資計算可能使用了錯誤的工作天數，或 COO 的報表使用了錯誤的基準。

這種問題往往很難被發現，因為:

- 程式碼可以正常編譯和執行
- 單元測試可能都會通過(如果測試不夠全面)
- 錯誤可能在實際使用時才逐漸浮現，而非立即暴露

!!! example "錯誤逐漸浮現的過程"

    承接上面的例子，假設合併後 `workingDays` 變成了 20，這會影響薪資計算。但這個錯誤可能不會立即被發現:

    - **第一個月**: 新的薪資計算開始使用，但因為差異不大(22 vs 20 天)，可能沒有人注意到
    - **第二個月**: 一些員工開始覺得薪水好像有點不對，但不確定
    - **第三個月**: 財務部門在對帳時發現數字有些異常
    - **第四個月**: 問題才被正式發現和確認

    在這段期間內，錯誤的薪資計算一直在進行，可能已經影響了數百名員工的薪水，需要大量的時間和成本來修正。

    相對地，如果是明顯的錯誤(比如程式當機)，通常在第一次執行時就會被發現。

這種合併問題會導致:

1. **開發效率降低**: 原本可以平行開發的功能，現在需要協調合併
2. **增加錯誤風險**: 合併過程中可能引入新的 bug
3. **測試負擔加重**: 每次合併後都需要重新測試所有相關功能

## 解決方案

**核心策略**: 將原本混雜在一個模組中的多重職責分離，讓每個模組只對一個 actor 負責。

問題的解決方案很多，最直覺的方法是將資料與函式分離:

```kroki-plantuml
hide circle
hide empty members

class PayCalculator {
    calculatePay()
}

class HourReporter {
    reportHours()
}

class EmployeeSaver {
    saveEmployee()
}

class EmployeeData {
}

PayCalculator --> EmployeeData
HourReporter --> EmployeeData
EmployeeSaver --> EmployeeData
```

這種設計將原本的 `Employee` 類別分解為:

- **EmployeeData**: 純粹的資料結構，沒有行為邏輯
- **PayCalculator**: 專門負責 CFO 相關的薪資計算邏輯
- **HourReporter**: 專門負責 COO 相關的工時報表邏輯
- **EmployeeSaver**: 專門負責 CTO 相關的資料儲存邏輯

每個類別都只對一個 actor 負責，當某個部門的需求變更時，只需要修改對應的類別，不會影響到其他部門的功能。

**可選的 Facade 模式**：有些開發人員喜歡使用 Facade 模式來簡化多個類別的管理。這樣可以保持原本的簡單介面，同時在底層維持職責分離：

```kroki-plantuml
hide circle
hide empty members

class EmployeeFacade {
    calculatePay()
    reportHours()
    save()
}

class PayCalculator {
    calculatePay()
}

class HourReporter {
    reportHours()
}

class EmployeeSaver {
    saveEmployee()
}

class EmployeeData {
}

EmployeeFacade --> PayCalculator
EmployeeFacade --> HourReporter
EmployeeFacade --> EmployeeSaver
PayCalculator --> EmployeeData
HourReporter --> EmployeeData
EmployeeSaver --> EmployeeData
```

Facade 提供了與原本 `Employee` 類別相同的公開方法，讓既有的客戶端代碼不需要修改就能繼續使用，但需要注意不要讓它重新引入職責混雜的問題。理想的 Facade 應該是一層薄薄的介面，僅負責將呼叫委派給對應的職責類別，而不包含任何業務邏輯。

**另一種實作方式**：有些開發人員偏好將資料與使用該資料的方法放得更近，他們會讓 `Employee` 類別保留資料和公開方法，但將實際的業務邏輯委派給專門的服務物件：

```kroki-plantuml
allowmixing
hide circle
hide empty members

class Employee {
    -employeeData
    calculatePay()
    reportHours()
    save()
}

class PayCalculator {
    calculatePay(employee)
}

class HourReporter {
    reportHours(employee)
}

class EmployeeSaver {
    saveEmployee(employee)
}

Employee --> PayCalculator
Employee --> HourReporter
Employee --> EmployeeSaver

actor CFO
actor COO
actor CTO

CFO ..> Employee
COO ..> Employee
CTO ..> Employee
```

在這種設計中，`Employee` 類別仍然作為資料的載體和統一的介面，但實際的業務邏輯都委派給對應的專門類別。這樣既保持了資料與方法的接近性，又確保了每個業務邏輯只對一個 actor 負責。

### 結果

應用 SRP 後，系統會獲得以下改善：

#### 解決原有問題

**1. 消除 Accidental duplication**

- 不同 actors 的邏輯分離在各自的類別中，不再共用可能語意不同的方法
- CFO 的薪資計算邏輯在 `PayCalculator` 中，COO 的工時報表邏輯在 `HourReporter` 中
- 各自的需求變更不會意外影響到其他角色

**2. 減少 Merge conflicts**

- **大幅降低衝突頻率**: 不同部門的需求修改發生在不同的檔案中，直接業務邏輯的衝突大幅減少
- **平行開發**: 開發者可以同時修改不同角色的需求，不會互相干擾
- **衝突範圍更明確**: 即使發生衝突(如共用依賴、介面變更、建構設定等)，影響範圍更容易識別和解決
- **更安全的合併**: 即使需要手動解決衝突，也不用擔心意外破壞其他角色的業務邏輯

#### 帶來的好處

**開發效率提升**

- **平行開發**: 多個開發者可以同時修改不同角色的需求，不會互相干擾
- **更快的編譯與測試**: 只需要重新編譯和測試受影響的特定模組
- **更容易的程式碼審查**: 每次修改的範圍更明確，審查更聚焦

**維護性改善**

- **單一變更原因**: 每個類別只會因為一個角色的需求而改變
- **更容易理解**: 每個類別的職責明確，新進開發者更容易理解
- **更安全的修改**: 修改某個功能時，不用擔心意外破壞其他功能

**測試品質提升**

- **獨立測試**: 每個職責類別可以獨立進行單元測試
- **更精確的測試**: 測試範圍明確，更容易設計有效的測試案例
- **更快的測試回饋**: 不需要每次都測試整個巨大的類別

#### 可能的新挑戰

**複雜度增加**

- 類別數量增加，需要管理更多的元件
- 可能需要額外的依賴注入或 factory 來組織物件

**學習成本**

- 團隊需要適應新的架構模式
- 需要建立新的開發和測試慣例

**過度設計風險**

- 在簡單的場景中可能造成不必要的複雜性
- 需要根據實際情況權衡是否值得重構

整體而言，SRP 的應用通常能提升系統的可維護性、可測試性和開發效率。雖然會帶來一些新的複雜性，但在多數中大型系統中，這些好處往往超越其成本。
