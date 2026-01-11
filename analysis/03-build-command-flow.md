# Deep Analysis #3: Build Command Flow - From Script to Artifacts

## Overview

This document provides a deep technical analysis of the build command flow for the Pixel 9 Pro XL kernel, from the initial `build_caimito.sh` script execution through to the final build artifacts. This includes understanding how Bazel processes targets, the build order, prebuilt GKI handling, and how to build from source.

## Navigation

- [Analysis Index](README.md)
- [Main Documentation Index](../README.md)
- [Overview](../overview.md)

---

## Table of Contents

1. [Build Script Entry Point](#1-build-script-entry-point)
2. [Bazel Target Execution](#2-bazel-target-execution)
3. [Build Dependency Graph](#3-build-dependency-graph)
4. [Prebuilt GKI Handling](#4-prebuilt-gki-handling)
5. [Build Artifacts and Distribution](#5-build-artifacts-and-distribution)
6. [Summary and Key Takeaways](#6-summary-and-key-takeaways)

---

## 1. Build Script Entry Point

### 1.1 What is build_caimito.sh?

The `build_caimito.sh` script is the entry point for building the Pixel 9 Pro XL (caimito) kernel. It's a wrapper script that calls Bazel with the appropriate configuration.

### 1.2 Script Structure

From `build_caimito.sh`:

```bash
exec tools/bazel run \
    ${parameters} \
    --config=stamp \
    --config=caimito \
    //private/devices/google/caimito:zumapro_caimito_dist "$@"
```

**Key components:**
- `tools/bazel`: The Bazel build tool
- `run`: Execute a target as a runnable program
- `--config=stamp`: Configuration for reproducible builds (stamp build information)
- `--config=caimito`: Device-specific configuration
- `//private/devices/google/caimito:zumapro_caimito_dist`: The target to build
- `"$@"`: Pass through any additional arguments

### 1.3 What Does build_caimito.sh Do?

The script:
1. **Sets up environment**: Uses Bazel configuration files
2. **Runs Bazel**: Executes the `zumapro_caimito_dist` target
3. **Handles output**: Bazel builds and runs the target, producing artifacts

---

## 2. Bazel Target Execution

### 2.1 What is zumapro_caimito_dist?

The `zumapro_caimito_dist` target is a `copy_to_dist_dir` rule that collects all build artifacts and copies them to the distribution directory.

### 2.2 Target Definition

From `private/devices/google/caimito/BUILD.bazel`:

```264:267:private/devices/google/caimito/BUILD.bazel
alias(
    name = "zumapro_caimito_dist",
    actual = ":dist",
)
```

The `zumapro_caimito_dist` is an alias that points to the `:dist` target:

```225:247:private/devices/google/caimito/BUILD.bazel
copy_to_dist_dir(
    name = "dist",
    data = [
        # keep sorted
        ":insmod_cfgs",
        ":kernel",
        ":kernel_images",
        ":kernel_modules_install",
        ":kernel_unstripped_modules_archive",
        ":merged_kernel_uapi_headers",
        "//common:android/abi_gki_aarch64_pixel",
        "//common:kernel_aarch64",
        "//common:kernel_aarch64_gki_boot_image",
        "//common:kernel_aarch64_headers",
        "//private/devices/google/common:kernel_gki_modules",
    ] + select({
        "//private/devices/google/common:enable_download_fips140": ["@gki_prebuilt_fips140//fips140.ko"],
        "//private/devices/google/common:disable_download_fips140": [],
    }),
    dist_dir = "out/caimito/dist",
    flat = True,
    log = "info",
)
```

**Key components:**
- `copy_to_dist_dir`: Bazel rule that copies artifacts to a distribution directory
- `data`: List of dependencies to copy (includes kernel, images, modules, headers, etc.)
- `dist_dir`: Output directory for artifacts (`out/caimito/dist`)
- `flat`: Flatten directory structure (copy files directly to dist_dir)

### 2.3 What Gets Built?

The `zumapro_caimito_dist` target depends on:
1. **kernel_images**: Boot images (boot.img, vendor_kernel_boot.img, etc.)
2. **kernel_modules_install**: Installed kernel modules
3. **kernel_unstripped_modules_archive**: Archive of unstripped modules for debugging

---

## 3. Build Dependency Graph

### 3.1 Dependency Chain

The build dependency chain is:

```
zumapro_caimito_dist
    ↓
kernel_images
    ↓
kernel_build
    ↓
kernel_config
    ↓
kernel_env
    ↓
kernel_build_config
```

### 3.2 Build Order

When Bazel executes the build:

1. **kernel_build_config**: Concatenates build.config files
2. **kernel_env**: Creates build environment script (depends on kernel_build_config)
3. **kernel_config**: Generates .config file from defconfig + fragments (depends on kernel_env)
4. **kernel_build**: Compiles kernel, modules, device trees (depends on kernel_config)
5. **kernel_modules_install**: Installs modules (depends on kernel_build)
6. **kernel_unstripped_modules_archive**: Archives unstripped modules (depends on kernel_build)
7. **kernel_images**: Creates boot images from kernel outputs (depends on kernel_build and kernel_modules_install)
8. **dist** (copy_to_dist_dir): Copies everything to dist_dir (depends on all above)

**Note**: Some targets can build in parallel if they don't depend on each other. For example:
- `kernel_modules_install` and `kernel_unstripped_modules_archive` can both run after `kernel_build` completes
- `kernel_images` must wait for both `kernel_build` and `kernel_modules_install`

### 3.3 Parallel Execution

Bazel can execute independent targets in parallel. For example:
- `kernel_config` can run while `kernel_build_config` runs
- Module builds can run in parallel with kernel build (if not dependent)

---

## 4. Prebuilt GKI Handling

### 4.1 What is Prebuilt GKI?

Prebuilt GKI (Generic Kernel Image) refers to a pre-compiled GKI kernel binary that is downloaded instead of being built from source. This speeds up builds by reusing a common kernel binary.

### 4.2 How is Prebuilt GKI Used?

When using `base_kernel` in `kernel_build`, the device kernel build:
- **Does NOT compile the kernel Image** (the main kernel binary)
- **Only builds modules and device trees**
- **Uses the GKI Image from the base kernel**

From `private/devices/google/caimito/BUILD.bazel`:

```70:95:private/devices/google/caimito/BUILD.bazel
kernel_build(
    name = "kernel",
    srcs = [":kernel_sources"],
    outs = [
        ".config",
    ] + [
        "zuma/{}".format(dtb)
        for dtb in ZUMA_DTBS + ZUMA_DPM_DTBOS
    ] + [
        "zumapro/{}".format(dtb)
        for dtb in ZUMAPRO_DTBS + ZUMAPRO_DPM_DTBOS
    ] + CAIMITO_DTBOS,
    base_kernel = "//common:kernel_aarch64",
    build_config = ":build_config",
    collect_unstripped_modules = True,
    defconfig_fragments = [":defconfig_fragments"],
    dtstree = ":dtstree",
    kconfig_ext = ":kconfig_ext",
    kmi_symbol_list = "//common:android/abi_gki_aarch64_pixel",
    make_goals = [
        "modules",
        "dtbs",
    ],
    module_outs = ZUMAPRO_MODULE_OUTS,
    strip_modules = True,
)
```

**Key point**: `make_goals = ["modules", "dtbs"]` - Note that `"Image"` is NOT in the list. This means the kernel Image binary is NOT built by this target. It comes from the `base_kernel`.

### 4.3 How to Disable Prebuilt GKI?

To build the full kernel from source (including the Image), you need to:

1. **Remove base_kernel attribute**: Don't extend a base kernel
2. **Add Image to make_goals**: Include "Image" in the build targets
3. **Build full kernel**: Compile everything from source

**Warning**: This will take significantly longer and may not be compatible with the device's bootloader/system if not done correctly.

---

## 5. Build Artifacts and Distribution

### 5.1 What Artifacts Are Produced?

The build produces several types of artifacts:

1. **Kernel Images**:
   - `Image`: Compressed kernel binary
   - `Image.lz4`: LZ4-compressed kernel binary
   - `Image.gz`: Gzip-compressed kernel binary

2. **Boot Images**:
   - `boot.img`: Main boot image (kernel + ramdisk)
   - `vendor_kernel_boot.img`: Vendor kernel boot image
   - `dtbo.img`: Device tree overlay image
   - `dtb.img`: Device tree blob image

3. **Modules**:
   - `.ko` files: Kernel modules
   - Modules in vendor_dlkm.img
   - Modules in system_dlkm.img

4. **Debug Artifacts**:
   - `vmlinux`: Unstripped kernel binary
   - `System.map`: Kernel symbol map
   - Unstripped modules archive

### 5.2 Distribution Directory Structure

Artifacts are copied to `out/dist/` (or custom `dist_dir`) with structure like:

```
out/dist/
├── boot.img
├── vendor_kernel_boot.img
├── dtbo.img
├── dtb.img
├── vendor_dlkm.img
├── system_dlkm.img
└── ...
```

### 5.3 How Are Artifacts Used?

These artifacts can be:
- **Flashed to device**: Using fastboot or other flashing tools
- **Used for testing**: Load kernel and modules for testing
- **Debugged**: Use unstripped binaries for debugging
- **Distributed**: Share with other developers or devices

---

## 6. Summary and Key Takeaways

### Key Concepts

1. **Build Script**:
   - `build_caimito.sh` is a wrapper that calls Bazel
   - Executes `zumapro_caimito_dist` target
   - Uses device-specific configuration

2. **Bazel Targets**:
   - Targets form a dependency graph
   - Bazel executes targets in dependency order
   - Independent targets can run in parallel

3. **Prebuilt GKI**:
   - Default behavior uses prebuilt GKI Image
   - Only modules and DTBs are built from source
   - Full kernel build requires removing `base_kernel`

4. **Build Artifacts**:
   - Multiple image types (boot, vendor, DTB, etc.)
   - Modules in various formats
   - Debug artifacts for troubleshooting

### Build Flow Summary

```
build_caimito.sh
    → bazel run zumapro_caimito_dist
        → kernel_images (needs kernel_build)
            → kernel_build (needs kernel_config)
                → kernel_config (needs kernel_env)
                    → kernel_env (needs kernel_build_config)
                        → kernel_build_config (concatenates build.config files)
        → kernel_modules_install
        → kernel_unstripped_modules_archive
    → copy_to_dist_dir (collects all artifacts)
    → out/dist/ (final artifacts)
```

### Practical Implications

- **To build**: Run `./build_caimito.sh`
- **To customize**: Modify BUILD.bazel targets
- **To debug**: Check individual target outputs
- **To flash**: Use artifacts in `out/dist/`
- **To build from source**: Remove `base_kernel` and add "Image" to `make_goals`

### Next Steps

- Understand module system and how modules are built
- Understand boot process and how images are created
- Understand flashing process and how to deploy to device
- Understand how to customize builds for development

