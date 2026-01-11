# Overview: Pixel 9 Pro XL Kernel Build Process

## Introduction

This document provides a comprehensive overview of the kernel build process for Pixel 9 Pro XL device (codename: **caimito**) in the Android kernel source tree. This overview focuses on understanding the standard way developers compile kernels from source locally while maintaining maximal compatibility with the complete system.

**Note**: For deep technical analysis of specific aspects, see the [Analysis](analysis/README.md) section. For step-by-step guides, see the [Guides](guides/README.md) section.

## Navigation

- [Main Documentation Index](README.md)
- [Deep Analysis](analysis/README.md)
- [Guides](guides/README.md)
- [Reference](reference/README.md)

---

## 1. Device Information

### 1.1 Device Codename
- **Primary codename**: `caimito`
- **Platform/SoC**: `zumapro` (extends from `zuma` platform)
- **Architecture**: `aarch64` (ARM64)

### 1.2 Device Variants Covered by "caimito"
The `caimito` device configuration supports multiple Pixel 9 device variants:
- **caiman** - One variant of Pixel 9 Pro XL
- **komodo** - Another variant  
- **tokay** - Another variant
- **ripcurrent/ripcurrent24/ripcurrentpro** - Development/reference board variants
- **husky** - Additional variant
- **shiba** - Additional variant

Each variant has its own:
- Device Tree Source (DTS) files
- `init.insmod.*.cfg` files for module loading
- Panel configurations (for display modules)

### 1.3 Build Script Location
- **Main build script**: `private/devices/google/caimito/build_caimito.sh`
- **Root-level wrapper**: `build_caimito.sh` (symlink or wrapper)

---

## 2. Build System Architecture

### 2.1 Build System: Bazel (Kleaf)

The kernel uses **Bazel** as the build system through the **Kleaf** framework:
- **Kleaf** = Kernel + Leaf (Android Platform Build with Bazel is called "Roboleaf")
- Replaces traditional `build.sh` approach
- Provides hermetic builds
- Better dependency tracking

**Key Documentation Location**: `build/kernel/kleaf/docs/`

### 2.2 Build Configuration Hierarchy

The build configuration follows a layered approach:

```
common/ (base GKI kernel)
  └── kernel_aarch64 (common Android kernel)
      └── zumapro/ (SoC platform layer)
          └── caimito/ (device-specific layer)
```

**Configuration Files:**
1. `common/build.config.gki.aarch64` - Base GKI configuration
2. `private/devices/google/zumapro/build.config.zumapro` - SoC platform config
3. `private/devices/google/caimito/build.config.caimito` - Device-specific config
4. `private/devices/google/caimito/caimito_defconfig` - Kernel defconfig fragments

### 2.3 BUILD.bazel Structure

Main build definition: `private/devices/google/caimito/BUILD.bazel`

**Key Components:**

1. **kernel_build_config** - Combines build configs from zumapro and caimito
2. **kernel_dtstree** - Device Tree Source files
3. **kernel_build** - Main kernel build target
4. **kernel_module_group** - External kernel modules
5. **kernel_images** - Boot images and partition images
6. **copy_to_dist_dir** - Distribution target

---

## 3. GKI (Generic Kernel Image) Architecture

### 3.1 What is GKI?

GKI is Android's Generic Kernel Image architecture that separates:
- **Base kernel** (GKI): Common Android kernel (in `//common`)
- **Device kernel extensions**: Device-specific modules and DTB overlays

This allows:
- Faster OTA updates (only GKI needs updating)
- Better security (kernel modules are signed)
- Compatibility guarantees via KMI (Kernel Module Interface)

### 3.2 Current Build Configuration

**Current State (as analyzed):**
- `kernel_build` uses `base_kernel = "//common:kernel_aarch64"` (line 82)
- This means the build **extends** the GKI base kernel
- Device-specific code is built as **modules** and **DTB overlays**

**Build artifacts include:**
- Device Tree Blob Overlays (DTBO files)
- External kernel modules (vendor_dlkm, system_dlkm)
- vendor_kernel_boot.img (contains vendor-specific kernel components)

### 3.3 Prebuilt GKI vs Custom Build

**Default Configuration:**
- `private/devices/google/common/device.bazelrc` sets:
  ```
  build --use_prebuilt_gki=true
  build --action_env=KLEAF_DOWNLOAD_BUILD_NUMBER_MAP="gki_prebuilts=13202960,..."
  ```
