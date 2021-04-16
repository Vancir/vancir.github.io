---
title: 回到2015年看看那时ClusterFuzz的设计
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

