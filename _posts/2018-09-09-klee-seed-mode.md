---
layout: post
title: KLEE中Seed模式流程解析及相关选项介绍
date: 2018-09-09 15:23:05
---

## main.cpp

``` c
  cl::list<std::string>
  SeedOutFile("seed-out");

  cl::list<std::string>
  SeedOutDir("seed-out-dir");
```

在main.cpp中首先会对ReplayKTestDir和ReplayKTestFile做判断. 如果用户有指定, 那么会继续判断SeedOutDir和SeedOutFile是否为空. 

``` c
 if (!ReplayKTestDir.empty() || !ReplayKTestFile.empty()) {
    assert(SeedOutFile.empty());
    assert(SeedOutDir.empty());
     ...
     // replay ktest 文件模式
 } else {	// 没有指定replay ktest模式
     // 直接从SeedOutFile读取seed
     for(){
         KTest *out = kTest_fromFile(it->c_str());
     }
     seeds.push_back(out);
 }
```

最终读取到的seed都保存在**std::vector<KTest *> seeds**中. 然后判断是否有读取到seed

``` c
    if (!seeds.empty()) {
      klee_message("KLEE: using %lu seeds\n", seeds.size());
      // 使用 seed
      interpreter->useSeeds(&seeds);
    }
```

既然读取到了seed, 那就**interpreter->useSeeds(&seeds);**来使用seed. 这之后除了切换目录以及清理seeds的操作, 我们需要重点关注的是**interpreter->runFunctionAsMain(mainFn, pArgc, pArgv, pEnvp);**. 在interpreter设置好seed后, seed的使用过程会在**runFunctionAsMain**开始.

``` c
    if (RunInDir != "") {... // 切换目录
    }
    // 运行main函数
    interpreter->runFunctionAsMain(mainFn, pArgc, pArgv, pEnvp);

    while (!seeds.empty()) {	// 将seeds里的内容全部释放掉
      kTest_free(seeds.back());
      seeds.pop_back();
    }
```

## useSeeds

既然谈到**interpreter->useSeeds(&seeds);**那就来看看这个**interpreter**

``` c
  Interpreter *interpreter =
    theInterpreter = Interpreter::create(ctx, IOpts, handler);
```

这里实例化了一个interpreter. 而该方法的声明是在**interpreter.h**里, 实现却是在**Executor.cpp**

``` c
Interpreter *Interpreter::create(LLVMContext &ctx, const InterpreterOptions &opts,
                                 InterpreterHandler *ih) {
  return new Executor(ctx, opts, ih);
}
```

也就是说实际上就是创建了一个Executor实例. 而**Executor**继承自**interpreter类**, 像**useSeeds**和**runFunctionAsMain**都在**Executor类**里进行了定义. 所以**interpreter->useSeeds(&seeds)**实际上就是**Executor::useSeeds**, **interpreter::runFunctionAsMain**就是**Executor::runFunctionAsMain**. 那么接下来就只看**Executor**

``` c
// Executor.h
const std::vector<struct KTest *> *usingSeeds;  
void useSeeds(const std::vector<struct KTest *> *seeds) override {
  usingSeeds = seeds;
}
```

这里将seeds传递给了Executor的私有成员**usingSeeds**. 那么这也就是**interpreter->useSeeds()**所做的操作了

## runFunctionAsMain

回顾一下runFunctionAsMain的使用: **interpreter->runFunctionAsMain(mainFn, pArgc, pArgv, pEnvp);**

