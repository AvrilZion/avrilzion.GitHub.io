---
title: dns服务
date: 2023-04-21 23:32:43
tags:
---



### 2. DNS服务

#### 2.1 防火墙设置

在Ubuntu下，可以使用 `ufw` 工具来管理防火墙规则。首先，需要启用防火墙，并设置默认规则为拒绝所有入站流量。

```
sudo ufw enable
sudo ufw default deny incoming
```

然后，需要允许 DNS 服务的流量通过防火墙。假设 DNS 服务使用的端口为 53（默认情况下是这个端口），则可以使用以下命令来放行该端口：

```
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
```

#### 2.2 NTP服务设置

使用 `chrony` 工具来配置NTP服务。首先，在linux1上安装 `chrony`：

```
sudo apt-get install chrony
```

然后，在 `/etc/chrony/chrony.conf` 文件中添加以下内容：

```
allow 192.168.0.0/24  # 允许本地网络中的主机使用NTP服务
```

最后，启动 `chrony` 服务并将其设置为开机启动：

```
sudo systemctl start chrony
sudo systemctl enable chrony
```

#### 2.3 SSH认证设置

为了禁用密码认证，我们需要使用公钥认证。首先，在每个Linux主机上生成公私钥对：

```
ssh-keygen -t rsa
```

然后，在每个主机上将公钥添加到授权文件中：

```
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

接下来，我们需要修改SSH配置文件 `/etc/ssh/sshd_config` ，禁用密码认证。找到以下两个配置项，并将其值改为 `no`：

```
PasswordAuthentication no
ChallengeResponseAuthentication no
```

最后，重启 SSH 服务：

```
sudo systemctl restart sshd
```

#### 2.4 DNS设置

首先，在linux1上安装 `bind`：

```
sudo apt-get install bind9
```

然后，在 `/etc/bind/named.conf.options` 文件中配置 DNS 服务器：

```
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { any; };
    forwarders {
        8.8.8.8; # Google DNS
        8.8.4.4;
    };
};

zone "skills.lan" IN {
    type master;
    file "/etc/bind/db.skills.lan";
    allow-update { none; };
};

zone "0.168.192.in-addr.arpa" IN {
    type master;
    file "/etc/bind/db.192";
    allow-update { none; };
};
```

这个配置文件中，我们允许任何主机进行 DNS 查询，同时将未知的 DNS 请求转发给 Google DNS 服务器。我们还配置了两个 DNS 区域：`skills.lan` 和 `0.168.192.in-addr.arpa`（这是内部网络的反向解析区域）。这些区域的信息将存储在 `/etc/bind/db.skills.lan` 和 `/etc/bind/db.192` 文件中。

现在，我们需要创建这些区域文件。
首先，创建`/etc/bind/db.skills.lan` 文件：

```
$TTL 3H
@       IN SOA  linux1.skills.lan. root.linux1.skills.lan. (
                1       ; Serial
                3H      ; Refresh
                15M     ; Retry
                1W      ; Expire
                1D      ; Minimum TTL
)
        IN NS   linux1.skills.lan.
        IN A    192.168.0.1
linux1  IN A    192.168.0.1
linux2  IN A    192.168.0.2
```

这个文件中定义了 `skills.lan` 区域的信息。第一行是 TTL（Time to Live），表示 DNS 记录在缓存中的时间。接下来的几行定义了区域的 SOA 记录和 NS 记录，以及三个主机的 A 记录。

然后，创建 `/etc/bind/db.192` 文件：

```
$TTL 3H
@       IN SOA  linux1.skills.lan. root.linux1.skills.lan. (
                1       ; Serial
                3H      ; Refresh
                15M     ; Retry
                1W      ; Expire
                1D      ; Minimum TTL
)
        IN NS   linux1.skills.lan.
        IN PTR  skills.lan.
