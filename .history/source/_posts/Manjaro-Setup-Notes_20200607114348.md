---
title: 我的Manjaro+i3wm配置记录
date: 2017-12-11 15:30:52
tags:
---

以下是我安装完Manjaro-i3后的配置记录

## 0x01 添加国内源

``` bash
sudo pacman-mirrors -i -c China -m rank
# 选择清华和中科大的源
sudo vim /etc/pacman.conf
## 填入以下内容
[archlinuxcn]
SigLevel = Optional TrustedOnly
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch
Server = http://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch

[antergos]
SigLevel = TrustAll
Server = http://mirrors.tuna.tsinghua.edu.cn/antergos/$repo/$arch

[arch4edu]
SigLevel = TrustAll
Server = http://mirrors.tuna.tsinghua.edu.cn/arch4edu/$arch
# 将Color的注释删去

# 运行以下命令进行更新
sudo pacman -Syy
# 导入GPG
sudo pacman -S archlinuxcn-keyring 
sudo pacman -S antergos-keyring
# 更新系统
sudo pacman -Syu
```

## 0x02 安装常用CLI工具及软件

``` bash
sudo pacman -S yay git firefox netease-cloud-music screenkey
```



## 0x03 安装设置Rime输入法

``` bash
yay -S fcitx fcitx-im fcitx-configtool fcitx-rime
# 设置环境变量asd
vim ~/.xprofile
## 填入以下内容
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
# 重新启动/重新登录
sudo reboot
# 填写rime输入法的配置文件
vim ~/.config/fcitx/rime/default.custom.yaml
## 填入以下内容重新部署rime/重启fcitx即可生效
patch:
  schema_list:
    - schema: luna_pinyin_simp
    
  "ascii_composer/switch_key":
    Caps_Lock: noop
    Shift_L: commit_code 
    Shift_R: inline_ascii

  "punctuator/full_shape":
    "/": "/"
  "punctuator/half_shape":
    "/": "/"

  "menu/page_size": 9
```



## 0x04 解决音频输出的问题

TODO



## 0x05 配置Vim

todo

## 0x06 配置Zsh Shell

``` bash
yay -S fish
chsh -s /usr/bin/fish
# 挑一个喜欢的配色和提示符
fish_config 
```

## 0x07 安装终端软件alacritty

``` bash
yay -S alacritty
vim ~/.config/i3/config
# 将终端修改为alacritty
```



