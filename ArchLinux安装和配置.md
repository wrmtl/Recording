# 制作安装镜像U盘

# 镜像系统配置

## 联网

#### 虚拟机或网线

``` bash
dhcpcd
```

#### WiFi

```bash
systemctl start wpa_supplicant.service
nmcli dev wifi list
nmcli dev wifi connect "ssid" password "passwd"
```

####常见问题

```bash
#查看网卡状态
ip link
#开启网卡
ip link set 网卡名 up
#rfkill解锁
rfkill list
rfkill unblock wifi
```

####更新系统时间

```bash
timedatectl set-ntp true
```

#### 选择镜像源

```bash
vim /etc/pacman.d/mirrorlist
```

# 系统制作

####硬盘分区

输入 ``` fdisk -l ``` 指令查看分区情况

####创建根分区

```bash
#操作磁盘创建分区
fdisk /dev/sda
# n 创建新分区，p 查看分区，w 保存
#生成了/dev/sda1分区
```

####格式化分区

```bash
mkfs.ext4 /dev/sda1
```

#### 挂载分区

```bash
mount /dev/sda1 /mnt
```

#### 下载基本包

```bash
pacstrap /mnt base base-devel linux linux-firmware dhcpcd
```

#### 配置Fstab

```bash
genfstab -L /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab # 查看文件系统信息
```

#### 切换root设置时区

```bash
arch-chroot /mnt
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc
```

#### 安装基本软件

```bash
pacman -S vim dialog wpa_supplicant ntfs-3g networkmanager netctl
```

#### 设置语言

```bash
vim /etc/locale.gen
#生成配置文件
locale-gen
#编辑配置文件
#输入LANG=en_US.UTF-8
vim /etc/locale.conf
```

#### 设置主机名

```bash
vim /etc/hostname
#输入自己喜欢的主机名ahostname
```

#### 配置hosts

```bash
vim /etc/hosts  # 注意不是host是hosts
#添加
#127.0.0.1	localhost
#::1		localhost
#127.0.1.1	ahostname.localdomain	ahostname
#可以定义其它主机映射，可用于访问github
```

#### 配置root密码

```bash
passwd
```

#### 安装CPU微码

```bash
#intel
pacman -S intel-ucode
#amd
pacman -S amd-ucode
```

#### Bootloader

```bash
pacman -S os-prober ntfs-3g
pacman -S grub
grub-install --target=i386-pc /dev/sda # 对应自己的盘
grub-mkconfig -o /boot/grub/grub.cfg
vim /boot/grub/grub.cfg # 检查
```

#### 最后一步

```bash
exit
umount /mnt
reboot
```

#系统安装后基本配置

#### 联网

#### 创建交换文件

```bash
#也可以使用dd命令
#dd if=/dev/zero of=/swapfile bs=1M count=512 status=progress 
fallocate -l 512M /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
vim /etc/fstab
#添加文件系统信息
#/swapfile none swap defaults 0 0
```

#### 新建用户

```bash
useradd -m -G wheel aname
passwd aname
```

配置sudo

```bash
pacman -S sudo
ln -s /usr/bin/vim /usr/bin/vi
visudo
```

把%wheel ALL=(ALL)ALL 的注释取消

#### 配置源

```bash
sudo vim /etc/pacman.conf
#取消multilib的注释
#添加
#[archlinuxcn]
#Server = https://repo.archlinuxcn.org/$arch
#清华源
#[archlinuxcn]
#Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
```

更新源，下载密钥

```bash
sudo pacman -Syy
sudo pacman -S archlinuxcn-keyring
```

# 常用软件安装及配置

#### picom

```bash
sudo pacman -S picom
```

picom配置文件

```bash
sudo cp /etc/xdg/picom.conf .config/
#配置文件
#inactive-opacity = 0.5;
#active-opacity = 0.8;
#vsync = false; # 高刷新率屏幕会有问题
```

#### 安装蓝牙

```bash
sudo pacman -S bluez bluez-libs bluez-utils pulseaudio-bluetooth pavucontrol pulseaudio-alsa 
yay -S bluez-firmware
modinfo btusb
sudo systemctl enable bluetooth
```

开机启动

```bash
sudo vim /etc/bluetooth/main.conf
#AutoEnable=true
#ControllerMode=bredr
```

#### 连接蓝牙

确保

```bash
pulseaudio --start
```

然后启动蓝牙服务

