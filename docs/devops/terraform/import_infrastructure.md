# Import Infrastructure

## Overview

本主題概述了 Terraform 提供的幾個指令，這些指令可讓你將現有的基礎架構資源匯入，從而讓 Terraform 能夠接管並管理這些資源。

!!! note

    動手試試看：你可以參考「[Import Terraform Configuration](https://developer.hashicorp.com/terraform/tutorials/state/state-import?utm_source=WEBSITE&utm_medium=WEB_IO&utm_offer=ARTICLE_PAGE&utm_content=DOCS)」教學，實際操作匯入流程。

### Workflows

你可以透過 Terraform CLI 將現有資源匯入到 state 中，也可以使用 HCP Terraform 執行匯入操作。若要匯入多個資源，建議使用 `import` 區塊來處理。

#### Import to state

在執行 `terraform import` 之前，你必須手動撰寫該資源的設定區塊。這個 `resource` 區塊用來告訴 Terraform 要將匯入的資源對應到哪一個設定，才能正確地建立映射關係。

`terraform import` CLI 命令只能把資源匯入到 [state](https://developer.hashicorp.com/terraform/language/state)，不會自動產生任何 HCL 配置。如果你想為匯入的資源一併產生對應的設定檔，請[改用 import 區塊](https://developer.hashicorp.com/terraform/language/import)。

Terraform 預期每個遠端資源只能對應到一個資源位址（resource address）。你應該將每個遠端資源匯入到唯一的一個 Terraform 資源。如果同一個資源被重複匯入到多個地方，可能會導致非預期的行為。詳細說明請參考官方文件中的「[State](https://developer.hashicorp.com/terraform/language/state)」章節。

#### HCP Terraform

當你在命令列中使用 Terraform 搭配 HCP Terraform 時，像是 `apply` 這類指令會在 HCP Terraform 的環境中執行。但 `import` 指令是在本機執行的，因此無法存取 HCP Terraform 上的變數或設定。為了讓匯入操作順利進行，你可能需要在本機設定與 HCP Terraform 遠端 workspace 相同的變數，確保環境一致。

#### Import multiple resources

你可以在 `import` 區塊中指定多個資源，來一次匯入多筆資源。這些匯入操作也可以像平常一樣，透過 `plan` 和 `apply` 的流程進行預覽與套用。如需更多資訊，請參考 Terraform 設定語言文件中的 [import 區塊](https://developer.hashicorp.com/terraform/language/import)說明。

### Resource importability

每個 Terraform 資源都必須實現一些基本邏輯，才能讓它被匯入。因此，並不是所有的 Terraform 資源都能夠被匯入。

可以匯入的資源會在 [Terraform Registry](https://registry.terraform.io/) 每個資源的文件頁面底部註明。如果你在匯入資源時遇到問題，可以在對應的 provider 儲存庫中回報問題。

要讓資源可以被匯入，請參考[Resources - Import](https://developer.hashicorp.com/terraform/plugin/sdkv2/resources/import)。

## Import existing resources

本主題說明如何使用 `terraform import` 指令，將現有的基礎架構資源匯入，讓你可以以程式碼的方式來管理這些資源。

!!! note

    動手試試看：你可以參考「[Import Terraform Configuration](https://developer.hashicorp.com/terraform/tutorials/state/state-import?utm_source=WEBSITE&utm_medium=WEB_IO&utm_offer=ARTICLE_PAGE&utm_content=DOCS)」教學，實際操作匯入流程。

### Overview

使用 `terraform import` 指令可以將現有的基礎架構資源匯入到 Terraform 的 state 中。不過，`terraform import` 一次只能匯入一個資源，無法一次匯入整組資源，像是整個 AWS VPC。

要匯入資源，請依照以下步驟操作：
1. 在 Terraform 設定中加入你想管理的資源區塊(resource)。
2. 執行 `terraform import` 指令，將現有資源匯入到 Terraform 的 state 中。

!!! warning

    Terraform 預期每個它所管理的遠端資源只能對應到一個資源位址(resource address)，這在 Terraform 自行建立資源時通常是自動保證的。
    如果你是將現有資源匯入 Terraform，請特別注意：每個遠端資源只能匯入到一個資源位址。若同一個資源被重複匯入到多個位址，可能會導致 Terraform 出現非預期的行為。
    如需更多相關說明，請參考官方文件中的「[State](https://developer.hashicorp.com/terraform/language/state)」章節。

### Add the resource to your configuration

在你的設定中撰寫一個 resource 區塊，來描述你想匯入的資源。請為這個資源指定一個名稱，這個名稱是唯一的 ID，可供你在設定的其他地方引用這個資源。

在以下範例中，匯入的資源是一個名為 `example` 的 AWS instance：

```terraform
resource "aws_instance" "example" {
  # ...instance configuration...
}
```

你不需要立刻填寫 resource 區塊的內容，可以在匯入該實例之後再補上參數定義。

### Run the `terraform import` command

執行 `terraform import`，將現有的實例與你寫好的資源設定關聯起來：

```shell
$ terraform import aws_instance.example i-abcd1234
```

此指令會尋找 ID 為 `i-abcd1234` 的 AWS EC2 實例，然後將該實例的現有設定(由 EC2 API 描述)附加到模組中的 `aws_instance.example` 資源名稱。在這個範例中，模組路徑表示使用的是根模組。最後，這個映射關係會被儲存在 Terraform 的 state 檔案中。

也可以使用資源的路徑將資源匯入到子模組中，或者將具有 `count` 或 `for_each` 設定的資源實例匯入。詳情請參閱 [Resource Addressing](https://developer.hashicorp.com/terraform/cli/state/resource-addressing) 了解如何指定目標資源。

給定 ID 的語法取決於要匯入的資源類型。例如，AWS EC2 實例使用由 EC2 API 發行的不透明 ID，而 AWS Route53 區域則使用域名本身。請參閱每個可匯入資源的文件，以瞭解所需 ID 的具體格式。

執行上述指令後，該資源會被記錄在 state 檔案中。接著，你可以執行 `terraform plan` 來查看設定與匯入的資源之間的差異，並對設定進行調整，以使其與匯入資源的當前狀態(或期望狀態)一致。

### Complex Imports

上述匯入被視為「簡單匯入」：將一個資源匯入到 state 檔案中。匯入也可能會是「複雜匯入」，即同時匯入多個資源。例如，匯入 AWS 網路 ACL 時，不僅會匯入一個 `aws_network_acl`，還會為每條規則匯入一個 `aws_network_acl_rule`。

在這種情況下，次級資源(secondary resources)在設定中尚不存在，因此需要參考匯入輸出，並為每個次級資源在設定中建立一個資源區塊。如果未進行此操作，Terraform 在下一次執行時會計畫刪除這些匯入的物件。

## terraform import command reference

`terraform import` 指令用來將現有的資源匯入到 Terraform 中。詳細資訊請參閱 [Import](https://developer.hashicorp.com/terraform/cli/import) 相關文檔。

!!! note

    動手做：試試看「[Import Terraform Configuration](https://developer.hashicorp.com/terraform/tutorials/state/state-import?utm_source=WEBSITE&utm_medium=WEB_IO&utm_offer=ARTICLE_PAGE&utm_content=DOCS)」教學，實際體驗如何將現有資源匯入 Terraform 設定中。

### Usage

用法：`terraform import [選項] ADDRESS ID`

`terraform import` 會根據提供的 `ID` 查找現有資源，並將其匯入到指定的 `ADDRESS` 中，將該資源添加到 Terraform 的 state 檔案中。

`ADDRESS` 必須是[有效的資源位址](https://developer.hashicorp.com/terraform/cli/state/resource-addressing)。由於任何資源位址都是有效的，因此 `terraform import` 指令可以將資源匯入到模組中，也可以直接匯入到 state 的根目錄中。

`ID` 的格式取決於你要匯入的資源類型。例如，對於 AWS EC2 實例，它是實例的 `ID` (如 `i-abcd1234`)； 而對於 AWS Route53 區域，則是區域的 `ID` (如`Z12ABC4UGMOZ2N`)。請參考各個 provider 的官方文件來確認正確的 `ID` 格式。如果你不確定，也可以直接嘗試匯入一個 `ID` - 如果格式錯誤，只會收到錯誤訊息，不會造成其他影響。

!!! warning

    Terraform 預期每個它所管理的遠端物件都只會對應到一個資源位址(resource address)，這在 Terraform 自行建立所有資源時通常可以自動保證。

    如果你是將現有的資源匯入 Terraform，請務必只將每個遠端資源匯入到一個資源位址。若你將同一個資源重複匯入多次，Terraform 可能會出現非預期的行為。

    更多相關說明，請參閱官方文件中的「[State](https://developer.hashicorp.com/terraform/language/state)」章節。

你也可以不用手動匯入資源，而是在 Terraform 設定中加入 `import` 區塊，這樣當你執行 `terraform apply` 時，Terraform 就會自動匯入資源。透過這種方式，你可以把資源匯入的流程自動化，整合進你的 CI/CD 流程中。 更多詳細資訊請參閱[官方文件中的 `import` block](https://developer.hashicorp.com/terraform/language/import) 說明。

指令列的旗標(flags)都是可選的。以下是可用的旗標列表：

- `-config=path`：指定包含 Terraform 設定檔的目錄路徑，用來設定匯入時所需的 provider。預設值是你目前的工作目錄。如果這個目錄中沒有 Terraform 設定檔，你就必須透過手動輸入或環境變數來設定 provider。
- `-input=true`: 是否要求使用者輸入 provider 設定。
- `-lock=false`: 匯入時不鎖定 state。若其他人也在操作同一個 workspace，這樣做可能會導致競爭狀況，請小心使用。
- `-lock-timeout=0s`: 如果 state 被鎖定，要重試取得鎖定的等待時間。
- `-no-color`: 輸出不加上顏色，方便在純文字環境下閱讀。
- `-parallelism=n`: 當 Terraform [建立資源依賴圖](https://developer.hashicorp.com/terraform/internals/graph#walking-the-graph)並執行操作時，可以限制同時執行的操作數量。預設值為 10。
- `-provider=provider`: 已棄用。用來覆蓋匯入時使用的 provider 設定。一般建議使用設定檔中定義的 provider，這樣較安全、穩定。
- `-var 'foo=bar'`: 傳入變數值到 Terraform 設定中。可重複使用多次。這些值會被當作 Terraform [語言中的表達式](https://developer.hashicorp.com/terraform/language/expressions/types)處理，所以可以傳遞 list 或 map。
- `-var-file=foo`: 從[變數檔](https://developer.hashicorp.com/terraform/language/values/variables#variable-definitions-tfvars-files)中載入變數。若目前目錄中有 `terraform.tfvars` 或 `.auto.tfvars` 檔案，Terraform 會自動載入。`terraform.tfvars` 會先載入，然後按字母順序載入 `.auto.tfvars`。若指定了 `-var-file`，它的值會覆蓋自動載入的內容。這個旗標可重複使用，且僅在搭配 `-config` 使用時有意義。

針對使用 [HCP Terraform CLI 整合](https://developer.hashicorp.com/terraform/cli/cloud)或 [`remote` backend](https://developer.hashicorp.com/terraform/language/backend/remote) 的設定，`terraform import` 也接受選項 [-ignore-remote-version](https://developer.hashicorp.com/terraform/cli/cloud/command-line-arguments#ignore-remote-version)，用來忽略對遠端 Terraform 版本的檢查。

針對僅使用 [`local` backend](https://developer.hashicorp.com/terraform/language/backend/local) 的設定，`terraform import` 也支援舊有選項 [`-state`、`-state-out` 和 `-backup`](https://developer.hashicorp.com/terraform/language/backend/local#command-line-arguments)。

### Provider Configuration

Terraform 會嘗試載入配置檔，這些檔案用來設定匯入所使用的 provider。如果沒有找到配置檔，或是沒有針對特定 provider 的配置，Terraform 會提示你輸入存取憑證。你也可以透過環境變數來設定 provider。

Terraform 讀取配置檔時唯一的限制是，匯入所用的 provider 配置不能依賴於非變數的輸入。例如，provider 配置不能依賴資料來源。

舉個實際範例，如果你正在匯入 AWS 資源，並且你的配置檔案內容如下，則 Terraform 會根據這個檔案來配置 AWS provider：

```terraform
variable "access_key" {}
variable "secret_key" {}

provider "aws" {
  access_key = var.access_key
  secret_key = var.secret_key
}
```

### Example: Import into Resource

這個範例將會把一個 AWS 實例匯入到名為 `foo` 的 `aws_instance` 資源中：

```shell
$ terraform import aws_instance.foo i-abcd1234
```

### Example: Import into Module

以下範例將會把一個 AWS 實例匯入到名為 `bar` 的 `aws_instance` 資源，並放入名為 `foo` 的模組中：

```shell
$ terraform import module.foo.aws_instance.bar i-abcd1234
```

### Example: Import into Resource configured with count

以下範例將會把一個 AWS 實例匯入到配置了 [`count`](https://developer.hashicorp.com/terraform/language/meta-arguments/count) 的第一個 `aws_instance` 資源，該資源名為 `baz`：

```shell
$ terraform import 'aws_instance.baz[0]' i-abcd1234
```

### Example: Import into Resource configured with for_each

以下範例將會把一個 AWS 實例匯入到使用 [`for_each`](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each) 配置的 `aws_instance` 資源中的名為 `baz` 的 “`example`” 實例：

Linux, Mac OS, and UNIX:

```shell
$ terraform import 'aws_instance.baz["example"]' i-abcd1234
```

PowerShell:

```shell
$ terraform import 'aws_instance.baz[\"example\"]' i-abcd1234
```

Windows (`cmd.exe`):

```shell
$ terraform import aws_instance.baz[\"example\"] i-abcd1234
```

## References

https://developer.hashicorp.com/terraform/cli/import
https://developer.hashicorp.com/terraform/cli/import/usage
https://developer.hashicorp.com/terraform/cli/commands/import

