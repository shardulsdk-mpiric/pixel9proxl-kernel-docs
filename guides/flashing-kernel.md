# Flashing Custom Kernel on Pixel 9 Pro XL

This guide provides safe, step-by-step instructions for flashing your custom kernel with KernelSU on the Pixel 9 Pro XL (caimito) device while maintaining system compliance and avoiding breakage.

## Prerequisites

### 1. Device Requirements
- **Unlocked Bootloader**: Your device must have an unlocked bootloader
- **USB Debugging**: Enabled in Developer Options
- **ADB and Fastboot**: Installed on your computer
- **Battery**: At least 50% charge (recommended 80%+)
- **Backup**: Complete backup of your device data (flashing can cause data loss)

### 2. Software Requirements
- `adb` and `fastboot` tools installed
- USB cable for device connection
- Build artifacts from `out/caimito/dist/`

### 3. Safety Checklist
- [ ] Bootloader is unlocked
- [ ] USB debugging enabled
- [ ] Device drivers installed
- [ ] Original boot images backed up
- [ ] Device data backed up
- [ ] Battery sufficiently charged

## Understanding the Partition Structure

Pixel 9 Pro XL uses **A/B (Seamless) Updates** with the following critical partitions:

### Primary Partitions
- **`boot`** / **`boot_a`** / **`boot_b`**: Contains kernel Image and ramdisk
- **`vendor_boot`** / **`vendor_boot_a`** / **`vendor_boot_b`**: Contains vendor ramdisk and DTBOs
- **`dtbo`** / **`dtbo_a`** / **`dtbo_b`**: Device Tree Overlay partition
- **`vendor_dlkm`** / **`vendor_dlkm_a`** / **`vendor_dlkm_b`**: Vendor Dynamic Loadable Kernel Modules
- **`system_dlkm`** / **`system_dlkm_a`** / **`system_dlkm_b`**: System Dynamic Loadable Kernel Modules

### Build Artifacts
From your build (`out/caimito/dist/`), you'll find:
- **`boot.img`**: Main boot image (kernel + ramdisk)
- **`vendor_kernel_boot.img`**: Vendor boot image (vendor ramdisk + DTBOs)
- **`dtb.img`**: Device Tree Blob image
- **`vendor_dlkm.img`**: Vendor modules image
- **`system_dlkm.img`**: System modules image

## Step-by-Step Flashing Procedure

### Step 1: Backup Current Partitions

**CRITICAL**: Always backup your current partitions before flashing. This allows you to restore if something goes wrong.

```bash
# Connect device via USB and enable USB debugging
adb devices

# Reboot to bootloader
adb reboot bootloader

# Wait for device to enter fastboot mode
fastboot devices

# Backup current partitions (replace /path/to/backup with your backup location)
BACKUP_DIR="/path/to/backup/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

# Determine current slot
CURRENT_SLOT=$(fastboot getvar current-slot 2>&1 | grep "current-slot" | awk '{print $2}')
echo "Current slot: $CURRENT_SLOT"

# Backup boot partition
fastboot flash boot_a "$BACKUP_DIR/boot_a_backup.img" 2>/dev/null || \
fastboot getvar current-slot 2>&1 | grep -q "a" && \
fastboot boot "$BACKUP_DIR/boot_a_backup.img" 2>/dev/null || \
echo "Note: Direct backup may not work, use 'fastboot boot' or extract from device"

# Alternative: Extract from device (if device is booted)
adb shell "dd if=/dev/block/by-name/boot_a of=/sdcard/boot_a_backup.img bs=1M"
adb pull /sdcard/boot_a_backup.img "$BACKUP_DIR/"
adb shell "dd if=/dev/block/by-name/boot_b of=/sdcard/boot_b_backup.img bs=1M"
adb pull /sdcard/boot_b_backup.img "$BACKUP_DIR/"

# Backup vendor_boot
adb shell "dd if=/dev/block/by-name/vendor_boot_a of=/sdcard/vendor_boot_a_backup.img bs=1M"
adb pull /sdcard/vendor_boot_a_backup.img "$BACKUP_DIR/"
adb shell "dd if=/dev/block/by-name/vendor_boot_b of=/sdcard/vendor_boot_b_backup.img bs=1M"
adb pull /sdcard/vendor_boot_b_backup.img "$BACKUP_DIR/"

# Backup dtbo
adb shell "dd if=/dev/block/by-name/dtbo_a of=/sdcard/dtbo_a_backup.img bs=1M"
adb pull /sdcard/dtbo_a_backup.img "$BACKUP_DIR/"
adb shell "dd if=/dev/block/by-name/dtbo_b of=/sdcard/dtbo_b_backup.img bs=1M"
adb pull /sdcard/dtbo_b_backup.img "$BACKUP_DIR/"

# Backup vendor_dlkm
adb shell "dd if=/dev/block/by-name/vendor_dlkm_a of=/sdcard/vendor_dlkm_a_backup.img bs=1M"
adb pull /sdcard/vendor_dlkm_a_backup.img "$BACKUP_DIR/"
adb shell "dd if=/dev/block/by-name/vendor_dlkm_b of=/sdcard/vendor_dlkm_b_backup.img bs=1M"
adb pull /sdcard/vendor_dlkm_b_backup.img "$BACKUP_DIR/"

# Backup system_dlkm
adb shell "dd if=/dev/block/by-name/system_dlkm_a of=/sdcard/system_dlkm_a_backup.img bs=1M"
adb pull /sdcard/system_dlkm_a_backup.img "$BACKUP_DIR/"
adb shell "dd if=/dev/block/by-name/system_dlkm_b of=/sdcard/system_dlkm_b_backup.img bs=1M"
adb pull /sdcard/system_dlkm_b_backup.img "$BACKUP_DIR/"

echo "Backups saved to: $BACKUP_DIR"
```

