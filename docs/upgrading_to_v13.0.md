# Upgrading to v13.0

The v13.0 release of *kubernetes-engine* is a backwards incompatible
release.

### `kubernetes` provider removed from the module

- `kubernetes` provider has been removed across all modules/submodules and need to be specified in the calling module.

To leverage Terraform v0.13 features such as custom variable validation and using `count`, `for_each` or `depends_on` in modules,
it is [required](https://www.terraform.io/docs/modules/providers.html#legacy-shared-modules-with-provider-configurations) that
a module does not contain any nested provider configuration and receives all of its provider configurations from the calling
module. This release adapts to this requirement.

```diff
+  data "google_client_config" "default" {}

+  provider "kubernetes" {
+    host                   = "https://${module.gke.endpoint}"
+    token                  = data.google_client_config.default.access_token
+    cluster_ca_certificate = base64decode(module.gke.ca_certificate)
+  }

 module "gke" {
   source                  = "terraform-google-modules/kubernetes-engine/google"
-  version                 = "~> 12.0"
+  version                 = "~> 13.0"
}
```

### ACM submodule `local_file` removed

[ACM submodule](https://github.com/terraform-google-modules/terraform-google-kubernetes-engine/tree/master/modules/acm) no longer creates a local file called `operator_cr.yaml`.
The yaml contents are rendered dynamically and passed via STDIN which fixes errors due to `operator_cr.yaml` file not being present between ephemeral pipeline runs.

This is destructive and will result in deletion and recreation of the ACM operator.

### Wait for cluster script removed

[wait-for-cluster.sh](https://github.com/terraform-google-modules/terraform-google-kubernetes-engine/blob/2c4a2b11b9be01c392c9d3a0c5c720973dbffebf/cluster.tf#L323) used to ensure that the cluster is in a ready state has been removed due to [improvements](https://github.com/terraform-google-modules/terraform-google-kubernetes-engine/issues/800) in the provider. As part of the upgrade, Terraform might indicate that the `wait_for_cluster` null_resource must be destroyed. This is no-op and can be safely applied:

```
  # module.example.module.gke.module.gcloud_wait_for_cluster.null_resource.module_depends_on[0] will be destroyed
  - resource "null_resource" "module_depends_on" {
      - id       = "8092231570921454387" -> null
      - triggers = {
          - "value" = "2"
        } -> null
    }

  # module.example.module.gke.module.gcloud_wait_for_cluster.null_resource.run_command[0] will be destroyed
  - resource "null_resource" "run_command" {
      - id       = "5198854261759708357" -> null
      - triggers = {
          - "arguments"             = "de7f9e59f7436cdec4c8fc34fb8e9833"
          - "create_cmd_body"       = "PROJECT_ID CLUSTER_NAME CLUSTER_LOCATION "
          - "create_cmd_entrypoint" = "../../../scripts/wait-for-cluster.sh"
          - "gcloud_bin_abs_path"   = "/google-cloud-sdk/bin"
          - "md5"                   = "4149df8255e42ea6781ec241e82e9e69"
        } -> null
    }

  # module.example.module.gke.module.gcloud_wait_for_cluster.null_resource.run_destroy_command[0] will be destroyed
  - resource "null_resource" "run_destroy_command" {
      - id       = "6787683359757867074" -> null
      - triggers = {
          - "destroy_cmd_body"       = "PROJECT_ID CLUSTER_NAME CLUSTER_LOCATION "
          - "destroy_cmd_entrypoint" = "../../../scripts/wait-for-cluster.sh"
          - "gcloud_bin_abs_path"    = "/google-cloud-sdk/bin"
        } -> null
    }

Plan: 0 to add, 0 to change, 3 to destroy.
```
