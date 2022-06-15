---
layout: post
title: 回到2015年看看那时ClusterFuzz的设计
date: 2021-04-17 11:59:38
---


2015年彼时ClusterFuzz尚不够成熟但也已经是初见规模, ClusterFuzz作为Google目前开发了11年的平台, 其大概是19年才算是得以广泛运用. 网上公开了Abhishek Arya于2015年NullCon的[分享PPT](https://nullcon.net/website/archives/ppt/goa-15/analyzing-chrome-crash-reports-at-scale-by-abhishek-arya.pdf), 这应该也是他首次全面地介绍了ClusterFuzz的模块设计, 发展至今依旧保留来很多内容, 也是全网对于ClusterFuzz设计最深入的一份资料. 如果你发现NullCon官网上该PDF的[链接](https://nullcon.net/website/archives/ppt/goa-15/analyzing-chrome-crash-reports-at-scale-by-abhishek-arya.pdf)已经失效, 那么你可以点击以下链接进行访问: [ClusterFuzz](https://www.yumpu.com/en/document/read/37169541/analyzing-chrome-crash-reports-at-scale-by-abhishek-arya)

## Architectural Overview

![architectural-overview.png](https://i.loli.net/2021/04/16/qfylSnuEOhxFBHP.png)

首先ClusterFuzz进行了前后端分离, 而所谓的AppEngine其实是Google云平台的一项服务, 相当于提供了一个平台让开发者去运行自己的代码而无需关心平台的维护. ClusterFuzz不可免地依赖了Google的不少云服务, 像图片里还有Google Compute Engine, Google Cloud Storage, Datastore, Blobstore就是如此. 

前端提供了以下模块: 

* ClusterFuzz UI: 前端的展示界面, 用户可以进行操作, 彼时的UI相比现在而言粗糙不少, 但展示文字内容肯定也没什么问题了. 
* Task Pull Queues: 这应该是ClusterFuzz的任务队列, ClusterFuzz的后端定义了众多功能不同的task. 
* High replication datastore: 存储持续fuzzing过程中产生的各种元数据, 其实对应的是如今的NDB. 
* Blobstore: Blobstore现在应该是属于Google弃用的存储服务. 现今版本的ClusterFuzz在Blob上也是基于NDB实现的, 但也是因为这个历史原因, 所以单独抽出来作为blobs.py并保留了部分旧版本兼容代码. Blob则是指一些二进制数据, 比如binary, testcase, fuzzers, data bundles等. 

后端则是从Task Pull Queues里获取Task并以Bot为单位执行, 其中Fuzz Task则是负责对Fuzz Target进行测试. 而这里Bots的Local Storage从我认知来看已经移除了, 目前保存在本地的数据应该只剩下缓存数据以及状态监控统计数据, 其他的数据基本都上传到云服务去了. 

Glusterfs是一项分布式文件系统, 彼时被用于同步Bots之间的数据. 而目前版本已经移除了Glusterfs, 我自己也尚不清楚目前是如何处理Bots之间的数据同步问题, 印象里似乎并没有同步Bots之间的数据. 这有待我后续仔细阅读这部分的源码得出结论. Builder Bots也让我很迷惑, 但从名称来看应该是用于对源码的构建. 在目前的ClusterFuzz版本中我也尚未发现有这部分相关代码. 

## ClusterFuzz的实现目标

ClusterFuzz有三个主要目标: 

1. 自动完成Crash的检测, 分析和管理
2. 可复现的Crash结果以及最小化的Testcases
3. 实时的回归测试以及修复性测试

### 0x01 Crash自动化检测/分析/管理

#### Fuzzer编写要求

Fuzzer应当具备相同的使用命令以供调用, 比如`run.* --input_dir=A --output_dir=B --no_of_files=C`设计一套统一的命令行选项. 定义标签前缀, 比如“fuzz-”, “http-”, “flag-”等等. 具备跨平台性, 支持对多种编程语言开发的软件进行Fuzz. 使用Data Bundle来统一管理Fuzz资源, 并且也可以提供其他的脚本比如launcher, coverage等.

#### 持续Fuzz平台的功能

设置好环境并对项目代码进行构建, 能够运行应用程序以及相关的测试代码. 能调整平台的参数比如设置手势, 设置工具的选项, 增加超时限制等. 能解决程序等资源依赖问题, 并对产出的Crash进行复现和去重, 将Crash, 覆盖率, 统计数据等进行妥善保存等.

#### Testcase的重复判定

禁用inline frames, 比如llvm-symbolizer关闭选项-inlining. 根据Crash Type, Crash State以及Security Flag来对Crash进行去重. 这里Crash State则是崩溃时StackTraces的前3个Frame, 并进行了一定的归一化, 比如保留namespace, 移除具体的line_numbers后作为特征进行去重. (这里的Security Flag我尚不清楚具体指的是什么, 但我想根据Crash Type和Crash State应该就能做到不错的去重效果了.)

### Fuzzer的类型

Fuzzer分为三种: 基于生成的Fuzzer, 基于变异的Fuzzer, 可进化的Fuzzer. 

基于生成的Fuzzer适用性更窄, 依赖于具体的格式或API. 能够迅速的找到Bug, 但无效也更快, 并且需要编写复杂的策略提高效果, 适合用于检验回归测试. 

基于变异的Fuzzer则依赖于初始测试的样例, 而策略比较简单, 能够很好地适用于文件格式fuzz, 协议fuzz, 并且可以源源不断地挖掘Bugs. 但也有一些难以解决的情况比如计算checksum, 数据压缩操作等. 

进化式Fuzzer则是基于反馈的Fuzzer, 目前主流的反馈指标则是使用的代码覆盖率. 代码在编译时会进行插装来反馈测试样例运行时的覆盖率信息, 并且通过共享存储聚合多个代码之间的覆盖率情况. 

代码覆盖率部分彼时还是使用的fuzzer_utils来控制, 但从目前来看这些这些API都有了比较大的变化. 彼时针对Testcase会有以下规则: 对于原始Testcase, 如果增加了新的覆盖分支, 那么就将其加入到Optimal文件列表里去, 而如果该测试用例没有产生新的覆盖的话, 那么就会将其删除. 而对于Testcase发生改动并且产出了新覆盖率的时候, 就会将其加入到Corpus和Optimal文件列表中去. (无新覆盖分支那么就直接忽略.)

### 内存调试工具

Valgrind会产生10-300x的性能开销, 启动缓慢并且只能适用于堆漏洞, 因此并没有选择使用Valgrind. 

ClusterFuzz选择了结合使用多种内存调试工具. 

* AddressSanitizer(ASan): 检测UAF, Buffer Overflows(heap, stack, globals), Stack-Use-After-Return, Container-Overflow等问题. CPU开销2x, 内存开销1.5x-3x
* ThreadSanitizer(TSan): 检测Data Race, ESP on UAF, object vptr. CPU开销4x-10x, 内存开销5x-8x
* MemorySanitizer(MSan): 检测Uninitialized Memory Reads. CPU开销3x, 内存开销2x
* UndefinedBehaviorSanitizer(UBSan): 检测多约19种类型的bugs, esp on type confusion等. 
* 还提高了一些其他的工具, 但我想应该并没有使用上: SyzyASAN以及DrMemory. 

### 0x02 可复现的Crash结果以及最小化Testcases

作者提出一种方式叫做Delta Debugging可以并行多线程地进行Minimize任务. 并且支持为某些文件类型进行Minimizer的定制. 

Minimizer首先会将Input进行token化, 并假定某组Token并不是Crash所需要的, 假设移除掉这些Token后再次执行, 如果程序崩溃, 那么我们的移除就是可靠的. 而没有复现Crash, 那么就将移除的Token再细分成更小的组.

 ### 0x03 实时的回归测试以及修复性测试

作者提出了一个工具叫FindIt, 可以用于找出崩溃相关的ChangeLog(CL). 它的执行过程如下: 

1. 解析StackTrace获取相关的文件, 对应的崩溃代码行号. 
2. 在回归测试范围内解析ChangeLog, 获得代码文件相关的CL编号.
3. 根据ChangeLog和之前解析的StackTrace, 对于一个CL编号, 如果该次改动影响了StackTrace中解析出来的代码行号位置, 那么就认为这个CL是可疑的. 并提供一个可疑CL列表. 
4. 如果StackTrace没有关联到相应的Cl编号, 那么就展示Blame信息看谁负责该处代码. 

