---
layout: default
---
I did not find any thread on how to create a fully encrypted Debian system on a DreamPlug, so now wrote my own. I found a lot of information in the <a href="https://newit.co.uk/forum/" target="_blank">New IT Forum</a> so this thread is striped down to a "short" howto with links to where you can find further informations.

Although this post is a little older, if you replace all programs with their current versions, this post still works well.


Additionally, parts marked with "*Note:*" are sideways you might take. My goal was also to keep the original system on the micro SD-Card as is, to have a fallback system or to just bring you system to another device.

You are able to do everything in this thread form within your running default DreamPlug system, so (except for the JTAG parts) it is completely independent from the remote accessing system you use.

**Usual warning:** Read through all of this thread (maybe twice) before touching you system. And of course everything at you own risk. The writer of this text guarantees for nothing...


## Prerequisites

### Required hardware

- Dreamplug
- SD-Card (external slot)
- JTAG unit


### Naming conventions

- **Internal micro SD-Card:** /dev/sda => Its first partition (vfat) /dev/sda1, its second partition (ext3) /dev/sda2
- **External SD-Card:** /dev/sdb => First partition for /boot is /dev/sdb1, second partition for the encrypted filesystem is /dev/sdb2

This is likely to be the naming on your device, too.


## Step 1 - Install you JTAG

Find a thread in the forum according to you operating system and configure your JTAG connection to you computer.

- **Mac OS X (did not work for me):** <http://www.newit.co.uk/forum/index.php/topic,2254.0.html>
- **Linux (Fedora):** <http://www.newit.co.uk/forum/index.php/topic,2064.0.html>


## Step 2 - Firmware update

You need to enable the uBoot bootloader to load from the external SD-Card. I found a solution in the following thread: <http://www.newit.co.uk/forum/index.php/topic,1977.0.html>.

You might skip this step.

- If your firmware is at least 2011-03-28 (so it can boot from the external SD-Card).
- If you place your kernel on the internal SD-Card
- Or even want to replace you internal mirco-SD-Cards system.

Your uBoots version is printed when the system boots up at you JTAG console.
I want to have everything external and do not want to touch the preinstalled system e.g. for rescueing purposes.

*Note:* If you replace the system on the internal micro SD-Card, in order to follow this tutorial, you have to take the card out of the plug and connect it to an other machine (a Debian would suite best) and to the work from there.


## Step 3 - Create the encryptet disk

I will use dm-crypt, as it is THE widely used standard for (full) disk encryption on Linux systems. As of that, i will not explain it here, just google or wikipedia it to find further information.

You have to create two partitions on you external SD-Card.

- Partition 1: 100MB (vfat) for the kernel
- Partition 2: The rest of your available space (to be encrypted ext4) for the system

If in some (hopefully near) future, the firmware can boot from ext partitions (ext2load instead of fatload), you might use ext2 instead of vfat for the first partition. The upside of this is for example, that you can symlink kernels instead of renaming or moving them.

*Note:* If you continue to use your internal SD-Cards vfat partition for the kernel in the new system, you of course use your complete external cards space ;-) In this case, your /boot partition is /dev/sda1 and your system partition is /dev/sdb1. If you use your complete internal card, keep its partitioning scheme, its /boot on /dev/sda1, system on /dev/sda2.

You should now verify cryptsetup and dosfstools are installed (and cfdisk, if you prefer it instead of fdisk). As root do:

    dd if=/dev/urandom of=/dev/sdb bs=10M # If your device is filled with random data, one can not distinguish between used and unused data on the system partition
    cfdisk /dev/sdb # Set up as said above or as you wish at your risk ;-)
    mkfs.vfat /dev/sdb1
    cryptsetup --verbose --cipher=aes-cbc-essiv:sha256 --verify-passphrase --key-size=256 luksFormat /dev/sdb2
    cryptsetup luksOpen /dev/sdb2 sdb2_crypted
    mkfs.ext4 /dev/mapper/sdb2_crypted
    mount /dev/mapper/sdb2_crypted /mnt