``` c
void Executor::runFunctionAsMain(Function *f,
				 int argc,
				 char **argv,
				 char **envp) {
  std::vector<ref<Expr> > arguments;

  // force deterministic initialization of memory objects
  srand(1);
  srandom(1);
  
  // MemoryObject 表示分配在某个地址上的对象
  MemoryObject *argvMO = 0;

  int envc;
  for (envc=0; envp[envc]; ++envc) ;  // envc表示环境变量的数量

  unsigned NumPtrBytes = Context::get().getPointerWidth() / 8;
  // 这里f对应的函数为 main, 使用KFunction进行结构化
  KFunction *kf = kmodule->functionMap[f];
  assert(kf);
  // 枚举main好参数
  Function::arg_iterator ai = f->arg_begin(), ae = f->arg_end();
  // 为argc个参数和envc个环境变量分配空间
  ......
	
  ExecutionState *state = new ExecutionState(kmodule->functionMap[f]);
  
  if (pathWriter) 
    // 初始化 记录混合符号执行路径的流 pathOS
    state->pathOS = pathWriter->open();
  if (symPathWriter) 
    // 初始化 记录单纯符号执行路径的流 pathOS
    state->symPathOS = symPathWriter->open();


  if (statsTracker)
    // 记录符号执行时的状态切换, 
    // 参数 parentFrame 为0, 说明是初始状态
    statsTracker->framePushed(*state, 0);

  assert(arguments.size() == f->arg_size() && "wrong number of arguments");
  for (unsigned i = 0, e = f->arg_size(); i != e; ++i)
    bindArgument(kf, i, *state, arguments[i]);

  // 将参数用 MemoryObject 和 ObjectState 等类进行封装, 以便klee使用
  ...... 
      
  // 初始化了一些state相关的全局变量
  initializeGlobals(*state);

  // 实例化一个当前进程的进程树
  processTree = new PTree(state);
  // 将当前进程作为进程树的根结点
  state->ptreeNode = processTree->root;
  run(*state);
  // run(*state)的善后处理
  delete processTree;
  processTree = 0;

  // hack to clear memory objects
  delete memory;
  memory = new MemoryManager(NULL);

  globalObjects.clear();
  globalAddresses.clear();

  if (statsTracker)
    statsTracker->done();
}
```

我用 **......**省略了在 **runFunctionAsMain** 中有关参数的部分, 以便更加清晰地来看这个函数. **runFunctionAsMain**的前半部分都是在做参数的相关操作, 比如为参数分配空间, 将参数进行封装, 将参数与当前状态绑定之类. 后半部分则是关于**state**的操作, 再为state进行了一些初始化操作后, 使用**run(*state)**来运行状态. 这也是**runFunctionAsMain**里我们的重要关注点. 

## run

联系上部分的**interpreter->useSeeds**, 在设置好**usingSeeds**, 而**usingSeeds**仅在**Executor::run**中使用. 那么我们就直接来分析**run()**来看seed对klee执行流程的影响

``` c
void Executor::run(ExecutionState &initialState) {
  // 对模块中每个函数的每条指令的常数进行处理
  // 将处理后的结果存储在某个数据结构中
  bindModuleConstants();

  // 初始化计时器
  initTimers();
  // 将初始状态 initialState 添加到 states 这个全局变量中
  states.insert(&initialState);

  if (usingSeeds) { ... // 有传入seed
  }
  // 创建一个searcher实例
  searcher = constructUserSearcher(*this);
  // 初始化searcher操作
  std::vector<ExecutionState *> newStates(states.begin(), states.end());
  searcher->update(0, newStates, std::vector<ExecutionState *>());

  while (!states.empty() && !haltExecution) {
    // 通过searcher采取的搜索策略来搜索执行下一个state
    ExecutionState &state = searcher->selectState();
    // ki是当前指令
    KInstruction *ki = state.pc;
    // 指向下一条指令
    stepInstruction(state);
    // 基于llvm指令集, 对指令字节码进行具体解析并模拟执行
    executeInstruction(state, ki);
    // 处理计时器
    processTimers(&state, MaxInstructionTime);
    // 检查内存使用情况
    checkMemoryUsage();
    // 对states进行更新
    // 不同的搜索策略, 其更新states的策略不同
    updateStates(&state);
  }

  // 善后工作
  delete searcher;
  searcher = 0;

  doDumpStates();
}

```

