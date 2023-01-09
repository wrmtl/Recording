#荔枝派zero移植

##开发板介绍

- 荔枝派zero
- CPU：sun8i-v3s
- TF卡启动
- 带扩展板
  - 以太网接口
  - 5寸液晶屏
  - SDIO wifi：rtl8723bs
  - microUSB
  - 3.5 mm音频接口

##移植过程简介

- 配置开发环境
  - Linux系统
  - 交叉编译链
  - 依赖库和工具
  - 源码

- 移植部分（采用主线`Linux`）
  - Uboot
  - Linux内核及驱动
  - 根文件系统`buildroot`

##配置开发环境

### 参考教程

- [参考教程1](https://yuanze.wang/posts/build-boot-image-for-v3s/)
- [参考教程2](https://blog.csdn.net/a480694856/article/details/115328891)
- [参考教程3](https://blog.csdn.net/qq_28877125/article/details/117969199)

### HOST Linux系统的选择

- 建议使用`Debian`系或者`Ubuntu`（安装库和工具的时候方便且可靠）

  我使用的时`Deepin20`，`debian`文件系统（需要先安装`apt`工具）

- 推荐使用虚拟机或者`docker`

  - 系统环境纯净稳定
  - 方便后续开发

###安装交叉编译链

`V3s`为`ARM`架构，采用`HOST`交叉编译的方法，生成目标机器的代码。本文选用的时`linaro`公司推出的`arm-linux-gnueabihf`交叉编译工具。公司维护编译代码可靠性高稳定性好。在学习使用`buildroot`编译根文件系统时，也可以源码编译生成交叉编译工具，但个人并不推荐（开源才是真的香）。我的`HOST`时`x86_64`的系统，所使用的时`x86_64`版本的交叉编译器，编译器版本为`7.5.0`。

```bash
cd ~/Download
wget https://releases.linaro.org/components/toolchain/binaries/latest-7/arm-linux-gnueabihf/gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf.tar.xz
sudo tar xf gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf.tar.xz -C /opt
sudo mv gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf arm-linux-gnueabihf-gcc #重命名文件夹为arm-linux-gnueabihf-gcc
```

下载并安装后，配置编译器的路径到`PATH`，方便`make`。

```bash
echo "PATH=\$PATH:/opt/arm-linux-gnueabihf-gcc/bin" >> ~/.bashrc
source ~/.bashrc
```

##系统移植

###编译U-boot

从`gihub`上下载荔枝派官方修改过的`U-boot`源码。

```bash
cd ~/code/licheepi-zero
git clone https://github.com/Lichee-Pi/u-boot -b v3s-current
```

编译前，可能需要安装一些依赖。

```bash
sudo apt install u-boot-tools python3-distutils python3-dev swig build-essential libncurses5-dev git 
```

进入代码文件，根据自己的开发板（屏幕分辨率），配置并编译。

```bash
cd u-boot
# make ARCH=arm LicheePi_Zero_480x272LCD_defconfig
make ARCH=arm LicheePi_Zero_800x480LCD_defconfig #根据你使用的屏幕分辨率进行选择
# make ARCH=arm LicheePi_Zero_defconfig #没有屏幕
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j4
```

编译生成的`U-boot`镜像在`u-boot`的根目录下，`u-boot-sunxi-with-spl.bin`文件。

###编译Linux内核

从`github`上下载荔枝派官方的`Linux`源码。这里使用的是`zero-5.2.y`分支，配置文件和驱动已经深度定制，已经适配了音频、有线网卡等的驱动，并配置完毕`dtb`。

```bash
cd ~/code/licheepi-zero
git clone https://github.com/Lichee-Pi/linux.git -b zero-5.2.y
```

可能需要安装一些依赖包和工具。

```bash
sudo apt install flex bison libssl-dev device-tree-compiler
```

进入源码文件夹，配置并编译。

```bash
cd linux
# make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- sunxi_defconfig	# linux官方提供的sunxi_deconfig配置
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- licheepi_zero_defconfig	# lichee官方提供的配置，驱动、dtb都修改了
```

`sunxi_defconfig`并没有开启有线网卡的驱动。如果需要使用有线网卡，还需要在`menuconfig`中开启。首先执行。

```bash
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make menuconfig
```

并在菜单中，开启如下的有线网卡驱动支持。

```txt
Device Drivers —>
 Network device support —>
  Ethernet driver support —>
   [*]   STMicroelectronics devices
     <*>     STMicroelectronics 10/100/1000/EQOS Ethernet driver
     <*>       STMMAC Platform bus support
     < >         Support for snps,dwc-qos-ethernet.txt DT binding.
     <*>         Generic driver for DWMAC
     <*>         Allwinner GMAC support
     <*>         Allwinner sun8i GMAC support
```

此仓库中源码内的dts文件中只开启了`uart0`这一个串口作为调试串口使用。若需要使用所有3个串口，还需要对`dts`文件进行修改。首先对`\arch\arm\boot\dts\sun8i-v3s.dtsi`进行修改，加入对串口引脚的定义。

```dts
uart0_pins_a: uart0@0 { pins = "PB8", "PB9"; function = "uart0";bias-pull-up; };
uart1_pins_a: uart1@0 { pins = "PE21", "PE22"; function = "uart1";bias-pull-up; };
uart2_pins_a: uart2@0 { pins = "PB0", "PB1"; function = "uart2";bias-pull-up; };
```

然后修改`\arch\arm\boot\dts\sun8i-v3s-licheepi-zero.dts`，使能这3个串口。

```dts
&uart0 { pinctrl-0 = <&uart0_pins_a>; pinctrl-names = "default";status = "okay"; };
&uart1 { pinctrl-0 = <&uart1_pins_a>; pinctrl-names = "default";status = "okay"; };
&uart2 { pinctrl-0 = <&uart2_pins_a>; pinctrl-names = "default";status = "okay"; };
```

配置完成后，对源码进行编译。

```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j12
ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- make modules_install INSTALL_MOD_PATH=`pwd`/output/		#  install 驱动
ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- make modules_install INSTALL_MOD_PATH=/media/zzt/d1d3824d-62b2-417d-81ee-4a6e2750bc601		# install 驱动到根文件系统
```

`modules_install`安装驱动的过程中会使用`depmod`生成驱动的依赖关系，在使用`modprobe yout_driver.ko`加载驱动时会加载依赖的驱动。推荐直接 install 到接下要制作的根文件系统。

得到内核镜像`zImage`和设备树`sun8i-v3s-licheepi-zero-dock.dtb`，分别位于`arch/arm/boot/zImage`和`arch/arm/boot/dts/sun8i-v3s-licheepi-zero-dock.dtb`。

### 编译rootfs

查询编译工具链所在位置：

```bash
which arm-linux-gnueabihf-gcc
```

读取编译工具链里的内核版本:

```bash
cat /opt/gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf/arm-linux-gnueabihf/libc/usr/include/linux/version.h

查询得到
#define LINUX_VERSION_CODE 264707
#define KERNEL_VERSION(a,b,c) (((a) << 16) + ((b) << 8) + (c))

```

十进制数264707的十二进制为0x40A03，则对应的内核版本号为4.10.3。

从[BuildRoot](https://buildroot.org/download.html)网站上下载`Buildroot`源码并解压。这里我以Buildroot `2020.02`为例。

```bash
cd ~/Downloads
wget https://buildroot.org/downloads/buildroot-2020.02.8.tar.gz
tar xf buildroot-2020.02.8.tar.gz -C ~/code/licheepi-zero
cd ~/code/licheepi-zero/buildroot-2020.02.8
mv buildroot-2020.02.8 buildroot-2020.02 # 重命名
make menuconfig
```

在编译之前，需要安装BuildRoot的依赖项。

```bash
sudo apt install python texinfo unzip
```

通过`menuconfig`配置

```txt
Target Architecture (ARM (little endian))  --->
Target Binary Format (ELF)  --->
Target Architecture Variant (cortex-A7)  --->
Target ABI (EABIhf)  --->
Floating point strategy (VFPv4-D16)
ARM instruction set (ARM)

Target options  --->
	Target Architecture (ARM (little endian))  --->
	Target Binary Format (ELF)  --->
	Target Architecture Variant (cortex-A7)  --->
	Target ABI (EABIhf)  --->
	Floating point strategy (VFPv4)  --->
	ARM instruction set (ARM)  ---> 


### 编译工具链配置：
Toolchain  --->
	Toolchain type (External toolchain)  --->
	*** Toolchain External Options ***
	Toolchain (Custom toolchain)  --->
	Toolchain origin (Pre-installed toolchain)  --->
	/opt/gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf/arm-linux-gnueabihf/) 
	Toolchain path
	($(ARCH)-linux-gnueabihf) Toolchain prefix 改为工具链前缀是： arm-linux-gnueabihf
	External toolchain gcc version (7.0.x)  --->
	External toolchain kernel headers series (4.10.x)  --->
	External toolchain C library (glibc/eglibc)  --->
	[*] Toolchain has SSP support? (NEW)
	[*] Toolchain has RPC support? (NEW)
	[*] Toolchain has C++ support? 
	[*] Enable MMU support (NEW) 
```

可以在`Target packages`中选择自己需要的模块。这里以`minicom`与`python3`为例

```txt
Target packages  --->
   Hardware handling  --->
     [*] minicom
   Interpreter languages and scripting  --->
     [*] python3
           python3 module format to install (.py sources and .pyc compiled)  --->
           core python3 modules  --->
         External python modules  --->
           [*] python-pip
           [*] python-serial
```

编译

```bash
make
```

如果出错则使用`make clean`然后再编译。
编译完成后，生成的根文件系统在`output/images/rootfs.tar`。

可以使用如下配置对应工具

```bash
make busybox-menuconfig			# 配置busybox
```

测试屏幕

```bash
cat /dev/urandom > /dev/fb0			# 雪花
cat /dev/zero > /dev/fb0			# 黑屏
```

##烧录镜像

对`tf`卡进行分区，注意查看自己的`tf`卡的设备，假设设备为`/dev/sdb`。使用`GParted`对`tf`卡进行格式化和分区。

- 新建存放系统镜像设备树与启动脚本的`boot`分区。右键点击未分配的空间，选择新建，新建一个前部有`1M`空闲空间，大小为`32M`的`FAT32`分区，命名为`boot`。

- 选择剩余的所有空间，创建一个`ext4`格式的`rootfs`分区（即为点击新建后默认的选项）。
- 确认无误后，点击绿色的对勾提交更改，TF卡格式化就完毕了。

###烧录uboot

```bash
cd ~/code/licheepi-zero/u-boot
sudo dd if=u-boot-sunxi-with-spl.bin of=/dev/sdb bs=1024 seek=8
```

uboot启动配置

使用启动脚本。启动脚本用于指定`u-boot`的启动行为，包括内核与设备树的文件名、应该将哪个分区挂载为`rootfs`等。

在`~/code/licheepi-zero/`目录下，新建`boot.cmd`，将下面内容写入。

```bash
setenv bootargs console=ttyS0,115200 panic=5 rootwait root=/dev/mmcblk0p2 earlyprintk rw
load mmc 0:1 0x41000000 zImage
load mmc 0:1 0x41800000 sun8i-v3s-licheepi-zero.dtb
bootz 0x41000000 - 0x41800000
```

`ttyS0`是使用`uart0`作为内核启动加载信息的输出对象，可以选用`tty0`直接将信息显示到`tft lcd`屏幕上。当然可以同时都开启。

然后，编译启动脚本。

```bash
mkimage -C none -A arm -T script -d boot.cmd boot.scr
```

得到的`boot.scr`即为启动脚本，将其复制到`boot`分区，即完成了可启动`tf`卡的制作。

###烧录Linux内核和设备树

将`licheepi-zero/linux/arch/arm/boot/zImage`和`licheepi-zero/linux/arch/arm/boot/dts/sun8i-v3s-licheepi-zero.dtb`两个文件复制到`boot`分区中。

###烧录根文件系统

将`licheepi-zero/buildroot-2020.02/output/images/rootfs.tar`解压到`rootfs`分区中。这必须要使用命令行操作，请确认`rootfs`挂载的路径，通常来说为`/media/用户名/rootfs`

```bash
sudo tar xf licheepi-zero/buildroot-2020.02/output/images/rootfs.tar -C /media/用户名/rootfs
sudo sync # 将更改写入卡中
```

## 常用功能的添加

### 编译驱动注意事项

- 驱动编译的内核树要和Linux内核的源码一致，否则`insmod`报`invalid module format`错误（最好将驱动放入Linux源码一起编译）

###WIFI RTL8723bs

涉及到`Linux`内核和`buildroot`根文件系统

参考教程

- [参考教程1](https://www.cnblogs.com/ZQQH/p/8366992.html)

- [参考教程2](https://www.its404.com/article/u012577474/104678522)

####Linux内核配置

`menuconfig`配置开启r8723bs驱动

```
Device Drivers
	Staging drivers
		<M> Realtek RTL8723BS SDIO Wireless LAN NIC driver
```

通过`module_install`将驱动和依赖安装到根文件系统

添加`WIFI`固件

```bash
mkdir -p  /lib/firmware/rtlwifi/
#拷贝 rtl8723bs_nic.bin 到根文件系统的 /lib/firmware/rtlwifi/ 目录下
```

加载驱动

```bash
modprobe r8723bs.ko
```

#### 根文件系统配置

添加网络配置工具

```txt
 buildroot 
-> make menuconfig
    -> Target packages 
    	-> Networking applications
			<*> wireless tools
			<*> wpa_supplicant
				<*> 所有子功能
```

`wpa_supplicant`添加配置文件: 编辑`/etc/wpa_supplicant.conf`文件

```bash
ctrl_interface=/var/run/wpa_supplicant  
ctrl_interface_group=0  
ap_scan=1  
network={
    ssid="ZQH"        
    scan_ssid=1
    key_mgmt=WPA-EAP WPA-PSK IEEE8021X NONE
    pairwise=TKIP CCMP
    group=CCMP TKIP WEP104 WEP40
    psk="123123123"  
    priority=5              
}
```

`wifi`配置

```bash
ifconfig wlan0 up	# 启动wlan0
wpa_supplicant -B -c /etc/wpa_supplicant.conf -i wlan0		# 通过配置文件链接wifi
udhcpc -i wlan0		# 自动获取ip地址

wpa_cli -iwlan0 status		# 查看网络状态
ping 192.168.1.1			#最后尝试下ping网络
```

开机自启动

添加`/etc/init.d/connect_wifi.sh`

```bash
#!/bin/sh
insmod /usr/lib/r8723bs.ko #加入驱动
ifconfig wlan0 up　　　　　　#开启wifi
wpa_supplicant -B -d -i wlan0 -c /etc/wpa_supplicant.conf　　　　#搜索wifi
sleep 3s
udhcpc -i wlan0　　　　　　　#连接wifi
```

编辑`/etc/init.d/rcS`

```bash
添加以下内容让他开机启动
# Add By ZQH 2018.1.27  start
if [ -e /etc/init.d/connect_wx.sh ]; then
        /etc/init.d/connect_wx.sh
fi
```

###Qt移植

可以自己下载源码编译，也可以通过`buildroot`完成构建

参考教程

- [交叉编译sqlite3和zlib](https://blog.51cto.com/u_15127667/4376901)
- 源码交叉编译Qt
  - [Qt4](https://blog.csdn.net/lionfire/article/details/107251022)
  - [Qt5.9](https://www.kancloud.cn/lichee/lpi0/418898)
- 通过buildroot编译Qt
  - [参考1](https://blog.csdn.net/u012577474/article/details/103365647)
  - [参考2](https://blog.csdn.net/Jun626/article/details/103688213)

注意事项

- `Qt`的交叉编译器要与根文件系统和`Linux`内核一致，保证共享库文件(`.so`)不冲突

- 我测试的，根文件系统`\lib`文件下包括的`.so`共享文件

  - 交叉编译器编译最后要链接的库文件

    一般位于`arm-linux-gnueabihf-gcc/arm-linux-gnueabihf-gcc/lib`下，而不是`arm-linux-gnueabihf-gcc/lib`(该文件夹是HOST交叉编译工具运行时需要的共享库)

  - `Qt install`生成的l`lib`目录下的共享库

- 配置`target`系统的`/etc/profile`下的环境变量

- `gnueabihf`不支持`gnueabi`下`softfloat`库

- 编译`Qt`的程序，需要修改`Qt install`的`qws/`下的`qconfig.h`对于交叉编译工具链的定义的变量

### uboot设备树和内核设备树

#### 点屏之SPI屏

参考教程

- [参考1](http://zero.lichee.pro/%E9%A9%B1%E5%8A%A8/SPI_LCD.html)

内核

- 内核添加fbtft驱动
- 设备树使能`spi0`，并配置`spi`下的`lcd`设备
- 删除默认的`RGB`的`framebuff`节点

Uboot

- `linux`内核设备树删除`framebuff`节点，`uboot`仍然会初始化`RGB`的驱动，只是内核会使用spi显示
- 如果需要完全去除`RGB`上的显示，需要在`uboot`里关闭显示

### Ethernet

思路

- `STMicroelectronics`中移植对`V3S`的`MAC`和`PHY`的代码支持
- `licheepi_zero_defconfig`中配置打开`STMMAC_ETH`及其依赖项
- `menuconfig`中按上面打开相应配置支持

### `SPI`屏幕的支持(`fbtft`的移植)

思路 : 通过`dmesg | grep fb`查看`fb`驱动模块载入信息

- 根据内核版本选择合适的`fbtft`版本
- 修改`fbtft`中的`bug`
  - `GPIO`的申请
  - `fbtft_bus.c`对`cs`的数据传输置位支持
- `DTS`配置`SPI`
  - `bias-pull-up` : 配置`MOSI`和`MISO`的上拉
  - `bgr`
  - `reset-gpios`、`dc-gpios`、`led-gpios`
  - `fps` : 配置合适的大小
- `fb_st7735s.c` : 添加对具体设备的支持
  - 关键是`init`的配置
  - 屏幕要支持`CS`

##Debian系统的移植

###移植部分简介

- Debian根文件系统
- 驱动移植
- 桌面系统的移植

###Debian根文件系统制作

参考教程

- [`Debian 9.9`文件系统](https://blog.csdn.net/u012577474/article/details/104661511?spm=1001.2014.3001.5502)
- [创建`swap`分区](https://blog.csdn.net/u012577474/article/details/104661511?spm=1001.2014.3001.5502)

`HOST`环境: `Deepin20`

```bash
###---------------------------------①----------------------------------------------------------
# 创建文件系统目录
mkdir /opt/rootfs -p

# 安装移植的必需库
# debootstrap用于制作Debian文件系统
# qemu-user-static用于模拟arm环境，实现非arm的HOST正常chroot
apt-get install debootstrap -y
apt-get install qemu-user-static -y

# 制作文件系统step 1 - 下载
cd /opt/
debootstrap --foreign --verbose --arch=armhf  stretch rootfs http://ftp2.cn.debian.org/debian 
#debootstrap --foreign --verbose --arch=armhf stretch rootfs http://ftp.de.debian.org/debian


###挂载###
mount --bind /dev   /opt/rootfs/dev/
mount --bind /sys   /opt/rootfs/sys/
mount --bind /proc   /opt/rootfs/proc/
mount --bind /dev/pts /opt/rootfs/dev/pts/

cp /usr/bin/qemu-arm-static /opt/rootfs/usr/bin/
chmod +x /opt/rootfs/usr/bin/qemu-arm-static
   
###解压###
LC_ALL=C LANGUAGE=C LANG=C chroot rootfs /debootstrap/debootstrap --second-stage --verbose

###可以在这个时候, 安装任何东西###
LC_ALL=C LANGUAGE=C LANG=C chroot rootfs
	#安装SSH
	apt install ssh
		##方向键变ABCD问题解决如下
 		 echo "set nocp" >> ~/.vimrc
		 source ~/.vimrc
	#修改sshd_config文件SSH支持root登录
	vi /etc/ssh/sshd_config
	添加 PermitRootLogin yes

###删除qemu-arm-static, 深藏功与名###
rm rootfs/usr/bin/qemu-arm-static

###卸载###
umount /opt/rootfs/dev/
umount /opt/rootfs/sys/
umount /opt/rootfs/proc/
umount /opt/rootfs/dev/pts/

cd /opt/rootfs/

压缩
tar cvzf ../debian9.9.rootfs.gz .

###解压debian9.9.rootfs.gz到SD卡
tar xvzf debian9.9.rootfs.gz /media/user_name/sdb
###-----------------------------------②----------------------------------------------------------------

#如果前面没设置root密码，后续设置root密码操作：
cp /usr/bin/qemu-arm-static /opt/rootfs/usr/bin/
chmod +x /opt/rootfs/usr/bin/qemu-arm-static
LC_ALL=C LANGUAGE=C LANG=C chroot rootfs
passwd  root
输入两次密码。
exit
cd /opt/rootfs/
tar cvzf ../debian9.9.rootfs.gz .
###解压debian9.9.rootfs.gz到SD卡


###-----------------------------------③----------------------------------------------------------------

后面安装软件
cp /usr/bin/qemu-arm-static /opt/rootfs/usr/bin/
chmod +x /opt/rootfs/usr/bin/qemu-arm-static
LC_ALL=C LANGUAGE=C LANG=C chroot rootfs
####安装ifconfig支持软件
	apt install net-tools

exit
cd /opt/rootfs/
tar cvzf ../debian9.9.rootfs.gz .
###解压debian9.9.rootfs.gz到SD卡


####其他

更改源：
vi /etc/apt/sources.list
deb http://ftp2.cn.debian.org/debian stretch main

```

当移植并运行Xorg时（或者其它吃内存软件），会报内存不足，需要配置`swap`缓解内存压力

- `Gparted`制作`linux swap`分区

  通过`Gparted`分配`1G`大小的`linux swap`分区

  在`/etc/fstab`中添加`swap`描述

  ```bash
  mmk0p3          swap            swap    defaults        0 0
  ```

- 系统运行后添加`swap`

  ```bash
  #从文件系统划分出512M大小空间
  dd if=/dev/zero of=/swapfile bs=1024 count=512k
  #格式化分区
  mkswap /swapfile
  #激活 Swap
  swapon /swapfile
  #配置/etc/fstab, 开机自动挂载
  /swapfile          swap            swap    defaults        0 0
  #赋予 Swap 文件适当的权限
  chown root:root /swapfile 
  chmod 0600 /swapfile驱动移植
  ```

###驱动的移植

```bash
#在linux内核移植后, 安装到制作的根文件系统中, depmod的使用
ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- make modules_install INSTALL_MOD_PATH=
```

### 桌面系统的移植

本来打算在`HOST`安装，但`Xorg`默认使用使用`HOST`的配置行不通，最终在`target`上通过`apt`安装了`lxde`

```bash
#最新的Xorg可以自动识别硬件,进行合理的Display和INPUT的配置
sudo apt install lxde
```

### 移植`SDLPAL`

参考教程

- [`SDL`介绍](https://blog.csdn.net/weixin_37921201/article/details/120440090)
- [`sdlpal`教程](https://gitee.com/sdlpal/sdlpal?_from=gitee_search)

基于SDL的仙剑奇侠传

```bash
#安装sdlpal
apt install libsdl2-dev
#进入源代码编译,下载官方游戏数据,将生成的程序放入其中执行
cd unix
HOST=arm-linux-gnueabihf- make
```

`SDL`可以基于`Xlib`，也可以基于直接`frambuff`

### 支持`alsa`软件混音

参考教程

- [alsa配置文件](https://blog.csdn.net/u010312436/article/details/47839229)

- [alsa使用](https://blog.csdn.net/xiaolong1126626497/article/details/105718239)

  ```bash
  # !default是默认声音设置, 系统默认使用dmixer软件混音
  # "hw:0,0" 表示第一个声卡的第一个设备
  # aplay -D plug:dmix 1.wav &
  
  pcm.!default {
  	type plug
  	slave.pcm "dmixer"
  }
   
  pcm.dmixer  {
   	type dmix
   	ipc_key 1024
   	slave {
  		pcm "hw:0,0"
  		period_time 0
  		period_size 1024
  		buffer_size 4096
  		rate 44100
  	}
  	bindings {
  		0 0
  		1 1
  	}
  }
  
  ctl.dmixer {
  	type hw
  	card 0
  }
  ```

### X桌面系统

`Linux`下的桌面系统使用`Xserver`和`Xclient`的模式，`Xserver`负责管理显示、输入等，`Xclient`也就是我们常说的桌面应用程序，通过和`Xserver`通信，完成显示等的输出和获取输入设备的信息。但本质上，`Xserver`就是一个普通的程序，只不过它遵循了`Xorg`协议罢了，而`Xclient`使用了`Xsever`提供的开发接口。不同`Linux`系统的配置方式可能略有区别需要各子对待，但原理相同。

参考教程

- [`xinit`及`X`介绍和启动过程](https://blog.csdn.net/qq_39101111/article/details/78728857)
- [`linux`移植`xserver`、`tslib`、`gtk`和桌面系统](https://blog.csdn.net/w6980112/article/details/47155691/)

#### `xinit`

`xinit`负责启动`Xserver`并管理`Xsession`

```bash
# xinit使用X11默认的配置启动Xserver并管理Xsession,并启动dwm桌面管理器
# 设置可以启动某个遵循Xorg协议的程序,只要启动好依赖的程序，便可以让该程序独享所有资源
xinit dwm
```

#### `xinput`

`xinput`负责管理所有的输入设备

```bash
# 禁用或启用某个输入设备
# xinput list查询所有输入设备
xinput disable 
xinput enable
```

####`startx`和`xinit`介绍

参考教程

- [`startx`和`xinit`使用和介绍](https://blog.csdn.net/qq_39101111/article/details/78728857)

##注意事项

- Usb-Hub只能有一个设备工作，直插的鼠标乱飘
  - 原因：开发板的microUSB口不能提供足够的驱动电流，带不起鼠标和Usb-Hub
  - 解决：推荐使用外部独立供电的Usb-Hub