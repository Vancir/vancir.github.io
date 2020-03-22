---
title: TaintDroid动态污点分析工具部署
date: 2017-11-23 15:28:36
tags:
---

TaintDroid是一款著名的Android动态污点分析工具. 我的安装环境是**ubuntu 16.04 x64**

## 安装 JDK 6u45

TaintDroid仅支持jdk6及以下的版本, 如果有装高版本的jdk的话, 需要进行替换

``` bash
cd ~/Downloads
proxychains wget http://85-207-0-21.static.bluetone.cz/java/1.6.0_45/jdk-6u45-linux-x64.bin
sudo mkdir -p /usr/local/java  
cd /usr/local/java  
sudo cp ~/Downloads/jdk-6u45-linux-x64.bin .
sudo chmod +x jdk-6u45-linux-x64.bin
```

我之前是解压jdk8到/usr/local底下, 然后将路径添加到~/.zshrc里, 因此我自己还得将这里的添加环境变量的几行代码注释掉. 

``` bash
sudo vim /etc/profile
    ...
    JAVA_HOME=/usr/local/java/jdk1.6.0_45    
    PATH=$PATH:$HOME/bin:$JAVA_HOME/bin    
    export JAVA_HOME    
    export PATH  
```

然后配置JDK和JRE的位置

``` bash
sudo update-alternatives --install /usr/bin/java java /usr/local/java/jdk1.6.0_45/bin/java 1
sudo update-alternatives --install /usr/bin/javac javac /usr/local/java/jdk1.6.0_45/bin/javac 1   
sudo update-alternatives --install /usr/bin/javaws javaws /usr/local/java/jdk1.6.0_45/bin/javaws 1 
```

配置Oracle为系统默认JDK/JRE：

``` bash
sudo update-alternatives --set java /usr/local/java/jdk1.6.0_45/bin/java    
sudo update-alternatives --set javac /usr/local/java/jdk1.6.0_45/bin/javac    
sudo update-alternatives --set javaws /usr/local/java/jdk1.6.0_45/bin/javaws  
source /etc/profile
```

然后重新打开一个终端, 检测java版本

``` bash
java -version 
    java version "1.6.0_45"
    Java(TM) SE Runtime Environment (build 1.6.0_45-b06)
    Java HotSpot(TM) 64-Bit Server VM (build 20.45-b01, mixed mode)
```

## 安装一些必要的软件包

``` bash 
sudo apt-get install git-core gnupg flex bison gperf build-essential zip curl \
    zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 lib32ncurses5-dev x11proto-core-dev \ 
    libx11-dev lib32z-dev ccache libgl1-mesa-dev libxml2-utils xsltproc unzip libswitch-perl  
```

## 下载Android源码

``` bash
mkdir -p  ~/Documents/code/Android
mkdir -p  ~/Documents/code/Android/repo
mkdir -p  ~/Documents/code/Android/aosp
cd ~/Documents/code/Android/repo
curl https://storage-googleapis.proxy.ustclug.org/git-repo-downloads/repo > repo  
chmod a+x repo  
cd ../aosp
```

下载repo工具及初始化repo

``` bash
sudo apt-get install phablet-tools  
proxychains repo init -u git://mirrors.ustc.edu.cn/aosp/platform/manifest -b android-4.3_r1  
repo sync  #同步源码需要很长时间
```

## 编译源码

``` bash
cd /aosp/build  
sudo bash envsetup.sh
lunch #模拟器选择aosp_arm-eng
sudo make -j4 #-j后的数字根据机器的CPU核数配置, CPU核数的一或两倍   
emulator #确保编译无问题
```

这里的**envsetup.sh**用于配置环境变量, 成功配置好环境变量时才可以使用lunch命令, 必须注意的是, 这里的**envsetup.sh**必须在bash下运行, 像我用的是zsh, 因此就无法运行这个envsetup.sh, 建议之后的命令都在**bash**下运行. 

从zsh切换到bash也很简单, 就是输入**bash**命令即可

注意在make的时候, 可能会出现Make的版本问题, 比如我的是4.1版本就会无法编译, 需要使用3.81或3.82版本的make进行编译. 

``` bash 
build/core/main.mk:45: ********************************************************************************
build/core/main.mk:46: *  You are using version 4.1 of make.
build/core/main.mk:47: *  Android can only be built by versions 3.81 and 3.82.
build/core/main.mk:48: *  see https://source.android.com/source/download.html
build/core/main.mk:49: ********************************************************************************
```

解决方法就是make降版本