*Note:* I have decided to use aes-cbc-essiv with a 256-bit key (after long hours of thinking and bugging a friend of mine, sorry G) as it is Debians default and a good compromise between read/write performance and security. If you like to have a little more performance, try to use --key-size=128 or maybe 192. If you are slightly more paranoid and like to have a bit more security in exchange to a noticeable but not to high slowdown and an experimental mode try --cipher=xts-plain and --key-size=512 (note that xts needs a 256 bit key for itself, so it only uses 256-bit for the aes key, so you need to set key-size to 512).

**Remeber:** Use a good password (>= 20 characters, upper and lower case, numbers, special characters, ... you know what i mean)


## Step 4 - Bootstrap you new system

You can use debootstrap (the classic way) or multistrap (a newer approach, does the same thing completely different) to build you own brandnew minimal system. There are (of course) tutorials in this forum for that. As for this guide, skip the parts regarding the kernel installation.

- **Bootstrap method:** <http://www.newit.co.uk/forum/index.php/topic,2027.msg5773.html#msg5773> or from plugcomputer.org: <http://www.plugcomputer.org/plugwiki/index.php/Creating_a_Debian_Image>
- **Multistrap method:** Derived from <http://www.newit.co.uk/forum/index.php/topic,2073.0.html>, but as i install on another disk i do not need to reboot, instead i just chroot.

*Note:* As an easier alternative to this step, you can just copy your existing system to the encrypted partition or use a default system from the globalscale download site. If you use the bootstrap method: Stay in the chroot environment.

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

I recommend using chmod like this (in most howtos only /proc is mounted,) to have you system fully functional:

    mount -o bind /dev /mnt/dev
    mount -o bind /dev/shm /mnt/dev/shm
    mount -o bind /proc /mnt/proc
    mount -o bind /sys /mnt/sys
    # mount -o bind /run /mnt/run # On newer systems
    cp /etc/resolv.conf /mnt/etc/
    chroot /mnt

Post install you should do some basic stuff in the chroot environment:

    passwd # Set root password
    /var/lib/dpkg/info/dash.preinst install # Run unrun post-install hooks
    dpkg --configure -a # Run unrun post-install hooks
    dpkg-reconfigure locales # I would choose yourlanguage.UTF-8 and en_US.UTF-8, choose yourlanguage.UTF-8 as default
    rm /etc/apt/sources.list.d/multistrap-debian.list # Cleen multistrap's remains
    vim /etc/apt/sources.list # Create accordingly, mine see appendix D
    apt-get update && apt-get upgrade
    apt-get install cryptsetup dosfstools uboot-mkimage # Packages needed to boot
    echo "dm-crypt" >> /etc/modules # Modules needed to boot


## Step 5 - Install your kernel

Now, follow a kernel install thread of your choice. In the end you have to have a kernel and an initrd image (the latter is not included or created in many threads). Both have to be uBoot images.

### Dreamplug way

Some random kernel without initial ramdisk but with at least initramfs support built in.

You can take any kernel with no bundled initial ramdisk, wich is the default in most cases for the dreamplug. The preinstalled kernel from globalscale, the newit prerelease kernel and the "new" (aka current) kernel in the thread linked below (which is the one i used) are among those. If in doubt you have to check the kernels config(file) for initrd support (CONFIG_BLK_DEV_INITRD=y).

You will find the current user provided kernel and further information here: <http://www.newit.co.uk/forum/index.php/topic,2116.0.html>

Copy the shellscript for the dreamplug from <http://sheeva.with-linux.com/sheeva/> (and of course read the readme), choose a kernel version, and start installing (at the time of writing this document, the current kernel is 2.6.39.3, change it accordingly).

The following lines will install an allready uBoot enabled kernel image to /boot (where /dev/sdb1 should be mounted to) and the according modules to e.g. /lib/modules/2.6.39.3/.

    wget http://sheeva.with-linux.com/sheeva/README-DREAM-UPDATE.sh
    chmod +x README-DREAM-UPDATE.sh
    ./README-DREAM-UPDATE.sh 2.6.39.3

