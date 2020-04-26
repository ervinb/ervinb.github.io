---
layout: article
title: Fixing Fedora black screen on boot (AMDGPU)
tags: linux tips fedora amdgpu
key: fedora-black-screen-amdgpu
---

Having your OS hang on you with a black screen, is not the best start of a day.
The main suspects are usually a failing mount in `/etc/fstab`, the GPU driver,
a kernel upgrade or all three.
In my case, the kernel was upgraded and it wasn't booting. Starting the previous
one did work however.

To quickly weed out the GPU, you can try booting with `nomodeset`. While on the GRUB
screen, press 'e' (for edit) and add `nomodeset` to the boot options.
This disables KMS ([Kernel mode setting](https://fedoraproject.org/wiki/Features/KernelModesetting#Summary)),
which moved the loading of the GPU driver from user space to kernel space. This is why it's stuck at boot when the
driver doesn't load properly.

Having `nomodeset` in the boot options will prevent the `amdgpu` kernel module from
being loaded and hopefully you can boot into the OS to find the root cause.

By being in the OS now (rescue mode works too), we can investigate what went wrong,
by looking into the *previous* boot log.

```
$ sudo journalctl --system --list-boot
...
-12 db5d180a75ea46b98bd5abfd6ddf2de3
-11 36ba4e53a6864702bdc72b243fb64a80
-10 964a1285dd5545b699bae81282077fa3 # << offending boot!
-9 932f400f4d1c4078b8c972cb841a1c57
-8 8d0abb5acb284d7d90b7ff5814c63389
-7 444fe6eb8ebe43e98373523414a9be05
...

# show the previous boot by providing its ID
$ sudo journalctl --boot 964a1285dd5545b699bae81282077fa3
...
kernel: amdgpu 0000:03:00: Direct firmware load for amdgpu/navi10_gpu_info.bin failed with error -2
kernel: amdgpu 0000:03:00: Failed to load gpu_info firmware "amdgpu/navi10_gpu_info.bin"
kernel: amdgpu 0000:03:00: Fatal error during GPU init
kernel: [drm] amdgpu: finishing device.
kernel: ------------[ cut here  ]------------
kernel: sysfs group 'fw_version' not found for kobject '0000:03:00.0'
kernel: WARNING: CPU: 1 PID: 422 at fs/sysfs/group.c:278 sysfs_remove_group+0x74/0x80
kernel: Modules linked in: amdgpu(+) amd_iommu_v2 gpu_sched i2c_algo_bit ttm crc32c_intel drm_kms_helper e1000e(+) n>
kernel: CPU: 1 PID: 422 Comm: systemd-udevd Not tainted 5.5.10-100.fc30.x86_64 #1
kernel: Hardware name: To Be Filled By O.E.M. To Be Filled By O.E.M./Z270 Extreme4, BIOS P1.20 11/03/2016
kernel: RIP: 0010:sysfs_remove_group+0x74/0x80
kernel: Code: ff 5b 48 89 ef 5d 41 5c e9 39 be ff ff 48 89 ef e8 c1 b9 ff ff eb cc 49 8b 14 24 48 8b 33 48 c7 c7 e8 >
kernel: RSP: 0018:ffffb941003d7a20 EFLAGS: 00010282
kernel: RAX: 0000000000000000 RBX: ffffffffc0884a00 RCX: 0000000000000007
kernel: RDX: 0000000000000007 RSI: 0000000000000092 RDI: ffff91f15ec99cc0
kernel: RBP: 0000000000000000 R08: 00000000000003c9 R09: 0000000000000003
kernel: R10: 0000000000000000 R11: 0000000000000001 R12: ffff91f1599020b0
kernel: R13: ffff91f150b34d98 R14: ffff91f159da5bc0 R15: 0000000000000000
kernel: FS:  00007f63a53f9940(0000) GS:ffff91f15ec80000(0000) knlGS:0000000000000000
kernel: CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
kernel: CR2: 000055c153964000 CR3: 0000000452912004 CR4: 00000000003606e0
kernel: DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
kernel: DR3: 0000000000000000 DR6: 00000000fffe0ff0 DR7: 0000000000000400
kernel: Call Trace:
kernel:  amdgpu_device_fini+0x43d/0x471 [amdgpu]
kernel:  amdgpu_driver_unload_kms+0x4a/0x90 [amdgpu]
kernel:  amdgpu_driver_load_kms.cold+0x39/0x5b [amdgpu]
...
```

Zooming in:
```
Direct firmware load for amdgpu/navi10_gpu_info.bin failed with error -2
```

The [AMDGPU](https://wiki.archlinux.org/index.php/AMDGPU#Loading) is not loading after the kernel upgrade.

Naturally, the first step is to see if the `amdgpu/navi10_gpu_info.bin` file is there.
In my case it was there, but if it's missing for you, [download it](https://askubuntu.com/a/1124256).

As you probably already now by all the purple links in your Google search results,
this error can be caused by almost anything. However, before dwelling into re-installing
the driver and whatnot, give this simple solution a go.

## Solution

After digging around a lot and searching for everything with `amd` in its name,
I've found that there are some `amdgpu-pro` related files in the `/etc/dracut.conf.d/` directory.

[Dracut](https://fedoraproject.org/wiki/Dracut) is generating the initial ramdisk
which boots the system (more on this), so it looked fishy. One of the files had
the previous kernel version in its name. The natural instinct in this case is
remove it into oblivion.

After restarting, sure enough, the new kernel was able to boot and the `amdgpu`
firmware loaded properly. At one point I did install the `amdgpu-pro` driver, but
removed it soon after. `amdgpu-uninstall` didn't do a proper job in cleaning up
after itself.

After the next kernel update, this happened again. The `amdgpu-pro` driver was
long removed but there were some remnants still.

Putting together `kernel upgrade` + `new device related file appearing` results
in [DKMS](https://wiki.archlinux.org/index.php/Dynamic_Kernel_Module_Support).
DKMS rebuilds third-party kernel modules with each new version:

> Dynamic Kernel Module Support (DKMS) is a program/framework that enables generating Linux kernel modules whose sources generally reside outside the kernel source tree. The concept is to have DKMS modules automatically rebuilt when a new kernel is installed.
> This means that a user does not have to wait for a company, project, or package maintainer to release a new version of the module.

Checking DKMS status confirms this:

```
$ dkms status
amdgpu-pro, 16.50-362463.el7: added
```

Searching the installed packages revealed:

```
$ dnf list installed | grep amd
amdgpu-pro-dkms.noarch                             16.50-362463.el7                     @amdgpu-pro-local

$ sudo dnf remove amdgpu-pro-dkms.noarch
```

Uninstalling the `amdgpu-pro-dkms.noarch` package will not remove the generated files,
it has to be don manually:
```
$ sudo rm -rf /etc/dracut.conf.d/amdgpu*
```

Lastly, the current initramfs is regenerated to exclude the old module.

```
$ sudo dracut --force
```

From this point on, every new kernel worked out of the box.
