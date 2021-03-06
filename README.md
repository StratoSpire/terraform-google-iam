# Google IAM Terraform Module

This Terraform module makes it easier to non-destructively manage multiple IAM roles for resources on Google Cloud Platform.

## Usage
Full examples are in the [examples](./examples/) folder, but basic usage is as follows for managing roles on two projects:

```hcl
module "iam_binding" {
  source = "github.com/terraform-google-modules/terraform-google-iam?ref=v1.0.0"

  projects = ["project-123456", "project-9876543"]

  bindings = {
    "roles/storage.admin" = [
      "group:test_sa_group@lnescidev.com",
      "user:someone@google.com",
    ]

    "roles/compute.networkAdmin" = [
      "group:test_sa_group@lnescidev.com",
      "user:someone@google.com",
    ]

    "roles/compute.imageUser" = [
      "user:someone@google.com",
    ]
  }
}
```

The module also offers an **authoritative** mode which will remove all roles not assigned through Terraform. This is an example of using the authoritative mode to manage access to a storage bucket:

```hcl
module "storage_buckets_iam_binding" {
  source          = "github.com/terraform-google-modules/terraform-google-iam"
  storage_buckets = ["my-storage-bucket"]

  mode = "authoritative"

  bindings = {
    "roles/storage.legacyBucketReader" = [
      "user:josemanuelt@google.com",
      "group:test_sa_group@lnescidev.com",
    ]

    "roles/storage.legacyBucketWriter" = [
      "user:josemanuelt@google.com",
      "group:test_sa_group@lnescidev.com",
    ]
  }
}
```

### Variables
Following variables are the most important to control module's behavior:

- Mode

    This variable controls the module's behavior, by default is set to "additive", possible options are:

      - additive: add members to role, old members are not deleted from this role.
      - authoritative: set the role's members, other roles' members are not deleted.

- Bindings

  Is a map of role (key) and list of members (value) with member type prefix, for example:

        ```hcl
        bindings = {
            "roles/<some_role>" = [
                "user:someone@somewhere.com",
                "group:somepeople@somewhereelse.com"
            ]
        }
        ```

- Project

  This variable can be defined either on `provider` section and the module calling itself. It is only used for the following resources:

  - `service_accounts`
  - `pubsub_topics`
  - `pubsub_subscriptions`

Please see the variables.tf file on root folder for more information about types and default values for module's variables.

### Outputs
There is no outputs for this module.

## Caveats

### Referencing values/attributes from other resources

This Terraform module performs operations over some variables before making any changes on the IAM bindings in GCP.

Because of that, it is important to note that putting a value or attribute of a resource within the following variables, will cause an error:

- `bindings`
- `projects`
- `organizations`
- `folders`
- `service_accounts`
- `subnets`
- `storage_buckets`
- `pubsub_topics`
- `pubsub_subscriptions`
- `kms_key_rings`
- `kms_crypto_keys`

For example, this will fail:

```hcl
resource google_folder "my_new_folder" {
  display_name = "folder-test"
  parent = "76543265432"
}

resource "google_service_account" "my_service_account" {
  account_id   = "my-new-service-account"
}

module "iam_binding" {
  source = "github.com/terraform-google-modules/terraform-google-iam"
  mode   = "authoritative"

  folders = ["${google_folder.my_new_folder.id}"]

  bindings = {
    "roles/storage.admin" = [
      "group:test_sa_group@lnescidev.com",
      "serviceAccount:${google_service_account.my_service_account.id}",
    ]

    "roles/compute.networkAdmin" = [
      "group:test_sa_group@lnescidev.com",
      "user:someone@google.com",
    ]
  }
}
```

First, because the `folders` variable has a reference to a resource that is not already created (`my_new_folder`). Second because the `bindings` variable has a reference to `my_service_account` and it is not created yet. The error output is as follows: `(...) value of 'count' cannot be computed`

#### Workaround

To avoid this error, use values or attributes of resources that are already created before calling this module.

Note that as soon as the resources have been created once they **can** be referenced successfully (once they are in the Terraform state file).

Therefore, a simple workaround is as follows:

1. Comment out the call to this module.
2. Run `terraform apply` to create the other resources and persist them to the state file.
3. Uncomment this module.
4. Run `terraform apply` to apply the bindings.

## IAM Bindings
You can choose the following resources type for apply the IAM bindings:

- Projects (`projects` variable)
- Organizations(`organizations` variable)
- Folders (`folders` variable)
- Service Accounts (`service_accounts` variable)
- Subnetworks (`subnets` variable)
- Storage buckets (`storage_buckets` variable)
- Pubsub topics (`pubsub_topics` variable)
- Pubsub subscriptions (`pubsub_subscriptions` variable)
- Kms Key Rings (`kms_key_rings` variable)
- Kms Crypto Keys (`kms_crypto_keys` variable)

