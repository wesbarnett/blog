---
title: "Arch Linux Installation"
description: "A guide for installing a minimal booting system."
layout: post
toc: true
comments: true
hide: false
search_exclude: false
categories: [linux]
---


{% include warning.html content="This is my personal guide for installing an Arch
Linux system. Use it only as a guide as you follow along with the official
<a href='https://wiki.archlinux.org/index.php/Installation_Guide'>Installation Guide</a>." %}

This guide gives several options along the way, depending on your system.

* For UEFI, EFISTUB is used to boot the kernel directly.
* For systems needing BIOS, syslinux is used.
* Gives options for encrypting the system if desired.
* Gives notes on using btrfs subvolumes if desired.

You may need additional packages for video drivers, etc.

## Pre-installation


[Download](https://www.archlinux.org/download/) the Arch ISO and GnuPG signature.

### Verify signature

If you have GnuPG installed on your current system, verify the download:

    $ gpg --keyserver-options auto-key-retrieve --verify archlinux-<version>-dual.iso.sig

#### Create bootable disk

Create a bootable USB drive by doing the following on an existing Linux installation:

    # dd bs=4M if=/path/to/archlinux-<version>-x86_64.iso of=/dev/sdx status=progress && sync

where `/dev/sdx` is the USB drive.


### Boot the live environment

Now boot from the USB drive.


### Set the keyboard layout

If using a keymap other than US, temporarily set the keyboard layout by doing:

    # loadkeys de-latin1

Change `de-latin1` to a layout found in `/usr/share/kbd/keymaps/**/*.map.gz`.


### Verify the boot mode

Verify that you have booted with UEFI mode by checking that
`/sys/firmware/efi/efivars` exists. If you're not booted in the UEFI, you
should setup your motherboard to do so to follow this installation guide. If
you are not able to use UEFI, this guide has an option to boot from BIOS using
syslinux.


### Connect to the internet

If you have a **wired** connection, it should connect automatically.

If you have a ***wireless*** connection, first stop the wired connection to prevent conflicts:

<pre>
# systemctl stop dhcpcd@<i>interface</i>.service
</pre>

A list of interfaces can be found with:

    # ip link

Then connect to a wifi network with `wpa_supplicant`:

<pre>
# wpa_supplicant -B -i <i>interface</i> -C/run/wpa_supplicant
# wpa_cli -i <i>interface</i>
> scan
> scan_results
> add_network
> set_network 0 ssid "<i>SSID</i>"
> set_network 0 psk "<i>passphrase</i>"
> enable_network 0
> quit
</pre>


Get an ip address with dhcpcd:

    # dhcpcd

For both **wired** and **wireless** connections, check your connection with:

    # ping archlinux.org


### Update the system clock

    # timedatectl set-ntp true


### Partition the disk

{% include warning.html content="All data on your disk will be erased. Backup any data you wish to keep." %}

{% include note.html content="This guide assumes your disk is at <code>/dev/sda</code>. Change if needed." %}

Use `lsblk` to identify existing file systems.

Here are the possible layouts this guide uses. Modify to your needs.

**UEFI, not encrypted**

<table>
<tr>
  <th>Mount point</th>
  <th>Partition</th>
  <th>Partition type</th>
  <th>Suggested size</th>
</tr>
<tr>
  <td><code>/mnt/boot</code></td>
  <td><code>/dev/sda1</code></td>
  <td>
    EFI system partition
    </ul>
  </td>
  <td>550 MiB</td>
</tr>

<tr><td><code>/mnt</code></td>
<td><code>/dev/sda2</code></td>
<td>Linux x86-64 root (/)</td><td>Remainder of the device</td></tr>
</table>

**BIOS, not encrypted**

<table>
<tr>
  <th>Mount point</th>
  <th>Partition</th>
  <th>Partition type</th>
  <th>Suggested size</th>
</tr>
<tr>
  <td><code>/mnt/boot</code></td>
  <td><code>/dev/sda1</code></td>
  <td>
    boot partition
  </td>
  <td>550 MiB</td>
</tr>

<tr><td><code>/mnt</code></td>
<td>
<code>/dev/sda2</code></td>
<td>Linux x86-64 root (/)</td><td>Remainder of the device</td></tr>
</table>

**UEFI, encrypted**
<table>
<tr>
  <th>Mount point</th>
  <th>Partition</th>
  <th>Partition type</th>
  <th>Suggested size</th>
</tr>
<tr>
  <td><code>/mnt/boot</code></td>
  <td><code>/dev/sda1</code></td>
  <td>
    EFI system partition
  </td>
  <td>550 MiB</td>
</tr>

<tr><td><code>/mnt</code></td>
<td>
<code>/dev/mapper/cryptroot</code></td>
<td>Linux x86-64 root (/)</td><td>Remainder of the device</td></tr>
</table>

**BIOS, encrypted**
<table>
<tr>
  <th>Mount point</th>
  <th>Partition</th>
  <th>Partition type</th>
  <th>Suggested size</th>
</tr>
<tr>
  <td><code>/mnt/boot</code></td>
  <td><code>/dev/sda1</code></td>
  <td>
    boot partition
  </td>
  <td>550 MiB</td>
</tr>

<tr><td><code>/mnt</code></td>
<td><code>/dev/mapper/cryptroot</code></td>
<td>Linux x86-64 root (/)</td><td>Remainder of the device</td></tr>
</table>

Use GPT fdisk to format the disk:

    # gdisk /dev/sda

1. Create a new empty GUID partition table by typing `o` at the prompt.
2. Create a new partition by typing `n` at the prompt. Hit `Enter` when
prompted for the partition number, keeping the default of `1`. Hit
`Enter` again for the first section, keeping the default. For the last
sector, type in `+550M` and hit `Enter`.  
3. For an **EFI** system, type in `EF00` to indicate it is an EFI system
partition for
the partition type. Otherwise, use the default.
4. Now create at least one more partition for the installation. To create just
one more partition to fill the rest of the disk, type `n` at the prompt
and use the defaults for the partition number, first sector, last sector, and
hex code.  
5. If setting up a **BIOS** system with syslinux, enter
expert mode by entering `x`. Then enter `a` and then `1` to set
an attribute for partition 1. Then enter `2` to set it as a legacy BIOS
partition and then `Enter` to exit the set attribute menu.  6.  Finally,
write the table to the disk and exit by entering `w` at the prompt.


{% include tip.html content="Here are one-liners for the above layout.<br>
<br>
For <b>EFI</b>:
<pre># sgdisk /dev/sda -o -n 1::+550M -n 2 -t 2:EF00</pre>
For <b>BIOS</b>:
<pre># sgdisk /dev/sda -o -n 1::+550M -n 2 -A 1:set:2i</pre>
" %}


#### Create LUKS container

If encrypting your system with dm-crypt/LUKS, do:

    # cryptsetup lukFormat --type luks2 /dev/sda2
    # cryptsetup open /dev/sda2 cryptroot

Othwerwise, skip this step.


### Format the partitions

Format your ESP as FAT32:

    # mkfs.fat -F32 /dev/sda1

Replace `ext4` with the file system you are using in all of the following.

To format the LUKS container on an **encrypted** system do:

    # mkfs.ext4 /dev/mapper/cryptroot

To format a **regular** (not encrypted) system, do:

    # mkfs.ext4 /dev/sda2

### Mount the file systems

For the **encrypted** setup mount the LUKS container:

    # mount /dev/mapper/cryptroot /mnt

For the **regular** setup mount the root partition:

    # mount /dev/sda2 /mnt

In both cases make a mount point for the boot partition and mount it:

    # mkdir -p /mnt/boot
    # mount /dev/sda1 /mnt/boot

{% include tip.html content="If using btrfs, create any subvolumes you wish to use as mount points
now. Then unmount <code>/mnt</code> and remount your subvolumes to the appropriate mount
points. For example, for the <b>encrypted</b> setup:
<pre>
# mount /dev/mapper/cryptroot /mnt
# btrfs subvolume create /mnt/@
# btrfs subvolume create /mnt/@home
# umount /mnt
# mount -o compress=zstd,subvol=@ /dev/mapper/cryptroot /mnt
# mkdir -p /mnt/home
# mount -o compress=zstd,subvol=@home /dev/mapper/cryptroot /mnt/home
# mkdir -p /mnt/boot
# mount /dev/sda1 /mnt/boot
</pre>
" %}

## Installation

### Select the mirrors

If you desire, edit `/etc/pacman.d/mirrorlist` to select which mirrors have
priority. Higher in the file means higher priority. This file will be copied to
the new installation.

### Install packages

    # pacstrap /mnt base btrfs-progs linux man-db man-pages texinfo vim which

Append additional packages you wish to install to the line. You will have the
opportunity to install more packages in the chroot environment and when you
boot into the new system.

If you have an AMD or Intel processor, you will want to go ahead and install
the `amd-ucode` or `intel-ucode` packages, respectively to enable
microcode updates later in the guide.

## Configure the system

### Fstab

Generate the fstab for your new installation:

    # genfstab -U /mnt > /mnt/etc/fstab

### Chroot

chroot into the new installation:

    # arch-chroot /mnt

### Time zone

Set the time zone:

    # ln -sf /usr/share/zoneinfo/<Region>/<City> /etc/localtime

Set the hardware clock from the system clock:

    # hwclock --systohc

### Localization

Uncomment needed locales in `/etc/locale.gen` (*e.g.*, `en_US.UTF-8`). Then run:

    # locale-gen

Set the `LANG` environment variable the locale in the following file.

{% include hc.html header="/etc/locale.conf" body="LANG=en_US.UTF-8" %}

If you are not using a US keymap, make the keyboard layout permanent:

{% include hc.html header="/etc/vconsole.conf" body="KEYMAP=de-latin1" %}

### Network configuration

#### Hostname

Set the hostname. Change <i>`hostname`</i> to your preferred hostname in the following:

{% include hc.html header="/etc/hostname" body="<i>hostname</i>" %}
{% include hc.html header="/etc/hosts" body="127.0.0.1    localhost
::1    	     localhost
127.0.1.1    <i>hostname</i>.localdomain    <i>hostname</i>" %}

#### Configuration

systemd-networkd will be used to connect to the internet after installation is complete. 

Create a minimal systemd-networkd configuration file with the following
contents. Here <i>`interface`</i> is the wireless interface or the wired interface
if not using wireless.

{% include hc.html header="/etc/systemd/network/config.network" body="
[Match]
Name=<i>interface</i>

[Network]
DHCP=yes" %}

Enable the `systemd-networkd.service` unit.

#### DNS resolution

To use systemd-resolved for DNS resolution, create a symlink as follows: 

    # ln -s /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf

Enable the `systemd-resolved.service` unit.

#### Wireless


If using a wireless interface, install the `iwd` package now:

    # pacman -S iwd

Additionally enable the `iwd.service` for wireless on boot.

{% include note.html content="To prevent a known race condition that causes wireless device
renaming problems, create a drop-in file for <code>iwd.service</code> with the following:
<pre>
[Unit]
After=systemd-udevd.service systemd-networkd.service
</pre>" %}

### Initramfs


If using an **encrypted** system, add the `encrypt` hook to your mkinitcpio
configuration as shown below. Add the `keymap` hook if you are not using the
default US keymap. If using btrfs, you can remove the `fsck` hook.

{% include hc.html header="/etc/mkinitcpio.conf" body="HOOKS=(base udev autodetect modconf block filesystems keyboard fsck <b>keymap encrypt</b>)" %}

{% include tip.html content="Move <code>keyboard</code> in front of <code>autodetect</code> if using an external
USB keyboard that was not connected when the image is created." %}


Regenerate initramfs:

    # mkinitcpio -p linux

### Root password

Set the root password:

    # passwd

{% include tip.html content="You can skip this step if you are giving your normal user super user privileges via sudo." %}

### Add normal user


Install the `sudo` package:

    # pacman -S sudo

Add a normal user, add it to the `wheel` group, and set the password as
follows, where `user` is the name of your user:

    # useradd -m -G wheel user
    # passwd user

Open the sudoers file and uncomment the `wheel` group, giving that user access
to sudo:

    # EDITOR=vim visudo

### Swap file

If using btrfs, first create a subvolume for the swap file to reside on.
Then, create an empty swap file and set it to not use COW:

    # btrfs subvolume create /.swap
    # truncate -s 0 /.swap/swapfile
    # chattr +C /.swap/swapfile

If not using btrfs, simply create a directory:

    # mkdir /.swap

In all cases do:

    # dd if=/dev/zero of=/.swap/swapfile bs=1M count=2048
    # chmod 600 /.swap/swapfile
    # mkswap /.swap/swapfile

Update fstab with a line for the swap file as follows:

    # /etc/fstab
    /.swap/swapfile none swap defaults 0 0

{% include warning.html content="If using btrfs instead of ext4, do <b>not</b> use a swap
file for kernels before v5.0, since it may cause file system corruption. Instead, use a
swap partition (not covered here)." %}

### Boot loader

Two options are provided here. If you have a UEFI motherboard, use EFISTUB. Otherwise,
use the BIOS setup with syslinux. In either case, you will need to set your kernel
parameters.


#### EFISTUB

In the UEFI setup we are not using a boot loader. Instead we are booting the
kernel directly via EFISTUB. Previously you should have created a EFI
system partition of the size 550MiB and marked it with the partition type
`EF00`.


Exit the chroot and reboot the system. From your Arch Linux live disk, boot
into the UEFI Shell v2. Then do:

    Shell> map

Note the disk number for the hard drive where you are installing Arch Linux.
This guide assumes it is `1`.

Now create two UEFI entries using bcfg.

    Shell> bcfg boot add 0 fs1:vmlinuz-linux "Arch Linux"
	Shell> bcfg boot add 1 fs1:vmlinuz-linux "Arch Linux (Fallback)"


Create a file with your kernel parameters as a single line:


    Shell> edit fs1:options.txt

{% include note.html content="Create at least one additional space before the first character of your
boot line in your options files. Otherwise, the first character gets squashed
by a byte order mark and will not be passed to the initramfs, resulting in an
error when booting. Additionally, your options file should be one line, and one
line only." %}


Press `F2` to save and `F3` to quit. Now add that file as the options to your first boot entry:

    Shell> bcfg boot -opt 0 fs1:options.txt

Repeat the above process for your second, fallback entry, creating a text file
named `options-fallback.txt` containing a single line with your kernel
parameters, chaning the intird to the fallback image (*i.e.*,
`/initramfs-linux-fallback.img`).

Add it to the entry using `bcfg boot -opt 1 fs:1\options-fallback.txt`.

#### syslinux

In the BIOS setup we are using syslinux. Previously you should have created a
boot partition that was marked with the attribute "legacy BIOS bootable".

Now, while still in the chroot install `syslinux`:

    # pacman -S syslinux

Create the following configuration file:

{% include hc.html header="/boot/syslinux/syslinux.cfg" body="
DEFAULT arch
LABEL arch
  MENU LABEL Arch Linux
  LINUX ../vmlinuz-linux
  APPEND <i>kernel-parameters</i>

LABEL archfallback
  MENU LABEL Arch Linux
  LINUX ../vmlinuz-linux
  APPEND <i>fallback-kernel-parameters</i>" %}

where <i>`kernel-parameters`</i> is from the kernel parameters you will create in the
next section.
<i>`fallback-kernel-parameters`</i> is exactly the same except with `initrd`
pointing to the fallback initramfs (i.e.,
`/initramfs-linux-fallback.img`).

Exit the chroot and unmount all of the partitions:

    # umount -R /mnt

Then install the bootloader:

    # syslinux --directory syslinux /dev/sda1

And install the MBR:

    # dd bs=440 count=1 conv=notrunc if=/usr/lib/syslinux/bios/gptmbr.bin of=/dev/sda

#### Kernel parameters

No matter what boot loader you use, you need to pass some kernel parameters to
it as indicated in the above sections.

For an **encrypted** system, the kernel parameters will contain at least:

    root=/dev/mapper/cryptroot ro initrd=/initramfs-linux.img init=/usr/lib/systemd/systemd cryptdevice=/dev/sda2:cryptroot

{% include tip.html content="Add <code>:allow-discards</code> after <code>cryptroot</code> to allow trimming if using an
SSD. Then enable the <code>fstrim.timer</code> to trim the device weekly." %}


For a **regular** system, the kernel parameters will contain at least:

    root=/dev/sda2 ro initrd=/initramfs-linux.img init=/usr/lib/systemd/systemd

{% include note.html content="In either case, if using btrfs, and you want to boot from a specific
subvolume, add <code>rootfstype=btrfs rootflags=subvol=/@</code>, where <code>@</code> is the
subvolume you will mount as <code>/</code>." %}

{% include note.html content="If you have an Intel or AMD CPU, enable microcode updates by adding an
<code>/intel-ucode.img</code> or <code>/amd-ucode.img</code>, respectively to
<code>initrd=</code> with a comma separating the two images. It ''must'' be the first
initrd entry on the line. For example:
<code>initrd=/intel-ucode.img,/initramfs-linux.img</code>." %}

## Reboot

When you reboot you should be prompted for your LUKS password if you decided to encrypt the system.