Now the most complicated part: Building you own initramfs. To do that, i followed <http://en.gentoo-wiki.com/wiki/Initramfs> (which, of all of the linked pages, you SHOULD read at least if you choose this way!). The basic steps are creating a subdirectory containing all nessecary binaries, the libraries used by that binaries and the needed kernel modules. I used the devtmpfs approach to populate my /dev folder, which worked (best) in my case.

*Note:* The used pathes might change according to system, release, and even update states, so i added how to find the needed files, too.

    mkdir /usr/src/initramfs
    cd /usr/src/initramfs
    mkdir bin  dev  etc  lib  mnt  proc  root  sbin  sys
    apt-get install busybox
    which busybox # gives you the path to the binary
    cp -aL /bin/busybox /usr/src/initramfs/bin/ # copy the binary
    ldd /bin/busybox # find all needed libraries
    cp -aL WHATEVER_LISTED_LIBRARY /usr/src/initramfs/lib/ # copy the needed libraries
    # repeat for cryptsetup, insmod and switch_root
    # cp -aL, ldd, cp -aL ...
    # remember to use the -L switch, or else you will copy symlinks as symlinks
    cp -a /lib/modules/2.6.39.3/kernel/drivers/md/dm-crypt.ko   /usr/src/initramfs/lib/modules/
    cp -a /lib/modules/2.6.39.3/kernel/drivers/md/dm-mod.ko     /usr/src/initramfs/lib/modules/
    cp -a /lib/modules/2.6.39.3/kernel/crypto/sha256_generic.ko /usr/src/initramfs/lib/modules/
    vim /usr/src/initramfs/init # The initrds startup script executed pre-boot, see appendix E
    chmod +x /usr/src/initramfs/init
    find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../my-initramfs.cpio.gz
    mkimage -A arm -O linux -T ramdisk -C gzip -a 0x00000000 -e 0x00000000 -n initramfs -d ../my-initramfs.cpio.gz /boot/manual-uInitrd

Now you should be able to boot you system with the following envvars:

    setenv mainlineLinux yes
    setenv arcNumber 2659
    usb start
    fatload usb 1 0x00800000 /dream-2.6.39.3-uImage
    fatload usb 1 0x01100000 /manual-uInitrd
    setenv bootargs console=ttyS0,115200 rootdelay=10 # You do not need a root=... parameter, as this i handled by the initramfs.
    bootm 0x00800000 0x01100000

Search the forum on how to save envvars.

### The alternative way

Debians Kernel or any other kernel providing its own initial ramdisk

*Note:* This way is not tested!

The upside of using Debians own kernel is, that you will get automated updates on the kernel and an update on the initial ramdisk (if you use a managed package like linux-image-kirkwood meta package), whereas you have to update you kernel manually in the first way and will propably never change you initial ramdisk once it works ;-) However the downside is, that you still have to convert the kernel and initramfs image to ubook format on every update and of course that it did not work in my case, god knows why. But here is how it _should_ work (in short):

Install the debian default kernel for the kirkwood platform, enable dm-crypt support and convert the resulting kernel image and initramfs to uBoot format. It is (currently) 2.6.32-5-kirkwood. The images will be installed to /boot per default, if your vfat partition is mounted there (as written above), this is not a problem, but no symlinks can be created on fat and thus you will see error messages. Furthermore you can not use these images, you have to convert them to uboot format. Maybe you might mount /dev/sdb1 to /uboot and do the conversion to this folder (as in the example below).

