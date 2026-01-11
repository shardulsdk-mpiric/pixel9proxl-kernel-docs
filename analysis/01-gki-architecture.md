# Deep Analysis #1: GKI (Generic Kernel Image) Architecture

## Overview

This document provides a deep technical analysis of the Generic Kernel Image (GKI) architecture, mixed builds, base kernel extension mechanism, and how to build full kernel replacements. This analysis is essential for understanding how Android kernels are structured and how to customize them.

## Navigation

- [Analysis Index](README.md)
- [Main Documentation Index](../README.md)
- [Overview](../overview.md)

---

## Analysis Tree: GKI Architecture

### 1. What is GKI? (Foundation)
- [ ] Definition and purpose
- [ ] Why was GKI created?
- [ ] Benefits of GKI
- [ ] GKI vs traditional monolithic kernels

### 2. GKI Structure
- [ ] What is in the base GKI kernel?
- [ ] What files does GKI contain?
- [ ] What gets built as part of GKI?
- [ ] What is NOT in GKI (device-specific code)?

### 3. Mixed Build Mechanism
- [ ] What is KBUILD_MIXED_TREE?
- [ ] How does base_kernel work?
- [ ] How are base kernel files copied to mixed tree?
- [ ] How does kernel build use mixed tree?
- [ ] What happens during "make" when KBUILD_MIXED_TREE is set?

### 4. Base Kernel vs Full Kernel
- [ ] What is base_kernel attribute?
- [ ] What happens WITH base_kernel (extension mode)?
- [ ] What happens WITHOUT base_kernel (full build)?
- [ ] Code differences in BUILD.bazel
- [ ] Build process differences
- [ ] Output differences

### 5. KMI (Kernel Module Interface)
- [ ] What is KMI?
- [ ] Why does KMI exist?
- [ ] What is KMI symbol list?
- [ ] How does KMI enforcement work?
- [ ] What happens if KMI is violated?
- [ ] When is KMI not applicable?

### 6. Build Artifacts
- [ ] What artifacts does GKI produce?
- [ ] What artifacts does device kernel produce?
- [ ] How are they combined?
- [ ] Where do artifacts go?

### 7. Runtime Behavior
- [ ] How does boot.img use GKI?
- [ ] How does vendor_kernel_boot.img work?
- [ ] How are modules loaded?
- [ ] How does system combine GKI + device kernel?

---

## Current Understanding (To Be Expanded)

### Key Finding: KBUILD_MIXED_TREE

From code analysis in `build/kernel/kleaf/impl/kernel_build.bzl`:

```python
def _create_kbuild_mixed_tree(ctx):
    """Adds actions that creates the `KBUILD_MIXED_TREE`."""
    if base_kernel_utils.get_base_kernel(ctx):
        # Create a directory for KBUILD_MIXED_TREE
        kbuild_mixed_tree = ctx.actions.declare_directory("{}_kbuild_mixed_tree".format(ctx.label.name))
        
        # Copy base kernel files to mixed tree
        kbuild_mixed_tree_command = """
            export KBUILD_MIXED_TREE=$(realpath {kbuild_mixed_tree})
            rm -rf ${KBUILD_MIXED_TREE}
            mkdir -p ${KBUILD_MIXED_TREE}
            for base_kernel_file in {base_kernel_files}; do
              cp -a -t ${KBUILD_MIXED_TREE} $(readlink -m ${base_kernel_file})
            done
        """
```

**What this tells us:**
- When `base_kernel` is set, a `KBUILD_MIXED_TREE` directory is created
- Base kernel files are copied to this directory
- The kernel build then uses `--srcdir ${KBUILD_MIXED_TREE}` flag

**This is a "mixed build" because:**
- Base kernel source files come from GKI (via KBUILD_MIXED_TREE)
- Device-specific source files come from device kernel tree
- The build combines both sources

**What gets copied to KBUILD_MIXED_TREE:**
From `build/kernel/kleaf/constants.bzl`:
- `DEFAULT_GKI_OUTS = ["System.map", "modules.builtin", "modules.builtin.modinfo", "vmlinux", "vmlinux.symvers", "Image", "Image.lz4", "Image.gz"]`
- These are the built artifacts from the GKI kernel
- The device kernel build uses these as inputs (especially `vmlinux.symvers` and `System.map`)

**How KBUILD_MIXED_TREE is used:**
1. **Module building** (`scripts/Makefile.modpost`):
   - Uses `$(mixed-build-prefix)vmlinux.symvers` for module symbol resolution
   - Gets symbol information from base kernel, not device kernel