- This means **prebuilt GKI images are downloaded** by default from CI builds

**To Build from Source:**
- Must set `--use_prebuilt_gki=false` or omit the flag
- The base kernel `//common:kernel_aarch64` will be built from source
- This is the approach for custom kernel development

---

## 4. Build Process Flow

### 4.1 Entry Point

```bash
./build_caimito.sh [options]
```

**What it does:**
1. Sets up Bazel environment
2. Runs: `tools/bazel run --config=caimito //private/devices/google/caimito:zumapro_caimito_dist`
3. Builds and copies all artifacts to `out/caimito/dist/`

### 4.2 Build Stages

1. **Kernel Configuration**
   - Merges defconfig fragments:
     - `//private/devices/google/zumapro:zumapro_defconfig`
     - `private/devices/google/caimito/caimito_defconfig`
   - Applies Kconfig extensions
   - Generates `.config`

2. **Kernel Build**
   - Builds base kernel (`//common:kernel_aarch64`) if not using prebuilt
   - Builds device-specific extensions (modules, DTBs)
   - Outputs: DTB files, kernel modules

3. **Module Installation**
   - Runs `depmod` for module dependencies
   - Creates module directory structure

4. **Image Creation**
   - Builds `vendor_kernel_boot.img`
   - Builds `dtbo.img` (Device Tree Blob Overlay)
   - Builds `vendor_dlkm.img` (vendor Dynamic Loadable Kernel Modules)
   - Builds `system_dlkm.img` (system Dynamic Loadable Kernel Modules)
   - Builds initramfs if configured

5. **Distribution**
   - Copies all artifacts to `out/caimito/dist/`

### 4.3 Build Artifacts

**Key Output Files:**
- `Image` - Uncompressed kernel image
- `Image.gz`, `Image.lz4` - Compressed kernel images
- `vmlinux` - ELF kernel image (for debugging)
- `System.map` - Kernel symbol map
- `dtb.img` - Device Tree Blob
- `dtbo.img` - Device Tree Blob Overlay
- `vendor_kernel_boot.img` - Vendor kernel boot image
- `vendor_dlkm.img` - Vendor DLKM partition image
- `system_dlkm.img` - System DLKM partition image
- `*.ko` files - Kernel modules

---

## 5. Device-Specific Components

### 5.1 Device Tree Sources (DTS)

Location: `private/devices/google/caimito/dts/`

**Structure:**
- Common files: `zumapro-caimito-common.dtsi`
- Variant-specific: `zumapro-caiman-*.dts`, `zumapro-komodo-*.dts`, `zumapro-tokay-*.dts`
- Board variants: evt1, dvt1, pvt1, mp (Mass Production)

**DTBO Files:** Defined in `constants.bzl` as `CAIMITO_DTBOS`

### 5.2 External Kernel Modules

**Module Groups:**
1. **SoC Modules** (`//private/devices/google/zumapro:kernel_ext_modules`)
   - Must load first (module dependency order issue noted in code)

2. **Device-Specific Modules:**
   - Display panels (caimito-specific panels)
   - Touch controllers
   - Fingerprint sensors
   - Audio amplifiers
   - Bluetooth/WiFi modules
   - UWB (Ultra-Wideband)
   - NFC
   - GPS/GNSS

**Module Lists:**
- `vendor_ramdisk.modules.caimito` - Modules loaded from vendor ramdisk
- `vendor_dlkm.modules` - Modules in vendor_dlkm partition
- `system_dlkm.modules` - Modules in system_dlkm partition
- `vendor_dlkm.blocklist.caimito` - Modules to exclude from vendor_dlkm

### 5.3 Module Loading Configuration

**insmod_cfg Files:**
- `insmod_cfg/init.insmod.caiman.cfg`
- `insmod_cfg/init.insmod.komodo.cfg`
- `insmod_cfg/init.insmod.tokay.cfg`
- etc.

These specify module loading order for each device variant.

---

## 6. Building Custom Kernel (Without Prebuilt GKI)

### 6.1 Standard Approach (Extending GKI)

**Current default approach:**
- Uses base GKI kernel from `//common:kernel_aarch64`
- Builds device-specific extensions as modules
- Maintains KMI (Kernel Module Interface) compatibility
- **Recommended** for most use cases

**Build Command:**
```bash
# Use prebuilt GKI (default)
./build_caimito.sh

# Build GKI from source
./build_caimito.sh --use_prebuilt_gki=false
```