``` bash
cd ~/Downloads
proxychains wget http://ftp.gnu.org/gnu/make/make-3.81.tar.gz
tar -xzvf make-3.81.tar.gz 
cd make-3.81
./configure 
sh build.sh
sudo make install
make -v 
    GNU Make 3.81
    Copyright (C) 2006  Free Software Foundation, Inc.
    This is free software; see the source for copying conditions.
    There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
    PARTICULAR PURPOSE.

    This program built for x86_64-unknown-linux-gnu
```

另外一个问题, 在make的过程中提示 **/bin/bash: jar: command not found** 而无法继续编译, 那么这其实是因为切换到root身份时并没有重新加载/etc/profile, 也就意味着你的环境变量根本没有加载进来, 所以找不到jar. 

一个可行的做法是在 **/usr/bin** 建立一个软链接. 同理后续还有**javadoc**, 也用一样的方法解决

```bash
cd /usr/bin
sudo ln -s -f /usr/local/java/jdk1.6.0_45/bin/jar
sudo ln -s -f /usr/local/java/jdk1.6.0_45/bin/javah /bin/javah
sudo ln -s -f /usr/local/java/jdk1.6.0_45/bin/javadoc /bin/javadoc
```

然后重新运行**sudo make -j4**即可

## 下载 TaintDroid 源码

将下列代码复制到 **/Android/aosp/.repo/local_manifests/local_manifest.xml**, 这个**locla_manifests**目录以及**local_manifest.xml**我在同步源码后是没有的, 我本来以为是源码同步的时候出了什么问题没有同步完整, 在网上找了相关资料都没有相关问题, 所以我就直接创建了这样一个文件夹和xml文件. 

``` xml
<manifest>  
  <remote name="github" fetch="git://github.com"/>  
  <remove-project name="platform/dalvik"/>  
  <project path="dalvik" remote="github" name="TaintDroid/android_platform_dalvik" revision="taintdroid-4.3_r1"/>  
  <remove-project name="platform/libcore"/>  
  <project path="libcore" remote="github" name="TaintDroid/android_platform_libcore" revision="taintdroid-4.3_r1"/>  
  <remove-project name="platform/frameworks/base"/>  
  <project path="frameworks/base" remote="github" name="TaintDroid/android_platform_frameworks_base" revision="taintdroid-4.3_r1"/>  
  <remove-project name="platform/frameworks/native"/>  
  <project path="frameworks/native" remote="github" name="TaintDroid/android_platform_frameworks_native" revision="taintdroid-4.3_r1"/>  
  <remove-project name="platform/frameworks/opt/telephony"/>  
  <project path="frameworks/opt/telephony" remote="github" name="TaintDroid/android_platform_frameworks_opt_telephony" revision="taintdroid-4.3_r1"/>  
  <remove-project name="platform/system/vold"/>  
  <project path="system/vold" remote="github" name="TaintDroid/android_platform_system_vold" revision="taintdroid-4.3_r1"/>  
  <remove-project name="platform/system/core"/>  
  <project path="system/core" remote="github" name="TaintDroid/android_platform_system_core" revision="taintdroid-4.3_r1"/>  
  <remove-project name="device/samsung/manta"/>  
  <project path="device/samsung/manta" remote="github" name="TaintDroid/device_samsung_manta" revision="taintdroid-4.3_r1"/>  
  <remove-project name="device/samsung/tuna"/>  
  <project path="device/samsung/tuna" remote="github" name="TaintDroid/android_device_samsung_tuna" revision="taintdroid-4.3_r1"/>  
  <project path="packages/apps/TaintDroidNotify" remote="github" name="TaintDroid/android_platform_packages_apps_TaintDroidNotify"  
      revision="taintdroid-4.3_r1"/>  
</manifest>  
```

然后执行以下命令

``` bash
cd aosp 
repo sync --force-sync 
repo forall dalvik libcore frameworks/base frameworks/native frameworks/opt/telephony system/vold system/core device/samsung/manta device/samsung/tuna \
       packages/apps/TaintDroidNotify -c 'git checkout -b taintdroid-4.3_r1 --track github/taintdroid-4.3_r1 && git pull'
```

## 编译 TaintDroid

在**TaintDroid**根目录下, 也就是我们的**aosp**目录下新建一个**buildspec.mk**和**build.mk**文件,  然后写入以下代码到**buildspec.mk**和**build.mk**里(比如我, 一开始就只新建了**buildspec.mk**, 然后编译出错, 网上的解答也很少, 仅三篇, 但无疑就是这个文件的问题, 所以后面就再新建了**build.mk**, 而且**Android**和**aosp**目录下都有放, 所以就能编译成功)

