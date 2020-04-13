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

<details>
<summary>Day2: 学习CTF Wiki中级ROP和格式化字符串漏洞</summary>

> 传送门: [CTF Wiki: Linux Pwn](https://ctf-wiki.github.io/ctf-wiki/pwn/readme-zh/)

- [x] [Intermediate ROP](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/stackoverflow/medium-rop-zh/):
    - [x] [ret2csu](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/stackoverflow/medium-rop-zh/#_1):
        * x64寄存器传参的顺序为`rdi, rsi, rdx, rcx, r8, r9`, 超出数量的参数根据函数调用约定压入栈中(比如从右向左压栈)
        * `__libc_csu_init`是`__libc_start_main`调用的用于初始化的函数. 参考: [Linux X86 程序启动–main函数是如何被执行的？](https://luomuxiaoxiao.com/?p=516)
        * 示例的level5应是[ctf-challenges](https://github.com/ctf-wiki/ctf-challenges)里的[hitcon-level5](https://raw.githubusercontent.com/ctf-wiki/ctf-challenges/master/pwn/stackoverflow/ret2__libc_csu_init/hitcon-level5/level5), 而非蒸米提供的[level5](https://github.com/zhengmin1989/ROP_STEP_BY_STEP/tree/master/linux_x64)
        * 使用`ROPgadget`搜索可用的gadget是可以发现, 程序并没有直接的控制传参用的寄存器, 大多都是控制`r12-r15`, 这也就是分析`__libc_csu_init`的关键: 我们需要其中的`mov`语句, 通过`r13-r15`控制x64传参用的前三个寄存器.
        * 分析`__libc_csu_init`的目的是掌握可控制的寄存器, 也就是能控制`rbx, rbp, r12, r13=>rdx, r14=>rsi, r15=>edi`, 同时可控的`r12`和`rbx`以及`call qword ptr [r12+rbx*8]`能控制调用的函数地址(`r12`为函数地址, `rbx`直接为0). `add rbx, 1; cmp rbx, rbp; jnz 400600`则是约束条件`rbx+1==rbp`, 故而`rbx=0则rbp=1`. 这样来看这是一段非常优雅的`gadget`. 
        * `write (fd, &buf, count)`中, linux下`fd=0/1/2`分别对应`stdin/stdout/stderr`. 
        1. libc延迟绑定机制, 因此需要等待`write`输出`Hello, World`后泄露函数地址. 
        2. 泄露函数地址后获取libc基址, 然后获取`execve`地址
        3. 利用csu执行`read()`向bss段写入`execve`地址和参数`/bin/sh`
        4. 利用csu执行`execve(/bin/sh)`
        <details>
        <summary>Q1: 为什么要先<code>read()</code>写<code>execve</code>地址, 而不是直接调用<code>execve</code>函数呢?</summary>
        因为<code>call qword ptr [r12+rbx*8]</code>指令, 实际上我们通过csu控制的是一个地址, 而该地址指向的内容才是真正函数的调用地址. 而<code>read()</code>写到bss段的是<code>execve</code>的地址, 但csu调用的时候提供的是bss段的地址, 这样才能完成函数调用. 如果直接传<code>execve</code>地址, 那么是无法调用成功的.
        </details>
        <details>
        <summary>Q2: 为什么可以用写入的<code>/bin/sh</code>地址能成功, 而直接用libc内的<code>/bin/sh</code>地址就不能成功呢?</summary>
        我一个可能性比较高的推测是, 回顾我们的gadget, 对于x64传参的第一个寄存器<code>rdi</code>, 其实我们的gadget只能控制寄存器<code>rdi</code>的低32位(<code>edi</code>). 而对于bss段地址来说, 它实际上是一个32位的地址(高32位为0), 而libc内的<code>/bin/sh</code>是一个64位的地址(高32位不为0), 所以没有办法传递完整的地址进去. 所以只能通过bss上写入的<code>/bin/sh</code>地址进行传参. 
        </details>
        <details>
        <summary>csu函数实现</summary>
        ``` python
        def csu(func_addr, arg3, arg2, arg1, ret_addr):
           rbx = 0
           rbp = 1
           r12 = func_addr
           r13 = arg3
           r14 = arg2
           r15 = arg1
        
           # pop rbx rbp r12 r13 r14 r15 retn
           csu_pop_gadget = 0x000000000040061A

           # r13=>rdx r14=>rsi r15=>edi 
           # call func
           # rbx+1 == rbp
           # add rsp, 8
           # csu_pop_gadget
           csu_mov_gadget = 0x0000000000400600

           # pop 6 registers and `add rsp, 8`
           stack_balance = b'\x90' * 0x8 * (6+1)

           payload = flat([
               b'\x90'*0x80, b'fake_rbp', p64(csu_pop_gadget),
               p64(rbx), p64(rbp), p64(r12), p64(r13), p64(r14), p64(r15),
               p64(csu_mov_gadget), stack_balance, p64(ret_addr)
           ])

           io.send(payload)
           sleep(1)
        ```
        </details>
    - [x] [BROP](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/stackoverflow/medium-rop-zh/#brop): 盲打的方式通过程序是否崩溃来推测信息. 适用于Nginx, MySQL, Apache, OpenSSH等服务器应用, 因此该攻击还有着一定的实际应用价值.
        > 理论知识主要参考 [Blind Return Oriented Programming (BROP) Attack-攻击原理](https://wooyun.js.org/drops/Blind%20Return%20Oriented%20Programming%20(BROP)%20Attack%20-%20%E6%94%BB%E5%87%BB%E5%8E%9F%E7%90%86.html), 示例程序参考 [HCTF2016-出题人失踪了(brop)](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/stackoverflow/brop/hctf2016-brop)
        * 实现攻击必需的2个条件:
            1. 存在栈溢出漏洞, 且攻击者可以通过输入轻松触发. (没有程序没有源码没有信息, 打也打不崩, 那还玩什么)
            2. 程序崩溃后会重新运行, 并且重新运行的进程地址不会再次随机化. (能稳定复现, 获取稳定地址, 包括Stack Canary也不能随机化)
        * 描述了4种gadget:
            1. stop gadget: 程序跳转到该gadget片段后, 程序并没有崩溃, 而是进入某种hang/loop状态, 能与攻击者保持连接. 
            2. (potentially) useful gadget: 找到stop gadget后, 通过一定的内存布局而发现的更多的`不会崩溃`的gadget. (当然包括新发现的stop gadget)
            3. brop gadget: 一种特殊的`useful gadget`, 能帮助我们控制x64传参用的寄存器. 典型示例就是`__libc_csu_init()`尾部的rop链. gadget能通过指令错位(`+7/+9`)的方式得到单独控制`rsi`和`rdi`寄存器的新gadget.
            4. trap gadget: 就是会让程序崩溃的gadget. 
        * 攻击思路:
            1. 通过爆破, 获取程序崩溃时的字符串填充长度. 
            2. 通过单字节枚举, 逐字节地泄露出栈上保存的`Canary`. (当然也可以枚举出栈上保存的寄存器和原本的返回地址.)
            3. 寻找`stop gadget`: 早期能得到的信息只有程序崩溃和不崩溃, 所以我们需要获得第一个程序不会崩溃的stop gadget. 
            4. 寻找`useful gadget`: 通过合理的布局栈上的内存, 我们可以利用`stop gadget`来发掘更多的`useful gadget`, 并且是能确认该`useful gadget`弹栈数量的.
                * 比如栈上的布局情况为: `...| buffer | gadget | trap x N | stop | trap|...`  则表明该gadget有`N`个pop指令(`N=0,1,...`).
            5. 从`useful gadget`里筛选出真正有帮助的`brop gadget`. 这里就以`__libc_csu_init()`的尾部gadget为例, 该gadget能弹栈`6`次, 通常认为符合这种性质的gadget很少, 所以有一定把握去判断, 并且该gadget可以通过错位得到单独控制`rsi`和`rdi`的gadget, 也可以通过`减去0x1a`来获取其上的另一个gadget. 
            6. 寻找`PLT`项. PLT在盲打时有这样的特征: 每一项都有`3`条指令共`16`个字节长. 偏移`0`字节处指向`fast path`, 偏移`6`字节处指向`slow path`. 如果盲打时发现有连续的`16`字节对齐的地址都不会造成程序崩溃, 这些地址加`6`后也不会崩溃. 那么就推断为`PLT`地址. 
            7. 确定`PLT`项内的`strcmp`和`write(也可以是put)`: 
               * 确定`strcmp`的目的在于: 目前只能通过`brop gadget`控制传参用的前2个寄存器(rdi和rsi), 第3个寄存器`rdx`尚且没法用gadget控制. 因此转变思路通过`strcmp`和控制字符串长度来给`rdx`赋值, 变相控制第三个传参用的寄存器.
               * 确定`write`的目的在于: 需要通过`write`将内存代码都写回给攻击者. 通常是将`fd`设置为连接的`socket描述符`. 而`write`需要3个参数, 这也是为什么借用`strcmp`控制`rdx`的原因. 
               * 确定`strcmp`的方法在于控制函数的两个地址: `readable`和`bad(0x00)`地址. 这样就有`4`种参数形式, 并且只有两个参数地址都是`readable`时函数才会正确执行, 其他情况都没有正确执行, 那么就推断这个plt项对应的是`strcmp`. 
               * 确定`write`的方法在于确定写入的`fd`, 就只能尽量枚举文件描述符来测试了. 建议用较大的文件描述符数字. 
               * 如果是寻找`puts`的话, 就比较容易确定. 因为我们只需要控制输出`0x400000`地址的内容, 该地址通常为ELF文件的头部, 内容为`\x7fELF`. 构造的payload形式为`buffer |pop_rdi_ret | 0x400000 | puts_addr | stop`. 
            8. 有能力控制输出函数后, 攻击者可以输出更多的.text段代码. 也可以去寻找一些其他函数, 比如`dup2`或`execve`等:
               * 将`socket`输出重定向到`stdin/stdout`.
               * 寻找`/bin/sh`, 或者利用`write`写入到某块内存.
               * 执行`execve`或构造系统调用. 
               * 泄露`puts`在内存的实际地址, 然后确认libc基址, 获取`system`地址并构造rop链.
- [x] [Format String Vulnerability](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/fmtstr/fmtstr_intro-zh/):
    * 格式化字符串漏洞的本质在于信任了用户的输入, 攻击者通过输入构造好的格式化字符串来泄露栈上的内存数据.
        * `%x`或`%p`用于泄露栈内存数据.
        * `%s`用于泄露变量对应地址的内容, 存在`\x00`截断.
        * `%n$x`用于泄露输出函数的第`n+1`个参数. 这里的`n`是相对于格式化字符串而言的. 
    * 可以通过`func@plt%N$s`将内存中的`func`实际地址泄露出来. `N`表示其在栈上相对格式化字符串而言是第`N`个参数.
    * 确定了偏移后, 使用`...[overwrite addr]....%[overwrite offset]$n`. `%n`写入的值可通过增加输出的字符数量进行调整.
    * 覆写的地址没有位置的要求, 只需要找到对应偏移即可. 
    * 利用`%hhn`进行单字节写入, `%hn`进行双字节写入.
</details>

<details>
<summary>Day3: 回顾软件安全保护技术和学习ARM汇编基础</summary>

- [x] 软件保护技术: 
    - [x] 反调试:
        * 利用WinAPI检测调试状态: [IsDebuggerPresent](https://ctf-wiki.github.io/ctf-wiki/reverse/windows/anti-debug/isdebuggerpresent-zh/).
        * 内存数据检查: 比如通过`PEB`的字段(`BeingDebug`), 堆上的标志信息([Heap flags](https://ctf-wiki.github.io/ctf-wiki/reverse/windows/anti-debug/heap-flags-zh/))来检测调试.
        * 调试驱动检测: 基于一些使用了驱动的调试器的行为特征, 比如`调试器会在启动后创建相应的驱动链接符号`, 来确定是否存在调试器.
        * [进程窗口检测](https://ctf-wiki.github.io/ctf-wiki/reverse/windows/anti-debug/example-zh/#_3): 比如枚举当前所有进程名/窗口名来检查是否存在已知调试器.
        * 特征码检测: 枚举当前所有正在运行的进程, 匹配特定调试器的内存代码数据来判断是否有调试器. 
        * [时间差检测](https://ctf-wiki.github.io/ctf-wiki/reverse/windows/anti-debug/example-zh/#_2): 通过调试和非调试模式下程序运行的时间差异来判断是否存在调试. 
        * 断点检测/[异常检测](https://ctf-wiki.github.io/ctf-wiki/reverse/windows/anti-debug/example-zh/#seh): 断点检测在于判断内存代码是否被修改为`int3`, `int 2d`等软中断指令, 异常检测在于故意触发异常,如果调试器接管了异常则认定为存在调试器.
        * 功能破坏: 基于大部分程序通常都不会使用系统提供的调试功能这一假设, 保证程序正常运行的前提下, 破坏系统提供的调试相关功能. 比如在创建线程时指定`ThreadHideFromDebugger`属性可以隐藏线程引发的异常, 接收不到异常调试器就无法正常工作. 
        * 双进程保护: 基于一个进程只能同时被一个调试器调试的前提, 以调试方式启动被保护的程序, 通过占用调试行为的方式来阻止攻击者去调试分析受保护程序.
    - [x] 反虚拟机: 
        * BIOS信息检测: 虚拟机软件厂商的BIOS通常具有明显的品牌特征. 
        * 字符串特征检测: 虚拟机产品明显的字符串特征.
        * [后门端口检测](https://ctf-wiki.github.io/ctf-wiki/reverse/windows/anti-debug/example-zh/#vmware): 比如VMWARE的后门I/O端口`0x5658("VX")`读取数据得到`VMXh`
    - [x] 数据校验:
        * 文件校验: 实现计算好程序文件的校验值, 然后运行时再校验比对判断文件本身是否被修改. 
        * 内存校验: 通常程序运行时, `.text/.rsrc`等区段是不会修改的, 通过运行时计算内存数据的校验值来判断内存数据是否被修改.
    - [x] 导入表加密: 保护导入表能阻止攻击者去获取对应的符号信息, 增大分析难度. 
        1. 可以简单地劫持导入表函数调用处来隐藏调试器/反汇编器提供的符号信息.
        2. 也可以预先将导入表函数地址加密存储到某个位置, 然后将导入表RVA指向解密代码, 解密代码运行后得到真实的函数地址, 并跳转过去执行.
        3. 另一种方式就是, 将导入表函数的入口代码进行加密或虚拟化, 在运行时解密.
        4. IAT模拟: 自己实现一些程序可能调用的外部函数, 然后替换导入表内的原始函数.
    - [x] 模块拷贝移位: 用于对抗代码Hook的技术, 方法是复制移位模块, 然后映射模块内的数据到内存以及重定位, 替换原模块函数调用地址.
    - [x] 资源加密: 
        1. 在程序运行时将资源解压/解密, 然后修正PE文件头的资源指向.
        2. Hook各种与资源相关的函数, 然后在调用函数时动态解密资源.
    - [x] 代码加密: 代码加密的目的是将原始代码转换为等价的, 极其复杂的, 更多的代码. 
        * 代码膨胀/变形: 将1条或多条指令转变为等价的其他指令, 更多是用于膨胀. 
        * [垃圾代码(花指令)](https://ctf-wiki.github.io/ctf-wiki/reverse/windows/anti-debug/junk-code-zh/): 目的也是膨胀, 但是方式就是插入无用的或者干扰(误导)调试器反汇编算法的代码. 
        * 代码乱序(平坦化): 通过跳转指令打乱指令的正常顺序, 增大分析难度.
        * 多分支: 也是花指令的一种, 只是这里的花指令着重在分支跳转指令上, 这些分支跳转大部分是根本不会执行的deadcode, 但是会让攻击者在分析时难以确定代码的具体执行流程.
        * call链: 通过call指令来打乱执行流程. 
    - [x] 代码虚拟化: 设计一套虚拟机和对应的opcode来在保证语义的前提下, 模拟原本的指令. 
        虚拟机本质也是程序代码, 运行虚拟机本身也会影响当前的上下文, 因此虚拟机设计时需要保存/恢复上下文, 解决虚拟机和原始代码在上下文的使用冲突. 通常有以下两种方案:
        * 堆机: 开辟新的栈空间来运行虚拟机代码, 代码执行完后恢复原始的栈空间地址即可. 
        * 栈机: 不开辟新空间, 在原有栈空间分出一部分专门给虚拟机使用, 并避免原始指令影响到虚拟机专用的栈空间.
    * 脚本引擎: 将程序的部分功能分交给脚本引擎解释执行.
    * 网络加密: 将程序的部分代码放到服务器执行, 服务器只返回代码的执行结果. 
    * 硬件加密: 类似网络加密, 只是将关键数据/代码转移到了硬件介质里.
    * 代码签名: 利用签名严重算法, 对程序文件数据进行签名, 将对这些签名的校验作为能否运行该软件的判断条件.
- [x] [ARM汇编基础](https://azeria-labs.com/writing-arm-assembly-part-1/)
    - [x] [Introduction to ARM Assembly](https://azeria-labs.com/writing-arm-assembly-part-1/)
        * ARM为RISC指令, 相比CISC具有精简的指令和更多的通用寄存器.
        * ARM只能使用操作寄存器的指令, 并且使用`Load/Store`模型访问内存(也就是只有`Load/Store`指令能访问内存). 
        * 指令精简可以带来更快的运行速度, 但同时在可用指令有限的情况下难以高效地编写软件. 
        * ARM有两种模式`ARM模式`和`Thumb模式`. `Thumb模式`下的指令长度既可以是`2字节`也可以是`4字节`.
        * `ARMv3`前使用`小端`, 之后支持`双端`并且可以切换字节序.
    - [x] [Data Types Registers](https://azeria-labs.com/arm-data-types-and-registers-part-2/)
        * `s`后缀表示`signed`, `b`表示`byte`长度为8, `h`表示`halfword`长度为16. ARM的`word`是`32`位长.
        * 大小端的切换由`CPSR`寄存器的第`9`位`E`来指示. 
        * 寄存器数量取决于ARM的版本. 通常有`30`个32位寄存器, 前`16`个寄存器用户模式下可用, 其他寄存器只有特权模式下可用. 
          * `R0-R6`为通用寄存器, 其中`R0`对应`EAX`
          * `R7`用于保存系统调用号
          * `R8-R10`也是通用寄存器
          * `R11(FP)`类似于`EBP`, 也就是栈基寄存器
          * `R12(IP)`即`Intra Procedural Call`内部过程调用寄存器.(x86没有接触过呢)
          * `R13(SP)`类似于`ESP`, 也就是栈顶寄存器
          * `R14(LR)`:即`Link Register`, 链接寄存器
          * `R15(PC)`: 程序计数器, 类似于`EIP`.
          * `CPSR`: 当前程序状态寄存器, 类似于`EFLAGS`.
        * ARM上的函数调用约定: 前四个参数存储在寄存器`R0-R3`中.
        * 链接寄存器`R14(LR)`: 据[解释](https://baike.baidu.com/item/%E9%93%BE%E6%8E%A5%E5%AF%84%E5%AD%98%E5%99%A8/8767852?fr=aladdin), `LR`实际上是函数调用时用于保存函数的返回地址, 意义在于快速进入和返回`叶函数`. 
        * 程序计数器`R15(PC)`: ARM模式下指令长度为4, Thumb模式下长度为2. PC会根据所处模式来递增相应的指令长度. 执行分支指令时, 会将分支跳转的目的地址保存到`PC`. 但程序执行过程中, `PC`存储的总是当前执行指令的`后2条`指令(ARM模式就+8, Thumb模式就+4).
</details>

<details>
<summary>Day4: 学习ARM汇编基础和CTF Wiki的花式ROP</summary>

> 传送门: [azeria-labs](https://azeria-labs.com/writing-arm-assembly-part-1/) / [ROP Tricks](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/stackoverflow/fancy-rop-zh/)

- [x] [ARM Assembly](https://azeria-labs.com/writing-arm-assembly-part-1/)
    - [x] [ARM Instruction Set](https://azeria-labs.com/arm-instruction-set-part-3/)
        * ARM模式亦或是Thumb模式跟所处的特权等级无关. 
        * 开发ARM Shellcode时需要尽量避免`NULL`空字节出现, 因此常用Thumb指令
        * ARM版本之间会有略微差别, 需要根据对应版本查询[官方文档](http://infocenter.arm.com/help/index.jsp)
        * Thumb有三个版本:
            1. Thumb-1: 16比特长, 用于ARMv6及早期版本
            2. Thumb-2: 16/32比特长, 扩展了Thumb-1, 支持更多的指令. 适用于`ARMv6T2`和`ARMv7`.
            3. ThumbEE: 包括一些对动态生成代码的变化.
        * ARM和Thumb指令的区别:
            1. 条件执行: ARM所有指令都可以条件执行, Thumb只能通过`IT`指令允许部分指令有条件地执行.
            2. 32位表示: 32位的Thumb指令会多一个`.w`的后缀
            3. 桶式移位器(ARM独有特性): 能用于精简指令. 
        * 要切换处理器执行状态, 需要满足以下两者条件其一:
            1. 使用分支指令`BX`或`BLX`并将目标寄存器的最低有效位设置为`1`(通过`+1`实现)
            2. 状态寄存器置位T
        * ARM汇编指令格式`MNEMONIC{S}{condition} {Rd}, Operand1, Operand2`. 注意`Operand2`的使用稍有灵活, 并且有些指令中`Operand1`是隐含的.
    - [x] [Memory Instructions: Loading and Storing Data](https://azeria-labs.com/memory-instructions-load-and-store-part-4/)
        * `[pc, #12]`表示`pc`相对寻址. 不过要注意, ARM里的`pc`指的是当前指令的下`2`条指令位置, 也就是ARM模式下`+8`, Thumb模式下`+4`
        * 地址模式: offset / pre-indexed / post-indexed
            * 以`立即数`作为偏移的情况:
                * `str r2, [r1, #2]`: 地址模式: offset. 直接将r2寄存器中的值存到`r1+2`所表示的地址处. `r1`没有变化
                * `str r2, [r1, #4]!`: 地址模式: pre-indexed(`!`是一个标识的特征). 类似offset寻址模式, 寻址`r1+4`, 寻址存储完执行`r1=r1+4`
                * `ldr r3, [r1], #4`: 地址模式: post-indexed. 寻址`r1`, 寻址完执行`r1=r1+4`
            * 以`寄存器`作为偏移的情况: 类似立即数作偏移的情况, 很好理解. 
            * 以`移位寄存器`作为偏移的情况: 类似立即数作偏移的情况, 不过移位的优先级是最高的, 比如`str r2, [r1, r2, LSL#2]`就是将r2内的值保存到`r1+r2<<2`的地址处.
        * ARM使用立即数: ARM使用立即数的方式很不灵活, 格式为`v = n ror 2*r` 其中`n in [0-255]`, `r in [0-30]`. 对于不能合规的立即数, 考虑拆分成两个更小的数加起来, 或者使用`LDR`指令比如`LDR r1, =511`
    - [x] [Load and Store Multiple](https://azeria-labs.com/load-and-store-multiple-part-5/)
        * 多次加载/存储可以使用`LDM`和`STM`指令
        * `LDM`和`LDR`的方向是相反的, 同样`STM`和`STR`方向也相反
        * 扩展`-IA (increase after), -IB (increase before), -DA (decrease after), -DB (decrease before)`
        * `PUSH`和`POP`和x86汇编基本一致. 
        * `PUSH`等价于`STMDB sp! reglist`
        * `POP`等价于`LDMIA sp! reglist`
    - [x] [Conditional Execution and Branching](https://azeria-labs.com/arm-conditional-execution-and-branching-part-6/)
        * 分支条件在标志寄存器中会相应地置位, 这点跟x86一致, 区别主要在标志寄存器各个位的含义略有不同. ARM的分支通过在指令后加相应的条件码来实现.
            | Condition Code | Meaning (for cmp or subs) | Status of Flags  |
            | ---- | -- | -- |
            | CS or HS | Unsigned Higher or Same (or Carry Set) | C==1 | 
            | CC or LO | Unsigned Lower (or Carry Clear) | C==0 |
            | MI | Negative (or Minus) | N==1 |
            | PL | Positive (or Plus) | N==0 |
            | AL | Always executed | - |
            | NV | Never executed | - |
            | VS | Signed Overflow | V==1 |
            | VC | No signed Overflow | V==0 |
            | HI | Unsigned Higher | (C==1) && (Z==0) |
            | LS | Unsigned Lower or same | (C==0) || (Z==0) |
        * `IT`是`IF-Then-(Else)`的缩写.
        * `IT`指令格式: `IT{x{y{z}}} cond`, 也就是最多可以有条件地执行`4`条指令
            * `cond`指定`IT`块中第`1`条指令的条件
            * `x`指定第`2`条指令的条件, `y`指定第`3`条, `z`指定第`4`条
        * `IT`块里`T`的条件要跟`I`保持一致, `E`的条件要跟`I`和`T`相反. (这也很好理解, 就是ARM划分分支的一种形式)
        * 条件码的反义就不硬背了, 直接看`ITE`就可以判断`IT`块里的情况. 
        * `branch`指令跟x86的类似, 只是助记符不一致, 理解还是很好理解的. 
            * `B`: 单纯跳转分支
            * `BL`: 将`PC+4`保存到`LR`然后跳转分支
            * `BX/BLX`: 相比多了一个`Exchange`, 也就是切换指令集(`ARM <-> Thumb`)
            * `BX/BLX`通常会使用类似`add r2, pc, #1; bx r2`的方法先取`pc`然后`+1`的方法使最低有效位置为1(`0`转ARM，`1`转Thumb), 然后用`BX/BLX`切换指令集. (这里不用担心内存块对齐`4`的问题, CPU会自动屏蔽没有对齐的那个bit1/0). 
    - [x] [Stack and Functions](https://azeria-labs.com/functions-and-the-stack-part-7/)
        * 栈的部分不必多说
        * 函数部分熟悉`Prologue`, `Body`和`Epilogue`
            * `prologue`: `push {r11, lr}; add r11, sp, #0; sub sp, sp, #16`
            * `body`: `mov r0, #1; mov r1, #2; bl max`
            * `epilogue`: `sub sp, r11, #0; pop {r11, pc}`
- [x] [ROP Tricks](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/stackoverflow/fancy-rop-zh/)
    - [x] [stack pivoting](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/stackoverflow/fancy-rop-zh/#stack-pivoting)
        * 直接劫持栈指针指向攻击者的内存, 可以以较少的指令达成攻击, 对于开启PIE保护的程序也可以适用. 
        * 利用的gadget为`pop rsp/esp`, 也可以通过`libc_csu_init`的gadget经过错位获得. 
        * 有办法控制到`esp`后, 还需要想办法将`esp`的值指向写入的shellcode部分. 可以加`\x90`垫.
    - [x] [Stack smash](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/stackoverflow/fancy-rop-zh/#stack-smash)
        * Canary检查到溢出后, 程序会执行`__stack_chk_fail`函数打印`argv[0]`指针. 而攻击思路就是借栈溢出覆盖`argv[0]`实现信息泄露. 
        * 攻击需要确定溢出到`argv[0]`所需要的字节数, 以及需要溢出的地址. 
</details>

## 相关资源

* [CTF Wiki](https://ctf-wiki.github.io/ctf-wiki/): 起初是X-Man夏令营的几位学员, 由[iromise](https://github.com/iromise)和[40huo](https://github.com/40huo)带头编写的CTF知识维基站点. 我早先学习参与CTF竞赛的时候, CTF一直没有一个系统全面的知识索引. [CTF Wiki](https://ctf-wiki.github.io/ctf-wiki/)的出现能很好地帮助初学者们渡过入门的那道坎. 我也有幸主要编写了Wiki的Reverse篇. 
* [漏洞战争:软件漏洞分析精要](https://book.douban.com/subject/26830238/): [riusksk](http://riusksk.me/)写的分析大量漏洞实例的书, 一般建议先学习过[《0day安全:软件漏洞分析技术》](https://book.douban.com/subject/6524076/)后再阅读该书. 我早先阅读过该书的大部分内容, 一般我看漏洞分析的文章都有点跟不太上, 但是看该书的时候作者讲的还是蛮好的. 另外该书是按`漏洞类型`和`CVE实例`划分章节, 所以可以灵活挑选自己需要看的内容. 
* [0day安全:软件漏洞分析技术](https://book.douban.com/subject/6524076/): Windows漏洞分析入门的必看书就不多介绍了. 这本书曾一度抄到千元的价格, 好在[看雪](https://www.kanxue.com/)近年组织重新印刷了几次, 我也是那时候入手的该书, 可以多关注下看雪的活动. 该书的内容很多也很厚实, 入门看的时候可谓痛不欲生, 看不懂的就先跳过到后面, 坚持看下来就能渡过入门的痛苦期了.
* [软件保护及分析技术](https://book.douban.com/subject/26841178/): 该书分为2个部分, 前半部分讲保护和破解的技术, 后半部分造轮子. 前半部分讲的技术分类都蛮多的, 不过大多都是点到即止深度不够, 所以我一般都是看前半部分的当速查和回顾的工具书. 我接下来的目标是该书后半的造轮子部分. 

## 腾讯玄武实验室

### 招聘情报

<details>
<summary>2020/4/8 实习生招募</summary>

> 来自玄武实验室微信公众号当日推送

基本要求: 
1. 在任意系统环境(`Android/Linux/MacOS/iOS/Win`)下有丰富`逆向调试经验`, 并熟悉`安全机制`和`底层架构`.
2. 熟练使用一种`编译型语言`和一种`脚本语言`

加分项:
1. `现实漏洞研究分析经验`, `实际挖掘过漏洞`, `写过利用代码`. 
2. 掌握漏洞研究所需的各种能力, 包括`IDA插件开发`, `Fuzzer开发`, `代码脱壳加密`, `网络协议分析`等.

优劣势分析: 
1. 我有足量时间的`Android/Linux/Win`的逆向调试经验, 对于`Linux/Win`的安全机制和底层架构有一定了解, 不了解`Android`的安全机制和底层架构.
2. 编译型语言(`C/C++`)我的掌握程度一般, 脚本语言(`Python`)掌握良好. 
3. 漏洞研究分析经验是工作的必要内容, IDA插件开发部分, 我曾学习过[IDAPython](https://github.com/Vancir/IDAPython-Scripts)的内容, 对于7.0以上版本还需要了解. `Fuzzer`开发部分是我欠缺的, 我仅详细阅读过`FuzzIL`和`AFL`的源码实现, 并未有实际的开发经验. 有着一定的代码脱壳加密经验, 不过仍需多加练习. 网络协议分析我不擅长也不喜欢, 可以忽略.
</details>

## 关于X-Man夏令营

非常感谢[赛宁网安](http://www.cyberpeace.cn/), [诸葛建伟老师](https://netsec.ccert.edu.cn/chs/people/zhugejw/)和[陈启安教授](https://information.xmu.edu.cn/info/1018/3156.htm)的帮助才让我有幸成为第一期X-Man夏令营的成员. 我也是在X-Man夏令营里认识了[A7um](https://github.com/A7um), [iromise](https://github.com/iromise), [40huo](https://github.com/40huo)等一众大佬. 就我同期的X-Man夏令营学员们, 几乎都投身于国内的安全事业, 如今学员们遍地开花, 也是诸葛老师非常欣慰看见的吧. 

## 关于作者

我本科毕业于厦门大学, 毕业后就职于[奇安信技术研究院](https://research.qianxin.com/). 奇安信技术研究院有着非常自由的工作和学术研究环境, 我对初入安全行业而选择的第一份工作非常满意.

但玄武实验室对于国内安全从业人员的吸引力, 就如同谷歌对广大程序员的吸引一般, 我渴望着得到玄武实验室的工作. 而我认识的[A7um](https://github.com/A7um)也在玄武实验室, A7um是我初学安全时仰慕的偶像之一, 我期待着能与玄武实验室里才华横溢的大佬们一起共事研究. 

从初入社会到如今的这半年多时间里, 我找到了生活工作和学习的节奏, 我并没有选择急于去钻研技术, 而是阅读了更多的非技术类书籍, 这教导了我为人处世的经验, 在北京站稳了脚跟, 顺利从刚毕业的懵懂小生过渡到现在略有成熟的青年. 而如今我要展开脚步, 去追求梦想的工作了, 所以我创建了该项目, 既是对自我的激励监督, 也是向分享我的学习历程.