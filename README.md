# Documentation
This sets up a central master server aka meastro or gepetto which hosts terraform and ansible scripts.

[TOC]



# Prerequisites

## System-Requirements

| Requirement | Value                                                        |
| ----------- | ------------------------------------------------------------ |
| vCPU        | 4                                                            |
| RAM         | 8 GB                                                         |
| DISK        | 100 GB                                                       |
| Network     | vmxnet3<br />Subnet: 10.0.249.0/24<br />GW: 10.0.249.1<br />DNS: 10.0.249.53 |
| Hostname    | maestro                                                      |
| Domain      | home.local                                                   |



## ISO-Image-Operating System

* CentOS 7.7.1908
  * Server 64-Bit	http://vault.centos.org/7.7.1908/isos/x86_64/CentOS-7-x86_64-DVD-1908.iso

## Network

### IP

| IP-address   | Subnet              | Gateway    | DNS-Server  | MAC-address       |
| ------------ | ------------------- | ---------- | ----------- | ----------------- |
| 10.0.249.250 | 255.255.255.0 (/24) | 10.0.249.1 | 10.0.249.53 | 00:50:56:a6:ee:ee |

### DNS

| FQDN               |
| ------------------ |
| maestro.home.local |

### Storage

* root-partition:
  * 100 GB thin-provisioned

## User

* root

* Test1234

  

# Installation-process

## Terraform-for-Install-VM

### main.tf

```bash
#cat main.tf
provider "vsphere" {
  vsphere_server = var.vsphere_server
  user = var.vsphere_user
  password = var.vsphere_password
  allow_unverified_ssl = true
}

# --- VARIABLE-DECLARATION
data "vsphere_datacenter" "dc" {
  name = "dc-home"
}

data "vsphere_compute_cluster" "cluster" {
  name = "cluster-home"
  datacenter_id = data.vsphere_datacenter.dc.id
}

data "vsphere_resource_pool" "pool" {
  name = "rp-home"
  datacenter_id = data.vsphere_datacenter.dc.id
}

data "vsphere_datastore" "datastore" {
  name = "openshift_storage"
  datacenter_id = data.vsphere_datacenter.dc.id
}

data "vsphere_network" "network" {
  name = "dpg-home-prod"
  datacenter_id = data.vsphere_datacenter.dc.id
}

variable "ocp-folder" {
  default = "/dc-home/vm/ocp43-patrick"
}

# --- Create VM MAESTRO --- #
resource "vsphere_virtual_machine" "maestro" {
  name = var.vm_name_maestro_host
  folder = var.ocp-folder
  guest_id = var.guest_id_tag
  resource_pool_id = data.vsphere_resource_pool.pool.id
  firmware = "bios"
  datastore_id = data.vsphere_datastore.datastore.id
  num_cpus = 4
  memory = 8192
  wait_for_guest_ip_timeout = 10
  network_interface {
    network_id = data.vsphere_network.network.id
    adapter_type = "vmxnet3"
    use_static_mac = true
    mac_address = "00:50:56:a6:ee:ee"
  }
  disk {
    label = "rootvolume"
    size  = "100"
    thin_provisioned  = "true"
  }
}
```



### variables.tf

```bash
#cat variables.tf
# vcenter-connection
variable "vsphere_server" {
  default = "vcenter.home.local"
}

variable "vsphere_user" {
  default = "administrator@home.local"
}

variable "vsphere_password" {
  default = "Test1234!"
}

# VM-Variables
variable "vm_name_maestro_host" {
  default = "maestro"
}

variable "vm_folder" {
  default = "ocp43-patrick"
}

variable "guest_id_tag" {
  default = "centos7_64Guest"
}

# vcenter-objects
variable "vsphere_datacenter" {
  default = "dc-home"
}

variable "vsphere_compute_cluster" {
  default = "cluster-home"
}

variable "vsphere_resource_pool" {
  default = "rp-home"
}

variable "vsphere_datastore" {
  default = "openshift_storage"
}

variable "vsphere_network" {
  default = "dpg-home-prod"
}
```



