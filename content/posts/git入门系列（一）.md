---
title: git入门系列（一）
date: 2025-03-12 15:37:15
layout : 'singlet'
---





# git是什么？





##  以键值数据库为基础的文件版本控制系统

参考：https://git-scm.com/book/zh/v2/%e8%b5%b7%e6%ad%a5-Git-%e6%98%af%e4%bb%80%e4%b9%88%ef%bc%9f

提醒： 本篇大部分在git官网都存在，只是顺序不一致，按我理解，这样有助于直接理解git的钩子机制。



传统的文件版本数据库通过累计文件差异来区分文件版本，而git则是通过不断累计文件快照，然后使用链接管理当前版本应该显示的文件！

- 好处：如果存储文件差异，则不可避免的需要从文件内部开始操作，所带来的是复杂且绝不能出错的高安全性要求，而git管理文件链接，换言之，他只需要保存每个版本的文件链接列表即可，对于真实的文件数据，他们互相隔离。
- 坏处：文件系统容易变得臃肿庞大。







参考:https://git-scm.com/book/zh/v2/Git-%e5%86%85%e9%83%a8%e5%8e%9f%e7%90%86-Git-%e5%af%b9%e8%b1%a1



文件快照存储的方式则是键值数据库，位于.git/objects 目录。



存储值，并返回唯一键（SHA-1 哈希值）：`echo 'test content' | git hash-object -w --stdin`

```
$ echo 'test content' | git hash-object -w --stdin
d670460b4b4aece5915caf5c68d12f560a9fe3e4
```

取出： 

```
$ git cat-file -p d670460b4b4aece5915caf5c68d12f560a9fe3e4
test content
```







此外还有树对象用于组织这些键值。

![image-20250312155433934](https://blog.wenzhuo4657.org/img/image-20250312155433934.png)

但是这仍然不是我们所熟知的git commit提交对象，这并没有提交信息的对应，这一点简单的说即是sha-1哈希值和文本的对应。随意找一个仓库都可以看到他们之间的关联。

![image-20250312155805995](https://blog.wenzhuo4657.org/img/image-20250312155805995.png)

## 待补充









# 钩子函数

git的钩子位于`.git\hooks`,默认情况下都是.sample结尾，去掉这个后缀即可启用。



![image-20250312160203955](https://blog.wenzhuo4657.org/img/image-20250312160203955.png)

尝试打开即可他先，他们都是统一的shell脚本，所以如果想要看懂是需要一点基础的。

统一机制：当以非零退出脚本时，停止提交。



提醒：shell脚本的运行并不是git管理的一部分，他们只是使用了git命令的一段编码，通常使用/bin/bash解释器运行。



## pre-commit：提交前执行



作用：当前分支是否存在，如果不存在会提供一个sha-1哈希值。





git rev-parse --verify HEAD  （该命令可以验证参数是否可以转换为sha-1哈希值，即他是否在对象数据库中有对应关系，常见有分支名、标签名）





## prepare-commit-msg：提交前执行

作用：

1，修改默认提交消息

2，添加gpt签名

3，将 git diff 结果插入提交消息的注释部分（咱不理解，自行查询）



修改默认消息最直接的途径就是在文件末尾`echo "default msg" > $1`,这样就会覆盖

`/usr/bin/perl -i.bak -ne 'print unless(m/^. Please enter the commit message/..m/^#$/)' "$COMMIT_MSG_FILE"`







# 推荐文档

[腾讯云](https://cloud.tencent.com/developer/doc/1096)

[git](https://git-scm.com/book/en/v2)

