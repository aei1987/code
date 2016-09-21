---
layout: post
title:  "Motan源码解读-源码结构"
date:  2016-09-21
categories: [Motan源码解读]
---

Motan源码解读-源码结构


### motan包结构

	motan-benchmark // 压测相关
	motan-core	//motan核心代码，也是源码解析的主要包
	motan-demo	// 示例，快速上手的最佳方法
	motan-extension // 扩展
	motan-manager // 管理后台
	motan-registry-consul	//基于consul的服务发现实现
	motan-registry-zookeeper // 基于zookeeper的服务发现实现
	motan-springsupport // spring容器的支持，也支持springboot
	motan-transport-netty // netty传输的封装


#### motan-core


```
└─com
    └─weibo
        └─api
            └─motan
                ├─cluster
                │  ├─ha //[motan-容灾策略](http://zhizus.com/code/motan-hastratege)
                │  ├─loadbalanc // [motan-负载均衡](http://zhizus.com/code/motan-hastratege)
                │  └─support
                ├─codec // 编解码
                ├─common
                ├─config // 配置
                │  ├─annotation
                │  └─handler
                ├─core
                │  └─extension //SPI机制扩展
                ├─exception // 异常
                ├─filter // 过滤器
                ├─log // 日志
                ├─protocol // 协议部分
                │  ├─injvm
                │  ├─rpc
                │  └─support
                ├─proxy //代理
                │  └─spi
                ├─registry //服务注册
                │  └─support
                │      └─comman
                ├─rpc
                ├─serialize //序列化
                ├─switcher
                ├─transport
                │  └─support
                └─util
```

这个包是motan的核心实现，里面的大部分代码，接下的文章都会涉及到。
