# htb Oopsie 渗透笔记

核心思路：扫描端口-->IDOR拿到admin信息-->修改cookie-->文件上传配合反弹shell-->找到数据库配置文件-->连接ssh-->提权

![image-20260218151616865](C:\Users\22295\AppData\Roaming\Typora\typora-user-images\image-20260218151616865.png)

打开靶机
先用nmap扫描端口
![image-20260218151643298](C:\Users\22295\AppData\Roaming\Typora\typora-user-images\image-20260218151643298.png)

访问80端口网站
dirsearch先扫一下，没啥有用的

![image-20260218162952347](C:\Users\22295\AppData\Roaming\Typora\typora-user-images\image-20260218162952347.png)后面f12找到了可以路由/cdn-cgi/login
访问看到了登录页面
![image-20260218154006315](C:\Users\22295\AppData\Roaming\Typora\typora-user-images\image-20260218154006315.png)

尝试以游客身份登录
看到url里有id参数，改成1，直接就是admin了

![image-20260218154123066](C:\Users\22295\AppData\Roaming\Typora\typora-user-images\image-20260218154123066.png)

![image-20260218154315955](C:\Users\22295\AppData\Roaming\Typora\typora-user-images\image-20260218154315955.png)

然后根据htb提示cookie，更改cookie进入了uploads界面

![image-20260218153841180](C:\Users\22295\AppData\Roaming\Typora\typora-user-images\image-20260218153841180.png)

这里选择反弹shell，用的是kali下面自带的反弹shell文件，注意使用前要先修改默认ip和端口,改为自己虚拟机的ip地址(这里要额外注意，ip地址是vpn之后的ip，不能是192.168.xxx)

![image-20260218164201863](C:\Users\22295\AppData\Roaming\Typora\typora-user-images\image-20260218164201863.png)

![image-20260218162558729](C:\Users\22295\AppData\Roaming\Typora\typora-user-images\image-20260218162558729.png)

然后上传

根据前面dirsearch的扫描结果，知道上传路径是/uploads

上传之后访问一下，触发服务器解析文件

![image-20260218164629926](C:\Users\22295\AppData\Roaming\Typora\typora-user-images\image-20260218164629926.png)



成功连上，发现网站文件下的db.php文件下有用户robert和密码
尝试

```
ssh robert@10.129.95.191
```

成功

直接sudo被告诉没有权限。

执行下id，发现归属于bugtraker组

![image-20260218172002927](C:\Users\22295\AppData\Roaming\Typora\typora-user-images\image-20260218172002927.png)

看一下文件信息

```
find / -group bugtracker 2>/dev/null
```

查找改组下的所有文件，发现只有一个/usr/bin/bugtracker

![image-20260218172236975](C:\Users\22295\AppData\Roaming\Typora\typora-user-images\image-20260218172236975.png)

发现是一个suid文件。

执行一下它，发现调用了cat命令

![image-20260218191149248](C:\Users\22295\AppData\Roaming\Typora\typora-user-images\image-20260218191149248.png)

这里能直接获取root.txt文件，payload:  ../root.txt


但是如果要得到最高权限要通过 **使用PATH劫持提权** 的方法，这里就不接着说了。
