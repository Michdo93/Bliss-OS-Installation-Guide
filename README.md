# Bliss OS Installation Guide

A practical guide to installing **Bliss OS 16.9.7 (x86_64)** as a virtual machine on **Proxmox VE**, including UEFI configuration, disk partitioning, GRUB configuration, and automatic boot from the virtual hard disk.

This guide documents a working installation using the official Bliss OS GApps ISO and an OVMF-based Proxmox VM.

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Proxmox VM Configuration](#proxmox-vm-configuration)
- [Create the Installation ISO](#create-the-installation-iso)
- [Install Bliss OS](#install-bliss-os)
- [Disk Partitioning](#disk-partitioning)
- [Configure GRUB](#configure-grub)
- [Configure the UEFI Fallback Bootloader](#configure-the-uefi-fallback-bootloader)
- [Configure Proxmox Boot Order](#configure-proxmox-boot-order)
- [Troubleshooting](#troubleshooting)
- [References](#references)

---

## Overview

Bliss OS is an Android-based operating system designed to run on x86 and x86_64 hardware.

When installing Bliss OS as a virtual machine on Proxmox VE, the operating system can be installed successfully while the VM still fails to boot directly from its virtual hard disk.

One issue encountered during installation is that the EFI bootloader files are installed under:

```text
EFI/BlissOS/
```

However, the UEFI firmware may not automatically find a suitable bootloader through the standard fallback path:

```text
EFI/BOOT/BOOTX64.EFI
```

This guide addresses that issue by configuring GRUB to boot the installed Android kernel directly and providing the necessary EFI files in the standard UEFI fallback directory.

## Requirements

- Proxmox VE
- Bliss OS 16.9.7 x86_64 ISO
- A virtual disk for the operating system
- UEFI firmware (OVMF)
- Basic knowledge of Proxmox and GRUB

### Tested configuration

| Component | Configuration |
|---|---|
| Hypervisor | Proxmox VE |
| Operating system | Bliss OS 16.9.7 |
| Architecture | x86_64 |
| ISO | Bliss-v16.9.7-x86_64-OFFICIAL-gapps-20241011.iso |
| Firmware | OVMF (UEFI) |
| Machine type | q35 |
| CPU type | host |
| CPU cores | 4 |
| Memory | 4096 MB |
| Disk controller | VirtIO SCSI |
| Virtual disk | 128 GB |

The exact configuration can be adapted to the available hardware.

---

## Proxmox VM Configuration

Create a new virtual machine in Proxmox VE.

Recommended settings:

### General

Choose a suitable VM ID and name, for example:

```text
Name: blissos-vm
```

### OS

Select the Bliss OS ISO as the installation medium.

### System

Configure:

```text
Machine: q35
BIOS: OVMF (UEFI)
EFI Storage: local-lvm
```

Create an EFI disk when prompted.

**Important:** The Proxmox EFI disk (`efidisk0`) stores UEFI firmware variables. It is not the EFI System Partition used by Bliss OS. The EFI System Partition must be created separately on the actual virtual hard disk.

### CPU

```text
Cores: 4
Type: host
```

### Memory

```text
Memory: 4096 MB
```

### Hard Disk

Create a virtual disk with the following settings:

```text
Bus/Device: SCSI
Storage: local-lvm
Size: 128 GB
SCSI Controller: VirtIO SCSI
```

### Boot Order

During installation, the ISO must be available as a boot device.

The virtual hard disk should be configured as the first boot device once installation is complete.

---

## Create the Installation ISO

Download the official Bliss OS ISO.

Example:

```text
Bliss-v16.9.7-x86_64-OFFICIAL-gapps-20241011.iso
```

Upload the ISO to the Proxmox ISO storage and attach it to the virtual machine.

Start the VM and boot from the ISO.

---

## Install Bliss OS

Start the Bliss OS installer from the ISO boot menu.

Follow the installation procedure.

When prompted to select the installation target, choose the 128 GB virtual hard disk.

The installation target should be partitioned manually to ensure a separate EFI System Partition.

---

## Disk Partitioning

For this installation, the virtual disk was partitioned as follows:

| Partition | Size | Filesystem | Purpose |
|---|---:|---|---|
| `/dev/sda1` | 1 GB | FAT32 | EFI System Partition |
| `/dev/sda2` | Remaining space | ext4 | Bliss OS installation |

The partition labels used were:

```text
/dev/sda1: ESP
/dev/sda2: BlissOS
```

The EFI System Partition is mounted by the installer or manually when required.

**Important:** The EFI System Partition is `/dev/sda1`. Do not confuse it with the separate Proxmox EFI variables disk.

Complete the Bliss OS installation and install its bootloader when prompted.

---

## Configure GRUB

After installation, the VM may display a GRUB menu but fail to start Bliss OS automatically.

The installed GRUB configuration is located at:

```text
/EFI/BlissOS/grub.cfg
```

on the EFI System Partition.

The original configuration typically contains:

```grub
search.fs_uuid <EXT4_UUID> root
set timeout=5
set debug_mode="(DEBUG mode)"
set prefix=($root)'/boot/grub'
set kdir=/android-2024-10-11

source $prefix/android.cfg
```

This loads the additional Android boot configuration from the ext4 partition.

For the tested installation, a direct kernel boot entry was used instead.

### 1. Identify the partitions

Boot into the GRUB menu and press `C` to open the GRUB command line.

The available devices can be listed with:

```grub
ls
```

Inspect the EFI partition:

```grub
ls (hd0,gpt1)
```

Inspect the Bliss OS partition:

```grub
ls (hd0,gpt2)/
```

The expected layout is:

```text
(hd0,gpt1) = EFI System Partition
(hd0,gpt2) = Bliss OS ext4 partition
```

The ext4 partition contains:

```text
/android-2024-10-11/
    kernel_a
    initrd_a.img
    kernel_b
    initrd_b.img
    system_a.img
    system_b.img
```

Identify the ext4 filesystem UUID using the existing GRUB configuration or `blkid` from the Linux debug shell.

### 2. Test direct kernel boot

Before modifying the configuration, test whether Bliss OS can boot directly from the installed kernel.

In the GRUB command line, enter:

```grub
set root=(hd0,gpt2)
linux /android-2024-10-11/kernel_a root=/dev/ram0 androidboot.slot_suffix=_a
initrd /android-2024-10-11/initrd_a.img
boot
```

This bypasses the additional GRUB configuration and directly starts the installed Android kernel and initrd.

If the installation enters a debug shell, use:

```bash
exit
```

to continue booting. In the tested DEBUG=2 configuration, two `exit` commands may be necessary.

Once Bliss OS starts successfully, continue with the permanent configuration.

### 3. Boot into the Linux debug shell

To access the Linux shell during startup, use the following commands from the GRUB command line:

```grub
set root=(hd0,gpt2)
linux /android-2024-10-11/kernel_a root=/dev/ram0 androidboot.slot_suffix=_a DEBUG=2
initrd /android-2024-10-11/initrd_a.img
boot
```

When the debug shell appears, do not immediately enter `exit`.

The Linux shell provides access to utilities such as `mount`, `mkdir`, `cp`, and `vi`.

Note that these are Linux commands, not GRUB commands. They cannot be executed directly at the GRUB command prompt.

### 4. Mount the EFI System Partition

In the Linux debug shell, identify the partitions:

```bash
blkid
```

For the tested Proxmox installation:

```text
/dev/sda1 = EFI System Partition (ESP)
/dev/sda2 = Bliss OS ext4 partition
```

Create a mount point and mount the EFI partition:

```bash
mkdir -p /mnt
mount -t vfat /dev/sda1 /mnt
```

Verify the existing EFI files:

```bash
ls /mnt/EFI/BlissOS/
```

Expected files include:

```text
BOOTx64.EFI
bootia32.efi
grub.cfg
grubx64.efi
mmia32.efi
mmx64.efi
```

### 5. Back up the original GRUB configuration

Before making changes:

```bash
cp /mnt/EFI/BlissOS/grub.cfg \
   /mnt/EFI/BlissOS/grub.cfg.backup
```

### 6. Create a direct GRUB boot entry

Edit the configuration:

```bash
vi /mnt/EFI/BlissOS/grub.cfg
```

Replace the existing content with:

```grub
search --no-floppy --fs-uuid --set=root <EXT4_UUID>

set default=0
set timeout=5

menuentry 'Bliss OS' {
    linux /android-2024-10-11/kernel_a root=/dev/ram0 androidboot.slot_suffix=_a
    initrd /android-2024-10-11/initrd_a.img
}
```

Replace `<EXT4_UUID>` with the actual UUID of the ext4 partition containing the Bliss OS installation.

For example, the UUID can be obtained using:

```bash
blkid /dev/sda2
```

The important kernel parameter is:

```text
androidboot.slot_suffix=_a
```

Make sure to spell `slot` correctly.

The `DEBUG=2` parameter is deliberately omitted from the permanent configuration. It is only required when debugging the boot process.

### 7. Save the configuration

In `vi`:

1. Press `i` to enter insert mode.
2. Edit the file.
3. Press `Esc`.
4. Enter `:wq`.
5. Press Enter.

Verify the configuration:

```bash
cat /mnt/EFI/BlissOS/grub.cfg
```

Write pending changes and unmount the EFI partition:

```bash
sync
umount /mnt
```

---

## Configure the UEFI Fallback Bootloader

At this point, GRUB can boot Bliss OS when started manually, but the Proxmox OVMF firmware may still fail to boot directly from the virtual hard disk.

The reason in the tested installation was the missing standard UEFI fallback directory.

The existing files were installed under:

```text
EFI/BlissOS/
```

However, the standard x86_64 UEFI fallback location is:

```text
EFI/BOOT/BOOTX64.EFI
```

The following steps provide the necessary files in this directory.

### 1. Boot into the Linux debug shell

Use the same GRUB commands described above with `DEBUG=2`.

Do not enter `exit` until the EFI configuration has been updated.

### 2. Mount the EFI partition

```bash
mkdir -p /mnt
mount -t vfat /dev/sda1 /mnt
```

### 3. Create the standard UEFI directory

```bash
mkdir -p /mnt/EFI/BOOT
```

### 4. Copy the required EFI files

Copy the following files from the Bliss OS EFI directory:

```bash
cp /mnt/EFI/BlissOS/BOOTx64.EFI /mnt/EFI/BOOT/
cp /mnt/EFI/BlissOS/grubx64.efi /mnt/EFI/BOOT/
cp /mnt/EFI/BlissOS/mmx64.efi /mnt/EFI/BOOT/
cp /mnt/EFI/BlissOS/grub.cfg /mnt/EFI/BOOT/
```

The resulting directory should contain:

```text
EFI/
├── BlissOS/
│   ├── BOOTx64.EFI
│   ├── grubx64.efi
│   ├── grub.cfg
│   ├── mmx64.efi
│   ├── bootia32.efi
│   └── mmia32.efi
│
└── BOOT/
    ├── BOOTx64.EFI
    ├── grubx64.efi
    ├── mmx64.efi
    └── grub.cfg
```

**Important:** Copying only `BOOTx64.EFI` is insufficient for this tested installation. The bootloader also needs the accompanying GRUB executable and configuration file in the fallback directory.

### 5. Verify and unmount

```bash
ls -l /mnt/EFI/BOOT/
sync
umount /mnt
```

The EFI System Partition now contains the standard UEFI fallback bootloader path.

---

## Configure Proxmox Boot Order

After configuring the EFI fallback files, check the VM configuration in Proxmox.

Under:

```text
VM → Options → Boot Order
```

Set the virtual hard disk (`scsi0`) as the first boot device.

The CD/DVD drive (`ide2`) can remain attached temporarily for troubleshooting.

Once the hard disk boots successfully without the ISO, eject or detach the installation ISO.

The final VM should boot directly from the installed Bliss OS system.

---

## Troubleshooting

### GRUB command line: `file not found`

Check the partition layout:

```grub
ls
ls (hd0,gpt1)/
ls (hd0,gpt2)/
```

Verify that the EFI and Android installation directories exist.

### `Detecting Android-x86...`

This indicates that the kernel has started but Bliss OS has not successfully detected its installation.

Verify:

- The correct ext4 partition is selected.
- The kernel and initrd paths are correct.
- The Android installation directory is correct.
- The kernel parameters match the installed Android slot.

The tested direct boot command is:

```grub
set root=(hd0,gpt2)
linux /android-2024-10-11/kernel_a root=/dev/ram0 androidboot.slot_suffix=_a
initrd /android-2024-10-11/initrd_a.img
boot
```

### UEFI cannot find `grubx64.efi`

If the firmware or bootloader reports that `grubx64.efi` cannot be found, check whether the fallback directory contains all required files:

```text
EFI/BOOT/BOOTX64.EFI
EFI/BOOT/grubx64.efi
EFI/BOOT/mmx64.efi
EFI/BOOT/grub.cfg
```

The files should be copied from the original `EFI/BlissOS/` directory.

### Linux commands are not available

Commands such as:

```text
mount
mkdir
cp
vi
```

are Linux commands.

They are not available in the GRUB command line.

To use them, boot the Bliss OS Linux debug shell with `DEBUG=2`.

### `parted` is not available

The Bliss OS debug environment may not include `parted`.

Use:

```bash
blkid
```

to identify the available partitions and filesystems.

For the tested installation, the partition labels and filesystem types were sufficient to identify the EFI and Bliss OS partitions.

---

## References

- [Bliss OS Official Website](https://blissos.org/)
- [Bliss OS Documentation](https://docs.blissos.org/)
- [Bliss OS GitHub Repository](https://github.com/BlissRoms-x86)
- [Proxmox VE Documentation](https://pve.proxmox.com/pve-docs/)

---

## Final Result

The completed installation uses:

- OVMF UEFI firmware.
- A dedicated 1 GB FAT32 EFI System Partition.
- An ext4 partition containing Bliss OS.
- A direct GRUB kernel boot entry.
- The standard UEFI fallback boot path at `EFI/BOOT/BOOTX64.EFI`.
- The required GRUB EFI executable and configuration files in the fallback directory.

This configuration allows the Proxmox VM to boot Bliss OS directly from its virtual hard disk without relying on the installation ISO.
