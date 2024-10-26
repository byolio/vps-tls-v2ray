# 如何搭建v2ray服务器及其使用方法(还未完成)
## 前言
本文是搭建V2Ray服务器（ws+tls）的教程。除了搭建V2Ray+ws+tls的过程，还包括打开BBR加速，以及简单的防止检测。
随着GFW封杀力度的不断加大, 很多纯VMess都被封杀掉了。如果你打算自己搭建翻墙服务，强烈推荐V2Ray+ws+tls（CDN, BBR可选）一步到位。

本文不面向小白, 要求拥有一定的linux知识储备且会使用SSH连接服务器。如果你都满足, 那么看懂本教程就毫无压力。

本文不推荐使用一键脚本，很多一键脚本都存在安全隐患，轻则屏蔽掉几个网站，重则把你的服务器变成肉鸡（即黑客攻击别人电脑的跳板）。就算用脚本搭建服务器，脚本也无法自动完成本文中购买域名，配置域名解析等等。你恐怕还要亲自购买域名，亲自配置域名解析，亲自登录VPS执行脚本。本文中的方法只比一键脚本多出几步，但是可以大大降低安全风险。

自建V2Ray服务器首先要购买VPS（在国外的虚拟主机）和 域名 (用于tls证书分发)，为避免广告嫌疑，正文中不推荐VPS和域名购买网站, 需要大家自行购买。

V2Ray+ws+tls是目前最稳定的翻墙技术之一, V2Ray+ws+tls用真正的https流量翻墙, 在GFW看来，你的流量和不计其数的https流量没有任何区别。

## 域名解析(自行购买vps和域名后)
我这边使用的是cloudflare进行域名解析, 大家也可以使用DNSPod, 阿里云DNS, 百度云DNS, Amazon Route, Google Cloud DNS, GoDaddy等的DNS解析服务。
