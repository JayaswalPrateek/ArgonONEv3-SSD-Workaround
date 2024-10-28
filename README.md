# Argon ONE v3 SSD Workaround

## Problem Overview
Some NVMe SSDs, despite being detected by the system, are incompatible with direct booting on the Raspberry Pi 5 when using the Argon ONE V3 M.2 NVMe PCIe Case. This guide provides a workaround by using a hybrid boot setup: SD card for boot partition and NVMe SSD for root partition.

## Prerequisites
- Raspberry Pi 5
- Argon ONE V3 M.2 NVMe PCIe Case
- A microSD card (8GB minimum recommended)
- An NVMe SSD that's detected but won't boot
- A working Raspberry Pi OS installation on the SD card(Any variant: Regular/Full/Lite-Headless)

## Symptoms
- The SSD is detected by `lspci` and `lsblk` commands after following the manual
  - link to the manual: https://argon40.com/blogs/argon-resources/argon-one-v3-case-and-m-2-nvme-pcie-case-manual
- Boot attempts result in errors like `Failed to open device: "nvme"`
- The Pi 5 fails to boot from the SSD and falls back to other boot devices(like SD Card)

## Technical Background
The Raspberry Pi 5's boot process requires two main partitions:
1. Boot partition (FAT32) - Contains firmware, bootloader, and kernel
2. Root partition (ext4) - Contains the operating system

The issue typically occurs because some NVMe SSD controllers don't properly handle the boot partition interaction with the Pi 5's firmware. However, these SSDs can still function perfectly for the root partition.

## Step-by-Step Workaround

### 1. Verify Current Setup
1. Boot your Pi 5 from the SD card
2. Verify SSD detection:
   ```bash
   lspci | grep -i nvme
   lsblk
   ```
3. Note down your device names (typically `/dev/mmcblk0` for SD card and `/dev/nvme0n1` for NVMe SSD)

### 2. Clone Disk Layout
1. Download the backup script provided above
2. Make it executable:
   ```bash
   chmod +x backup
   ```
3. Clone the SD card layout to the NVMe SSD:
   ```bash
   sudo ./backup /dev/mmcblk0 /dev/nvme0n1
   ```
   **WARNING**: This will erase all data on the NVMe SSD!

### 3. Configure Boot and Root Partitions
1. Get the partition UUIDs:
   ```bash
   sudo blkid
   ```
2. Note down:
   - SD card boot partition UUID (PARTUUID for `/dev/mmcblk0p1`)
   - NVMe SSD root partition UUID (PARTUUID for `/dev/nvme0n1p2`)

3. Update boot configuration to correctly locate the root partition on the SSD:
   ```bash
   sudo nano /boot/firmware/cmdline.txt
   ```
   - Locate the `root=PARTUUID=XXXXXXXX-02` parameter
   - Replace `XXXXXXXX-02` with your NVMe SSD's root partition PARTUUID as noted previously

4. Mount and update the SSD's root partition to correctly locate the boot partition on the SD Card:
   ```bash
   sudo mount /dev/nvme0n1p2 /mnt
   sudo nano /mnt/etc/fstab
   ```
   - Locate the line containing `/boot/firmware`
   - Update the PARTUUID to match your SD card's boot partition
   - Save and unmount:
   ```bash
   sudo umount /mnt
   ```

### 4. Configure Boot Order
1. Run the configuration utility:
   ```bash
   sudo raspi-config
   ```
2. Navigate: Advanced Options → Boot Order → B1 SD Card Boot
3. Select `<Ok>` and exit

### 5. Final Steps
1. Verify all changes are saved
2. Perform a clean reboot:
   ```bash
   sudo reboot
   ```

## Post-Installation Notes

### Partition Management
- The root partition on the SD card is now redundant
- The boot partition on the NVMe SSD is unused
- You can optionally remove these partitions using `gparted` or similar tools
- The backup script expands the NVMe SSD's root partition to use all remaining space

### Performance Considerations
- The SD card is only accessed during boot
- Regular operation runs entirely from the NVMe SSD
- No significant performance impact compared to full NVMe boot

### Maintenance Notes
- When updating the kernel or bootloader:
  - Updates will be applied to the SD card's boot partition
  - No special action needed for the NVMe root partition
- Keep both SD card and NVMe SSD inserted during updates

### Troubleshooting
If the system fails to boot:
1. Verify partition UUIDs match in both configuration files
2. Check for typos in `cmdline.txt` and `fstab`
3. Ensure both devices are properly connected
4. Try repeating the cloning process if issues persist
5. This is a good reason for keeping the root partition on the SD card around for recovery.
