Each Terraform configuration has an associated backend that defines how Terraform executes operations and where Terraform stores persistent data, like state.

- The persistent data stored in the backend belongs to a workspace.
- The backend initially has only one workspace containing one Terraform state associated with that configuration.
- Some backends support multiple named workspaces, allowing multiple states to be associated with a single configuration.
- The configuration still has only one backend, but you can deploy multiple distinct instances of that configuration without configuring a new backend or changing authentication credentials.

## Backends supporting multiple workspaces

AzureRM, Consul, COS, GCS, Kubernetes, Local, OSS, Postgres, Remote, S3

## Using workspaces

- Terraform starts with a single, default workspace named `default` that you cannot delete.
- If you have not created a new workspace, you are using the default workspace in your Terraform working directory.
- When you run `terraform plan` in a new workspace, Terraform does not access existing resources in other workspaces. These resources still physically exist, but you must switch workspaces to manage them.

## Current workspace interpolation

- Within your Terraform configuration, you may include the name of the current workspace using the `${terraform.workspace}` interpolation sequence. This can be used anywhere interpolations are allowed.
- Referencing the current workspace is useful for changing behavior based on the workspace.

    ```terraform
    resource "aws_instance" "example" {
        count = terraform.workspace == "default" ? 5 : 1

        # ...
    }
    ```

    or

    ```terraform
    resource "aws_instance" "example" {
        tags = {
            Name = "web - ${terraform.workspace}"
        }

        # ...
    }
    ```

## Reference

https://developer.hashicorp.com/terraform/language/state/workspaces

