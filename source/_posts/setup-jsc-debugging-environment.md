---
title: 编译JavaScriptCore并搭建调试环境
date: 2019-01-28 15:34:53
tags:
---


由于需要对JavaScriptCore进行模糊测试的原因, 像Safari这样采用JSC的浏览器在MacOS平台上运行, 而且由于浏览器沙箱逃逸经常需要利用到内核漏洞, 所以搭建一个MacOS环境是必需的.

## 0x01 配置MacOS Majave 14.10 虚拟机

- 主机: Ubuntu 16.04 64bits
- 虚拟机: VMware Workstation 14 Pro 14.1.3 build-9474260
- 安装镜像: macOS Mojave 10.14 18A391 Lazy Installer.cdr 7.7GB
- 解锁工具: [unlocker v3.0.0](https://github.com/DrDonk/unlocker) (建议使用最新的3.0.2版本, 因为3.0.0版本在安装vmware tools时出现问题)

关闭VMware进程, 以root身份运行unlocker内的**lnx-install.sh**和**lnx-update-tools.sh**

```bash
chmod +x *.sh
sudo ./lnx-install.sh
sudo ./lnx-update-tools.sh
```

然后启动VMware, 选择新建虚拟机, 这时已经可以选择新建MacOS的虚拟机, 在选择镜像时默认是iso文件, 因此需要选择显示所有文件, 才能选取我们的cdr文件. 其他就跟普通的虚拟机创建方法类似. 配置情况为2处理器2核心3G内存.

创建好虚拟机后, 先不用启动虚拟机, 还需要做一点改动: 进入虚拟机文件位置, 修改vmx文件(比如我的是**macOS 10.14.vmx**), 找到其中的**smc.present = "TRUE"**, 在这行之下添加一行**smc.version = "0"**然后保存. 这时就可以启动虚拟机了.

启动虚拟机后, 点击顶部的**实用工具-磁盘工具**, 将左侧菜单中**vmware虚拟磁盘**抹除(选中后点击上方菜单**编辑-抹掉**). 然后选择将Macos安装在你刚才抹掉的这个磁盘上即可开始安装. 安装好后就跟着向导进行设置即可.

安装好后需要安装**vmware tools**. 我在安装的时候出了两个问题:
\1. **Could not find component on update server. Contact VMware Support or your system administrator.** 这是我使用的**unlocker**版本问题, **3.0.2**版本已经修复该问题. 或者捏可以手动挂载**darwin.iso**的方法来解决. 安装好重新启动即可.
\2. 即便安装好**vmware tools**也没能生效全屏(我是这样), 而这是因为MacOS的System Integrity Protection(SIP)保护, 我们需要重新启动操作系统, 然后在启动的过程中一直按下(或不断按)Command+R键(也可以尝试Win+R, 我有点乱敲的感觉), 进入到恢复模式. 然后打开顶部菜单中**实用工具-终端**, 输入**csrutil disable**即可关闭该保护. 重新启动后, **vmware tools**即可生效(没生效就多重启几次).

进入Mac后做一些性能优化, 比如减少透明度, 关闭不需要的扩展, 修改Dock的神奇效果为缩放效果, 卸载不必要的程序等等.

## 0x02 下载编译JavaScriptCore

配置好Mac环境后, 我们需要安装好以下几个工具. 当然要从App Store获取软件你需要创建一个AppleID

1. JDK – Java version “11.0.2”
2. Git, CMake
3. Xcode & Xcode Command Line Tools
4. proxychains-ng, v2ray / shadowsocks

那么我们就可以开始获取WebKit源码并编译JavaScriptCore

``` bash
git config --global http.postBuffer 52428800000 #设置postBuffer缓存为50G
git clone git://git.webkit.org/WebKit.git WebKit
cd Webkit
Tools/Scripts/build-webkit --jsc-only --cmakeargs="-DENABLE_STATIC_JSC=ON -DUSE_THIN_ARCHIVES=OFF"
```

Webkit需要在10.10.5版本以上进行编译, 但我在尝试的过程中CMake会提示需要提供C++14标准支持. 更新GCC版本并没能解决该问题. 但在10.14版本中没有该问题.