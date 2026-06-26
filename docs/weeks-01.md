# Week 1 - Building the Foundation

## Objective

Deploy a virtualized Ubuntu Server using Proxmox VE while understanding the reasoning behind each configuration decision.

## Hardware

* Dell OptiPlex 3040
* Intel Core i5-6500
* 16 GB RAM
* 256 GB SSD

## What I Built

* Installed Proxmox VE
* Created an Ubuntu Server virtual machine
* Configured virtual networking
* Connected to the server using SSH from my gaming PC

## Technical Decisions

### CPU

Allocated 2 vCPUs to balance performance while leaving resources available for future virtual machines.

### Memory

Allocated 2 GB RAM after learning Ubuntu Server has relatively low memory requirements.

### Storage

Created a 32 GB virtual disk.

Used LVM because it allows storage to be expanded more easily in the future.

Formatted the filesystem using ext4.

### Networking

Configured the VM to use the vmbr0 bridge.

Allowed DHCP to automatically assign an IP address.

Installed OpenSSH Server to enable secure remote administration.

## Concepts Learned

* Hypervisor
* Host vs Guest Operating System
* Virtual Machines
* Virtual Switches
* DHCP
* DNS
* Gateway
* SSH
* LVM
* ext4 Filesystem
* EFI Partition
* Ubuntu Mirrors

## Reflection

Today I learned that virtualization is more than simply running another operating system. I now understand that a VM has its own virtual CPU, memory, storage and network adapter, allowing it to function as an independent machine on my home network.
