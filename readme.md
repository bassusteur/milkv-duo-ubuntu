# Ubuntu on the Milk-V Duo
![F-MbY2FXUAAk21P](https://github.com/bassusteur/milkv-duo-ubuntu/assets/42449683/3dcdbd84-12a6-4c86-969c-92a2e9e92496)

## Setup 
1. Ubuntu 22.04 LTS installed on a virtual machine
2. Setup ![duo-buildroot-sdk](https://github.com/milkv-duo/duo-buildroot-sdk#prepare-the-compilation-environment) on your machine

## Creating the rootfs
```bash
# Install prerequisites
sudo apt install debootstrap binfmt-support dpkg-cross --no-install-recommends
# generate minimal bootstrap rootfs
sudo debootstrap --arch=riscv64 --foreign jammy ./temp-rootfs http://ports.ubuntu.com/ubuntu-ports
```
