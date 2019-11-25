python * 特殊用法

一般用于函数参数中

`def fun(func, *args, **kwargs):`

args为`可变参数`: 一个可变长度的参数,这些参数传入函数中,会自动打包为tuple

kwargs 为`关键字参数`:一个可变参数的含参数名的一个参数,这些关键字参数在函数内部自动组装为一个dict

args 一定在kwargs 之前

```python
def h(a,*args,**kargs):
    print(a,args,kargs)

h(1,2,3,x=4,y=5)
```

当需要限制kwargs的参数名时,需要用到`命名关键字参数`:如下

```python

def person(name, age, *, city, job):
    print(name, age, city, job)
    
    //*后面表示命名关键字参数的参数名字;
    //实例如下 person('Jack', 24, city='Beijing', job='Engineer')
def person(name, age, *args, city, job): //如命名关键字参数前有可变参数,则可以省略*
    print(name, age, city, job)  
```

*的另外一个用法，是形参的匹配形式(将列表变成可变形参或者关键字形参)，即应用于实参,代码如下

```python
test（a，b，*args）：

​    print a,b,args,

argument=(1,2,3,4,5,6,7,8)

test(*argument) ==test(1,2,3,4,5,6,7,8)

此时会给a=1,b=2,args = (3,4,5,6,7,8)
```



***

`dict`

dict的key必须是**不可变对象**。

这是因为dict根据key来计算value的存储位置，如果每次计算相同的key得出的结果不同，那dict内部就完全混乱了。这个通过key计算位置的算法称为哈希算法（Hash）。

```python
d = {'Michael': 95, 'Bob': 75, 'Tracy': 85}
```

d为字典,字典有唯一键值对,访问如下:

```Python
d['Michael'] #如果不存在会报错
d.get('Thomas', a) #如果不存在会返回a,默认为none
d.pop('Thomas')# 删除键值对
```

`set`

set和dict类似，也是一组key的集合，但不存储value。由于key不能重复，所以，在set中，没有重复的key。

需要提供一个list作为输入集合：

```Python
>>> s = set([1, 2, 3])
    s.add(4)
    s.remove(4)
```







对于一个可迭代对象,可以用for进行迭代

如字典a;

```python
for key in a: #输出键名
for key in a.values() #输出键
for k, v in a.items() #输出键名
```

`enumerate`可将list变成一个索引和值对,如

```python
>>> for i, value in enumerate(['A', 'B', 'C']):
...     print(i, value)
...
0 A
1 B
2 C
```

列表生成器

```python
L = [x * x for x in range(10)]
```

列表生成式

```python
l = (x * x for x in range(10))
next(l) #访问下一个元素
for i in l: #常用for遍历l

```

生成器是一个列表,生成式是一个函数,节省空间.



`迭代器`

可以直接作用于`for`循环的数据类型有以下几种：

一类是集合数据类型，如`list`、`tuple`、`dict`、`set`、`str`等；

一类是`generator`，包括生成器和带`yield`的generator function。

这些可以直接作用于`for`循环的对象统称为可迭代对象：`Iterable`。

可以使用`isinstance()`判断一个对象是否是`Iterable`对象：

凡是可作用于`for`循环的对象都是`Iterable`类型；

凡是可作用于`next()`函数的对象都是`Iterator`类型，它们表示一个惰性计算的序列；

集合数据类型如`list`、`dict`、`str`等是`Iterable`但不是`Iterator`，不过可以通过`iter()`函数获得一个`Iterator`对象。



`map`

`map()`函数接收两个参数，一个是函数，一个是`Iterable`，`map`将传入的函数依次作用到序列的每个元素，并把结果作为新的`Iterator`返回。

```python
>>> def f(x):
...     return x * x
...
>>> r = map(f, [1, 2, 3, 4, 5, 6, 7, 8, 9])
>>> list(r) #r为Iterator
[1, 4, 9, 16, 25, 36, 49, 64, 81]
```

​	`reduce`把一个函数作用在一个序列`[x1, x2, x3, ...]`上，这个函数必须接收两个参数，`reduce`把结果继续和序列的下一个元素做累积计算，其效果就是：

```python
# reduce(f, [x1, x2, x3, x4]) = f(f(f(x1, x2), x3), x4)
>>> from functools import reduce
>>> def fn(x, y):
...     return x * 10 + y
...
>>> reduce(fn, [1, 3, 5, 7, 9])
13579
```

`filter()`也接收一个函数和一个序列。和`map()`不同的是，`filter()`把传入的函数依次作用于每个元素，然后根据返回值是`True`还是`False`决定保留还是丢弃该元素。

```Python
def is_odd(n): # 判断奇偶
    return n % 2 == 1

list(filter(is_odd, [1, 2, 4, 5, 6, 9, 10, 15]))
```





***

`匿名函数`

```Python
lambda x: x * x
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





# multiprocessing

多线程



```

with Manager() as manager:
     index = 0
     l = manager.list()
     num = multiprocessing.cpu_count()
     p = Pool(num)
     #lock = Lock()
     for i in root.iterdir():
         for j in i.iterdir():
             '''
             index = index+1

             if index % num == 0:
                 p.close()
                 # 进程池对象调用join，会等待进程吃中所有的子进程结束完毕再去结束父进程
                 p.join()
                 p = Pool(num)
             else:
             '''
             p.apply_async(run, args=(i.name, j.stem))
     p.close()
     # 进程池对象调用join，会等待进程吃中所有的子进程结束完毕再去结束父进程
     p.join()
```