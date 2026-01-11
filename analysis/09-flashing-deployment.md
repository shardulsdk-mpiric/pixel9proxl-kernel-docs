# Deep Analysis #9: Flashing and Deployment Process

## Overview

This document provides a deep technical analysis of the flashing and deployment process for the Pixel 9 Pro XL kernel build. This includes understanding the exact flash sequence, A/B slot system, fastboot commands and partition names, recovery procedures, and backup/restore processes.

## Navigation

- [Analysis Index](README.md)
- [Main Documentation Index](../README.md)
- [Overview](../overview.md)

---

## Table of Contents

1. [Flashing Overview](#1-flashing-overview)
2. [Partition Names and Structure](#2-partition-names-and-structure)
3. [A/B Slot System](#3-ab-slot-system)
4. [Flash Sequence](#4-flash-sequence)
5. [Fastboot Commands](#5-fastboot-commands)
6. [Recovery Procedures](#6-recovery-procedures)
7. [Backup and Restore](#7-backup-and-restore)
8. [Summary and Key Takeaways](#8-summary-and-key-takeaways)

---

## 1. Flashing Overview

### 1.1 What is Flashing?

Flashing is the process of writing kernel build artifacts (boot images, modules, device trees) to device partitions. This is how a custom kernel is deployed to the device.

### 1.2 Flashing Methods

There are several ways to flash kernel artifacts:

1. **Fastboot**: Direct partition flashing (most common for development)
2. **Recovery**: Flashing via recovery mode
3. **OTA**: Over-the-air updates (for production)

### 1.3 Build Artifacts for Flashing

From the build process, the following artifacts are produced (see [Build Command Flow](03-build-command-flow.md)):

- `boot.img`: Main boot image (GKI kernel + base ramdisk)
- `vendor_kernel_boot.img`: Vendor kernel boot image (vendor ramdisk + DTBOs)
- `dtbo.img`: Device tree overlay image
- `dtb.img`: Device tree blob image
- `vendor_dlkm.img`: Vendor Dynamic Loadable Kernel Modules partition
- `system_dlkm.img`: System Dynamic Loadable Kernel Modules partition

These artifacts are copied to `out/caimito/dist/` by the `copy_to_dist_dir` rule.

---

## 2. Partition Names and Structure

### 2.1 Partition Names

Android devices use specific partition names for kernel-related partitions:

- **boot**: Contains `boot.img` (GKI kernel + base ramdisk)
- **vendor_boot**: Contains `vendor_kernel_boot.img` (vendor ramdisk + DTBOs)
- **vendor_dlkm**: Contains `vendor_dlkm.img` (vendor kernel modules)
- **system_dlkm**: Contains `system_dlkm.img` (system kernel modules)
- **dtbo**: Contains `dtbo.img` (device tree overlays)

### 2.2 A/B Slot Partitions

With A/B slots, partitions may have slot suffixes:
- **boot_a / boot_b**: Boot partition for slot A or B
- **vendor_boot_a / vendor_boot_b**: Vendor boot partition for slot A or B
- **vendor_dlkm_a / vendor_dlkm_b**: Vendor DLKM partition for slot A or B
- **system_dlkm_a / system_dlkm_b**: System DLKM partition for slot A or B
- **dtbo_a / dtbo_b**: DTBO partition for slot A or B

### 2.3 Partition Layout

Partitions are typically laid out as:
- **Static partitions**: Fixed size (boot, vendor_boot, dtbo)
- **Dynamic partitions**: Variable size (vendor_dlkm, system_dlkm)

---

## 3. A/B Slot System

### 3.1 What is A/B Slot System?

The A/B slot system allows Android devices to have two sets of partitions (slot A and slot B). This enables seamless updates where one slot can be updated while the other remains bootable.

### 3.2 A/B Slot Benefits

Benefits of A/B slots:
- **Seamless updates**: Update inactive slot while active slot runs
- **Rollback capability**: Boot previous slot if update fails
- **Reduced downtime**: Updates don't require device reboot during installation
- **Safety**: Always have a bootable slot

### 3.3 A/B Slot Operation

A/B slot operation:
1. **Active slot**: Currently booted slot (A or B)
2. **Inactive slot**: Not currently booted slot
3. **Update process**: Update inactive slot
4. **Switch**: Boot into updated slot on next boot
5. **Rollback**: Boot previous slot if update fails

### 3.4 Slot Selection

The bootloader:
- **Detects active slot**: Checks which slot is marked active
- **Boots active slot**: Loads kernel from active slot
- **Switches on update**: Marks updated slot as active
- **Rollback on failure**: Switches back to previous slot

---

## 4. Flash Sequence

### 4.1 Complete Flash Sequence

For Pixel 9 Pro XL (caimito), the complete flash sequence is:

1. **Enter fastboot mode**: Boot device into fastboot mode
2. **Verify device**: Check device is connected and recognized
3. **Determine active slot**: Check which slot is active (A or B)
4. **Flash boot.img**: Flash to `boot` or `boot_<slot>`
5. **Flash vendor_kernel_boot.img**: Flash to `vendor_boot` or `vendor_boot_<slot>`
6. **Flash dtbo.img**: Flash to `dtbo` or `dtbo_<slot>` (if needed)
7. **Flash vendor_dlkm.img**: Flash to `vendor_dlkm` or `vendor_dlkm_<slot>`
8. **Flash system_dlkm.img**: Flash to `system_dlkm` or `system_dlkm_<slot>`
9. **Reboot**: Reboot device to boot new kernel

### 4.2 Flash Order

The flash order matters:
1. **Boot images first**: `boot.img` and `vendor_kernel_boot.img`
2. **Device trees**: `dtbo.img` (if separate)
3. **Modules**: `vendor_dlkm.img` and `system_dlkm.img`
4. **Reboot**: Boot with new kernel

### 4.3 Critical Partitions

Critical partitions that must match:
- **boot.img and vendor_kernel_boot.img**: Must be from same build
- **vendor_dlkm.img**: Must match kernel version
- **system_dlkm.img**: Must match kernel version

---

## 5. Fastboot Commands

### 5.1 Basic Fastboot Commands

Common fastboot commands for flashing:

```bash
# Check device connection
fastboot devices

# Get device information
fastboot getvar all

# Check current slot
fastboot getvar current-slot

# Flash boot image
fastboot flash boot boot.img

# Flash vendor boot image
fastboot flash vendor_boot vendor_kernel_boot.img

# Flash DTBO
fastboot flash dtbo dtbo.img

# Flash vendor DLKM
fastboot flash vendor_dlkm vendor_dlkm.img

# Flash system DLKM
fastboot flash system_dlkm system_dlkm.img

# Reboot device
fastboot reboot
```

### 5.2 Slot-Specific Commands

For A/B slot systems:

```bash
# Flash to specific slot
fastboot flash boot_a boot.img
fastboot flash boot_b boot.img

# Flash to current slot (automatically determined)
fastboot flash boot boot.img

# Set active slot
fastboot set_active a
fastboot set_active b

# Reboot to specific slot
fastboot reboot fastboot
fastboot --set-active=a
fastboot reboot
```

### 5.3 Advanced Fastboot Commands

Additional useful commands:

```bash
# Boot without flashing (temporary)
fastboot boot boot.img

# Flash all images at once
fastboot flash boot boot.img \
  vendor_boot vendor_kernel_boot.img \
  vendor_dlkm vendor_dlkm.img \
  system_dlkm system_dlkm.img

# Verify partition
fastboot getvar partition-size:boot
fastboot getvar has-slot:boot

# Unlock bootloader (one-time, wipes data)
fastboot flashing unlock
```

---

## 6. Recovery Procedures

### 6.1 What If Flash Fails?

If flashing fails or device doesn't boot:

1. **Don't panic**: Device may still be recoverable
2. **Boot to fastboot**: Hold power + volume down to enter fastboot
3. **Check active slot**: Verify which slot is active
4. **Flash to other slot**: Flash to inactive slot if active slot is corrupted
5. **Switch slots**: Boot into working slot
6. **Flash stock images**: Restore stock images if both slots fail

### 6.2 Recovery Steps

Recovery procedure:

1. **Enter fastboot**: Power + volume down
2. **Check slots**: `fastboot getvar current-slot`
3. **Try other slot**: `fastboot set_active <other-slot>`
4. **Reboot**: `fastboot reboot`
5. **If still fails**: Flash stock images from device manufacturer

### 6.3 Boot Loop Recovery

If device is in boot loop:

1. **Force fastboot**: Power + volume down (hold until fastboot appears)
2. **Check bootloader state**: Should see fastboot screen
3. **Flash stock boot**: Flash stock boot.img
4. **Flash stock vendor_boot**: Flash stock vendor_kernel_boot.img
5. **Reboot**: Should boot normally

### 6.4 Emergency Recovery

If device is completely unresponsive:

1. **Download mode**: Some devices have download mode (Power + volume keys)
2. **OEM tools**: Use manufacturer's flashing tools
3. **Factory reset**: May be required to recover
4. **Warranty**: Check warranty status before attempting recovery

---

## 7. Backup and Restore

### 7.1 Why Backup?

Before flashing custom kernel:
- **Safety**: Ability to restore stock kernel
- **Recovery**: Backup if flash fails
- **Testing**: Compare stock vs custom behavior

### 7.2 Backup Procedure

To backup partitions:

```bash
# Backup boot partition
fastboot boot boot.img  # Boot custom kernel temporarily first
adb shell
su
dd if=/dev/block/by-name/boot of=/sdcard/boot_backup.img

# Backup vendor_boot partition
dd if=/dev/block/by-name/vendor_boot of=/sdcard/vendor_boot_backup.img

# Backup vendor_dlkm partition
dd if=/dev/block/by-name/vendor_dlkm of=/sdcard/vendor_dlkm_backup.img

# Backup system_dlkm partition
dd if=/dev/block/by-name/system_dlkm of=/sdcard/system_dlkm_backup.img

# Copy backups to computer
adb pull /sdcard/boot_backup.img .
adb pull /sdcard/vendor_boot_backup.img .
adb pull /sdcard/vendor_dlkm_backup.img .
adb pull /sdcard/system_dlkm_backup.img .
```

### 7.3 Restore Procedure

To restore from backup:

```bash
# Restore boot partition
fastboot flash boot boot_backup.img

# Restore vendor_boot partition
fastboot flash vendor_boot vendor_boot_backup.img

# Restore vendor_dlkm partition
fastboot flash vendor_dlkm vendor_dlkm_backup.img

# Restore system_dlkm partition
fastboot flash system_dlkm system_dlkm_backup.img

# Reboot
fastboot reboot
```

### 7.4 Stock Image Recovery

To restore to stock:

1. **Download stock images**: Get factory images from device manufacturer
2. **Extract images**: Extract boot.img, vendor_boot.img, etc.
3. **Flash stock images**: Flash all stock partitions
4. **Lock bootloader** (optional): `fastboot flashing lock` (wipes data)

---

## 8. Summary and Key Takeaways

### Key Concepts

1. **Flashing Overview**:
   - Process of writing kernel artifacts to device partitions
   - Multiple methods: fastboot, recovery, OTA
   - Artifacts from `out/caimito/dist/` directory

2. **Partition Names**:
   - boot, vendor_boot, vendor_dlkm, system_dlkm, dtbo
   - A/B slot variants: boot_a/boot_b, etc.
   - Static vs dynamic partitions

3. **A/B Slot System**:
   - Two sets of partitions for seamless updates
   - Active/inactive slot concept
   - Rollback capability

4. **Flash Sequence**:
   - Boot images first (boot.img, vendor_kernel_boot.img)
   - Then modules (vendor_dlkm.img, system_dlkm.img)
   - Finally reboot

5. **Fastboot Commands**:
   - `fastboot flash <partition> <image>`: Flash partition
   - `fastboot getvar <variable>`: Get device information
   - `fastboot reboot`: Reboot device
   - `fastboot set_active <slot>`: Set active slot

6. **Recovery Procedures**:
   - Enter fastboot mode
   - Switch slots if needed
   - Flash stock images if necessary

7. **Backup and Restore**:
   - Backup partitions before flashing
   - Restore from backup if needed
   - Use stock images for full recovery

### Practical Implications

- **To flash kernel**: Use fastboot commands with images from `out/caimito/dist/`
- **To check slot**: Use `fastboot getvar current-slot`
- **To recover**: Switch slots or flash stock images
- **To backup**: Use `dd` commands via adb shell
- **To restore**: Flash backed-up images via fastboot

### Flash Workflow

```
Build kernel
    ↓
Artifacts in out/caimito/dist/
    ↓
Enter fastboot mode
    ↓
Flash boot images
    ↓
Flash module images
    ↓
Reboot device
    ↓
Verify kernel boot
```

### Safety Recommendations

1. **Always backup**: Backup partitions before flashing
2. **Test first**: Use `fastboot boot` for temporary testing
3. **Keep stock images**: Keep stock images for recovery
4. **Understand A/B slots**: Know which slot is active
5. **Verify compatibility**: Ensure images match kernel version
6. **Have recovery plan**: Know how to recover if flash fails

### Next Steps

- Understand fastboot protocol in detail
- Learn device-specific flashing procedures
- Understand bootloader unlocking requirements
- Learn recovery mode usage
- Understand OTA update process

