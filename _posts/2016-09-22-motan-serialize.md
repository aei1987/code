---
layout: post
title:  "Motan源码解读-serialize"
date:  2016-09-22
categories: [Motan源码解读]
---
Motan源码解读-serialize

>**序列化：** 将数据结构或对象转换成二进制串的过程。
> **反序列化：**将在序列化过程中所生成的二进制串转换成数据结构或者对象的过程。

将RPC`请求中的参数、结果等对象进行序列化与反序列化`，即进行对象与字节流的互相转换；默认使用对java更友好的hessian2进行序列化。（注意序列化的是请求的参数，和返回的结果，其他的地方不需要序列化）


![](/code/images//motan/motan-register-server-client.jpg)

motan目前支持两种序列化：

- hession2

```
<motan:protocol serialization="hessian2" />		
```

- fastjson

```
<motan:protocol serialization="fastjson" />		
```


### 各种序列化性能对比

![](/code/images//motan/serialize.png)

### motan序列化的实现
``` java
// 这个可以可以扩展的，也就是你可以扩展motan用你想要的序列化方式
@Spi(scope=Scope.PROTOTYPE)
public interface Serialization {
	// 将对象序列话称字节数组
	byte[] serialize(Object obj) throws IOException;
	// 将字节数组反序列化成对象
	<T> T deserialize(byte[] bytes, Class<T> clz) throws IOException;
}
```
具体的实现要看hession2和fastjson了。


----
后记：

1.fastjson确实够快，使用也挺方便的。

2.如果我们需要扩展motan实现thrift或者protobuf还需要hession或者fastjson来辅助序列化吗？

首先hession可以排除，使用hession跨语言调用就比较难实现了。
那么需要辅助的fastjson来序列化吗？没必要。
