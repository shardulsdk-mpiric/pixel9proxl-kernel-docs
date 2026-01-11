# Deep Analysis #4: Module System

## Overview

This document provides a deep technical analysis of the kernel module system for the Pixel 9 Pro XL kernel build. This includes understanding how modules are structured, how they're built, module signing, loading sequence, dependencies, and the different module types (vendor_dlkm, system_dlkm, vendor_ramdisk).

## Navigation

- [Analysis Index](README.md)
- [Main Documentation Index](../README.md)
- [Overview](../overview.md)

---

## Table of Contents

1. [Module Structure and Build](#1-module-structure-and-build)
2. [Module Signing](#2-module-signing)
3. [Module Loading and Dependencies](#3-module-loading-and-dependencies)
4. [Module Types and Locations](#4-module-types-and-locations)
5. [Module Loading Configuration](#5-module-loading-configuration)
6. [Summary and Key Takeaways](#6-summary-and-key-takeaways)

---

## 1. Module Structure and Build

### 1.1 What are Kernel Modules?

Kernel modules are loadable pieces of code that extend the kernel's functionality. They can be loaded and unloaded at runtime without rebooting the system. In the Pixel 9 Pro XL kernel build, modules are built separately from the kernel and packaged into different images.

### 1.2 How are Modules Built?

Modules are built using the `kernel_module` rule in Bazel. Multiple modules can be grouped using `kernel_module_group`. The build process compiles module source code into `.ko` (kernel object) files.

### 1.3 Module Build Example

From `private/devices/google/caimito/BUILD.bazel`:

```97:134:private/devices/google/caimito/BUILD.bazel
kernel_module_group(
    name = "kernel_ext_modules",
    srcs = [
        # FIXME: SoC modules need to be loaded first or the device will crash.
        #        Module dependencies should not rely on the order. This needs be fixed.
        "//private/devices/google/zumapro:kernel_ext_modules",
    ] + [
        # keep sorted
        "//private/google-modules/amplifiers/cs35l41",
        "//private/google-modules/amplifiers/cs40l26",
        "//private/google-modules/amplifiers/snd_soc_wm_adsp:snd-soc-wm-adsp",
        "//private/google-modules/bluetooth/broadcom:bluetooth.broadcom",
        "//private/google-modules/display/common/panel_tests",
        "//private/google-modules/display/common/panels",
        "//private/google-modules/display/panels/caimito:panel-gs-cm4",
        "//private/google-modules/display/panels/caimito:panel-gs-km4",
        "//private/google-modules/display/panels/caimito:panel-gs-tk4a",
        "//private/google-modules/display/panels/caimito:panel-gs-tk4b",
        "//private/google-modules/display/panels/caimito:panel-gs-tk4c",
        "//private/google-modules/display/panels/caimito:panel-gs-tk4c-test",
        "//private/google-modules/fingerprint/qcom/qfs4008:qbt_handler",
        "//private/google-modules/gps/broadcom/bcm47765",
        "//private/google-modules/nfc",
```

This shows a `kernel_module_group` that includes multiple module targets from different locations.

---

## 2. Module Signing

### 2.1 Why are Modules Signed?

Module signing provides security by ensuring modules haven't been tampered with. In the Pixel 9 Pro XL kernel build, modules can be signed using a module signing key.

### 2.2 How are Modules Signed?

Modules are signed during the build process using the `module_signing_key` attribute in `kernel_build`. The signing process happens after compilation but before installation.

---

## 3. Module Loading and Dependencies

### 3.1 Module Loading Sequence

Modules are loaded in a specific order to satisfy dependencies. The build system uses module lists and dependency information to determine loading order.

### 3.2 Module Dependencies

Modules can depend on other modules. The build system tracks these dependencies to ensure modules are loaded in the correct order. From the code comment in BUILD.bazel:

```100:101:private/devices/google/caimito/BUILD.bazel
        # FIXME: SoC modules need to be loaded first or the device will crash.
        #        Module dependencies should not rely on the order. This needs be fixed.
```

This indicates that module loading order matters, and SoC modules must be loaded before device-specific modules.

---

## 4. Module Types and Locations

### 4.1 Vendor DLKM Modules

Vendor DLKM (Dynamic Loadable Kernel Modules) modules are vendor-specific modules that are loaded from the vendor_dlkm partition.

### 4.2 System DLKM Modules

System DLKM modules are system-level modules loaded from the system_dlkm partition.

### 4.3 Vendor Ramdisk Modules

Vendor ramdisk modules are loaded early in the boot process from the vendor ramdisk (in vendor_kernel_boot.img).

---

## 5. Module Loading Configuration

### 5.1 Module Lists

Module lists specify which modules go into which partition. From BUILD.bazel:

```148:174:private/devices/google/caimito/BUILD.bazel
create_file(
    name = "vendor_ramdisk_modules_list",
    srcs = [
        # do not sort
        "//private/devices/google/zumapro:vendor_ramdisk_modules_list",
        "vendor_ramdisk.modules.caimito",
    ],
    out = "vendor_ramdisk.modules",
)

create_file(
    name = "system_dlkm_modules_list",
    srcs = ["//private/devices/google/zumapro:system_dlkm_modules_list"],
    out = "system_dlkm.modules",
)

create_file(
    name = "vendor_dlkm_modules_list",
    srcs = ["//private/devices/google/zumapro:vendor_dlkm_modules_list"],
    out = "vendor_dlkm.modules",
)
```

### 5.2 Module Blocklists

Module blocklists specify modules that should NOT be included:

```176:184:private/devices/google/caimito/BUILD.bazel
create_file(
    name = "vendor_dlkm_modules_blocklist",
    srcs = [
        # do not sort
        "//private/devices/google/zumapro:vendor_dlkm_modules_blocklist",
        "vendor_dlkm.blocklist.caimito",
    ],
    out = "vendor_dlkm.blocklist",
)
```

### 5.3 Insmod Configuration Files

Insmod configuration files control how modules are loaded. From BUILD.bazel:

```192:195:private/devices/google/caimito/BUILD.bazel
filegroup(
    name = "insmod_cfgs",
    srcs = glob(["insmod_cfg/*.cfg"]),
)
```

These files are included in vendor_dlkm:

```217:217:private/devices/google/caimito/BUILD.bazel
    vendor_dlkm_etc_files = [":insmod_cfgs"],
```

---

## 6. Summary and Key Takeaways

### Key Concepts

1. **Module Structure**:
   - Modules are built using `kernel_module` and `kernel_module_group` rules
   - Compiled into `.ko` files
   - Can be signed for security

2. **Module Loading**:
   - Modules have dependencies that determine loading order
   - SoC modules must load before device modules
   - Loading order is critical for system stability

3. **Module Types**:
   - Vendor DLKM: Vendor-specific modules in vendor_dlkm partition
   - System DLKM: System-level modules in system_dlkm partition
   - Vendor Ramdisk: Early boot modules in vendor_kernel_boot.img

4. **Module Configuration**:
   - Module lists specify which modules go where
   - Blocklists exclude specific modules
   - Insmod config files control loading behavior

### Practical Implications

- To add a module: Add it to `kernel_ext_modules` group
- To change module location: Update the appropriate modules list
- To exclude a module: Add it to the blocklist
- To control loading: Modify insmod_cfg files

### Next Steps

- Understand boot process and how modules are loaded at runtime
- Understand partition structure and how modules are deployed
- Understand module dependencies in detail
- Understand module signing process