如果没有传入seed, **usingSeeds**为空, 那么**run()**的流程从上到下就是:

1. 对模块中每条指令中的常数进行绑定
2. 将初始状态加入到state中
3. 创建一个searcher实例并初始化
4. 不断循环, 直到state为空
   - 根据searcher的搜索策略选择下一个要执行的state
   - 获取当前指令, 保存为ki, 并将pc指向下一条指令
   - executeInstruction执行当前状态的当前指令
   - 处理计时器, 检查内存使用情况, 更新state
5. 清理工作

实际上, 也就是常规符号执行的正常流程. 也就是说这里并没有对seed进行处理. seed也没有影响到这里的常规执行

那么我们重点关注的地方就是**usingSeeds**部分

## usingSeeds

``` c
if (usingSeeds) {
    // 获得一个 seedMap[&initialState] 的引用
    std::vector<SeedInfo> &v = seedMap[&initialState];
    // 通过引用, 将 usingSeeds 里的 seed 内容存进 seedMap[&initialState] 中
    for (std::vector<KTest*>::const_iterator it = usingSeeds->begin(), 
           ie = usingSeeds->end(); it != ie; ++it)
      v.push_back(SeedInfo(*it));

    // 辅助用的变量 初始化
    int lastNumSeeds = usingSeeds->size()+10;
    double lastTime, startTime = lastTime = util::getWallTime();
    ExecutionState *lastState = 0;
    while (!seedMap.empty()) {
      // 中断
      if (haltExecution) {
        doDumpStates();
        return;
      }

      std::map<ExecutionState*, std::vector<SeedInfo> >::iterator it = 
        seedMap.upper_bound(lastState);
      if (it == seedMap.end())
        it = seedMap.begin();
      lastState = it->first;
      unsigned numSeeds = it->second.size();
      ExecutionState &state = *lastState;
      // ki是当前指令
      KInstruction *ki = state.pc;
      // 指向下一条指令
      stepInstruction(state);
      // 基于llvm指令集, 对指令字节码进行具体解析并模拟执行
      executeInstruction(state, ki);
      // 处理计时器
      processTimers(&state, MaxInstructionTime * numSeeds);
      // 对states进行更新
      updateStates(&state);
      
      // 每执行 1000 条语句 打印一次提示信息
      if ((stats::instructions % 1000) == 0) {...
      }
    }

    klee_message("seeding done (%d states remain)", (int) states.size());

    // XXX total hack, just because I like non uniform better but want
    // seed results to be equally weighted.
    for (std::set<ExecutionState*>::iterator
           it = states.begin(), ie = states.end();
         it != ie; ++it) {
      (*it)->weight = 1.;
    }

    if (OnlySeed) {
      doDumpStates();
      return;
    }
  }

```

再看**usingSeeds**分支内的语句. 首先通过引用, 将 usingSeeds 里的 seed 内容存进 seedMap[&initialState] 中, 然后初始化了一些辅助变量后, 进入一个循环知道seedMap为空. 那么很显然在这个循环里, 一定对seedMap进行了处理, 不然就是一个死循环了. 

来看循环里的代码, 跟之前**usingSeeds**以外的常规符号执行流程中, 最明显的区别就是没有用到searcher, 没有用到对应的搜索策略来选择下一个要执行的state. 而是使用的seedMap里各个seed对应的state. 

那么在这个循环里面, 我们检查了这几个函数. 只有**executeInstruction(state, ki);**有可能包含对seed的操作. 

因为在执行指令的时候并没有显式地将seedMap作参在执行时使用, 而seedMap是全局变量, 可以随时被调用, 而只需要比对state也能找到处理state时应该对应的seed. 因此我们就来看下整个klee工程里哪里有用到**seedMap**

