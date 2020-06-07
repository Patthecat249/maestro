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
# cat 01-00-50-56-a6-ee-ee
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
# cat 01-00-50-56-a6-ee-ee
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

%post
cat << EOF >> /etc/yum.repos.d/CentOS-SIG-ansible-29.repo
# CentOS-SIG-ansible-29.repo
#
# Please see https://wiki.centos.org/SpecialInterestGroup/ConfigManagementSIG/Ansible
# for more information

[centos-ansible-29]
name=CentOS Configmanagement SIG - ansible-29
mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=configmanagement-ansible-29
#baseurl=http://mirror.centos.org/$contentdir/7/configmanagement/$basearch/ansible-29/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-ConfigManagement

[centos-ansible-29-testing]
name=CentOS Configmanagement SIG - ansible-29 Testing
baseurl=http://buildlogs.centos.org/centos/7/configmanagement/$basearch/ansible-29/
gpgcheck=0
enabled=0

[centos-ansible-29-debuginfo]
name=CentOS Configmanagement SIG - ansible-29 Debug
baseurl=http://debuginfo.centos.org/$contentdir/7/configmanagement/$basearch/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-ConfigManagement

[centos-ansible-29-source]
name=CentOS Configmanagement SIG - ansible-29 Source
baseurl=http://vault.centos.org/$contentdir/7/configmanagement/Source/ansible-29/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-ConfigManagement

EOF

cat << EOF >> /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-ConfigManagement
-----BEGIN PGP PUBLIC KEY BLOCK-----
Version: GnuPG v2.0.22 (GNU/Linux)

mQENBFqyzbABCADt/iuaCPSqEqhQHIW9ThAWS1MTN/2poztrsG6O+BJeyEewPvCw
nfmkTCep3A/vZ2vb0tsX3A/xtiF9HkEKBvGGIxxHKX5/fGRGz6lIhZyNTRQW5WAC
6HRlJe2XSKp3ANaUXaXeCs73ce7qQbLEGPmV6Yn+WnNrfL8xB6fymWG/BjTd/dMw
1quYGynHN1z57wup78G2o+boEqRNtJ6wW3flstCq+OXYoB7kOT/nja2O5lyZBqyV
NjCi/mmj7j9U+1xlpJEb8/OKynTKEJ2wIAa/IlLc6u5a5bAawqCBpRU3xpiu6XB0
ysNSLqSs9Z+W2D4iWOvB/6rqdZPNAADnhCjpABEBAAG0dUNlbnRPUyBDb25maWcg
TWFuYWdlbWVudCBTSUcgKGh0dHBzOi8vd2lraS5jZW50b3Mub3JnL1NwZWNpYWxJ
bnRlcmVzdEdyb3VwL0NvbmZpZ01hbmFnZW1lbnRTSUcpIDxzZWN1cml0eUBjZW50
b3Mub3JnPokBOQQTAQIAIwUCWrLNsAIbAwcLCQgHAwIBBhUIAgkKCwQWAgMBAh4B
AheAAAoJEBrhEPpui36K814IAM0o2BHQ8YXJhxKTas0Nx8KkcejaZMbCNx2JdOFj
9b73LPp4LW68QFNdRib0Clw1IYKLItztSystNUINn856fVyzsiZZ0K00oab8TTYK
eYvjfk0neJDhVfZqCVymwYtV+slJ7MnMACnpT2ijjelJtd6rgzfgd2zVOiwx0qqU
m5NEZDosWwgGvH/9Bpy489OSNstBbahZ6FZ9Day1AHj+vriYSsnPzNcz/cLAdt+G
2w3DzYG97bgBhffSwc8HoF96s3lU7MK0t5WweqxdaiT1jJ6jbY6Y1gEyYybrHd+A
XMqYI79GA/VEwchmxXenXVqL6uA8TlvCDYIdzqa+MXBXlWI=
=HUNw
-----END PGP PUBLIC KEY BLOCK-----
EOF

yum install ansible -y
%end

```



## Install-Packages

### terraform

### ansible