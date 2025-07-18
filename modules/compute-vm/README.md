# Google Compute Engine VM module

This module can operate in two distinct modes:

- instance creation, with optional unmanaged group
- instance template creation

In both modes, an optional service account can be created and assigned to either instances or template. If you need a managed instance group when using the module in template mode, refer to the [`compute-mig`](../compute-mig) module.

## Examples

<!-- BEGIN TOC -->
- [Examples](#examples)
  - [Instance using defaults](#instance-using-defaults)
  - [Service account management](#service-account-management)
    - [Compute default service account](#compute-default-service-account)
    - [Custom service account](#custom-service-account)
    - [Custom service account, auto created](#custom-service-account-auto-created)
    - [No service account](#no-service-account)
  - [Disk management](#disk-management)
    - [Disk sources](#disk-sources)
    - [Disk types and options](#disk-types-and-options)
    - [Boot disk as an independent resource](#boot-disk-as-an-independent-resource)
  - [Network interfaces](#network-interfaces)
    - [Internal and external IPs](#internal-and-external-ips)
    - [Using Alias IPs](#using-alias-ips)
    - [Using gVNIC](#using-gvnic)
    - [PSC interfaces](#psc-interfaces)
  - [Metadata](#metadata)
  - [IAM](#iam)
  - [Spot VM](#spot-vm)
  - [Confidential compute](#confidential-compute)
  - [Disk encryption with Cloud KMS](#disk-encryption-with-cloud-kms)
  - [Advanced machine features](#advanced-machine-features)
  - [Instance template](#instance-template)
    - [Global template](#global-template)
    - [Regional template](#regional-template)
  - [Instance group](#instance-group)
  - [Instance Schedule](#instance-schedule)
  - [Snapshot Schedules](#snapshot-schedules)
  - [Resource Manager Tags](#resource-manager-tags)
  - [Sole Tenancy](#sole-tenancy)
- [Variables](#variables)
- [Outputs](#outputs)
- [Fixtures](#fixtures)
<!-- END TOC -->

### Instance using defaults

The simplest example leverages defaults for the boot disk image and size, and uses a service account created by the module. Multiple instances can be managed via the `instance_count` variable.

```hcl
module "simple-vm-example" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "test"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
}
# tftest modules=1 resources=1 inventory=defaults.yaml e2e
```

### Service account management

VM service accounts can be managed in four different ways:

- in its default configuration, the module uses the Compute default service account with a basic set of scopes (`devstorage.read_only`, `logging.write`, `monitoring.write`)
- a custom service account can be used by passing its email in the `service_account.email` variable
- a custom service account can be created by the module and used by setting the `service_account.auto_create` variable to `true`
- the instance can be created with no service account by setting the `service_account` variable to `null`

Scopes for custom service accounts are set by default to `cloud-platform` and `userinfo.email`, and can be further customized regardless of which service account is used by directly setting the `service_account.scopes` variable.

#### Compute default service account

```hcl
module "vm-managed-sa-example" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "test1"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
}
# tftest inventory=sa-default.yaml e2e
```

#### Custom service account

```hcl
module "vm-managed-sa-example2" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "test2"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
  service_account = {
    email = module.iam-service-account.email
  }
}
# tftest inventory=sa-custom.yaml fixtures=fixtures/iam-service-account.tf e2e
```

#### Custom service account, auto created

```hcl
module "vm-managed-sa-example2" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "test2"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
  service_account = {
    auto_create = true
  }
}
# tftest inventory=sa-managed.yaml e2e
```

#### No service account

```hcl
module "vm-managed-sa-example2" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "test2"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
  service_account = null
}
# tftest inventory=sa-none.yaml e2e
```

### Disk management

#### Disk sources

Attached disks can be created and optionally initialized from a pre-existing source, or attached to VMs when pre-existing. The `source` and `source_type` attributes of the `attached_disks` variable allows several modes of operation:

- `source_type = "image"` can be used with zonal disks in instances and templates, set `source` to the image name or self link
- `source_type = "snapshot"` can be used with instances only, set `source` to the snapshot name or self link
- `source_type = "attach"` can be used for both instances and templates to attach an existing disk, set source to the name (for zonal disks) or self link (for regional disks) of the existing disk to attach; no disk will be created
- `source_type = null` can be used where an empty disk is needed, `source` becomes irrelevant and can be left null

This is an example of attaching a pre-existing regional PD to a new instance:

```hcl
module "vm-disks-example" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "test"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
  attached_disks = [{
    name        = "repd-1"
    size        = 10
    source_type = "attach"
    source      = "regions/${var.region}/disks/repd-test-1"
    options = {
      replica_zone = "${var.region}-c"
    }
  }]
  service_account = {
    auto_create = true
  }
}
# tftest modules=1 resources=2
```

And the same example for an instance template (where not using the full self link of the disk triggers recreation of the template)

```hcl
module "vm-disks-example" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "test"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
  attached_disks = [{
    name        = "repd"
    size        = 10
    source_type = "attach"
    source      = "https://www.googleapis.com/compute/v1/projects/${var.project_id}/regions/${var.region}/disks/repd-test-1"
    options = {
      replica_zone = "${var.region}-c"
    }
  }]
  service_account = {
    auto_create = true
  }
  create_template = {}
}
# tftest modules=1 resources=2
```

#### Disk types and options

The `attached_disks` variable exposes an `option` attribute that can be used to fine tune the configuration of each disk. The following example shows a VM with multiple disks

```hcl
module "vm-disk-options-example" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "test"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
  attached_disks = [
    {
      name        = "data1"
      size        = "10"
      source_type = "image"
      source      = "image-1"
      options = {
        auto_delete  = false
        replica_zone = "${var.region}-c"
      }
    },
    {
      name        = "data2"
      size        = "20"
      source_type = "snapshot"
      source      = "snapshot-2"
      options = {
        type = "pd-ssd"
        mode = "READ_ONLY"
      }
    }
  ]
  service_account = {
    auto_create = true
  }
}
# tftest inventory=disk-options.yaml
```

#### Boot disk as an independent resource

To create the boot disk as an independent resources instead of as part of the instance creation flow, set `boot_disk.use_independent_disk` to `true` and optionally configure `boot_disk.initialize_params`.

This will create the boot disk as its own resource and attach it to the instance, allowing to recreate the instance from Terraform while preserving the boot disk.

```hcl
module "simple-vm-example" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "test"
  boot_disk = {
    initialize_params    = {}
    use_independent_disk = true
  }
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
  service_account = {
    auto_create = true
  }
}
# tftest inventory=independent-boot-disk.yaml e2e
```

### Network interfaces

#### Internal and external IPs

By default VNs are create with an automatically assigned IP addresses, but you can change it through the `addresses` and `nat` attributes of the `network_interfaces` variable:

```hcl
module "vm-internal-ip" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "vm-internal-ip"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
    addresses  = { internal = "10.0.0.2" }
  }]
}

module "vm-external-ip" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "vm-external-ip"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
    nat        = true
    addresses  = { external = "8.8.8.8" }
  }]
}
# tftest inventory=ips.yaml
```

#### Using Alias IPs

This example shows how to add additional [Alias IPs](https://cloud.google.com/vpc/docs/alias-ip) to your VM. `alias_ips` is a map of subnetwork additional range name into IP address.

```hcl
module "vm-with-alias-ips" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "test"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
    alias_ips = {
      services = "100.71.1.123/32"
    }
  }]
}
# tftest inventory=alias-ips.yaml e2e
```

#### Using gVNIC

This example shows how to enable [gVNIC](https://cloud.google.com/compute/docs/networking/using-gvnic) on your VM by customizing a `cos` image. Given that gVNIC needs to be enabled as an instance configuration and as a guest os configuration, you'll need to supply a bootable disk with `guest_os_features=GVNIC`. `SEV_CAPABLE`, `UEFI_COMPATIBLE` and `VIRTIO_SCSI_MULTIQUEUE` are enabled implicitly in the `cos`, `rhel`, `centos` and other images.

Note: most recent Google-provided images do enable `GVNIC` and no custom image is necessary.

```hcl
resource "google_compute_image" "cos-gvnic" {
  project      = var.project_id
  name         = "my-image"
  source_image = "https://www.googleapis.com/compute/v1/projects/cos-cloud/global/images/cos-89-16108-534-18"

  guest_os_features {
    type = "GVNIC"
  }
  guest_os_features {
    type = "SEV_CAPABLE"
  }
  guest_os_features {
    type = "UEFI_COMPATIBLE"
  }
  guest_os_features {
    type = "VIRTIO_SCSI_MULTIQUEUE"
  }
}

module "vm-with-gvnic" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "test"
  boot_disk = {
    initialize_params = {
      image = google_compute_image.cos-gvnic.self_link
      type  = "pd-ssd"
    }
  }
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
    nic_type   = "GVNIC"
  }]
  service_account = {
    auto_create = true
  }
}
# tftest inventory=gvnic.yaml
```

#### PSC interfaces

[Private Service Connect interfaces](https://cloud.google.com/vpc/docs/about-private-service-connect-interfaces) can be configured via the `network_attached_interfaces` variable, which is a simple list of network attachment ids, one per interface. PSC interfaces will be defined after regular interfaces.

```hcl

# create the network attachment from a service project
module "net-attachment" {
  source     = "./fabric/modules/net-address"
  project_id = var.project_id
  network_attachments = {
    svc-0 = {
      subnet_self_link      = module.vpc.subnet_self_links["${var.region}/ipv6-internal"]
      producer_accept_lists = [var.project_id]
    }
  }
}

module "vm-psc-interface" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "vm-internal-ip"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
  network_attached_interfaces = [
    module.net-attachment.network_attachment_ids["svc-0"]
  ]
}
# tftest fixtures=fixtures/net-vpc-ipv6.tf e2e
```

### Metadata

You can define labels and custom metadata values. Metadata can be leveraged, for example, to define a custom startup script.

```hcl
module "vm-metadata-example" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "nginx-server"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
  labels = {
    env    = "dev"
    system = "crm"
  }
  metadata = {
    startup-script = <<-EOF
      #! /bin/bash
      apt-get update
      apt-get install -y nginx
    EOF
  }
  service_account = {
    auto_create = true
  }
}
# tftest inventory=metadata.yaml e2e
```

### IAM

Like most modules, you can assign IAM roles to the instance using the `iam` variable.

```hcl
module "vm-iam-example" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "webserver"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
  iam = {
    "roles/compute.instanceAdmin" = [
      "group:${var.group_email}",
    ]
  }
}
# tftest inventory=iam.yaml e2e

```

### Spot VM

[Spot VMs](https://cloud.google.com/compute/docs/instances/spot) are ephemeral compute instances suitable for batch jobs and fault-tolerant workloads. Spot VMs provide new features that [preemptible instances](https://cloud.google.com/compute/docs/instances/preemptible) do not support, such as the absence of a maximum runtime.

```hcl
module "spot-vm-example" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "test"
  options = {
    spot               = true
    termination_action = "STOP"
  }
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
}
# tftest inventory=spot.yaml e2e
```

### Confidential compute

You can enable confidential compute with the `confidential_compute` variable, which can be used for standalone instances or for instance templates.

```hcl
module "vm-confidential-example" {
  source               = "./fabric/modules/compute-vm"
  project_id           = var.project_id
  zone                 = "${var.region}-b"
  name                 = "confidential-vm"
  confidential_compute = true
  instance_type        = "n2d-standard-2"
  boot_disk = {
    initialize_params = {
      image = "projects/debian-cloud/global/images/family/debian-12"
    }
  }
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
}

module "template-confidential-example" {
  source               = "./fabric/modules/compute-vm"
  project_id           = var.project_id
  zone                 = "${var.region}-b"
  name                 = "confidential-template"
  confidential_compute = true
  create_template      = {}
  instance_type        = "n2d-standard-2"
  boot_disk = {
    initialize_params = {
      image = "projects/debian-cloud/global/images/family/debian-12"
    }
  }
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
}

# tftest inventory=confidential.yaml e2e
```

### Disk encryption with Cloud KMS

This example shows how to control disk encryption via the the `encryption` variable, in this case the self link to a KMS CryptoKey that will be used to encrypt boot and attached disk. Managing the key with the `../kms` module is of course possible, but is not shown here.

```hcl
module "project" {
  source          = "./fabric/modules/project"
  name            = "gce"
  billing_account = var.billing_account_id
  prefix          = var.prefix
  parent          = var.folder_id
  services = [
    "cloudkms.googleapis.com",
    "compute.googleapis.com",
  ]
}

module "kms" {
  source     = "./fabric/modules/kms"
  project_id = module.project.project_id
  keyring = {
    location = var.region
    name     = "${var.prefix}-keyring"
  }
  keys = {
    "key-regional" = {
    }
  }
  iam = {
    "roles/cloudkms.cryptoKeyEncrypterDecrypter" = [
      module.project.service_agents.compute.iam_email
    ]
  }
}

module "vpc" {
  source     = "./fabric/modules/net-vpc"
  project_id = module.project.project_id
  name       = "my-network"
  subnets = [
    {
      ip_cidr_range = "10.0.0.0/24"
      name          = "production"
      region        = var.region
    },
  ]
}

module "kms-vm-example" {
  source     = "./fabric/modules/compute-vm"
  project_id = module.project.project_id
  zone       = "${var.region}-b"
  name       = "kms-test"
  network_interfaces = [{
    network    = module.vpc.self_link
    subnetwork = module.vpc.subnet_self_links["${var.region}/production"]
  }]
  attached_disks = [{
    name = "attached-disk"
    size = 10
  }]
  service_account = {
    auto_create = true
  }
  encryption = {
    encrypt_boot      = true
    kms_key_self_link = module.kms.keys.key-regional.id
  }
}
# tftest inventory=cmek.yaml e2e
```

### Advanced machine features

Advanced machine features can be configured via the `options.advanced_machine_features` variable.

```hcl
module "simple-vm-example" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "test"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
  options = {
    advanced_machine_features = {
      enable_nested_virtualization = true
      enable_turbo_mode            = true
      threads_per_core             = 2
    }
  }
}
# tftest modules=1 resources=1
```

### Instance template

#### Global template

This example shows how to use the module to manage an instance template that defines an additional attached disk for each instance, and overrides defaults for the boot disk image and service account. Instance templates are created global by default.

```hcl
module "cos-test" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "test"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
  boot_disk = {
    initialize_params = {
      image = "projects/cos-cloud/global/images/family/cos-stable"
    }
  }
  attached_disks = [
    { size = 10 }
  ]
  service_account = {
    email = module.iam-service-account.email
  }
  create_template = {}
}
# tftest inventory=template.yaml fixtures=fixtures/iam-service-account.tf e2e
```

#### Regional template

A regional template can be created by setting `var.create_template.regional`.

```hcl
module "cos-test" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "test"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
  boot_disk = {
    initialize_params = {
      image = "projects/cos-cloud/global/images/family/cos-stable"
    }
  }
  attached_disks = [
    { size = 10 }
  ]
  service_account = {
    email = module.iam-service-account.email
  }
  create_template = {
    regional = true
  }
}
# tftest inventory=template-regional.yaml fixtures=fixtures/iam-service-account.tf
```

### Instance group

If an instance group is needed when operating in instance mode, simply set the `group` variable to a non null map. The map can contain named port declarations, or be empty if named ports are not needed.

```hcl
locals {
  cloud_config = "my cloud config"
}

module "instance-group" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "ilb-test"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
  boot_disk = {
    initialize_params = {
      image = "projects/cos-cloud/global/images/family/cos-stable"
    }
  }
  service_account = {
    email  = var.service_account.email
    scopes = ["https://www.googleapis.com/auth/cloud-platform"]
  }
  metadata = {
    user-data = local.cloud_config
  }
  group = { named_ports = {} }
}
# tftest inventory=group.yaml e2e
```

### Instance Schedule

Instance start and stop schedules can be defined via an existing or auto-created resource policy. This functionality requires [additional permissions on Compute Engine Service Agent](https://cloud.google.com/compute/docs/instances/schedule-instance-start-stop#service_agent_required_roles)

To use an existing policy pass its id to the `instance_schedule` variable:

```hcl
module "instance" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "schedule-test"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
  boot_disk = {
    initialize_params = {
      image = "projects/cos-cloud/global/images/family/cos-stable"
    }
  }
  instance_schedule = {
    resource_policy_id = "projects/${var.project_id}/regions/${var.region}/resourcePolicies/test"
  }
}
# tftest inventory=instance-schedule-id.yaml
```

To create a new policy set its configuration in the `instance_schedule` variable. When removing the policy follow a two-step process by first setting `active = false` in the schedule configuration, which will unattach the policy, then removing the variable so the policy is destroyed.

```hcl
module "project" {
  source = "./fabric/modules/project"
  name   = var.project_id
  project_reuse = {
    use_data_source = false
    project_attributes = {
      name             = var.project_id
      number           = var.project_number
      services_enabled = ["compute.googleapis.com"]
    }
  }
  iam_bindings_additive = {
    compute-admin-service-agent = {
      member = module.project.service_agents["compute"].iam_email
      role   = "roles/compute.instanceAdmin.v1"
    }
  }
}

module "instance" {
  source     = "./fabric/modules/compute-vm"
  project_id = module.project.project_id
  zone       = "${var.region}-b"
  name       = "schedule-test"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
  boot_disk = {
    initialize_params = {
      image = "projects/cos-cloud/global/images/family/cos-stable"
    }
  }
  instance_schedule = {
    create_config = {
      vm_start = "0 8 * * *"
      vm_stop  = "0 17 * * *"
    }
  }
  depends_on = [module.project] # ensure that grants are complete before creating schedule / instance
}
# tftest inventory=instance-schedule-create.yaml e2e
```

### Snapshot Schedules

Snapshot policies can be attached to disks with optional creation managed by the module.

```hcl
module "instance" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "schedule-test"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
  boot_disk = {
    initialize_params = {
      image = "projects/cos-cloud/global/images/family/cos-stable"
    }
    snapshot_schedule = ["boot"]
  }
  attached_disks = [
    {
      name              = "disk-1"
      size              = 10
      snapshot_schedule = ["boot"]
    }
  ]
  snapshot_schedules = {
    boot = {
      schedule = {
        daily = {
          days_in_cycle = 1
          start_time    = "03:00"
        }
      }
    }
  }
}
# tftest inventory=snapshot-schedule-create.yaml e2e
```

### Resource Manager Tags

Resource manager tags bindings for use in IAM or org policy conditions are supported via three different variables:

- `network_tag_bindings` associates tags to instances after creation, and is meant for use with network firewall policies
- `tag_bindings` associates tags to instances and disks after creation, and is meant for use with IAM or organization policy conditions
- `tag_bindings_immutable` associates tags to instances and disks during the instance or template creation flow; these bindings are immutable and changes trigger resource recreation

The non-immutable variables follow our usual interface for tag bindings, and support specifying a map with arbitrary keys mapping to tag key or value ids. To prevent a provider permadiff also pass in the project number in the `project_number` variable.

The immutable variable uses a different format enforced by the Compute API, where keys need to be tag key ids, and values tag value ids.

This is an example of setting non-immutable tag bindings:

```hcl
module "simple-vm-example" {
  source         = "./fabric/modules/compute-vm"
  project_id     = var.project_id
  project_number = 12345678
  zone           = "${var.region}-b"
  name           = "test"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
  tag_bindings = {
    dev = "tagValues/1234567890"
  }
}
# tftest modules=1 resources=2
```

This example uses immutable tag bindings, and will trigger recreation if those are changed.

```hcl
module "simple-vm-example" {
  source     = "./fabric/modules/compute-vm"
  project_id = var.project_id
  zone       = "${var.region}-b"
  name       = "test"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
  tag_bindings_immutable = {
    "tagKeys/1234567890" = "tagValues/7890123456"
  }
}
# tftest inventory=tag-bindings.yaml
```

### Sole Tenancy

You can add node affinities (and anti-affinity) configurations to allocate the VM on sole tenant nodes.

```hcl
module "sole-tenancy" {
  source        = "./fabric/modules/compute-vm"
  project_id    = var.project_id
  zone          = "${var.region}-b"
  instance_type = "n1-standard-1"
  name          = "test"
  network_interfaces = [{
    network    = var.vpc.self_link
    subnetwork = var.subnet.self_link
  }]
  options = {
    node_affinities = {
      workload = {
        values = ["frontend"]
      }
      cpu = {
        in     = false
        values = ["c3"]
      }
    }
  }
}
# tftest inventory=sole-tenancy.yaml
```
<!-- BEGIN TFDOC -->
## Variables

| name | description | type | required | default |
|---|---|:---:|:---:|:---:|
| [name](variables.tf#L273) | Instance name. | <code>string</code> | ✓ |  |
| [network_interfaces](variables.tf#L285) | Network interfaces configuration. Use self links for Shared VPC, set addresses to null if not needed. | <code title="list&#40;object&#40;&#123;&#10;  network    &#61; string&#10;  subnetwork &#61; string&#10;  alias_ips  &#61; optional&#40;map&#40;string&#41;, &#123;&#125;&#41;&#10;  nat        &#61; optional&#40;bool, false&#41;&#10;  nic_type   &#61; optional&#40;string&#41;&#10;  stack_type &#61; optional&#40;string&#41;&#10;  addresses &#61; optional&#40;object&#40;&#123;&#10;    internal &#61; optional&#40;string&#41;&#10;    external &#61; optional&#40;string&#41;&#10;  &#125;&#41;, null&#41;&#10;  network_tier &#61; optional&#40;string&#41;&#10;&#125;&#41;&#41;">list&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> | ✓ |  |
| [project_id](variables.tf#L370) | Project id. | <code>string</code> | ✓ |  |
| [zone](variables.tf#L483) | Compute zone. | <code>string</code> | ✓ |  |
| [attached_disk_defaults](variables.tf#L17) | Defaults for attached disks options. | <code title="object&#40;&#123;&#10;  auto_delete  &#61; optional&#40;bool, false&#41;&#10;  mode         &#61; string&#10;  replica_zone &#61; string&#10;  type         &#61; string&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code title="&#123;&#10;  auto_delete  &#61; true&#10;  mode         &#61; &#34;READ_WRITE&#34;&#10;  replica_zone &#61; null&#10;  type         &#61; &#34;pd-balanced&#34;&#10;&#125;">&#123;&#8230;&#125;</code> |
| [attached_disks](variables.tf#L37) | Additional disks, if options is null defaults will be used in its place. Source type is one of 'image' (zonal disks in vms and template), 'snapshot' (vm), 'existing', and null. | <code title="list&#40;object&#40;&#123;&#10;  name        &#61; optional&#40;string&#41;&#10;  device_name &#61; optional&#40;string&#41;&#10;  size              &#61; string&#10;  snapshot_schedule &#61; optional&#40;list&#40;string&#41;&#41;&#10;  source            &#61; optional&#40;string&#41;&#10;  source_type       &#61; optional&#40;string&#41;&#10;  options &#61; optional&#40;&#10;    object&#40;&#123;&#10;      auto_delete  &#61; optional&#40;bool, false&#41;&#10;      mode         &#61; optional&#40;string, &#34;READ_WRITE&#34;&#41;&#10;      replica_zone &#61; optional&#40;string&#41;&#10;      type         &#61; optional&#40;string, &#34;pd-balanced&#34;&#41;&#10;    &#125;&#41;,&#10;    &#123;&#10;      auto_delete  &#61; true&#10;      mode         &#61; &#34;READ_WRITE&#34;&#10;      replica_zone &#61; null&#10;      type         &#61; &#34;pd-balanced&#34;&#10;    &#125;&#10;  &#41;&#10;&#125;&#41;&#41;">list&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> |  | <code>&#91;&#93;</code> |
| [boot_disk](variables.tf#L82) | Boot disk properties. | <code title="object&#40;&#123;&#10;  auto_delete       &#61; optional&#40;bool, true&#41;&#10;  snapshot_schedule &#61; optional&#40;list&#40;string&#41;&#41;&#10;  source            &#61; optional&#40;string&#41;&#10;  initialize_params &#61; optional&#40;object&#40;&#123;&#10;    image &#61; optional&#40;string, &#34;projects&#47;debian-cloud&#47;global&#47;images&#47;family&#47;debian-11&#34;&#41;&#10;    size  &#61; optional&#40;number, 10&#41;&#10;    type  &#61; optional&#40;string, &#34;pd-balanced&#34;&#41;&#10;  &#125;&#41;, &#123;&#125;&#41;&#10;  use_independent_disk &#61; optional&#40;bool, false&#41;&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code title="&#123;&#10;  initialize_params &#61; &#123;&#125;&#10;&#125;">&#123;&#8230;&#125;</code> |
| [can_ip_forward](variables.tf#L116) | Enable IP forwarding. | <code>bool</code> |  | <code>false</code> |
| [confidential_compute](variables.tf#L122) | Enable Confidential Compute for these instances. | <code>bool</code> |  | <code>false</code> |
| [create_template](variables.tf#L128) | Create instance template instead of instances. Defaults to a global template. | <code title="object&#40;&#123;&#10;  regional &#61; optional&#40;bool, false&#41;&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code>null</code> |
| [description](variables.tf#L137) | Description of a Compute Instance. | <code>string</code> |  | <code>&#34;Managed by the compute-vm Terraform module.&#34;</code> |
| [enable_display](variables.tf#L143) | Enable virtual display on the instances. | <code>bool</code> |  | <code>false</code> |
| [encryption](variables.tf#L149) | Encryption options. Only one of kms_key_self_link and disk_encryption_key_raw may be set. If needed, you can specify to encrypt or not the boot disk. | <code title="object&#40;&#123;&#10;  encrypt_boot            &#61; optional&#40;bool, false&#41;&#10;  disk_encryption_key_raw &#61; optional&#40;string&#41;&#10;  kms_key_self_link       &#61; optional&#40;string&#41;&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code>null</code> |
| [gpu](variables.tf#L159) | GPU information. Based on https://cloud.google.com/compute/docs/gpus. | <code title="object&#40;&#123;&#10;  count &#61; number&#10;  type  &#61; string&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code>null</code> |
| [group](variables.tf#L194) | Define this variable to create an instance group for instances. Disabled for template use. | <code title="object&#40;&#123;&#10;  named_ports &#61; map&#40;number&#41;&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code>null</code> |
| [hostname](variables.tf#L202) | Instance FQDN name. | <code>string</code> |  | <code>null</code> |
| [iam](variables.tf#L208) | IAM bindings in {ROLE => [MEMBERS]} format. | <code>map&#40;list&#40;string&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [instance_schedule](variables.tf#L214) | Assign or create and assign an instance schedule policy. Either resource policy id or create_config must be specified if not null. Set active to null to dtach a policy from vm before destroying. | <code title="object&#40;&#123;&#10;  resource_policy_id &#61; optional&#40;string&#41;&#10;  create_config &#61; optional&#40;object&#40;&#123;&#10;    active          &#61; optional&#40;bool, true&#41;&#10;    description     &#61; optional&#40;string&#41;&#10;    expiration_time &#61; optional&#40;string&#41;&#10;    start_time      &#61; optional&#40;string&#41;&#10;    timezone        &#61; optional&#40;string, &#34;UTC&#34;&#41;&#10;    vm_start        &#61; optional&#40;string&#41;&#10;    vm_stop         &#61; optional&#40;string&#41;&#10;  &#125;&#41;&#41;&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code>null</code> |
| [instance_type](variables.tf#L249) | Instance type. | <code>string</code> |  | <code>&#34;f1-micro&#34;</code> |
| [labels](variables.tf#L255) | Instance labels. | <code>map&#40;string&#41;</code> |  | <code>&#123;&#125;</code> |
| [metadata](variables.tf#L261) | Instance metadata. | <code>map&#40;string&#41;</code> |  | <code>&#123;&#125;</code> |
| [min_cpu_platform](variables.tf#L267) | Minimum CPU platform. | <code>string</code> |  | <code>null</code> |
| [network_attached_interfaces](variables.tf#L278) | Network interfaces using network attachments. | <code>list&#40;string&#41;</code> |  | <code>&#91;&#93;</code> |
| [network_tag_bindings](variables.tf#L306) | Resource manager tag bindings in arbitrary key => tag key or value id format. Set on both the instance only for networking purposes, and modifiable without impacting the main resource lifecycle. | <code>map&#40;string&#41;</code> |  | <code>&#123;&#125;</code> |
| [options](variables.tf#L313) | Instance options. | <code title="object&#40;&#123;&#10;  advanced_machine_features &#61; optional&#40;object&#40;&#123;&#10;    enable_nested_virtualization &#61; optional&#40;bool&#41;&#10;    enable_turbo_mode            &#61; optional&#40;bool&#41;&#10;    enable_uefi_networking       &#61; optional&#40;bool&#41;&#10;    performance_monitoring_unit  &#61; optional&#40;string&#41;&#10;    threads_per_core             &#61; optional&#40;number&#41;&#10;    visible_core_count           &#61; optional&#40;number&#41;&#10;  &#125;&#41;&#41;&#10;  allow_stopping_for_update &#61; optional&#40;bool, true&#41;&#10;  deletion_protection       &#61; optional&#40;bool, false&#41;&#10;  graceful_shutdown &#61; optional&#40;object&#40;&#123;&#10;    enabled           &#61; optional&#40;bool, false&#41;&#10;    max_duration_secs &#61; optional&#40;number&#41;&#10;  &#125;&#41;&#41;&#10;  max_run_duration &#61; optional&#40;object&#40;&#123;&#10;    nanos   &#61; optional&#40;number&#41;&#10;    seconds &#61; number&#10;  &#125;&#41;&#41;&#10;  node_affinities &#61; optional&#40;map&#40;object&#40;&#123;&#10;    values &#61; list&#40;string&#41;&#10;    in     &#61; optional&#40;bool, true&#41;&#10;  &#125;&#41;&#41;, &#123;&#125;&#41;&#10;  spot               &#61; optional&#40;bool, false&#41;&#10;  termination_action &#61; optional&#40;string&#41;&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code title="&#123;&#10;  allow_stopping_for_update &#61; true&#10;  deletion_protection       &#61; false&#10;  spot                      &#61; false&#10;  termination_action        &#61; null&#10;&#125;">&#123;&#8230;&#125;</code> |
| [project_number](variables.tf#L375) | Project number. Used in tag bindings to avoid a permadiff. | <code>string</code> |  | <code>null</code> |
| [scratch_disks](variables.tf#L381) | Scratch disks configuration. | <code title="object&#40;&#123;&#10;  count     &#61; number&#10;  interface &#61; string&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code title="&#123;&#10;  count     &#61; 0&#10;  interface &#61; &#34;NVME&#34;&#10;&#125;">&#123;&#8230;&#125;</code> |
| [service_account](variables.tf#L393) | Service account email and scopes. If email is null, the default Compute service account will be used unless auto_create is true, in which case a service account will be created. Set the variable to null to avoid attaching a service account. | <code title="object&#40;&#123;&#10;  auto_create &#61; optional&#40;bool, false&#41;&#10;  email       &#61; optional&#40;string&#41;&#10;  scopes      &#61; optional&#40;list&#40;string&#41;&#41;&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code>&#123;&#125;</code> |
| [shielded_config](variables.tf#L403) | Shielded VM configuration of the instances. | <code title="object&#40;&#123;&#10;  enable_secure_boot          &#61; bool&#10;  enable_vtpm                 &#61; bool&#10;  enable_integrity_monitoring &#61; bool&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code>null</code> |
| [snapshot_schedules](variables.tf#L413) | Snapshot schedule resource policies that can be attached to disks. | <code title="map&#40;object&#40;&#123;&#10;  schedule &#61; object&#40;&#123;&#10;    daily &#61; optional&#40;object&#40;&#123;&#10;      days_in_cycle &#61; number&#10;      start_time    &#61; string&#10;    &#125;&#41;&#41;&#10;    hourly &#61; optional&#40;object&#40;&#123;&#10;      hours_in_cycle &#61; number&#10;      start_time     &#61; string&#10;    &#125;&#41;&#41;&#10;    weekly &#61; optional&#40;list&#40;object&#40;&#123;&#10;      day        &#61; string&#10;      start_time &#61; string&#10;    &#125;&#41;&#41;&#41;&#10;  &#125;&#41;&#10;  description &#61; optional&#40;string&#41;&#10;  retention_policy &#61; optional&#40;object&#40;&#123;&#10;    max_retention_days         &#61; number&#10;    on_source_disk_delete_keep &#61; optional&#40;bool&#41;&#10;  &#125;&#41;&#41;&#10;  snapshot_properties &#61; optional&#40;object&#40;&#123;&#10;    chain_name        &#61; optional&#40;string&#41;&#10;    guest_flush       &#61; optional&#40;bool&#41;&#10;    labels            &#61; optional&#40;map&#40;string&#41;&#41;&#10;    storage_locations &#61; optional&#40;list&#40;string&#41;&#41;&#10;  &#125;&#41;&#41;&#10;&#125;&#41;&#41;">map&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [tag_bindings](variables.tf#L456) | Resource manager tag bindings in arbitrary key => tag key or value id format. Set on both the instance and zonal disks, and modifiable without impacting the main resource lifecycle. | <code>map&#40;string&#41;</code> |  | <code>&#123;&#125;</code> |
| [tag_bindings_immutable](variables.tf#L463) | Immutable resource manager tag bindings, in tagKeys/id => tagValues/id format. These are set on the instance or instance template at creation time, and trigger recreation if changed. | <code>map&#40;string&#41;</code> |  | <code>null</code> |
| [tags](variables.tf#L477) | Instance network tags for firewall rule targets. | <code>list&#40;string&#41;</code> |  | <code>&#91;&#93;</code> |

## Outputs

| name | description | sensitive |
|---|---|:---:|
| [external_ip](outputs.tf#L17) | Instance main interface external IP addresses. |  |
| [group](outputs.tf#L26) | Instance group resource. |  |
| [id](outputs.tf#L31) | Fully qualified instance id. |  |
| [instance](outputs.tf#L36) | Instance resource. | ✓ |
| [internal_ip](outputs.tf#L42) | Instance main interface internal IP address. |  |
| [internal_ips](outputs.tf#L50) | Instance interfaces internal IP addresses. |  |
| [login_command](outputs.tf#L58) | Command to SSH into the machine. |  |
| [self_link](outputs.tf#L63) | Instance self links. |  |
| [service_account](outputs.tf#L68) | Service account resource. |  |
| [service_account_email](outputs.tf#L73) | Service account email. |  |
| [service_account_iam_email](outputs.tf#L78) | Service account email. |  |
| [template](outputs.tf#L87) | Template resource. |  |
| [template_name](outputs.tf#L96) | Template name. |  |

## Fixtures

- [iam-service-account.tf](../../tests/fixtures/iam-service-account.tf)
- [net-vpc-ipv6.tf](../../tests/fixtures/net-vpc-ipv6.tf)
<!-- END TFDOC -->
