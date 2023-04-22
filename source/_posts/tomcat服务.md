---
title: tomcat服务
date: 2023-04-22 00:07:10
tags:
---



1. 配置linux2为nginx服务器

a. 安装Nginx

使用以下命令安装Nginx：

```
sudo apt-get update
sudo apt-get install nginx
```

b. 配置默认文档

默认情况下，Nginx的默认文档是index.html。打开默认文档的配置文件，并将默认文档修改为"hellonginx"：

```
sudo vim /etc/nginx/sites-available/default
```

找到index指令，将其修改为：

```
index  hellonginx;
```

保存并退出文件。

c. 配置HTTPS

安装certbot工具来申请免费的Let's Encrypt SSL证书：

```
sudo apt-get update
sudo apt-get install certbot python3-certbot-nginx
```

运行以下命令为Nginx服务器配置HTTPS：

```
sudo certbot --nginx -d your-domain-name
```

将your-domain-name替换为您的域名。按照命令提示完成配置。

d. 配置HTTP重定向到HTTPS

为了强制使用HTTPS，可以将所有HTTP请求重定向到HTTPS。打开Nginx配置文件：

```
sudo vim /etc/nginx/nginx.conf
```

在http块中添加以下代码：

```
server {
    listen 80;
    server_name your-domain-name;
    return 301 https://$server_name$request_uri;
}
```

将your-domain-name替换为您的域名。保存并退出文件。

重启Nginx服务以应用更改：

```
sudo service nginx restart
```


2. 利用nginx反向代理，实现linux3和linux4的tomcat负载均衡

a. 安装Tomcat

在linux3和linux4上分别安装Tomcat：

```
sudo apt-get update
sudo apt-get install tomcat9
```

b. 配置Tomcat

打开Tomcat的server.xml配置文件：

```
sudo vim /etc/tomcat9/server.xml
```

添加以下配置，启用AJP协议和负载均衡：

```
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
<Engine name="Catalina" defaultHost="localhost">
  <Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster">
    <Manager className="org.apache.catalina.ha.session.DeltaManager" expireSessionsOnShutdown="false" notifyListenersOnReplication="true"/>
    <Channel className="org.apache.catalina.tribes.group.GroupChannel">
      <Membership className="org.apache.catalina.tribes.membership.McastService" address="228.0.0.4" port="45564" frequency="500" dropTime="3000"/>
      <Receiver className="org.apache.catalina.tribes.transport.nio.NioReceiver" address="auto" port="4000" autoBind="100" selectorTimeout="5000" maxThreads="6"/>
      <Sender className="org.apache.catalina.tribes.transport.ReplicationTransmitter">
        <Transport className="org.apache.catalina.tribes.transport.nio.PooledParallelSender"/>
      </Sender>
      <Interceptor className="org.apache.catalina.tribes.group.interceptors.TcpFailureDetector"/>
      <Interceptor className="org.apache.catalina.tribes.group.interceptors.MessageDispatch        />
    </Channel>
    <Valve className="org.apache.catalina.ha.tcp.ReplicationValve" filter=""/>
    <Valve className="org.apache.catalina.ha.session.JvmRouteBinderValve"/>
    <ClusterListener className="org.apache.catalina.ha.session.ClusterSessionListener"/>
  </Cluster>
</Engine>
```

保存并退出文件。

c. 配置Nginx

打开Nginx的配置文件：

```
sudo vim /etc/nginx/sites-available/default
```

添加以下配置，用于反向代理Tomcat服务器：

```
upstream tomcat_backend {
    server linux3:8080 weight=5;
    server linux4:8080 weight=5;
}

server {
    listen 443 ssl;
    server_name tomcat.skills.lan;

    ssl_certificate /etc/ssl/skills.jks;
    ssl_certificate_key /etc/ssl/skills.jks;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://tomcat_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

server {
    listen 80;
    server_name tomcat.skills.lan;
    return 301 https://$server_name$request_uri;
}
```

将upstream指令中的服务器地址和端口修改为您的Tomcat服务器的地址和端口。将ssl_certificate和ssl_certificate_key指令中的路径修改为您的SSL证书路径。保存并退出文件。

重启Nginx服务以应用更改：

```
sudo service nginx restart
```


3. 配置linux3和linux4为Tomcat服务器

a. 配置默认首页

在Tomcat的webapps目录下创建ROOT文件夹，并在该文件夹中创建index.jsp文件。打开index.jsp文件，并将默认内容修改为"tomcatA"或"tomcatB"：

```
sudo mkdir /var/lib/tomcat9/webapps/ROOT
sudo vim /var/lib/tomcat9/webapps/ROOT/index.jsp
```

```
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
	<title>TomcatA</title>
</head>
<body>
	<h1>TomcatA</h1>
</body>
</html>
```

保存并退出文件。

在另一个Tomcat服务器上重复此过程，将index.jsp文件中的内容修改为"tomcatB"。

b. 配置HTTP和HTTPS

打开Tomcat的server.xml配置文件：

```
sudo vim /etc/tomcat9/server.xml
```

在<Connector>元素中添加以下配置，启用HTTP和HTTPS连接：

```
<Connector port="80" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="443" />

<Connector port="443" protocol="HTTP/1.1" SSLEnabled="true"
           maxThreads="150" scheme="https" secure="true"
           keystoreFile="/etc/ssl/skills.jks"
           keystorePass="your-password"
           clientAuth="false" sslProtocol="TLS" />
```

将keystoreFile指令中的路径和keystorePass指令中的密码修改为您的SSL证书路径和密码。保存并退出文件。

重启Tomcat服务以应用更改：

```
sudo service tomcat9 restart
```

在另一个Tomcat服务器上重复此过程。

现在，您应该能够通过https://tomcat.skills.lan访问负载均衡Tomcat集群，并在每个Tomcat服务器上看到不同的默认首页内容。

以上就是采用Tomcat搭建动态网站的完整步骤。希望这对您有所帮助！
