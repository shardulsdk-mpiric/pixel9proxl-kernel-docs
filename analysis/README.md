# Deep Analysis Documentation

This directory contains deep-dive analyses of specific aspects of the Pixel 9 Pro XL kernel build system. These documents provide detailed technical understanding of how the system works.

## Available Analyses

1. [Analysis Plan](00-plan.md) - Planned areas for deep analysis and investigation
2. [GKI Architecture](01-gki-architecture.md) - Understanding Generic Kernel Image architecture, mixed builds, base kernel extension, and how to build full kernel replacements
3. [Build Configuration System](02-build-configuration.md) - Understanding build.config file hierarchy, defconfig fragment merging, Kconfig extensions, and configuration inheritance
4. [Build Command Flow](03-build-command-flow.md) - Understanding build_caimito.sh execution, Bazel target processing, build order, and artifact distribution
5. [Module System](04-module-system.md) - Understanding kernel module structure, build process, signing, loading sequence, and module types (vendor_dlkm, system_dlkm, vendor_ramdisk)
6. [Boot Process and Partition Structure](05-boot-process.md) - Understanding boot.img, vendor_kernel_boot.img, partition structure, and boot process
7. [Root Requirements and Kernel Modifications](06-root-requirements.md) - Understanding kernel changes needed for root, SELinux modifications, root methods (kernel vs modules), and security/stability impact
8. [Device Tree System](07-device-tree-system.md) - Understanding DTS/DTSI structure, DTBO overlays, device tree compilation, variant support, and runtime behavior

## Analysis Status

- ✅ [GKI Architecture](01-gki-architecture.md) - Completed
- ✅ [Build Configuration System](02-build-configuration.md) - Completed
- ✅ [Build Command Flow](03-build-command-flow.md) - Completed
- ✅ [Module System](04-module-system.md) - Completed
- ✅ [Boot Process and Partition Structure](05-boot-process.md) - Completed
- ✅ [Root Requirements and Kernel Modifications](06-root-requirements.md) - Completed
- ✅ [Device Tree System](07-device-tree-system.md) - Completed
- ⏳ Additional analyses coming...

## How to Read These Documents

These analyses are designed to be read:
1. **Independently**: Each analysis can be read on its own, though cross-references to other documents are provided
2. **In order**: For newcomers, reading analyses in numerical order (01, 02, etc.) provides a logical progression
3. **By topic**: Jump to the specific analysis that addresses your question

## Navigation

- Return to [Main Documentation Index](../README.md)
- See [Overview](../overview.md) for high-level understanding
- See [Guides](../guides/README.md) for step-by-step instructions
- See [Reference](../reference/README.md) for quick reference materials

