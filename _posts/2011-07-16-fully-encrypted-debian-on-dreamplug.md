---
title: Fully encrypted Debian on DreamPlug
layout: default
---
# Intention: A fully encrypted up-to-date Debian on a dm-crypt encrypted external SD-Card

Hi Guys,

i did not find any thread on how to create a fully encrypted system on a DreamPlug, so now wrote my own here. I am not a real pro, but i did what i could. I found a lot of information so this thread is striped down to a "short" howto with links to where you can find further informations. Additionally, parts marked with "*Note:*" are sideways you might take. My goal was also to keep the original system on the micro SD-Card as is, to have a fallback system or to just bring you system to another device.

You are able to do everything in this thread form within your running default DreamPlug system, so (except for the JTAG parts) it is completely independent from the remote accessing system you use.

**Usual warning:** Read through all of this thread (maybe twice) before touching you system! And of course everything at you own risk. The writer of this text guarantees for nothing...

You're welcome to post on any mistakes i made, misspelling, shortcuts other users may take or whatever you like to say. I hope this thread can evolve to a simple all-in-one howto/solution for others.


# Required hardware


- Dreamplug
- SD-Card (external slot)
- JTAG unit



# Naming conventions

Internal micro SD-Card: /dev/sda => Its first partition (vfat) /dev/sdb1, its second partition (ext3) /dev/sda2
External SD-Card: /dev/sdb => I will create a similar partition sceme => First partition for /boot is /dev/sdb1, second partition for the encrypted filesystem is /dev/sdb2
This is likely to be the naming on your device, too.


**As this has grown to a huge howto, here a link to skip to the discussion below: http://www.newit.co.uk/forum/index.php/topic,2308.msg6517.html#msg6517**


# Step 1 - Install you JTAG

Find a thread in this forum according to you operating system and configure your JTAG connection to you computer.
Mac OS X (did not work for me): http://www.newit.co.uk/forum/index.php/topic,2254.0.html
Linux (Fedora): http://www.newit.co.uk/forum/index.php/topic,2064.0.html



# Step 2 - Firmware update

You need to enable the uBoot bootloader to load from the external SD-Card. I found a solution in the following thread in this forum: http://www.newit.co.uk/forum/index.php/topic,1977.0.html
You might skip this step.
a) If your firmware is at least 2011-03-28 (so it can boot from the external SD-Card).
b) If you place your kernel on the internal SD-Card
c) Or even want to replace you internal mirco-SD-Cards system.
Your uBoots version is printed when the system boots up at you JTAG console.
I want to have everything external and do not want to touch the preinstalled system e.g. for rescueing purposes.

*Note:* If you replace the system on the internal micro SD-Card, in order to follow this tutorial, you have to take the card out of the plug and connect it to an other machine (a Debian would suite best) and to the work from there.


# Step 3 - Create the encryptet disk

I will use dm-crypt, as it is THE widely used standard for (full) disk encryption on Linux systems. As of that, i will not explain it here, just google or wikipedia it to find further information.
You have to create two partitions on you external SD-Card. Partition 1: 100MB (vfat) for the kernel. Partition 2: The rest of your available space (to be encrypted ext4) for the system. If in some (hopefully near) future, the firmware can boot from ext partitions (ext2load instead of fatload), you might use ext2 instead of vfat for the first partition. The upside of this is for example, that you can symlink kernels instead of renaming or moving them.

*Note:* If you continue to use your internal SD-Cards vfat partition for the kernel in the new system, you of course use your complete external cards space ;-) In this case, your /boot partition is /dev/sda1 and your system partition is /dev/sdb1. If you use your complete internal card, keep its partitioning scheme, its /boot on /dev/sda1, system on /dev/sda2.

You should now verify cryptsetup and dosfstools are installed (and cfdisk, if you prefer it instead of fdisk). As root do:

```shell
dd if=/dev/urandom of=/dev/sdb bs=10M # If your device is filled with random data, one can not distinguish between used and unused data on the system partition
cfdisk /dev/sdb # Set up as said above or as you wish at your risk ;-)
mkfs.vfat /dev/sdb1
cryptsetup --verbose --cipher=aes-cbc-essiv:sha256 --verify-passphrase --key-size=256 luksFormat /dev/sdb2
cryptsetup luksOpen /dev/sdb2 sdb2_crypted
mkfs.ext4 /dev/mapper/sdb2_crypted
mount /dev/mapper/sdb2_crypted /mnt
```

*Note:* I have decided to use aes-cbc-essiv with a 256-bit key (after long hours of thinking and bugging a friend of mine, sorry G) as it is Debians default and a good compromise between read/write performance and security. If you like to have a little more performance, try to use --key-size=128 or maybe 192. If you are slightly more paranoid and like to have a bit more security in exchange to a noticeable but not to high slowdown and an experimental mode try --cipher=xts-plain and --key-size=512 (note that xts needs a 256 bit key for itself, so it only uses 256-bit for the aes key, so you need to set key-size to 512).

**Remeber:** Use a good password (>= 20 characters, upper and lower case, numbers, special characters, ... you know what i mean)



# Step 4 - Bootstrap you new system

You can use debootstrap (the classic way) or multistrap (a newer approach, does the same thing completely different) to build you own brandnew minimal system. There are (of course) tutorials in this forum for that. As for this guide, skip the parts regarding the kernel installation.
Bootstrap method: http://www.newit.co.uk/forum/index.php/topic,2027.msg5773.html#msg5773 or from plugcomputer.org: http://www.plugcomputer.org/plugwiki/index.php/Creating_a_Debian_Image
My multistrap method is derived from http://www.newit.co.uk/forum/index.php/topic,2073.0.html as i install on another disk, i do not need to reboot. Instead i just chroot.

