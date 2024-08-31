# Physical PC preparation

Installing Terraform:
https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli

Put the interface of the PC running proxmox in Promisc mode if running linux: 

```bash
sudo chmod a+rw /dev/vmnet0
```

Enable :
```bash
sudo ifconfig enp2s0 promisc
```

Disable :
```bash
ifconfig enp2s0 -promisc
```

---

# Proxmox server preparation

Into the ProxMox Server you need to install libguestfs-tools:
```bash
apt update -y && apt install libguestfs-tools -y
```

Create a directory where you will stock Cloud-Images to propose for template
```bash
mkdir cloud-image
#Ubuntu Server cloud image
wget https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
#Debian cloud image
wget https://cloud.debian.org/images/cloud/bullseye/latest/debian-11-generic-amd64.qcow2
```

Preinstalling qemu-guest-agent into the cloud image
```bash
virt-customize -a debian-11-generic-amd64.qcow2 --install qemu-guest-agent
virt-customize -a focal-server-cloudimg-amd64.img --install qemu-guest-agent
```

Creating a user for Terraform
```bash
pveum role add TerraformProv -privs "Datastore.AllocateSpace Datastore.Audit Pool.Allocate Sys.Audit Sys.Console Sys.Modify VM.Allocate VM.Audit VM.Clone VM.Config.CDROM VM.Config.Cloudinit VM.Config.CPU VM.Config.Disk VM.Config.HWType VM.Config.Memory VM.Config.Network VM.Config.Options VM.Migrate VM.Monitor VM.PowerMgmt SDN.Use"

pveum user add terraform-prov@pve --password <password>

pveum aclmod / -user terraform-prov@pve -role TerraformProv
```

Next for this user create an API Token from the web interface of proxmox & deselect privilege when creating it

---

# First Terraform action

Now that everything is setup you will need to create multiple file for Terraform to work

First we would need an provider.tf file 

**Note : Normally we would never write the TokenID,TokenSecret,ApiURL in the provider.tf file. For this im using the root user and not the terraform user**

provider.tf:
```bash
terraform {

  required_version = ">= 0.13.0"

  required_providers {
    proxmox = {
      source = "telmate/proxmox"
      version = "3.0.1-rc3"
    }
  }
}

provider "proxmox" {
  pm_api_url = "https://192.168.1.29:8006/api2/json"
  proxmox_api_token_id = "root@pam!root"
  proxmox_api_token_secret = "15489609-3cfe-4691-9b8a-e1cbb30901fa"

  pm_tls_insecure = true
}
```

After you should run :
```bash
terraform init
```

To see if everything is working good , for me it was not working then I had to compile from source using the official doc :
https://github.com/Telmate/terraform-provider-proxmox/blob/master/docs/guides/installation.md

The variables.tfvars will be created automatically in the bash script we will see that afterward

As my project is to automate as much as possible the process of provising VM to be used as template and Copy of said template I'll start creating my bash script that will be used .

First thing the user would have to already have set up Terraform providers

Script:
```bash
#!/bin/bash

# Function to create a VM template
create_vm_template() {
    echo "Creating VM template..."
    read -p "Enter the template name: " template_name
    read -p "Enter the base cloud image path: " base_image
    read -p "Enter the VM ID: " vm_id
    read -p "Enter the number of CPU cores: " cpu_cores
    read -p "Enter the amount of RAM (in MB): " ram_size
    read -p "Enter the bridge interface name: " bridge_interface
    read -p "Enter the VLAN tag (if any, or leave blank): " vlan_tag
    read -p "Enter the number of disks: " disk_count

    # Loop to collect disk sizes
    declare -a disk_sizes
    for ((i=1; i<=disk_count; i++)); do
        read -p "Enter the size of disk $i (in GB): " disk_size
        disk_sizes+=("$disk_size")
    done

    read -p "Enter the Proxmox node for creation: " proxmox_node
    read -p "Enter the Proxmox storage location: " storage_location
    # To Implement
}

# Function to copy VMs from a template
copy_from_template() {
    echo "Copying VMs from template..."
    read -p "Enter the template name: " template_name
    read -p "Enter the number of VMs to create: " vm_count
    read -p "Enter the VM ID of the template: " template_vm_id
    read -p "Enter the starting VM ID: " start_vm_id

    # To Implement
}

# Main script execution
echo "Select an option:"
echo "1) Create VM Template"
echo "2) Copy from Template"
read -p "Enter your choice (1 or 2): " choice

case $choice in
    1)
        create_vm_template
        ;;
    2)
        copy_from_template
        ;;
    *)
        echo "Invalid option. Please run the script again and choose either 1 or 2."
        exit 1
        ;;
esac

echo "Operation completed."
```

Now we will create 2 different directory that will have our different script

```bash
mkdir create_vm_template
mkdir copy_vm_template
```

---

# Create_Vm_Template

