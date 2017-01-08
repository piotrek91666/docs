1. Toolchain:
-------------

    wget https://releases.linaro.org/components/toolchain/binaries/latest/arm-linux-gnueabihf/gcc-linaro-6.2.1-2016.11-x86_64_arm-linux-gnueabihf.tar.xz
    tar xvf gcc-linaro-6.2.1-2016.11-x86_64_arm-linux-gnueabihf.tar.xz -C ~/tools/
    export PATH=~/tools/gcc-linaro-6.2.1-2016.11-x86_64_arm-linux-gnueabihf/bin/:$PATH


2. Kernel:
----------

    git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git
    cd linux-stable/
    git tag
    git checkout v4.9
    nano Makefile
    make CROSS_COMPILE=arm-linux-gnueabihf- clean

    # Need for missing sun8i-emac driver in kernel 4.9
    wget https://patchwork.kernel.org/patch/9365783/raw/ -O sun8i-emac-patch-1.patch
    wget https://patchwork.kernel.org/patch/9365773/raw/ -O sun8i-emac-patch-4.patch
    wget https://patchwork.kernel.org/patch/9365757/raw/ -O sun8i-emac-patch-5.patch
    wget https://patchwork.kernel.org/patch/9365767/raw/ -O sun8i-emac-patch-7.patch
    wget https://patchwork.kernel.org/patch/9365779/raw/ -O sun8i-emac-patch-9.patch
    for patch in 1 4 5 7 9 ; do patch -p1 < sun8i-emac-patch-${patch}.patch; done

    ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make sunxi_defconfig
    ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make menuconfig


### See:
- https://wiki.gentoo.org/wiki/Kernel/Gentoo_Kernel_Configuration_Guide
- iptables: https://wiki.gentoo.org/wiki/Iptables
- LVM: https://wiki.gentoo.org/wiki/LVM
- DRBD: https://wiki.gentoo.org/wiki/DRBD
- USB: https://wiki.gentoo.org/wiki/USB/Guide
- Make drbd,sit,can,usblp,ftdi_sio,pl2303 as module
- Add autofs4 module
- Unselect "Multimedia support","Sound card support"


    ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make -j5 zImage dtbs
    ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=out/ make modules modules_install

Will output:

