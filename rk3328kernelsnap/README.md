## Prepare Image
```
sudo apt-get install u-boot-tools mtools libudev-dev libusb-1.0-0-dev python-linaro-image-tools autotools-dev \
     libsigsegv2 m4 libdrm-dev sed make binutils build-essential gcc g++ patch gzip bzip2 perl tar cpio python \
     unzip rsync file bc wget libncurses5 libglib2.0-dev openssh-client lib32stdc++6 -y
sudo apt-get install bc bison build-essential curl device-tree-compiler dosfstools flex gcc-aarch64-linux-gnu \
     gcc-arm-linux-gnueabihf gdisk git gnupg gperf libc6-dev libncurses5-dev libpython-dev libssl-dev libssl1.0.0 \
     lzop mtools parted repo swig tar zip -y
sudo apt-get install qemu qemu-user-static binfmt-support debootstrap -y

git clone https://github.com/FireflyTeam/repo.git && sudo cp repo/repo /usr/bin/
repo init --repo-url https://github.com/FireflyTeam/repo.git -u https://github.com/FireflyTeam/manifests.git -b linux-sdk -m rk3328/rk3328_linux_release.xml
repo sync -c

cp <your-rootfs.img> rootfs/rk3328-ubuntu1804-arm64-rootfs.img
echo 'export RK_ROOTFS_IMG=aaa' |sed -r 's/export RK_ROOTFS_IMG=(.+)/export RK_ROOTFS_IMG=rootfs\/rk3328-ubuntu1804-arm64-rootfs.img/g;t;d'

./build.sh roc-rk3328-cc.mk
ls -l device/rockchip/.BoardConfig.mk

./build.sh uboot

ls kernel/arch/arm64/boot/dts/rockchip/rk3328-roc-cc.dts
cd ../kernel
export ARCH=arm64
make firefly_linux_defconfig                #use default config first
make menuconfig                             #custom your config
CONFIG_USBIP_CORE=y
make savedefconfig                          #save as default, to generate the file defconfig
cp defconfig arch/arm64/configs/firefly_linux_defconfig  #use defconfig
#make ARCH=arm64 rk3328-roc-cc.img -j8
cd .. && ./build.sh kernel                  #compile kernel
ls kernel/arch/arm64/boot/Image

$ ls ./rockdev/
boot.img  idbloader.img  MiniLoaderAll.bin  misc.img  oem.img  parameter.txt  rootfs.img  trust.img  uboot.img  userdata.img
```
## make rootfs

