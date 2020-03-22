---
title: KLEE中Seed模式相关选项介绍
date: 2018-09-09 15:23:20
tags:
---

测试使用的命令为:

``` bash
klee -seed-out=./klee-out-0/test000002.ktest -only-seed -only-replay-seeds get_sign.bc
```

测试结果:

``` bash
varas@varas-virtual-machine:~/Downloads/klee/examples/get_sign$ klee -seed-out=./klee-out-0/test000002.ktest -only-seed -only-replay-seeds get_sign.bc 
KLEE: output directory is "/home/varas/Downloads/klee/examples/get_sign/klee-out-6"
KLEE: Using STP solver backend
KLEE: KLEE: using 1 seeds

KLEE: seeding done (0 states remain)

KLEE: done: total instructions = 21
KLEE: done: completed paths = 1
KLEE: done: generated tests = 1
varas@varas-virtual-machine:~/Downloads/klee/examples/get_sign$ cd klee-out-6/
varas@varas-virtual-machine:~/Downloads/klee/examples/get_sign/klee-out-6$ ls
assembly.ll  info  messages.txt  run.istats  run.stats  test000001.ktest  warnings.txt
varas@varas-virtual-machine:~/Downloads/klee/examples/get_sign/klee-out-6$ ktest-tool test000001.ktest 
ktest file : 'test000001.ktest'
args       : ['get_sign.bc']
num objects: 1
object    0: name: 'a'
object    0: size: 4
object    0: data: '\x01\x01\x01\x01'
varas@varas-virtual-machine:~/Downloads/klee/examples/get_sign/klee-out-6$ 
```

## -seed-out

**-seed-out=**指定的seed文件会保存到**SeedOutFile**中, 在main函数中进行使用. 

``` c++
  if (!ReplayKTestDir.empty() || !ReplayKTestFile.empty()) { ...
  } else {
    // 指定了 SeedOutFile 读取 seed 内容
    std::vector<KTest *> seeds;
    for (std::vector<std::string>::iterator
           it = SeedOutFile.begin(), ie = SeedOutFile.end();
         it != ie; ++it) {
      KTest *out = kTest_fromFile(it->c_str());
      if (!out) {
        klee_error("unable to open: %s\n", (*it).c_str());
      }
      // SeedOutFile 方式获取 seed
      seeds.push_back(out);
    }
    for (std::vector<std::string>::iterator ...
    }

    if (!seeds.empty()) {
      klee_message("KLEE: using %lu seeds\n", seeds.size());
      // 使用 seed
      interpreter->useSeeds(&seeds);
    }
    if (RunInDir != "") { ...
    }
    // 运行main函数
    interpreter->runFunctionAsMain(mainFn, pArgc, pArgv, pEnvp);

    while (!seeds.empty()) {
      kTest_free(seeds.back());
      seeds.pop_back();
    }
  }
```

## -only-seed

**-only-seed**与布尔值**OnlySeed**绑定, 默认关闭. 

描述是**在seed模式结束后, 不做常规搜索直接停止执行**

``` c++
  cl::opt<bool>
  OnlySeed("only-seed",
	   cl::init(false),
           cl::desc("Stop execution after seeding is done without doing regular search (default=off)."));
```

仅在**Executor:run()**里使用到

``` c++
    if (OnlySeed) {
      doDumpStates();
      return;
    }
```

再来看**doDumpStates()**

``` c++
void Executor::doDumpStates() {
  if (!DumpStatesOnHalt || states.empty())
    return;

  klee_message("halting execution, dumping remaining states");
  for (const auto &state : states)
    terminateStateEarly(*state, "Execution halting.");
  updateStates(nullptr);
}
```

- **DumpStatesOnHalt**是默认开启的选项, 在退出时将所有活动的状态都dump成测试用例文件.
- 之后的操作是相当于提前终止, 因为**onlySeed**紧接着**seeding done**执行, 因此可以确保在seed模式执行结束后终止其他state

## -only-replay-seeds

**-only-replay-seed**与布尔值**OnlyReplaySeeds**绑定, 默认关闭. 

描述是**执行时忽略不含有seed的状态**

``` c++
  cl::opt<bool>
  OnlyReplaySeeds("only-replay-seeds",
		  cl::init(false),
                  cl::desc("Discard states that do not have a seed (default=off)."));
```

仅在**Executor::branch**和**Executor::fork**有用到

``` c++
    if (OnlyReplaySeeds) {
      for (unsigned i=0; i<N; ++i) {
        if (result[i] && !seedMap.count(result[i])) {
          terminateState(*result[i]);
          result[i] = NULL;
        }
      } 
    }
```

手动指定**OnlyReplaySeeds**可以终止状态

``` c++
// Fix branch in only-replay-seed mode, if we don't have both true
// and false seeds.
if (isSeeding && 
      (current.forkDisabled || OnlyReplaySeeds) && 
      res == Solver::Unknown) {
    bool trueSeed=false, falseSeed=false;
    // Is seed extension still ok here?
    for (std::vector<SeedInfo>::iterator siit = it->second.begin(), 
           siie = it->second.end(); siit != siie; ++siit) {
      ref<ConstantExpr> res;
      bool success = 
        solver->getValue(current, siit->assignment.evaluate(condition), res);
      assert(success && "FIXME: Unhandled solver failure");
      (void) success;
      if (res->isTrue()) {
        trueSeed = true;
      } else {
        falseSeed = true;
      }
      if (trueSeed && falseSeed)
        break;
    }
    if (!(trueSeed && falseSeed)) {
      assert(trueSeed || falseSeed);
      
      res = trueSeed ? Solver::True : Solver::False;
      addConstraint(current, trueSeed ? condition : Expr::createIsZero(condition));
    }
  }
```