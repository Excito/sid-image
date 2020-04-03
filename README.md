# buster-image

## Description

In this repository you will find all the steps and extra files used to create the sid image for the Bubba|2 platform.

Given the fact that image creation is not done every day there is no scripted/automated method to create it ; instead all the steps needed (and taken) to create the released image will be described in this README. It will be updated with each release.

The current version for the excito Bubba|2 sid image is **1.0**

## Contents

- This README.md file, the "image create manual"
- `first-boot` which contains first-boot scripts run ... on the first boot; Currently these scripts:
  - Generate new ssh host keys
  - Remove themselves

## Image creation

### Pre-requisites

In order to create an image you need a Bubba|2 running whatever standard Linux OS you wish, ~1GiB free space and an internet connection; The only software needed on the host is the debootstrap package.

The released image are created from the [install/rescue system](https://github.com/Excito/buildroot) which provides all the necessary tools.

Everything must be run as `root`.

### Boostraping and chroot into the system

- Create a directory that will host the image (it doesn't have to be a mounted partition):
```
mkdir /mnt/target
```
- Debootstrap sid from the debian ports servers:
```
debootstrap --arch=powerpc --exclude=powerpc-utils --include debian-ports-archive-keyring sid /mnt/target http://ftp.ports.debian.org/debian-ports
```
- Mount the kernel filesystems:
```
mount -t proc none /mnt/target/proc
mount -t sysfs none /mnt/target/sys
mount -o bind /dev /mnt/target/dev
mount -t devpts none /mnt/target/dev/pts
mount -t tmpfs none /mnt/target/dev/shm
```
- Create the `policy-rc.d` file which will prevent the daemons to be run inside the chroot:
```
cat > /mnt/target/usr/sbin/policy-rc.d << EOF
#!/bin/sh
exit 101
EOF
chmod 755 /mnt/target/usr/sbin/policy-rc.d
```
- Download the `excito-release-sid` package for later install:
```
wget -O/mnt/target/root/excito-release-sid.deb http://repo.excito.org/excito-release-sid.deb

```
- Chroot into the system and setup the environment:
```
chroot /mnt/target /bin/bash
source /etc/profile
cd /root
export PS1="(chroot) $PS1"
```

### APT configuration and standard package install

- Create the `/etc/mtab` link:
```
ln -s /proc/mounts /etc/mtab
```
- Install the `excito-release-sid` package:
```
dpkg -i excito-release-sid.deb
rm excito-release-sid.deb
```
- Update apt cache and ugprade the system:
```
apt-get update
apt-get -y dist-upgrade
```
- Install kernel and Bubba|2 utils:
```
apt-get -y install bubba3-kernel bubbatwo-utils
```
- Install locales and the ssh server:
```
apt-get -y install locales
tasksel install --new-install ssh-server
```

### [Optional] Install u-boot-tools to access u-boot configuration

Released images provide `fw_setenv` and `fw_printenv` which allows access and modification of the platform bootloader. These tools are only 'nice to have'. Beware that misuse can prevent the system from booting.

- Create the `fw_env.config` file:
```
cat > /etc/fw_env.config << EOF
# MTD definition for Bubba|2
# MTD device name  Device offset  Env. size  Flash sector size  Number of sectors
/dev/mtd0          0x50000        0x002000   0x10000
/dev/mtd0          0x60000        0x002000   0x10000
EOF
```
- Install the `u-boot-tools` and the `mtd-utils` package:
```
apt-get -y install u-boot-tools mtd-utils
```

### User and password creation

- Set the root password ('excito' without quotes on the released images):
```
passwd
```
- Create the `excito` user and set its password ('excito' without quotes on the released images):
```
useradd -m -U -s /bin/bash excito
passwd excito
```

### System configuration

- Configure the network:
```
cat >> /etc/network/interfaces << EOF

allow-hotplug eth0
iface eth0 inet dhcp

allow-hotplug eth1
iface eth1 inet dhcp
EOF
```
- Create `/etc/fstab`:
```
cat > /etc/fstab << EOF
/dev/sda1   /   ext3    noatime 0   1
EOF
```
- Set the hostname:
```
echo "bubbatwo" > /etc/hostname
```
- Disable interface predictable naming:
```
ln -s /dev/null /etc/systemd/network/99-default.link
```

### Cleanup

- Empty apt cache:
```
apt-get clean
```
- Remove previously created ssh keys (they will be recreated by the `first-boot` files):
```
rm /etc/ssh/*key*
```
- Exit the chroot:
```
exit
```
- Remove the shell history file and the previously created `policy-rc.d` file:
```
rm /mnt/target/usr/sbin/policy-rc.d /mnt/target/root/.bash_history
```
- Unmount kernel filesystem:
```
umount /mnt/target/dev/shm
umount /mnt/target/dev/pts
umount /mnt/target/dev
umount /mnt/target/sys
umount /mnt/target/proc
```

### `first-boot` files and tarball creation ###

- Download and extract the first-boot release tarball into target:
```
wget -O/mnt/target/first-boot.txz https://github.com/Excito/sid-image/releases/download/v1.0/first-boot-1.0.txz
( cd /mnt/target; tar -xvf first-boot.txz )
rm /mnt/target/first-boot.txz
```
- Now that the image files are ready, go ahead and create the final tarball:
```
( cd /mnt/target; tar -cJvf /root/sid-image.txz .)
```
