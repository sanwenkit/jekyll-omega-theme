---
layout: post
title: 使用openssl构建多层CA体系
description: "记录PKI证书信任链体系的生成配置"
category: develop
tags: [develop]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---

# 使用openssl构建多层CA体系
### 生成RootCA
在/etc/ssl/openssl.conf中增加配置项
```
[ banma_ca ]
basicConstraints = critical,CA:TRUE,pathlen:3
keyUsage =  cRLSign,keyCertSign
```

使用以下命令生成RootCA公私钥：
`openssl req -x509 -sha256 -nodes -days 10950  -config /etc/ssl/openssl.conf -extensions banma_ca -newkey rsa:2048 -keyout rootCA.key -out rootCA.crt`

### 利用RootCA签发二级Fota CA

创建二级FOTA CA私钥及CSR请求文件
`openssl genrsa -out secondca.key 2048`
`openssl req -new -sha256 -key secondca.key -out secondca.csr`
并根据提示完成DN配置

配置ca.conf文件：
```
[ ca ]
default_ca = myca
[ crl_ext ]
issuerAltName=issuer:copy
authorityKeyIdentifier=keyid:always
[ myca ]
dir =./
new_certs_dir = $dir
unique_subject =no
certificate = $dir/rootca.crt
database = $dir/certindex
private_key = $dir/rootca.key
serial = $dir/certserial
default_days =10950
default_md = sha256
policy = myca_policy
x509_extensions = myca_extensions
crlnumber = $dir/crlnumber
default_crl_days =730
[ myca_policy ]
countryName = optional
stateOrProvinceName = supplied
localityName		= supplied
organizationName = supplied
organizationalUnitName = optional
commonName = supplied
emailAddress = optional
[ myca_extensions ]
basicConstraints = critical,CA:TRUE,pathlen:3
keyUsage =  cRLSign,keyCertSign
[ v3_ca ]
basicConstraints = critical,CA:TRUE,pathlen:3
keyUsage =  cRLSign,keyCertSign
```

初始化certindex、certserial、crlnumber
```
touch certindex
echo 1000 > certserial
echo 1000 > crlnumber
```

利用RootCA公私钥签发证书
`openssl ca -batch -config ca.conf -extensions v3_ca -notext -in secondca.csr -out secondca.crt`

### 利用Fota CA证书签发三级Fota upgrade CA

创建三级FOTA CA私钥及CSR请求文件
`openssl genrsa -out thirdca.key 2048`
`openssl req -new -sha256 -key thirdca.key -out thirdca.csr `
并根据提示完成DN配置

利用Fota CA公私钥签发证书
`openssl ca -batch -config ca.conf -extensions v3_ca -notext -in thirdca.csr -out thirdca.crt -cert keys/secondca.crt -keyfile keys/secondca.key`

将private key格式转化为二进制（pem —> der）
`openssl pkcs8 -topk8 -in thirdca.key -nocrypt -inform PEM -outform DER -out thirdca.der`

### 利用Fota upgrade CA生成终端证书

创建endcert.conf配置文件：
```
[ ca ]
default_ca = myca
[ myca ]
dir =.
new_certs_dir = $dir
unique_subject =no
certificate = $dir/secondca.crt
database = $dir/certindex
private_key = $dir/secondca.key
serial = $dir/certserial
default_days =10950
default_md = sha256
policy = myca_policy
x509_extensions = myca_extensions
[ myca_policy ]
countryName     = match
stateOrProvinceName = match
localityName    = supplied
organizationName    = match
organizationalUnitName  = optional
commonName      = supplied
emailAddress        = optional
[ myca_extensions ]
basicConstraints = CA:FALSE
keyUsage =  digitalSignature,keyEncipherment
[ v3_ca ]
basicConstraints = CA:FALSE
keyUsage =  digitalSignature,keyEncipherment
```

生成终端证书：
```
openssl genrsa -out fota.key 2048
openssl req -new -sha256 -key fota.key -out fota.csr
openssl ca -batch -config endcert.conf -extensions v3_ca -notext -in fota.csr -out fota.crt -cert thirdca.crt -keyfile thirdca.key
```

### CTL可信证书列表签名

file.txt 为所有有效证书SerialNum1 in Hex的拼接，使用Fota CA私钥进行sha256签名
`openssl dgst -sign RSA.pem -sha256 -out sign.txt file.txt`

将二进制签名文件转位base64
`openssl base64 -in sign.txt -out base64.txt`

### 证书链验证

将Fota CA证书导入/etc/ssl/certs
```
cp secondca.crt /etc/ssl/certs
cd /etc/ssl/certs
ln -s secondca.crt `openssl x509 -hash -noout -in secondca.crt`.0
```

使用openssl验证证书链：
`openssl verify -CApath /etc/ssl/certs/ -untrusted <pathToIntermediateCert> <pathToYourCert>`

验证结果应为：fota.crt: OK
