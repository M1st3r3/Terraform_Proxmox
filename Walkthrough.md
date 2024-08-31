SET interface to promiscous mode on vSwitch & Physical NIC

sudo chmod a+rw /dev/vmnet0

Enable :sudo ifconfig enp2s0 promisc

Disable : ifconfig eth0 -promisc

####################################

/etc/sysctl.conf

net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 0
net.bridge.bridge-nf-call-ip6tables = 0

####################################



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




