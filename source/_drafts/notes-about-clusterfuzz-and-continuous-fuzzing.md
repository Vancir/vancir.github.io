---
title: 关于ClusterFuzz以及规模化持续Fuzzing的笔记
---

ClusterFuzz的魅力在于它向我打开了一扇大门, 在降低了Fuzzing的使用门槛的同时让Fuzzing易于规模化, 只需要编写少量的代码就能让fuzzer持续地运行下去为你服务, 将过程中的崩溃等统计信息实时地展现在你面前, 我觉得这极具诱惑力并且蕴含着庞大的价值, 这就像安全研究的一个梦想, 一直督促我去学习这方面的内容. 

## 持续Fuzzing

* Fuzzing非常善于通过探索非预期的状态来找到Bugs.
* 应当将Fuzzing无缝地嵌入到软件开发的生命周期里去, 用户提交类似单元测试一般的Fuzzer源码, 能通过Fuzzing规模化产出Bugs, Fuzzing Statistics和Code Coverage.
* ClusterFuzz的目的在于将Fuzzing生命周期中, 除“fuzzer编写”和“Bug修复”过程外的所有流程都自动化运行起来. 
  
## Fuzzing生命周期: 

### 0x01 Write fuzzers

fuzzing分为Blackbox fuzzing, Greybox fuzzing和Structure-aware fuzzing. 而对于Fuzzing规模化重要的不是增加服务器的CPU核心数, 而是引导开发者学习使用Fuzzing, 提供丰富的文档和示例便于编写Greybox fuzzer, 提供高效fuzzing的建议(种子池,字典等), 让Greybox fuzzing成为像单元测试那样的一等公民. 

### 0x02 Build fuzzers

使用编译时插桩(ASan, MSan, 覆盖率插桩等), 跟fuzzing引擎或驱动链接起来(libFuzzer: `clang -fsanitize=address,fuzzer`). 确保Release版本经过了充分fuzzing, 持续地构建fuzzer(理想情况是能加入到已有的CI流程).

### 0x03 Fuzz at scale

* Fuzzing任务管理: 可抢占VMs源源不断地产出新的Crashes, 非抢占式VMs则对Crashes进行处理(Minimize, Bisect等), 两种VMs都会将信息写入到Task queue以及DB上.
* 挑选目标: 大型项目可能会包含有成千上万的fuzz目标, 因此需要具备自动发现fuzz目标的能力, 并且能根据fuzz目标的质量进行优先级挑选(高产出目标>低产出目标>无法正常启动的目标). 同时也可以针对Sanitizer进行优先级排列(ASan>MSan>Others(UBSan/CFI/TSAN))
* Fuzzing策略: 并没有完美的启发式搜索策略, 可参考的策略包含Corpus subset, Value profiling, Custom mutators, Limiting maximum length of inputs. 另外ClusterFuzz里还采用里Radamsa mutator和ML-based RNN mutator来增强Corpus.
* Fuzzing策略选择: 多臂老虎机(MAB)能够减少在低效fuzzing策略上的资源浪费, 尽量选择提高覆盖率的策略组合. 
* Triage crashes: 
  * De-duplication: 基于Stacktraces对崩溃进行去重, 选取前3“有趣”的栈帧作为CrashState, 包含debug和release断言, 去除内联frames, 公共库和debug函数. 忽略OOM和超时相关的Stacktrace. 
  * Grouping: 第二阶段的去重(相对较慢), 使用编辑距离将所有相似的Crashes划分为同一个组, 去除那些只有轻微差异的相同crash, 就实际效果来看还不错. 
  * Testcase minimization: 缩减Testcase更利于root cause分析. Greybox fuzzer通常会提供工具进行快速的缩减, 而Blackbox fuzzers则会使用基于Delta debugging的缩减方法, 效率低一些但胜在可以并行化.


### 0x04 Triage crashes

### 0x05 Improving fuzzers


## 参考资料: 

1. [ClusterFuzz: Fuzzing At Google Scale](https://i.blackhat.com/eu-19/Wednesday/eu-19-Arya-ClusterFuzz-Fuzzing-At-Google-Scale.pdf)
1. [Analyzing Chrome Crash Reports At Scale](https://nullcon.net/website/archives/ppt/goa-15/analyzing-chrome-crash-reports-at-scale-by-abhishek-arya.pdf)
1. [An Empirical Study of OSS-Fuzz Bugs](https://arxiv.org/abs/2103.11518)
1. [OSS-Fuzz: Google's continuous fuzzing service for open source software](https://www.usenix.org/conference/usenixsecurity17/technical-sessions/presentation/serebryany)
1. [OSS-Fuzz: Fuzzing Everything](https://f.hubspotusercontent20.net/hubfs/7466322/FuzzCon%20-%20WebSec%20Edition%20Slides/Slides%20-%20Abhishek%20Arya%20-%20OSS-Fuzz%20Fuzzing%20Everything.pdf)
1. [OSS-Fuzz: 容器和云计算在模糊测试中的应用](http://bos.itdks.com/204cab8281b4406980b5fb2dc470b7b4.pdf)
1. [Fuzzing OpenSSL](https://courses.csail.mit.edu/6.857/2019/project/11-Chen-Kim-Lam.pdf)