linux1  IN A    192.168.0.1
linux2  IN A    192.168.0.2
```

这个文件中定义了内部网络的反向解析信息。第一行是 TTL，接下来的几行是 SOA 记录和 NS 记录，以及两个主机的 PTR（Pointer）记录，用于反向解析。

最后，启动 `bind` 服务并将其设置为开机启动：

```
sudo systemctl start bind9
sudo systemctl enable bind9
```

然后，在linux2上也安装 `bind`，并将其配置为备用 DNS 服务器。配置方法类似于linux1，只需要将 `/etc/bind/named.conf.options` 中的 `forwarders` 改为：

```
forwarders {
    192.168.0.1; # 主DNS服务器
};
```

#### 2.5 CA证书设置

首先，在linux1上安装 `easy-rsa` 工具：

```
sudo apt-get install easy-rsa
```

然后，使用 `easy-rsa` 工具初始化 CA（证书颁发机构）：

```
cd /usr/share/easy-rsa
sudo ./easyrsa init-pki
```

接下来，生成 CA 证书和私钥：

```
sudo ./easyrsa build-ca
```

在生成证书和私钥时，需要输入一些信息，如国家、省份、城市、组织名称等。这些信息将出现在证书中。

然后，创建服务器证书签名请求（CSR）：

```
sudo ./easyrsa gen-req server nopass
```

这个命令将生成一个名为 `server.req` 的文件，其中包含服务器的公钥和一些其他信息。在生成 CSR 时，需要输入服务器的公共名称（Common Name），即 `skills.lan`。

接下来，使用 CA签名服务器证书：

```
sudo ./easyrsa sign-req server server
```

这个命令将使用 CA 的私钥对 `server.req` 文件进行签名，生成一个名为 `server.crt` 的服务器证书文件。

然后，将证书和私钥文件复制到 `/etc/ssl` 目录：

```
sudo cp pki/issued/server.crt /etc/ssl/certs/
sudo cp pki/private/server.key /etc/ssl/private/
```

接下来，在 Apache2 中启用 SSL 模块：

```
sudo a2enmod ssl
```

然后，编辑 `/etc/apache2/sites-available/default-ssl.conf` 文件，配置 SSL 证书和私钥文件的路径：

```
SSLCertificateFile /etc/ssl/certs/server.crt
SSLCertificateKeyFile /etc/ssl/private/server.key
```

然后，启用 SSL 站点：

```
sudo a2ensite default-ssl.conf
```

最后，重启 Apache2 服务：

```
sudo systemctl restart apache2
```

现在，访问 `https://linux1.skills.lan` 就可以看到证书信息了，浏览器不会出现警告信息。如果需要为其他 Linux 服务器颁发证书，可以使用类似的方式生成 CSR、签名证书，然后将证书和私钥复制到对应服务器的 `/etc/ssl` 目录即可。

(4) 配置 DNS 服务器

接下来，我们要配置 DNS 服务器，为所有 Linux 主机提供冗余的 DNS 正反向解析服务。在这个例子中，我们将使用 BIND9 作为 DNS 服务器软件。

首先，在 `linux1` 上安装 BIND9：

```
sudo apt-get update
sudo apt-get install bind9 bind9utils bind9-doc
```

然后，编辑 BIND9 配置文件 `/etc/bind/named.conf.local`，添加以下内容：

```
zone "skills.lan" IN {
    type master;
    file "/etc/bind/db.skills.lan";
    allow-update { none; };
};

zone "1.168.192.in-addr.arpa" IN {
    type master;
    file "/etc/bind/db.192.168.1";
    allow-update { none; };
};
```

这段配置文件指定了 BIND9 的两个 DNS 区域：`skills.lan` 和 `1.168.192.in-addr.arpa`。`skills.lan` 区域用于域名解析，`1.168.192.in-addr.arpa` 区域用于 IP 地址反向解析。

然后，创建两个区域文件 `/etc/bind/db.skills.lan` 和 `/etc/bind/db.192.168.1`：

