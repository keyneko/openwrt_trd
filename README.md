# 下载固件
https://archive.openwrt.org/releases/17.01.7/targets/ar71xx/generic/lede-17.01.7-ar71xx-generic-tl-wr703n-v1-squashfs-sysupgrade.bin
https://openwrt.org/toh/tp-link/tl-wr703n

# 更新软件包
http://downloads.lede-project.org/releases/17.01.7/targets/ar71xx/generic/packages/
https://mirror-03.infra.openwrt.org/
```bash
opkg update
opkg list
opkg list-installed
opkg list-upgradable
opkg search

# 配置软件安装目录
mount /dev/sda /mnt/sda
cd /mnt/sda
mkdir opkg
vi /etc/opkg.conf
  dest usb /mnt/sda/opkg

# usb挂载
opkg -d usb install kmod-usb-storage block-mount kmod-fs-ext4 e2fsprogs

# python
opkg -d usb install python
opkg -d usb install python-pip
opkg -d usb install python3
opkg -d usb install python3-pip
opkg -d usb install python-pyserial


# 修改python默认链接指向
root@OpenWrt:~# ls -l /mnt/sda/opkg/usr/bin/python*
lrwxrwxrwx    1 root     root             9 Feb 26 07:21 /mnt/sda/opkg/usr/bin/python -> python2.7
lrwxrwxrwx    1 root     root             9 Feb 26 07:21 /mnt/sda/opkg/usr/bin/python2 -> python2.7
-rwxr-xr-x    1 root     root          4117 Sep 23  2019 /mnt/sda/opkg/usr/bin/python2.7
lrwxrwxrwx    1 root     root             9 Feb 26 08:11 /mnt/sda/opkg/usr/bin/python3 -> python3.6
lrwxrwxrwx    1 root     root            16 Feb 26 08:15 /mnt/sda/opkg/usr/bin/python3-config -> python3.6-config
-rwxr-xr-x    1 root     root          4122 Sep 23  2019 /mnt/sda/opkg/usr/bin/python3.6
-rwxr-xr-x    1 root     root          3852 Sep 23  2019 /mnt/sda/opkg/usr/bin/python3.6-config

rm /mnt/sda/opkg/usr/bin/python
ln -s /mnt/sda/opkg/usr/bin/python3.6 /mnt/sda/opkg/usr/bin/python
ln -s python3.6 /mnt/sda/opkg/usr/bin/python


# pip换源
touch /etc/pip.conf
vi /etc/pip.conf

[global]
timeout = 6000
index-url = https://pypi.tuna.tsinghua.edu.cn/simple/ 
trusted-host = pypi.tuna.tsinghua.edu.cn

# 安装Django
python -m pip install Django --verbose
python -m pip install "Django<3.3"
python -m pip install "Django==2.0.7" > django_install.log 2>&1 &
python -m pip install "Django<2" > django_install.log 2>&1 &
```

# 预编译django内存溢出，使用sdk交叉编译ipk包
```bash
https://downloads.openwrt.org/releases/17.01.7/targets/ar71xx/generic/lede-sdk-17.01.7-ar71xx-generic_gcc-5.4.0_musl-1.1.16.Linux-x86_64.tar.xz
make package/django/compile V=s

scp ~/Desktop/django_1.8.12-1_mips_24kc.ipk root@10.168.1.115:/mnt/sda
opkg -d usb install django_1.8.12-1_mips_24kc.ipk

vi /etc/profile
export PYTHONPATH=$PYTHONPATH:/mnt/sda/opkg/usr/lib/python2.7/site-packages
export LD_LIBRARY_PATH="/mnt/sda/opkg/usr/lib:/mnt/sda/opkg/lib"
export PATH=/usr/bin:/usr/sbin:/bin:/sbin:/mnt/sda/opkg/usr/bin:/mnt/sda/opkg/usr/sbin

ln -s /mnt/sda/opkg/usr/lib/python2.7/site-packages/django/bin/django-admin.py /mnt/sda/opkg/usr/bin/django-admin

django-admin --version
django-admin startproject myproject
cd myproject
python2 manage.py startapp myapp
python2 manage.py makemigrations
python2 manage.py migrate
python2 manage.py runserver
```

# 内存带不动django, 改用bottle
```bash
opkg -d usb install python3-bottle_0.12.8-1_mips_24kc.ipk
vi /etc/init.d/myapp
```

```sh
#!/bin/sh /etc/rc.common

START=99
STOP=10

start() {
    echo "Starting myapp"
    python /mnt/sda/bottle/app.py >/mnt/sda/bottle/myapp.log 2>&1 &
}

stop() {
    echo "Stopping myapp"
    killall python
}
```

```bash
chmod +x /etc/init.d/myapp
/etc/init.d/myapp enable
/etc/init.d/myapp start
/etc/init.d/myapp stop
```


