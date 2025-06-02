# Intro to Terraform

## What is Terraform?

HashiCorp Terraform 是一種基礎設施即程式碼(Infrastructure as Code, IaC)工具，讓你可以用人類可讀的設定檔來定義雲端或內部的(on-prem)資源，這些設定檔可以被版本控制、重複使用、以及分享。 你可以透過一致的工作流程，在基礎設施的整個生命週期中佈署和管理所有資源。 Terraform 能管理像是運算資源、儲存空間、網路等低階元件，也能處理像是 DNS 記錄或 SaaS 功能等高階元件。

!!! question "on-perm resources"

    ChatGPT: 自己買的設備，而非雲端設備。

!!! note

    透過「入門教學(Get Started tutorials)」，實際操作如何在常見的雲端平台上管理基礎架構，包括：[Amazon Web Services(AWS)](https://developer.hashicorp.com/terraform/tutorials/aws-get-started)、[Azure](https://developer.hashicorp.com/terraform/tutorials/azure-get-started)、[Google Cloud Platform(GCP)](https://developer.hashicorp.com/terraform/tutorials/gcp-get-started)、[Oracle Cloud Infrastructure](https://developer.hashicorp.com/terraform/tutorials/oci-get-started)，以及 [Docker](https://developer.hashicorp.com/terraform/tutorials/docker-get-started)。

## How does Terraform work?

Terraform 是透過雲端平台或其他服務所提供的 API 來建立和管理各種資源。只要某個平台或服務有可用的 API，Terraform 就能透過對應的「Provider」來操作它，幾乎可以支援任何系統。

![](https://developer.hashicorp.com/_next/image?url=https%3A%2F%2Fcontent.hashicorp.com%2Fapi%2Fassets%3Fproduct%3Dterraform%26version%3Drefs%252Fheads%252Fv1.11%26asset%3Dwebsite%252Fimg%252Fdocs%252Fintro-terraform-apis.png%26width%3D2048%26height%3D644&w=3840&q=75&dpl=dpl_43ouZGSA2EfNQqaj2sVyURFLfppb)

HashiCorp 和 Terraform 的社群已經撰寫了成千上萬個 Provider，能用來管理各種不同類型的資源與服務。 你可以在 [Terraform Registry](https://registry.terraform.io/) 上找到所有公開提供的 Provider，例如 Amazon Web Services(AWS)、Azure、Google Cloud Platform(GCP)、Kubernetes、Helm、GitHub、Splunk、DataDog 等等，還有更多其他服務。

Terraform 的核心工作流程分成三個階段:

1. Write: 你會先定義想要建立的資源，這些資源可以來自多個不同的雲端平台或服務。例如，你可能會寫一份設定檔，用來在 Virtual Private Cloud (VPC) 網路中建立虛擬機、設定安全群組(security groups)和負載平衡器(load balancer)來部署一個應用程式。

2. Plan: Terraform 會根據現有的基礎架構和你寫的設定內容，產生一份「執行計畫」，說明它會建立、更新或刪除哪些資源。

3. Apply: 經過你確認後，Terraform 會依照計畫進行操作，並根據資源之間的依賴順序，確保執行的先後順序正確。 舉例來說，如果你更新了 VPC 的設定，並且同時變更了 VPC 裡虛擬機的數量，Terraform 會先重建 VPC，再調整虛擬機的數量。

![](https://developer.hashicorp.com/_next/image?url=https%3A%2F%2Fcontent.hashicorp.com%2Fapi%2Fassets%3Fproduct%3Dterraform%26version%3Drefs%252Fheads%252Fv1.11%26asset%3Dwebsite%252Fimg%252Fdocs%252Fintro-terraform-workflow.png%26width%3D2038%26height%3D1773&w=3840&q=75&dpl=dpl_43ouZGSA2EfNQqaj2sVyURFLfppb)

## Why Terraform?

### Manage any infrastructure

你可以在 [Terraform Registry](https://registry.terraform.io/) 上找到許多你已經在使用的平台和服務所對應的 Provider。你也可以[自行撰寫](https://developer.hashicorp.com/terraform/plugin)自己的 Provider。 Terraform 採用「[不可變基礎架構(immutable infrastructure)](https://www.hashicorp.com/resources/what-is-mutable-vs-immutable-infrastructure)」的方式來管理資源，能降低升級或修改服務與基礎架構時的複雜度。

!!! question "Terraform 採用「不可變基礎架構(immutable infrastructure)」的方式來管理資源"

    ChatGPT: 所謂的「不可變」，意思是，一旦建立後，就不修改，而是重建。

### Track your infrastructure

Terraform 會在修改你的基礎架構之前，先產生一份執行計畫(plan)，並提示你確認是否要執行。 它也會透過一個「[state 檔案](https://developer.hashicorp.com/terraform/language/state)」來記錄實際的基礎架構狀態，這個檔案就像是你環境的真實來源(source of truth)。 Terraform 會根據這個 state 檔案，判斷需要對基礎架構做哪些變更，讓實際狀況與你寫的設定檔保持一致。

### Automate changes

Terraform 的設定檔是宣告式(declarative)的，也就是說你只需要描述「想要的基礎架構最終長什麼樣子」。 你不需要寫下每一步建立資源的流程，因為 Terraform 會自動處理背後的執行邏輯。

Terraform 會建立一張資源關係圖，用來判斷資源之間的依賴順序，並同時(平行)建立或修改沒有相依關係的資源。這樣可以讓 Terraform 更有效率地佈署資源。

### Standardize configurations

Terraform 支援可重複使用的設定元件，稱為 [module(模組)](https://developer.hashicorp.com/terraform/language/modules)，它可以用來定義一組可調整的基礎架構資源，幫助你節省時間，同時推廣最佳實務。你可以使用 Terraform Registry 上公開提供的模組，也可以自己撰寫模組來符合專案需求。

### Collaborate

由於 Terraform 的設定是寫在檔案裡的，你可以把它提交到版本控制系統(VCS)中，例如 Git，並透過 [HCP Terraform](https://developer.hashicorp.com/terraform/intro/terraform-editions#hcp-terraform) 來有效率地管理團隊的 Terraform 工作流程。

HCP Terraform 會在一個一致且可靠的環境中執行 Terraform，並提供以下功能：安全存取共享的狀態檔與敏感資料、角色權限控管(role-based access controls, RBAC)、私有模組與 Provider 的分享註冊庫(Private Registry)，以及更多進階功能。

!!! question "HCP Terraform 會在一個一致且可靠的環境中執行 Terraform"

    ChatGPT: HCP Terraform 幫你把 Terraform 的執行環境標準化，不管是哪個人、在哪台電腦下指令，結果都會是一樣的，而且不會出錯。

    平常你可能是在自己電腦跑 terraform apply，但每個人的環境(例如 Terraform 版本、設定檔、憑證)都可能不一樣，容易出問題。

    而 HCP Terraform 幫你：

	- 在雲端執行 Terraform，不依賴每個人自己的機器
	- 幫你固定好執行環境(例如版本、變數、provider)
	- 減少「在我這邊可以跑，在你那邊壞掉」的狀況
	- 適合多人協作，避免重複 apply 或出現衝突

!!! tip

    你可以進一步了解 [Terraform 的使用情境](https://developer.hashicorp.com/terraform/intro/use-cases)，以及[它與其他工具的比較](https://developer.hashicorp.com/terraform/intro/vs)。

## Community

我們歡迎來自社群的提問、建議與貢獻！
- 有問題可以在 [HashiCorp Discuss](https://discuss.hashicorp.com/c/terraform-core/27) 發問
- 請參考我們的 [貢獻指南(contributing guide)](https://github.com/hashicorp/terraform/blob/main/.github/CONTRIBUTING.md)
- 若發現 bug 或有功能需求，歡迎[提交 issue](https://github.com/hashicorp/terraform/issues/new/choose)

## References

https://developer.hashicorp.com/terraform/intro
