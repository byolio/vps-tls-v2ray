# 如何搭建v2ray服务器及其使用方法
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

6. 修改v2ray服务配置文件:
```bash
vim /etc/systemd/system/v2ray@.service
```
修改成如下配置
```text
[Unit]
Description=V2Ray Service
Documentation=https://www.v2fly.org/
After=network.target nss-lookup.target

[Service]
User=nobody
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
NoNewPrivileges=true
ExecStart=/usr/local/bin/v2ray run -config /usr/local/etc/v2ray/%i.json
Restart=on-failure
RestartPreventExitStatus=23

[Install]
WantedBy=multi-user.target
```
### 安装和配置nginx
1. 安装nginx
```bash
sudo apt update
sudo apt install nginx
nginx -v
```
1. 配置nginx
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
为了用真正的https流量翻墙, 我们的网站必须有合法的SSL证书。我用自动化工具Certbot申请证书, 只要把以下命令复制到命令窗口, 依次执行即可。
这里说的证书，申请完全是免费的，申请时需要邮箱地址(可以用匿名邮箱)。

1. 我这边使用python安装certbot, 当然你也可以使用官方推荐的snap去安装certbot
```bash
apt install python3.10
pip3 install certbot
```
2. 如果你的系统安装了防火墙, 请关闭它, 防止申请证书时报错, 如果没有防火墙且想要安装我之后会说
```bash
systemctl stop firewalld && systemctl disable firewalld
```
3. 申请SSL证书 这一步做个填空题，把这条命令里的域名和邮箱，换成你自己的信息。
```text
certbot certonly --standalone --agree-tos -n -d www.&&&&&& -d &&&&&& -m *******
```
    1. `&&&&&`不带www.的域名
    2. `*****`换成邮箱地址
4. 考虑到该证书只有3个月时效, 证书的自动化更新下面会讲
### 启动服务
1. 依次执行以下步骤启动 v2ray + nginx
```bash
sudo systemctl daemon-reload
sudo systemctl enable v2ray
sudo systemctl start v2ray
sudo systemctl enable nginx
sudo systemctl start nginx
```
2. 检查 v2ray + nginx 服务是否处于active(running)状态
```bash
sudo systemctl status v2ray
sudo systemctl status nginx
```
(如果发现运行状态为failed: 请使用`sudo journalctl -u v2ray.service -xe` 查看日志找出错误并修改错误)
3. 使用任务调度自动更新证书, 添加下述内容
```bash
0 0 1 */2 * service nginx stop; certbot renew; service nginx start;
```
此时服务配置完毕
### ufw使用(服务器未添加防火墙)
ufw和firewalld都是用于管理Linux系统防火墙的工具, 相比于firewalld的配置复杂ufw
体积小、资源占用低，配置简单, 特别适合VPS等环境。
1. 安装ufw
```bash
sudo apt update
sudo apt install ufw -y
```
2. 启动ufw
```bash
sudo ufw enable
```
3. 查看ufw状态
```bash
sudo ufw status
```
4. 开放ufw端口
```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```
5. 查看端口是否开放
```bash
sudo ufw status
```
配置防火墙完成
## 配置v2ray客户端
配置完服务端之后, 需要在windows上配置对应的[v2rayN客户端](https://github.com/2dust/v2rayN)
下好后配置步骤如下:
1. 点击服务器->添加vmess服务器
2. 编辑服务器:
   * 别名随便取一个
   * 地址填写你的vps公网ip
   * 端口填写443
   * 用户id需要与之前填写的uuid一致
   * 额外id填写0
   * 加密方式是: auto就行
   * 传输协议选ws
   * 伪装类型选none
   * 伪装域名写你买的域名: 要求有www.
   * path写之前的随机字符串前面添加/
   * 传输层安全选择tls
   * 内核要使用v2fly
3. 保存后开启Tun使用即可
## 其他细节(可选)
1. 关于配置CDN隐藏IP : 在cloudflare域名管理界面点一下灰色的云, 让颜色变成橙色即可。
2. iOS客户端使用全部要收费, 常用有Shadowrocket (小火箭) 等, 配置方法不做展开 。
3. 如果过程出现报错可以看看是否设置selinux为宽松模式:
```bash
sudo setenforce 0
sestatus
```
1. 关于443端口伪装逃避检查:
在nginx的root目录下通过xftp等工具将index.html换掉成你写的网页即可, 选正常一点的网页界面的可以有效防止被检测到。

1. 关于BBR加速:
BBR是谷歌开发的拥塞控制算法，可以降低延迟，加快访问速度。 \
如果linux内核版本大于4.10就可以用BBR了，把以下三条命令复制到命令窗口执行：
```bash
bash -c 'echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf'
bash -c 'echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf'
sysctl -p
```
然后运行以下命令，查看BBR是否启动成功：
```bash
sysctl net.ipv4.tcp_congestion_control
```
如果显示
```bash
net.ipv4.tcp_congestion_control = bbr
```
就表示成功启动了BBR加速。
## 总结
以上是v2ray服务器建立的全部过程, 不同环境下可能会稍有不同, 多试几遍总能成功, 祝你建出属于自己的v2ray服务器吧。
