---
title: tomcat源码阅读（二）
date: 2025-03-10 14:59:35
categories: 源码
---





# tomcat源码阅读

## 日志模块梳理



现在日志通常分为日志门面和日志的具体实现。



日志门面：充当应用程序和日志框架之间的沟通媒介，可以在程序无感的条件下更换日志框架。

日志的具体实现：直接记录日志(console、file)，并且需要注意的是，这些信息都是由日志门面交给日志实现的。



对于非日志编程的程序员来说，我们只需要明白如何根据日志门面切换日志实现即可，没必要阅读日志实现的代码。





常见的日志门面：JCL、slf4j

常见的日志实现：JUL（java.util.logging）、log4j、logback、log4j2





### tomcat的日志门面JULI

该日志门面位于源码包`org.apache.juli`,基于JCL实现的。

较为重要的三个类：

org.apache.juli.logging.Log： 日志接口

org.apache.juli.logging.DirectJDKLog：tomcat的默认日志实现。

org.apache.juli.logging.LogFactory: 与tomcat进行交互获取log接口实现。



在源码中可以看到，无论是什么类要记录日志都必须使用到Log的实现类，而对于tomcat源码来说，唯一获取该实例的途径就是LogFactory#getLog(Class<?> clazz)。



并且在LogFactory#release(ClassLoader classLoader)也可以看到日志实现默认为JUL,硬编码控制的。



```
    public static void release(ClassLoader classLoader) {
        // JULI's log manager looks at the current classLoader so there is no
        // need to use the passed in classLoader, the default implementation
        // does not so calling reset in that case will break things
        if (!LogManager.getLogManager().getClass().getName().equals(
                "java.util.logging.LogManager")) {
            LogManager.getLogManager().reset();
        }
    }
```



## 关于日志实现log的加载

在LogFactory的无参构造器器中看到关键语句

```

        ServiceLoader<Log> logLoader = ServiceLoader.load(Log.class);
        Constructor<? extends Log> m=null;
        for (Log log: logLoader) {
            Class<? extends Log> c=log.getClass();
            try {
                m=c.getConstructor(String.class);
                break;
            }
            catch (NoSuchMethodException | SecurityException e) {
                throw new Error(e);
            }
        }
        discoveredLogConstructor=m;
        
```



其中ServiceLoader.load(Log.class)是用javaspi加载META-INF/services/类权全限定名称实例的方法，所以如果查看常见日志框架实现的源码的话，应该是有对应文件的。





但是需要注意的是tomcat的默认实现DirectJDKLog并不在这里加载，即便我们自主添加了资源类也会报错，这是因为DirectJDKLog只有一个String参数的构造器，而ServiceLoader在遍历服务时会默认使用无参构造器。





那么DirectJDKLog是如何使用的呢？追溯iscoveredLogConstructor调用找到

`LogFactory#getInstance(String name);`

```
    public Log getInstance(String name) throws LogConfigurationException {
        if (discoveredLogConstructor == null) {
            return DirectJDKLog.getInstance(name); //默认实现、
        }

        try {
            return discoveredLogConstructor.newInstance(name);
        } catch (ReflectiveOperationException | IllegalArgumentException e) {
            throw new LogConfigurationException(e);
        }
    }
```



## 日志格式化

格式化大致就是一些日志的信息读取格式和输出格式，这一部分一定会在经过程序编码，所以直接调试即可。



例如： 

StringManager:大部分都会通过这个类格式化字符串，其余可以自行查看日志门面的实现。





## 关于文件转码

java读取配置文件默认iso8859-1编码格式，如果我们已经读取了这个格式代码，那么该如何转变为UTF-8呢？



理论上来说如果有iso8859-1编码的字节，直接进行转码即可，但是我们可能会对一个字符串进行多次转码，可能导致字符串的编码不再是iso8859-1。

例如tomcat会在中途对字符串进行转码，所以到达StringManager时字符串编码已经变为了“UTF-8”，所以直接进行转码会失败

`str= new String(str.getBytes(), StandardCharsets.*UTF_8*);` 失败，追溯str.getBytes()发现底层编码已经改变。这时我们可以强行指定编码。

`str= new String(str.getBytes(StandardCharsets.ISO_8859_1), StandardCharsets.UTF_8);`

