- out/lib/modules/**/VERSION/**
- arch/arm/boot/**zImage**
- arch/arm/boot/dts/**YOUR_BOARD.dtb**

#### Make Debian packages (optional):

    make -j2 deb-pkg KDEB_PKGVERSION=YOUR_VERSION LOCALVERSION=-lime KBUILD_DEBARCH=armhf ARCH=arm 'DEBFULLNAME=YOUR_USERNAME' DEBEMAIL=YOUR_EMAIL CROSS_COMPILE=arm-linux-gnueabihf-


3. U-Boot:
----------

    git clone git://git.denx.de/u-boot.git
    cd u-boot/
    git tag
    git checkout v2016.11
    ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make orangepi_one_defconfig
    ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make menuconfig # Optional
    ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make -j5

Will output:
- **u-boot-sunxi-with-spl.bin**


4. Filesystem:
--------------

    dd if=/dev/zero of=/dev/sdb bs=1M count=8
    fdisk /dev/sdb


- Type o. This will clear out any partitions on the drive.
- Type p to list partitions. There should be no partitions left.
- Now type n, then p for primary, 1 for the first partition on the drive, 2048 for the first sector, and then press ENTER to accept the default last sector.
- Write the partition table and exit by typing w.


    mkfs.ext4 /dev/sdb1 # For e2fsprogs < 1.43
    mkfs.ext4 -O ^metadata_csum,^64bit /dev/sdX1 # For e2fsprogs >= 1.43

    dd if=u-boot-sunxi-with-spl.bin of=/dev/sdb bs=1024 seek=8


### Debian:

    mount /dev/sdb1 /mnt/
    debootstrap --include=openssh-server,debconf-utils,dbus --arch=armhf --foreign stretch /mnt/
    cp /usr/bin/qemu-arm-static /mnt/usr/bin/
    chroot /mnt/ /usr/bin/qemu-arm-static /bin/bash -i
    update-binfmts --enable # Need if you have Exec format error


#### Inside chroot:

    export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    /debootstrap/debootstrap --second-stage

    # Debian repos
    cat <<EOT > /etc/apt/apt.conf.d/71-no-recommends
    APT::Install-Recommends "0";
    APT::Install-Suggests "0";
    EOT

    cat <<EOT > /etc/apt/sources.list
    deb http://http.debian.net/debian stretch main contrib non-free
    deb http://http.debian.net/debian stretch-updates main contrib non-free
    deb http://security.debian.org/debian-security stretch/updates main contrib non-free
    EOT

    apt update

    # Locales, timezone
    apt install locales
    dpkg-reconfigure locales
    cp /usr/share/zoneinfo/Europe/Warsaw /etc/localtime

    cat <<EOT >> etc/fstab
    # Mountpoints
    /dev/mmcblk0p1  /  ext4  defaults,noatime,nodiratime,commit=600,errors=remount-ro  0  1
    none /tmp tmpfs defaults,noatime,mode=1777 0 0
    EOT

    ln -s /proc/self/mounts /etc/mtab

    # Network
    cat <<EOT >> /etc/network/interfaces
    auto eth0
    allow-hotplug eth0
    iface eth0 inet dhcp
    EOT

    # Tools
    apt install aptitude python bash-completion mc htop lvm2 ncdu deborphan

    # Postconfig
    passwd
    echo Your_Hostname > /etc/hostname

    # Cleanup
    apt clean
    apt purge `deborphan --guess-all` # Many times
    aptitude purge ~c

    rm /mnt/usr/bin/qemu-arm-static
    exit


### Arch Linux:

    mount /dev/sdb1 /mnt/
    wget http://os.archlinuxarm.org/os/ArchLinuxARM-armv7-latest.tar.gz
    bsdtar -xpf ArchLinuxARM-armv7-latest.tar.gz -C /mnt/
    sync

Or [Building in a Clean Chroot](https://wiki.archlinux.org/index.php/DeveloperWiki:Building_in_a_Clean_Chroot)

### Install kernel and bootloader:

#### On installed filesystem:


/boot/boot.cmd:

    setenv bootargs console=ttyS0,115200 root=/dev/mmcblk0p1 rootwait panic=10 rw
    ext4load mmc 0 ${fdt_addr_r} /boot/dtb/${fdtfile}
    ext4load mmc 0 ${kernel_addr_r} /boot/zImage
    env set fdt_high ffffffff
    bootz ${kernel_addr_r} - ${fdt_addr_r}

Then:

    mkimage -C none -A arm -T script -d boot.cmd boot.scr

Copy:

- out/lib/modules/**/VERSION/** → /lib/modules/
- arch/arm/boot/**zImage** → /boot/
- arch/arm/boot/dts/**YOUR_BOARD.dtb** → /boot/dtb/





## Initramfs (optional):
#TODO

mkdir -p initramfs/rootfs/{bin,dev,etc,lib,mnt/root,proc,root,sbin,sys}
cd initramfs/
sudo cp -va /dev/{null,console,tty} rootfs/dev/
git clone git://git.busybox.net/busybox
cd busybox/
git tag
git checkout 1_26_0
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make clean
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make defconfig
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make menuconfig
Busybox Settings --> Build Options --> Build Busybox as a static binary (no shared libs)
Busybox Settings --> Build Options --> Cross compiler prefix = "arm-linux-gnueabihf-"
Busybox Settings --> Don't use /usr
Linux Module Utilities --> () Default directory containing modules = ""
Linux Module Utilities --> () Default name of modules.dep = ""
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make -j5
cd ..
cp busybox/busybox rootfs/bin/
nano rootfs/init

find . -print0 | cpio --null -ov --format=newc > ../initramfs.cpio
cd ..
mkimage -A arm -T ramdisk -C none -n uInitrd -d initramfs.cpio uInitrd























With ramdisk
/boot/boot.cmd:
setenv bootargs console=ttyS0,115200 root=/dev/mmcblk0p1 rootwait panic=10 rw
ext4load mmc 0 ${fdt_addr_r} /boot/dtb/${fdtfile}
ext4load mmc 0 ${kernel_addr_r} /boot/zImage
ext4load mmc 0 ${ramdisk_addr_r} /boot/uInitrd
env set fdt_high ffffffff
bootz ${kernel_addr_r} ${ramdisk_addr_r} ${fdt_addr_r}
