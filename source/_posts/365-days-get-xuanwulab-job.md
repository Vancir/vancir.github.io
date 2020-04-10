---
title: 给自己365天时间获取腾讯玄武实验室的工作 
date: 2020-04-10 23:44:19
tags: 
---

## 这是什么? 

这是一份我给自己365天内获取腾讯玄武实验室工作定下的学习进度清单, 用于记录我在这一年时间里每天的学习收获. 

因为知识积累的差异, 该清单并不适用于纯粹的新手, 但我常认为自己是一个愚笨的人, 所以即便是刚入行的小白, 在补足了一定的基础知识后, 该清单依然具有一定的参考价值. 

## 学习进度

<details>
<summary>Day1: 学习CTF Wiki栈溢出基础和ROP基础</summary>

> 传送门: [CTF Wiki: Linux Pwn](https://ctf-wiki.github.io/ctf-wiki/pwn/readme-zh/)

- [x] [Stack Overflow Principle](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/stackoverflow/stackoverflow-basic-zh/): 通过栈溢出覆盖掉函数栈帧的返回地址, 当函数返回时就会跳入攻击者覆写的地址继续执行代码. 
    1. 确认溢出的长度可以到达栈帧返回地址
    2. 确认没有开启Stack Canary
    3. 确认覆写的地址所在的段具有执行权限
    * 编译选项`-fno-stack-protector`用于关闭Stack Canary
    * 编译时需要加`-no-pie`确保不会生成位置无关文件
    *  关闭ASLR: `echo 0 > /proc/sys/kernel/randomize_va_space`
- [x] [Basic ROP](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/stackoverflow/basic-rop-zh/): 在栈溢出的基础上, 通过利用文件本身的gadget来控制寄存器和变量来控制程序流程.
    - [x] ret2text: 跳转到程序已有的高危代码处(`system("/bin/sh")`), 直接触发高危操作.
    - [x] ret2shellcode: 栈溢出的同时布置shellcode(可以理解为预写好的高危功能性汇编代码), 在溢出时跳转到布置好的shellcode处继续执行.
        1. 因为有执行, 所以需要确保shellcode所在位置有可执行权限.
        2. gef的`vmmap`可以查看内存段的权限.
        3. pwntool获取shellcode: `asm(shellcraft.sh())`
    - [x] ret2syscall: 没有执行权限时, 可以通过系统调用来实现控制. 
        1. 开启NX保护后, 再如何部署高危代码都没法执行. 所以需要转向利用内核的系统调用实现高危操作. 
        2. 可以通过`/usr/include/asm/unistd_32.h`查看当前内核对应的系统调用号. 比如`#define __NR_execve 11`, 也就是`execve`的系统调用号为`0xb`
        3. 使用`ROPgadget`可用获取寄存器和字符串的gadget.
           * `ROPgadget --binary rop  --only 'pop|ret' | grep 'ebx' | grep 'ecx'`
           * `ROPgadget --binary rop  --string '/bin/sh'`
           * `ROPgadget --binary rop  --only 'int'`
        4. 使用`flat`来直观地表示ROP链: `flat(['A' * 112, pop_eax_ret, 0xb, pop_edx_ecx_ebx_ret, 0, 0, binsh, int_0x80])` 
           * 形式为: `溢出用的填充数据, gadget1(函数原本的返回地址), value1, gadget2, value2, ... , int 0x80`  
    - [x] ret2libc: 
        - [x] ret2libc1: 跳转到libc的高危代码(`system`)并模拟函数调用
            1. 注意跳转到libc的函数去执行, 需要模拟函数调用, 因此跟gadget在栈上的部署方式不一样, 正确的形式为`PLT地址, 函数返回地址, 函数参数地址...`
            2. 获取`system()`的plt地址方法: `objdump -d ret2libc1 | grep system`, 也就是地址是写在汇编里的.
        - [x] ret2libc2: 如果缺少函数调用的条件(缺少函数参数字符串`/bin/sh`)
            1. 利用libc里的`gets`函数, 并手动输入相应的函数参数字符串即可弥补.
            2. `['a' * 112, gets_plt, pop_ebx, buf2, system_plt, 0xdeadbeef, buf2]`需要注意的是`pop_ebx`作为`gets`的返回地址, 它还将buf2给弹出栈, 使得程序继续向下执行`system`函数部分.
        - [x] ret2libc3: 既没有函数参数字符串(`/bin/sh`)也没有高危libc函数地址(`system`)
            1. libc之间函数偏移是固定的, 因此可以通过某个已知的libc函数偏移, 来获取任意其他libc函数地址. 
            2. libc有延迟绑定机制, 只有执行过的函数它的GOT才是正确的. 
            3. libc内自带有`/bin/sh`字符串. 
            4. 可以利用`__libc_start_main`地址来泄露偏移.
            5. 利用思路就是 => 构造ROP链通过`puts`泄露`__libc_start_main`的got地址 => 使用`LibcSearcher`获取libc的基址从而获取`system`地址和`/bin/sh`地址 => 重载程序 => 构造payload控制.
</details>

## 相关资源

* [CTF Wiki](https://ctf-wiki.github.io/ctf-wiki/): 起初是X-Man夏令营的几位学员, 由[iromise](https://github.com/iromise)和[40huo](https://github.com/40huo)带头编写的CTF知识维基站点. 我早先学习参与CTF竞赛的时候, CTF一直没有一个系统全面的知识索引. [CTF Wiki](https://ctf-wiki.github.io/ctf-wiki/)的出现能很好地帮助初学者们渡过入门的那道坎. 我也有幸主要编写了Wiki的Reverse篇. 

## 关于X-Man夏令营

非常感谢[赛宁网安](http://www.cyberpeace.cn/), [诸葛建伟老师](https://netsec.ccert.edu.cn/chs/people/zhugejw/)和[陈启安教授](https://information.xmu.edu.cn/info/1018/3156.htm)的帮助才让我有幸成为第一期X-Man夏令营的成员. 我也是在X-Man夏令营里认识了[A7um](https://github.com/A7um), [iromise](https://github.com/iromise), [40huo](https://github.com/40huo)等一众大佬. 就我同期的X-Man夏令营学员们, 几乎都投身于国内的安全事业, 如今学员们遍地开花, 也是诸葛老师非常欣慰看见的吧. 

## 关于作者

我本科毕业于厦门大学, 毕业后就职于[奇安信技术研究院](https://research.qianxin.com/). 奇安信技术研究院有着非常自由的工作和学术研究环境, 我对初入安全行业而选择的第一份工作非常满意.

但玄武实验室对于国内安全从业人员的吸引力, 就如同谷歌对广大程序员的吸引一般, 我渴望着得到玄武实验室的工作. 而我认识的[A7um](https://github.com/A7um)也在玄武实验室, A7um是我初学安全时仰慕的偶像之一, 我期待着能与玄武实验室里才华横溢的大佬们一起共事研究. 

从初入社会到如今的这半年多时间里, 我找到了生活工作和学习的节奏, 我并没有选择急于去钻研技术, 而是阅读了更多的非技术类书籍, 这教导了我为人处世的经验, 在北京站稳了脚跟, 顺利从刚毕业的懵懂小生过渡到现在略有成熟的青年. 而如今我要展开脚步, 去追求梦想的工作了, 所以我创建了该项目, 既是对自我的激励监督, 也是向分享我的学习历程.