``` c
branch:
lib/Core/Executor.cpp:    seedMap.find(&state);
lib/Core/Executor.cpp:  if (it != seedMap.end()) {
lib/Core/Executor.cpp:    seedMap.erase(it);
lib/Core/Executor.cpp:        seedMap[result[i]].push_back(*siit);
lib/Core/Executor.cpp:        if (result[i] && !seedMap.count(result[i])) {
fork:
lib/Core/Executor.cpp:    seedMap.find(¤t);
lib/Core/Executor.cpp:  bool isSeeding = it != seedMap.end();
lib/Core/Executor.cpp:    if (it != seedMap.end()) {
lib/Core/Executor.cpp:      std::vector<SeedInfo> &trueSeeds = seedMap[trueState];
lib/Core/Executor.cpp:      std::vector<SeedInfo> &falseSeeds = seedMap[falseState];
lib/Core/Executor.cpp:        seedMap.erase(trueState);
lib/Core/Executor.cpp:        seedMap.erase(falseState);
addConstraint:
lib/Core/Executor.cpp:    seedMap.find(&state);
lib/Core/Executor.cpp:  if (it != seedMap.end()) {
executeGetValue:
lib/Core/Executor.cpp:    seedMap.find(&state);
lib/Core/Executor.cpp:  if (it==seedMap.end() || isa<ConstantExpr>(e)) {
updateStates:
lib/Core/Executor.cpp:      seedMap.find(es);
lib/Core/Executor.cpp:    if (it3 != seedMap.end())
lib/Core/Executor.cpp:      seedMap.erase(it3);
        run:
        lib/Core/Executor.cpp:    std::vector<SeedInfo> &v = seedMap[&initialState];
        lib/Core/Executor.cpp:    while (!seedMap.empty()) {
        lib/Core/Executor.cpp:        seedMap.upper_bound(lastState);
        lib/Core/Executor.cpp:      if (it == seedMap.end())
        lib/Core/Executor.cpp:        it = seedMap.begin();
        lib/Core/Executor.cpp:               it = seedMap.begin(), ie = seedMap.end();
terminateState:
lib/Core/Executor.cpp:      seedMap.find(&state);
lib/Core/Executor.cpp:    if (it3 != seedMap.end())
lib/Core/Executor.cpp:      seedMap.erase(it3);
terminateStateEarly:
lib/Core/Executor.cpp:      (AlwaysOutputSeeds && seedMap.count(&state)))
terminateStateOnExit:
lib/Core/Executor.cpp:      (AlwaysOutputSeeds && seedMap.count(&state)))
executeMakeSymbolic:
lib/Core/Executor.cpp:      seedMap.find(&state);
lib/Core/Executor.cpp:    if (it!=seedMap.end()) { // In seed mode we need to add this as a

```

总结一下, 有这些函数用到了**seedMap**: **branch**, **fork**, **addConstraint**, **executeGetValue**, **updateStates**, **run**, **terminateState**, **terminateStateEarly**, **terminateStateOnExit**, **executeMakeSymbolic**一共十处

那么我们就一个个来看(其中**run**已经分析过, 除外), **seedMap**到底是如何影响klee的执行流程的, 以及**executeInstruction**是通过哪一种方式来操作**seedMap**

## branch

首先明确**Executor::branch**仅在**executeGetValue**和**executeInstruction**的**IndirectBr**和**switch**分支用到. 

``` c
    std::vector<ExecutionState*> branches;
    branch(state, conditions, branches);
```

作用则是在遇到分支的时候, 判断条件是否满足, 并将对应的条件复制到各自的branches里. 而在**branch**里

``` c
  if (MaxForks!=~0u && stats::forks >= MaxForks) {
    unsigned next = theRNG.getInt32() % N;
    for (unsigned i=0; i<N; ++i) {
      if (i == next) {
        result.push_back(&state);
      } else {
        result.push_back(NULL);
      }
    }
  } else {
    stats::forks += N-1;

    // XXX do proper balance or keep random?
    result.push_back(&state);
    for (unsigned i=1; i<N; ++i) {
      ExecutionState *es = result[theRNG.getInt32() % i];
      ExecutionState *ns = es->branch();
      addedStates.push_back(ns);
      result.push_back(ns);
      es->ptreeNode->data = 0;
      std::pair<PTree::Node*,PTree::Node*> res = 
        processTree->split(es->ptreeNode, ns, es);
      ns->ptreeNode = res.first;
      es->ptreeNode = res.second;
    }
  }

```

