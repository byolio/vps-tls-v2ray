# 如何搭建v2ray服务器及其使用方法(主笔正在奋笔疾书中...)
## 前言
本文是搭建V2Ray服务器（ws+tls）的教程。除了搭建V2Ray+ws+tls的过程，还包括打开BBR加速，以及简单的防止检测。
随着GFW封杀力度的不断加大, 很多纯VMess都被封杀掉了。如果你打算自己搭建翻墙服务，强烈推荐V2Ray+ws+tls（CDN, BBR可选）一步到位。

本文不面向小白, 要求拥有一定的linux和DNS知识储备且会使用SSH连接服务器。如果你都满足, 那么看懂本教程就毫无压力。

本文不推荐使用一键脚本，很多一键脚本都存在安全隐患，轻则屏蔽掉几个网站，重则把你的服务器变成肉鸡（即黑客攻击别人电脑的跳板）。就算用脚本搭建服务器，脚本也无法自动完成本文中购买域名，配置域名解析等等。你恐怕还要亲自购买域名，亲自配置域名解析，亲自登录VPS执行脚本。本文中的方法只比一键脚本多出几步，但是可以大大降低安全风险。

自建V2Ray服务器首先要购买VPS（在国外的虚拟主机）和 域名 (用于tls证书分发)，为避免广告嫌疑，正文中不推荐VPS和域名购买网站, 需要大家自行购买。

V2Ray+ws+tls是目前最稳定的翻墙技术之一, V2Ray+ws+tls用真正的https流量翻墙, 在GFW看来，你的流量和不计其数的https流量没有任何区别。

## 域名解析(自行购买vps和域名后)
我使用的是cloudflare进行域名解析, 大家也可以使用DNSPod, 阿里云DNS, 百度云DNS, Amazon Route, Google Cloud DNS, GoDaddy等的DNS解析服务。
我这边选择的是cloudflare的free计划, 其足够完成该项目了。
![free](https://cdn.jsdelivr.net/gh/byolio/mytc@main/img/free-plan.png)
接下来配置DNS记录, 更新服务商配置, 完成DNS解析。
解析记录中常见的几种为:
1. A记录：即地址（Address）记录，将域名映射到一个 IPv4 地址。
2. AAAA记录: 将域名映射到一个 IPv6 地址。。
3. CNAME：即规范名字（Canonical Name）记录，将一个域名指向另一个“规范”域名, 简单说就是“别名”。
4. NS：域名服务器记录，指定负责某个域名或子域名的 DNS 服务器。
## 配置linux服务器(重点)
因为 centos 7 已经 EOF 了, 所以这边我选择使用Ubuntu 20.04 LTS（Focal Fossa）作为系统安装 nginx + V2ray。 \
首先先用ssh链接linux服务器, 我这边推荐开源的WindTerm, 介于之前windTerm官网被仿冒事件, 建议从github上下载[WindTerm](https://github.com/kingToolbox/WindTerm), 当然你也可以选择使用XShell, openSSH, tabby等替代方案。 \
输入用户名和密码登陆后, 就可以按照一下顺序在linux上进行配置了。 \
### 准备
我这边使用vim进行修改配置文件, 如果你的linux系统上没有vim的话, 可以使用下面的语句
```bash
sudo apt update
sudo apt install vim
```
检查vim版本情况:
```bash
vim --version
```
安装完成后，你就可以通过在终端中输入 vim 来启动Vim编辑器。
### 安装和配置v2ray
1. 确保你的 Ubuntu 系统是最新的:
```bash
sudo apt update && sudo apt upgrade -y
```
2. buntu 默认安装了 curl，如果没有安装，可以通过以下命令安装：
```bash
sudo apt install curl -y
```
3. 使用以下命令来动用官方脚本安装 V2Ray
```bash
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
```
4. 安装完成后查看是否完成安装:
```bash
/usr/local/bin/v2ray version
```
5. 修改v2ray配置文件(这里使用vim修改配置文件)
```bash
vim /usr/local/etc/v2ray/config.json
```
修改配置文件如下
```json
{
"inbound": {
    "protocol": "vmess",
    "listen": "127.0.0.1",
 "port": 8964,
 "settings": {"clients": [
        {"id": "#############"}
    ]},
 "streamSettings": {
 "network": "ws",
 "wsSettings": {"path": "/*******"}
    }
},

"outbound": {"protocol": "freedom"}
}
```
1. 其中`******`为一个随机的字符串, 如果自己选择难受可以用[这个网站生成](https://www.random.org/strings/?num=1&len=7&digits=on&upperalpha=on&loweralpha=on&unique=off&format=html&rnd=new)
2. 其中`######`为随机生成的uuid, 可以用[这个网站生成](https://1024tools.com/uuid)

### 安装和配置nginx
1. 安装nginx
```bash
sudo apt update
sudo apt install nginx
nginx -v
```
2. 配置nginx
```bash
vim /etc/nginx/nginx.conf
```
配置内容如下
```text
events {
    worker_connections 1024;
}
http {
server {
    ### 1:
    server_name &&&&&&&&&;

    listen 80;
    rewrite ^(.*) https://$server_name$1 permanent;
    if ($request_method  !~ ^(POST|GET)$) { return  501; }
    autoindex off;
    server_tokens off;
}

server {
    ### 2:
    ssl_certificate /etc/letsencrypt/live/&&&&&&&&&/fullchain.pem;

    ### 3:
    ssl_certificate_key /etc/letsencrypt/live/&&&&&&&&&&/privkey.pem;

    ### 4:
    location /********
    {
        proxy_pass http://127.0.0.1:8964;
        proxy_redirect off;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;

        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_requests 25600;
        keepalive_timeout 300 300;
        proxy_buffering off;
        proxy_buffer_size 8k;
    }

    listen 443 ssl http2;
    server_name www.byolio.buzz;
    charset utf-8;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!3DES:!MD5:!PSK;
    ssl_prefer_server_ciphers on;

    ssl_session_cache shared:SSL:60m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 10s;

    # Security settings
    if ($request_method  !~ ^(POST|GET)$) { return 501; }
    add_header X-Frame-Options DENY;
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options nosniff;
    add_header Strict-Transport-Security max-age=31536000 always;
    autoindex off;
    server_tokens off;

    index index.html index.htm index.php;
    root /usr/share/nginx/html;
    location ~ .*\.(js|jpg|JPG|jpeg|JPEG|css|bmp|gif|GIF|png)$ { access_log off; }
    }
}
```
1. `&&&&&` 这里换成带www的域名, 就是你申请的域名
2. `*****` 这里换成和v2ray完全相同的字符串

### 配置SSL证书
我这边使用python
```bash
apt install python3.10
pip3 install certbot
```
### 启动服务

### ufw使用

## 其他细节
1. 关于配置CDN隐藏IP : 在cloudflare域名管理界面点一下灰色的云，让颜色变成橙色即可。
2. iOS客户端使用全部要收费，常用有Shadowrocket（小火箭）等, 配置方法不做展开 。