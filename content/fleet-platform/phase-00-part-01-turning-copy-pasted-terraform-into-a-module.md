---
title: "Phase 0, Part 1: Turning Copy-Pasted Terraform Into a Reusable Proxmox Module"
date: 2026-09-22
series: ["fleet-platform"]
tags: ["terraform", "proxmox", "lxc", "cloud-init", "git-subtree", "dns"]
phase: "0"
summary: "Refactoring duplicated Proxmox VM and LXC declarations into a shared module, centralizing network outputs, and using git subtree for offline disaster recovery."
draft: false
---

My original infrastructure is three bare-metal Proxmox machines clustered together. The clustering just makes administration a bit easier. Those run the VMs that hold my k3s nodes and other lab systems like an Ansible Automation Platform. All of it was hand-configured.

My goal is to have the foundation infrastructure come up from code too, the same way the cluster does, so I can add machines or spin everything down and back up without clicking through a UI. It's also a good exercise on its own. This piece is the DNS setup using Technitium.

---

## Why an LXC Container for DNS

The plan for DNS is to spin up an LXC container, provision it, and back it up. I chose an LXC container because it's lighter than a full VM while keeping the features I care about.

The reason it's not just the containerized (Docker) version of Technitium: in that version the logs show every request coming from the bridge, so you can't tell who actually asked. A real OS install fixes that, but I don't want to hand a whole VM's worth of resources to one small service.

I also don't want all my services on one VM. If that VM goes down, that's one big blast surface, so I'd rather have a bunch of small machines. LXC containers act like smaller VMs for services you treat like pets instead of cattle. I can run them on Proxmox, on bare metal with `systemd-nspawn`, or on another hypervisor through Incus. I still get backups, and if I ever need to pull a service out, the migration path is easier because the backup is just the filesystem, so I can mount it somewhere else later.

---

## Centralizing the Common Values

I worked in a strange order, doing things as they came to me. One was centralizing the important values, so I made a `common-outputs` folder that just exports the things every foundation system needs.

The thing that pushed me to it was upstream DNS versus the DNS server. Most machines need to use the `dns_server`, and then the DNS server itself needs to use the `upstream_dns_server`. Keeping those in one place beat redefining them in every repo.

```hcl
output "proxmox_endpoint"    { value = "https://10.0.0.10:8006/" }
output "gateway"             { value = "10.0.0.1" }
output "dns_server"          { value = "10.0.0.31" }   # Technitium, guests resolve here
output "upstream_dns_server" { value = "10.0.0.1" }    # router, what the resolver forwards to
output "provider_ssh_user"   { value = "terraform" }
output "machine_ssh_user"    { value = "jon" }
output "ssh_keys"            { value = [trimspace(file("~/.ssh/id_ed25519.pub"))] }
```

---

## The Code I Kept Copying

I wanted to stop copying the same block of Terraform around. This resource has spun up VMs across at least three of my repos, probably more, and every copy was one more place to fix when something changed. Here's the block that was living in every repo, reaching into a common module for the shared values:

```hcl
resource "proxmox_virtual_environment_vm" "vms" {
  for_each  = local.vms
  name      = each.key
  node_name = each.value.node_name

  clone { vm_id = each.value.template_vm_id }
  agent { enabled = true }
  cpu    { cores = each.value.cores }
  memory { dedicated = each.value.memory }

  initialization {
    dns { servers = [module.common.dns_server] }
    ip_config {
      ipv4 {
        address = each.value.ip_address
        gateway = module.common.gateway
      }
    }
    user_account {
      username = "jon"
      keys     = [trimspace(file("~/.ssh/id_ed25519.pub"))]
    }
  }

  disk {
    interface = each.value.disk.interface
    size      = each.value.disk.size
  }
}
```

---

## Making It a Module

A module is just a few files: `main.tf` for the resource, `variables.tf` for the inputs, `outputs.tf` for what it returns, a `versions.tf` to pin the provider, and a `README`. The resource itself barely changes. Every hardcoded or `module.common.*` value becomes a `var.*`, so the block stops knowing anything about my specific setup. The real edits:

```diff
-  for_each  = local.vms
+  for_each  = var.input_vms

-    dns { servers = [module.common.dns_server] }
+    dns { servers = [var.dns_server] }
-        gateway = module.common.gateway
+        gateway = var.gateway

-      username = "jon"
-      keys     = [trimspace(file("~/.ssh/id_ed25519.pub"))]
+      username = var.ssh_user
+      keys     = var.ssh_keys
```

