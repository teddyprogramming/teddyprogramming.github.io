# State

Terraform 需要儲存有關你所管理基礎架構和設定的狀態。這些狀態可用來對應實際資源與設定、追蹤相關的中繼資料，以及在管理大型基礎架構時提升效能。

預設情況下，這個狀態會儲存在名為 terraform.tfstate 的本機檔案中。不過我們建議將它儲存在 [HCP Terraform](https://developer.hashicorp.com/terraform/cloud-docs/migrate)，以便進行版本控管、加密，並安全地與團隊成員共享。

Terraform 會透過狀態來判斷需要對基礎架構進行哪些變更。在執行任何操作之前，Terraform 都會先進行 [refresh](https://developer.hashicorp.com/terraform/cli/commands/refresh)，以將狀態與實際的基礎架構同步更新。

Terraform 狀態的主要目的，是用來儲存遠端系統中的物件與你在設定中宣告的資源實例之間的對應關係。當 Terraform 因設定變更而建立遠端物件時，會將該物件的識別資訊記錄在對應的資源實例上，之後在設定再次變更時，就可以根據這些資訊去更新或刪除相應的遠端物件。

若想了解更多關於為什麼 Terraform 需要狀態，以及為什麼沒有狀態就無法運作，請參考文件中的 [state purpose](https://developer.hashicorp.com/terraform/language/state/purpose) 頁面。

## Inspection and Modification

雖然狀態檔的格式只是 JSON，但不建議直接編輯這些檔案。Terraform 提供了 [terraform state](https://developer.hashicorp.com/terraform/cli/commands/state) 指令，讓你可以透過 CLI 執行基本的狀態修改操作。

Terraform state 指令的使用方式與輸出格式設計上是為了方便搭配 Unix 工具(像是 grep、awk 等)使用。此外，CLI 也會幫使用者隔離狀態格式的變動，讓你不需要關心底層格式是否有改變。即使未來狀態格式有所調整，Terraform 專案也會確保 CLI 操作依然可用。

Terraform 預期設定中的資源實例與遠端物件之間是一對一的對應關係。這種對應通常是由 Terraform 自己負責建立的，也就是由 Terraform 建立每個物件並在狀態中記錄其識別資訊，或在銷毀物件後從狀態中移除對應的關聯。

如果你透過其他方式新增或移除狀態中的對應關係，例如使用 `terraform import` 匯入外部建立的物件，或用 `terraform state rm` 讓 Terraform「忘記」某個現有物件，那你就需要自行確保這個一對一的對應規則仍然成立。舉例來說，你可能需要手動刪除被「忘記」的物件，或重新匯入它，讓它綁定到另一個資源實例上。

!!! question "你可能需要手動刪除被「忘記」的物件"

    ChatGPT:

    當你使用 `terraform state rm` 指令讓 Terraform「忘記」某個資源後，Terraform 就不再追蹤它了，但實際上那個資源還存在於遠端（像是 AWS、GCP 上的某個實體資源）。 這樣會造成狀態檔裡沒有這個資源的紀錄，但遠端的實體還在，導致 Terraform 和實際環境不一致。

    所以你可能需要手動到雲端平台上刪掉那個實際存在的資源，這樣才不會留下「殘留資源」，確保 Terraform 的狀態和實際環境一致。

## Format

狀態快照是以 JSON 格式儲存的，並且新版本的 Terraform 通常會向後相容於舊版本產生的狀態快照。然而，狀態格式可能會在新版本的 Terraform 中有所變動，因此如果你開發的軟體需要直接解析或修改狀態，你應該預期在未來版本中隨著狀態格式的變化，仍需持續維護這些軟體。

另外，也有幾個整合點會產生專門供外部軟體使用的 JSON 輸出：

- [`terraform output` 指令](https://developer.hashicorp.com/terraform/cli/commands/output)有一個 `-json` 選項，可以用來取得最新狀態快照中的完整根模組輸出值集，或是某個特定名稱的輸出值。
- [`terraform show` 指令](https://developer.hashicorp.com/terraform/cli/commands/show)有一個 `-json` 選項，可以用來檢視完整的最新狀態快照，也可以檢視保存的計劃檔案，這些計劃檔案包含了計劃執行時的先前狀態副本。

在 Terraform 自動化執行的情況下，這些指令的典型使用方式是，在成功執行 `terraform apply`` 後立即運行它們，以獲取最新狀態快照的表示，然後將該結果儲存為與自動化運行相關聯的工件，這樣其他軟體就可以直接使用這些資料，而不需要重新執行 Terraform。

## References

https://developer.hashicorp.com/terraform/language/state

