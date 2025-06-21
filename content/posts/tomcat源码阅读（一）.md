---
title: tomcat源码阅读（一）
date: 2025-03-07 16:07:46
categories: 源码
---





# tomcat源码



##  idea环境搭建



源码版本apache-tomcat-9.0.43-src

https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.43/src/



注意： 不同版本的搭建方式可能略有不同，但这源码核心是大差不差的。







### 安装ant(已安装的可以直接跳过)

Apache Ant 是一个帮助构建软件的 Java 库和命令行工具。

官网：https://ant.apache.org/bindownload.cgi



直接下载最新版即可，解压之后设置环境变量



```
ANT_HOME=/Library/Apache/apache-ant-1.9.15
PATH=$PATH:$ANT_HOME/bin
```





使用`ant -verison`验证安装是否成功

![image-20250307171403529](https://blog.wenzhuo4657.org/img/image-20250307171403529.png)

### 构建项目



*在打开idea之前操作，这一点非常重要！！！否则会由于编译器的一些操作导致XXXXX*

直接将res目录idea-support中的.idea文件粘贴到根目录中的.idea（手动创建

![image-20250309134412453](https://blog.wenzhuo4657.org/img/image-20250309134412453.png)

![image-20250309134454556](https://blog.wenzhuo4657.org/img/image-20250309134454556.png)

![image-20250309134544101](https://blog.wenzhuo4657.org/img/image-20250309134544101.png)





此时打开项目可以发现项目被正确加载，但是会依赖报错

![image-20250309135659189](https://blog.wenzhuo4657.org/img/image-20250309135659189.png)



尝试构建项目，启动test

![image-20250309135743059](https://blog.wenzhuo4657.org/img/image-20250309135743059.png)



![image-20250309134654470](https://blog.wenzhuo4657.org/img/image-20250309134654470.png)



将这些jar包和ant目录下的lib库加入项目的外部库即可。

![image-20250309141125493](https://blog.wenzhuo4657.org/img/image-20250309141125493.png)

## 启动



找到启动类Bootstrap



![image-20250309141245515](https://blog.wenzhuo4657.org/img/image-20250309141245515.png)

![image-20250309141253099](https://blog.wenzhuo4657.org/img/image-20250309141253099.png)



# 中文乱码



对于乱码，网上搜索在日志配置中更改编码，

https://blog.csdn.net/weixin_44109450/article/details/126544310



但是对于源码编译环境来说，似乎并不会读取conf下的配置，追溯源码找到`java.util.logging.ConsoleHandler`.

断点调试发现，并没有获取到配置

![image-20250309142742777](https://blog.wenzhuo4657.org/img/image-20250309142742777.png)

并且由于该类属于依赖包jdk的一部分，![image-20250309142835530](https://blog.wenzhuo4657.org/img/image-20250309142835530.png)

进一步追溯之后发现关键读取文件的方法LogManager#readConfiguration（）

```
   String fname = System.getProperty("java.util.logging.config.file");
        if (fname == null) {
            fname = System.getProperty("java.home");
            if (fname == null) {
                throw new Error("Can't find java.home ??");
            }
            File f = new File(fname, "lib");
            f = new File(f, "logging.properties");
            fname = f.getCanonicalPath();
        }
        try (final InputStream in = new FileInputStream(fname)) {
            final BufferedInputStream bin = new BufferedInputStream(in);
            readConfiguration(bin);
        }
```

首先说结论，无论使更改成conf还是jdk目录下的logging.properties,他们所影响的编码都只是外层的一部分，不能影响tomcat输出的报错信息。

但是我注意到了另外一个属性java.util.logging.ConsoleHandler.formatter，该属性用于格式化输出的文本，但是默认的使jdk的jar包实现，所以我转到tomcat的配置文件中寻找实现，尝试修改启动后

```
java.util.logging.ConsoleHandler.level = INFO
# java.util.logging.ConsoleHandler.formatter = java.util.logging.SimpleFormatter
java.util.logging.ConsoleHandler.formatter = org.apache.juli.OneLineFormatter
java.util.logging.ConsoleHandler.encoding= UTF-8
```



修改后无效，放弃处理，在SimpleFormatter#format后追溯输出语句，最终追溯的源码包下的适配类`org.apache.juli.logging.DirectJDKLog#log`,查看输出类后发现他们都使用`StringManager.getString(final String key, final Object... args) `格式化。





最终追踪到资源包类`PropertyResourceBundle`,发现资源读取的全是乱码，而编码格式均是正确的。

追溯读取资源的源码后找到关键方法

`bundle = *findBundle*(cacheKey, candidateLocales, formats, 0, control, baseBundle);`





查看输出乱码的类，去寻找StringManager的绑定资源可发现

对应资源名称为`org.apache.catalina.startup.LocalStrings zh_CN `

![image-20250309170614673](https://blog.wenzhuo4657.org/img/image-20250309170614673.png)

![image-20250309170334320](https://blog.wenzhuo4657.org/img/image-20250309170334320.png)







## 解决

https://blog.csdn.net/weixin_54430656/article/details/122440049



最终解决实际上是我的一个误区，在前面的编码转换时我总是将错误的编码转换为utf-8,而忽略了java读取默认时`iso8859-1`的编码读取才导致的中文乱码。

```
      if (bundle != null) {
                str = bundle.getString(key);
                str= new String(str.getBytes(StandardCharsets.ISO_8859_1), StandardCharsets.UTF_8);
            }
            
```



![image-20250309172916003](https://blog.wenzhuo4657.org/img/image-20250309172916003.png)









