# Deep Analysis #8: KMI (Kernel Module Interface) System

## Overview

This document provides a deep technical analysis of the KMI (Kernel Module Interface) system for the Pixel 9 Pro XL kernel build. This includes understanding what KMI is, why it exists, KMI symbol lists, KMI enforcement during build, what happens if KMI is violated, and how to add new KMI symbols.

## Navigation

- [Analysis Index](README.md)
- [Main Documentation Index](../README.md)
- [Overview](../overview.md)

---

## Table of Contents

1. [What is KMI?](#1-what-is-kmi)
2. [Why Does KMI Exist?](#2-why-does-kmi-exist)
3. [KMI Symbol Lists](#3-kmi-symbol-lists)
4. [KMI Enforcement](#4-kmi-enforcement)
5. [KMI Violations](#5-kmi-violations)
6. [Adding KMI Symbols](#6-adding-kmi-symbols)
7. [Summary and Key Takeaways](#7-summary-and-key-takeaways)

---

## 1. What is KMI?

### 1.1 KMI Definition

KMI (Kernel Module Interface) is a stable interface between the GKI (Generic Kernel Image) kernel and device-specific kernel modules. It defines which kernel symbols (functions, variables, data structures) can be used by modules.

### 1.2 KMI Purpose

KMI ensures that:
- **Module compatibility**: Modules built against one kernel version work with other kernel versions that share the same KMI
- **Interface stability**: Kernel developers know which symbols are part of the stable interface
- **Binary compatibility**: Device-specific modules don't need to be rebuilt when GKI kernel is updated

### 1.3 KMI in GKI

In the GKI architecture:
- **GKI kernel**: Provides stable KMI interface
- **Device modules**: Use only KMI symbols
- **KMI enforcement**: Build system checks that modules only use KMI symbols

---

## 2. Why Does KMI Exist?

### 2.1 Problem KMI Solves

Without KMI:
- **Module breakage**: Kernel updates break device-specific modules
- **Rebuild required**: Modules must be rebuilt for every kernel version
- **Fragmentation**: Different devices need different kernel versions
- **Maintenance burden**: Constant module updates for kernel changes

### 2.2 KMI Benefits

With KMI:
- **Stable interface**: Modules work across kernel versions
- **Reduced rebuilds**: Modules don't need rebuilding for kernel updates
- **Better compatibility**: Single kernel version supports multiple devices
- **Easier maintenance**: Kernel and modules can be updated independently

### 2.3 GKI and KMI

GKI (Generic Kernel Image) architecture relies on KMI:
- **Common kernel**: Single GKI kernel binary for all devices
- **Device modules**: Device-specific functionality as modules
- **KMI interface**: Stable interface between kernel and modules
- **ABI compatibility**: KMI ensures ABI compatibility

---

## 3. KMI Symbol Lists

### 3.1 What are KMI Symbol Lists?

KMI symbol lists are files that define which kernel symbols are part of the stable interface. These symbols can be used by device-specific modules.

### 3.2 KMI Symbol List Location

From `private/devices/google/caimito/BUILD.bazel`:

```88:88:private/devices/google/caimito/BUILD.bazel
    kmi_symbol_list = "//common:android/abi_gki_aarch64_pixel",
```

The KMI symbol list is located at `//common:android/abi_gki_aarch64_pixel`, which points to a symbol list file defining the KMI for Pixel devices.

### 3.3 Symbol List Format

KMI symbol lists typically contain:
- **Function symbols**: Exported functions that modules can call
- **Variable symbols**: Exported variables that modules can access
- **Type symbols**: Data structures that modules can use
- **Symbol versions**: Version information for symbol compatibility

### 3.4 Symbol List Generation

KMI symbol lists are typically:
- **Generated**: From kernel source code analysis
- **Maintained**: Updated when new symbols are added to KMI
- **Validated**: Checked during build to ensure modules only use KMI symbols

---

## 4. KMI Enforcement

### 4.1 How is KMI Enforced?

KMI is enforced during the build process:

1. **Symbol extraction**: Extract symbols used by modules
2. **Symbol checking**: Compare module symbols against KMI symbol list
3. **Violation detection**: Identify symbols not in KMI list
4. **Build failure**: Fail build if non-KMI symbols are used

### 4.2 KMI Checking Process

The build system:
- **Analyzes modules**: Extracts symbols used by device modules
- **Checks against KMI**: Verifies all symbols are in KMI symbol list
- **Reports violations**: Lists any non-KMI symbols found
- **Enforces compliance**: Prevents build completion if violations exist

### 4.3 KMI Enforcement Tools

KMI enforcement uses:
- **ABI tools**: Extract and compare kernel ABIs
- **Symbol checkers**: Verify symbol usage in modules
- **Build integration**: Integrated into kernel build process

---

## 5. KMI Violations

### 5.1 What Happens if KMI is Violated?

If a module uses symbols not in the KMI symbol list:

1. **Build failure**: Build fails with error messages
2. **Violation report**: Lists non-KMI symbols used
3. **Debugging required**: Developer must fix symbol usage
4. **Options**: Either add symbols to KMI or change module implementation

### 5.2 Common KMI Violations

Common causes of KMI violations:
- **Using internal symbols**: Using non-exported kernel functions
- **Direct structure access**: Accessing kernel internal data structures
- **Undocumented symbols**: Using symbols not in KMI list
- **Version mismatch**: Using symbols from wrong kernel version

### 5.3 Fixing KMI Violations

To fix KMI violations:
1. **Identify symbols**: Determine which symbols are causing violations
2. **Check KMI list**: Verify symbols are not in KMI symbol list
3. **Add to KMI**: If appropriate, add symbols to KMI symbol list
4. **Refactor module**: If not appropriate, change module to use KMI symbols
5. **Rebuild**: Rebuild and verify KMI compliance

---

## 6. Adding KMI Symbols

### 6.1 When to Add Symbols to KMI

Symbols should be added to KMI when:
- **Module needs**: Device modules need to use the symbol
- **Stable interface**: Symbol is part of the stable kernel interface
- **Long-term support**: Symbol will be maintained across kernel versions
- **API design**: Symbol is intentionally exported for module use

### 6.2 How to Add Symbols to KMI

To add symbols to KMI:
1. **Update symbol list**: Add symbols to KMI symbol list file
2. **Document symbols**: Document symbol usage and purpose
3. **Test compatibility**: Verify modules work with new symbols
4. **Submit changes**: Submit KMI updates through review process

### 6.3 KMI Symbol Maintenance

KMI symbols require:
- **Long-term support**: Symbols must remain stable
- **Version compatibility**: Symbols must work across kernel versions
- **Documentation**: Symbols must be documented
- **Testing**: Symbols must be tested for compatibility

---

## 7. Summary and Key Takeaways

### Key Concepts

1. **KMI Definition**:
   - Stable interface between GKI kernel and device modules
   - Defines which kernel symbols modules can use
   - Ensures module compatibility across kernel versions

2. **KMI Purpose**:
   - Enables single kernel binary for multiple devices
   - Reduces need to rebuild modules for kernel updates
   - Maintains binary compatibility between kernel and modules

3. **KMI Symbol Lists**:
   - Files defining stable kernel interface
   - Contain function, variable, and type symbols
   - Located at `//common:android/abi_gki_aarch64_pixel`

4. **KMI Enforcement**:
   - Build-time checking of module symbol usage
   - Prevents use of non-KMI symbols
   - Fails build if violations detected

5. **KMI Violations**:
   - Occurs when modules use non-KMI symbols
   - Results in build failure
   - Must be fixed by adding to KMI or refactoring

6. **Adding KMI Symbols**:
   - Requires updating KMI symbol list
   - Symbols must be stable and documented
   - Long-term maintenance required

### Practical Implications

- **To use kernel symbols**: Ensure symbols are in KMI symbol list
- **To add symbols**: Update KMI symbol list and maintain compatibility
- **To debug violations**: Check build error messages for non-KMI symbols
- **To maintain modules**: Use only KMI symbols for compatibility
- **To update kernel**: KMI ensures modules continue working

### KMI Workflow

```
Module development
    ↓
Use KMI symbols only
    ↓
Build system checks symbols
    ↓
KMI compliance verified
    ↓
Module works with kernel
```

### Next Steps

- Understand KMI symbol list format in detail
- Learn how to read and interpret KMI symbol lists
- Understand KMI versioning and compatibility
- Learn KMI debugging techniques
- Understand KMI update process and review requirements