2. **depmod** (`scripts/depmod.sh`):
   - Uses `${KBUILD_MIXED_TREE}System.map` for module dependency resolution
   - Gets kernel symbol map from base kernel

3. **Kernel Image** (`arch/arm64/Makefile`):
   - Image is NOT built when KBUILD_MIXED_TREE is set
   - The Image comes from the base kernel

### Key Finding: base_kernel Attribute

From `kernel_build.bzl` documentation:

```
base_kernel: A label referring the base kernel build.

If set, the list of files specified in the `DefaultInfo` of the rule specified in
`base_kernel` is copied to a directory, and `KBUILD_MIXED_TREE` is set to the directory.
Setting `KBUILD_MIXED_TREE` effectively enables mixed build.
```

**For caimito:**
- `base_kernel = "//common:kernel_aarch64"` (line 82 in BUILD.bazel)
- This means: extend the common GKI kernel
- Build process will use mixed build mode
- `make_goals = ["modules", "dtbs"]` (lines 89-91) - **Only builds modules and DTBs, NOT the kernel Image!**

**Key Discovery:**
From `aosp/arch/arm64/Makefile` line 170:
```makefile
# Don't compile Image in mixed build with "all" target
ifndef KBUILD_MIXED_TREE
all:	$(notdir $(KBUILD_IMAGE))
endif
```

This means: **When KBUILD_MIXED_TREE is set (i.e., when using base_kernel), the kernel Image is NOT built by the device kernel build.** The Image comes from the base kernel (GKI).

---

## Next Steps

1. Find what files are in `//common:kernel_aarch64`
2. Understand what gets copied to KBUILD_MIXED_TREE
3. Understand how kernel build uses the mixed tree
4. Understand what device kernel builds (modules vs kernel image)
5. Understand KMI and its role

---

## Questions to Answer

1. ✅ **ANSWERED**: Does device kernel with base_kernel build a kernel Image?
   - **NO!** When `base_kernel` is set, the device kernel build does NOT build the kernel Image
   - It only builds modules and DTB files
   - Evidence: `arch/arm64/Makefile` line 170: `ifndef KBUILD_MIXED_TREE all: $(notdir $(KBUILD_IMAGE)) endif`
   
2. ✅ **ANSWERED**: Where does the actual kernel Image come from?
   - **From GKI base kernel!** The Image comes from `//common:kernel_aarch64`
   - Device kernel build only produces modules and DTBs
   - Evidence: caimito BUILD.bazel line 82 has `base_kernel = "//common:kernel_aarch64"`, and make_goals = ["modules", "dtbs"]

3. **IN PROGRESS**: How do GKI modules vs device modules differ?
   - Where are GKI modules stored?
   - Where are device modules stored?

4. **PENDING**: What is the relationship between:
   - boot.img (contains GKI kernel?)
   - vendor_kernel_boot.img (contains device kernel components?)
   
5. **PENDING**: How does the system load and execute?
   - Does it load GKI kernel first?
   - Then load device modules?
   - How are they combined at runtime?

---

## Documentation Progress

- [x] Created analysis tree
- [x] Gather code evidence - IN PROGRESS
- [x] Answer key questions - PARTIALLY COMPLETE
- [ ] Create visual diagrams
- [ ] Write comprehensive explanation
- [ ] Validate understanding

---

## CRITICAL FINDINGS: How Boot Images Work

### Finding: Boot Image Creation Process

From `build/kernel/kleaf/impl/image/boot_images.bzl`:

**Step 1: Copy kernel outputs to DIST_DIR (lines 128-133)**
```python
kernel_build_outs = depset(
    transitive = [
        ctx.attr.kernel_build[KernelBuildInfo].base_kernel_files,  # From GKI
        ctx.attr.kernel_build[KernelBuildInfo].outs,                # From device kernel
    ],
    order = "postorder",  # Device outputs OVERRIDE base kernel files
)

command += """
    mkdir -p ${DIST_DIR}
    cp {kernel_build_outs} ${DIST_DIR}
"""
```

**What this means:**
- Both base_kernel_files (from GKI) AND device kernel outs are copied to DIST_DIR
- Device kernel outputs are copied LAST (postorder), so they override base kernel files
- If device kernel has Image, it overrides GKI Image
- If device kernel doesn't have Image (only modules), GKI Image is used

**Step 2: Set KERNEL_BINARY (lines 191-201)**
```python
kernel_binary = "Image"
if ctx.attr.ramdisk_compression == "lz4":
    kernel_binary = "Image.lz4"
elif ctx.attr.ramdisk_compression == "gzip":
    kernel_binary = "Image.gz"

kernel_binary_cmd = "KERNEL_BINARY={kernel_binary}".format(kernel_binary = kernel_binary)
```