```bash
bluetoothctl
power on 
agent on
scan on
pair 00:10:20:30:40:50 # 配对
trust 00:10:20:30:40:50
connect 00:10:20:30:40:50 # 连接
```

#### 声音设置

```bash
sudo pacman -S alsa alsa-utils kmix pulsemixer
alsamixer
```

#### oh-my-zsh

安装

```bash
sudo pacman -S zsh
```

配置系统shell

```bash
#添加/bin/zsh
vim /etc/shells
chsh -s /bin/zsh
#查看系统shell
echo $SHELL
```

配置文件

```bash
#编辑~/.zshrc文件
vim ~/.zshrc
#修改主题
#ZSH_THEME="agnoster"
```

注意事项

chsh切换为zsh后，用户级.bashrc失效了，需要将配置转移到zsh的配置文件

#### 中文输入法

```bash
sudo pacman -S fcitx fcitx-im fcitx-googlepinyin fcitx-configtool
sudo vim ~/.pam_environment
```

fcitx-configtool用于输入法配置

环境中加入

```bash
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
```

#### 代理软件

KDE支持的系统proxy配置

终端命令行

```bash
#配置系统全局环境
sock_proxy=
http_proxu=
https_proxy=
all_proxy=
#或者使用软件proxychains
```

v2raya透明代理

# KDE桌面

#### 新建用户

需要创建普通用户用于登录

#### 配置源

#### 显卡驱动

https://blog.csdn.net/qq_25675517/article/details/120733383

#### 安装KDE

```bash
sudo pacman -S xorg plasma kde-applications sddm network-manager-applet
```

启动服务，重启进入kde

```bash
sudo systemctl enable sddm
sudo systemctl disable netctl
sudo systemctl enable NetworkManager
```

安装中文字体

```bash
sudo pacman -S wqy-microhei wqy-microhei-lite wqy-bitmapfont wqy-zenhei ttf-arphic-ukai ttf-arphic-uming adobe-source-han-sans-cn-fonts adobe-source-han-serif-cn-fonts noto-fonts-cjk
```

安装yay

```bash
sudo pacman -S git
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
#可修改goproxy加速
#alias go_up="export GO111MODULE=on && export GOPROXY=https://goproxy.cn"
```

把语言改成中文

```bash
sudo vim /etc/locale.conf
改成 LANG=zh_CN.UTF-8
```

# DWM桌面

#### 安装 xorg + xorg-xinit

```bash
sudo pacman -S xorg xorg-xinit
```

#### 安装yay

#### 安装字体

```bash
sudo pacman -S ttf-fira-code noto-fonts-emoji wqy-microhei
yay -S ttf-symbola nerd-fonts-fira-code
```

#### 安装终端模拟器

##### st

从suckless下载源码

```bash
git clone https://git.suckless.org/st
cd st
```

配置config.def.h

修改comfig.mk

```bash
X11INC = /usr/include/X11
X11LIB = /usr/lib/X11
```

编译并安装

```bash
make
rm -rf config.h && sudo make clean install
```

打patches

patches放在st/pathces下

```bash
cd st
patch < patches/a.diff
```

#####alacritty

```bash
sudo pacman -S alacritty
```

alacrity的配置文件为~/.config/alacritty/alacritty.yml

```bash
cp /usr/share/doc/alacritty/example/alacritty.yml ~/.config/alacritty/alacritty.yml
```

#####将虚拟终端配置到dwm的config文件

####安装DWM

下载源码

```bash
git clone https://git.suckless.org/dwm
```

配置config.def.h

修改config.mk

编译并安装

```bash
make
rm -rf config.h && sudo make clean install
```

#####打patches

patches放在dwm/pathces下

```bash
cd dwm
patch < patches/a.diff
```

#####添加命令+定义快捷键

编辑config.def.h

添加flameshot截图命令和快捷键

```c
static const char *flameshot[] = {"flameshot", "gui", NULL};
{ MODKEY,	XK_s,	spawn,	{.v = flameshot} },
```

#####推荐patches

```bash
dwm-alpha-20201019-61bb8b2.diff
dwm-autostart-20210120-cb3f58a.diff
dwm-barpadding-20211020-a786211.diff
dwm-uselessgap-20200719-bb2e722.diff
```

#####状态栏右侧显示信息

```bash
xsetroot -name "hello dwm"
```

可配合dwm-autostart-20210120-cb3f58a.diff使用

#####使用autostart补丁

dwm.c下代码

