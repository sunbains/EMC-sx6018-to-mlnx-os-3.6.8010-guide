# Converting an EMC OEM Mellanox SX6018 (`100-886-230-02`) to Standard MLNX-OS `3.6.8010`

## Fetching the OS Image
Download the OS image from NVIDIA's [archive](https://network.nvidia.com/support/firmware/lenovo-archive/) - click on "Switches" and search the page by OS version number.

## Overview & Hardware Verification
- **Switch Model:** EMC OEM Mellanox SX6018 (`100-886-230-02` / `MSX6018F-2SFS` / `MSX6018-2BFS`)
- **CPU / Architecture:** PowerPC 460EX (`PPC_M460EX`, 2 GB RAM, 16 MB NOR flash + 1 GiB Micron NAND flash)
- **Switch ASIC:** SwitchX-2
- **Target Firmware:** `image-PPC_M460EX-3.6.8010.img`
- **U-Boot Layout Profile:** `MFL1` (`-L MFL1 -d /dev/mtd -k uni -w M460EX`)

Because EMC OEM SX6018 switches ship with a stripped-down EMC SwitchOS (`1.xxx`) that lacks standard `show version` or `image fetch` commands, conversion is performed via the U-Boot bootloader using a serial console (`9600 8N1`) and a FAT32 USB flash drive. Existing U-Boot (`U-Boot 2009.01 SX_PPC_M460EX`) already includes the standard Mellanox `mlxlinux`, `mfg_usb`, and `boot_mlxlinux` environment variables, so **U-Boot itself does not need to be reflashed**.

---

## Prerequisites
1. **Serial Console Connection:** RJ45-to-DB9/USB console cable set to **`9600` baud, `8N1`, no flow control**.
2. **USB Flash Drive:** Formatted as **FAT32** with an **MBR** partition table.
3. **Host Machine (macOS):** Terminal with Homebrew `u-boot-tools` (`mkimage`) installed:
   ```bash
   brew install u-boot-tools cpio
   ```
4. **Source Firmware Image:** `image-PPC_M460EX-3.6.8010.img` (standard `PPC_M460EX` MLNX-OS 3.6.8010 TPKG image).

---

## Phase 1: Prepare the USB Recovery Files on macOS

### 1. Unpack the `.img` Archive and Extract Kernel & Device Tree
A Mellanox `.img` file is a ZIP archive containing a `.tgz` rootfs tarball (`image-PPC_M460EX-ppc-m460ex-20180820-180416.tgz`).

```bash
mkdir -p ~/mlnx_extract && cd ~/mlnx_extract
unzip /path/to/image-PPC_M460EX-3.6.8010.img

# Extract the PowerPC uniprocessor kernel and device tree blob
tar -xzf image-PPC_M460EX-ppc-m460ex-20180820-180416.tgz ./boot/vmlinuz-uni ./boot/fdt-uni
cp boot/vmlinuz-uni vmlinuz
cp boot/fdt-uni fdt
```

### 2. Build a Compact PowerPC Recovery `rootfs` (`uRamdisk`)
U-Boot on the SX6018 allocates `180224` KB (~176 MB) for the initial ramdisk (`mfg_ramdisk_size=180224`) and reads USB at USB 1.1 speeds. Unpacking the full `.tgz` (~1.1 GB) exceeds RAM limits and triggers macOS case-insensitive `terminfo` / `/dev` extraction errors. Extract only the required OS binaries/libraries and `/lib/modules`, create a recovery `/init`, and wrap it as a PowerPC `cpio.gz` U-Boot ramdisk:

```bash
mkdir -p staging

# Extract base filesystem excluding /dev nodes and case-colliding terminfo/doc directories
sudo tar -xzf image-PPC_M460EX-ppc-m460ex-20180820-180416.tgz -C staging \
  --exclude='./dev/*' \
  --exclude='./usr/share/terminfo*' \
  --exclude='./usr/share/doc*' \
  --exclude='./usr/share/man*' \
  --exclude='./usr/share/locale*' \
  --exclude='./usr/lib/python*' \
  --exclude='./usr/lib/ruby*' \
  --exclude='./debug_symbols*' \
  --exclude='./boot/*'

# Remove heavy SwitchX SDK, firmware, debug symbols, and non-essential user binaries
# (Keep /bin, /sbin, /lib, /lib/modules, /etc, and core /usr tools)
sudo rm -rf staging/opt \
            staging/var/opt \
            staging/lib/.debug \
            staging/usr/lib/.debug \
            staging/usr/lib/mft \
            staging/usr/lib/libissd* \
            staging/usr/lib/libtestx* \
            staging/usr/lib/libsdkrouter* \
            staging/usr/lib/libnss3* \
            staging/usr/sbin/opensm \
            staging/usr/sbin/httpd \
            staging/usr/sbin/tcpdump \
            staging/usr/bin/gdb \
            staging/usr/bin/lri \
            staging/usr/bin/mlxfwmanager \
            staging/usr/bin/mlx* \
            staging/usr/bin/mget_temp* \
            staging/usr/bin/flint* \
            staging/usr/bin/sx_* \
            staging/usr/bin/switchx* \
            staging/usr/bin/topo_mgr \
            staging/usr/bin/ib* \
            staging/usr/bin/itrace

# Confirm uncompressed staging size is ~140 MB (under 176 MB limit)
sudo du -sh staging
```

Create `/init` inside `staging/`:

```bash
sudo tee staging/init > /dev/null << 'EOF'
#!/bin/sh
export PATH=/sbin:/bin:/usr/sbin:/usr/bin:/opt/tms/bin
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev 2>/dev/null || mount -t tmpfs tmpfs /dev
mkdir -p /dev/pts /mnt/usb /var/root/tmp /var/run /var/lock
mount -t devpts devpts /dev/pts

[ -c /dev/console ] || mknod -m 622 /dev/console c 5 1
[ -c /dev/ttyS0 ]   || mknod -m 666 /dev/ttyS0 c 4 64
[ -c /dev/null ]    || mknod -m 666 /dev/null c 1 3
[ -c /dev/zero ]    || mknod -m 666 /dev/zero c 1 5

echo "=== Mellanox PPC_M460EX Recovery Shell ==="
exec /bin/sh </dev/ttyS0 >/dev/ttyS0 2>&1
EOF
sudo chmod +x staging/init
```

Pack `staging` into `rootfs.cpio.gz` and wrap with `mkimage` for PowerPC (`-A ppc`):

```bash
sudo find staging -name ".DS_Store" -delete
(cd staging && sudo find . | sudo cpio -H newc -o | gzip -9 > ../rootfs.cpio.gz)
mkimage -A ppc -O linux -T ramdisk -C gzip -n "MLNX460EX Recovery" -d rootfs.cpio.gz rootfs
ls -lh rootfs
```
*(Resulting `rootfs` size is ~54–57 MB compressed.)*

### 3. Populate the FAT32 USB Flash Drive
Copy the following four files onto the FAT32 USB drive using this exact directory structure:

```text
/
├── image-PPC_M460EX-3.6.8010.img
└── mlnx460ex/
    ├── vmlinuz
    ├── fdt
    └── rootfs
```

---

## Phase 2: Boot into the Recovery Linux Shell via U-Boot

1. Insert the FAT32 USB drive into the SX6018 USB port.
2. Connect via serial console (`9600 8N1`) and power on the switch.
3. When `Hit any key to stop autoboot` appears, press any key to enter the **`=>`** U-Boot prompt.
4. Override `mfg_extra_args` to launch `/bin/sh` directly (bypassing SysVinit `rc.sysinit`) and boot from USB:

```text
setenv mfg_extra_args ramdisk=65536 init=/bin/sh
run mfg_usb
```

5. Wait **~60–90 seconds** while U-Boot transfers `rootfs` over USB 1.1. The kernel will boot and drop to the **`sh-4.1#`** prompt.

---

## Phase 3: Mount Filesystems, Load USB Drivers & Install MLNX-OS

### 1. Mount Virtual Filesystems and Verify MTD Partitions
At the `sh-4.1#` prompt, mount `/proc`, `/sys`, and `devtmpfs`, and verify the NOR/NAND MTD partitions:

```bash
export PATH=/sbin:/bin:/usr/sbin:/usr/bin:/opt/tms/bin
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev
mkdir -p /var/root/tmp /var/run /var/lock /mnt/usb

cat /proc/mtd
```

Expected `/proc/mtd` layout (`MFL1`):
- `mtd0`–`mtd5` (NOR flash): `KERNEL_1`, `FDT_1`, `KERNEL_2`, `FDT_2`, `UBOOTENV`, `UBOOT`
- `mtd6`–`mtd9` (1 GiB Micron NAND flash): `ROOT_1`, `ROOT_2`, `CONFIG`, `VAR`

### 2. Load USB Storage & SCSI Disk Kernel Modules and Mount USB
Because USB and SCSI disk support are built as loadable modules in `vmlinuz-uni`, load them in order and mount `/dev/sda1` with `-t vfat` explicit:

```bash
depmod -a
modprobe usb-storage
modprobe vfat
modprobe ehci-hcd
modprobe sd_mod
sleep 3

ls -la /dev/sd*
mount -t vfat /dev/sda1 /mnt/usb
ls -lh /mnt/usb
```

### 3. Run `writeimage.sh` in Manufacture Mode (`MFL1`)
Execute Mellanox's manufacturing installer to repartition/format the MTD devices and install MLNX-OS `3.6.8010` (note that `-w M460EX` must be uppercase to match the image's hardware compatibility string):

```bash
writeimage.sh -m -f /mnt/usb/image-PPC_M460EX-3.6.8010.img -L MFL1 -d /dev/mtd -k uni -w M460EX -I -v
```

> **Important:** Do not interrupt power while `writeimage.sh` erases NOR flash and formats/populates the JFFS2 NAND partitions (`ROOT_1`, `ROOT_2`, `CONFIG`, `VAR`).

### 4. Reboot Back to U-Boot
Once `writeimage.sh` finishes cleanly, sync, unmount the USB stick, remove it, and force-reboot (`reboot -f` is required because `/sbin/init` is not running):

```bash
sync
umount /mnt/usb
reboot -f
```

---

## Phase 4: Permanently Switch U-Boot to Boot Standard MLNX-OS

1. During reboot, interrupt U-Boot (`Hit any key to stop autoboot`) to reach the **`=>`** prompt.
2. Run the built-in `boot_mlxlinux` macro to permanently change `flash_jffs2` from `run emcflash` to `run mlxlinux` and save the environment to NOR flash:

```text
run boot_mlxlinux
```

3. Boot into standard MLNX-OS `3.6.8010`:

```text
run mlxlinux
```

---

## Phase 5: First-Boot Verification
1. Allow **3–5 minutes** on first boot for MLNX-OS to complete `firstboot.sh` and apply initial configuration.
2. At the `switch login:` prompt, log in with default credentials:
   - **Username:** `admin`
   - **Password:** `admin`
3. Enter privileged mode and verify the OS and SwitchX-2 ASIC firmware versions:

It will ask you a few questions during first boot, hostname, IPV6, the answers are obvious.

```text
enable
show version
show inventory
show asic-version
```
