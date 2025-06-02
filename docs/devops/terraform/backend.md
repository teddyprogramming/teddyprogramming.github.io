`backend` defines where Terraform stores its state date files.

There are some important limitations on backend configuration:

- A configuration can only provide one backend block.
- A backend block cannot refer to named values (like input variables, locals, or data source attributes)
- You cannot reference values declared within backend blocks elsewhere in the configuration.

## Backend types

- Terraform users a backend called `local` by default.
    - The `local` backend type stores state as a local file on disk.
- remote backend

    ```terraform
    backend "remote" {
        organization = "example_corp"
    }
    ```

    - Using environment variables to supply credentials and other sensitive data is recommended.
    - If you use `-backend-config` or hardcode these values directly in your configuration, Terraform will include these values in both the `.terraform` subdirectory and in plan files.
    - Terraform writes the backend configuration in plan text in two separate files.
        - The `.terraform/terraform.tfstate` file contains the backend configuration for the current working directory.
        - All plan files capture the information in `.terraform/terraform.tfstate` at the time the plan was created. This help ensure Terraform is applying the plan to correct set of infrastructure.
    - When applying a plan that you previously saved to a file, Terraform uses the backend configuration stored in that file instead of the current backend settings. If that configuration contains time-limited credentials, they may expire before you finish applying the plan. Use environment variables to pass credentials when you need to use different values between the plan and apply steps.

## Initialize the backend

- When you change a backend's configuration, you must run `terraform init` again to validate and configure the backend before you can perform any plans, applies, or state operations.
- After you initialize, Terraform creates a `.terraform/` directory locally. This directory contains the most recent backend configuration, including any authentication parameters you provided to the Terraform CLI. **Do not check this directory into Git, as it may contain sensitive credentials for your remote backend.**
- When you change backends, Terraform gives you the option to migrate your state to the new backend. This lets you adopt backends without losing any existing state.

!!! warning "Important: Before migrating to a new backend, we strongly recommend manually backing up your state by coping your `terraform.tfstate` file to another location."

## Partial configuration

- You do not need to specify every required argument in the backend congiguration. Omitting certain arguments may be desirable if some arguments are provided automatically by an automation script running Terraform.
- When some or all of the arguments are omitted, we call this a partial configuration.
- With a partial configuration, the remaining configuration arguments must be provided as part of the initialization process.
- There are several ways to supply the remaining arguments:
    1. Files

        - A configuration file may be specified via the `init` command line. To specify a file, use the `-backend-config=PATH` option when running `terraform init`.
        - The partial configuration must have a `backend` block containing keys set to empty values. When you run the `terraform init -backend-config="<path-to-remaining-configuration>"` command, Terraform populates the keys in the partial `backend` configuration with matching key values from the specified configuration file.

        ```shell
        $ terraform init -backend-config="./state.config"
        ```

        ```terraform title="state.tf"
        terraform {
            backend "s3" {
                bucket  = ""
                key     = ""
                region  = ""
                profile = ""
            }
        }
        ```

        ```config title="state.config"
        buckety = "your-bucket"
        key     = "your-state.tfstate"
        region  = "eu-central-1"
        profile = "Your_Profile"
        ```

        - When your configuration file contains secrets, you can store them in a secure date store, such as Vault, in which case it must be downloaded to the local disk before running Terraform.

    2. Command-line key/value pairs

        - Key/value paires can be specified via the `init` command line.

            !!! warning "Note that many shells retain command-line flags in a history file, so this isn't recommended for secrets."

        - To specify a single key/value pair, use the `-backend-config="KEY=VALUE"` option when running `terraform init`.

    3. Interactively

        - Terraform will interactively ask you for the required values, unless interactive input is disabled.
        - Terraform will not prompt for optional values.

- When using partial configuration, Terraform requires at a minimum that an empty backend configuration is specified in one of the root Terraform configuration files, to specify the backend type.

    ```backend
    terraform {
        backend "consul" {}
    }
    ```

    - File:

        - A backend configuration file has the contents of the `backend` block as top-level attributes, without the need to wrap it in another `terraform` or `backend` block:

        - `*.backendname.tfbackend` (e.g., `config.consul.tfbackend`) is the recommended naming pattern.
            - Terraform will not prevent you from using other name but following this convention will help your editor understand the content and likely provide better editing experience as a result.

        ```config title="config.consul.tfbackend"
        address = "demo.consul.io"
        path    = "example_app/terraform_state"
        schema  = "https"
        ```

    - Command-line key/value pairs

        ```shell
        $ terraform init \
            -backend-config="address=demo.consul.io" \
            -backend-config="path=example_app/terraform_state" \
            -backend-config="schema=https"
        ```

        - The Consul backend also requires a Consul access token. Per the recommendation above of omitting credentials from the configuration and using other mechanisms, the Consul token would be provided by setting either the `CONSUL_HTTP_TOKEN` or `CONSUL_HTTP_AUTH` environment variables.

## Change configuration

- You can change your backend configuration at any time.
    - You can change both the configuration itself as well as the type of backend (for example from "consul" to "s3")
- Terraform will automatically detect any changes in your configuration and request a reinitialization. As part of the reinitialization process, Terraform will ask if you'd like to migrate your existing state to the new configuration. This allows you to easily switch from one backend to another.
- If you're using multiple workspaces, Terraform can copy all workspaces to the destination. If Terraform detects you have multiple workspaces, it will ask if this is whay you want to do.
- If you're just reconfiguration the same backend, Terraform will still ask if you want to migrate you state. You can respond "no" in this scenario.

## Remove a backend configuration

- Remove the `backend` block from your configuration and reinitialize the directory when prompted. Terraform also prompts you to migrate the state to the degault `local` backend.

## References

- https://developer.hashicorp.com/terraform/language/backend
