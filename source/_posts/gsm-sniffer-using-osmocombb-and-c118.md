---
title: osmocombb+c118进行GSM短信嗅探实验
date: 2017-08-07 15:27:53
tags:
---

## 0x01 实验环境准备

* ubuntu 12.04.5 desktop i386
* 摩托罗拉 C118 - 淘宝价35.00元
* FT232RL模块 - 淘宝价25.50元
* GSM Sniffer 耳机音频插头转杜邦转接线 - 淘宝价4.00元

注意：这里的实验环境最好选择ubuntu 12.04.5 desktop i386，其他版本容易在编译OsmocomBB步骤时无法通过(不过也可能是我当时其他几步哪里有问题导致)。

这里我们需要用转接线将C118与FT232RL连接起来，再通过FT232RL的USB口插入到笔记本上完成连接。

![c118](https://ae01.alicdn.com/kf/HTB1Z3eOe8Cw3KVjSZFlq6AJkFXaA.jpg)

![wire](https://ae01.alicdn.com/kf/HTB1memIe8iE3KVjSZFMq6zQhVXai.jpg)

![ft232](https://ae01.alicdn.com/kf/HTB1u_mQeW1s3KVjSZFAq6x_ZXXaA.jpg)

因为这个实验是前段时间完成的，中间安装的步骤都没有截图下来，所以理解和操作起来可能会有点懵逼。这里可以推荐youtube上的一个视频，也可以跟着他的做，就是画质很渣，看看大致怎样一个流程就是了：[c118-osmocombb-gsm sniffer-嗅探短信](https://www.youtube.com/watch?v=hrXVWRAqJQU)

## 0x02 依赖软件包的安装

``` bash
sudo aptitude install libusb-0.1-4 libpcsclite1 libccid pcscd libtool shtool autoconf git-core pkg-config make gcc build-essential libgmp3-dev libmpfr-dev libx11-6 libx11-dev texinfo flex bison libncurses5 libncurses5-dbg libncurses5-dev libncursesw5 libncursesw5-dbg libncursesw5-dev zlibc zlib1g-dev libmpfr4 libmpc-dev libpcsclite-dev subversion libosip2-dev libortp-dev libusb-1.0-0-dev g++ sqlite3 libsqlite3-dev erlang libreadline6-dev libncurses5-dev libtalloc-dev   wireshark
```

这些依赖关系就得解决好，有多少装多少，不然到后面手动一个个修复依赖很麻烦。

## 0x03 手动安装talloc

``` bash
cd ~/Documents
wget https://www.samba.org/ftp/talloc/talloc-2.1.7.tar.gz
tar -zxvf talloc-2.1.7.tar.gz
cd talloc-2.1.7/
./configure
make
sudo make install
```

安装好上一步的依赖之后，我们还需要手动安装talloc，否则貌似是在编译libosmocore的过程会出现缺少该依赖的问题。不过你也可以到时再手动安装，这里我就统一先把都安装好吧。

## 0x04 搭建ARM交叉编译环境

这里我们就统一建立了一个名为osmocombb的文件夹作为我们这次实验的总文件夹把。我们随后会将下载好的osmocom-bb项目、libosmocore等都放在这个文件夹里。

``` bash
cd ~
mkdir osmocombb
cd osmocombb
mkdir build install src
wget http://bb.osmocom.org/trac/raw-attachment/wiki/GnuArmToolchain/gnu-arm-build.3.sh
cd src
wget http://ftp.gnu.org/gnu/gcc/gcc-4.8.2/gcc-4.8.2.tar.bz2
wget http://ftp.gnu.org/gnu/binutils/binutils-2.21.1a.tar.bz2
wget ftp://sources.redhat.com/pub/newlib/newlib-1.19.0.tar.gz
cd ..
chmod +x gnu-arm-build.3.sh
./gnu-arm-build.3.sh
```

这个shell脚本会执行相当长的时间，所以会花好些时间。而且上述命令执行时，可能**newlib-1.19.0.tar.gz**下载很慢，那么你就需要通过google去一些站点直接下载回来。

接下来便是**添加环境变量**：将ARM编译器路径添加到环境变量里头去

``` bash
cd install/bin
pwd #复制打印出的完整路径
#现在编辑~/.bashrc
cd ~
vim ./.bashrc
#在最后一行添加
export PATH=$PATH:/home/vancir/Documents/osmocombb/install/bin
```

这里的path是/install/bin的路径，要根据自己的完整路径来设置环境变量！

设置成功与否，我们可以简单地通过在终端输入**arm**后按两次**Tab**键来查看是否输出了对应的编译器列表

## 0x05 编译

## 下载项目源代码

``` bash
cd ~/Documents/osmocombb
git clone git://git.osmocom.org/libosmocore.git
git clone git://git.osmocom.org/osmocom-bb.git
git clone git://git.osmocom.org/libosmo-dsp.git
```

接下来的编译

## 编译libosmocore

``` bash
cd libosmocore/
autoreconf -i
./configure
make
sudo make install
cd ..
```

## 编译osmocombb

``` bash
cd osmocom-bb
git checkout --track origin/luca/gsmmap
cd src/
make
```

整个的编译过程花费的时间也不短。但是这里的编译过程很关键，一定要认真查看最后的输出信息，看是否编译成功。否则很容易在**osmocombb编译**这一步卡住，那么你就需要重新进行编译了。

## 0x06 修改问题文件

在编译完成后，我们还需要修改一些问题文件，否则在后续的扫描步骤会出现错误。

``` bash
vim /home/vancir/Documents/osmocombb/osmocom-bb/src/target/firmware/board/compal/highram.lds
vim /home/vancir/Documents/osmocombb/osmocom-bb/src/target/firmware/board/compal/ram.lds
vim /home/vancir/Documents/osmocombb/osmocom-bb/src/target/firmware/board/compal_e88/flash.lds
vim /home/vancir/Documents/osmocombb/osmocom-bb/src/target/firmware/board/compal_e88/loader.lds
vim /home/vancir/Documents/osmocombb/osmocom-bb/src/target/firmware/board/mediatek/ram.lds
```

在上述文件中找到 **KEEP(\*(SORT(.ctors)))** 之后再下一行添加如下代码：**KEEP(*(SORT(.init_array)))**

等全部修改完毕，再进入到 osmocom-bb/src 重新编译

``` bash
make -e CROSS_TOOL_PREFIX=arm-none-eabi-
```

如果一路下来都没有遇到什么问题，那么恭喜你，软件部分已经准备好了。接下来我们进入硬件部分

## 0x07 硬件准备

这里不需要进行硬件改造，只需要简单地使用**转接线**将**C118**与**FT232RL**模块连接即可。（这里C118不需要开机，也先不要去开机）

值得注意的是，黑线接**GND**，红线接**TXD**，绿线(白线)接**RXD**，一般都有黑线和红线来区分，但是第三根线就可能是绿线或白线了，看店家。

连接好后，将FT232RL的USB头插入ubuntu 12.04.4即可

![ft232_wire](https://ae01.alicdn.com/kf/HTB1ZWiHe21H3KVjSZFBq6zSMXXaE.jpg)

## 0x08 刷入固件

我们进入osmocom-bb/src/host/osmocon目录下，准备刷入固件

``` bash
cd src/host/osmocon
sudo ./osmocon -m c123xor -p /dev/ttyUSB0 ../../target/firmware/board/compal_e88/layer1.compalram.bin
```

执行上述命令后，终端始终处于一种等待状态，这时我们按**一下**C118的开机键即可

按下开机键后，终端开始输出信息，当提示**THIS FIRMWARE WAS COMPILED WITHOUT TX SUPPORT!!!**时表明固件刷入完毕。

## 0x09 扫描基站信息

固件刷入完毕后，新建一个终端，进入 osmocom-bb/src/host/layer23/src/misc 目录

``` bash 
cd osmocom-bb/src/host/layer23/src/misc
sudo ./cell_log -O
```

终端会输出很多基站信息格式大致如下：

``` bash 
<000e> cell_log.c:248 Cell: ARFCN=69 PWR=-83dB MCC=460 MNC=00 (China, China Mobile)
```

执行上述命令后紧接着执行

``` bash 
sudo ./ccch_scan -i 127.0.0.1 -a ARFCN
```

其中ARFCN就是我们先前运行**cell_log**时输出的ARFCN标号，比如上述基站信息中**ARFCN=69**，那我们就是

``` bash 
sudo ./ccch_scan -i 127.0.0.1 -a 69
```

## 0x0A GSM嗅探

使用**wireshark**来帮我们抓取过滤数据包

``` bash 
sudo wireshark -k -i lo -f 'port 4729'
```

命令会启动**wireshark**，之后我们在**filter**处输入**gsm_sms**便可筛选出我们的gsm嗅探数据。

稍等片刻，就能嗅探到该基站的短信啦。我们可以通过wireshark得到短信的**发信人手机号**、**接信者手机号**，**短信内容**

这就是GSM短信嗅探的全部内容，下次我们再来介绍**如何在GSM短信嗅探的基础上给身边的人发送伪造短信**。

有兴趣的小伙伴可以去了解下伪基站，不过私自搭建伪基站并使用是违法的，而GSM短信嗅探的话，其实嗅探范围很局限，而且也仅能针对移动和联通用户有效。

不管怎样，一定不要将技术应用于非法途径！

PS: 参考资料列举有点麻烦，就不列举了，搜索关键字是【GSM 短信嗅探】【GSM Sniffer】【osmocombb】【osmocombb+c118】