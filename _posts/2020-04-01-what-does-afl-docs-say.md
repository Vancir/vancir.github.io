---
layout: post
title: Google的AFL仓库文档到底说了什么?
date: 2020-04-01 15:11:17
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

## 状态看板

以下是一个AFL模糊测试过程中的状态看板: 

{% include figure.html path="https://i.loli.net/2020/10/25/n2lhwZR1HOrWYUN.png" class="img-fluid rounded z-depth-1" zoomable=true %}

数据看板有很好地颜色来区分重要等级, 看板也有不同的小的板块组成, 那么接下来就来讲讲各个板块. 

1. `Process timing`: 就是一个时间统计的板块. 不过需要注意的是, 如果在刚开始fuzz有一小会之后, 还是没有fuzz出新的路径的话, 那么很有可能是目标程序没有被正确地运行起来. 可能的原因是没能正确地解析输入, 另一个就是给定的内存太少, 程序无法载入内存.
2. `Overall results`: 显示fuzzer目前总计进行的论数, fuzz出了多少有趣的测试用例, 有多少各异的崩溃. 需要注意的是`cycles done`的颜色: 它在最初是洋红色, 随后如果有新发现就会变成黄色, 蓝色以此类推. 直到很长时间没有新发现了才会变成`绿色`
3. `Cycle progress`: 显示当前的测试用例ID以及超时的路径数量. 有时会在第一行显示`*`后缀表明该当前处理的路径不是`首选`的.
4. `Map coverage`: 提供覆盖率情况. `map density`指示目前已经命中多少分支元组.
5. `Stage progress`: 这个板块能告知fuzzer目前正处在的阶段. fuzzer的阶段有以下几种:
   1. `calibration`校验: 在模糊测试前进行的阶段, 检查执行路径以发现异常, 评估基准速度等. 
   2. `trim L/S`: 另一个预准备阶段, 测试用例会被裁剪为同效的最简形式. 
   3. `bitflip L/S`: 比特翻转. 在任意时间翻转L个比特, S个比特的步长遍历输入文件.
   4. `arith L/8`: 算数操作. fuzzer对`8/16/32`比特的数值加减某个小整数.
   5. `extras`: 填充字典项. fuzzer根据是使用用户提供的字典(-x)还是自动创建的字典, 来显示为"user"或"auto". 如果是覆写数据则是"over", 插入数据则是"insert".
   6. `havoc`: 随机调整. 该阶段会进行包括比特翻转, 使用"随机有趣"证书进行覆写, 块删除, 块复制, 字典等各种操作. 
   7. `splice`: 在任意选择的中点将队列中的两个随机输入拼接在一起.
   8. `sync`: 只在并行fuzz时出现, 该阶段会同步其他fuzzer的状态信息. 
6. `Findings in depth`: 显示一些度量标准, 比如有趣的路径数量, 新发现的边数量, crash总数等. 
7. `Fuzzing strategy yields`: 模糊测试的度量信息. 
8. `Path geometry`: 
   * `levels`: 表示导向型fuzzing过程所到达的路径深度, 路径越深表明该导向的价值越高. 
   * `pending`: 表示尚未经过任何模糊处理的输入的数量. 
   * `pend fav`: 表示fuzzer认为队列中可能有趣的输入数量. 
   * `own finds`: 表示模糊过程发现的新路径数量.
   * `imported`: 表示并行fuzz过程中从其他fuzzer中导入的新路径数量.
   * `stability`: 表示相同输入在目标程序中产生可变行为的程度, 这可以表明观察到的行为的一致性. 如果该数值较低, 就表明行为的不确定性, AFL也很难区分对输入文件变异带来的影响. 
9. `CPU load`: 显示CPU的利用率.
    

## QEMU模式

QEMU模式是AFL能进行黑盒模糊测试的关键, 在无法获取源码使用`afl-gcc`构建的时候, 就需要`QEMU`这样的全平台模拟工具来运行二进制文件和测试. 不过全平台模拟带来的开销也是可怕的(2到5倍的性能开销, 但也好过`DynamoRIO`和`PIN`). 

启用该模式需要对QEMU源码进行补丁, AFL选用的QEMU版本为`2.10.0`, 并提供了脚本`qemu_mode/build_qemu_support.sh`下载/配置/编译QEMU工具. 一旦安装完成, 即可通过`afl-fuzz`使用`-Q`选项启用QEMU模式. 

当然QEMU是平台无关的, 因此你可以在运行`build_qemu_support.sh`之前设置`CPU_TARGET`环境变量以构建特定架构的QEMU支持. 比如`CPU_TARGET=arm`