*Note:* As an easier alternative to this step, you can just copy your existing system to the encrypted partition or use a default system from the globalscale download site. If you use the bootstrap method: Stay in the chroot environment.

```shell
multistrap -f multistrap.conf # For my config see appendix A
mount /dev/sdb1 /mnt/boot/
mknod /mnt/dev/console c 5 1
mknod /mnt/dev/random c 1 8
mknod /mnt/dev/urandom c 1 9
mknod /mnt/dev/null c 1 3
mknod /mnt/dev/ptmx c 5 2
vim /etc/fstab # For my fstab see appendix B
touch /mnt/etc/mtab
echo "my-dreamplugs-hostname" > /mnt/etc/hostname # To use your old you can do "cat /etc/hostname > /mnt/etc/hostname"
vim /mnt/etc/network/interfaces # you can propably use you old config via "cat /etc/network/interfaces > /mnt/etc/network/interfaces"
touch /mnt/etc/udev/rules.d/75-persistent-net-generator.rules
vim /mnt/etc/inittab # Basically add the JTAG console and comment out tty1-6, config see appendix C
echo "vm.mmap_min_addr=32768" >> /mnt/etc/sysctl.d/local.conf # Was recommended, so what ;-)
```

I recommend using chmod like this (in most howtos only mounting proc is used) to have you system fully functional:

```shell
mount -o bind /dev /mnt/dev
mount -o bind /dev/shm /mnt/dev/shm
mount -o bind /proc /mnt/proc
mount -o bind /sys /mnt/sys
cp /etc/resolv.conf /mnt/etc/
chroot /mnt
```

Post install you should do some basic stuff in the chroot environment:

```shell
passwd
/var/lib/dpkg/info/dash.preinst install
dpkg --configure -a
dpkg-reconfigure locales # I would choose all of yourlanguage* and en_US*, choose yourlanguage.UTF-8 as default
rm /etc/apt/sources.list.d/multistrap-debian.list
vim /etc/apt/sources.list # Create accordingly, mine see appendix D
apt-get update && apt-get upgrade
apt-get install cryptsetup dosfstools uboot-mkimage
echo "dm-crypt" >> /etc/modules
```



# Step 5 - Install your kernel

Now, follow a kernel install thread of your choice. In the end you have to have a kernel and an initrd image (the latter is not default in most cases). Both have to be uBoot images.

**Dreamplug way: Some random kernel without initial ramdisk but with at least initramfs support**

You can take any kernel with no bundled initial ramdisk, wich is the default in most cases for the dreamplug. The preinstalled kernel from globalscale, the newit prerelease kernel and the "new" (aka current) kernel in the thread linked below (which is the one i used) are among those. If in doubt you have to check the kernels config(file) for CONFIG_BLK_DEV_INITRD=y.
**@everyone:** Would someone please check/report if this is the correct flag?
You will find the current user provided kernel and further information here: http://www.newit.co.uk/forum/index.php/topic,2116.0.html
Copy the shellscript for the dreamplug from http://sheeva.with-linux.com/sheeva/ (and of course read the readme), choose a kernel version, and start installing (at the time of writing this document, the current kernel is 2.6.39.3, change it accordingly).

The following lines will install an allready uBoot enabled kernel image to /boot (where /dev/sdb1 should be mounted to) and the according modules to e.g. /lib/modules/2.6.39.3/.

```shell
wget http://sheeva.with-linux.com/sheeva/README-DREAM-UPDATE.sh
chmod +x README-DREAM-UPDATE.sh
./README-DREAM-UPDATE.sh 2.6.39.3
```

Now the most complicated part: Building you own initramfs. To do that, i followed http://en.gentoo-wiki.com/wiki/Initramfs (which, of all of the linked pages, you SHOULD read at least!). The basic steps are creating a subdirectory containing all nessecary binaries, the libraries used by that binaries and the needed kernel modules. I used the devtmpfs approach to populate my /dev folder, which worked (best) in my case.

*Note:* The used pathes might change according to system, release and even update states, so i added how to find the needed files, too.

```shell
mkdir /usr/src/initramfs
cd /usr/src/initramfs
mkdir bin  dev  etc  lib  mnt  proc  root  sbin  sys
apt-get install busybox
which busybox # gives you the path to the binary
cp -aL /bin/busybox /usr/src/initramfs/bin/ # copy the binary
ldd /bin/busybox # find all needed libraries
cp -aL WHATEVER_LISTED_LIBRARY /usr/src/initramfs/lib/ # copy the needed libraries
d # repeat for cryptsetup, insmod and switch_root
d # cp -aL, ldd, cp -aL ...
d # remember to use the -L switch, or else you will copy symlinks as symlinks
cp -a /lib/modules/2.6.39.3/kernel/drivers/md/dm-crypt.ko   /usr/src/initramfs/lib/modules/
cp -a /lib/modules/2.6.39.3/kernel/drivers/md/dm-mod.ko     /usr/src/initramfs/lib/modules/
cp -a /lib/modules/2.6.39.3/kernel/crypto/sha256_generic.ko /usr/src/initramfs/lib/modules/
vim /usr/src/initramfs/init # The initrds script executed pre-boot, see Appendix E
chmod +x /usr/src/initramfs/init
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../my-initramfs.cpio.gz
mkimage -A arm -O linux -T ramdisk -C gzip -a 0x00000000 -e 0x00000000 -n initramfs -d ../my-initramfs.cpio.gz /boot/manual-uInitrd
```