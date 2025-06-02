# Style Guide

## Code style

- Run `terraform fmt` and `terraform validate` before committing your code ot version control.
    - You can use mechanisms such as Git pre-commit hooks to automatically run this command each time you commit your code.
- Use a linter such as TFLint to enforce your organization's own coding best practices.
- User `#` for single and multi-line comments.
- Use nouns for resource names and do not include the resource type in the name.
- Use underscores (`_`) to separate multiple words in names. Wrap the resource type and name in double quotes (`"`) in your resource definition.
- Let your code build on itself: define dependent resources after the resources that reference them.

    !!! note

        先定義會用到別人資源的那個東西，然後再定義被它用到的資源。這樣可以讓程式碼依照依賴順序，一步一步地建構起來。

        例如: A 用到 B
        <span style="color: red">:fontawesome-solid-xmark:</span> 定義 B -> 定義 A (引用 B)
        <font style="color: green">:fontawesome-solid-check:</font> 定義 A (引用 B) -> 定義 B

- Include a type and description for every variable.
- Include a description for every output.
- Avoid overuse of variables and local values.
- Always include a default provider configuration.
- User `count` and `for_each` sparingly.

## Code formatting

- Indent two spaces for each nesting level

- When multiple arguments with single-line values appear on consecutive lines at the same nesting level, align their equals signs:

    ```config
    ami           = "abc123"
    instance_type = "t2.micro"
    ```

- When both arguments and blocks appear together inside a block body, place all of the arguments together at the top and then place nested blocks below them. Use one blank line to separate the arguments from the blocks.

- Use empty lines to separate logical groups of arguments within a block.

- For blocks that contain both arguments and "meta-arguments" (as defined by the Terraform language semantics), list meta-arguments first and separate them from other arguments with one blank line. Place meta-argument blocks last and separate them from other blocks with one blank line.

    ```terraform
    resource "aws_instance" "example" {
      # meta-argument first
      count = 2

      ami           = "abc123"
      instance_type = "t2.micro"

      network_interface {
        # ...
      }

      # meta-argument block last
      lifecycle {
        create_before_destroy = true
      }
    }
    ```

- Top-level blocks should always be separated from one another by one blank line. Nested blocks should also be separated by blank lines, except when grouping together related blocks of the same type (like multiple `provisioner` blocks in a resource).
- Avoid grouping multiple blocks of the same type with other blocks of a different type, unless the block types are defined by semantics to form a family.
    - For example: `root_block_device`, `ebs_block_device` and `ephemeral_block_device` on `aws_instance` form a family of block types describing AWS block devices, and can therefore be grouped together and mixed.

The `terraform fmt` command formats your Terraform configuration to a subset of the above recommendations. By default, the `terraform fmt` command will only modify your terraform code in the directory that your execute it in, but you can include the `-recursive` flag to modify code in all subdirectories as well.

## Code validation

The `terraform validate` command checks that your configuration is syntactically valid and internally consistent. The `validate` command does not check if argument values are valid for a specific provider, but it will verify that they are the correct type. It does not evaluate any existing state.

!!! note "internally consistent"

    ChatGPT: 內部一致，意思是資料源之間的引用是否合理，例如引用了一個不存在的資源或屬性，這種情況就會被偵測出來。

The `terraform validate` command is safe to run automatically and frequently. You can configure your text editor to run this command as a post-save check, define it as a pre-commit hook in a Git repository, or run it as a step in a CI/CD pipeline.

## File names

We recommend the following file naming conventions:

- A `backend.tf` file that contains your backend configuration. You can define multiple `terraform` blocks in your configuration to separate your backend configuration from your Terraform and provider versioning configuration.
- A `main.tf` file that contains all resource and data source blocks.
- A `outputs.tf` file that contains all output blocks in alphabetical order.
- A `providers.tf` file that contains all `provider` blocks and configuration.
- A `terraform.tf` file that contains a single `terraform` block which defines your `required_version` and `requred_providers`.
- A `veriables.tf` file that contains all variable blocks in alphabetical order.
- A `locals.tf` file that contains local values.
- A `override.tf` file that contains override definitions for your configuration. Terraform loads this and all files ending with `_override.tf` last. Use them sparingly and add comments to the original resource definitions, as these overrides make your code harder to reason about.

As your codebase grows, limiting it to just these file can become difficult to maintain. If your code becomes hard to navigate due to its size, we recommend that your organize resources and data sources in separate files by logical groups. For example, if your web application requires networking, storage, and compute resources, you might create the following files:

- A `network.tf` file that contains your VPC, subnets, load balancers, and all other networking resources.
- A `storage.tf` file that contains your object storage and related permissions configuration.
- A `compute.tf` file that contains your compute instances.

No matter how you decide to split your code, it should be immediately clear where a maintainer can find a specific resource or date source definition.

As your configuration grows, you may need to separate it into multiple state files.

## TODO ..

## References

https://developer.hashicorp.com/terraform/language/style