Then the inputs those vars come from. `ssh_user` and `gateway` get defaults so a caller only overrides them when they differ. `dns_server` has no default on purpose, because the whole point was that some machines get the resolver and the resolver gets upstream, so the caller must say which. `input_vms` is the typed shape of the machine map:

```hcl
variable "dns_server" {
  type        = string
  description = "the dns server to use, can be upstream or downstream"
}

variable "input_vms" {
  type = map(object({
    node_name      = string
    template_vm_id = number
    cores          = number
    memory         = number
    ip_address     = string
    disk = object({
      interface = string
      size      = number
    })
  }))
  description = "The list of vms to be created"
}
```

One output change worth calling out: the old version grabbed `ipv4_addresses[1][0]`, which assumes the second NIC entry is the real one and the first is loopback. That's fragile. I changed it to filter loopback out explicitly instead of trusting the index:

```diff
-  value = { for k, v in proxmox_virtual_environment_vm.vms : k => v.ipv4_addresses[1][0] }
+  value = {
+    for k, v in proxmox_virtual_environment_vm.vms :
+    k => [for addr in flatten(v.ipv4_addresses) : addr if addr != "127.0.0.1"][0]
+  }
```

---

## The LXC Version

Once the VM module existed, the container version was only a few changes off it. The resource type changes, a hostname shows up, there's no `agent` block, the VM has a name the container doesn't, and the disk loses its interface:

```diff
-resource "proxmox_virtual_environment_vm" "vms" {
+resource "proxmox_virtual_environment_container" "containers" {
-  for_each  = var.input_vms
-  name      = each.key
+  for_each  = var.input_containers
   node_name = each.value.node_name

-  agent { enabled = true }

   initialization {
+    hostname = each.key
     dns { servers = [var.dns_server] }

     user_account {
-      username = var.ssh_user
       keys     = var.ssh_keys
     }

   disk {
-    interface = each.value.disk.interface
     size      = each.value.disk.size
   }
```

The `input_containers` variable is the same shape as `input_vms` minus the disk interface, and the output reads `.ipv4` off the container instead of filtering the VM's address list:

```diff
-  value = {
-    for k, v in proxmox_virtual_environment_vm.vms :
-    k => [for addr in flatten(v.ipv4_addresses) : addr if addr != "127.0.0.1"][0]
-  }
+  value = { for k, v in proxmox_virtual_environment_container.containers : k => v.ipv4 }
```

---

## Using the Module in the Foundation Repo

With the module written, each foundation system just describes its machines and calls it. Here's the DNS one, `2-dns/terraform/technitium.tf`. It pulls shared values from common, defines the one VM, and hands it to the module:

```hcl
module "common" {
  source = "../../common-outputs/"
}

locals {
  vms = {
    technitium = {
      node_name      = "nibbler"
      template_vm_id = 9000
      cores          = 4
      memory         = 8192
      ip_address     = "10.0.0.31/24"
      disk = {
        interface = "scsi0"
        size      = 60
      }
    }
  }
}

module "proxmox_create_vms" {
  source = "../../terraform-common/proxmox_create_vms"

  input_vms  = local.vms
  ssh_user   = module.common.machine_ssh_user
  ssh_keys   = module.common.ssh_keys
  dns_server = module.common.upstream_dns_server
  gateway    = module.common.gateway
}

output "vm_ipv4_address" {
  value = module.proxmox_create_vms.vm_ipv4_address
}
```

The provider block is tiny too, since it also reads from common:

```hcl
provider "proxmox" {
  endpoint = module.common.proxmox_endpoint
  insecure = true

  ssh {
    agent    = true
    username = module.common.provider_ssh_user
  }
}
```

The Forgejo system (`3-git/terraform/forgejo.tf`) is the same file with a different machine map, pointing at the same module. That's the whole payoff. Adding a new foundation service is now a `locals` block and a module call, not another copy of the resource.

---

## Why Git Subtree Instead of a Submodule or a Registry

One decision I made about how the module actually lives in the foundation repo: the module code is its own thing, but I pulled it into the foundation repo with `git subtree` rather than referencing it as a submodule or a remote source.

The reason is backups. This foundation repo, state included, is going on a USB stick as a recovery copy. It also lives in Forgejo, but I want to be able to rebuild foundation infra from just the USB, with nothing else reachable. A submodule or a remote module source would mean the USB copy is incomplete without pulling from somewhere. A subtree copies the module's files directly into this repo, so the one repo on the stick has everything.

The state going on the stick is a nice-to-have, not critical. If I lost it I could `terraform import` my way back, but that's work I'd rather avoid, so it rides along.
