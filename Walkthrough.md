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
**I used Terraform on Windows as well and using Proxmox version 7.4 as I could not run terraform init from my kubuntu machine**

Windows Terraform download : https://developer.hashicorp.com/terraform/install

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

  pm_user       = "root@pam"
  pm_password   = "[password]"

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

This is a Terraform file that will create a vm using the cloud-image of ubuntu

```bash

```


---

# Create_Vm_Template

We will start by doing the create_vm_template, we will have a main.tf file and a var.tfvars and dont forget your provider.tf

main.tf

---


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




