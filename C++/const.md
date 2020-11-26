# 顶层const



指针是一个对象，指针本身类型和所指对象的类型是独立事件。

只有指针有顶层const，引用，或者其他类型没有（引用不是对象）

`代码清单1`  非常量类型的const修饰

 ```c++
int const a = 1;  // 等价于 const int a = 1 ；
const int &c = a;
 ```



指针如果添加const修饰符时有两种情况：



1 **指向常量的指针**：代表**不能改变其指向内容**的指针。声明时const可以放在类型名前后都可，（对于int类型来说，声明时：const int和int const 是等价的）。声明指向常量的指针也就是**底层const**，下面举一个例子：

```cpp
int num_a = 1;
int const  *p_a = &num_a; //底层const  等价于const int *p_a = &num_a;
//*p_a = 2;  //错误，指向“常量”的指针不能改变所指的对象
```



注意：指向“常量”的指针不代表它所指向的内容一定是常量，只是代表不能通过**解引用符**（操作符*）来改变它所指向的内容。上例中指针p_a指向的内容就不是常量，可以通过赋值语句：num_a=2;  来改变它所指向的内容。

2 **指针常量**：代表指针本身是常量，声明时必须初始化，之后**它存储的地址值就不能再改变**。声明时const必须放在指针符号*后面，即：\*const 。声明常量指针就是**顶层const**，下面举一个例子：

```cpp
int num_b = 2;
int *const p_b = &num_b; //顶层const
//p_b = &num_a;  //错误，常量指针不能改变存储的地址值
// 注意下
const int a = 2;
int *const b = &a; // error
//因为a是一个int类型，b是一个指向int的const指针，所以就需要将const int转化为int，这会报错。可以改为
const int *const b = &a;
```

## 区分顶层const和底层const的作用

为啥非要区分顶层const和底层const呢，根据C++primer的解释，区分后有两个作用。

1 执行对象拷贝时有限制，常量的底层const不能赋值给非常量的底层const。也就是说，你只要能正确区分顶层const和底层const，你就能避免这样的赋值错误。下面举一个例子：

```cpp
int num_c = 3;
const int *p_c = &num_c;  //p_c为底层const的指针
//int *p_d = p_c;  //错误，不能将底层const指针赋值给非底层const指针
const int *p_d = p_c; //正确，可以将底层const指针复制给底层const指针
```

2 使用命名的强制类型转换函数const_cast时，需要能够分辨底层const和顶层const，因为const_cast只能改变运算对象的底层const。下面举一个例子：

```cpp
int num_e = 4;
const int *p_e = &num_e;
//*p_e = 5;  //错误，不能改变底层const指针指向的内容
int *p_f = const_cast<int *>(p_e);  //正确，const_cast可以改变运算对象的底层const。但是使用时一定要知道num_e不是const的类型。
*p_f = 5;  //正确，非顶层const指针可以改变指向的内容
cout << num_e;  //输出5
```