### 6.2 Full Kernel Build (Advanced)

**When to use:**
- Making changes to core kernel source files (e.g., `init/main.c`)
- Need full control over kernel configuration
- Developing kernel features that require kernel source modifications

**What needs to change:**
- Remove `base_kernel` from `kernel_build` in BUILD.bazel
- Remove `kmi_symbol_list` (KMI not applicable for full builds)
- Add kernel image outputs (Image, Image.gz, Image.lz4, vmlinux)
- Set `trim_nonlisted_kmi = False`
- Build full kernel from device-specific source

**Note:** This approach is **NOT** the standard approach and breaks GKI compatibility. Should only be used when absolutely necessary.

### 6.3 Building for Root/Development

**Key Considerations:**
1. **Kernel changes for root:**
   - May need to modify kernel to disable SELinux enforcing mode
   - May need to add root-related kernel features
   - These changes might require full kernel build

2. **Compatibility:**
   - Modifying base kernel breaks KMI compatibility
   - May cause issues with system updates
   - Modules compiled against different kernel may not load

3. **Testing:**
   - Need to flash and test on device
   - Should maintain backup of working kernel

---

## 7. Flashing Process

### 7.1 Flash Script Example

Based on documentation and build system, typical flash sequence:

```bash
# Flash static partitions (slot_b for A/B devices)
fastboot flash boot_b out/caimito/dist/boot.img
fastboot flash vendor_kernel_boot_b out/caimito/dist/vendor_kernel_boot.img
fastboot flash dtbo_b out/caimito/dist/dtbo.img

# Reboot to fastboot for dynamic partitions
fastboot reboot fastboot

# Flash dynamic partitions
fastboot flash vendor_dlkm_b out/caimito/dist/vendor_dlkm.img
fastboot flash system_dlkm_b out/caimito/dist/system_dlkm.img

# Reboot
fastboot reboot
```

**Note:** Use appropriate slot (_a or _b) based on current active slot.

### 7.2 Partition Structure

**Static Partitions:**
- `boot` - Contains GKI kernel + initramfs
- `vendor_kernel_boot` - Vendor-specific kernel components
- `dtbo` - Device Tree Blob Overlays

**Dynamic Partitions:**
- `vendor_dlkm` - Vendor Dynamic Loadable Kernel Modules
- `system_dlkm` - System Dynamic Loadable Kernel Modules

---

## 8. Key Configuration Files

### 8.1 Build Configuration Files

1. **build.config.caimito**
   - Location: `private/devices/google/caimito/build.config.caimito`
   - Content: `BCMDHD=4390` (WiFi chip model)
   - Extends: `build.config.zumapro`

2. **caimito_defconfig**
   - Location: `private/devices/google/caimito/caimito_defconfig`
   - Kernel configuration fragments
   - Merged with zumapro defconfig

3. **Kconfig.ext.caimito**
   - Kconfig extensions for caimito-specific options

### 8.2 Bazel Configuration

1. **device.bazelrc**
   - Location: `private/devices/google/common/device.bazelrc`
   - Sets prebuilt GKI flags
   - Configures build variants (factory, testing, kunit, 16k page size)

2. **caimito BUILD.bazel**
   - Main build definition
   - Defines all build targets

---

## 9. Build Environment

### 9.1 Prerequisites

- Bazel (provided via `tools/bazel` in repo)
- JDK 11 (prebuilt in repo)
- Python (for build scripts)
- Clang toolchain (prebuilt in repo)
- Standard Linux build tools

### 9.2 Build Commands

**Standard build:**
```bash
./build_caimito.sh
```

**Build with custom options:**
```bash
# Build without prebuilt GKI
./build_caimito.sh --use_prebuilt_gki=false

# Build with debugging
./build_caimito.sh --config=testing

# Build for factory
./build_caimito.sh --config=factory

# Build 16k page size kernel
./build_caimito.sh --config=16k
```

**Direct Bazel commands:**
```bash
# Build kernel only
tools/bazel build --config=caimito //private/devices/google/caimito:kernel

# Build and distribute
tools/bazel run --config=caimito //private/devices/google/caimito:zumapro_caimito_dist

# Build without prebuilt GKI
tools/bazel build --config=caimito --use_prebuilt_gki=false //private/devices/google/caimito:kernel
```

---

## 10. Important Notes and Considerations

### 10.1 GKI Compatibility

