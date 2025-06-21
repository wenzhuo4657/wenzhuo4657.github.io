---
title: maven插件新手解惑
date: 2025-04-06 16:44:01
categories: 教程

---













## maven插件的定义



maven插件的定义是在pom文件下的

```java
<plugins>
    <plugin>
    		<groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <version>2.6.0</version>   
     </plugin>
 </plugins>
```





与常规的依赖定位一致，而插件的配置则是在<executions>、<configuration>等标签下定义的，他们或许是环境变量或许是在maven生命周期的插入点。





简单来说maven插件的两个用途

1,融入maven生命周期，增加外挂式的功能

2,不参入maven生周期，直接使用（例如，在微服务领域下，将服务的定义IDL文件编译为特定语言的接口文件。）





## maven插件的原理



**Maven 是一个意第绪语单词，意为 知识的积累者，最初是为了简化 Jakarta Turbine 项目的构建过程而创建的。当时有多个项目，每个项目都有自己的 Ant 构建文件，而且每个文件都略有不同。JAR 被签入 CVS。我们希望有一种标准的方法来构建项目，明确定义项目包含的内容，一种发布项目信息的简便方法，以及一种在多个项目之间共享 JAR 的方法。**





这段话取自[官网](https://maven.apache.org/what-is-maven.html)对于maven的定义,可以看出maven的核心在于管理jar文件，便于多个不同的项目去引入相同的jar。





**“Maven” 实际上只是一组 Maven 插件的核心框架。换句话说，插件是执行大部分实际操作的地方，插件用于：创建 jar 文件、创建 war 文件、编译代码、单元测试代码、创建项目文档等等。您能想到的在项目上执行的几乎所有操作都是作为 Maven 插件实现的。**



这段话取自[插件开发简介](https://maven.apache.org/guides/introduction/introduction-to-plugins.html),也就是说maven的插件的目标是生命周期，而生命周期在官方术语中被称为`Mojo`.





进而去查看maven插件开发的示例即可发现，插件实际上就是依托与maven这个框架所提供的扩展点所构建出来的可执行jar文件。







至此对于maven插件的和依赖的之间的关系就已经很明确了，他们实际上是一样，都可以作为依赖去使用，实际上更准确的说法是，可以融入maven生命周期的依赖被称为maven插件。





*如果想要学习maven插件开发，就一定要仔细查看maven的[官方文档](https://maven.apache.org/plugin-developers/index.html)*



## maven插件示例解读







### spring-boot-maven-plugin







#### 前置知识



什么是springboot?通常我们说springboot是对于各种依赖自动配置的boot工具，便于快速开发，避免依赖的困扰，这种方式目前已经不止是springboot的独有，只是它依托与spring生态最为出名而已。



springboot与spring最大的区别在我这个小白看来则是注解开发，@SpringBootApplication定义程序的启动入口，所有的程序都需要依托于这个主类进行加载，而传统的spring项目则需要依托与web容器启动servlet配置，也就WebApplicationInitializer（简单来说是用于将项目中所有的servlet加载到web容器当中，参如http请求当中的rest控制器部分）。



无论如何我们需要明白一个jar包的执行是需要一个入口！而不同项目的入口是不一样！



对于普通jar包来说，他只需要将当前项目的class文件打包即可，而一个可执行的jar包则额外的需要依赖、入口、引导器。（如果稍微了解一点jvm知识就应该明白，java程序并不会在加载阶段去校验import是否存在，至于执行更是需要一个明确的入口。）






可执行jar:  

- 将依赖引入jar包当中
- META-INF/MANIFEST.MF文件中存在启动类等配置
- 使用命令`java -jar` 执行

不可执行jar:
- 内部只有项目的class文件
-  META-INF/MANIFEST.MF文件中不存在启动类等配置
- 不能使用`java -jar` 执行。

java -jar执行逻辑： JVM会根据MANIFEST.MF文件中的Main-Class属性找到程序的入口点（即主类），然后开始执行该类的main方法









注意： 我找不到2.6的文档，凑和看[spring官方文档](https://docs.spring.io/spring-boot/maven-plugin/getting-started.html)的最新版本解说。

#### spring-boot-maven-plugin:repackage



*该repackage目标不应在命令行上单独使用，因为它会对 阶段生成的源jar（或）进行操作*



翻译成人话就是他需要在maven生命周期的打包阶段之后执行，并且无法单独执行，只能绑定在该打包阶段伴随执行。



```
           <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <version>2.6.0</version>

                <executions>
<!--                    每一个<execution>都是一个归档包的定义 -->
<!--                    可执行jar包配置-->
                    <execution>
                        <id>repackage</id>
                        <goals>
<!--                            绑定生命周期在package后-->
                            <goal>repackage</goal>
                        </goals>
                        <configuration>
<!--                            为可执行jar包新增后缀名，避免和普通jar包冲突-->
                            <classifier>exec</classifier>
                        </configuration>
                    </execution>
        </plugin>
```



classifier: 分类器。官网的解释很绕口，总之实现结果就是给重新打包的jar增加后缀，否则，就会替换普通jar，普通jar最终也只会在构建的target中出现。





该配置定义了一个将普通jar包重新打包、新增后缀exec的可执行jar包，且需要注意的是，他并不能影响原有的生命周期，具体来说则是maven仓库中依赖的存储，在仓库中依旧只有原本的普通jar的路径，无法将这个新增的jar引入其他项目当中。





如果我们想要引入可执行的jar包（无论处何种目的，总之如果有这种需求的话）



attach：将要安装到本地 Maven 存储库或部署到远程存储库的重新打包的存档附加到其中。如果未配置分类器，它将替换普通 jar。如果classifier已配置，使得普通 jar 和重新打包的 jar 不同，它将与普通 jar 一起附加

```
       <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <version>2.6.0</version>

                <executions>
<!--                    每一个<execution>都是一个归档包的定义 -->
<!--                    可执行jar包配置-->
                    <execution>
                        <id>repackage</id>
                        <goals>
<!--                            绑定生命周期在package后-->
                            <goal>repackage</goal>
                        </goals>
                        <configuration>
      
                 			<attach>true</attach>
                        </configuration>
                    </execution>
        </plugin>
       
```



这样配置之后，在仓库中存在的就是可执行的jar。而普通jar只会在target构建目录中出现。









#### spring-boot-maven-plugin:build-image





构建镜像用的，默认使用本地docker,远程docker要配置额外的东西，总之很麻烦，只在本地尝试了以下。





#### spring-boot-maven-plugin:help



插件各个功能的简单介绍。









除此之外还有一些，新版插件新增了aot提前编译功能，有机会可以自行尝试。







## maven官方插件与非官方插件





官方：maven-${prefix}-plugin
第三方：${prefix}-maven-plugin

官方可用插件：[Available Plugins – Maven](https://maven.apache.org/plugins/index.html)
非官方可用插件：[MojoHaus · GitHub](https://github.com/mojohaus)





参考文档：

[打包可执行存档文件 (Packaging Executable Archives) | Spring Boot3.4.0中文文档|Spring官方文档|SpringBoot 教程|Spring中文网](https://www.spring-doc.cn/spring-boot/3.4.0/maven-plugin_packaging.html)

[Spring Boot 打包成的 jar 和普通jar有什么区别\_spring boot 打成的 jar 和普通的 jar 有什么区别-CSDN博客](https://blog.csdn.net/weixin_45760137/article/details/118725697)

[Maven-插件使用与配置什么是插件 插件可以理解为一组任务的集合，用户可以通过命令行直接运行指定插件的任务，也可以将插 - 掘金](https://juejin.cn/post/7055598592999817253)



https://maven.apache.org/guides/introduction/introduction-to-plugins.html







## 结语



插件这东西吧，最开始总是搞不懂，一直都是复制粘贴别人的配置，经过这次学习，倒不是说技术成长了多少，哈哈，其实甚至没有技术，只是看文档尝试功能而已。



没错，就是看文档，一定学会看官方文档！！！











