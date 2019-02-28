---
layout: post
title: NixOS on a Dell XPS15 9560
tags: [ nixos, sysadmin, luks ]
---

## Foreword

From a several months now, I have been using NixOS, as much for personal stuff than for work.

In particular, its declarative and reproducible system configuration allows me to have a GitHub repository, [Pamplemousse/laptop](https://github.com/pamplemousse/laptop), "representing" what the software setup of my machine is.

The idea with this is being able to automate the installation of my computer, easing the pain of installing and setting up a new machine.

I am not at all an expert in system administration, and this "project" is far from being mature, but this had done the job so far.


## Until...

One day, I received a Dell XPS15 9560 which I needed to setup, thus I wanted to install NixOS on it.

It was painful ; but I realized I was not the only one running into trouble installing Linux on this machine: [see this comparative result on the installation of several distributions](https://medium.com/@kemra102/linux-on-the-dell-xps-15-919e6d472aa3).

Among the bit that caused me much trouble:
  * Move from `MBR/BIOS` to `GPT/UEFI`<sup id="a1">[1](#f1)</sup>;
  * LUKS encryption;
  * Machine freezing, very likely due to the [nouveau](https://nouveau.freedesktop.org/wiki/), and [bbswitch](https://github.com/Bumblebee-Project/bbswitch) modules for Nvidia graphic card.

Miraculously, I discovered two blog posts that literally saved my day (or shall I say my week):

  * [NixOS on a Dell 9560](http://grahamc.com/blog/nixos-on-dell-9560) (by Graham Christensen)
  * [Installing NixOS on a XPS 9560](https://jtanguy.cleverapps.io/installing-nixos-on-a-xps-9560/) (by Julien Tanguy)

Thus, this post stands on their shoulders and comes essentially as a wrap up of the work they have provided.


## Full disk encryption

<!-- TODO -->
> I don't think we can make these devices harder to lose; that's a human problem and not a technological one. But we can make the loss just cost money, not privacy.
<sup id="a2">[2](#f2)</sup>

Solution: use [Luks](https://guardianproject.info/code/luks/) to encrypt partitions ; Here is what the partitioning looks like.

<img alt="partitioning of the disk" src="/assets/images/2019-02-28-NixOS%20on%20a%20Dell%20XPS15%209560/partitioning.png">

  1. `/dev/sda1`: BIOS
  2. `/dev/sda2`: EFI
  3. `/dev/sda3`, `/dev/mapper/cryptkey`: LUKS key
  4. `/dev/sda4`, `/dev/mapper/cryptswap`: swap partition
  5. `/dev/sda5`, `/dev/mapper/cryptroot`: root filesystem

#### Why this partitioning?

We want the swap partition to be encrypted not to leak the RAM content on hibernation.

For user-friendliness, we create a partition `/dev/mapper/cryptkey` that will be used as a keyfile to unlock both the swap (`/dev/mapper/cryptswap`) and the root (`/dev/mapper/cryptroot`) partitions.
This keyfile will then be encrypted using a user passphrase.

Hence, a passphrase will be asked only once, to decrypt the keyfile, which will then be used to decrypt the swap and root partitions.

#### And if `/dev/mapper/cryptkey` gets corrupted?

As is, that would mean that **the swap and root partitions would be lost**.
For the latter one, that is very bad: all data on the root partition (system and user data) would then be inaccessible.

One solution is to create a random passphrase (not meant to be remembered by yourself, that you store securely store elsewhere), and then allow it to decrypt the root partition.

#### Here is how we proceeded:

*Note that some space is left at the beginning of the disk for the GPT to take place.*<sup id="a3">[3](#f3)</sup>

```bash
# partitioning
DISK=/dev/sda
sgdisk -og "$DISK"
sgdisk -n 1:2048:4095 -c 1:"BIOS boot partition" -t 1:ef02 "$DISK"
sgdisk -n 2:0:+550MiB -c 2:"EFI system partition" -t 2:ef00 "$DISK"
sgdisk -n 3:0:+3MiB -c 3:"cryptsetup luks key" -t 3:8300 "$DISK"
sgdisk -n 4:0:+"${RAM}"GiB -c 4:"swap space (hibernation)" -t 4:8300 "$DISK"
sgdisk -n 5:0:"$(sgdisk -E "$DISK")" -c 5:"root filesystem" -t 5:8300 "$DISK"

# encrypting
cryptsetup luksFormat "${DISK}3"
cryptsetup luksOpen "${DISK}3" cryptkey

dd if=/dev/random of=/dev/mapper/cryptkey bs=1024 count=14000

cryptsetup luksFormat --key-file=/dev/mapper/cryptkey "${DISK}4"

cryptsetup luksFormat "${DISK}5"
cryptsetup luksAddKey "${DISK}5" /dev/mapper/cryptkey

# labeling, mounting and generating the base config
cryptsetup luksOpen --key-file=/dev/mapper/cryptkey "${DISK}4" cryptswap
mkswap -L swap /dev/mapper/cryptswap
swapon /dev/disk/by-label/swap

cryptsetup luksOpen --key-file=/dev/mapper/cryptkey "${DISK}5" cryptroot
mkfs.ext4 -L nixos /dev/mapper/cryptroot
mount /dev/disk/by-label/nixos /mnt

mkfs.vfat -n efi "${DISK}2"
mkdir /mnt/boot
mount /dev/disk/by-label/efi /mnt/boot

nixos-generate-config --root /mnt
```

#### And what the NixOS configuration looks like:

We created an extra file called `/etc/nixos/luks-devices-configuration.nix`, containing the following:
```
{
  boot.initrd.luks.devices = {
    cryptkey = {
      device = "/dev/sda3";
    };

    cryptroot = {
      device = "/dev/sda5";
      keyFile = "/dev/mapper/cryptkey";
    };

    cryptswap = {
      device = "/dev/sda4";
      keyFile = "/dev/mapper/cryptkey";
    };
  };
}
```

And then, included it in the general `/etc/nixos/configuration.nix` file:
```
{ config, pkgs, ... }:

{
  imports =
    [ # Include the results of the hardware scan.
      ./hardware-configuration.nix
      ./luks-devices-configuration.nix
    ];

  # ...
}
```

## Machine freezing

As mentioned earlier in the post, I experienced many freezes of the laptop, and adopted the solution proposed in [jtanguy.cleverapps.io/installing-nixos-on-a-xps-9560](https://jtanguy.cleverapps.io/installing-nixos-on-a-xps-9560/#graphics) by adding the following to my `/etc/nixos/configuration.nix`:

```
boot.blacklistedKernelModules = [ "nouveau" "bbswitch" ];
boot.extraModulePackages = [ pkgs.linuxPackages.nvidia_x11 ];

hardware.bumblebee.enable = true;
hardware.bumblebee.pmMethod = "none";
```


## Conclusion

My GitHub repository [Pamplemousse/laptop](https://github.com/Pamplemousse/laptop) should contain the most up-to-date state of my configuration.

However, it does not concern specifically the Dell XPS15 9560, and not all that I have presented here is merged into the master branch (in particular the Kernel modules blacklisting or the `bumblebee` configuration).
Despite, missing pieces can be found in the [test branch](https://github.com/Pamplemousse/laptop/tree/test) of the same repository.

#### Warning / Need to be improved

**It is worth nothing as I do not run these script as is.**

So far, my work on this repository is actually more about having handful "templates" and/or bits of configuration to speed-up my laptop's installation rather than having "production ready" autonomous scripts that have been thoroughly tested.

Some areas of improvements that are worth mentioning:

  * Install script, pay attention to what is generated in `/etc/nixos/hardware-configuration.nix` and what might overwrite stuff from the `/etc/nixos/luks-devices-configuration.nix` during the configuration generation;
  * `/boot` being located on `/dev/sda2` is not encrypted.


**Aside from that, I am happy now that my laptop is functional! (Pun intended.)**

---
<b id="f1">1</b> MBR, BIOS, GPT, UEFI definitions: <a href='https://wiki.manjaro.org/index.php?title=Some_basics_of_MBR_v/s_GPT_and_BIOS_v/s_UEFI' target='blank'>https://wiki.manjaro.org/index.php?title=Some_basics_of_MBR_v/s_GPT_and_BIOS_v/s_UEFI</a> . [↩](#a1)

<b id="f2">2</b> Source: <a href='https://www.wired.com/2006/01/big-risks-come-in-small-packages/' target='blank'>https://www.wired.com/2006/01/big-risks-come-in-small-packages/</a> . [↩](#a2)

<b id="f3">3</b> GUID Partition Table: <a href='https://en.wikipedia.org/wiki/GUID_Partition_Table' target='blank'>https://en.wikipedia.org/wiki/GUID_Partition_Table</a> . [↩](#a3)