首先会判断是否超出了状态fork的最大数量**MaxForks**, 如果没有超过, 那么就会示例化一个**ExecutionState**对象**es**, 并使用**es->branch()**来分出对应的分支. 这里的**branch()**的原型是**ExecutionState::branch()**, 就不要混淆了. 

``` c
ExecutionState *ExecutionState::branch() {
  depth++;

  ExecutionState *falseState = new ExecutionState(*this);
  falseState->coveredNew = false;
  falseState->coveredLines.clear();

  weight *= .5;
  falseState->weight -= weight;

  return falseState;
}
```

以上是**ExecutionState::branch()**的代码判断. 可以看到这里以当前状态复制了一个**falseState**出来并进行了相关设置, 也就达成了分支所需要的效果.

> 在下一部分的**fork()**中也是使用**branch()**的方法对状态进行分支. 

然后我们再来看**Executor::branch**中跟seed相关的代码

``` c
  // 如有必要, 重新分配种子以满足条件 
  // 必要时根据OnlyReplaySeeds来终止状态 (低效但简单)
  
  // 返回 state 对应的迭代器 
  std::map< ExecutionState*, std::vector<SeedInfo> >::iterator it = 
    seedMap.find(&state);
  
  if (it != seedMap.end()) {
    std::vector<SeedInfo> seeds = it->second;
    seedMap.erase(it);

    // 假定每个种子仅满足一个条件(当条件互相排斥但结合起来又重言时, 就必然是true)
    for (std::vector<SeedInfo>::iterator siit = seeds.begin(), 
           siie = seeds.end(); siit != siie; ++siit) {
      unsigned i;
      for (i=0; i<N; ++i) {
        ref<ConstantExpr> res;
        bool success = 
          solver->getValue(state, siit->assignment.evaluate(conditions[i]), 
                           res);
        assert(success && "FIXME: Unhandled solver failure");
        (void) success;
        if (res->isTrue())
          break;
      }
      
      // If we didn't find a satisfying condition randomly pick one
      // (the seed will be patched).
      if (i==N)
        i = theRNG.getInt32() % N;

      // Extra check in case we're replaying seeds with a max-fork
      if (result[i])
        seedMap[result[i]].push_back(*siit);
    }

    if (OnlyReplaySeeds) {
      for (unsigned i=0; i<N; ++i) {
        if (result[i] && !seedMap.count(result[i])) {
          terminateState(*result[i]);
          result[i] = NULL;
        }
      } 
    }
  }

```

这段代码中重点需要关注的是**solver->getValue(current, siit->assignment.evaluate(condition), res);**. 这一句的作用是用来判断我们的seed是否满足相应的condition, 并将结果保存在res里. 如果seed满足**condition**, 那么就直接跳出循环执行seed指定的路径. 最后的**OnlyReplaySeeds**也是用来提前终止那些不包含seed的状态. 

## fork

``` c
  std::map< ExecutionState*, std::vector<SeedInfo> >::iterator it = 
    seedMap.find(¤t);
  bool isSeeding = it != seedMap.end();

```

首先这里取了一个迭代器**it**以及布尔值**isSeeding**判断当前是否还在seed模式下. 那么我们重点关注迭代器**it**的相关操作

``` c
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
    if (!(trueSeed && falseSeed)) { // trueSeed和falseSeed至少有1个为假
      assert(trueSeed || falseSeed);	// trueSeed和falseSeed至少有1个为真
      
      res = trueSeed ? Solver::True : Solver::False;
      addConstraint(current, trueSeed ? condition : Expr::createIsZero(condition));
    }
  }

```

