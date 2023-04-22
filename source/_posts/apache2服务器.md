---
title: apache2服务器
date: 2023-04-22 00:15:52
tags:
---

title: 安装Apache2服务器
author: John Doe
date: 2022-01-11 21:06:21
tags:
---
Welcome to [Avrilzion blog](https://avrilzion.github.io/ifyblog)! This is your very first post.

1. 安装Apache2服务器

使用以下命令在Linux1上安装Apache2服务器：

```
sudo apt update
sudo apt install apache2
```

2. 配置Apache2服务器

打开Apache2的配置文件`/etc/apache2/sites-available/000-default.conf`，在`<VirtualHost *:80>`标签内添加以下内容：

```
ServerName www.skills.lan
ServerAlias any.skills.lan
RedirectMatch ^/(.*)$ https://www.skills.lan/$1
```

该配置将任何使用`skills.lan`或`any.skills.lan`访问的请求自动重定向到`www.skills.lan`。

接着，打开Apache2的默认配置文件`/etc/apache2/sites-available/default-ssl.conf`，修改以下内容：

```
SSLCertificateFile /etc/ssl/apache.crt
SSLCertificateKeyFile /etc/ssl/apache.key
```

并添加以下内容：

```
SSLEngine on
SSLVerifyClient require
SSLVerifyDepth 10
SSLCACertificateFile /etc/ssl/ca.crt
```

这将启用SSL，要求客户端提供证书，并使用`/etc/ssl/ca.crt`中的证书作为可信CA证书。

最后，将`/var/www/html/index.html`文件替换为以下内容：

```
<!DOCTYPE html>
<html>
<head>
	<title>apache</title>
</head>
<body>
	<h1>Welcome to Apache!</h1>
</body>
</html>
```

3. 生成证书文件

使用以下命令将`/etc/ssl/skills.crt`和`/etc/ssl/skills.key`转换为`/etc/ssl/skills.pfx`：

```
openssl pkcs12 -export -out /etc/ssl/skills.pfx -inkey /etc/ssl/skills.key -in /etc/ssl/skills.crt
```

接着，使用以下命令将`/etc/ssl/skills.pfx`转换为`/etc/ssl/skills.pem`：

```
openssl pkcs12 -in /etc/ssl/skills.pfx -out /etc/ssl/skills.pem -nodes
```

最后，使用以下命令从`/etc/ssl/skills.pem`提取证书和私钥，并分别保存到`/etc/ssl/apache.crt`和`/etc/ssl/apache.key`：

```
openssl x509 -in /etc/ssl/skills.pem -out /etc/ssl/apache.crt
openssl rsa -in /etc/ssl/skills.pem -out /etc/ssl/apache.key
```

4. 重启Apache2服务

使用以下命令重启Apache2服务：

```
sudo service apache2 restart
```

现在，当使用`skills.lan`或`any.skills.lan`访问Linux1时，请求将自动重定向到`www.skills.lan`，且要求客户端提供SSL证书。
