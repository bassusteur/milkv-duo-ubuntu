# Ubuntu on the Milk-V Duo
![F-MbY2FXUAAk21P](https://github.com/bassusteur/milkv-duo-ubuntu/assets/42449683/3dcdbd84-12a6-4c86-969c-92a2e9e92496)

## Setup 
1. Ubuntu 22.04 LTS installed on a virtual machine
2. Setup ![duo-buildroot-sdk](https://github.com/milkv-duo/duo-buildroot-sdk#prepare-the-compilation-environment) on your machine

## Before anything else
```bash
# We need to enable a few modules in the kernel configuration before we can continue, so:
nano ~/duo-buildroot-sdk/build/boards/cv180x/cv1800b_milkv_duo_sd/linux/cvitek_cv1800b_milkv_duo_sd_defconfig

# and add at the end:
CONFIG_CGROUPS=y
CONFIG_CGROUP_FREEZER=y
CONFIG_CGROUP_PIDS=y
CONFIG_CGROUP_DEVICE=y
CONFIG_CPUSETS=y
CONFIG_PROC_PID_CPUSET=y
CONFIG_CGROUP_CPUACCT=y
CONFIG_PAGE_COUNTER=y
CONFIG_MEMCG=y
CONFIG_CGROUP_SCHED=y
CONFIG_NAMESPACES=y
CONFIG_OVERLAY_FS=y
CONFIG_AUTOFS4_FS=y
CONFIG_SIGNALFD=y
CONFIG_TIMERFD=y
CONFIG_EPOLL=y
CONFIG_IPV6=y
CONFIG_FANOTIFY

# optional (enable zram):
CONFIG_ZSMALLOC=y
CONFIG_ZRAM=y
```
Important: to reduce ram usage follow point n.2 of the ![faq](https://github.com/milkv-duo/duo-buildroot-sdk/tree/develop#faqs), 
To increase the rootfs partition size you can edit ```bash duo-buildroot-sdk/milkv/genimage-milkv-duo.cfg```
at line 16 replace ```size = 768M``` with ```size = 1G``` or higher as desired
then follow the ![instructions](https://github.com/milkv-duo/duo-buildroot-sdk#step-by-step-compilation) to manually compile buildroot and the kernel and pack it. 

## Creating the rootfs
```bash
# install prerequisites
sudo apt install debootstrap qemu qemu-user-static binfmt-support dpkg-cross --no-install-recommends
# generate minimal bootstrap rootfs
sudo debootstrap --arch=riscv64 --foreign jammy ./temp-rootfs http://ports.ubuntu.com/ubuntu-ports
# chroot into the rootfs we just created
sudo chroot temp-rootfs /bin/bash
# run 2nd stage of deboostrap
/debootstrap/debootstrap --second-stage
# add package sources
cat >/etc/apt/sources.list <<EOF
deb http://ports.ubuntu.com/ubuntu-ports jammy main restricted

deb http://ports.ubuntu.com/ubuntu-ports jammy-updates main restricted

deb http://ports.ubuntu.com/ubuntu-ports jammy universe
deb http://ports.ubuntu.com/ubuntu-ports jammy-updates universe

deb http://ports.ubuntu.com/ubuntu-ports jammy multiverse
deb http://ports.ubuntu.com/ubuntu-ports jammy-updates multiverse

deb http://ports.ubuntu.com/ubuntu-ports jammy-backports main restricted universe multiverse

deb http://ports.ubuntu.com/ubuntu-ports jammy-security main restricted
deb http://ports.ubuntu.com/ubuntu-ports jammy-security universe
deb http://ports.ubuntu.com/ubuntu-ports jammy-security multiverse
EOF

# update and install some packages
apt-get update
apt-get install --no-install-recommends -y util-linux haveged openssh-server systemd kmod initramfs-tools conntrack ebtables ethtool iproute2 iptables mount socat ifupdown iputils-ping vim dhcpcd5 neofetch sudo chrony
# optional for zram
apt-get install zram-config
systemctl enable zram-config

# Create base config files
mkdir -p /etc/network
cat >>/etc/network/interfaces <<EOF
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
EOF

cat >/etc/resolv.conf <<EOF
nameserver 1.1.1.1
nameserver 8.8.8.8
EOF

# write text to fstab (this is with swap enabled if you want to disable it just put a # before the swap line)
cat >/etc/fstab <<EOF
# <file system>	<mount pt>	<type>	<options>	<dump>	<pass>
/dev/root	/		ext2	rw,noauto	0	1
proc		/proc		proc	defaults	0	0
devpts		/dev/pts	devpts	defaults,gid=5,mode=620,ptmxmode=0666	0	0
tmpfs		/dev/shm	tmpfs	mode=0777	0	0
tmpfs		/tmp		tmpfs	mode=1777	0	0
tmpfs		/run		tmpfs	mode=0755,nosuid,nodev,size=64M	0	0
sysfs		/sys		sysfs	defaults	0	0
/dev/mmcblk0p3  none            swap    sw              0       0
EOF

# set hostname
echo "milkvduo-ubuntu" > /etc/hostname

# set root passwd
echo "root:riscv" | chpasswd

# enable root login through ssh
sed -i "s/#PermitRootLogin.*/PermitRootLogin yes/g" /etc/ssh/sshd_config

# exit chroot
exit
sudo tar -cSf Ubuntu-jammy-rootfs.tar -C temp-rootfs .
gzip Ubuntu-jammy-rootfs.tar
rm -rf temp-rootfs

```
## Flashing
next up, we flash the image on the sd card like so:
```bash
dd if=milkv-duo.img of=/dev/sdX status=progress #replace X with your device name
```
we mount the rootfs partition and we delete all the files inside with ```bash sudo rm -r /media/yourusername/rootfs```