同样使用**solver->getValue()**来获取seed与condition的关系, 并将结果保存到**res**里. **res->isTrue**表明种子满足条件, 为真, 否则不满足条件为假. 当判定seed是trueSeed还是falseSeed后则添加相应的约束. 如果无法判定真假, 则跳出循环

## addConstraint

``` c
void Executor::addConstraint(ExecutionState &state, ref<Expr> condition) {
  if (ConstantExpr *CE = dyn_cast<ConstantExpr>(condition)) {
    if (!CE->isTrue())
      llvm::report_fatal_error("attempt to add invalid constraint");
    return;
  }

  // Check to see if this constraint violates seeds.
  std::map< ExecutionState*, std::vector<SeedInfo> >::iterator it = 
    seedMap.find(&state);
  if (it != seedMap.end()) {
    bool warn = false;
    for (std::vector<SeedInfo>::iterator siit = it->second.begin(), 
           siie = it->second.end(); siit != siie; ++siit) {
      bool res;
      bool success = 
        solver->mustBeFalse(state, siit->assignment.evaluate(condition), res);
      assert(success && "FIXME: Unhandled solver failure");
      (void) success;
      if (res) {
        siit->patchSeed(state, condition, solver);
        warn = true;
      }
    }
    if (warn)
      klee_warning("seeds patched for violating constraint"); 
  }

  state.addConstraint(condition);
  if (ivcEnabled)
    doImpliedValueConcretization(state, condition, 
                                 ConstantExpr::alloc(1, Expr::Bool));
}
```

在**addConstraint**里, 使用**seedMap**来取得state对应的seed. 并用**solver->mustBeFalse(state, siit->assignment.evaluate(condition), res);**判断seed与该condition是否满足永假关系, 也即约束是否跟seed相违背. 如果相违背则**patchSeed**并输出警告信息

## executeGetValue

**executeGetValue**仅在**SpecialFunctionHandler::handleGetValue**中用到

``` c
  add("klee_get_valuef", handleGetValue, true),
  add("klee_get_valued", handleGetValue, true),
  add("klee_get_valuel", handleGetValue, true),
  add("klee_get_valuell", handleGetValue, true),
  add("klee_get_value_i32", handleGetValue, true),
  add("klee_get_value_i64", handleGetValue, true),
......
void SpecialFunctionHandler::handleGetValue(ExecutionState &state,
                                            KInstruction *target,
                                            std::vector<ref<Expr> > &arguments) {
  assert(arguments.size()==1 &&
         "invalid number of arguments to klee_get_value");

  executor.executeGetValue(state, arguments[0], target);
}
```

arguments[0]则是**klee_get_value**的第一个参数. 也就是通过**executeGetValue**来获取对应的符号的值

``` c
void Executor::executeGetValue(ExecutionState &state,
                               ref<Expr> e,
                               KInstruction *target) {
  e = state.constraints.simplifyExpr(e);
  std::map< ExecutionState*, std::vector<SeedInfo> >::iterator it = 
    seedMap.find(&state);
  if (it==seedMap.end() || isa<ConstantExpr>(e)) {
    ref<ConstantExpr> value;
    bool success = solver->getValue(state, e, value);
    assert(success && "FIXME: Unhandled solver failure");
    (void) success;
    bindLocal(target, state, value);
  } else {
    std::set< ref<Expr> > values;
    for (std::vector<SeedInfo>::iterator siit = it->second.begin(), 
           siie = it->second.end(); siit != siie; ++siit) {
      ref<ConstantExpr> value;
      bool success = 
        solver->getValue(state, siit->assignment.evaluate(e), value);
      assert(success && "FIXME: Unhandled solver failure");
      (void) success;
      values.insert(value);
    }
    
    std::vector< ref<Expr> > conditions;
    for (std::set< ref<Expr> >::iterator vit = values.begin(), 
           vie = values.end(); vit != vie; ++vit)
      conditions.push_back(EqExpr::create(e, *vit));

    std::vector<ExecutionState*> branches;
    branch(state, conditions, branches);
    
    std::vector<ExecutionState*>::iterator bit = branches.begin();
    for (std::set< ref<Expr> >::iterator vit = values.begin(), 
           vie = values.end(); vit != vie; ++vit) {
      ExecutionState *es = *bit;
      if (es)
        bindLocal(target, *es, *vit);
      ++bit;
    }
  }
}
```

