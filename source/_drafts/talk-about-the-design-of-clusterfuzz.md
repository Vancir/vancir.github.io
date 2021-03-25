---
title: 谈谈谷歌持续测试平台ClusterFuzz的模块设计
---


## 缓存

clusterfuzz一共定义了三种不同类型的缓存: `FifoInMemory`, `FifoOnDisk`和`Memcache`, 这三种类型均设计为线程所使用, 因此`threading.local()`来为每一个线程创建单独的存储来避免资源竞争问题. 三种缓存都基于不同的方法进行实现: 
* `FifoInMemory`基于`collections.OrderedDict()`实现, 以`dict`的键值对作为缓存
* `FifoOnDisk`基于`persistent_cache`实现, 实际上就是将键值以文件的形式存储到磁盘上. 文件使用相应的key来命名, 文件后缀为`.persist`
* `Memcache`基于redis实现, redis是著名的高速内存数据库, 支持多种数据类型. 