- `/etc/bind/db.skills.lan` 文件内容：

```
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     linux1.skills.lan. admin.skills.lan. (
                     2023042001         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      linux1.skills.lan.
@       IN      A       192.168.1.101
linux1  IN      A       192.168.1.101
linux2  IN      A       192.168.1.102
```

- `/etc/bind/db.192.168.1` 文件内容：

```
;
; BIND reverse data file for local loopback interface
;
$TTL    604800
@       IN      SOA     linux1.skills.lan. admin.skills.lan. (
                     2023042001         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      linux1.skills.lan.
101     IN      PTR     linux1.skills.lan.
102     IN      PTR     linux2.skills.lan.
```

这两个文件分别定义了 `skills.lan` 区域和 `1.168.192.in-addr.arpa` 区域的记录。

最后，重启 BIND9 服务：

```
sudo systemctl restart bind9
```

现在，所有 Linux 主机都可以通过 `linux1` 和 `linux2` 进行 DNS 解析和反向解析了。如果 `linux1` 挂了，`linux2` 会自动接管 DNS 服务。

(5) 配置 CA 服务器和证书

最后，我们要配置 CA 服务器，并为 Linux 主机颁发证书。在这个例子中，我们将使用OpenSSL 作为 CA 服务器软件。

首先，在 `linux1` 上安装 OpenSSL：

```
sudo apt-get update
sudo apt-get install openssl
```

然后，生成 CA 证书和私钥：

```
cd /etc/ssl
sudo mkdir CA
cd CA
sudo mkdir certs crl newcerts private
sudo chmod 700 private
sudo touch index.txt
echo 1000 > serial
sudo openssl genrsa -aes256 -out private/ca.key.pem 4096
sudo chmod 400 private/ca.key.pem
sudo openssl req -config /etc/ssl/openssl.cnf \
    -key private/ca.key.pem \
    -new -x509 -days 3650 -sha256 -extensions v3_ca \
    -out certs/ca.cert.pem
sudo chmod 444 certs/ca.cert.pem
```

这段命令生成了一个名为 `ca.key.pem` 的私钥和一个名为 `ca.cert.pem` 的 CA 证书。私钥被加密以保护其安全性。接下来，我们需要将证书复制到其他 Linux 主机。

```
sudo scp /etc/ssl/CA/certs/ca.cert.pem user@linux2:/tmp/
sudo scp /etc/ssl/CA/private/ca.key.pem user@linux2:/tmp/
```

这里的 `user` 是你在 `linux2` 主机上的用户名。

然后，在 `linux2` 上创建一个名为 `skills.lan.cnf` 的配置文件：

```
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn

[dn]
C = CN
ST = Beijing
L = Beijing
O = skills
OU = system
CN = skills.lan
```

这个配置文件用于创建证书签名请求。

接下来，在 `linux2` 上生成一个证书签名请求：

```
sudo openssl req -new -config skills.lan.cnf -keyout skills.key -out skills.csr
```

这个命令将生成一个名为 `skills.key` 的私钥和一个名为 `skills.csr` 的证书签名请求。

然后，将 `skills.csr` 文件复制到 `linux1` 上，并使用 `ca.cert.pem` 和 `ca.key.pem` 为证书签名请求签名：


```
sudo scp skills.csr user@linux1:/tmp/
sudo ssh user@linux1 "sudo openssl ca -config /etc/ssl/openssl.cnf \
    -extensions server_cert -days 1825 -notext -md sha256 \
    -in /tmp/skills.csr \
    -out /tmp/skills.cert.pem \
    -batch"
```

这个命令使用 `ca.cert.pem` 和 `ca.key.pem` 签署了 `skills.csr`，并生成了名为 `skills.cert.pem` 的证书。

最后，将 `skills.cert.pem` 和 `skills.key` 文件复制到需要证书的 Linux 服务器的 `/etc/ssl` 目录：