插桩过程仅针对链接过程中遇到的第一个ELF文件的`.text`代码段, 故而`afl-fuzz`不会去跟踪共享库文件, 也就是说:

1. 你要分析的任何库文件, 都必须以静态链接的方式编入ELF文件中去. 不过好在大部分的闭源程序都会这样干. 
2. 标准C库和其他不必关系的库应该动态链接的方式加载, 否则AFL将不可避免在此处产生开销. 

设置`AFL_INST_LIBS=1`可以用来绕过`.text`检测逻辑, 进而检测遇到的每个基本块. 

### 坑点

1. 如果目标程序带有文件校验, 插桩会改动文件本身, 那么你需要修复文件的校验或者移除相应的校验代码. 
2. 不要将QEMU和`ASAN/MSAN`或类似技术混用起来, 因此QEMU不兼容这些sanitizer的`Shadow VM`技术, 混用会使得QEMU占光内存. 
3. QEMU不能确保安全性, 对于不可信的二进制需要先在沙箱和杀软运行一遍. 

### 二进制重写

在QEMU模拟执行前重写二进制, 在汇编代码内写入插桩代码, 能使得运行更快性能更高. 但是目前的二进制重写困难重重, 它需要正确且完整地重新建模程序的控制流, 所以这是一个可以努力的方向. 如果你想尝试的话, 也可以试试 [afl-dyninst](https://github.com/vrtadmin/moflow/tree/master/afl-dyninst). 

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

## 并行Fuzz

单个`afl-fuzz`只会占用1个CPU, 因此在多核系统下可以同时进行多个fuzz任务. 如果只是针对多个不相关的二进制进行fuzz的话, 只需要运行多个`afl-fuzz`程序即可, 但如果想针对同一个目标的话, fuzz的信息需要共享给其他的`afl-fuzz`实例. 

### 单系统并行

如果是在单个机器上对单个目标进行并发fuzz, 那么只需要创建一个所有`afl-fuzz`可用的共享文件夹, 然后指定每一个`afl-fuzz`实例命名即可. 

比如运行第一个实例:

``` bash
./afl-fuzz -i testcase_dir -o sync_dir -M fuzzer01 [...other stuff...]
```

然后类似地创建接二连三的实例:

``` bash
./afl-fuzz -i testcase_dir -o sync_dir -S fuzzer02 [...other stuff...]
./afl-fuzz -i testcase_dir -o sync_dir -S fuzzer03 [...other stuff...]
```

而每个实例会将它的状态单独保存在分隔的子文件夹内, 类似`/path/to/sync_dir/fuzzer01/`这样. 每个实例会周期性地扫描共享目录内的测试样例, 对于感兴趣的测试用例就会加入到自身的fuzz过程. `-M`是指定`master`, 而`-S`是指定`slave`. 主从实例的区别在于主实例会执行确定性检查, 而仆实例只进行随机调整. 

当然如果你也可以用`-S`来运行所有实例也没关系, 对于非常慢/复杂的目标这样做常常效果不错. 但是如果你想运行多个主实例的话, 那么就会带来巨大的资源浪费. 并且需要像以下这样运行`afl-fuzz`实例.

``` bash
./afl-fuzz -i testcase_dir -o sync_dir -M masterA:1/3 [...]
./afl-fuzz -i testcase_dir -o sync_dir -M masterB:2/3 [...]
./afl-fuzz -i testcase_dir -o sync_dir -M masterC:3/3 [...]
```

AFL还提供了`afl-whatsup`工具用来监视各个任务的进展情况. 不过还有一点要注意: 如果你在并发时使用`-f`指定输入文件的话, 需要确保文件相互独立. 不过要是你用`@@`不带`-f`的话就不会有这样的问题. 

### 多系统并行

多系统并行和单系统类似, 关键区别在于需要额外编写一个简单脚本, 执行以下两个动作来同步各个fuzz实例的状态:

1. 使用SSH在机器之间同步共享文件夹内的状态文件. 

    ``` bash
    for s in {1..10}; do
        ssh user@host${s} "tar -czf - sync/host${s}_fuzzid*/[qf]*" >host${s}.tgz
    done
    ```

2. 在剩余机器之间分发状态文件.

    ``` bash
    for s in {1..10}; do
        for d in {1..10}; do
            test "$s" = "$d" && continue
            ssh user@host${d} 'tar -kxzf -' <host${s}.tgz
        done
    done
    ```

示例可以参考`experimental/distributed_fuzzing/`下的脚本, 或者还有另一个具有特色的实验性工具 [disfuzz-afl](https://github.com/MartijnB/disfuzz-afl), 以及另一种客户端-服务端实现 [roving](https://github.com/richo/roving).
