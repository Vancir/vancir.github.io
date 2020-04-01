---
title: Google的AFL的仓库文档里到底说了什么?
date: 2020-04-01 15:11:17
tags:
---

## AFL的安装说明

文档给出了`Linux/BSD/MacOS/Solaris/Non-x86 Systems/Others`这6个情景下的安装说明. 但是就x86架构而言, 我只想说除开Linux外其他都是邪教. 当然对于其他情景比如非x86架构其实我觉得也是值得一提的(主要是ARM架构, 以Android和一众ARM设备为代表). 什么? 你说我Windows系统就不配拥有姓名吗? 抱歉其实有另一个`WinAFL`的项目, 不过那个是基于`DynamoRIO`实现的, 暂且不谈.

所以接下来我就主要简述这两种情景为主. 

### Linux x86

如果是在Linux的话, 安装简单自然不必多说, 只需要准备好依赖的构建工具(make)和编译器(gcc或clang)即可. 编译器如果选择clang的话, 可以启用LLVM模式, 该模式会有显著的性能提升. 

``` bash
git clone https://github.com/google/AFL && cd AFL
make 
sudo make install
```

#### LLVM模式安装

LLVM模式下的安装需要`clang`和`llvm-config`工具的支持. 你需要将`llvm-config`的位置放在`PATH`环境变量里, 或者用`LLVM_CONFIG`环境变量指向. 如果LLVM的安装出了困难, 你也可以去官网下载预编译好的二进制文件来使用. 

设置好环境后进入AFL源码目录下, 只需运行`make`即可开始编译. 编译完成会生成`afl-clang-fast`和`afl-clang-fast++`. 接着你就可以用这两个程序来对第三方的代码进行插桩. 比如像下面这样:

``` bash
CC=/path/to/afl/afl-clang-fast ./configure [...options...]
make
```

如果是C++程序的话, 就将`CXX`(而非`CC`)设置为`afl-clang-fast++`即可. 


### 非x86架构

非x86架构无法按标准安装步骤进行安装. 但是依然有以下两种方式可供选择:

1. LLVM模式. LLVM不需要依赖特定的x86指令, 并且十分高效和健壮. 
2. QEMU模式. 该模式还可以用于对跨平台的二进制文件进行fuzz, 但是相对更慢和脆弱, 没有源码也能使用.

如果在有源码的条件下推荐使用LLVM模式. 使用如下命令编译AFL:

``` bash
AFL_NO_X86=1 gmake && gmake -C llvm_mode
```

并使用`afl-clang-fast`或`afl-clang-fast++`来编译你的测试目标程序.


## 快速使用

编译安装好`AFL`后, 就可以以一个小的使用demo来快速上手这个工具了. 以下都是针对于有源码的目标程序进行插桩和fuzz:

1. 使用`afl-gcc`(或其他的afl工具)来编译目标程序(插桩用). 常见的方式如下: 
    ``` bash
    CC=/path/to/afl-gcc CXX=/path/to/afl-g++ ./configure --disable-shared
    make
    ```
2. 创建一个精简有效的输入样例. 如果输入有指定的格式要求(比如SQL/HTTP等), AFL也提供了方式创建字典进行描述, 但这里是快速使用的阶段, 我们大致了解使用流程为紧, 后续内容再详细描述. 
3. 如果程序的输入是通过`stdin`来读取, 那么可以用以下方式来运行:
    ``` bash
    ./afl-fuzz -i testcase_dir -o findings_dir -- \
        /path/to/tested/program [...program's cmdline...]
    ```

4. 如果程序输入是通过`文件`进行读取, 那么也只需要在上述命令后面添加`@@`即可. AFL会帮你填上自动生成文件的路径名.
5. 然后就静观AFL的数据看板跑跑跑就可以啦!


## 性能优化要点

### 01 精简测试样例

大型的测试样例会使得fuzz的整体效率大打折扣, 尽量地精简测试样例. 如果你实在是需要使用大型, 第三方的语料库作为输入, 那么请使用`afl-cmin`并设置适当的超时限制. 

### 02 选择小型目标

选择目标时尽量选择小型的目标. 比如两个程序功能相近, 优先选择小型目标能带来显著的性能提升. 

### 03 使用LLVM模式

使用LLVM模式可以带来2倍的性能提升. 同时LLVM还提供了一种`持久进程内fuzz`的模式, 对于特定的自包含库文件可以提升5-10倍的性能. 而对于启动开销大的目标程序, 则提供了`延迟启动fork server`的模式可以带来提升. 这两种模式都仅需编辑少数一两行策略代码即可启用. 

### 04 剖析优化目标

确认目标在编译时是否有能提高性能的选项或设置. 如果检查过选项后还是很慢, 可以通过`strace -tt`或类似的方法去检查二进制文件是否在做一些笨拙的事. 有时你可以单单将配置文件指定为`/dev/null`, 或者关闭一些对测试无关的功能, 就可以显著地提升性能. 常见的资源消耗大多是`exec*()`, `popen()`, `system()`, `sleep()`或类似的调用带来的. 

测试时可以暂时关闭`ASAN`, 只在手动检查生成的语料库的时候单独拿一个启用了ASAN的二进制文件进行测试. 

### 05 缩减插桩范围

只针对你需要进行fuzz的功能/库文件进行插桩, 而在不必要fuzz的部分使用正常未插桩的二进制文件即可. 

### 06 并行fuzz

AFL是为单核处理单任务设计的, 但是在多核机器上完全可以将fuzz任务并发运行. `afl-gotcpu`可以帮助测量空闲的CPU情况. 

### 07 限制内存和超时

在合理的区间内, 使用少的内存和超时能确保测试能更有效地进行, 而避免一些畸形输入占用和花费大量的时间和内存. 

### 08 检查系统配置

1. 系统负载. 尽量关闭不必要的程序占用CPU.
2. 磁盘读写. fuzzer会频繁读写访问磁盘文件.
3. CPU性能模式. Linux默认"按需"调整CPU的性能, 你可以用以下方式将CPU设置为性能模式:   
   
   ``` bash
    cd /sys/devices/system/cpu
    echo performance | tee cpu*/cpufreq/scaling_governor
   ```

4. 透明大页. 一些内存分配器会启用透明大页, 这会严重降低fuzz性能. 可以使用`echo never > /sys/kernel/mm/transparent_hugepage/enabled`进行关闭.
5. 任务调度策略. 不同策略带来的提升也不一样, 通常`SCHED_RR`能带来提升. 按以下方式设置:
    
    ``` bash
    echo 1 >/proc/sys/kernel/sched_child_runs_first
    echo 1 >/proc/sys/kernel/sched_autogroup_enabled
    ```

### 09 还是不行?试试-d

如果还是不给力, 那么就使用`-d`选项, 这样能跳过所有的确定性模糊测试步骤, 虽然这会使得测试不够深入. 

