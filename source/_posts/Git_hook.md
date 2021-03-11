---
title: Git的webhook应用：代码push后自动构建网站
date: 2021-01-11 17:38:38
tags: 
    - git
categories:
    - Coding
---

## 1. Git上配置webhook

如图，填写1、2的信息

![image](https://dony-1257037510.cos.ap-chengdu.myqcloud.com/blog/webhook.png)

## 2. 在上面配置的URL中处理钩子请求

我的网站是php开发，构建起来比较简单，基本上就两个步骤：

(1) `git pull` 更新项目代码
(2) 覆盖一些配置文件

处理方式也很简单，`获取post请求信息`->`签名校验`->`执行shell命令`，简略代码如下

```php
// 1. 获取body
$content = file_get_contents('php://input');

// 2. 签名算法
$secret = ''; // Git上配置的secret
$signature = "sha1=" . hash_hmac('sha1', $content, $secret);

// 3. 和请求头中带的签名信息比较
if (strcmp($signature, $gitsign) == 0) { // 签名验证成功，执行shell命令
    ... // shell命令
} 
```

## 3. 给nginx运行用户拉取代码的权限

假设nginx的运行用户是`webuser`

(1) 把`webuser`用户的ssh-key添加到 **Git** 中

```shell
# 用webuser登录 Generating public/private ed25519 key pair
ssh-keygen -t ed25519 -C "yourmail@email.com"

# 复制公钥 添加到Git的配置中 
cd /home/apache/.ssh && cat id_ed25519.pub
```

(2) 给`webuser`网站项目的.git文件夹的权限

不然会报错：*error: cannot open .git/FETCH_HEAD: Permission denied*

```shell
cd /webroot && chown -R webuser:webuser .git/
```
