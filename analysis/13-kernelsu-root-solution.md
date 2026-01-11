# Deep Analysis #13: KernelSU - Root Solution for GKI Devices

## Overview

This document provides a deep technical analysis of KernelSU as a root solution for the Pixel 9 Pro XL kernel build. KernelSU is a kernel-based root solution specifically designed for Android GKI devices, providing root access through kernel-space hooks rather than userspace modifications.

## Navigation

- [Analysis Index](README.md)
- [Main Documentation Index](../README.md)
- [Overview](../overview.md)

---

## Table of Contents

1. [What is KernelSU?](#1-what-is-kernelsu)
2. [KernelSU vs Magisk](#2-kernelsu-vs-magisk)
3. [How KernelSU Works](#3-how-kernelsu-works)
4. [KernelSU Integration](#4-kernelsu-integration)
5. [Building Kernel with KernelSU](#5-building-kernel-with-kernelsu)
6. [Installation and Usage](#6-installation-and-usage)
7. [Summary and Key Takeaways](#7-summary-and-key-takeaways)

---

## 1. What is KernelSU?

### 1.1 KernelSU Definition

**KernelSU** is a root solution for Android GKI (Generic Kernel Image) devices that works in **kernel mode**. Unlike traditional root solutions that work in userspace, KernelSU grants root permission to userspace apps directly in kernel space.

**Key Characteristics:**
- **Kernel-based**: Works entirely in kernel mode
- **GKI-compatible**: Designed specifically for GKI devices
- **Kernel hooks**: Uses kernel hooks to intercept system calls
- **Metamodule system**: Provides pluggable architecture for modules

### 1.2 KernelSU Features

According to the [KernelSU documentation](https://kernelsu.org/guide/what-is-kernelsu.html), KernelSU provides:

1. **Kernel-space root access**: Root permissions granted directly in kernel
2. **Hardware breakpoints**: Can add hardware breakpoints to any process
3. **Memory access**: Access physical memory of any process invisibly
4. **System call interception**: Intercept any system call (syscall) within kernel space
5. **Metamodule system**: Pluggable architecture for module management
6. **Systemless modifications**: Can provide systemless modifications via metamodules

### 1.3 Why KernelSU for GKI?

KernelSU is particularly well-suited for GKI devices because:
- **GKI architecture**: Works with the GKI kernel architecture
- **No userspace modifications**: Doesn't require modifying userspace
- **Kernel-level control**: Provides kernel-level control and capabilities
- **Compatibility**: Maintains compatibility with GKI updates

---

## 2. KernelSU vs Magisk

### 2.1 Key Differences

**KernelSU** (kernel-based):
- Works in **kernel mode**
- Requires **kernel source modification**
- Provides **kernel-level hooks**
- **GKI-specific** design
- **Metamodule system** for extensions

**Magisk** (userspace-based):
- Works in **userspace**
- **No kernel modification** required (works with stock kernels)
- Uses **boot image patching**
- **Universal** (works with many devices)
- **Module system** for extensions

### 2.2 When to Use KernelSU

Use KernelSU when:
- **Building custom kernel**: You're already building kernel from source
- **GKI device**: Device uses GKI architecture
- **Kernel-level features needed**: Need kernel-level capabilities
- **Better hiding**: Kernel-level root hiding may be better
- **Performance**: Kernel-level may have better performance

### 2.3 When to Use Magisk

Use Magisk when:
- **Stock kernel**: Want to root without kernel modifications
- **Quick setup**: Need quick root without building kernel
- **Universal compatibility**: Device may not be GKI
- **Established ecosystem**: Want access to Magisk modules

---

## 3. How KernelSU Works

### 3.1 Kernel Hooks

KernelSU works by installing **kernel hooks** that intercept system calls and kernel operations. These hooks run in kernel space, providing root access directly from the kernel.

### 3.2 System Call Interception

KernelSU intercepts system calls (syscalls) to:
- **Grant root permissions**: Intercept permission checks (e.g., `capable()` calls)
- **Modify process behavior**: Change how processes execute
- **Access control**: Control access to resources
- **Security bypass**: Bypass security restrictions (SELinux, capabilities)

The core hooking mechanism in `core_hook.c` intercepts kernel functions that check permissions, allowing KernelSU to grant root access to specific processes based on its internal policy.

### 3.3 KernelSU Components

From the codebase structure in `/mnt/work_4gb/sources/android/pixel_9_pro_xl/kernel/aosp/drivers/kernelsu/`:

```
aosp/drivers/kernelsu/
├── core_hook.c          # Core hooking mechanism
├── ksu.h                # KernelSU header definitions
├── Kconfig              # Kernel configuration
├── Makefile             # Build configuration
└── ...                  # Other components
```

The core components include:
- **core_hook.c**: Implements kernel hooks for system call interception
- **ksu.h**: Defines KernelSU data structures and interfaces
- **Kconfig**: Provides kernel configuration options for KernelSU
- **Makefile**: Integrates KernelSU into kernel build system

### 3.4 Kernel Configuration

KernelSU requires kernel configuration options. From `aosp/drivers/kernelsu/Kconfig`:

The kernel must be configured with KernelSU support enabled. This typically involves:
- Enabling `CONFIG_KERNELSU` option
- Configuring KernelSU features (if any additional options)
- Building kernel with KernelSU support

The Kconfig file defines the configuration option that must be enabled in the kernel's `.config` file.

---

## 4. KernelSU Integration

### 4.1 Integration Process

KernelSU is integrated into the kernel source tree. Based on the setup script (`setup.sh`), the integration process:

1. **Clone KernelSU**: KernelSU code is added to kernel source
2. **Patch kernel**: Kernel patches are applied
3. **Configure build**: Kernel configuration updated
4. **Build kernel**: Build kernel with KernelSU support

### 4.2 Code Location

In the Pixel 9 Pro XL kernel source:
- **KernelSU code**: `aosp/KernelSU/`
- **Kernel driver**: `aosp/drivers/kernelsu/`
- **Setup script**: `aosp/KernelSU/setup.sh`

### 4.3 Build Integration

KernelSU is integrated into the kernel build system:
- **Kconfig**: Kernel configuration options (in `aosp/drivers/kernelsu/Kconfig`)
- **Makefile**: Build system integration (in `aosp/drivers/kernelsu/Makefile`)
- **Source files**: Kernel driver source code (in `aosp/drivers/kernelsu/`)

### 4.4 KernelSU Setup Script

The KernelSU setup script (`aosp/KernelSU/setup.sh`) handles:
- **Integration**: Integrating KernelSU into kernel source
- **Patching**: Applying necessary kernel patches
- **Configuration**: Setting up build configuration
- **Verification**: Verifying integration is correct

---

## 5. Building Kernel with KernelSU

### 5.1 Prerequisites

To build kernel with KernelSU:
1. **Kernel source**: Kernel source code with KernelSU integrated
2. **Kernel configuration**: Enable KernelSU in kernel config
3. **Build system**: Standard kernel build system (Bazel/Kleaf)

### 5.2 Configuration Steps

1. **Enable KernelSU**: Set `CONFIG_KERNELSU=y` in kernel config
2. **Configure features**: Configure KernelSU features as needed
3. **Build kernel**: Build kernel normally with KernelSU support
4. **Flash kernel**: Flash kernel to device

### 5.3 Build Process

The build process is the same as building a normal kernel:
1. **Configure**: Enable KernelSU in kernel config
2. **Build**: Build kernel using standard build process
3. **Flash**: Flash kernel to device
4. **Install app**: Install KernelSU manager app

---

## 6. Installation and Usage

### 6.1 Installation Process

According to [KernelSU installation guide](https://kernelsu.org/guide/installation.html):

1. **Build kernel**: Build kernel with KernelSU support
2. **Flash kernel**: Flash kernel to device
3. **Install app**: Install KernelSU manager app (APK)
4. **Grant permissions**: Grant necessary permissions
5. **Verify root**: Verify root access works

### 6.2 Usage

KernelSU provides:
- **Root access**: Grant root to apps
- **Module system**: Install KernelSU modules
- **App profile**: Per-app root control
- **Systemless modifications**: Via metamodules

### 6.3 Metamodules

KernelSU's metamodule system allows:
- **Systemless modifications**: Modify `/system` without actually modifying it
- **Module management**: Install/remove modules
- **Overlay filesystem**: Use overlayfs for modifications

---

## 7. Summary and Key Takeaways

### Key Concepts

1. **KernelSU Definition**:
   - Kernel-based root solution for GKI devices
   - Works in kernel mode, not userspace
   - Provides kernel-level hooks and capabilities

2. **KernelSU vs Magisk**:
   - KernelSU: Kernel-based, requires kernel source
   - Magisk: Userspace-based, works with stock kernels
   - Choose based on needs and capabilities

3. **How KernelSU Works**:
   - Uses kernel hooks to intercept system calls
   - Grants root permissions in kernel space
   - Provides kernel-level control and capabilities

4. **Integration**:
   - Integrated into kernel source tree
   - Requires kernel configuration changes
   - Built as part of kernel build process

5. **Building with KernelSU**:
   - Enable KernelSU in kernel config
   - Build kernel normally
   - Flash kernel to device
   - Install KernelSU manager app

### Practical Implications

- **For Pixel 9 Pro XL**: KernelSU is a good choice since we're building custom kernel
- **Integration**: KernelSU code is already in the kernel source
- **Configuration**: Need to enable KernelSU in kernel config
- **Build**: Build kernel with KernelSU support
- **Installation**: Flash kernel and install manager app

### Advantages of KernelSU

1. **Kernel-level control**: More powerful than userspace solutions
2. **Better hiding**: Kernel-level root hiding may be more effective
3. **GKI-optimized**: Designed specifically for GKI devices
4. **Performance**: Kernel-level may have better performance
5. **Metamodules**: Flexible module system

### Disadvantages of KernelSU

1. **Requires kernel build**: Must build kernel from source
2. **Kernel modification**: Requires kernel source changes
3. **Less universal**: GKI-specific, not universal like Magisk
4. **Setup complexity**: More complex setup than Magisk

### Next Steps

1. **Verify integration**: Check if KernelSU is properly integrated
2. **Enable in config**: Enable KernelSU in kernel configuration
3. **Build kernel**: Build kernel with KernelSU support
4. **Test**: Test KernelSU functionality
5. **Document**: Document Pixel 9 Pro XL specific integration

---

## References

- [KernelSU Official Website](https://kernelsu.org)
- [What is KernelSU?](https://kernelsu.org/guide/what-is-kernelsu.html)
- [KernelSU Installation Guide](https://kernelsu.org/guide/installation.html)
- [How to Build KernelSU](https://kernelsu.org/guide/how-to-build.html)
- [KernelSU vs Magisk](https://kernelsu.org/guide/difference-with-magisk.html)