类似, 同**solver->getValue(state, siit->assignment.evaluate(e), value);**

## updateStates

在**updateStates**里进行的操作是将seedMap里的state给擦除掉. 

``` c
void Executor::updateStates(ExecutionState *current) {
  if (searcher) {
    searcher->update(current, addedStates, removedStates);
    searcher->update(nullptr, continuedStates, pausedStates);
    pausedStates.clear();
    continuedStates.clear();
  }
  
  states.insert(addedStates.begin(), addedStates.end());
  addedStates.clear();

  // 逐个擦除掉
  for (std::vector<ExecutionState *>::iterator it = removedStates.begin(),
                                               ie = removedStates.end();
       it != ie; ++it) {
    ExecutionState *es = *it;
    std::set<ExecutionState*>::iterator it2 = states.find(es);
    assert(it2!=states.end());
    states.erase(it2);
    // 在seedMap里擦除掉es
    std::map<ExecutionState*, std::vector<SeedInfo> >::iterator it3 = 
      seedMap.find(es);
    if (it3 != seedMap.end())
      seedMap.erase(it3);
    processTree->remove(es->ptreeNode);
    delete es;
  }
  removedStates.clear();
}
```

**searcher->update**会将需要擦除的**removedStates**列出来, 然后在循环里将**removeStates**从**states**中擦除, 如果在**seedMap**里有**removedStates**的话, 也一并擦除掉. 

## terminateState相关

**terminateState**, **terminateState**, **terminateStateOnExit**使用**seedMap**都是对state进行擦除做的清理工作. 因为这个state已经被终结了, 因此会对seedMap进行擦除, 以提高效率

## executeMakeSymbolic

**executeMakeSymbolic**只由**handleMakeSymbolic**一处调用, 而**handleMakeSymbolic**只在处理**klee_make_symbolic**时会起作用. 

``` c
add("klee_make_symbolic", handleMakeSymbolic, false),
```

因此可以确定的是**executeMakeSymbolic**也只有在处理**klee_make_symbolic**时会触发. 

