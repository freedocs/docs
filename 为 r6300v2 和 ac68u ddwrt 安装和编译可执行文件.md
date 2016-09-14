## 系统环境

ddwrt 固件版本: Netgear R6300V2 DD-WRT v3.0-r29875M kongac (06/11/16)

编译系统： ubuntu 16.04 64 bit

安装二进制程序： n2n edge， kms server

## 确认可执行文件格式

将 ddwrt 固件中的 busybox 拷贝到本地， `file busybox` 可以看到 

```
busybox: ELF 32-bit LSB  executable, ARM, EABI5 version 1 (SYSV), dynamically linked (uses shared libs), corrupted section header size
```

再执行

```
readelf -d busybox
```

可以看到

```
  Tag        Type                         Name/Value
 0x00000001 (NEEDED)                     Shared library: [libgcc_s.so.1]
 0x00000001 (NEEDED)                     Shared library: [libc.so]
```

其调用了 `libc.so` ，在路由中检查 libc 版本

```
cd /lib/
./libc.so
```

可以看到 

```
musl libc
Version 1.1.11
Dynamic Program Loader
Usage: ./libc.so [options] [--] pathname [args]
```

所以 ddwrt 使用的是 [musl libc](http://www.etalabs.net/compare_libcs.html)，再查看 openwrt 的 [编译发布](https://downloads.openwrt.org/snapshots/trunk/bcm53xx/generic/)

从 `OpenWrt-SDK-bcm53xx_gcc-5.3.0_musl-1.1.14_eabi.Linux-x86_64.tar.bz2` 可以看到也是 `musl libc`，在 packages 目录下载 [aria2](https://downloads.openwrt.org/snapshots/trunk/bcm53xx/generic/packages/packages/aria2_1.24.0-1_bcm53xx.ipk)

改名为 `aria2_1.24.0-1_bcm53xx.tar.gz` ，解压缩，再解压缩包中的 `data.tar.gz` 得到 `data/usr/bin/aria2c` 文件。

将 `aria2c` 文件拷贝到 ddwrt `/jffs` 目录下，运行测试 `./aria2c --help` 查看结果

```
./aria2c --help
Usage: aria2c [OPTIONS] [URI | MAGNET | TORRENT_FILE | METALINK_FILE]...
Printing options tagged with '#basic'.
```

可以看到可以正确的执行，说明 openwrt 编译的文件可以直接被用于 ddwrt。

## 编译 n2n edge

下载交叉编译环境并解压

```
wget https://downloads.openwrt.org/snapshots/trunk/bcm53xx/generic/OpenWrt-SDK-bcm53xx_gcc-5.3.0_musl-1.1.14_eabi.Linux-x86_64.tar.bz2
tar xf OpenWrt-SDK-bcm53xx_gcc-5.3.0_musl-1.1.14_eabi.Linux-x86_64.tar.bz2
```

下载 n2n 源码

```
svn checkout https://svn.ntop.org/svn/ntop/trunk/n2n/n2n_v2/
```

配置环境变量

```
PATH=$PATH:$HOME/OpenWrt-SDK-bcm53xx_gcc-5.3.0_musl-1.1.14_eabi.Linux-x86_64/staging_dir/toolchain-arm_cortex-a9_gcc-5.3.0_musl-1.1.14_eabi/bin
export PATH
STAGING_DIRPATH=$HOME/OpenWrt-SDK-bcm53xx_gcc-5.3.0_musl-1.1.14_eabi.Linux-x86_64/staging_dir/toolchain-arm_cortex-a9_gcc-5.3.0_musl-1.1.14_eabi
export STAGING_DIRPATH
STAGING_DIR=$HOME/OpenWrt-SDK-bcm53xx_gcc-5.3.0_musl-1.1.14_eabi.Linux-x86_64/staging_dir/toolchain-arm_cortex-a9_gcc-5.3.0_musl-1.1.14_eabi
export STAGING_DIR

CFLAGS=-I$HOME/OpenWrt-SDK-bcm53xx_gcc-5.3.0_musl-1.1.14_eabi.Linux-x86_64/staging_dir/target-arm_cortex-a9_musl-1.1.14_eabi/usr/include
LDFLAGS=-L$HOME/OpenWrt-SDK-bcm53xx_gcc-5.3.0_musl-1.1.14_eabi.Linux-x86_64/staging_dir/target-arm_cortex-a9_musl-1.1.14_eabi/usr/lib

export CFLAGS
export LDFLAGS
```

添加 `$(LDFLAGS)` 到每一个编译参数， 即修改 `Makefile`

`CFLAGS+=$(DEBUG) $(OPTIMIZATION) $(WARN) $(OPTIONS) $(PLATOPTS) $(N2N_DEFINES)`
为
`CFLAGS+=$(DEBUG) $(OPTIMIZATION) $(WARN) $(OPTIONS) $(PLATOPTS) $(N2N_DEFINES) $(LDFLAGS)`


编译

```
make CC=arm-openwrt-linux-muslgnueabi-gcc LD=arm-openwrt-linux-muslgnueabi-ld
```

注意如果缺少 `openssl/aes.h` 等头文件，可以先在交叉环境中编译 nginx 来安装依赖。如果不怕麻烦要手动安装依赖，可以参考 [这里](https://forum.openwrt.org/viewtopic.php?id=57657)

```
./scripts/feeds update
./scripts/feeds install nginx
make package/feeds/packages/nginx/compile V=s
```

拷贝 n2n 目录下生成的 `supernode` 和 `edge` 文件到路由器



## 编译 vlcmsd

和 n2n 相同，下载源代码并解压后，在目录直接执行

```
make CC=arm-openwrt-linux-muslgnueabi-gcc LD=arm-openwrt-linux-muslgnueabi-ld
```

拷贝生成的 `vlmcsd` `vlmcs` 文件到路由即可。


## 编译 shellinabox

安装依赖并下载源码 

```
apt-get install git libssl-dev libpam0g-dev zlib1g-dev dh-autoreconf
git clone https://github.com/shellinabox/shellinabox
cd shellinabox
```

通过编译 `openssh-server-pam` 为交叉编译环境安装 `libpam` 依赖

```
./scripts/feeds install openssh-server-pam
make package/feeds/packages/openssh/compile V=s
```

配置

```
autoreconf -i
./configure
```

修改 `shellinabox/launcher.c` 文件

```
diff --git a/shellinabox/launcher.c b/shellinabox/launcher.c
index 2bac171..2df1f3f 100644
--- a/shellinabox/launcher.c
+++ b/shellinabox/launcher.c
@@ -81,6 +81,10 @@
 #include <sys/uio.h>
 #endif

+#ifndef TTYDEF_IFLAG
+#include <sys/ttydefaults.h>
+#endif
+
 #ifdef HAVE_UTIL_H
 #include <util.h>
 #endif
@@ -90,7 +94,8 @@
 #endif

 #ifdef HAVE_UTMPX_H
-#include <utmpx.h>
+//#include <utmpx.h>
+#undef HAVE_UTMPX_H
 #endif

 #if defined(HAVE_SECURITY_PAM_APPL_H)
```

编译

```
make CC=arm-openwrt-linux-muslgnueabi-gcc LD=arm-openwrt-linux-muslgnueabi-ld
```

拷贝生成的 `shellinaboxd`，在路由上执行

```
/usr/bin/shellinaboxd -t -s /:LOGIN --localhost-only --background=/var/run/shellinabox.pid
```

这样就在 `127.0.0.1:4200` 端口做了 `http shell` 的监听

通过 nginx 做 https 的转发

```
    location /shellinabox {
        auth_basic "Authentication required";
        auth_basic_user_file /etc/nginx/.dlpasswd;
        proxy_pass http://127.0.0.1:4200/;
    }

```

访问 `https://route.example.com/shellinabox` 就可以控制路由了

可以通过下边的 crontab 脚本管理服务

```
#!/bin/bash

# */1 * * * * /root/bin/shellinabox >> /var/log/shellinabox.log 2>&1

if [ $(/usr/bin/ps -ef|grep shellinaboxd|grep -v grep|wc -l) -eq 0 ];then
    /usr/bin/shellinaboxd -t -s /:LOGIN --localhost-only --background=/var/run/shellinabox.pid
    echo $(date) -- shellinaboxd started
fi
```

## 安装 transmission

新版本的 kong ddwrt 没有集成 transsmion， 但是仍然可以将其安装在 /jffs 目录下

下载文件

```
cd /jffs
curl -k -s http://downloads.openwrt.org/snapshots/trunk/bcm53xx/generic/packages/packages/transmission-daemon-openssl_2.92-3_bcm53xx.ipk > t.tar.gz
tar xzf t.tar.gz
tar xzf data.tar.gz
rm control.tar.gz debian-binary t.tar.gz data.tar.gz

curl -k -s https://downloads.openwrt.org/snapshots/trunk/bcm53xx/generic/packages/packages/transmission-web_2.92-3_bcm53xx.ipk > t.tar.gz
tar xzf t.tar.gz
tar xzf data.tar.gz
rm control.tar.gz debian-binary t.tar.gz data.tar.gz

curl -k -s https://downloads.openwrt.org/snapshots/trunk/bcm53xx/generic/packages/base/libevent2_2.0.22-1_bcm53xx.ipk >t.tar.gz
tar xzf t.tar.gz
tar xzf data.tar.gz
rm control.tar.gz debian-binary t.tar.gz data.tar.gz
```

执行 `transmission-daemon --help` 测试，检查是否输出正常。

运行

```
mkdir /mnt/usb
mount --bind /mnt/sdb1 /mnt/usb
/jffs/usr/bin/transmission-daemon -g /mnt/usb/transmission --logfile /jffs/log/transmission-daemon.log --pid-file /jffs/run/transmission-daemon.pid
```

crontab 脚本

```
#!/bin/sh

# file locaton: /jffs/bin/transmission

# crontab: */1 * * * * root /jffs/bin/transmission >> /jffs/log/transmission.log

LD_LIBRARY_PATH=/lib:/usr/lib:/jffs/lib:/jffs/usr/lib:/jffs/usr/local/lib:/mmc/lib:/mmc/usr/lib:/opt/lib:/opt/usr/lib
PATH=/bin:/usr/bin:/sbin:/usr/sbin:/jffs/sbin:/jffs/bin:/jffs/usr/sbin:/jffs/usr/bin:/mmc/sbin:/mmc/bin:/mmc/usr/sbin:/mmc/usr/bin:/opt/sbin:/opt/bin:/opt/usr/sbin:/opt/usr/bin
TRANSMISSION_WEB_HOME=/jffs/usr/share/transmission/web

export LD_LIBRARY_PATH
export PATH
export TRANSMISSION_WEB_HOME

if [ ! -d '/mnt/usb' ];then
    mount --bind /mnt/sdb1 /mnt/usb
fi

if [ ! -d '/mnt/usb' ];then
    echo ‘disk not mounted.’
    exit -1
fi

if [ $(/bin/ps|grep transmission-daemon|grep -v grep|wc -l) -eq 0 ];then
    /jffs/usr/bin/transmission-daemon -g /mnt/usb/transmission --logfile /jffs/log/transmission-daemon.log --pid-file /jffs/run/transmission-daemon.pid 
    echo $(date) -- transmission-daemon started
fi
```
