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

Below is an example of how to use a module. The module is a folder named `modules/my_module`, we just need to specify the variables to define the module 
```
module "compute_engine" {
  source = "./modules/my_module"

  ce_name                 = "compute-engine-west"
  machine_type            = var.machine_type
  zone                    = var.zone_west1
  image                   = var.image
  metadata_startup_script = "${path.module}/startup.sh"
  vpc                     = google_compute_network.vpc.self_link
  subnet                  = google_compute_subnetwork.subnet-west1.self_link
  static_ip_name          = "static-ip-west"
  region                  = var.region-west

}
```

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

locals {
  compute-engines = {
    "compute-engine-west"  = { region = "us-west1", zone = "us-west1-a", subnet = google_compute_subnetwork.subnet-west1.name, machine_type = "f1-micro", image = "debian-cloud/debian-11", static_ip_name = "static-ip-west" },
    "compute-engine-east"  = { region = "us-east1", zone = "us-east1-b", subnet = google_compute_subnetwork.subnet-east1.name, machine_type = "f1-micro", image = "debian-cloud/debian-11", static_ip_name = "static-ip-east" },
    "compute-engine-west1" = { region = "us-west1", zone = "us-west1-a", subnet = google_compute_subnetwork.subnet-west1.name, machine_type = "f1-micro", image = "debian-cloud/debian-11", static_ip_name = "static-ip-west1" }
  }
}

module "configured_compute_engine" {
  source                  = "./modules/my-module"
  for_each                = local.compute-engines
  ce_name                 = each.key
  machine_type            = each.value.machine_type
  zone                    = each.value.zone
  image                   = each.value.image
  metadata_startup_script = var.metadata_startup_script
  vpc                     = google_compute_network.vpc.self_link
  static_ip_name          = each.value.static_ip_name
  region                  = each.value.region
  subnet                  = each.value.subnet

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

### Use startup scripts for compute engines
Use `metadata_startup_script` to pass a `.sh` script for instructions when creating a compute engine

### Assign a static external IP address
Reserve a static IP address and assign it to the VM

```
# reserve static IP address
resource "google_compute_address" "default" {
  name   = "my-test-static-ip-address"
  region = "us-central1"
}

# assign it to a VM
resource "google_compute_instance" "default" {
  name         = "dns-proxy-nfs"
  machine_type = "n1-standard-1"
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      image = "ubuntu-1404-trusty-v20160627"
    }
  }

  network_interface {
    network = "default"
    access_config {
      nat_ip = google_compute_address.default.address
    }
  }
}
```

### Best practices
- Enable API in module and set `disable_services_on_destroy` to `False`
```
module project-services {
 source  = "terraform-google-modules/project-factory/google//modules/project_services"
 version = "~> 14.0"
 project_id  = var.project
 enable_apis = var.enable_apis
 activate_apis = [
   		  "cloudresourcemanager.googleapis.com",
        "servicenetworking.googleapis.com",
        "sql-component.googleapis.com",
        "sqladmin.googleapis.com",
        "redis.googleapis.com"
]
disable_services_on_destroy = false
}
```
- Protect stateful resource
- Don't declare providers or backends in module
- Use only remote state
