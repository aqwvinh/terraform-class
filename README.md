# terraform-class

Personal notes

## Terraform modules
A Terraform module is a set of Terraform configuration files in a single directory. 
<br>Technically, a folder with a `.tf` file is a module.
<br>
<br>Even a simple configuration consisting of a single directory with one or more `.tf` files is a module (like `main.tf`, `variables.tf` and `outputs.tf`). 
<br>When you run Terraform commands directly from such a directory, it is considered the root module. So in this sense, every Terraform configuration is part of a module. 
<br>
<br>Terraform commands will only directly use the configuration files in one directory, which is usually the current working directory. However, your configuration can use module blocks to call modules in other directories. 
<br>When Terraform encounters a module block, it loads and processes that module's configuration files.
<br>A module that is called by another configuration is sometimes referred to as a "child module" of that configuration.
<br>
<br>**In many ways, Terraform modules are similar to the concepts of libraries/packages**

<br>You can create your own modules and store them in github

## Storing states
Store state per workspace in the same bucket.
<br>Never store state file in version control (likely to forget to push/pull down the latest changes, locking and secrets)

## Safely destroy
Alternative to `terraform destroy` can be to delete in the terraform files the resources and run `terraform apply` 

## Differences between locals and variables
Locals are useful to use when you want to give the result of an expression and then re-use that result throughout your configuration.
<br>Unlike variable values, local values can use dynamic expressions and resource arguments.

## Useful commands
`terraform show` displays the current state
<br>`terraform refresh` updates the current state (automatically done with `terraform apply`)

## Meta arguments
### for_each
We can create multiple similar resources with one block (like a for loop), with either map or set of string (set because we need unicity).
```
resource "google_compute_instance" "main" {
  for_each = {
    "name-1" = {vm_type = "e2-small", zone = "us-west1-a" }
    "name-2" = {vm_type = "e2-medium", zone = "us-west1-b" }
    "name-3" = {vm_type = "e2-small", zone = "us-west1-c" }
    }
   name         = each.key
   machine_type = each.value.vm_type
   zone         = each.value.zone
}
```

### dynamic
With `dynamic`, we create subblocks of the same type within a resource
```
variable "allow_list" {
  type = map(object({
    protocol = string
    ports    = list(string)

  }))

  default = {
    "allow1" = { protocol = "tcp", ports = ["80", "8080"] }
    "allow2" = { protocol = "tcp", ports = ["22"] }
    "allow3" = { protocol = "tcp", ports = ["443"] }
  }
}

resource "google_compute_firewall" "rules" {
  project = var.project
  name    = "my-firewall-rule"
  
  dynamic "allow" {
    for_each = var.allow_list
    content {
      protocol = allow.value.protocol
      ports    = allow.value.ports
    }
  }
}
```