## Install-Operating-System-CentOS-7.7-1908

### Download-ISO-Image

| Item                          | Value                                                        |
| ----------------------------- | ------------------------------------------------------------ |
| Operating System              | CentOS 7.7.1908                                              |
| Download-Link (Server 64-Bit) | http://vault.centos.org/7.7.1908/isos/x86_64/CentOS-7-x86_64-DVD-1908.iso |
| Download-Folder               | \\nas\nfs-iso\downloaded-iso\linux                           |

### PXELINUX-Config-File-kickstart-installation

```bash
# cat 01-00-0c-29-ee-ee-ee
DEFAULT menu.c32
PROMPT 0

MENU TITLE Maestro-Server-Install-Menu
MENU AUTOBOOT Starting CentOS in # seconds

timeout 18
ONTIMEOUT centOS

LABEL centOS
menu label Install maestro-Server
kernel kernels_initrd/centos_7.7.1908/vmlinuz
append ip=dhcp initrd=kernels_initrd/centos_7.7.1908/initrd.img inst.repo=nfs:nas.home.local:/volume1/nfs-iso/downloaded-iso/linux/CentOS-7-x86_64-DVD-1908.iso inst.ks=nfs:nas.home.local:/volume1/nfs-iso/kickstart-configs/maestro.cfg
```

### PXELINUX-Config-File-interactive-installation

```bash
# cat 01-00-0c-29-ee-ee-ee
DEFAULT menu.c32
PROMPT 0

MENU TITLE Maestro-Server-Install-Menu
MENU AUTOBOOT Starting CentOS in # seconds

timeout 18
ONTIMEOUT centOS

LABEL centOS
menu label Install maestro-Server
kernel kernels_initrd/centos_7.7.1908/vmlinuz
append ip=dhcp initrd=kernels_initrd/centos_7.7.1908/initrd.img inst.repo=nfs:nas.home.local:/volume1/nfs-iso/downloaded-iso/linux/CentOS-7-x86_64-DVD-1908.iso
```



### Kickstart-File-mastro.cfg

```bash
#version=DEVEL
# System authorization information
auth --enableshadow --passalgo=sha512
# Use graphical install
graphical
# Use NFS installation media
nfs --server=nas.home.local --dir=/volume1/nfs-iso/downloaded-iso/linux/CentOS-7-x86_64-DVD-1908.iso
# Run the Setup Agent on first boot
firstboot --enable
ignoredisk --only-use=sda
# Keyboard layouts
keyboard --vckeymap=de --xlayouts='de'
# System language
lang en_US.UTF-8

# Network information
network  --bootproto=dhcp --device=ens192 --ipv6=auto --activate
network  --hostname=maestro.home.local

# Root password
rootpw --iscrypted $6$VZP5YYgWUN0wBjc/$6eERueMDPVCdzcBmpvu3pii4YZRwjeTQmpZNwr1s9PlKFAnbxWL2AKXUH.f6k0nxHhdcrFwMzjFDf3D0kvxHW0
# System services
services --enabled="chronyd"
# System timezone
timezone Europe/Berlin --isUtc --ntpservers=router.home.local
# System bootloader configuration
bootloader --location=mbr --boot-drive=sda
autopart --type=lvm
# Partition clearing information
clearpart --none --initlabel

%packages
@^minimal
@core
@debugging
@development
@security-tools
@system-admin-tools
chrony

%end

%addon com_redhat_kdump --disable --reserve-mb='auto'

%end

%anaconda
pwpolicy root --minlen=6 --minquality=1 --notstrict --nochanges --notempty
pwpolicy user --minlen=6 --minquality=1 --notstrict --nochanges --emptyok
pwpolicy luks --minlen=6 --minquality=1 --notstrict --nochanges --notempty
%end

```



## Install-Packages

### terraform

### ansible