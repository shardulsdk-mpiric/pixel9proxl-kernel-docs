# Deep Analysis #5: Boot Process and Partition Structure

## Overview

This document provides a deep technical analysis of the boot process and partition structure for the Pixel 9 Pro XL kernel build. This includes understanding what boot.img and vendor_kernel_boot.img contain, how they are created, how they combine at boot, partition layout, and how the bootloader loads and executes the kernel.

## Navigation

- [Analysis Index](README.md)
- [Main Documentation Index](../README.md)
- [Overview](../overview.md)

---

## Table of Contents

1. [Boot Images Overview](#1-boot-images-overview)
2. [boot.img Structure](#2-bootimg-structure)
3. [vendor_kernel_boot.img Structure](#3-vendor_kernel_bootimg-structure)
4. [Boot Image Creation](#4-boot-image-creation)
5. [Partition Structure](#5-partition-structure)
6. [Boot Process](#6-boot-process)
7. [Summary and Key Takeaways](#7-summary-and-key-takeaways)

---

## 1. Boot Images Overview

### 1.1 What are Boot Images?

Boot images are container formats that package the kernel, device tree, ramdisk, and other components needed for the device to boot. In the Pixel 9 Pro XL kernel build, multiple boot images are created for different purposes.

### 1.2 Types of Boot Images

From `private/devices/google/caimito/BUILD.bazel`:

```197:223:private/devices/google/caimito/BUILD.bazel
kernel_images(
    name = "kernel_images",
    base_kernel_images = "//common:kernel_aarch64_images",
    # TODO: b/362627953 - Remove dtb.img
    boot_image_outs = [
        "dtb.img",
        "vendor_kernel_boot.img",
    ],
    build_dtbo = True,
    build_initramfs = True,
    build_system_dlkm = True,
    build_vendor_dlkm = True,
    build_vendor_kernel_boot = True,
    dtbo_srcs = [":kernel/" + dtbo for dtbo in CAIMITO_DTBOS],
    kernel_build = ":kernel",
    kernel_modules_install = ":kernel_modules_install",
    modules_list = ":vendor_ramdisk_modules_list",
    ramdisk_compression = "lz4",
    system_dlkm_modules_list = ":system_dlkm_modules_list",
    system_dlkm_props = ":system_dlkm_props",
    vendor_dlkm_archive = True,
    vendor_dlkm_etc_files = [":insmod_cfgs"],
    vendor_dlkm_modules_blocklist = ":vendor_dlkm_modules_blocklist",
    vendor_dlkm_modules_list = ":vendor_dlkm_modules_list",
    vendor_dlkm_props = ":vendor_dlkm_props",
    deps = ["//private/devices/google/common:sepolicy"],
)
```

The build creates several boot images:
- **boot.img**: Main boot image (from base kernel)
- **vendor_kernel_boot.img**: Vendor kernel boot image (device-specific)
- **dtbo.img**: Device tree overlay image
- **dtb.img**: Device tree blob image
- **vendor_dlkm.img**: Vendor Dynamic Loadable Kernel Modules partition
- **system_dlkm.img**: System Dynamic Loadable Kernel Modules partition

---

## 2. boot.img Structure

### 2.1 What is boot.img?

`boot.img` is the main boot image that contains the GKI (Generic Kernel Image) kernel, device tree, and ramdisk. This image is created by the base kernel build and is common across all devices using the same GKI version.

### 2.2 Contents of boot.img

The `boot.img` contains:
- **Kernel Image**: The GKI kernel binary (Image.lz4 or Image.gz)
- **Device Tree**: Device tree blob (DTB) for hardware description
- **Ramdisk**: Initial ramdisk (initramfs) for early boot
- **Boot Header**: Image format metadata

### 2.3 boot.img Creation

The `boot.img` is created by the base kernel build (`//common:kernel_aarch64_images`). It's referenced in the caimito build as `base_kernel_images`.

---

## 3. vendor_kernel_boot.img Structure

### 3.1 What is vendor_kernel_boot.img?

`vendor_kernel_boot.img` is the vendor-specific boot image that contains device-specific kernel components. It's created by the device kernel build and complements the base `boot.img`.

### 3.2 Contents of vendor_kernel_boot.img

The `vendor_kernel_boot.img` contains:
- **Vendor Ramdisk**: Device-specific ramdisk with vendor modules
- **Device Tree Overlays**: Device tree overlay blobs (DTBO)
- **Vendor Modules**: Early-boot kernel modules (from `vendor_ramdisk_modules_list`)

### 3.3 vendor_kernel_boot.img Creation

From BUILD.bazel:
- `build_vendor_kernel_boot = True`: Enables vendor kernel boot image creation
- `modules_list = ":vendor_ramdisk_modules_list"`: Specifies modules to include
- `ramdisk_compression = "lz4"`: Uses LZ4 compression for ramdisk

---

## 4. Boot Image Creation

### 4.1 How are Boot Images Created?

Boot images are created by the `kernel_images` rule using `mkbootimg` tool. The process involves:
1. **Collecting components**: Kernel Image, DTBs, modules, ramdisk
2. **Creating ramdisk**: Packaging modules and configuration files
3. **Building images**: Using mkbootimg to create image files

### 4.2 Image Build Process

From the `kernel_images` configuration:
- **boot.img**: Created by base kernel (`base_kernel_images`)
- **vendor_kernel_boot.img**: Created by device kernel build
- **dtbo.img**: Device tree overlay image
- **dtb.img**: Device tree blob image
- **vendor_dlkm.img**: Vendor modules partition
- **system_dlkm.img**: System modules partition

---

## 5. Partition Structure

### 5.1 Partition Layout

Android devices use multiple partitions for different purposes:

- **boot**: Contains boot.img (GKI kernel + base ramdisk)
- **vendor_boot**: Contains vendor_kernel_boot.img (vendor ramdisk + DTBOs)
- **vendor_dlkm**: Contains vendor_dlkm.img (vendor kernel modules)
- **system_dlkm**: Contains system_dlkm.img (system kernel modules)
- **dtbo**: Contains dtbo.img (device tree overlays)

### 5.2 Partition Usage

- **boot partition**: Loaded first, contains GKI kernel
- **vendor_boot partition**: Loaded second, contains vendor-specific components
- **vendor_dlkm partition**: Contains vendor modules loaded at runtime
- **system_dlkm partition**: Contains system modules loaded at runtime

---

## 6. Boot Process

### 6.1 How Does the Bootloader Load the Kernel?

The bootloader follows this sequence:

1. **Load boot.img**: Bootloader loads boot.img from boot partition
2. **Extract kernel**: Extracts GKI kernel Image from boot.img
3. **Load vendor_boot.img**: Loads vendor_kernel_boot.img from vendor_boot partition
4. **Combine components**: Combines base kernel with vendor ramdisk and DTBOs
5. **Start kernel**: Executes kernel with combined configuration

### 6.2 Module Loading at Boot

Modules are loaded in stages:
1. **Vendor Ramdisk Modules**: Loaded early from vendor_kernel_boot.img
2. **Vendor DLKM Modules**: Loaded from vendor_dlkm partition after boot
3. **System DLKM Modules**: Loaded from system_dlkm partition after system init

### 6.3 Device Tree Processing

Device trees are processed as follows:
1. **Base DTB**: From boot.img (describes base hardware)
2. **DTBOs**: From vendor_kernel_boot.img (overlays for device variants)
3. **Combined DTB**: Kernel combines base DTB with applicable DTBOs

---

## 7. Summary and Key Takeaways

### Key Concepts

1. **Boot Images**:
   - boot.img: Main boot image with GKI kernel
   - vendor_kernel_boot.img: Vendor-specific boot image
   - Multiple images for different purposes (dtbo, vendor_dlkm, system_dlkm)

2. **Image Contents**:
   - boot.img: GKI kernel + base ramdisk
   - vendor_kernel_boot.img: Vendor ramdisk + DTBOs + vendor modules
   - vendor_dlkm.img: Vendor kernel modules
   - system_dlkm.img: System kernel modules

3. **Boot Process**:
   - Bootloader loads boot.img first
   - Then loads vendor_kernel_boot.img
   - Combines components to start kernel
   - Modules loaded in stages (vendor ramdisk → vendor dlkm → system dlkm)

4. **Partition Structure**:
   - boot: Main boot image
   - vendor_boot: Vendor boot image
   - vendor_dlkm: Vendor modules
   - system_dlkm: System modules

### Practical Implications

- **To modify boot process**: Modify vendor_kernel_boot.img contents
- **To add modules**: Add to appropriate modules list (vendor_ramdisk, vendor_dlkm, system_dlkm)
- **To change device tree**: Modify DTBOs in vendor_kernel_boot.img
- **To debug boot**: Check logs from each boot stage

### Next Steps

- Understand module loading sequence in detail
- Understand device tree overlay processing
- Understand ramdisk creation process
- Understand partition flashing procedures

