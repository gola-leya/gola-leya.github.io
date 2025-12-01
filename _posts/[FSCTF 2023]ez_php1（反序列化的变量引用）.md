# [FSCTF 2023]ez_php1（反序列化的变量引用）

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed@main/20251117220757.png)

刚拿到这题想到是变量引用。

可以看见a已经被赋值能直接看flag，但是不会被输出，只有b会被输出。所以需要将b与a关联。

但是

一开始我只是将a赋值给b，也就是：

![image-20251117220956075](C:\Users\22295\AppData\Roaming\Typora\typora-user-images\image-20251117220956075.png)

传入data中发现没有回显flag，问了ai。

才知道原因：

- 反序列化时，`$a`和`$b`都被初始化为`null`（`N`）。
- 执行`__wakeup`：`$a`被更新为文件的 Base64 内容（此时`$b`仍然是之前的`null`，因为直接赋值是 “复制”，不会随`$a`的后续变化而更新）。
- 执行`__destruct`：输出`$b`，结果是`null`，无法得到文件内容。

简单来说就是赋值的顺序的问题，当魔术方法执行时a被赋值，b不会被赋值，最后b的值仍为null

所以单纯赋值没有用要用引用符号&

最后payload：

![image-20251117221217893](C:\Users\22295\AppData\Roaming\Typora\typora-user-images\image-20251117221217893.png)

**O:5:"Clazz":2:{s:1:"a";N;s:1:"b";R:2;}**