Set the specified variable on the module call to choose the resources to affect. Remember to set the `mode` [variable](#variables) and give enough [permissions](#permissions) to manage the selected resource as well.

## File structure
The project has the following folders and files:

- /: root folder.
- /examples: examples for using this module.
- /scripts: Scripts for specific tasks on module (see Infrastructure section on this file).
- /test: Folders with files for testing the module (see Testing section on this file).
- /main.tf: main file for this module, contains all the logic for operate the module.
- /*_iam.tf: files for manage the IAM bindings for each resource type.
- /variables.tf: all the variables for the module.
- /output.tf: the outputs of the module.
- /readme.MD: this file.

## Requirements
### Terraform plugins
- [Terraform](https://www.terraform.io/downloads.html) 0.10.x
- [terraform-provider-google](https://github.com/terraform-providers/terraform-provider-google) 1.10.0

### jq
- [jq](https://stedolan.github.io/jq/) 1.5

### Permissions
In order to execute this module you must have a Service Account with an appropriate role to manage IAM for the applicable resource. The appropriate role differs depending on which resource you are targeting, as follows:

- Organization:
  - Organization Administrator: Access to administer all resources belonging to the organization
    and does not include privileges for billing or organization role administration.
  - Custom: Add resourcemanager.organizations.getIamPolicy and
    resourcemanager.organizations.setIamPolicy permissions.
- Project:
  - Owner: Full access and all permissions for all resources of the project.
  - Projects IAM Admin: allows users to administer IAM policies on projects.
  - Custom: Add resourcemanager.projects.getIamPolicy and resourcemanager.projects.setIamPolicy permissions.
- Folder:
  - The Folder Admin: All available folder permissions.
  - Folder IAM Admin: Allows users to administer IAM policies on folders.
  - Custom: Add resourcemanager.folders.getIamPolicy and
    resourcemanager.folders.setIamPolicy permissions (must be added in the organization).
- Service Account:
  - Service Account Admin: Create and manage service accounts.
  - Custom: Add resourcemanager.organizations.getIamPolicy and
    resourcemanager.organizations.setIamPolicy permissions.
- Subnetwork:
  - Project compute admin: Full control of Compute Engine resources.
  - Project compute network admin: Full control of Compute Engine networking resources.
  - Project custom: Add compute.subnetworks.getIamPolicy	and
    compute.subnetworks..setIamPolicy permissions.
- Storage bucket:
  - Storage Admin: Full control of GCS resources.
  - Storage Legacy Bucket Owner: Read and write access to existing
    buckets with object listing/creation/deletion.
  - Custom: Add storage.buckets.getIamPolicy	and
  storage.buckets.setIamPolicy permissions.
- Pubsub topic:
  - Pub/Sub Admin: Create and manage service accounts.
  - Custom: Add pubsub.topics.getIamPolicy and pubsub.topics.setIamPolicy permissions.
- Pubsub subscription:
  - Pub/Sub Admin role: Create and manage service accounts.
  - Custom role: Add pubsub.subscriptions.getIamPolicy and
    pubsub.subscriptions.setIamPolicy permissions.
- Kms Key Ring:
  - Owner: Full access to all resources.
  - Cloud KMS Admin: Enables management of crypto resources.
  - Custom: Add cloudkms.keyRings.getIamPolicy and cloudkms.keyRings.getIamPolicy permissions.
- Kms Crypto Key:
  - Owner: Full access to all resources.
  - Cloud KMS Admin: Enables management of cryptoresources.
  - Custom: Add cloudkms.cryptoKeys.getIamPolicy	and cloudkms.cryptoKeys.setIamPolicy permissions.

## Install

### Terraform
Be sure you have the correct Terraform version (0.10.x), you can choose the binary here:
- https://releases.hashicorp.com/terraform/

### Terraform plugins
Be sure you have the compiled plugins on $HOME/.terraform.d/plugins/

- [terraform-provider-google](https://github.com/terraform-providers/terraform-provider-google) 1.10.0

See each plugin page for more information about how to compile and use them.

## Fast install (optional)
For a fast install, please configure the variables on init_centos.sh  or init_debian.sh script and then launch it.

The script will do:
- Environment variables setting
- Installation of base packages like wget, curl, unzip, gcloud, etc.
- Installation of go 1.9.0
- Installation of Terraform 0.10.x
- Download the terraform-provider-google plugin
- Compile the terraform-provider-google plugin
- Move the terraform-provider-google to the right location

## Development

### Requirements
- [bats](https://github.com/sstephenson/bats) 0.4.0
- [jq](https://stedolan.github.io/jq/) 1.5

### Integration Tests
The integration tests for this module are built with [bats](https://github.com/sstephenson/bats), basically the test checks the following:
- Perform `terraform init` command
- Perform `terraform get` command
- Perform `terraform plan` command and check that it'll create *n* resources, modify 0 resources and delete 0 resources.
- Perform `terraform apply -auto-approve` command and check that it has created the *n* resources, modified 0 resources and deleted 0 resources.
- Perform several `gcloud` commands and check the IAM bindings are in the desired state.
- Perform `terraform destroy -force` command and check that it has destroyed the *n* resources.

Please edit the *test/integration/iam-bindings-test/main_test.sh* file in order to specify the roles and members to be granted/used.

You can use the following command to run the integration tests in the folder */test/integration/*

  `. launch_additive.sh` for additive bindings testing.
  `. launch_authoritative.sh` for authoritative bindings testing.

### Linting
The makefile in this project will lint or sometimes just format any shell,
Python, golang, Terraform, or Dockerfiles. The linters will only be run if
the makefile finds files with the appropriate file extension.

All of the linter checks are in the default make target, so you just have to
run

```
make -s
```

The -s is for 'silent'. Successful output looks like this

```
Running shellcheck
Running flake8
Running gofmt
Running terraform validate
Running hadolint on Dockerfiles
Test passed - Verified all file Apache 2 headers
```

The linters
are as follows:
* Shell - shellcheck. Can be found in homebrew
* Python - flake8. Can be installed with 'pip install flake8'
* Golang - gofmt. gofmt comes with the standard golang installation. golang
is a compiled language so there is no standard linter.
* Terraform - terraform has a built-in linter in the 'terraform validate'
command.
* Dockerfiles - hadolint. Can be found in homebrew
