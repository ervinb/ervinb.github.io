---
layout: post
title: Fixing IOMMU issues on Ubuntu 14.04 Trusty
published: true
tags: iommu linux ubuntu trusty troubleshooting amd
---

Installation of Ubuntu 14.04 (or any other distro really) on a configuration
with an AMD 970 + SB950 chipset is not as straightforward as you may think. In my particular case, the
motherboard is a [Gigabyte GA-970A-DS3](http://www.gigabyte.com/products/product-page.aspx?pid=4122).

I will describe some workarounds I've found, while trying to make the penguins
happy.

## Tieing the shoolaces

The first issue arose when I created a bootable Ubuntu 14.04.1 USB with
[Universal USB Installer](http://www.pendrivelinux.com/universal-usb-installer-easy-as-1-2-3/).
Once the computer was at the Ubuntu loading screen, I noticed strange HDD
activity. The activity light was blinking in suspiciously regular periods.
Not much after this, I was greeted by a lovely message:

```
(initramfs) unable to find a live medium containing a live file system
```

My first thought was that the OS image on the USB stick was corrupted. I tried
it in another computer and it was working fine. I had some issues with the
IOMMU controller + Linux before, so I decided to see what's the current setup of
the BIOS. IOMMU was turned off. I turned it back on and gave the USB boot
another shot. It worked! But not for long.

## Hobbling to victory

Before the installation splash screen, a charming `AMD-Vi: Completion-Wait loop timed out`
showed up. After about 10 seconds, it continued with the startup process
and everything seemed to be working fine. The installation process had begun, which
of course, froze at 29%. Let's check out some IOMMU kernel flags.

I've seen people use `pt` (passthrough) in combination with `1` and `soft`. These
effectively bypass the physical device. I had most success with the `soft` variant.

After a restart, right after the USB starts to boot, pressing any key (I prefer
mashing "Space") enters the Ubuntu installation menu. Scroll to the installation
entry and press "F6". This shows some additional installation options which can be
ignored in this instance. What can't be ignored, is the input line which shows
up after this.
Append `iommu=soft` here and press "Enter". With that in place, the installation
should succeed. Note that this option can also be added when the bootable USB is
created but first you should be sure which options works for you.

Before you open a champaign, there's possibly one more lemon to chew. After the first
restart, you may encounter something funny. Based on my previous experiences
with IOMMU error messages,
`AMD-Vi: Event logged [IO_PAGE_FAULT]` isn't a good thing to happen at the first
boot of a freshly installed OS. Your EXT4 partition(s) used by the new Ubuntu
installation can become corrupted. Actually, I found it to be a rule. To work
around this, you have to edit the GRUB menu entry for Ubuntu. When you
reboot the machine after the OS installation completes, boot from the hard disk
which has Ubuntu installed on it. The GRUB menu will be shown with a countdown.
Pressing "E" on the `Ubuntu` menu item will enter edit mode, where `iommu=soft`
should be added to the end of the line which begins with `linux`. After this
the OS should boot up fine.

This is a fun thing to do, but if you want to make the change permanent,
you'll have the edit GRUB's configuration file. Run `sudo vi /etc/default/grub`
and change the option shown below to include `iommu=soft`:

```
GRUB_LINUX_CMD_LINE="iommu=soft"
```

To let GRUB know about this change for future boots, execute `sudo update-grub`.
Keep in mind that IOMMU is still enabled in the BIOS.

There are also some [alternative solutions](https://ubuntuforums.org/showthread.php?t=2254677),
which you might try.

You should also try this approach if you're having freezing issues with Ubuntu.

Now let's open that champaign!

## Update (03-01-2017)

I was never fully satisfied with the solution above, and with each new kernel release,
a new spark of hope arose. I found that other distros are having the exact same
problems like Ubuntu Trusty.

A particular [diff in kernel 4.8](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=dd9671172a06830071c8edb31fb2176f222a2c6e)
was really promising. Once I've came around to try it, [Fedora 25](https://fedoramagazine.org/fedora-25-released/) was released,
running kernel version 4.8, so I decided to give it a shot.

In short, this is probably the best distribution I've used. Everything is so
streamlined and polished.

And the best part is, the freezing issues are comple


I'm kidding (but not completely)! The boot log still shows an issue with `AMD-Vi`, but apparently,
the new kernel worked around the problem which caused the freezes. The installation
actually froze at the very last second. Luckily, this happened after everything
was set up. The OS booted and I ran an update, after which I rebooted the system.

To knock on the whole Amazon rainforest, everything is working as it should.
