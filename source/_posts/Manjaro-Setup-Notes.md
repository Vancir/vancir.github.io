---
title: 我的Manjaro+i3wm配置记录
date: 2020-06-07 19:44:52
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
sudo pacman -S yay git firefox netease-cloud-music screenkey tmux aria2 google-chrome feh rofi polybar betterlockscreen pywal-git imagemagick thefuck visual-studio-code-bin intellij-idea-ultimate-edition lxappearance deepin-wine-tim deepin-wine-wechat dolphin redshift deepin-screenshot foxitreader p7zip the_silver_searcher tig wps-office ttf-wps-fonts mpv
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
# 编辑~/.i3/config文件填入下面这行
exec_always --no-startup-id fcitx
```



## 0x04 解决音频输出的问题

``` bash
yay -S pulseaudio pavucontrol
pulseaudio --start
# 打开pavucontrol配合alsamixer将音量调高
# 然后右键下方状态栏最右边的声音按钮, 将输出调为耳机
```

## 0x05 配置Vim

安装neovim然后安装vim-plug用于管理插件, 然后装上YouCompleteMe插件

``` bash
yay -S neovim 
curl -fLo ~/.vim/autoload/plug.vim --create-dirs \
    https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim

vim ~/.config/nvim/init.vim
# 写入配置, 参考我的配置
# 安装YCM
yay -S vim-youcomplete-git
# 配置里写 Plug 'Valloric/YouCompleteMe'
# 安装好插件. 中间会有一些报错, 用以下命令
pip install neovim 
cd cd ~/.local/share/nvim/plugged/YouCompleteMe && python install.py
# 进入neovim运行 :YcmRestartServer
```


## 0x06 配置Fish Shell

``` bash
yay -S fish
chsh -s /usr/bin/fish
# 挑一个喜欢的配色和提示符
fish_config 
```

## 0x07 配置ZSH Shell

``` bash
yay -S zsh
# 安装oh my zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
# 安装插件
git clone git://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
# 编辑~/.zshrc
ZSH_SHELL="steeef"
plugins=(
	git
	zsh-autosuggestions
	zsh-syntax-highlighting
	z
	extract
	colored-man-pages
	fzf
)
# 修改提示符的样式
PROMPT=$'
%{$purple%}#${PR_RST} %{$orange%}%n${PR_RST} %{$purple%}@${PR_RST} %{$orange%}%m${PR_RST} in %{$limegreen%}%~${PR_RST} %{$limegreen%}$pr_24h_clock${PR_RST} $vcs_info_msg_0_$(virtualenv_info)
%{$hotpink%}$ ${PR_RST}'
```



## 0x08 安装终端软件alacritty

``` bash
yay -S alacritty
vim ~/.config/i3/config
# 将终端修改为alacritty
```

## 设置ZSH配置

``` bash
alias c clear
alias aria2c aira2c -s16 -x16
alias setproxy="export ALL_PROXY=XXXXXXX"
alias unsetproxy="unset ALL_PROXY"
alias ip='curl ip.sb'
alias grep='grep --color=auto'
alias ra='ranger'
```

## 安装字体图标主题等

``` bash
yay -S papirus-icon-theme wqy-microhei ttf-font-awesome

yay -S ttf-linux-libertine ttf-inconsolata ttf-joypixels ttf-twemoji-color noto-fonts-emoji ttf-liberation ttf-droid ttf-fira-code adobe-source-code-pro-fonts

yay -S wqy-bitmapfont wqy-microhei wqy-microhei-lite wqy-zenhei adobe-source-han-mono-cn-fonts adobe-source-han-sans-cn-fonts adobe-source-han-serif-cn-fonts
```

## 配置Rofi

``` bash
yay -S pywal-git
mkdir -p ~/.config/wal/templates
# 使用https://github.com/ameyrk99/no-mans-sky-rice-i3wm里的.i3/rofi.rasi放置在templates目录下
# 并重命名为config.rasi
# 编辑~/.i3/config将mod+d由dmeun修改为rofi
bindsym $mod+d exec rofi -show run
```

## 同步时间

``` bash
sudo hwclock --systohc
sudo ntpdate -u ntp.api.bz
```


## 调准鼠标滚轮速度

``` bash
yay -S imwheel
vim ~/.imwheelrc
# 填入以下内容
".*"
None,      Up,   Button4, 4
None,      Down, Button5, 4
Control_L, Up,   Control_L|Button4
Control_L, Down, Control_L|Button5
Shift_L,   Up,   Shift_L|Button4
Shift_L,   Down, Shift_L|Button5
# 将imwheel写到i3的配置里自动启动, 或者直接执行imwheel也行
imwheel
```

## 配置compton毛玻璃特效

``` bash
# manjaro i3自带compton, 但是该版本只能半透明而无法实现毛玻璃特效
# 我们需要使用另一个分支版的compton
# 卸载预装的compton
yay -Rc picom
# 需要安装asciidoc
yay -S asciidoc
git clone https://github.com/tryone144/compton
cd compton
make 
sudo make install
# 编辑 ~/.config/compton.conf里的opacity
```

## 配置polybar

把配置文件放进去将可以

## 安装WPS

``` bash
sudo pacman -S wps-office
sudo pacman -S ttf-wps-fonts
sudo vim /usr/bin/wps
# 在shebang下面填入
export XMODIFIERS="@im=fcitx"
export QT_IM_MODULE="fcitx"
```



