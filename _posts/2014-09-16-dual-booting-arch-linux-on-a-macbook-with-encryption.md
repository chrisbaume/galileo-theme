---
layout: post
title: Dual-booting Arch Linux on a Macbook with encryption
date: 2014-09-16 15:27:58.000000000 +01:00
type: post
published: true
status: publish
categories:
- Linux
tags: []
meta:
  _edit_last: '5947599'
  _publicize_pending: '1'
  _wpas_skip_facebook: '1'
  _wpas_skip_google_plus: '1'
  _wpas_skip_twitter: '1'
  _wpas_skip_linkedin: '1'
  _wpas_skip_tumblr: '1'
  _wpas_skip_path: '1'
author:
  login: chrisbaume
  email: chrisbaume@gmail.com
  display_name: chrisbaume
  first_name: ''
  last_name: ''
---
This was a tricky one to work out, but here’s how I got there with my MacbookPro 11,1. This may not be the best way,
but it’s the only one I could get working.

Firstly, use Disk Utility on OS/X to reduce it’s partition size, allowing enough free space to install Arch. Download
the [Archboot ISO](http://mirrors.manchester.m247.com/arch-linux/iso/archboot/latest/), put it on a
[USB](https://wiki.archlinux.org/index.php/USB_flash_installation_media) key and turn the Macbook on while holding down
Alt to boot from it.

Use the Archboot GUI to prepare the storage disk. Don’t let the script do it automatically; use the partitioning
program to add the following:

* EFI partition (128M, type af00)
* Boot partition (128M, type 8300)
* System partition (rest of space, type 8e00)

Exit the Archboot GUI, then follow [these
instructions](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS) up to the
mkinitcpio part. This will create a LUKS container in the system partition and install LVM partitions. Once that’s
done, reboot back into the Archboot GUI.

Set up your key mapping, date/time and network. In the storage disk preparation menu, mount the boot and encrypted
system partitions. Use Archboot to select and install all of the base, dev and support packages and configure the
system. Exit the GUI before you install the bootloader.

Use this command to log into your new system:

{% highlight shell %}
arch-chroot /install /bin/bash
{% endhighlight %}

[Configure
mkinitcpio](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Configuring_mkinitcpio) by adding
‘encrypt’ and ‘lvm2’ to the HOOKS property in /etc/mkinitcpio.conf, and run the following commands:

{% highlight shell %}
mkinitcpio -p linux-lts
mkinitcpio -p linux
{% endhighlight %}

Edit /etc/default/grub and update the following lines:

{% highlight shell %}
GRUB_CMDLINE_LINUX_DEFAULT="quiet rootflags=data=writeback libata.force=noncq"
GRUB_CMDLINE_LINUX="cryptdevice=/dev/sdaX:MyStorage root=/dev/mapper/MyStorage-rootvol"
{% endhighlight %}

To generate an EFI bootloader, use the following commands (taken from
[here](https://wiki.archlinux.org/index.php/MacBookPro11,x))

{% highlight shell %}
grub-mkconfig -o /boot/grub/grub.cfg
grub-mkstandalone -o boot.efi -d /usr/lib/grub/x86_64-efi -O x86_64-efi /boot/grub/grub.cfg
{% endhighlight %}

Insert a USB key, mount it and copy the boot.efi file.

{% highlight shell %}
mount /dev/sdX1 /mnt
cp boot.efi /mnt
{% endhighlight %}

In OS/X use Disk Utility to format (‘erase’) the 128MB EFI partition, move the boot.efi file into
System/Library/CoreServices/ and create a file in the same directory called SystemVersion.plist, with the following
contents:

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<plist version="1.0">
  <dict>
    <key>ProductBuildVersion</key>
    <string></string>
    <key>ProductName</key>
    <string>Linux</string>
    <key>ProductVersion</key>
    <string>Arch Linux</string>
  </dict>
</plist>
{% endhighlight %}

If you’re lucky, you should now be able to boot into Arch by holding down the Alt key while booting. If not, here are
some resource which should help you out. Good hunting!

* [Arch wiki – Macbook Pro 11,1](https://wiki.archlinux.org/index.php/MacBookPro11,x)
* [Arch wiki – LVM on LUKS](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS)
* [Arch wiki – beginner’s guide](https://wiki.archlinux.org/index.php/Beginners%27_guide)
