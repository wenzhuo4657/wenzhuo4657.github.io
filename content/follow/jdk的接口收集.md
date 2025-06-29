+++
date = '2025-06-29T11:10:33+08:00'
draft = false
title = 'Jdk的接口收集'
layouts = 'single'
+++







# jdk17





需要注意的，无论是何种依赖，他们需要jdk原因都是需要jdk定义的通用接口，换句话说，在执行时最底层一定会使用到他们，这对于问题调试非常有用，我们可以直接定位到这些接口，查看究竟是什么地方有问题。

## 外部接口收集





### sql

只要支持sql，应当都可以使用这些接口



```
java.sql.Connection     与特定数据库的连接 （会话
java.sql.Statement    用于执行静态 SQL 语句并返回其生成的结果的对象
javax.sql.DataSource  用于连接到此 DataSource 对象所表示的物理数据源的工厂
该 DataSource 接口由驱动程序供应商实现。有三种类型的实现：
基本实现 -- 生成标准 Connection 对象
连接池实现 -- 生成 Connection 将自动参与连接池的对象。此实现与中间层连接池管理器一起使用。
分布式事务实现 -- 生成 Connection 可用于分布式事务的对象，并且几乎总是参与连接池。此实现与中间层事务管理器一起使用，并且几乎总是与连接池管理器一起使用。

```







### ldap

```
com.sun.jndi.ldap.LdapCtx      LDAP上下文实现
```







### log

这个接口是jdk提供的日志实现，但是多数情况下，我们使用logback等外部实现。

```
java.util.logging.Logger     jul的日志记录根
java.util.logging.LogManager     有一个全局 LogManager 对象，用于维护一组关于 Logger 和日志服务的共享状态
```










