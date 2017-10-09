---
layout: post
title: NXP安全启动加固配置
description: "imx6芯片的安全启动特性支持"
category: develop
tags: [develop]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---


# NXP IMX6芯片安全启动

### 安全启动原理
NXP IMX6芯片提供安全启动功能，代码发布者可以通过RSA公私钥签名的方式对自己的固件代码进行签名。来确保芯片中运行的系统使用为可信任的程序，而不是被第三方恶意刷入的未知代码。

一个签名后的固件代码内存布局如下图所示：

![HABv4 Sign Code](http://7xwdx7.com1.z0.glb.clouddn.com/nxp-signed-image.png)

启动后，bootloader首先通过FUSE烧入的证书Hash验证SRK根证书，随后通过根证书完成证书链认证。最后，通过证书的公私钥签名认证完成代码的有效性确认。只有通过认证的代码才会被加载启动。

### 安全启动配置
对于IMX6芯片，NXP官方提供了代码签名工具“Freescale Code Signing Tool”，用来进行签名固件的创建。该工具可以用来创建用以进行签名的证书链PKI体系，并利用这些证书完成代码签名。

CST工具中维护的PKI证书体系结构如下图所示：

![PKI Tree](http://7xwdx7.com1.z0.glb.clouddn.com/NXP-PKI.png)
其中RootCA下最多可包含四个Super Root Key CA（SRK），这四个SRK CA作为immediateCA可以继续签发CSF证书、IMG证书。一个SRK CA可签发多个CSF证书、IMG证书。这些三级叶子证书用于真正的CSF和代码镜像的签名。

### 利用CST导入现有RootCA生成签名证书
由于在其他安全加固工作中，我们已经建设了私有的PKI/CA证书体系，因此在这里我们考虑将该证书体系的RootCA导入到CST工具中，用来生成签名证书。

签名证书的生成可以参考文档“Secure Boot on i.MX50, i.MX53, and i.MX 6
Series using HABv4”，主要流程为：

* 在keys目录下创建key_pass.txt文本文件，输入私钥加密密码
* 修改ca目录下v3_ca.cnf、v3_usr.cnf配置文件，注释掉authorityKeyIdentifier字段
* 运行keys目录下的hab4_pki_tree.sh脚本
* 通过脚本导入RootCA公私钥（输入的字符串+.pem代表导入的文件名）
* 选择对应的算法、密钥长度、有效期、SRK数量
* 生成对应证书

脚本正常运行后，会将生成的证书文件存放在crts目录，生成的私钥签名文件存放在keys目录。

### 利用CST生成FUSE烧录文件
为了完成SecureBoot的相关配置，我们还需要将生成的SRK证书hash制作成FUSE烧录文件，烧写到硬件中。利用CST工具，生成FUSE烧录文件的命令如下：

{% highlight shell %}

../linux64/srktool -h 4 -t SRK_1_2_3_4_table.bin -e SRK_1_2_3_4_fuse.bin -d sha256 -c ./SRK1_sha256_2048_65537_v3_ca_crt.pem,./SRK2_sha256_2048_65537_v3_ca_crt.pem,./SRK3_sha256_2048_65537_v 3_ca_crt.pem,./SRK4_sha256_2048_65537_v3_ca_crt.pem -f 1

{% endhighlight %}

### FUSE烧录结果确认
在拿到烧录完成的硬件后，可以通过如下步骤判断烧写的数据是否与原始数据一致。首先获取烧录二进制文件中的证书hash值：

{% highlight shell %}

hexdump -e '/4 "0x"' -e '/4 "%X"\n"' SRK_1_2_3_4_fuse.bin

{% endhighlight %}

随后在imx6主板的uboot中使用fuse命令，读取写入的hash值：

{% highlight shell %}

fuse prog 3 0-7

{% endhighlight %}

比较两者是否一致。
