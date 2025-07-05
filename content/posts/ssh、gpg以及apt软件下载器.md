+++
date = '2025-06-29T10:00:14+08:00'
draft = false
title = 'ssh、gpg以及apt软件下载器'
layout = 'singlet'
+++







# 小白级别的理解，勿cue





首先说结论，ssh、gpt是用于加密，且在加密这个分类下属于非对称加密（指公钥加密、私钥解密），而apt是ubunt等linux系统的一个软甲下载器。







## ssh、gpg

###    对称加密和非堆成加密
对称加密算法使用相同的密钥用于加密或者解密，这使通信双方必须完全信任对方，才能够发送密钥，否则密钥就很可能被泄露，并且在传输过程中泄露也很糟糕。

但是由于使用相同的密钥进行加密和解密，所以x速度比较快。

而非对称加密算法将密钥分为私钥和公钥，其中私钥用于解密，公钥用于加密。所以相比于同级别的对称加密，非对称加密速度更慢，但安全性更高，我们只需要将公钥发布到任意服务器上即可实现通信（例如 keyserver）



### ssh和gpg的区别？

ssh： 常用于服务器加密（实时交互，需要处理动态数据），强度相当于钥匙。

gpg： 常用于文件加密（静态数据），强度相当于保险箱。





### pgp、gpg、OpenPGP