### Step 2: Verify Build Artifacts

Ensure all required images are present:

```bash
cd /mnt/work_4gb/sources/android/pixel_9_pro_xl/002_kernel

# Check for required images
REQUIRED_IMAGES=(
    "out/caimito/dist/boot.img"
    "out/caimito/dist/vendor_kernel_boot.img"
    "out/caimito/dist/vendor_dlkm.img"
    "out/caimito/dist/system_dlkm.img"
)

for img in "${REQUIRED_IMAGES[@]}"; do
    if [ -f "$img" ]; then
        echo "✓ Found: $img ($(du -h "$img" | cut -f1))"
    else
        echo "✗ Missing: $img"
        exit 1
    fi
done

# Verify .config contains KernelSU
if grep -q "^CONFIG_KSU=y" out/caimito/dist/.config; then
    echo "✓ KernelSU is enabled in kernel config"
else
    echo "✗ KernelSU not found in kernel config!"
    exit 1
fi
```

### Step 3: Determine Target Slot

Pixel devices use A/B slots. Flash to the **inactive slot** first for safety:

```bash
# Reboot to bootloader
adb reboot bootloader
sleep 5

# Get current slot
CURRENT_SLOT=$(fastboot getvar current-slot 2>&1 | grep "current-slot" | awk '{print $2}')
echo "Current active slot: $CURRENT_SLOT"

# Determine target slot (flash to inactive slot)
if [ "$CURRENT_SLOT" = "a" ]; then
    TARGET_SLOT="b"
else
    TARGET_SLOT="a"
fi

echo "Will flash to slot: $TARGET_SLOT"
```

### Step 4: Flash Kernel Images

**Important**: Flash to the **inactive slot** first. This allows you to test without breaking your current boot.

```bash
cd /mnt/work_4gb/sources/android/pixel_9_pro_xl/002_kernel/out/caimito/dist

# Ensure device is in fastboot mode
fastboot devices

# Flash boot image (contains kernel Image)
echo "Flashing boot.img to slot $TARGET_SLOT..."
fastboot flash "boot_$TARGET_SLOT" boot.img

# Flash vendor_kernel_boot (contains vendor ramdisk and DTBOs)
echo "Flashing vendor_kernel_boot.img to slot $TARGET_SLOT..."
fastboot flash "vendor_boot_$TARGET_SLOT" vendor_kernel_boot.img

# Flash vendor_dlkm (vendor modules)
echo "Flashing vendor_dlkm.img to slot $TARGET_SLOT..."
fastboot flash "vendor_dlkm_$TARGET_SLOT" vendor_dlkm.img

# Flash system_dlkm (system modules)
echo "Flashing system_dlkm.img to slot $TARGET_SLOT..."
fastboot flash "system_dlkm_$TARGET_SLOT" system_dlkm.img

# If dtb.img exists, flash it (may be part of vendor_kernel_boot)
if [ -f "dtb.img" ]; then
    echo "Flashing dtb.img..."
    # Note: DTB may be included in vendor_kernel_boot, check if separate partition exists
    # fastboot flash "dtbo_$TARGET_SLOT" dtb.img
fi

echo "Flashing complete!"
```

### Step 5: Verify Flashed Images

```bash
# Verify partitions were flashed correctly
fastboot getvar "slot-count"
fastboot getvar "current-slot"
fastboot getvar "slot-success:$TARGET_SLOT"
fastboot getvar "slot-bootable:$TARGET_SLOT"
```

### Step 6: Test Boot (Optional - Boot Without Permanently Switching)

Before permanently switching slots, you can test boot from the new slot:

```bash
# Boot from the new slot without switching
fastboot boot "boot_$TARGET_SLOT" boot.img

# OR set the slot as bootable and reboot
fastboot set_active "$TARGET_SLOT"
fastboot reboot
```

**Note**: `fastboot boot` is a one-time boot and doesn't modify partitions. Use this for initial testing.

### Step 7: Switch Active Slot (After Verification)

Only switch slots after you've verified the kernel boots correctly:

```bash
# Switch to the new slot
fastboot set_active "$TARGET_SLOT"

# Verify slot switch
fastboot getvar current-slot

# Reboot
fastboot reboot
```