- **Standard approach**: Extending GKI base kernel maintains compatibility
- **Full kernel build**: Breaks GKI/KMI compatibility
- **Recommendation**: Use GKI extension approach unless absolutely necessary

### 10.2 Module Signing

- Modules are signed during build
- System requires signed modules (unless bootloader is unlocked)
- Module signing keys are generated during build

### 10.3 KMI (Kernel Module Interface)

- KMI ensures module compatibility across kernel versions
- When using `base_kernel`, KMI symbol lists are enforced
- Violations cause build failures
- Full kernel builds don't enforce KMI (since there's no base kernel)

### 10.4 Development Workflow

**Recommended workflow:**
1. Make changes to device-specific code (modules, DTS)
2. Build with `--use_prebuilt_gki=false` to build GKI from source
3. Test on device
4. Iterate

**For root/kernel testing:**
1. Consider if changes can be done via modules (preferred)
2. If kernel source changes needed, use full kernel build
3. Be aware of compatibility implications
4. Test thoroughly before daily use

### 10.5 Multi-Device Build

- `build_caimito.sh` builds for **all** caimito variants (caiman, komodo, tokay, etc.)
- All DTBO files are built
- Device-specific modules are built
- At runtime, device tree matches the actual hardware

---

## 11. Areas Requiring Further Investigation

### 11.1 Incomplete Understanding

1. **Boot.img composition:**
   - Exact relationship between boot.img and vendor_kernel_boot.img
   - How GKI and vendor components are combined at boot

2. **Module loading sequence:**
   - Exact module loading order at runtime
   - How insmod_cfg files are used

3. **KMI symbol list:**
   - What symbols are in the KMI list
   - How to add new symbols if needed

4. **Full kernel build:**
   - Complete procedure for full kernel build (if needed)
   - Trade-offs and implications

5. **Root-specific requirements:**
   - What kernel changes are needed for root
   - How to maintain system stability with root

6. **Build optimization:**
   - How to speed up builds during development
   - Incremental build strategies

### 11.2 Documentation Gaps

- Detailed flash procedure for Pixel 9 Pro XL
- Root setup and kernel modifications
- Debugging kernel issues
- Performance testing procedures
- Module development workflow

---

## 12. References and Documentation

### 12.1 Key Documentation Files

- `build/kernel/kleaf/docs/kleaf.md` - Kleaf overview
- `build/kernel/kleaf/docs/impl.md` - Building kernels and modules
- `build/kernel/kleaf/docs/download_prebuilt.md` - Prebuilt GKI usage
- `build/kernel/kleaf/docs/dist.md` - Distribution process
- `build/kernel/kleaf/docs/build_configs.md` - Build configuration reference

### 12.2 Code Locations

- Build scripts: `private/devices/google/caimito/`
- SoC platform: `private/devices/google/zumapro/`
- Common kernel: `common/`
- Build system: `build/kernel/kleaf/`
- Device trees: `private/devices/google/caimito/dts/`
- Modules: `private/google-modules/`

---

## 13. Summary

### 13.1 Current Build Process

1. **Default**: Downloads prebuilt GKI, builds device-specific extensions
2. **Custom**: Can build GKI from source with `--use_prebuilt_gki=false`
3. **Full kernel**: Requires BUILD.bazel modifications (not standard)

### 13.2 Recommendations for Your Use Case

**For root and kernel testing:**
1. Start with standard approach (extending GKI)
2. Build with `--use_prebuilt_gki=false` to build from source
3. Make changes to modules/DTS where possible
4. Only modify kernel source if absolutely necessary
5. Test thoroughly on device
6. Maintain backups of working kernel

**Compatibility:**
- GKI extension approach maintains maximum compatibility
- Full kernel builds may cause issues with system updates
- Consider impact on OTA updates

**Development efficiency:**
- Use `--config=local` for faster builds (reduces sandboxing)
- Use `--config=fast` for incremental builds
- Clean build cache when needed: `tools/bazel clean`

---

## Document Status

**Version**: Initial Analysis v1.0
**Date**: Analysis Date
**Status**: Preliminary - Requires further validation
**Next Steps**: 
- Validate build process with actual builds
- Test flashing procedure
- Document root-specific requirements
- Investigate full kernel build procedure if needed

---

**Note**: This analysis is based on code inspection and documentation review. Actual build behavior should be validated through testing. The codebase is large and complex, and some details may require deeper investigation or actual build experience to fully understand.

