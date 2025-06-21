---
title: logback官方文档阅读（一）
date: 2025-05-14 08:18:43
categories: 源码阅读
---





### logback

#### 架构

[Logback 架构](https://logback.qos.ch)
logback的核心依赖有：logback-core，logback-classic 和 logback-access

core： 核心实现，是其他两个包的基础
classic：
logback-classic 模块可以被同化为 log4j 1.x 的显着改进版本。此外，logback-classic 本机实现了 [SLF4J API](http://www.slf4j.org/)

access： 与 Servlet 容器（例如 Tomcat 和 Jetty）集成，以提供 HTTP 访问日志功能

#### 配置
Logback.xml配置的两个重要标签：`Logger`，`Appender`

logger： 指定日志记录根（全限定类名）和appender的关联
appender： 日志的具体输出实现，例如： 控制台输出、文件输出等




```
示例配置

<configuration scan="true" scanPeriod="60 seconds" debug="false">
<property name="console" value="==\n[%-5level] %red(%d{HH:mm}) Thread:[%thread]  Method:%green(%M) %cyan(%X{traceId})   classpath:%c \n%highlight(return):%m%n"></property>
<property name="log_dir" value="./data/log"></property>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
                 <pattern>${console}</pattern>
                <charset>utf8</charset>
            </encoder>
    </appender>


    <!--html格式日志文件输出appender-->
    <appender name="SERVICE_APPENDER" class="ch.qos.logback.core.FileAppender">
        <!--日志文件保存路径-->
        <file>${log_dir}/logback.html</file>
        <!--html 消息格式配置-->
        <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
            <layout class="ch.qos.logback.classic.html.HTMLLayout">
            </layout>
        </encoder>
    </appender>


        <appender name="All_APPENDER" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>${log_dir}/All/demo.log</file>
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
            <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
                <fileNamePattern>${log_dir}/All/demo.%d{yyyy-MM-dd}.log</fileNamePattern>
            </rollingPolicy>

    </appender>



<!--    </appender>-->

    <root level="DEBUG">
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="All_APPENDER"></appender-ref>
        <appender-ref ref="SERVICE_APPENDER"/>
    </root>



        <logger name="org/example" level="ERROR" additivity="true">
            <appender-ref ref="SERVICE_APPENDER"/>
        </logger>

</configuration>
```



### 官网文档阅读

[文档首页](https://logback.qos.ch/manual/introduction.html)

[依赖导入说明](https://logback.qos.ch/setup.html)


ps： 以下阅读并非纯粹的搬运、翻译，而是强调某些值得注意的地方。
#### Chapter 1: Introduction


##### 1，logback内部状态打印

```
import ch.qos.logback.classic.LoggerContext;  
import ch.qos.logback.core.util.StatusPrinter2;


  
LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();  
new StatusPrinter2().print(lc);
```



注意，上面使用的输出类为`StatusPrinter2()`,将打印logback上下文运状态，例如对配置文件的使用等，用于诊断logback的相关问题，且注意，


####   Chapter 2: Architecture


##### 1，依赖架构

> [!NOTE] 官网描述
> _核心_模块为其他两个模块奠定了基础。 _经典_模块扩展_了 core_。classic 模块对应于 log4j 的显著改进版本。Logback-classic 原生实现了 [SLF4J API](http://www.slf4j.org/)，以便您可以轻松地在 logback 和其他日志记录系统之间来回切换，例如 JDK 1.4 中引入的 log4j 或 java.util.logging （JUL）。第三个模块称为 _access_，它与 Servlet 容器集成以提供 HTTP 访问日志功能。一个单独的文档涵盖了 [Access Module 文档](https://logback.qos.ch/access.html) 。



（1）Logback-classic 原生实现了 [SLF4J API](http://www.slf4j.org/)
这句话在api层面的表现为
```
import ch.qos.logback.classic.Logger;
import org.slf4j.LoggerFactory;
Logger logger =(Logger) LoggerFactory.getLogger("chapters.introduction.HelloWorld1");  
logger.debug("Hello world.");
```

在`ch.qos.logback.classic.Logger`实现看到实现了`org.slf4j.Logger`这个接口，这就是代表他在扩展功能的同时，仍然支持原有的日志门面的功能，也就是**在 logback 和其他日志记录系统之间来回切换**。



##### 2，appender和layout

布局负责根据用户的意愿格式化日志记录请求，而 appender 负责将格式化的输出发送到其目的地。


默认配置：
```

```


##### 3，关于参数构造成本的优化

*目前好像无论是slf4j,还是logback-classic的logger都支持参数传递*
```

logger.debug("The new entry is "+entry+"."); 
logger.debug("The new entry is {}.", entry);
```


第一种写法会先构造message造成额外浪费，而第二种会在真正需要打印debug日志时，才会拼接。


##### 4，info调用的关键节点

- 1， 获取过滤器链决策
- 2，应用基本选择规则
- 3, 创建 `LoggingEvent` 对象： 该对象包含要打印的各种数据，
- 4，从logger context中获取 appender，并执行（并非执行完成，而是进入下一步）
- 5， 格式化输出：部分需要将``LoggingEvent``委托给布局进行输出，部分不需要。
	- 该步骤貌似是格式化一个完整的日志事件，包含输出的目的地等
- 6，发送 `LoggingEvent`


ps： 如何找到或者定义每个阶段？




#### Chapter 3: Logback configuration


##### 1,Configurator
全限定类名为：`import ch.qos.logback.classic.spi.Configurator;`

默认实现有三个： 
- BasicConfigurator： 最基本的默认实现，将输出指向控制台
- DefaultJoranConfigurator： 依次寻找`logback-test.xml`、`logback.xml`。
- SerializedModelConfigurator： 将于 2025 年 7 月 1 日从初始化序列中删除。



自定义实现：
- 实现`import ch.qos.logback.classic.spi.Configurator;`
- 文件资源位于 `META-INF/services/ch.qos.logback.classic.spi.Configurator` ，内容是自定义类的全限定类名


加载顺序： 只要找到一个实现就不会继续向下执行
- 1，自定义
- 2，SerializedModelConfigurator
- 3，DefaultJoranConfigurator
- 4，BasicConfigurator



###### 自定义



1，一个自定义的加载器应当具有以下格式
```
public class MyConfigurator  extends ContextAwareBase implements Configurator {  
  
  
    @Override  
    public ExecutionStatus configure(LoggerContext context) {  

1，加载配置文件
			ClassLoader myClassLoader= Loader.getClassLoaderOfObject(this);  
  
URL resource = getResource(configFile, myClassLoader);

2，处理加载文件，并判断是否该加载器是否运行成功
        if(resource==null){  
            return  ExecutionStatus.INVOKE_NEXT_IF_ANY;  //寻找其他  
        }  
        try {  
            configureByResource(resource);  
        } catch (JoranException e) {  
            throw new RuntimeException(e);  
  
        }  

        addInfo("自定义配置: logback-spring.xml 加载成功，");  
        return ExecutionStatus.DO_NOT_INVOKE_NEXT_IF_ANY;   //停止寻找其他  
  
    }
```


在以上自定义的过程中需要注意两点
1，适当的为上下文状态插入日志，也就是`ContextAwareBase`接口下实现的方法
2，ExecutionStatus菜单类的返回值，该返回值帮助调用者判断下一步该如何执行。




##### 2，上下文状态监听器

- 编码控制，动态添加： 由于无法获取添加之前的上下文状态，不去考虑
- 在配置文件使用使用标签`<statusListener>`指定`ch.qos.logback.core.status.StatusListener`的实现类
	-  可以自定义实现也可以使用官方的实现,且注意，该监听器并不能融入自定义的logger机制，他们只是对于上下文状态的日志监听！！

```
logback.xml
<configuration scan="true" scanPeriod="60 seconds" debug="false">  
  
    <property name="console" value="==\n[%-5level] %red(%d{HH:mm}) Thread:[%thread]  Method:%green(%M) %cyan(%X{traceId})   classpath:%c \n%highlight(return):%m%n"></property>  
  
    <statusListener class="ch.qos.logback.core.status.OnConsoleStatusListener" />  
    <statusListener class="ch.qos.logback.core.status.OnFileStatusListener" />  
  
    <root level="DEBUG">  
  
        <appender-ref ref="STDOUT"></appender-ref>  
    </root>  
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">  
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">  
            <pattern>${console}</pattern>  
            <charset>utf8</charset>  
        </encoder>    </appender></configuration>
```


##### 3，停止classic


> [!NOTE] Title
> 自1.1.10版本起，Logback-classic将自动请求Web服务器安装一个实现ServletContainerInitializer接口的LogbackServletContainerInitializer（在servlet-api 3.x及以后版本中可用）。这个初始化器反过来会安装一个LogbackServletContextListener的实例。当Web应用程序停止或重新加载时，这个监听器将停止当前的logback-classic上下文。


具体实现上，则是通过`shutdown`钩子实现的，且在1.5.18版本上，只有一个钩子`ch.qos.logback.core.hook.`



**这种关闭方式在底层的逻辑中关键语句为： loggerContext.stop();**，即关闭整个logback，那么同样也就是关闭了classic。


混淆点： 
- logbackContext并不是核心包的内容，或者说在真正使用中，单独的核心包并不能完成日志功能。
		`ch.qos.logback.classic.LoggerContext`

仔细排查之后发现，加载外部xml配置的Configurator都是classic的内容。




##### xml配置语法
![[Pasted image 20250514154750.png]]

ps： 有点长，等待阅读，给出一个我个人的控制台输出配置。
```
<property name="console" value="==\n[%-5level] %red(%d{HH:mm}) Thread:[%thread]  Method:%green(%M) %cyan(%X{traceId})   classpath:%c \n%highlight(return):%m%n"></property>

<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">  
    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">  
        <pattern>${console}</pattern>  
        <charset>utf8</charset>  
    </encoder></appender>
  
<root level="DEBUG">  
  
    <appender-ref ref="STDOUT"></appender-ref>  
</root>

<statusListener class="ch.qos.logback.core.status.OnConsoleStatusListener" />  
<statusListener class="ch.qos.logback.core.status.OnFileStatusListener" />
```