``` c
void Executor::executeMakeSymbolic(ExecutionState &state, 
                                   const MemoryObject *mo,
                                   const std::string &name) {
  // Create a new object state for the memory object (instead of a copy).
  if (!replayKTest) {
    // Find a unique name for this array.  First try the original name,
    // or if that fails try adding a unique identifier.
    unsigned id = 0;
    std::string uniqueName = name;
    while (!state.arrayNames.insert(uniqueName).second) {
      uniqueName = name + "_" + llvm::utostr(++id);
    }
    const Array *array = arrayCache.CreateArray(uniqueName, mo->size);
    bindObjectInState(state, mo, false, array);
    state.addSymbolic(mo, array);
    
    std::map< ExecutionState*, std::vector<SeedInfo> >::iterator it = 
      seedMap.find(&state);
    if (it!=seedMap.end()) { // In seed mode we need to add this as a
                             // binding.
      for (std::vector<SeedInfo>::iterator siit = it->second.begin(), 
             siie = it->second.end(); siit != siie; ++siit) {
        SeedInfo &si = *siit;
        // 如果指定NamedSeedMatching 获取seed中跟mo->name相同的对象
        // 否则, 从上至下依次返回mo的对象
        KTestObject *obj = si.getNextInput(mo, NamedSeedMatching);

        if (!obj) { // 没有可取对象
          if (ZeroSeedExtension) {
            std::vector<unsigned char> &values = si.assignment.bindings[array];
            values = std::vector<unsigned char>(mo->size, '\0');
          } else if (!AllowSeedExtension) {
            // seed跟内存对象mo的对象数量不一致
            terminateStateOnError(state, "ran out of inputs during seeding",
                                  User);
            break;
          }
        } else {
          if (obj->numBytes != mo->size &&  // 对象大小不匹配
              ((!(AllowSeedExtension || ZeroSeedExtension)
                && obj->numBytes < mo->size) || // obj大小 < mo大小 又不准扩展种子
               (!AllowSeedTruncation && obj->numBytes > mo->size))) { // obj大小 > mo大小, 又不准截断种子
               // 输出错误信息并退出
	    std::stringstream msg;
	    msg << "replace size mismatch: "
		<< mo->name << "[" << mo->size << "]"
		<< " vs " << obj->name << "[" << obj->numBytes << "]"
		<< " in test\n";

            terminateStateOnError(state, msg.str(), User);
            break;
          } else {
            // 开始处理 seed, 进行绑定
            std::vector<unsigned char> &values = si.assignment.bindings[array];
            values.insert(values.begin(), obj->bytes, 
                          obj->bytes + std::min(obj->numBytes, mo->size));
            if (ZeroSeedExtension) {
              for (unsigned i=obj->numBytes; i<mo->size; ++i)
                values.push_back('\0');
            }
          }
        }
      }
    }
  } else { ... // replay ktest 分支
  }
}
```

在**executeMakeSymbolic**中, 首先会根据**seedMap**提取出所有的**obj**, 并与**mo**进行比较, 如果跟**mo**的大小不匹配, 就会出现一些错误信息. 在真正处理seed时则会将其绑定给**si.assignment.bindings[array]**. 如果没有seed模式. 也就只能执行前半部分

``` c
	unsigned id = 0;
    std::string uniqueName = name;
    while (!state.arrayNames.insert(uniqueName).second) {
      uniqueName = name + "_" + llvm::utostr(++id);
    }
    const Array *array = arrayCache.CreateArray(uniqueName, mo->size);
    bindObjectInState(state, mo, false, array);
    state.addSymbolic(mo, array);
```

## Seed模式相关选项

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

### -seed-out

**-seed-out=**指定的seed文件会保存到**SeedOutFile**中, 在main函数中进行使用. 

``` c
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

### -only-seed

**-only-seed**与布尔值**OnlySeed**绑定, 默认关闭. 

描述是**在seed模式结束后, 不做常规搜索直接停止执行**

``` c
  cl::opt<bool>
  OnlySeed("only-seed",
	   cl::init(false),
           cl::desc("Stop execution after seeding is done without doing regular search (default=off)."));
```

仅在**Executor:run()**里使用到

``` c
    if (OnlySeed) {
      doDumpStates();
      return;
    }
```

再来看**doDumpStates()**

``` c
void Executor::doDumpStates() {
  if (!DumpStatesOnHalt || states.empty())
    return;

  klee_message("halting execution, dumping remaining states");
  for (const auto &state : states)
    terminateStateEarly(*state, "Execution halting.");
  updateStates(nullptr);
}
```

**DumpStatesOnHalt**是默认开启的选项, 在退出时将所有活动的状态都dump成测试用例文件.

之后的操作是相当于提前终止, 因为**onlySeed**紧接着**seeding done**执行, 因此可以确保在seed模式执行结束后终止其他state

### -only-replay-seeds

**-only-replay-seed**与布尔值**OnlyReplaySeeds**绑定, 默认关闭. 

描述是**执行时忽略不含有seed的状态**

``` c
  cl::opt<bool>
  OnlyReplaySeeds("only-replay-seeds",
		  cl::init(false),
                  cl::desc("Discard states that do not have a seed (default=off)."));
```

仅在**Executor::branch**和**Executor::fork**有用到

``` c
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

```c
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