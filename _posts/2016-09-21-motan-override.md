---
layout: post
title:  "Motan源码解读-源码结构"
date:  2016-09-21
categories: [Motan源码解读]
---

Motan源码解读-源码结构

旨在对motan包结构熟悉，顺带会根据包结构列出系列文章的索引。

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
                │  ├─ha //①容灾策略
                │  ├─loadbalanc //②负载均衡
                │  └─support
                ├─codec //③编解码
                ├─common
                ├─config //④配置
                │  ├─annotation
                │  └─handler
                ├─core
                │  └─extension //⑤ SPI机制扩展
                ├─exception // ⑥异常
                ├─filter //⑦过滤器
                ├─log //⑧日志
                ├─protocol //⑨责点到点的通讯(有别于我们理解的protocol)
                │  ├─injvm
                │  ├─rpc
                │  └─support
                ├─proxy //⑩代理
                │  └─spi
                ├─registry //⑪服务注册
                │  └─support
                │      └─comman
                ├─rpc
                ├─serialize //⑫序列化
                ├─switcher
                ├─transport
                │  └─support
                └─util
```
这个包是motan的核心实现，里面的大部分代码，接下的文章都会涉及到。

①[motan-容灾策略](http://zhizus.com/code/motan-hastratege)

②[motan-负载均衡](http://zhizus.com/code/motan-loadbalance)

⑤[motan-SPI机制扩展](http://zhizus.com/code/motan-spi)

⑦[motan-过滤器](http://zhizus.com/code/motan-filter)

⑨[motan-protocol](http://zhizus.com/code/motan-protocol)

#### motan-transport-netty

⑨[motan-netty-client异步消息](http://zhizus.com/code/motan-clientMessage)
