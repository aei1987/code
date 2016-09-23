---
layout: post
title:  "Motan源码解读-codec"
date:  2016-09-23
categories: [Motan源码解读]
---

Motan源码解读-codec


motan的codec负责rpc框架通讯协议的编解码。codec模块是每个rpc必备的模块，一个设计良好的codec关系到rpc框架编解码的性能和对不同协议的扩展。

dubbo支持http,thrift,dubbo,rmi,webservice,redis等等协议，而motan目前仅支持默认motan的协议，所以很多时候我们可能会扩展motan支持自己的协议，那么我们就得从codec入手了（有的时候甚至还会涉及到protocol）。

codec的类图如下：

![Alt text](/code/images//motan/codec.png)


下面 codec interface的定义：

``` java
// 可基于spi扩展
@Spi(scope=Scope.PROTOTYPE)
public interface Codec {
	// channel为motan定义的channel，非netty的channel
	byte[] encode(Channel channel, Object message) throws IOException;

	/**
	 * 
	 * @param channel
	 * @param remoteIp 用来在server端decode request时能获取到client的ip。
	 * @param buffer
	 * @return
	 * @throws IOException
	 */
	// 不明白为什么要把remoteIp定义到decode接口里面，channel里面已经包含ip的信息了
	Object decode(Channel channel, String remoteIp, byte[] buffer) throws IOException;

}
```
AbstractCodec里面实现了创建输入输出流的方法，用来将协议序列化成字节数组或将字节数组反序列化成协议的。（这种方式的性能还没压测，不知道好坏）

如果我们要实现自己的协议，可以仅仅实现codec的方法。

实现AbstractCodec的有三个类

- DefaultRpcCodec

> 默认的协议编解码实现

- MockDefaultRpcCodec

> 用于测试的

- CompresssRpcCodec

>经过压缩的协议编解码实现

为了不引入复杂性，我们先看DefaultRpcCodec的实现。
要知道motan如何对协议编解码，首先我们的非常清楚motan的协议格式。

CompresssRpcCodec这个类的注释中已经有协议各个模块的实现，但是还不够清晰简明，我们先整理一个图。

#### motan协议格式

- header


<table style="border-collapse:collapse;border-spacing:0;border-color:#999"><tr><th style="font-family:Arial, sans-serif;font-size:14px;font-weight:normal;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#fff;background-color:#26ADE4;text-align:center" colspan="7">header</th></tr><tr><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center">0-15</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center">16-23</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top" colspan="3">24-31</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top">32-95</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top">96-127</td></tr><tr><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center">magic</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center">version</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top" colspan="3">extend flag</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top">request id</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top">body content length</td></tr><tr><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center" rowspan="2">魔数</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center" rowspan="2">协议版本</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top">24-28</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top">29-30</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top">31</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top" rowspan="2"><br>消息id<br></td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top" rowspan="2"><br>body包长</td></tr><tr><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top">保留</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top">event( 可支持4种event，<br>如normal, exception等)</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top">0 is request , 1 is response</td></tr></table>


- request body

```  java
    /* 
     * <pre>
	 * 
	 * 	 body:
	 * 
	 * 	 byte[] data :  
	 * 
	 * 			serialize(interface_name, method_name, method_param_desc, method_param_value, attachments_size, attachments_value) 
	 * 
	 *   method_param_desc:  for_each (string.append(method_param_interface_name))
	 * 
	 *   method_param_value: for_each (method_param_name, method_param_value)
	 * 
	 * 	 attachments_value:  for_each (attachment_name, attachment_value)
	 * 
	 * </pre>
     * 
     * @param request
     * @return
     * @throws IOException
     */
    private byte[] encodeRequest(Channel channel, Request request) throws IOException ;
```

encodeRequest方法的注释很清晰的描述了request 的body部分的序列化，对jvm系列的来说非常友好，基本上把调用相关所有能用到的信息都序列化了。

我们可以把请求任何一个请求参数都反序列化java对象。所以motan默认的协议是可以支持方法内参数为java任意对象。
但是，`对于其他语言就不太友好了`。其他语言不太容易序列化java的对象。
如果希望更好的支持其他的语言，这个请求参数的类型序列化方式可能有所取舍了。

- response body

> `serialize (result) or serialize (exception)`：motan的response的message对象是通过hession2或者fastjson序列化生成的字节数组，所以这个地方就是一个字节数组（如果需要了解序列化的部分，可以看上篇博客）


#### motan协议打包

协议的打包简单说一下header打包的。

``` java
  private byte[] encode(byte[] body, byte flag, long requestId) throws IOException {
        // 协议1的header占16个字节，先创建一个长度为16的字节数组
        byte[] header = new byte[RpcProtocolVersion.VERSION_1.getHeaderLength()];
        int offset = 0;

        // 0 - 15 bit : magic
        // 这个方法实际上是把short类型的magic转成两个字节，打包到header
        ByteUtil.short2bytes(MAGIC, header, offset);
        offset += 2;

        // 16 - 23 bit : version
        // 打包version
        header[offset++] = RpcProtocolVersion.VERSION_1.getVersion();

        // 24 - 31 bit : extend flag
        // 打包flag
        header[offset++] = flag;

        // 32 - 95 bit : requestId
        // 打包requestId
        ByteUtil.long2bytes(requestId, header, offset);
        offset += 8;

        // 96 - 127 bit : body content length
        // 打包body包长
        ByteUtil.int2bytes(body.length, header, offset);

        byte[] data = new byte[header.length + body.length];

        System.arraycopy(header, 0, data, 0, header.length);
        System.arraycopy(body, 0, data, header.length, body.length);

        return data;
    }
```


到这里motan默认的协议基本分析完，但是还没说到CompresssRpcCodec 压缩的编解码，（这个压缩的类跟协议编解码写到一起，分离出来会不会更好[如果我们能确定哪些地方需要压缩]）
下一篇会单独就讲压缩部分。

---
后记：

个人并不是很喜欢motan codec的代码，故不会在这里耗太多时间。
以后遇到好的codec的代码，再来分享。