``` bash
# Enable core taint tracking logic (always add this)    
WITH_TAINT_TRACKING := true      
# Enable taint tracking for ODEX files (always add this)    
WITH_TAINT_ODEX := true      
# Enable taint tracking in the "fast" (aka ASM) interpreter (recommended)    
WITH_TAINT_FAST := true      
# Enable additional output for tracking JNI usage (not recommended)    
#TAINT_JNI_LOG := true      
# Enable byte-granularity tracking for IPC parcels    
WITH_TAINT_BYTE_PARCEL := true   
```

然后打开**Android/aosp/build/target/product/core.mk**并把**TaintDroidNotify**添加到**PRODUCT_PACKAGES**中(注意添加到第一个出现的PRODUCT_PACKAGES中)

``` bash
$RODUCT_PACKAGES += \    
                    BasicDreams \    
                    ...    
                    voip-common \    
                    TaintDroidNotify   
```

然后就可以开始编译**TaintDroid**了

``` bash
. build/envsetup.sh    
lunch 1
make clean    
sudo make -j4  
```

在执行**sudo make -j4**的过程中, 编译出了问题, 报错提示是

``` bash
dalvik/vm/mterp/out/InterpC-portable.cpp:2683:1: error: 'GET_REGISTER_TAINT_FLOAT' was not declared in this scope
dalvik/vm/mterp/out/InterpC-portable.cpp:2683:1: error: 'SET_REGISTER_TAINT_FLOAT' was not declared in this scope
dalvik/vm/mterp/out/InterpC-portable.cpp:3177:1: error: 'dvmSetStaticFieldTaintIntVolatile' was not declared in this scope
dalvik/vm/mterp/out/InterpC-portable.cpp:3197:1: error: 'dvmSetStaticFieldTaintLongVolatile' was not declared in this scope
dalvik/vm/mterp/out/InterpC-portable.cpp:3480:1: error: 'dvmSetStaticFieldTaintObjectVolatile' was not declared in this scope
make: *** [out/host/linux-x86/obj/SHARED_LIBRARIES/libdvm_intermediates/mterp/out/InterpC-portable.o] Error 1

```

这个的问题就在于**buildspec.xml**, 你应该像之前说的那样再创建一个**build.xml**文件, 然后从 **. build/envsetup.sh**重新开始执行命令(其实从**make clean**开始也行)

## Taintdroid 应用于 Emulator

这里的前提条件是我们之前有用**AVD Manager**下载了arm的安卓镜像. 我之前一直是用的**genymotion**所以我这次也要打开**Android Studio**下载对应的镜像. 因为我们编译的TaintDroid的原因(api要求必须为18), 我选择了**API:18 System image:armeabi-v7a Android:4.3**的**Jelly Bean**版本image下载安装

下载完成, 可以在**Android SDK**目录下发现一个**system-images**的目录, 比如我的目录路径是 **/AndroidSDK/system-images/android-18/google_apis/armeabi-v7a**. 

我们回到**aosp**目录下, 现在我们需要做的就是将之前**TaintDroid**编译生成的 **/aosp/out/target/product/generic/system.img**替换到 **/AndroidSDK/system-images/android-18/google_apis/armeabi-v7a/system.img**. 这样我们使用**emulator**启动AVD后就可以使用**TaintDroid**了

当然你也可以不用这么麻烦, 我们可以直接使用先前编译出的安卓版本来运行. 比如我的就是这样(sdcard.img需要自己生成, 百度一下吧). 

```
./emulator -system /home/vancir/Documents/code/Android/aosp/out/target/product/generic/system.img  -data /home/vancir/Documents/code/Android/aosp/out/target/product/generic/userdata.img -ramdisk /home/vancir/Documents/code/Android/aosp/out/target/product/generic/ramdisk.img  -sdcard /home/vancir/Documents/code/Android/aosp/out/target/product/generic/sdcard.img -kernel /home/vancir/Documents/code/Android/aosp/prebuilts/qemu-kernel/arm/kernel-qemu-armv7 

```

至此, 整个部署过程就已经完成了. 

1. [TaintDroid：用于在智能手机上实时监控隐私数据的信息流跟踪系统](https://wenku.baidu.com/view/b58f924f0975f46527d3e1bb.html)
2. [TaintDorid 官网](http://www.appanalysis.org/)
3. [TaintDorid google group](https://groups.google.com/forum/#!forum/taintdroid)