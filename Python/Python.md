python * 特殊用法

一般用于函数参数中

`def fun(func, *args, **kwargs):`

args为一个可变长度的参数,这些参数传入函数中,会自动打包为tuple

kwargs 为一个可变参数的含参数名的一个参数,这些关键字参数在函数内部自动组装为一个dict

args 一定在kwargs 之前

```python
def h(a,*args,**kargs):
    print(a,args,kargs)

h(1,2,3,x=4,y=5)
```

*的另外一个用法，是形参的匹配形式，即应用于实参,代码如下

```python
test（a，b，*args）：

​    print a,b,args,

argument=(1,2,3,4,5,6,7,8)

test(\*argument) ==test(1,2,3,4,5,6,7,8)

此时会给a=1,b=2,args = (3,4,5,6,7,8)
```



`partial`

python内置偏函数,主要是固定函数参数:

如Int函数可以将一个字符串的数字转化为数字,默认为十进制,

int函数有base参数,可以转化为对应进制位

当有大量转化需求时,可以定义一个int2函数如下;

```python
def int2(x, base=2):
    return int(x, base)

# 但是也可以通过partial函数
int2 = functools.partial(int, base=2)
```

