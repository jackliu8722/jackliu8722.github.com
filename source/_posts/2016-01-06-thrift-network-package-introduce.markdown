---
layout: post
title: "Thrift网络请求包格式详解"
date: 2016-01-06 14:54
comments: true
categories: Java{java} 分布式{distributed} 架构{arch}
---
写这篇文章的主要原因是近期项目通过Thrift进行通信时，thrift服务端解析客户端发送的数据错误，导致某些请求失败（后面解析其原因）。在查询问题原因的过程中把Thrift发送给服务端的数据包代码阅读了一下，在这里做一下笔记。

##Thrift支持的数据类型
Thrift客户端与服务端通信时，数据包按照一定的协议进行组织，协议最基本单元就是数据类型，以下是Thrift支持的数据类型：

```java
public final class TType {
  public static final byte STOP   = 0;
  public static final byte VOID   = 1;
  public static final byte BOOL   = 2;
  public static final byte BYTE   = 3;
  public static final byte DOUBLE = 4;
  public static final byte I16    = 6;
  public static final byte I32    = 8;
  public static final byte I64    = 10;
  public static final byte STRING = 11;
  public static final byte STRUCT = 12;
  public static final byte MAP    = 13;
  public static final byte SET    = 14;
  public static final byte LIST   = 15;
  public static final byte ENUM   = 16;
}
```
上面的TType类来自org.apache.thrift.protocol包，主要定义thrift目前支持的数据类型，在传输时根据应用层定义的消息按类型组织数据包

##消息发送过程
假设我们定义了一个thrift idl文件，内容如下：

```java
namespace java com.test.api

service Api{
    map<string,string> test(1:map<string,string> params);
}
```

通过thrift生成相应的java文件，当用户调用test方法时，具体的调用流程如下：
Api.Client.test -> Api.Client.send_test ->  TServiceClient.sendBase -> TBase.write
下面具体介绍各方法是怎么把消息组织起来的。

####Api.Client.test

```java
public Map<String,String> test(Map<String,String> params) throws org.apache.thrift.TException
{
 	send_test(params);
 	return recv_test();
}
```
该接口是应用层客户端调用的，其实它的工作很简单，传递参数并调用send_test进行发送消息，发送完成后接收服务端的响应，这个操作是一个同步操作，这里暂时不做介绍。

####Api.Client.send_test

```java
public void send_test(Map<String,String> params) throws org.apache.thrift.TException
{
 	test_args args = new test_args();
 	args.setParams(params);
 	sendBase("test", args);
}
```
send_test接口的工作其实也很简单，new一个参数类（参数类是thrif根据idl文件生成的），并设置应用层提供的参数，然后调用sendBase方法。

####TServiceClient.sendBase

```java
protected void sendBase(String methodName, TBase args) throws TException {
   this.oprot_.writeMessageBegin(new TMessage(methodName, (byte)1, ++this.seqid_));
   args.write(this.oprot_);
   this.oprot_.writeMessageEnd();
   this.oprot_.getTransport().flush();
}
```
该方法才是真正的组织消息包了，首先调用oprot_的writeMessageBegin将新建的TMessage对象写入到协议中，TMessage是什么？其实就是对方法名、类型和顺序号进行了封装，没做其它什么工作，具体的类如下：

```java
package org.apache.thrift.protocol;

public final class TMessage {
    public final String name;
    public final byte type;
    public final int seqid;

    public TMessage() {
        this("", (byte)0, 0);
    }

    public TMessage(String n, byte t, int s) {
        this.name = n;
        this.type = t;
        this.seqid = s;
    }

    public String toString() {
        return "<TMessage name:\'" + this.name + "\' type: " + this.type + " seqid:" + this.seqid + ">";
    }

    public boolean equals(Object other) {
        return other instanceof TMessage?this.equals((TMessage)other):false;
    }

    public boolean equals(TMessage other) {
        return this.name.equals(other.name) && this.type == other.type && this.seqid == other.seqid;
    }
}
```
把方法名等写入到协议后，就需要对方法的参数进行写入，参数的写入由TBase的子类实现（后面会进行分析），参数写入完成后，就完成了对协议的封装，就可以调用传输层进行消息的发送了。

####TBase.write
TBase.write是一个接口，其具体的实现是由test_args类完成的，其源码如下：

```java
public void write(org.apache.thrift.protocol.TProtocol oprot) throws org.apache.thrift.TException {
  schemes.get(oprot.getScheme()).getScheme().write(oprot, this);
}
```
咦，怎么还有一层封装，那我们看看schemes是什么东东。

```java
private static final Map<Class<? extends IScheme>, SchemeFactory> schemes = new HashMap<Class<? extends IScheme>, SchemeFactory>();
static {
 	schemes.put(StandardScheme.class, new test_argsStandardSchemeFactory());
 	schemes.put(TupleScheme.class, new test_argsTupleSchemeFactory());
}
```
原来是通过TProtocol的getScheme方法得到具体的scheme类作为Map的key获取SchemeFactory,然后调用相应SchemeFactory的write方法进行对参数的写入。那TProtocol的scheme是什么呢？我们看一下TProtocol的getScheme方法。