```
sudo apt-get install qemu qemu-user-static binfmt-support debootstrap
wget http://cdimage.ubuntu.com/ubuntu-base/releases/18.04/release/ubuntu-base-18.04.4-base-arm64.tar.gz
fallocate -l 2000M rootfs.img
#dd if=/dev/zero of=rootfs.img bs=1M count=0 seek=1024
sudo mkfs.ext4 -F -L ROOTFS rootfs.img
mkdir mnt
sudo mount rootfs.img mnt
sudo tar -xzvf ubuntu-base-18.04.4-base-arm64.tar.gz -C mnt/
sudo cp -a /usr/bin/qemu-aarch64-static mnt/usr/bin/  #qemu-aarch64-static can chroot into arm64 fs in x86_64
sudo chroot mnt/
USER=hua
HOST=nanopi
useradd -G sudo -m -s /bin/bash $USER
passwd $USER
echo $HOST > /etc/hostname
echo "127.0.0.1    localhost.localdomain localhost" > /etc/hosts
echo "127.0.0.1    $HOST" >> /etc/hosts
echo "auto eth0" > /etc/network/interfaces.d/eth0
echo "iface eth0 inet dhcp" >> /etc/network/interfaces.d/eth0
echo "nameserver 8.8.8.8" > /etc/resolv.conf
#mknod random c 1 8 && mknod urandom c 1 9
function enter_chroot() {
    echo "MOUNTING"
    sudo mount -t proc /proc ${1}proc
    sudo mount -t sysfs /sys ${1}sys
    sudo mount -o bind /dev ${1}dev
    #need to mount 'dev/pts' as well, or it will cause ssh to 'PTY allocation request failed on channel 0'
    sudo mount -o bind /dev/pts ${1}dev/pts		
    sudo chroot ${1}
}
function umount_chroot() {
    echo "UNMOUNTING"
    sudo umount ${1}proc
    sudo umount ${1}sys
    sudo umount ${1}dev/pts
    sudo umount ${1}dev
}
enter_chroot mnt/
apt-get update
apt-get upgrade
apt-get install ifupdown net-tools network-manager iputils-ping tcpdump traceroute-y
apt-get install udev sudo ssh vim bash bash-completion kmod snapd docker -y
apt-get install shadowsocks -y
ln -s /lib/systemd/system/serial-getty\@.service /etc/systemd/system/getty.target.wants/serial-getty@ttyS0.service
dpkg --add-architecture armhf #wait to verify
apt-get update
apt-get install libc6:armhf
exit
sudo sync
sudo umount mnt/
```
## package rk image
```
#package firmware
$ cat device/rockchip/rk3328/parameter-ubuntu.txt
FIRMWARE_VER: 8.1
MACHINE_MODEL: RK3328
MACHINE_ID: 007
MANUFACTURER: RK3328
MAGIC: 0x5041524B
ATAG: 0x00200800
MACHINE: 3328
CHECK_MASK: 0x80
PWR_HLD: 0,0,A,0,1
TYPE: GPT
CMDLINE: mtdparts=rk29xxnand:0x00002000@0x00004000(uboot),0x00002000@0x00006000(trust),0x00010000@0x0000a000(boot),0x00010000@0x0002a000(backup),-@0x0005a000(rootfs:grow)
uuid:rootfs=614e0000-0000-4b53-8000-1d28000054a9
$ cat ./tools/linux/Linux_Pack_Firmware/rockdev/package-file
# NAME          Relative path
#
#HWDEF          HWDEF
package-file    package-file
bootloader      Image/MiniLoaderAll.bin
parameter       Image/parameter.txt
trust           Image/trust.img
uboot           Image/uboot.img
#resource       Image/resource.img
#kernel         Image/kernel.img
boot            Image/boot.img
rootfs:grow     Image/rootfs.img
backup          RESERVED

./mkfirmware.sh       #update files to ./rockdev dir before packaging
$ ls ./rockdev/
boot.img  idbloader.img  MiniLoaderAll.bin  misc.img  oem.img  parameter.txt  rootfs.img  trust.img  uboot.img  userdata.img
./build.sh updateimg  #start to package
ls ./rockdev/update.img
scp rockdev/update.img root@baidu:/root/nanopi_r2s_ubuntu_arm64_update.img

分区介绍：请最少保留uboot,trust,boot,rootfs分区
注意：若发现根文件分区大小异常时，执行如下命令：　resize2fs /dev/mmcblk2p5
uboot    分区:  uboot编译出来的 uboot.img.
trust    分区:  uboot编译出来的 trust.img
misc     分区:  misc.img开机检测进入recovery模式.(可省略)
boot     分区:  编译出来的 boot.img包含kernel和设备树信息.
recovery 分区:  烧写 recovery.img.(可省略)
backup   分区:  预留,暂时没有用。后续跟 android 一样作为 recovery 的 backup 使用.
oem      分区:  给厂家使用,存放厂家的 app 或数据，只读，代替原来音箱的 data 分区，挂载在/oem 目录.(可省略)
rootfs   分区:  存放 buildroot 或者 debian 编出来的rootfs.img只读.
userdata 分区:  存放app临时生成的文件或者是给最终用户使用。可读写,挂载在/userdata目录下.(可省略)

http://wiki.t-firefly.com/ROC-RK3328-CC/fw_pack_unpack.html
http://www.t-firefly.com/download/Firefly-RK3399/docs/Rockchip%20Parameter%20File%20Format%20Ver1.3.pdf
CMDLINE含义：
CMDLINE:mtdparts=rk29xxnand:0x00002000@0x00004000(uboot),0x00002000@0x00006000(trust),0x00010000@0x0000a000(boot),0x00010000@0x0002a000(backup),-@0x0005a000(rootfs:grow)
#例: 0x00002000@0x00004000(uboot), @符号之前的数值是分区大小，@符号之后的数值是分区的起始位置
#单位都是sector,1个sector是512Byptes，echo $(( 0x002000 * 512 / 1024 / 1024 ))M = 4M,
#即uboot分区起始8M(0x00004000)大小为4M(0x00002000)，trust分区起始12M(0x00006000)大小为4M, boot分区起始20M大小为32M
#大小4M=4194304 - echo $(( 4 * 1024 * 1024)) = 4194304, echo $(( 32 * 1024 * 1024))=33554432
#16进制转10进制 0x00002000 是2097152 (https://tool.lu/hexconvert/)

#Burn write image - http://wiki.t-firefly.com/zh_CN/ROC-RK3328-CC/linux_compile.html
wget http://download.t-firefly.com/product/RK3328/Tools/Linux_Upgrade_Tool/Linux_Upgrade_Tool_1.34.zip
unzip Linux_Upgrade_Tool_v1.34.zip
cd Linux_Upgrade_Tool
sudo mv upgrade_tool /usr/local/bin
sudo chown root:root /usr/local/bin/upgrade_tool
sudo upgrade_tool uf update.img
#or burn write image by partion
sudo upgrade_tool ul $LOADER
sudo upgrade_tool di -p $PARAMETER
sudo upgrade_tool di -uboot $UBOOT
sudo upgrade_tool di -trust $TRUST
sudo upgrade_tool di -b $BOOT
sudo upgrade_tool di -rootfs $ROOTFS
```