[OpenPGP（PGP/GPG）深入浅出，完全入门指南](https://www.rmnof.com/article/openpgp-gnupg-introduction/)、

- PGP：由Phil Zimmermann开发，最终被赛门铁克收购，是一个商业软件，需要付费。
- OpenPGP：一种协议，定义了加密消息、签名、私钥和用于交换公钥的证书统一标准。
- GPG（GnuPG）：符合OpenPGP标准的开源加密软件，PGP的开源实现。



## 使用



### 创建密钥

官方推荐的工具

ssh：[OpenSSH](https://www.openssh.com/?spm=a2ty_o01.29997173.0.0.299fc921OTUceN)

gpg： [GnuPG](https://gnupg.org/index.html)



### 公钥服务器





[OpenPGP 密钥服务器](https://keyserver.ubuntu.com/?spm=a2ty_o01.29997173.0.0.299fc921OTUceN#)

[OpenPGP 密钥服务器](https://keyserver.ubuntu.com/?spm=a2ty_o01.29997173.0.0.299fc921OTUceN#)



### 使用gpg密钥





对于ssh直接进行服务器链接等操作即可，但是gpg密钥是令人迷惑的，因为他是属于静态文件的加密，这意味着灵活性低、安全性高，而这样的加密特性势必要求一个符合的场景，例如：通信的双方必须信任对方，否则没有进行交换文件的必要。





[2021年，用更现代的方法使用PGP（下） - C的博客 |UlyC](https://ulyc.github.io/2021/01/26/2021%E5%B9%B4-%E7%94%A8%E6%9B%B4%E7%8E%B0%E4%BB%A3%E7%9A%84%E6%96%B9%E6%B3%95%E4%BD%BF%E7%94%A8PGP-%E4%B8%8B/)

ps： 这篇文章中说的漏洞已经被解决了，需要注意的是里面所提到的交换公钥的方式







#### gpg的特征



这里的特征是指gpg密钥上的签名、指纹等名词，他们在很长一段时间一直困扰我。





##### 1，  创建
` gpg   --full-generate-key` 该选项表示功能齐全的密钥生成



```
PS C:\Users\25370> gpg  --full-generate-key
gpg (GnuPG) 2.4.3; Copyright (C) 2023 g10 Code GmbH
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Please select what kind of key you want:
   (1) RSA and RSA
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
   (9) ECC (sign and encrypt) *default*
  (10) ECC (sign only)
  (14) Existing key from card
Your selection? 1
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072) 2048
Requested keysize is 2048 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 1y
Key expires at 2026/5/25 21:39:55 �й���׼ʱ��
Is this correct? (y/N) y

GnuPG needs to construct a user ID to identify your key.

Real name: wenzhuo4657
Email address: 14783149521@163.com
Comment: test
You selected this USER-ID:
    "wenzhuo4657 (test) <14783149521@163.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? q
gpg: Key generation canceled.
PS C:\Users\25370>
```



这一部分定义了uid： wenzhuo4657 (test) <14783149521@163.com>

```
Real name: wenzhuo4657
Email address: 14783149521@163.com
Comment: test
You selected this USER-ID:
    "wenzhuo4657 (test) <14783149521@163.com>"
```





##### 2，指纹



```
PS C:\Users\25370> gpg --fingerprint 14783149521@163.com
pub   rsa2048 2025-04-30 [SC] [expires: 2028-04-30]   
      BDB8 0382 F6A8 9F91 BB32  686F 9C3D 67F6 22CD 65C3
uid           [ultimate] WENZHUO HOU <14783149521@163.com>
sub   rsa2048 2025-04-30 [E] [expires: 2028-04-30]

```



pub： 表示公钥，用于交换。

sub：表示私钥，不可泄露





`rsa2048 2025-04-30 [SC] [expires: 2028-04-30] ` ：对密钥的描述，值得注意的是描述符号[SC]、[E]



| 标志 | 含义         | 用途说明                              |
| ---- | ------------ | ------------------------------------- |
| `S`  | Sign         | 签名文件或提交（如 Git 提交签名）     |
| `C`  | Certify      | 认证 UID 或签名其他密钥（主密钥才有） |
| `E`  | Encrypt      | 加密文件或消息                        |
| `A`  | Authenticate | 用于身份认证（如 SSH 登录替代）       |





`BDB8 0382 F6A8 9F91 BB32  686F 9C3D 67F6 22CD 65C3`: 指纹，唯一标识密钥、验证身份、构建信任链





##### 3，子钥和公钥、签名和验签



这里需要强调： 

- gpg的公钥!=密钥当中的公钥！gpg公钥是公开的，其中包含所有附带的公钥以及*私钥的描述信息*，也就是说他并不包含真正私钥！

- 主公钥是描述一般是[SC],其中包含了对加密文件进行签名的功能，但真正执行签名的私钥！
- 对gpg密钥的修改（添加、删除子密钥等操作）都需要重新发布gpg公钥









#### 管理gpg密钥





除去正常的私钥和公钥的管理，gpg在面对多用户场景下，还支持密钥的溯源，毕竟一个用户不可能只使用一个密钥，也有可能是一个组织下有多个密钥，最终指向一个具备公信力的指纹！













### 使用ssh密钥



ssh的描述信息只有一个邮箱，而gpg拥有uid的唯一表示，且注意，对于gpg而言，他需要面临的时多用户场景，uid可以快速区分，而指纹可以唯一标识一个密钥。







由于场景单一、简单，没啥好说的，基本就是两个端点进行加密通信，一个加密一个解密。







#  apt





呃呃，实际上apt才是这篇博文的出发点，但是没想到拓展了这么多，甚至对于gpg密钥的管理还有很多没有了解、实操，hhhh







apt是ubuntu默认的软件安装器，并且默认添加了官方源。从使用上来说相当简单，基本就是`apt install``apt remove`等简单的api。



值得注意的是apt添加第三方软件源时，使用到gpg公钥进行加密传输！





## 软件源的添加



1， 查看软件源


-   官方软件源
		-  cat   /etc/apt/sources.list
-  第三方软件源
		-  ls /etc/apt/sources.list.d/
		-  该目录下的软件源单独存放在 `名称.list`文件中，格式同官方软件源


命令查看
`apt-cache policy | grep http | awk '{print $2 $3}' | sort | uniq`




2，添加软件源


软件源的定义
`类型 [选项] 仓库URL 发行版代号 组件1 [组件2 ...]`


对于第三方软件源通常需要使用gpt密钥验证安全性，
例如：
```
1,adoptium软件源定义
deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://mirrors.tuna.tsinghua.edu.cn/Adoptium/deb jammy main


ps： 其中signed-by代表gpt密钥的位置，通常使用curl命令下载到本地
```





（1），前置准备
`apt-get update && apt-get install -y wget apt-transport-https`
或
`apt update && apt install -y wget apt-transport-https`


wegt： 下载工具

apt-transport-https： 允许 APT 使用 HTTPS 协议访问仓库源



（2）下载gpt密钥
`wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc`



下载gpg文件  `wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public `

写入文件 `tee /etc/apt/keyrings/adoptium.asc``


(3)添加第三方软件源

ps： 如果是添加官方软件源，直接写入/etc/apt/sources.list即可


软件源定义 `deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://mirrors.tuna.tsinghua.edu.cn/Adoptium/deb jammy main`



完整命令  
`echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://mirrors.tuna.tsinghua.edu.cn/Adoptium/deb jammy main" | tee  /etc/apt/sources.list.d/adoptium.list `

(4) 更新索引，就可以正常使用了
`apt update







## apt的基本操作



这一部分主要是我的日常记录的命令，因为很不方便，例如安装的软件经常找不到或者忘记了名称，难以删除。



apt和apt-get的区别
功能上无区别，apt是apt-get的新版本

查看已安装软件版本
`apt show `

查看软件安装的位置
` dpkg -L  软件名称`

搜想相关软件包 
`apt search  软件包名称/相关单词`
例如： `apt search jdk `

查看软件版本
`apt-cache madison maven`

安装指定版本
`sudo apt-get install maven=3.6.0-1~18.04.1`
也可不指定，默认安装最新版本
`sudo apt-get install maven









