### Step 8: Post-Flash Verification

After device boots:

```bash
# Wait for device to fully boot, then connect via ADB
adb wait-for-device
sleep 30  # Wait for system to stabilize

# Verify kernel version
adb shell uname -r

# Verify KernelSU is present
adb shell "cat /proc/config.gz 2>/dev/null | zcat | grep CONFIG_KSU || echo 'Config not available, checking via KernelSU Manager app'"

# Check KernelSU status (if KernelSU Manager is installed)
adb shell "su -c 'ksud --version' 2>/dev/null || echo 'Install KernelSU Manager app to verify'"
```

## Safety Measures and Rollback

### If Device Fails to Boot

1. **Boot to Bootloader**:
   - Hold Power + Volume Down until device enters fastboot mode
   - Or use hardware key combination for your device

2. **Switch Back to Previous Slot**:
   ```bash
   fastboot set_active "$CURRENT_SLOT"  # The slot you were on before
   fastboot reboot
   ```

3. **Restore from Backup** (if slot switch doesn't work):
   ```bash
   # Restore boot partition
   fastboot flash "boot_$TARGET_SLOT" "$BACKUP_DIR/boot_${TARGET_SLOT}_backup.img"
   
   # Restore vendor_boot
   fastboot flash "vendor_boot_$TARGET_SLOT" "$BACKUP_DIR/vendor_boot_${TARGET_SLOT}_backup.img"
   
   # Restore other partitions as needed
   fastboot flash "vendor_dlkm_$TARGET_SLOT" "$BACKUP_DIR/vendor_dlkm_${TARGET_SLOT}_backup.img"
   fastboot flash "system_dlkm_$TARGET_SLOT" "$BACKUP_DIR/system_dlkm_${TARGET_SLOT}_backup.img"
   
   fastboot set_active "$TARGET_SLOT"
   fastboot reboot
   ```

### Maintaining System Compliance

1. **Keep Original Partitions**: Always keep backups of original partitions
2. **Test Before Permanent**: Use `fastboot boot` to test before switching slots
3. **Match Android Version**: Ensure your kernel matches your device's Android version
4. **Keep Modules in Sync**: Vendor and system modules must match kernel version
5. **Verify KMI Compatibility**: Your kernel should maintain KMI (Kernel Module Interface) compatibility

### Common Issues and Solutions

#### Issue: Device Bootloops
- **Solution**: Switch back to previous slot immediately
- **Prevention**: Always test with `fastboot boot` first

#### Issue: "Partition not found" Error
- **Solution**: Check partition name with `fastboot getvar all | grep partition-type`
- **Note**: Some devices may use different partition naming

#### Issue: "Remote: Partition flashing is not allowed"
- **Solution**: Ensure bootloader is unlocked: `fastboot flashing unlock`
- **Warning**: This will wipe device data!

#### Issue: Kernel Version Mismatch
- **Solution**: Rebuild kernel matching your device's Android version
- **Check**: `adb shell getprop ro.build.version.release`

## Advanced: Flashing Both Slots

For maximum safety, you can flash both slots:

```bash
# Flash slot A
fastboot flash boot_a boot.img
fastboot flash vendor_boot_a vendor_kernel_boot.img
fastboot flash vendor_dlkm_a vendor_dlkm.img
fastboot flash system_dlkm_a system_dlkm.img

# Flash slot B
fastboot flash boot_b boot.img
fastboot flash vendor_boot_b vendor_kernel_boot.img
fastboot flash vendor_dlkm_b vendor_dlkm.img
fastboot flash system_dlkm_b system_dlkm.img

# Keep current slot active
fastboot reboot
```

## Post-Flash: Installing KernelSU Manager

After successful boot:

1. Download KernelSU Manager APK from [official repository](https://github.com/tiann/KernelSU)
2. Install via ADB:
   ```bash
   adb install kernelsu-manager.apk
   ```
3. Open KernelSU Manager app
4. Verify root access works
5. Grant root permissions to apps as needed

## Verification Checklist

After flashing, verify:

- [ ] Device boots successfully
- [ ] Kernel version matches your build
- [ ] KernelSU Manager detects KernelSU
- [ ] Root access works
- [ ] All device functions work (WiFi, Bluetooth, etc.)
- [ ] No random reboots or crashes
- [ ] System stability maintained

## Important Notes

1. **Data Safety**: Flashing kernels can cause data loss. Always backup important data.
2. **Warranty**: Flashing custom kernels may void warranty.
3. **OTA Updates**: Custom kernels may break OTA updates. You'll need to reflash after system updates.
4. **Compatibility**: Ensure kernel matches your device's Android version and build number.
5. **Testing**: Test thoroughly before using as daily driver.

## Troubleshooting

See [KernelSU Troubleshooting Guide](kernelsu-troubleshooting.md) for issues specific to KernelSU.

For general kernel flashing issues, refer to:
- [Boot Process Analysis](../analysis/05-boot-process.md)
- [Flashing and Deployment Analysis](../analysis/09-flashing-deployment.md)