# 使用imagebuilder可以快速构建固件
https://downloads.openwrt.org/releases/17.01.7/targets/ar71xx/generic/lede-imagebuilder-17.01.7-ar71xx-generic.Linux-x86_64.tar.xz


# 从openwrt源码构建固件
```bash
git clone -b lede-17.01  https://github.com/openwrt/openwrt
cd openwrt
git checkout lede-17.01

feeds.conf.default
  src-git base https://git.lede-project.org/source.git;v17.01.7
  src-git packages https://git.lede-project.org/feed/packages.git^545d2fadd7245783e40f235fe2c5d8c3ab1549cd
  src-git luci https://git.lede-project.org/project/luci.git^71e2af4f51567061600840040508d642120a8532
  src-git routing https://git.lede-project.org/feed/routing.git^af18086ba8de36e7fbc696cb39f634f6a7ea34de
  src-git telephony https://git.lede-project.org/feed/telephony.git^31398a3759c99bdb2dca359b78e25c17b6dd434a

sudo apt-get -y install build-essential asciidoc binutils bzip2 gawk gettext git libncurses5-dev libz-dev patch python3 python2.7 unzip zlib1g-dev lib32gcc1 libc6-dev-i386 subversion flex uglifyjs git-core gcc-multilib p7zip p7zip-full msmtp libssl-dev texinfo libglib2.0-dev xmlto qemu-utils upx libelf-dev autoconf automake libtool autopoint device-tree-compiler g++-multilib antlr3 gperf wget curl swig rsync

./scripts/feeds update -a
./scripts/feeds install -a
make download

make menuconfig
选择 Target System 为 Atheros AR7xxx/AR9xxx
选择 Subtarget 为 Generic
选择 Target Profile 为 TP-LINK TL-WR703N

make kernel_menuconfig
  Kernel type -> [*] MIPS FPU Emulator

# 4M闪存加载太多东西会报错，修改成8M
vi target/linux/ar71xx/image/tp-link.mk
define Device/tl-wr703n-v1
    $(Device/tplink-8mlzma)
    DEVICE_TITLE := TP-LINK TL-WR703N
    DEVICE_PACKAGES := kmod-usb-core kmod-usb2
    BOARDNAME := TL-WR703N
    DEVICE_PROFILE := TLWR703
    TPLINK_HWID := 0x07030101
    CONSOLE := ttyATH0,115200

make V=s
```

# UCI配置无线网络
```bash
vi /etc/config/network
uci set network.lan.gateway='192.168.1.1'
uci set network.lan.dns='192.168.1.1'
uci commit network
/etc/init.d/network restart
```

# Samba网络共享
```bash
opkg install samba36-server
opkg install luci-app-samba

vi /etc/config/samba
config samba
        option name 'Lede'
        option workgroup 'WORKGROUP'
        option description 'Lede'
        option homes '1'

config sambashare
        option path '/mnt/sda'
        option name 'sda'
        option guest_ok 'yes'
        option create_mask '777'
        option dir_mask '777'
        option read_only 'no'

vi /etc/samba/smb.conf.template
[global]
  netbios name = |NAME|
  display charset = |CHARSET|
  interfaces = |INTERFACES|
  server string = |DESCRIPTION|
  unix charset = |CHARSET|
  workgroup = |WORKGROUP|
  browseable = yes
  deadtime = 30
  domain master = yes
  encrypt passwords = true
  enable core files = no
  guest account = nobody
  guest ok = yes
  #invalid users = root
  local master = yes
  load printers = no
  map to guest = Bad User
  max protocol = SMB2
  min receivefile size = 8192
  null passwords = yes
  obey pam restrictions = yes
  os level = 20
  passdb backend = smbpasswd
  preferred master = yes
  printable = no
  security = user
  smb encrypt = disabled
  smb passwd file = /etc/samba/smbpasswd
  socket options = TCP_NODELAY SO_RCVBUF=960000 SO_SNDBUF=960000
  syslog = 2
  use sendfile = yes
  use mmap = yes
  writeable = yes
  disable spoolss = yes
  host msdfs = no
  strict allocate = No

/etc/init.d/samba restart
```

# luci中文语言包
```bash
opkg install luci-base luci-i18n-base-zh-cn
```

# 安装shadowsocks-libev
```bash
opkg install luci-app-shadowsocks-libev luci-i18n-shadowsocks-libev-zh-cn
opkg install shadowsocks-libev

# 版本太旧，不支持新有服务端加密方式，放弃
opkg remove shadowsocks-libev shadowsocks-libev-mbedtls --force-removal-of-dependent-packages 
```



```
# ----------------------------------------------------------
System
Hostname  OpenWrt
Model TP-Link TL-WR703N v1
Firmware Version  LEDE Reboot 17.01.7 r4030-6028f00df0 / LuCI lede-17.01 branch (git-19.167.54478-71e2af4)
Kernel Version  4.4.182
Local Time  Mon Feb 26 06:41:59 2024
Uptime  0h 9m 38s
Load Average  0.01, 0.07, 0.06
```