```
sudo scp user@linux1:/tmp/skills.cert.pem /etc/ssl/
sudo scp user@linux1:/tmp/skills.key /etc/ssl/
sudo chmod 644 /etc/ssl/skills.cert.pem
sudo chmod 400 /etc/ssl/skills.key
```

现在，`linux1` 作为 CA 服务器可以为其他 Linux 主机颁发证书让我们继续完善 HTTPS 服务器的配置。

首先，在 `linux1` 上安装 Apache2：

```
sudo apt-get update
sudo apt-get install apache2
```

然后，在 `linux1` 上创建一个名为 `skills.lan.cnf` 的配置文件：

```
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn

[dn]
C = CN
ST = Beijing
L = Beijing
O = skills
OU = system
CN = skills.lan
```

这个配置文件用于创建服务器证书签名请求。

接下来，在 `linux1` 上生成一个服务器证书签名请求：

```
sudo openssl req -new -config skills.lan.cnf -keyout skills.key -out skills.csr
```

这个命令将生成一个名为 `skills.key` 的私钥和一个名为 `skills.csr` 的证书签名请求。

然后，将 `skills.csr` 文件复制到 `linux1` 上，并使用 `ca.cert.pem` 和 `ca.key.pem` 为证书签名请求签名：

```
sudo scp skills.csr user@linux1:/tmp/
sudo ssh user@linux1 "sudo openssl ca -config /etc/ssl/openssl.cnf \
    -extensions server_cert -days 1825 -notext -md sha256 \
    -in /tmp/skills.csr \
    -out /tmp/skills.cert.pem \
    -batch"
```

这个命令使用 `ca.cert.pem` 和 `ca.key.pem` 签署了 `skills.csr`，并生成了名为 `skills.cert.pem` 的服务器证书。

接下来，为 Apache2 配置 SSL：

```
sudo a2enmod ssl
sudo cp /etc/apache2/sites-available/default-ssl.conf /etc/apache2/sites-available/skills-ssl.conf
sudo vi /etc/apache2/sites-available/skills-ssl.conf
```

这个命令将复制默认的 SSL 配置文件，并将其重命名为 `skills-ssl.conf`。然后使用 vi 编辑器打开 `skills-ssl.conf` 文件，将以下内容添加到文件的末尾：

```
SSLEngine on
SSLCertificateFile /etc/ssl/skills.cert.pem
SSLCertificateKeyFile /etc/ssl/skills.key
```

保存并关闭文件。

接下来，启用 `skills-ssl.conf` 配置文件：

```
sudo a2ensite skills-ssl.conf
```

然后重新启动 Apache2：

```
sudo service apache2 restart
```

现在，当用户访问 `https://skills.lan` 时，Apache2 将使用 `skills.cert.pem` 和 `skills.key` 文件提供 HTTPS 服务。并且，因为我们已经在 `linux1` 作为 CA 服务器为所有 Linux 主机颁发了证书，所以当用户访问其他 Linux 主机上的 HTTPS 站点时，不会出现证书警告信息。

最后，让我们在其他 Linux 主机上测试 HTTPS 站点。

首先，让我们在 `linux2` 上测试 HTTPS 站点：

```
sudo apt-get update
sudo apt-get install openssl
openssl s_client -connect linux1:443
```

这个命令将使用 OpenSSL 的 `s_client` 工具连接到 `linux1` 上的 HTTPS 站点。如果一切正常，您将看到类似于以下内容的输出：

```
CONNECTED(00000003)
depth=1 C = CN, ST = Beijing, L = Beijing, O = skills, OU = system, CN = skills.lan
verify return:1
depth=0 C = CN, ST = Beijing, L = Beijing, O = skills, OU = system, CN = skills.lan
verify return:1
---
Certificate chain
 0 s:C = CN, ST = Beijing, L = Beijing, O = skills, OU = system, CN = skills.lan
   i:C = CN, ST = Beijing, L = Beijing, O = skills, OU = system, CN = skills.lan
---
Server certificate
-----BEGIN CERTIFICATE-----
MIID...<省略>...QT7H
-----END CERTIFICATE-----
subject=C = CN, ST = Beijing, L = Beijing, O = skills, OU = system, CN = skills.lan

issuer=C = CN, ST = Beijing, L = Beijing, O = skills, OU = system, CN = skills.lan

---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 1077 bytes and written 481 bytes
Verification: OK
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---
```