```c
void
runAutostart(void) {
  system("cd ~/.dwm/scripts; ./autostart.sh &")
}
```

编写autostart.sh，如：

```bash
#!/bin/sh

dwm_date () {
  date '+%Y年%m月%d日 %a %H:%M'
}

while true
do
  xsetroot -name "$(dwm_date)"
  sleep 1
done
```

默认文件位置为.dwm/autostart.sh

#### 安装dmenu

从suckless下载源码编译，在dwm中配置

可使用rofi替代

#### 配置startx

如果有kde的sddm，需要先禁用sddm

```bash
sudo systemctl disable sddm
```

配置xinitrc

```bash
cp /etc//etc/X11/xinit/xinitrc ~/.xinitrc
sudo vim ~/.xinitrc
```

注释以下

```bash
#twm &
#xclock -geometry 50x50-1+1 &
#xterm -geometry 80x50+494+51 &
#xterm -geometry 80x20+494-0 &
#exec xterm -geometry 80x66+0+0 -name logini
```

添加

```bash
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
#fcitx &
#sleep 2
exec dwm
```

也可以添加其它的启动程序，启动显卡驱动等，或者系统环境配置等（也可以配置到autostart.sh）

重启，使用```startx```进入桌面

##### 注意事项

配置.dwm/autostart.sh

```bash
#!/bin/zsh
#美化
picom &
#壁纸
feh --bg-fill --no-fehbg --randomize ~/.dwm/wallpaper/*  &
#输入法
fcitx &
#虚拟机沟通
VBoxClient-all
```

#### 安装vim

#####系统剪切板

如果没有系统剪切板功能可安装```gvim```

可通过vim -v查询支持功能

#####配置文件.vimrc

```bash
set hlsearch	#高亮搜索
syntax on		#语法检查
set cursorline	#鼠标线
set number		#行号
set incsearch	#
set mouse=a		#鼠标可用
```

other

```bash
" 语法高亮度显示
syntax on

" 设置行号
set nu

"防止中文注释乱码
set fileencoding=utf-8
set fenc=utf-8
set fencs=utf-8,usc-bom,euc-jp,gb18030,gbk,gb2312,cp936,big－5                    
set enc=utf-8
let &termencoding=&encoding

"设置字体
set guifont=Monospace\ 13

" 设置tab4个空格
set tabstop=4
set expandtab

"程序自动缩进时候空格数
set shiftwidth=4

"退格键一次删除4个空格
set softtabstop=4
autocmd FileType make set noexpandtab

" 在编辑过程中，在右下角显示光标位置的状态行
set ruler

" 搜索忽略大小写 
set ignorecase 

" vim使用自动对起，也就是把当前行的对起格式应用到下一行
set autoindent

" 依据上面的对起格式，智能的选择对起方式，对于类似C语言编写上很有用
set smartindent

" 在状态列显示目前所执行的指令
set showcmd

" 设置颜色主题
colorscheme darkblue

set nocompatible
set backspace=indent,eol,start

```



##### 常用命令

```bash
#搜索
/searchword
n	#下一个搜索项
#剪切
dd
ndd
#删除首字母
x
nx
#复制
yy
nyy		#复制当前行开始的n行
#黏贴
p
#向系统剪切板
"+命令
!wq		#强制保存退出
```

#### 系统常用命令

#####tar

```bash
# z gzip格式文件，.txz(.tar.xz)为默认压缩格式不需要制定
# v 显示过程
# f 制定备份文件
# c 创建备份文件
# x 从备份文件中还原
tar -xzvf test.tgz	#解压test.tgz
tar -czvf test.tgz test		#test文件夹压缩
# --exclude=path/file	排除path/file文件
```

#虚拟机

### VirtualBox

```bash
sudo pacman -S virtualbox-guest-utils
sudo systemctl enable vboxservice.service
```

使用DWM时，需要在启动文件添加

```bash
VBoxClient-all
```

#####虚拟机调整分辨率等功能

安装上述必要库后开启

#####添加USB

官网下载Extern Package，并在虚拟机里安装

虚拟机设置配置USB设备的类型

虚拟机内挂在USB设备

#####添加共享文件夹

虚拟机配置共享文件夹：主机文件夹和虚拟机内挂载点

虚拟机内：

```bash
#将用户添加到vboxsf用户组
useradd -m -G vboxsf yourusername
#挂载共享文件夹
sudo mount -t vboxsf ShareName ShareMountPoint
```

