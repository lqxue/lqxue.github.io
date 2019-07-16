---
title: git push port 22 Connection timed out
date: 2019-07-16
tags:
  - Git更改端口
categories: Git
---

## 问题

提交代码是出现问题：

```java
$ git push origin master
ssh: connect to host github.com port 22: Connection timed out
fatal: Could not read from remote repository.
```

接下来我们在重新尝试下ssh连接github

```java
$ ssh git@github.com
ssh: connect to host github.com port 22: Connection timed out
```

看来是端口22问题,更换443试试

<!-- more -->

```java
ssh -T -p 443 git@ssh.github.com
The authenticity of host '[ssh.github.com]:443 ([192.30.253.123]:443)' can't be established.
RSA key fingerprint is SHA256:nThbg6kXUpJWGl7E1IGOCspRomTxdCARLviKw6E5SY8.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

确定了是端口的问题

## 解决方法

更改ssh连接端口

在存放私钥公钥（id_rsa和id_rsa.pub）文件里，新建config文本。命令`vim ~/.ssh/config`,输入一下内容：

```java
Host github.com
Hostname ssh.github.com
PreferredAuthentications publickey
IdentityFile ~/.ssh/id_rsa
Port 443
```

`:wq`保存退出。

重新测试

```java
$ ssh git@github.com
Warning: Permanently added the RSA host key for IP address '[192.30.253.123]:443' to the list of known hosts.
PTY allocation request failed on channel 0
Hi lqxue! You've successfully authenticated, but GitHub does not provide shell access.
Connection to ssh.github.com closed.
```
