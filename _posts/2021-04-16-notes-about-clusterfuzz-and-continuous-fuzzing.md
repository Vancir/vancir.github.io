---
layout: post
title: 关于ClusterFuzz以及规模化持续Fuzzing的笔记
date: 2021-04-16 22:43:14
tags: ["clusterfuzz", "oss-fuzz"]
---


ClusterFuzz的魅力在于它向我打开了一扇大门, 在降低了Fuzzing的使用门槛的同时让Fuzzing易于规模化, 只需要编写少量的代码就能让fuzzer持续地运行下去为你服务, 将过程中的崩溃等统计信息实时地展现在你面前, 我觉得这极具诱惑力并且蕴含着庞大的价值, 这就像安全研究的一个梦想, 一直督促我去学习这方面的内容. 以下我将就OSS-Fuzz负责人Abhishek Arya于Black Hat Europe 2019发表的议题[ClusterFuzz: Fuzzing At Google Scale](https://i.blackhat.com/eu-19/Wednesday/eu-19-Arya-ClusterFuzz-Fuzzing-At-Google-Scale.pdf)进行阐述.

## 持续Fuzzing

* Fuzzing非常善于通过探索非预期的状态来找到Bugs.
* 应当将Fuzzing无缝地嵌入到软件开发的生命周期里去, 用户提交类似单元测试一般的Fuzzer源码, 能通过Fuzzing规模化产出Bugs, Fuzzing Statistics和Code Coverage.
* ClusterFuzz的目的在于将Fuzzing生命周期中, 除“fuzzer编写”和“Bug修复”过程外的所有流程都自动化运行起来. 
  
## Fuzzing生命周期: 

### 0x01 Write fuzzers

fuzzing分为Blackbox fuzzing, Greybox fuzzing和Structure-aware fuzzing. 而对于Fuzzing规模化重要的不是增加服务器的CPU核心数, 而是引导开发者学习使用Fuzzing, 提供丰富的文档和示例便于编写Greybox fuzzer, 提供高效fuzzing的建议(种子池,字典等), 让Greybox fuzzing成为像单元测试那样的一等公民. 

### 0x02 Build fuzzers

使用编译时插桩(ASan, MSan, 覆盖率插桩等), 跟fuzzing引擎或驱动链接起来(libFuzzer: clang -fsanitize=address,fuzzer). 确保Release版本经过了充分fuzzing, 持续地构建fuzzer(理想情况是能加入到已有的CI流程).

### 0x03 Fuzz at scale

* Fuzzing任务管理: 可抢占VMs源源不断地产出新的Crashes, 非抢占式VMs则对Crashes进行处理(Minimize, Bisect等), 两种VMs都会将信息写入到Task queue以及DB上.
* 挑选目标: 大型项目可能会包含有成千上万的fuzz目标, 因此需要具备自动发现fuzz目标的能力, 并且能根据fuzz目标的质量进行优先级挑选(高产出目标>低产出目标>无法正常启动的目标). 同时也可以针对Sanitizer进行优先级排列(ASan>MSan>Others(UBSan/CFI/TSAN))
* Fuzzing策略: 并没有完美的启发式搜索策略, 可参考的策略包含Corpus subset, Value profiling, Custom mutators, Limiting maximum length of inputs. 另外ClusterFuzz里还采用里Radamsa mutator和ML-based RNN mutator来增强Corpus.
* Fuzzing策略选择: 多臂老虎机(MAB)能够减少在低效fuzzing策略上的资源浪费, 尽量选择提高覆盖率的策略组合. 

### 0x04 Triage crashes

* De-duplication: 基于Stacktraces对崩溃进行去重, 选取前3“有趣”的栈帧作为CrashState, 包含debug和release断言, 去除内联frames, 公共库和debug函数. 忽略OOM和超时相关的Stacktrace. 
* Grouping: 第二阶段的去重(相对较慢), 使用编辑距离将所有相似的Crashes划分为同一个组, 去除那些只有轻微差异的相同crash, 就实际效果来看还不错. 
* Testcase minimization: 缩减Testcase更利于root cause分析. Greybox fuzzer通常会提供工具进行快速的缩减, 而Blackbox fuzzers则会使用基于Delta debugging的缩减方法, 效率低一些但胜在可以并行化.
* Bisection: (这一段二分我不太懂, 我觉得需要阅读完ClusterFuzz源码后才能确定这里的二分是什么含义)
* Variant analysis: Crash可以在sanitizers, fuzzing engines, platforms, architectures体现出不同的特征. 
* Automatic bug filing: 提供最小化的复现过程以及详细的崩溃报告
* Prioritization: 不要对Bugs做过于深入的分析, 因为这样并不适合大规模的任务. 最好是假设所有的内存破坏都是可利用的, 将这些问题抛出来让人来判断. 基于崩溃的类型以及崩溃发生点来对崩溃进行简单的优先级排序, 将更有价值的崩溃优先呈现出来.
* Fix verification: 验证Fix是否会造成崩溃, 并且关闭已经验证完毕的Bugs. 
* Vulnerability reward program: 可以提供额外的PoC来辅助漏洞的触发和缩减其他环节的任务. 让Fuzzer能持续地报告bug并产生高质量的报告.

### 0x05 Improving fuzzers

对Fuzzer的运行情况, Crash的详细信息进行数据收集和统计. 生成代码覆盖率的报告. 

## 理想的持续集成平台

* Fuzzing能融入到普通软件开发过程的一部分
* 可以结合不同的Fuzzing引擎和策略进行大规模测试
* 大型的软件项目可以仅由一个小团队负责维护的持续fuzzing平台进行高效fuzz.
* 将Fuzzing作为持续集成CI的一部分, 不断地在回归测试中捕获Bugs
* 平台也能用于对于一个fuzzer进行性能测试的解决方案
* 能持续地去提高fuzzing的效率. 
* 能支持对更多的编程语言开发的软件进行测试.
