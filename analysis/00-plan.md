# Deep Analysis Plan: Pixel 9 Pro XL Kernel Build Process

## Objective
Build custom kernel from source for Pixel 9 Pro XL to:
- Root the device
- Run kernel-related tests
- Flash locally built kernel with custom changes
- Maintain maximum compatibility with the complete system for regular device usage

---

## Areas Requiring Deep Analysis

### Priority 1: CRITICAL (Must Understand First)

1. **GKI Architecture - Base Kernel vs Full Kernel**
   - What is GKI and how does it work?
   - How does base_kernel extension work?
   - What happens when you use base_kernel vs not?
   - What gets included in GKI vs device kernel?
   - How do they combine at runtime?
   - KMI (Kernel Module Interface) - what is it, why does it matter?

2. **Build Configuration System**
   - How do build.config files merge?
   - Defconfig fragment merging process
   - Kconfig extension system
   - How device configs extend SoC configs extend base configs
   - What build.config variables control the build?

3. **Build Command Flow - From Script to Artifacts**
   - Exact execution flow of build_caimito.sh
   - How Bazel processes the BUILD.bazel file
   - What targets are built in what order?
   - How prebuilt GKI download works
   - How to disable prebuilt GKI and build from source
   - What gets built when, and where do outputs go?

### Priority 2: HIGH (Essential for Development)

4. **Module System**
   - How are kernel modules structured?
   - Module signing process
   - Module loading sequence and dependencies
   - vendor_dlkm vs system_dlkm vs vendor_ramdisk modules
   - How insmod_cfg files control module loading
   - Module dependencies and loading order

5. **Boot Process and Partition Structure**
   - What is boot.img and what does it contain?
   - What is vendor_kernel_boot.img and how does it differ?
   - How do these images combine at boot?
   - Partition layout (static vs dynamic)
   - How the bootloader loads and executes kernel
   - Initramfs and ramdisk structure

6. **Root Requirements and Kernel Modifications**
   - What kernel changes are needed for root?
   - SELinux modifications
   - Kernel feature flags for root
   - Can root be achieved with modules or needs kernel source changes?
   - Impact on system stability and security
   - Compatibility with system updates

### Priority 3: MEDIUM (Important for Optimization)

7. **Device Tree System**
   - How DTS/DTSI files are structured
   - How DTBO (overlays) work
   - Device tree compilation process
   - How device tree matches hardware at runtime
   - Multiple variant support (caiman, komodo, tokay)

8. **KMI (Kernel Module Interface) System**
   - What is KMI and why does it exist?
   - KMI symbol lists - what symbols are exposed?
   - KMI enforcement during build
   - What happens if KMI is violated?
   - How to add new KMI symbols if needed

9. **Flashing and Deployment Process**
   - Exact flash sequence for Pixel 9 Pro XL
   - A/B slot system
   - Fastboot commands and partition names
   - Recovery process if flash fails
   - Backup and restore procedures

### Priority 4: MEDIUM-LOW (Nice to Have)

10. **Build Optimization**
    - How to speed up builds
    - Incremental build strategies
    - Build caching
    - Parallel builds
    - Local vs sandboxed builds

11. **Error Handling and Debugging**
    - Common build errors and solutions
    - Module loading failures
    - Kernel boot failures
    - Debugging tools and techniques
    - Log analysis

12. **Code Organization**
    - Where is common kernel code?
    - Where is SoC-specific code (zumapro)?
    - Where is device-specific code (caimito)?
    - Where are external modules?
    - How code is organized and structured

---

## Analysis Order (Iterative Approach)

### Phase 1: Foundation (Current)
✅ Basic understanding of build system
✅ Identification of key files and locations
✅ High-level architecture overview

### Phase 2: GKI Deep Dive (STARTING NOW)
🔍 **Analysis 1: GKI Architecture - Complete Understanding**
   - This is THE most fundamental concept
   - Everything else builds on understanding GKI
   - Will clarify base_kernel vs full kernel decision
   - Will explain compatibility implications

### Phase 3: Build Process (Next)
🔍 Analysis 2: Build Configuration System
🔍 Analysis 3: Build Command Flow

### Phase 4: Runtime Behavior (Then)
🔍 Analysis 4: Module System
🔍 Analysis 5: Boot Process
🔍 Analysis 6: Root Requirements

### Phase 5: Advanced Topics (Finally)
🔍 Analysis 7-12: Remaining topics

---

## Documentation Structure

Each analysis will produce:
1. **Conceptual Explanation** - What is it? Why does it exist?
2. **How It Works** - Technical details
3. **Code Walkthrough** - Actual code analysis
4. **Practical Implications** - What this means for our objective
5. **Visual Diagrams** - Where helpful
6. **Key Takeaways** - Summary points

All documentation will be organized for newcomers with:
- Clear definitions
- Progressive complexity (simple → advanced)
- Real examples from codebase
- Practical implications
- Links between concepts

---

## Current Status

- ✅ Phase 1 Complete: Initial analysis done
- 🔄 Phase 2 Starting: GKI Architecture deep dive
- ⏳ Phase 3-5: Pending