**Step 3: build_boot_images uses KERNEL_BINARY from DIST_DIR**
From `build/kernel/build_utils.sh` line 629:
```bash
if [ ! -f "${DIST_DIR}/$KERNEL_BINARY" ]; then
  echo "kernel binary(KERNEL_BINARY = $KERNEL_BINARY) not present in ${DIST_DIR}"
  exit 1
fi
MKBOOTIMG_ARGS+=("--kernel" "${DIST_DIR}/${KERNEL_BINARY}")
```

**IMPLICATION FOR FULL KERNEL REPLACEMENT:**
- If device kernel builds Image (without base_kernel), the Image is in device kernel outs
- This Image gets copied to DIST_DIR and OVERRIDES the base kernel Image
- Therefore: **To replace GKI kernel, remove base_kernel and build Image in device kernel**

---

## HOW TO BUILD FULL KERNEL REPLACEMENT (Standard Interface)

### Current State (Extension Mode)
```python
kernel_build(
    name = "kernel",
    base_kernel = "//common:kernel_aarch64",  # Extends GKI
    make_goals = ["modules", "dtbs"],         # Only builds modules/DTBs
    # Image comes from base_kernel
)
```

### Full Kernel Build (Replacement Mode) - Standard Approach

Based on code analysis, the **standard way** to build a full kernel replacement is:

```python
kernel_build(
    name = "kernel",
    # base_kernel = REMOVED/None              # NO base_kernel
    make_goals = ["Image", "Image.lz4", "Image.gz", "modules", "dtbs"],  # Build Image
    outs = [
        "Image", "Image.lz4", "Image.gz",     # Kernel images
        "vmlinux", "System.map",              # Full kernel outputs
        # ... DTBs ...
    ],
    trim_nonlisted_kmi = False,               # No KMI enforcement
    kmi_symbol_list_strict_mode = False,      # No KMI strict mode
    # kmi_symbol_list = REMOVED               # Not needed without base_kernel
)
```

**Key Changes:**
1. Remove `base_kernel` attribute
2. Add `Image`, `Image.lz4`, `Image.gz` to `outs`
3. Change `make_goals` to include `Image`, `Image.lz4`, `Image.gz`
4. Remove or disable KMI-related attributes (no base kernel to maintain compatibility with)
5. Add full kernel outputs: `vmlinux`, `System.map`, etc.

**For kernel_images:**
```python
kernel_images(
    name = "kernel_images",
    # base_kernel_images = REMOVED            # No base kernel
    build_boot = True,                        # Build boot.img with device kernel Image
    # ... rest of config ...
)
```

**What happens:**
- Device kernel builds full Image from source
- Image is in device kernel outs
- When boot.img is built, device kernel Image overrides any base kernel Image
- boot.img uses the device kernel Image

---

## EVIDENCE FROM CODEBASE

### 1. Comments in caimito BUILD.bazel Suggest Full Kernel Build

From `private/devices/google/caimito/BUILD.bazel` lines 92-97:
```python
# Note: base_kernel removed to allow building full kernel Image from device kernel
# This is necessary when device-specific changes are made to kernel source files
# (e.g., modifications to init/main.c). Without base_kernel, the device kernel
# builds the complete Image including all device-specific code.
# When building full kernel (not extending base kernel), KMI enforcement is disabled
# since we're not maintaining KMI compatibility with a base kernel.
```

**BUT:** The actual file currently HAS `base_kernel = "//common:kernel_aarch64"` on line 82!

**This suggests:**
- The file was originally configured for full kernel build (comments describe that)
- It may have been changed back to extension mode
- OR the comments are aspirational/notes for future changes

### 2. Comparison: zumapro (SoC level) vs caimito

**zumapro BUILD.bazel (SoC - uses base_kernel):**
```python
kernel_build(
    name = "kernel",
    base_kernel = "//common:kernel_aarch64",  # ✅ HAS base_kernel
    make_goals = ["modules", "dtbs"],         # Only modules/DTBs
    outs = [".config", ... DTBs ...],        # NO Image in outs
)
```

**caimito BUILD.bazel (Device - should have base_kernel):**
```python
kernel_build(
    name = "kernel",
    base_kernel = "//common:kernel_aarch64",  # ✅ HAS base_kernel (current)
    make_goals = ["modules", "dtbs"],         # Only modules/DTBs
    outs = [".config", ... DTBs ...],        # NO Image in outs (current)
)
```

