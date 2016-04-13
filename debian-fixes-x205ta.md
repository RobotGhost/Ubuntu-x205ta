= Important Note =
### Any Important note, before the user starts installing Debian
### Like Bios, Disk, MBR Configuration. (default : ''none'')
{i} Before installing Debian, Secure Boot needs to be disabled.

{i} Starting with Jessie d-i RC2, the installer includes all needed modules and core changes to install and boot on this machine. Make sure you use this version or later to install, or you'll have to fight with lots of issues and it's not likely to be fun!

{i} The X205TA is a mixed mode EFI system (i.e. a 64-bit CPU combined with a 32-bit EFI) without any legacy BIOS mode. By default, the Jessie i386 installer images should boot on this machine via UEFI and let you install a complete 32-bit (i386) system. If you use the multi-arch amd64/i386 netinst or DVD image, you will also be able to install in 64-bit mode. You might expect slightly better performance that way, but the limited memory on the machine (2 GiB) will become more of an issue.

{i} During installation, the mmcblk0rpmb device creates a lot of timeouts and makes the process  painfully slow ([[https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=759656|Bug #759656]], [[https://bugs.launchpad.net/ubuntu/+source/udev/+bug/1333140|workaround]]). Simply removing /dev/mmcblk0rpmb solves the problem, and during regular use these timeouts do not seem to be noticable.

### uncomment one of the two lines 
## none
## {X} There is currently a bug that ... 


=== System Freeze Issue ===

There is a well documented system freeze issue -- see [[https://bugs.freedesktop.org/show_bug.cgi?id=88012|Freedesktop Bug #88012]] and [[https://bugzilla.kernel.org/show_bug.cgi?id=109051|Kernel Bug #109051]]. To correct this, we can add ''intel_idle.max_cstate=1'' to the kernel boot parameters. The easiest way is to edit the file ''/etc/default/grub'' and configure this line :
{{{
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_idle.max_cstate=1"
}}}

Then you simply need to reconfigure '''grub''' :
{{{
sudo update-grub
}}}

= Configuration =

== Display ==
## First say what's the Display adapter model (lspci may help)
## Then explain how to get it working (or just the module used)
## what about : 3D ; DRI ; xrandr ; external/secondary screen...
## attach your xorg.conf at the bottom of the page.

''This is no longer necessary, starting with kernel version '''4.1'''.''

Intel Graphics using i915 driver. X.org works flawlessly.

It might be needed to manually force the brightness level, which seems to default to a very low value ('''390''' out of a maximum of '''7812'''). There are several ways to do this, but a simple solution is to add this line in /etc/sysfs.conf (make sure you have the package ''sysfsutils'' installed) :
{{{
# Set brightness level, maximum is 7812, needed for Linux Kernel < 4.1
class/backlight/intel_backlight/brightness = 5000  
}}}

## If this computer has variants with various Display adapter, simply 
## duplicate this section, like "Display / Ati".

== Audio ==
## First say what's the Sound adapter model (lspci may help)
## Then explain how to get it working

The built-in card is a '''Realtek RT5648''' (''unverified''). At this moment there isn't any driver for this card (as of Linux Kernel 4.1.3) :
 * https://bugzilla.kernel.org/show_bug.cgi?id=95681
 * http://comments.gmane.org/gmane.linux.alsa.devel/138822

It may be possible that some models of the X205TA have a different card, a '''Realtek RT5640''' (''unverified'') :
 * https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=773835

== Power Management ==
## Have you tested Hibernation and/or
## Then explain how to get it working 

Battery status information is available since kernel >= '''3.19'''. The X205TA uses some ACPI 5.0 features that are not supported in kernels < '''3.19'''. 

If you add ''relative_sleep_states=1'' to the kernel command line, suspension (a.k.a: Suspend to RAM) will work. After resume, two things won't. First of all, the touchpad won't respond. To fix that, run these commands...
{{{
sudo rmmod elan_i2c
sudo modprobe elan_i2c
}}}

Secondly, wireless does not work after resume. To permanently fix this, blacklist the ''btsdio'' module with...
~-{{{
printf "# Blacklist the btsdio module as it breaks suspend\nblacklist btsdio\n" | sudo tee /etc/modprobe.d/btsdio-blacklist.conf
}}}-~

Bluetooth does not works for now, so it's okay to blacklist ''btsdio''.

Hibernation (usually) triggers a kernel oops+panic combo when thawing the system. After that, if you reboot the machine the hard way, the kernel boots into a clean session.

== WiFi ==

On-board SDIO device is a '''Broadcom 43341''' (vendor id ''0x02d0'', device id ''0xa94d''). A kernel patch for support for the device is currently [[https://patchwork.kernel.org/patch/5534731/|under review]].

With kernel 4.0 (e.g. from [[http://kernel.ubuntu.com/~kernel-ppa/mainline/]]) wifi is working. However, firmware and nvram file need to be installed.

Firmware can be found on Google's Android Git:
{{{
wget https://android.googlesource.com/platform/hardware/broadcom/wlan/+archive/master/bcmdhd/firmware/bcm43341.tar.gz
}}}

Then we simply need to copy in in the right place (the directory ''/lib/firmware/brcm/'' might not exist so it may need to be created), with the right name :
{{{
tar xf bcm43341.tar.gz
mkdir -p /lib/firmware/brcm/
cp fw_bcm43341.bin /lib/firmware/brcm/brcmfmac43340-sdio.bin
}}}

The nvram file can be found under ''/sys/firmware/efi/efivars/''. If the directory is empty, it may need to be (temporarily) mounted first by:
{{{
mount -t efivarfs efivarfs /sys/firmware/efi/efivars
}}}

Then the nvram-File needs to be copied and renamed:
{{{
cp /sys/firmware/efi/efivars/nvram-74b00bd9-805a-4d61-b51f-43268123d113 /lib/firmware/brcm/brcmfmac43340-sdio.txt
}}}

Note, that ''brcmfmac43340-sdio.txt'' then contains a wrong MAC address. However, this is not a problem and does not need to be changed, as the file is only a template.

[[http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=9faac7b95ea4f9e83b7a914084cc81ef1632fd91|This small commit]] in '''Linux Kernel 4.4''' breaks wifi for the '''Asus X205TA''' (see [[https://bugzilla.kernel.org/show_bug.cgi?id=106541|Kernel Bug #106541]] for details). You must revert it if you want to compile a 4.4 Kernel. Note that '''this regression is fixed in Linux Kernel 4.4.4''' and reverting the patch is now unneeded.
~-{{{
wget -q http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/rawdiff/drivers/mmc/host/sdhci.c?id=9faac7b95ea4f9e83b7a914084cc81ef1632fd91 -O sdhci_commit_9faac7b95ea4f9e83b7a914084cc81ef1632fd91.diff
patch -p1 -R < sdhci_commit_9faac7b95ea4f9e83b7a914084cc81ef1632fd91.diff
rm -f sdhci_commit_9faac7b95ea4f9e83b7a914084cc81ef1632fd91.diff
}}}-~

=== Conflict between sdhci-acpi and brcmfmac ===

Due to some conflict between ''sdhci-acpi'' and ''brcmfmac'' ([[https://bugzilla.kernel.org/show_bug.cgi?id=88061]]), a parameter has to be changed for the ''sdhci-acpi'' driver. There are several ways to do this, but a quick fix is to add this line in /etc/sysfs.conf (make sure you have the package ''sysfsutils'' installed), this way the option is passed before the ''brcmfmac'' driver is loaded :
{{{
# Disable SDHCI-ACPI for Wireless, otherwise WLAN doesn't work
bus/platform/drivers/sdhci-acpi/INT33BB:00/power/control = on
}}}

== microSD Card Reader ==

Create a file ''/etc/modprobe.d/sdhci.conf'' with the following content: 
~-{{{
# Adjustment to make micro SD card reader work
options sdhci debug_quirks=0x8000
}}}-~

Then run 
{{{
update-initramfs -u -k all
}}}

After a reboot the card reader should be working.

------
= System Summary =
## The content of some files are embedded here, because they are very useful for search engines

Both the Linux kernel and GRUB have gone a long way to get support for X205TA. Originally, GRUB wasn't even able to boot. As of now, the only remaining features are sound (Realtek and Asus appear not to care about this), (all the) hotkeys, proper suspend/hibernation (apparently, the crash occurs when ''resuming'' from suspension), and Bluetooth. You can track the most recent news and experimental support in [[http://ubuntuforums.org/showthread.php?t=2254322|this thread]].

== lspci ==
## Paste the output of "lspci -nn" here, within some {{{ }}} brackets. Optionally, attach "lspci -vnn"
##(lspci -nn  missing)
~-{{{
00:00.0 Host bridge [0600]: Intel Corporation Atom Processor Z36xxx/Z37xxx Series SoC Transaction Register [8086:0f00] (rev 0f)
00:02.0 VGA compatible controller [0300]: Intel Corporation Atom Processor Z36xxx/Z37xxx Series Graphics & Display [8086:0f31] (rev 0f)
00:14.0 USB controller [0c03]: Intel Corporation Atom Processor Z36xxx/Z37xxx Series USB xHCI [8086:0f35] (rev 0f)
00:1a.0 Encryption controller [1080]: Intel Corporation Atom Processor Z36xxx/Z37xxx Series Trusted Execution Engine [8086:0f18] (rev 0f)
00:1f.0 ISA bridge [0601]: Intel Corporation Atom Processor Z36xxx/Z37xxx Series Power Control Unit [8086:0f1c] (rev 0f)
}}}-~

== lsusb ==
## Uncomment and Paste the output of one of the three lines (within some {{{ }}}} brackets) 
## (lsusb missing)
## lsusb
## lsusb -v | grep -E '\<(Bus|iProduct|bDeviceClass|bDeviceProtocol)' 2>/dev/null 
{{{
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 003: ID 05e3:0610 Genesys Logic, Inc. 4-port hub
Bus 001 Device 002: ID 04f2:b483 Chicony Electronics Co., Ltd 
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
}}}