```java
public Class<? extends IScheme> getScheme() {
	return StandardScheme.class;
}
```
原来抽象类TProtocol将StandardScheme.class作为默认的scheme类，那就好办了，其实最终调用的是test_argsStandardSchemeFactory类的write方法对参数进行写入。我们再看看test_argsStandardSchemeFactory的write方法。

```java
private static final org.apache.thrift.protocol.TStruct STRUCT_DESC = new org.apache.thrift.protocol.TStruct("test_args");

private static final org.apache.thrift.protocol.TField PARAMS_FIELD_DESC = new org.apache.thrift.protocol.TField("params", org.apache.thrift.protocol.TType.MAP, (short)1);

public void write(org.apache.thrift.protocol.TProtocol oprot, test_args struct) throws org.apache.thrift.TException {
   struct.validate();

   oprot.writeStructBegin(STRUCT_DESC);//按照协议要求写入struct描述
   if (struct.params != null) {
     oprot.writeFieldBegin(PARAMS_FIELD_DESC); //按照协议要求写入fild描述
     {
     		//写入map对应key和value的类型，以及元素个数
       	oprot.writeMapBegin(new org.apache.thrift.protocol.TMap(org.apache.thrift.protocol.TType.STRING, org.apache.thrift.protocol.TType.STRING, struct.params.size()));
       	//遍历map，将key和value写入到协议中
        for (Map.Entry<String, String> _iter24 : struct.params.entrySet())
        {
         	oprot.writeString(_iter24.getKey());
         	oprot.writeString(_iter24.getValue());
        }
       	oprot.writeMapEnd();
     }
     oprot.writeFieldEnd();
   }
   oprot.writeFieldStop();
   oprot.writeStructEnd();
 }
```
其实代码逻辑已经非常清楚了，唯一可能还需要了解的oprot的实现，Thrift默认使用二进制协议TBinaryProtocol类写所有的数据。

##二进制协议类TBinaryProtocol
二进制协议类TBinaryProtocol作为Thrift默认协议进行读写数据，下面介绍一些主要方法，便于理解协议包的数据组织。

```java
public void writeMessageBegin(TMessage message) throws TException {
	if (strictWrite_) {
		 int version = VERSION_1 | message.type;
		 writeI32(version);
		 writeString(message.name);
		 writeI32(message.seqid);
	} else {
		 writeString(message.name);
		 writeByte(message.type);
		 writeI32(message.seqid);
	}
}

public void writeMessageEnd() {}
```
该方法将版本号、方法名和顺序号写入到协议中。

```java
public void writeFieldBegin(TField field) throws TException {
    writeByte(field.type);
    writeI16(field.id);
 }

public void writeFieldEnd() {}
```
该方法将参数类型和id写入到协议中。

```java
public void writeMapBegin(TMap map) throws TException {
	writeByte(map.keyType);
	writeByte(map.valueType);
	writeI32(map.size);
}

public void writeMapEnd() {}
```
当需要写入map时，将map对应key和value的类型，以及元素个数写入到协议中。

```java
public void writeString(String str) throws TException {
	try {
		byte[] dat = str.getBytes("UTF-8");
		writeI32(dat.length);
		trans_.write(dat, 0, dat.length);
	} catch (UnsupportedEncodingException uex) {
		throw new TException("JVM DOES NOT SUPPORT UTF-8");
	}
}
```
将string写入到协议中，首先写入string的长度，然后再写入string的字节数组。

其它方法就不在这里一一列出了，大家可以通过看源码进行了解。

##参数Map<String,String>的消息格式
下面我们以上面idl文件中test方法的参数作为例子，给出具体的协议格式，如下图：

{% img  center /images/2016-06/thrift-protocol-format.png %}

##开篇中的问题
在本文开始的时候有提到，我们业务同学在使用thrift进行远程通信时，部分请求在服务端抛出了消息格式解析失败的异常。到底是什么原因呢？容我慢慢道来。我们接口传递的参数类型为Map<String,String>，一般情况是用HashMap来实例化Map，我们知道HashMap的value是可以为NULL的，我们再看看writeString方法；

```java
public void writeString(String str) throws TException {
	try {
		byte[] dat = str.getBytes("UTF-8");
		writeI32(dat.length);
		trans_.write(dat, 0, dat.length);
	} catch (UnsupportedEncodingException uex) {
		throw new TException("JVM DOES NOT SUPPORT UTF-8");
	}
}
```
看到没有，writeString方法没有做NULL判断，那么当我们传递的参数为空时，会向上抛出异常NullPointerException，而在writeString没有进行处理，则会抛向test_argsStandardSchemeFactory的write方法中，而该类也未进行处理，则继续向上抛。通过跟踪源码发现，最终都未对该异常进行处理，那么最后当前客户端只是把部分数据写入到了协议的缓存中，且未调用TProtocol的flush方法进行发送。

当有新的调用时，会继续往该协议中写数据，如果这时Map中不存在为NULL的值，则会写入成功，然后调用flush方法将数据通过网络发送给服务端。这里协议数据包括两部分，第一部分是前一次写失败但有部分写入的数据，第二部分是一个完整的数据。当服务端接收到数据后通过协议进行解析时，是从第一部分开始的，由于数据不完整会造成解析失败，后面部分的数据也不会再进行处理了，这样的话就造成两个请求失败了。

以上就是问题的原因，最终的解决方法很简单，通知业务同学修改代码，对插入map的value进行空处理。更新代码后，此问题不再出现。





     