如果您看到上面的输出，则表示 HTTPS 站点已成功配置。

接下来，让我们在另一台 Linux 主机上测试 HTTPS 站点。假设该主机的 IP 地址为 `192.168.1.100`，请使用浏览器访问 `https://skills.lan`。如果您使用的是 Firefox 浏览器，可能会出现以下对话框：

![Firefox security warning](https://i.imgur.com/3Edd6wh.png)

这是因为 Firefox 不信任我们刚刚创建的自签名证书。单击 "Advanced..."，然后单击 "Accept the Risk and Continue"，即可访问 HTTPS 站点。

如果您使用的是 Chrome 浏览器，则不需要进行任何特殊操作。Chrome 将默认信任我们刚刚创建的自签名证书。

至此，我们已经成功地创建了 DNS 服务器、NTP 服务器、SSH 服务器、HTTPS 服务器和 CA 服务器，并为所有 Linux 主机提供了冗余 DNS 正反向解析服务和证书颁发服务。

最后一步是将证书和私钥文件复制到需要证书的 Linux 服务器的 `/etc/ssl` 目录。我们已经为 `linux1` 上的 HTTPS 服务器创建了证书和私钥文件。现在，我们将这些文件复制到 `linux2` 上。

假设您已经在 `linux1` 上创建了证书和私钥文件，那么请使用以下命令将它们复制到 `linux2` 上：

```
sudo scp /etc/ssl/skills.crt /etc/ssl/skills.key linux2:/etc/ssl/
```

这个命令将 `linux1` 上的 `/etc/ssl/skills.crt` 和 `/etc/ssl/skills.key` 文件复制到 `linux2` 上的 `/etc/ssl/` 目录中。

现在，我们已经在 `linux2` 上复制了证书和私钥文件，让我们使用以下命令验证 `linux2` 上的 HTTPS 服务器是否工作正常：

```
sudo apt-get update
sudo apt-get install openssl
openssl s_client -connect linux2:443
```

这个命令将使用 OpenSSL 的 `s_client` 工具连接到 `linux2` 上的 HTTPS 站点。如果一切正常，您将看到类似于以下内容的输出：

```
CONNECTED(00000003)
depth=1 C = CN, ST = Beijing, L = Beijing, O = skills, OU = system, CN = skills.lan
verify return:1
depth=0 C = CN, ST = Beijing, L = Beijing, O = skills, OU = system, CN = skills.lan
verify return:1
---
Certificate chain
 0 s:C = CN, ST = Beijing, L = Beijing, O = skills, OU = system, CN = skills.lan
   i:C = CN, ST = Beijing, L = Beijing, O = skills, OU = system, CN = skills.lan
---
Server certificate
-----BEGIN CERTIFICATE-----
MIID...<省略>...QT7H
-----END CERTIFICATE-----
subject=C = CN, ST = Beijing, L = Beijing, O = skills, OU = system, CN = skills.lan

issuer=C = CN, ST = Beijing, L = Beijing, O = skills, OU = system, CN = skills.lan

---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 1077 bytes and written 481 bytes
Verification: OK
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---
```

如果您看到上面的输出，则表示 `linux2` 上的 HTTPS 站点已成功配置。

至此，我们已经完成了所有任务，成功地创建了 DNS 服务器、NTP 服务器、SSH 服务器、HTTPS 服务器和 CA 服务器，并为所有 Linux 主机提供了冗余 DNS 正反向解析服务和证书颁发服务。
