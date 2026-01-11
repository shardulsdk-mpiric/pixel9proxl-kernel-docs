# Deep Analysis #2: Build Configuration System

## Overview

This document provides a deep technical analysis of the build configuration system used in the Pixel 9 Pro XL kernel build process. This includes understanding how `build.config` files merge, defconfig fragments work, Kconfig extensions, and the hierarchy of configuration inheritance.

## Navigation

- [Analysis Index](README.md)
- [Main Documentation Index](../README.md)
- [Overview](../overview.md)

---

## Table of Contents

1. [Build Config File Hierarchy](#1-build-config-file-hierarchy)
2. [Defconfig Fragment System](#2-defconfig-fragment-system)
3. [Kconfig Extension System](#3-kconfig-extension-system)
4. [Configuration Variable Inheritance](#4-configuration-variable-inheritance)
5. [Bazel Integration Flow](#5-bazel-integration-flow)
6. [Summary and Key Takeaways](#6-summary-and-key-takeaways)

---

## 1. Build Config File Hierarchy

### 1.1 What are build.config Files?

`build.config` files are shell scripts that set environment variables to control the kernel build process. They define things like:
- Compiler paths and flags
- Build directories
- Kernel configuration options
- Module lists
- Device-specific settings

These files are **sourced** (not executed) by the build system, meaning they run in the current shell context and can modify environment variables.

### 1.2 Build Config Inheritance Chain for caimito

For the Pixel 9 Pro XL (caimito) device, the build config hierarchy is:

```
build.config.caimito (device-specific)
    ↓
build.config.zumapro (SoC-specific)
    ↓
build.config.common (common kernel settings)
    ↓
build.config.aarch64 (architecture-specific)
    ↓
build.config.constants (base constants)
```

**In Bazel**, this is represented as:

```21:28:private/devices/google/caimito/BUILD.bazel
kernel_build_config(
    name = "build_config",
    srcs = [
        # do not sort
        "//private/devices/google/zumapro:build_config",
        "build.config.caimito",
    ],
)
```

The `srcs` list order matters - files are processed from **first to last**, with later files able to override earlier ones.

### 1.3 How kernel_build_config Rule Works

The `kernel_build_config` rule **concatenates** multiple build.config files into a single file:

```23:40:build/kernel/kleaf/impl/kernel_build_config.bzl
def _kernel_build_config_impl(ctx):
    out_file = ctx.actions.declare_file(ctx.attr.name + ".generated")
    hermetic_tools = hermetic_toolchain.get(ctx)
    command = hermetic_tools.setup + """
        cat {srcs} > {out_file}
    """.format(
        srcs = " ".join([src.path for src in ctx.files.srcs]),
        out_file = out_file.path,
    )
    debug.print_scripts(ctx, command)
    ctx.actions.run_shell(
        mnemonic = "KernelBuildConfig",
        inputs = ctx.files.srcs,
        tools = hermetic_tools.deps,
        outputs = [out_file],
        command = command,
        progress_message = "Generating build config {}".format(ctx.label),
    )
```

**Key points:**
- Uses `cat` to concatenate all `srcs` files in order
- Creates a single `.generated` file containing all config fragments
- Order matters: later files can override variables from earlier files
- The `# do not sort` comment prevents buildifier from alphabetizing the list

### 1.4 How Build Configs are Processed

When `kernel_env` rule runs, it:

1. **Exports BUILD_CONFIG**: Sets the `BUILD_CONFIG` environment variable to point to the generated build.config file
2. **Sources _setup_env.sh**: This script sources the build.config file(s)
3. **Variable inheritance**: Variables are set with `set -a` (auto-export), then the build.config is sourced (`. ${ROOT_DIR}/${BUILD_CONFIG}`), then `set +a`

From `build/_setup_env.sh`:

```55:60:build/_setup_env.sh
set -a
. ${ROOT_DIR}/${BUILD_CONFIG}
for fragment in ${BUILD_CONFIG_FRAGMENTS}; do
  . ${ROOT_DIR}/${fragment}
done
set +a
```

This means:
- All variables in build.config files are automatically exported
- Multiple fragments can be sourced (via `BUILD_CONFIG_FRAGMENTS`)
- Later variables override earlier ones (standard shell behavior)

### 1.5 Example: caimito Build Config Files

**build.config.zumapro** (SoC level):
```bash
. ${ROOT_DIR}/${KERNEL_DIR}/build.config.common
. ${ROOT_DIR}/${KERNEL_DIR}/build.config.aarch64

KCONFIG_EXT_MODULES_PREFIX=$(realpath ${ROOT_DIR} --relative-to ${ROOT_DIR}/${KERNEL_DIR})/
KCONFIG_SOC_GS_PREFIX=${KCONFIG_EXT_MODULES_PREFIX}/private/google-modules/soc/gs/

DEFCONFIG=gki_defconfig
BOOT_IMAGE_HEADER_VERSION=4
TRIM_UNUSED_MODULES=1
```

**build.config.caimito** (device level):
```bash
BCMDHD=4390
```

When combined, the final build.config contains all variables, with `BCMDHD=4390` from caimito added to all the zumapro/common/aarch64 settings.

---

## 2. Defconfig Fragment System

### 2.1 What are Defconfig Fragments?

Defconfig fragments are files that specify kernel configuration options (like `CONFIG_XXX=y` or `CONFIG_YYY=m`). Unlike full defconfig files, fragments contain only **additional or overridden** options.

### 2.2 How Defconfig Fragments are Merged

The kernel build system uses `scripts/kconfig/merge_config.sh` to merge defconfig fragments:

```19:40:build/kernel/kleaf/impl/config_utils.bzl
def _create_merge_dot_config_cmd(defconfig_fragments_paths_expr):
    """Returns a command that merges defconfig fragments into `$OUT_DIR/.config`

    Args:
        defconfig_fragments_paths_expr: A shell expression that evaluates
            to a list of paths to the defconfig fragments.

    Returns:
        the command that merges defconfig fragments into `$OUT_DIR/.config`
    """
    cmd = """
        ${{KERNEL_DIR}}/scripts/kconfig/merge_config.sh \\
            -m -r \\
            ${{OUT_DIR}}/.config \\
            {defconfig_fragments_paths_expr} > /dev/null
        mv ${{OUT_DIR}}/.config.tmp ${{OUT_DIR}}/.config
    """.format(
        defconfig_fragments_paths_expr = defconfig_fragments_paths_expr,
    )
    return cmd
```

**Merge process:**
1. Start with base `.config` (from `make defconfig` using `DEFCONFIG` variable)
2. Apply defconfig fragments in order using `merge_config.sh`
3. Run `make olddefconfig` to resolve dependencies and set defaults
4. Verify all fragment options are set correctly

### 2.3 Defconfig Fragment Order

From `kernel_config.bzl`, the merge happens **after** the base defconfig is created:

```547:556:build/kernel/kleaf/impl/kernel_config.bzl
    command = ctx.attr.env[KernelEnvInfo].setup + """
          {cache_dir_cmd}
        # Pre-defconfig commands
          eval ${{PRE_DEFCONFIG_CMDS}}
        # Actual defconfig
          make -C ${{KERNEL_DIR}} ${{TOOL_ARGS}} O=${{OUT_DIR}} ${{DEFCONFIG}}
        # Post-defconfig commands
          eval ${{POST_DEFCONFIG_CMDS}}
        # Re-config
          {reconfig_cmd}
```

The `_reconfig` function merges fragments:

```448:462:build/kernel/kleaf/impl/kernel_config.bzl
    if ctx.files.defconfig_fragments:
        transitive_deps += [target.files for target in ctx.attr.defconfig_fragments]
        defconfig_fragments_paths = [f.path for f in ctx.files.defconfig_fragments]

        apply_defconfig_fragments_cmd = config_utils.create_merge_dot_config_cmd(
            " ".join(defconfig_fragments_paths),
        )
        apply_defconfig_fragments_cmd += """
            need_olddefconfig=1
        """

        check_defconfig_fragments_cmd = config_utils.create_check_defconfig_cmd(
            ctx.label,
            " ".join(defconfig_fragments_paths),
        )
```

**Order matters**: Later fragments can override options from earlier fragments.

### 2.4 Example: caimito Defconfig

For caimito, defconfig fragments are defined as a `filegroup`:

```49:52:private/devices/google/zumapro/BUILD.bazel
filegroup(
    name = "defconfig_fragments",
    srcs = ["zumapro_defconfig"],
)
```

The device kernel (`kernel_build`) references these fragments:

```79:79:private/devices/google/zumapro/BUILD.bazel
    defconfig_fragments = [":defconfig_fragments"],
```

---

## 3. Kconfig Extension System

### 3.1 What is Kconfig.ext?

`Kconfig.ext` files allow adding custom kernel configuration options that aren't in the main kernel tree. They're used for device-specific or vendor-specific configuration options.

### 3.2 How kconfig_ext Works

The `kconfig_ext` attribute in `kernel_build` points to a file that gets merged into the kernel's Kconfig system. This allows external modules or device-specific code to add configuration options to `make menuconfig` and other configuration interfaces.

**In zumapro BUILD.bazel:**

```43:47:private/devices/google/zumapro/BUILD.bazel
create_file(
    name = "kconfig_ext",
    srcs = ["Kconfig.ext.zumapro"],
    out = "Kconfig.ext",
)
```

The `create_file` rule concatenates source files (similar to `kernel_build_config`):

```9:36:private/devices/google/common/kleaf/create_file.bzl
def _create_file_impl(ctx):
    hermetic_tools = hermetic_toolchain.get(ctx)
    srcs = list(ctx.files.srcs)
    out = ctx.outputs.out

    if ctx.attr.content:
        content_file = ctx.actions.declare_file(ctx.attr.name + "/content")
        ctx.actions.write(
            output = content_file,
            content = "\n".join(ctx.attr.content) + "\n",
        )
        srcs.append(content_file)

    add_newline = """
        echo >> "{out}"
    """.format(out = out.path)

    ctx.actions.run_shell(
        inputs = srcs,
        tools = hermetic_tools.deps,
        outputs = [out],
        command = hermetic_tools.setup + add_newline.join([
            """
            cat "{src}" >> "{out}"
            """.format(src = src.path, out = out.path)
            for src in srcs
        ]),
    )
```

### 3.3 How Kconfig.ext is Used

The `KCONFIG_EXT` environment variable points to the merged Kconfig.ext file, which is then included by the kernel's Kconfig system. This happens during the configuration phase when `make defconfig` or `make menuconfig` runs.

From `kernel_env.bzl`:

```178:183:build/kernel/kleaf/impl/kernel_env.bzl
    if kconfig_ext:
        command += """
              export KCONFIG_EXT={kconfig_ext}
            """.format(
            kconfig_ext = kconfig_ext.path,
        )
```

And later:

```305:307:build/kernel/kleaf/impl/kernel_env.bzl
           if [ -n "${{KCONFIG_EXT}}" ]; then
             export KCONFIG_EXT_PREFIX=$(realpath $(dirname ${{KCONFIG_EXT}}) --relative-to ${{ROOT_DIR}}/${{KERNEL_DIR}})/
           fi
```

---

## 4. Configuration Variable Inheritance

### 4.1 Variable Override Order

Variables are set in this order (later overrides earlier):

1. **Base configs** (build.config.common, build.config.aarch64, etc.)
2. **SoC configs** (build.config.zumapro)
3. **Device configs** (build.config.caimito)
4. **BUILD_CONFIG_FRAGMENTS** (if specified)
5. **Environment variables** (if already set)

### 4.2 Key Configuration Variables

Common variables in build.config files:

- `DEFCONFIG`: Base defconfig to use (e.g., `gki_defconfig`)
- `KERNEL_DIR`: Directory containing kernel sources
- `MAKE_GOALS`: What to build (e.g., `"Image modules"`)
- `KCONFIG_EXT_MODULES_PREFIX`: Path prefix for external modules
- `TRIM_UNUSED_MODULES`: Whether to trim unused modules
- `BOOT_IMAGE_HEADER_VERSION`: Version for boot image format
- `ARCH`: Target architecture (e.g., `arm64`)

### 4.3 Variable Export Mechanism

Variables are exported using `set -a` (auto-export) before sourcing, which means all variables become environment variables available to child processes:

```55:60:build/_setup_env.sh
set -a
. ${ROOT_DIR}/${BUILD_CONFIG}
for fragment in ${BUILD_CONFIG_FRAGMENTS}; do
  . ${ROOT_DIR}/${fragment}
done
set +a
```

---

## 5. Bazel Integration Flow

### 5.1 Complete Configuration Flow

Here's how configuration flows through the Bazel build system:

1. **kernel_build_config** rule:
   - Concatenates build.config files from `srcs`
   - Creates a single `.generated` file

2. **kernel_env** rule:
   - Takes the `build_config` file (from `kernel_build_config`)
   - Generates an environment script that:
     - Exports `BUILD_CONFIG` variable
     - Sources `_setup_env.sh` which sources the build.config
     - Sets up toolchains, paths, etc.

3. **kernel_config** rule:
   - Uses the environment from `kernel_env`
   - Runs `make defconfig` using `DEFCONFIG` variable
   - Merges defconfig fragments using `merge_config.sh`
   - Runs `make olddefconfig` to resolve dependencies
   - Produces final `.config` file

4. **kernel_build** rule:
   - Uses the `.config` from `kernel_config`
   - Builds kernel, modules, device trees, etc.

### 5.2 Configuration Dependency Graph

```
kernel_build_config
    ↓
kernel_env (uses build_config)
    ↓
kernel_config (uses env, defconfig_fragments, kconfig_ext)
    ↓
kernel_build (uses .config from kernel_config)
```

---

## 6. Summary and Key Takeaways

### Key Concepts

1. **Build Config Files**:
   - Shell scripts that set environment variables
   - Concatenated by `kernel_build_config` rule in order
   - Later files can override earlier variables

2. **Defconfig Fragments**:
   - Kernel configuration options (`CONFIG_XXX=y`)
   - Merged using `scripts/kconfig/merge_config.sh`
   - Applied after base defconfig, before `olddefconfig`

3. **Kconfig Extensions**:
   - Custom configuration options for external modules
   - Merged using `create_file` rule
   - Included via `KCONFIG_EXT` environment variable

4. **Configuration Hierarchy**:
   - Base → Architecture → SoC → Device
   - Each level can override settings from previous levels
   - Order matters in all merging operations

### Practical Implications

- To add a device-specific build setting: Add it to `build.config.caimito`
- To add a kernel config option: Add it to the defconfig fragment file
- To add a Kconfig option: Add it to `Kconfig.ext.zumapro` (or device-specific variant)
- Changes at device level override SoC level, which override base level

### Next Steps

- Understand how these configurations affect the actual kernel build
- See how modules are configured and built
- Understand how device trees are configured