**But comments suggest full kernel:**
- Lines 92-97 describe removing base_kernel
- Lines 82-91 show Image in outs (but they're commented as "required for full kernel build")

**DISCREPANCY TO INVESTIGATE:**
- Why does caimito BUILD.bazel have base_kernel set BUT have comments about removing it?
- Why does it list Image in outs if it's using base_kernel?
- Is the file in transition? Or is there a configuration option?

---

## Understanding: Boot Image Components

### boot.img (with base_kernel)
- Contains: GKI kernel Image (from base_kernel)
- Contains: Initramfs (with device modules)
- Contains: DTB files (from device kernel)
- Purpose: Boot device with GKI kernel + device modules

### boot.img (without base_kernel - full kernel)
- Contains: Device kernel Image (from device kernel build)
- Contains: Initramfs (with device modules)
- Contains: DTB files (from device kernel)
- Purpose: Boot device with fully custom kernel

### vendor_kernel_boot.img
- Contains: Vendor-specific kernel components
- Used when `build_vendor_kernel_boot = True`
- Alternative to `vendor_boot.img` when building custom kernel

---

## RESOLVED: Current Configuration vs Full Kernel Requirements

### Current caimito Configuration (Extension Mode)

After reading the actual BUILD.bazel file:

```python
kernel_build(
    name = "kernel",
    base_kernel = "//common:kernel_aarch64",  # ✅ Extends GKI
    make_goals = ["modules", "dtbs"],         # ✅ Only builds modules/DTBs
    outs = [".config", ... DTBs ...],        # ✅ NO Image in outs
    kmi_symbol_list = "//common:android/abi_gki_aarch64_pixel",  # ✅ KMI enforced
)

kernel_images(
    name = "kernel_images",
    base_kernel_images = "//common:kernel_aarch64_images",  # ✅ Uses GKI images
    build_vendor_kernel_boot = True,                        # ✅ Builds vendor_kernel_boot.img
    # Note: build_boot = False (default), so NO boot.img is built
    #       Only vendor_kernel_boot.img is built
)
```

**Current behavior:**
- Device kernel extends GKI (builds modules + DTBs only)
- GKI kernel Image is used for boot.img (if built separately)
- Vendor-specific components go into vendor_kernel_boot.img
- System_dlkm uses GKI's system_dlkm staging archive

---

## HOW TO BUILD FULL KERNEL REPLACEMENT (Standard Interface)

### Step 1: Modify kernel_build

**Remove base_kernel:**
```python
kernel_build(
    name = "kernel",
    # base_kernel = REMOVED  # ❌ Remove this line
    base_kernel = None,     # OR explicitly set to None
)
```

**Add Image to outs:**
```python
outs = [
    ".config",
    "Image",              # ✅ Add uncompressed kernel
    "Image.lz4",          # ✅ Add lz4 compressed kernel
    "Image.gz",           # ✅ Add gzip compressed kernel
    "vmlinux",            # ✅ Full kernel binary (for debugging)
    "System.map",         # ✅ Symbol map
    # ... DTBs ...
],
```

**Change make_goals to build Image:**
```python
make_goals = [
    "Image",              # ✅ Build uncompressed kernel
    "Image.lz4",          # ✅ Build lz4 compressed kernel
    "Image.gz",           # ✅ Build gzip compressed kernel
    "modules",            # ✅ Build modules
    "dtbs",               # ✅ Build device trees
],
```

**Disable KMI enforcement:**
```python
# kmi_symbol_list = REMOVED  # ❌ Not needed without base_kernel
trim_nonlisted_kmi = False,   # ✅ No KMI trimming
kmi_symbol_list_strict_mode = False,  # ✅ No strict mode
```

**Complete example:**
```python
kernel_build(
    name = "kernel",
    srcs = [":kernel_sources"],
    outs = [
        ".config",
        "Image",
        "Image.lz4",
        "Image.gz",
        "vmlinux",
        "vmlinux.symvers",
        "System.map",
        "modules.builtin",
        "modules.builtin.modinfo",
    ] + [
        # ... DTBs ...
    ],
    # base_kernel = REMOVED - Build full kernel from source
    build_config = ":build_config",
    collect_unstripped_modules = True,
    defconfig_fragments = [":defconfig_fragments"],
    dtstree = ":dtstree",
    kconfig_ext = ":kconfig_ext",
    # KMI attributes removed - no base kernel to maintain compatibility with
    make_goals = [
        "Image",
        "Image.lz4",
        "Image.gz",
        "modules",
        "dtbs",
    ],
    module_outs = ZUMAPRO_MODULE_OUTS,
    strip_modules = True,
)
```

### Step 2: Modify kernel_images

**Remove base_kernel_images:**
```python
kernel_images(
    name = "kernel_images",
    # base_kernel_images = REMOVED  # ❌ Remove this line
    base_kernel_images = None,     # OR explicitly set to None
)
```

**Enable boot.img building (if needed):**
```python
kernel_images(
    name = "kernel_images",
    build_boot = True,  # ✅ Build boot.img with device kernel Image
    # ... rest of config ...
)
```

**Note on system_dlkm:**
- If `base_kernel` is None in `kernel_build`, then `base_kernel_images` can be None
- system_dlkm will be built from device kernel modules only
- No GKI modules will be included (they're only in base kernel's system_dlkm)

**Complete example:**
```python
kernel_images(
    name = "kernel_images",
    # base_kernel_images = REMOVED - No base kernel
    boot_image_outs = [
        "boot.img",           # ✅ Build boot.img with device kernel
        "dtb.img",
        "vendor_kernel_boot.img",
    ],
    build_boot = True,        # ✅ Enable boot.img building
    build_dtbo = True,
    build_initramfs = True,
    build_system_dlkm = True, # System_dlkm from device kernel only
    build_vendor_dlkm = True,
    build_vendor_kernel_boot = True,
    kernel_build = ":kernel",
    kernel_modules_install = ":kernel_modules_install",
    ramdisk_compression = "lz4",  # Matches Image.lz4
    # ... rest of config ...
)
```

### Step 3: Verify Build Process

**What happens during build:**

1. **kernel_build:**
   - No `base_kernel`, so no `KBUILD_MIXED_TREE` is created
   - Device kernel builds full Image from source (including all device-specific code)
   - Image is in device kernel outs

2. **boot_images (in kernel_images):**
   - Copies kernel_build_outs to DIST_DIR
   - Sets `KERNEL_BINARY=Image.lz4` (matching ramdisk_compression)
   - Calls `build_boot_images` which uses `${DIST_DIR}/Image.lz4`
   - This is the device kernel Image, not GKI Image

3. **Result:**
   - boot.img contains device kernel Image
   - Device kernel Image is used at boot time
   - All device-specific kernel code changes are included

---

## EVIDENCE FROM CODEBASE

### 1. boot_images.bzl Line 53-61
Device kernel outputs override base kernel files when building boot.img:
```python
kernel_build_outs = depset(
    transitive = [
        ctx.attr.kernel_build[KernelBuildInfo].base_kernel_files,  # GKI files first
        ctx.attr.kernel_build[KernelBuildInfo].outs,                # Device files last (override)
    ],
    order = "postorder",  # Later items override earlier
)
```

### 2. boot_images.bzl Line 191-201
KERNEL_BINARY is set based on ramdisk compression:
```python
kernel_binary = "Image"  # Default
if ctx.attr.ramdisk_compression == "lz4":
    kernel_binary = "Image.lz4"
elif ctx.attr.ramdisk_compression == "gzip":
    kernel_binary = "Image.gz"
```

### 3. system_dlkm_image.bzl Line 61-110
When `base_kernel` is None, system_dlkm is built differently:
```python
if kernel_build_infos.images_info.base_kernel_label != None:
    # Device-specific system_dlkm: Use GKI's system_dlkm staging archive
    # ...
elif ctx.attr.base_kernel_images == None:
    # Full kernel: Build system_dlkm from device kernel modules only
    extra_flags_cmd = """
        SYSTEM_DLKM_RE_SIGN=0
    """
```

### 4. arm64/Makefile (from earlier analysis)
When `KBUILD_MIXED_TREE` is NOT set (no base_kernel), Image is built:
```makefile
# Don't compile Image in mixed build with "all" target
ifndef KBUILD_MIXED_TREE
all: $(notdir $(KBUILD_IMAGE))  # ✅ Build Image when no base_kernel
endif
```

---

## SUMMARY: Standard Interface for Full Kernel Replacement

**The standard interface provided by the build system is:**

1. **Remove `base_kernel` from `kernel_build`** → Device kernel builds full Image
2. **Add `Image`, `Image.lz4`, `Image.gz` to `outs`** → Declare kernel images as outputs
3. **Add `Image`, `Image.lz4`, `Image.gz` to `make_goals`** → Build kernel images during make
4. **Remove `base_kernel_images` from `kernel_images`** → Don't use GKI images
5. **Set `build_boot = True`** (if boot.img is needed) → Build boot.img with device kernel
6. **Disable KMI attributes** → No base kernel to maintain compatibility with

**The build system automatically:**
- Uses device kernel Image when building boot.img (because device outputs override base outputs)
- Builds system_dlkm from device kernel modules (when base_kernel_images is None)
- Handles all image creation correctly

**This is the standard way developers are expected to build full kernel replacements.**