If you keep the name for the uImage and uInitrd, you do not have to update uBoot envvars each time.

    apt-get install linux-image-kirkwood initramfs-tools
    blkid /dev/sdb2 # Gives the UUID for sdb2
    echo "sdb2_crypted UUID=XXXXXXXXXXXXXX none luks" >> /etc/crypttab # replace the Xs according to you uuid
    vim /etc/fstab # alter "/" entry to something like: /dev/mapper/sdb2_crypted / ext4 defaults 0 0
    vim  /etc/initramfs-tools/modules # add (one line each): aes aes_generic dm-crypt dm-mod sha256 sha256_generic lrw xts crypto_blkcipher gf128mul
    update-initramfs -u -k all
    mkimage -A arm -O linux -T kernel  -C none -a 0x00008000 -e 0x00008000 -n uImage-debian  -d /boot/vmlinuz-2.6.32-5-kirkwood    /uboot/uImage-debian
    mkimage -A arm -O linux -T ramdisk -C gzip -a 0x00000000 -e 0x00000000 -n uInitrd-debian -d /boot/initrd.img-2.6.32-5-kirkwood /uboot/uInitrd-debian

The uboot envvars should be similar to the ones above with different image names. In addition, you now have to provide a "root=" parameter again, as this is not automatically handled by a fitted initramfs (

    setenv mainlineLinux yes; setenv arcNumber 2659; usb start
    fatload usb 0 0x00800000 /uImage-debian
    fatload usb 0 0x01100000 /uInitrd-debian
    setenv bootargs console=ttyS0,115200 rootdelay=10 kopt=root=/dev/mapper/sdb2_crypted ro; bootm 0x00800000 0x01100000

It should be possible to add a hook-script for kernel updates to automatically convert the image...


## (optional) Step 6 - Enable remote unlock

A big disadvantage to the now running system is, that you have to connect you JTAG everytime you boot up your system, start it up, enter the password, and disconnect JTAG again. So lets add some lines to the steps above to enable an unlocking from a remote system.

My work is based on /usr/share/doc/cryptsetup/README.remote.gz

### Dreamplug way

Some random kernel without initial ramdisk but with at least initramfs support.

Basically add dropbear to the initramfs and modify the /init script not to mount the disk, but to ifconfig eth0, route add default gw, and start dropbrear.

Just adding dropbear (and libraries) did not work for me, as it did not accept my ssh keyfile. Also, adding the lines for root from /etc/{passwd,group,shadow} did not enable me to login via my password. I think you have to add many other programms managing user access (maybe pam, etc.). Note that roots homedirectory within the busybox/initramfs is / and not /root/, so propably the keyfiles have to be in /.ssh/ not in /root/.ssh/ (but i tried placing them anywhere...).

### The alternative way

Debians Kernel or any other kernel providing its own initial ramdisk.

You will propably have initramfs-tools installed with your kernel. If you install dropbear, the system will automatically add it to you initramfs-tools config, so just updating your initramfs will do the job (update-initramfs -u -k all)


## (optional) Step 7 - Multiboot

If you followed the howto with everything done on the external SD-Card, you will be able to follow the multiboot howto in order to boot either you new, encrypted system or the default one. More infos here: <http://www.newit.co.uk/forum/index.php/topic,2231.0.html>.


## Famous last words and further thoughts

Of course you know, every chain is only as strong as its weakest link. Therefor, if you are paranoid enought to follow this howto, be as nit-picky with ervery software you are running on your system. If someone can access your system at runtime, he can of course see all your data. ... And nothing on earth beats social engineering (because there is no patch for human stupidity).

If it is OK for you if a jealous boy- or girlfriend or a guy from a three letter agency can see, which programms are actually installed on you system, you can try not to encrypt (/usr)/(s)bin directories but /etc, /home, /var, .... Of course, in this case, you can not deny that there actually IS a system on the SD-Card, but in exchange you will get rid of the ramdisk.

So, now here i am with my fully encrypted system. For myself, as i did not worked out on how to unlock remotely, I folled my own tutorial to Step 5. I did not change my uboot envvars permanently, because i need to connect the JTAG anyway to enter my password. So when i am there, my encrypted system can boot by conecting the JTAG, entering the envvars and entering the password. If not, the default system, which still resides on /dev/sda, will be booted, as if nothing had happened.

According to the documents for the 88F6281 chip found at the marvell site, there is a so called "Security Engine" on the SoC, which supports AES, DES, 3DES, SHA-1 and MD5. As far as i can see, the 88F6281 is the SoC which is used in the DreamPlug (read at the globalscale site). According to the document it has an aes128/128, aes128/192 and aes128/256 support. Nevertheless the unit seems to only have an aes128 box and will do a few more loops if you use 256-bit keys. But encryption is hardware accelerated and i experience no significant slowdown, except on heavy disk I/O. My performance test on writing speed tells me, that the encrypted volume needs around 150% time compared to the unencrypted write time. cbc-essiv a bit less, xts-plain a bit more.

The default kernel supports cbc and xts modes, and i think most of the others do, too.

- Found the info about the Soc at <http://www.globalscaletechnologies.com/c-5-dreamplugs.aspx>
- Found the document at <http://www.marvell.com/processors/embedded/kirkwood/> -> Product: 88F6281-> Document: Functional Spec (PDF: <http://www.marvell.com/processors/embedded/kirkwood/assets/FS_88F6180_9x_6281_OpenSource.pdf>)

If found out, that xts lacks a little performance and (according to some warnings) is marked experimental.

You can find the supported encryption modes at /proc/crypto, but some only show, after loading the corresponding modules. In addition, cryptosetup will load modules automatically!

Some usefull links i did not mention in the howto, but are worth reading:

- Installing Debian on the Sheevaplug: <http://blog.bofh.it/debian/id_265>
- The unlock via ssh wich i try to adapt: <http://www.howtoforge.com/unlock-a-luks-encrypted-root-partition-via-ssh-on-ubuntu>
- Commands busybox is capable of: <http://book.opensourceproject.org.cn/embedded/embeddedprime/opensource/0136130550/app03.html> (you can also type busybox --help for that list)


## Appendix A - multistrap.conf

This should provide a minimal, yet well usable, base system.

    [General]
    arch=armel
    directory=/mnt
    cleanup=false
    noauth=false
    unpack=true
    aptsources=Debian
    debootstrap=Debian

    [Debian]
    packages=apt base-files bzip2 cpio file gettext-base hostname ifupdown iputils-ping isc-dhcp-client less libertas-firmware locales man-db manpages module-init-tools netbase net-tools readline-common ssh udev vim wget whois
    source=http://ftp.de.debian.org/debian
    keyring=debian-archive-keyring
    suite=squeeze
    components=main contrib non-free


## Appendix B - fstab

This is basically the original one, slightly fine-tuned

    rootfs          /               rootfs  relatime,rw                             0 0
    udev            /dev            tmpfs   rw,mode=0755                            0 0
    tmpfs           /dev/shm        tmpfs   rw,nosuid,nodev                         0 0
    devpts          /dev/pts        devpts  rw,noexec,nosuid,gid=5,mode=620         0 0
    proc            /proc           proc    rw,noexec,nosuid,nodev                  0 0
    sysfs           /sys            sysfs   rw,noexec,nosuid,nodev                  0 0
    tmpfs           /tmp            tmpfs   rw,nosuid,nodev                         0 0
    tmpfs           /var/cache/apt  tmpfs   rw,noexec,nosuid                        0 0
    /dev/sdb1       /boot           vfat    rw                                      0 0


## Appendix C - inittab

My modifications are marked with double hashes ("##").

    # /etc/inittab: init(8) configuration.
    # $Id: inittab,v 1.91 2002/01/25 13:35:21 miquels Exp $

    # The default runlevel.
    id:2:initdefault:

    # Boot-time system configuration/initialization script.
    # This is run first except when booting in emergency (-b) mode.
    si::sysinit:/etc/init.d/rcS

    # What to do in single-user mode.
    ~~:S:wait:/sbin/sulogin

    # /etc/init.d executes the S and K scripts upon change
    # of runlevel.
    #
    # Runlevel 0 is halt.
    # Runlevel 1 is single-user.
    # Runlevels 2-5 are multi-user.
    # Runlevel 6 is reboot.

    l0:0:wait:/etc/init.d/rc 0
    l1:1:wait:/etc/init.d/rc 1
    l2:2:wait:/etc/init.d/rc 2
    l3:3:wait:/etc/init.d/rc 3
    l4:4:wait:/etc/init.d/rc 4
    l5:5:wait:/etc/init.d/rc 5
    l6:6:wait:/etc/init.d/rc 6
    # Normally not reached, but fallthrough in case of emergency.
    z6:6:respawn:/sbin/sulogin

    # What to do when CTRL-ALT-DEL is pressed.
    ca:12345:ctrlaltdel:/sbin/shutdown -t1 -a -r now

    # Action on special keypress (ALT-UpArrow).
    #kb::kbrequest:/bin/echo "Keyboard Request--edit /etc/inittab to let this work."

    # What to do when the power fails/returns.
    pf::powerwait:/etc/init.d/powerfail start
    pn::powerfailnow:/etc/init.d/powerfail now
    po::powerokwait:/etc/init.d/powerfail stop

    # /sbin/getty invocations for the runlevels.
    #
    # The "id" field MUST be the same as the last
    # characters of the device (after "tty").
    #
    # Format:
    #  <id>:<runlevels>:<action>:<process>
    #
    # Note that on most Debian systems tty7 is used by the X Window System,
    # so if you want to add more getty's go ahead but skip tty7 if you run X.
    #
    ## You do not need the usual ttys
    #1:2345:respawn:/sbin/getty 38400 tty1
    #2:23:respawn:/sbin/getty 38400 tty2
    #3:23:respawn:/sbin/getty 38400 tty3
    #4:23:respawn:/sbin/getty 38400 tty4
    #5:23:respawn:/sbin/getty 38400 tty5
    #6:23:respawn:/sbin/getty 38400 tty6

    # Example how to put a getty on a serial line (for a terminal)
    #
    #T0:23:respawn:/sbin/getty -L ttyS0 9600 vt100
    #T1:23:respawn:/sbin/getty -L ttyS1 9600 vt100

    # Example how to put a getty on a modem line.
    #
    #T3:23:respawn:/sbin/mgetty -x0 -s 57600 ttyS3

    ## Add DreamPlugs serial console for JTAG
    T0:23:respawn:/sbin/getty -L ttyS0 115200 vt100


## Appendix D - souces.list

This is just the default one with the servers containing the armel packages. But as you might install from within the Ubuntu 9.04 and the sources.list from the Globalscale Debian image is bad, her is mine:

    # Debian Squeeze
    deb http://ftp.de.debian.org/debian/ squeeze main contrib non-free
    deb http://security.debian.org/ squeeze/updates main contrib non-free
    deb http://ftp.de.debian.org/debian squeeze-updates main contrib non-free
    deb http://backports.debian.org/debian-backports squeeze-backports main


## Appendix E - init

This script is executed if you initramfs is loaded. It has the same function as /sbin/init on regular systems. It will manage everything to be done before you "real" system can boot.

    #!/bin/busybox sh

    rescue_shell() {
        echo "Something somewhere went terribly wrong. Dropping you to a shell."
        busybox --install -s
        exec /bin/sh
    }

    # Mount the /proc and /sys filesystems.
    mount -t devtmpfs none /dev
    mount -t proc none /proc
    mount -t sysfs none /sys

    # Insert kernel modules for crypto system
    insmod -f /lib/modules/dm-mod.ko
    insmod -f /lib/modules/dm-crypt.ko
    insmod -f /lib/modules/sha256_generic.ko

    # Wait for /dev to be initialised. Many thanks to Confusticated for the tip!!
    # You might try to lower the value if you like to speed up you boot process by a second or two.
    sleep 5

    # Mount the root filesystem via JTAG
    cryptsetup -T 5 luksOpen /dev/sdb2 sdb2_crypted
    mount -t ext4 -o ro /dev/mapper/sdb2_crypted /mnt/

    # Clean up. (You might get error messages by switch_root, that /mnt/dev, /mnt/proc and /mnt/sys are not mounted, but thats ok)
    umount /dev
    umount /proc
    umount /sys

    # Boot the real thing.
    exec switch_root /mnt/ /sbin/init  || rescue_shell