We will start by doing the create_vm_template, we will have a main.tf file and a var.tfvars and dont forget your provider.tf

main.tf
```
# Proxmox provider configuration
provider "proxmox" {
  pm_api_url      = "https://proxmox.example.com:8006/api2/json"
  pm_user         = "root@pam"
  pm_password     = var.proxmox_password
  pm_tls_insecure = true
}

# Variable definitions
variable "template_name" {}
variable "base_image" {}
variable "start_vm_id" {}
variable "vm_count" {}
variable "cpu_cores" {}
variable "ram_size" {}
variable "bridge_interface" {}
variable "vlan_tag" {
  default = ""
}
variable "disk_count" {}
variable "disk_sizes" {
  type = list(string)
}
variable "proxmox_node" {}
variable "storage_location" {}

# Resource block for creating a VM template
resource "proxmox_vm_qemu" "vm_template" {
  name         = var.template_name
  vmid         = var.start_vm_id
  target_node  = var.proxmox_node
  agent        = 1
  cores        = var.cpu_cores
  memory       = var.ram_size
  bootdisk     = "scsi0"

  network {
    model  = "virtio"
    bridge = var.bridge_interface
    tag    = var.vlan_tag != "" ? var.vlan_tag : null
  }

  dynamic "disk" {
    for_each = range(var.disk_count)
    content {
      id     = disk.value
      size   = "${var.disk_sizes[disk.value]}G"
      storage = var.storage_location
      type   = "scsi"
    }
  }

  clone {
    base_template = var.base_image
  }
}

# Resource block for copying VMs from a template
resource "proxmox_vm_qemu" "vm_clone" {
  count        = var.vm_count
  name         = "${var.template_name}-clone-${count.index + 1}"
  vmid         = var.start_vm_id + count.index
  target_node  = var.proxmox_node
  agent        = 1
  cores        = var.cpu_cores
  memory       = var.ram_size

  network {
    model  = "virtio"
    bridge = var.bridge_interface
    tag    = var.vlan_tag != "" ? var.vlan_tag : null
  }

  disk {
    size   = "${var.disk_sizes[0]}G"
    storage = var.storage_location
    type   = "scsi"
  }

  clone {
    base_template = proxmox_vm_qemu.vm_template.name
  }
}


---


On the proxmox server cli

wget https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img

qm create 8000 --memory 2048 --name ubuntu-cloud --net0 virtio,bridge=vmbr0

qm importdisk 8000 focal-server-cloudimg-amd64.img local-lvm

qm set 8000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-8000-disk-0

qm set 8000 --ide2 local-lvm:cloudinit

qm set 8000 --boot c --bootdisk scsi0

qm set 8000 --serial0 socket --vga serial0


Everything is initialized now go onto the VM and initiliaze the cloud init settings -> USERNAME , PASSWORD , DHCP

Change settings if you want ( cpu,ram) and after right click on the image and convert to template !!! RESIZE THE DISK SIZE !!!


-------------------------------------------------

QEMU

Enable le QEMU Guest Agent dans les Options de la VM

sur le serveur : sudo apt install qemu-guest-agent

sudo systemctl enable qemu-guest-agent
sudo systemctl start qemu-guest-agent

--------------------------------------------------

TERRAFORM

Install on client :

snap install terraform

Install on server:

apt update -y && apt install libguestfs-tools -y

--------------------------------------------------

IMAGE CREATION AND CONVERT TO TEMPLATE

Install on different cloud image:

virt-customize -a [nom_image] --install qemu-guest-agent

Use this command to start creating the image for the template:
qm create [VMID] --memory [RAM] --name [ubuntu-cloud] --net0 virtio,bridge=[vmbr0-INT]

Import the disk image into the storage :
qm importdisk 8000 focal-server-cloudimg-amd64.img [local-lvm - location ]


qm set 8000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-8000-disk-0

qm set 8000 --ide2 local-lvm:cloudinit

qm set 8000 --boot c --bootdisk scsi0

qm set 8000 --serial0 socket --vga serial0

Enable qemu-guest-agent
qm set [VMID] --agent enabled=1

Convert Vm created to template:
qm template [VMID]


--------------------------------------------------

TERRAFORM SETTING WITH TELMATE

** On proxmox shell **

pveum role add TerraformProv -privs "Datastore.AllocateSpace Datastore.Audit Pool.Allocate Sys.Audit Sys.Console Sys.Modify VM.Allocate VM.Audit VM.Clone VM.Config.CDROM VM.Config.Cloudinit VM.Config.CPU VM.Config.Disk VM.Config.HWType VM.Config.Memory VM.Config.Network VM.Config.Options VM.Migrate VM.Monitor VM.PowerMgmt SDN.Use"
pveum user add terraform-prov@pve --password <password>
pveum aclmod / -user terraform-prov@pve -role TerraformProv


** On our machine export these variable**

export PM_USER="terraform_user@pve"

export PM_PASS="[password]"




