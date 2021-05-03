[博客园](https://www.cnblogs.com/jerry19880126/p/3308752.html)

### Item 1: View C++ as a federation of languages

首先C++，是一個语言集合

### Item 2: Prefer consts, enum s, and inlines to #defines

```c++
#define ASPECT_RATIO 1.653 
// 应改写为
const double AspectRatio = 1.653;
// 首先const是c++语言级的，其次#define 是单纯的字符替换，在多个文件中会被替换多次，这样就会copy多份
// 有时候在出错的时候，只会出现1.653，并不会出现ASPECT_RATIO

//对于一个类内的const成员，为了保证只有一个值，一般都会设置为static
class GamePlayer {
    private:
    static const int NumTurns = 5;
    int scores[NumTurns];
};

//也可以在类外进行赋值
class CostEstimate {
    private:
    static const double FudgeFactor;
};
const double CostEstimate::FudgeFactor = 1.35;

// 类内的常量是无法使用#define 进行定义的，因为#define 并没有作用于的概念，所以不能定义一个类的变量。
// 也可以使用enum进行完成，enum与#define 有一定的相似性，如无法取地址。（const变量可以）
class GamePlayer {
    private:
    enum { NumTurns = 5 };
    // "the enum hack" — makes
    // NumTurns a symbolic name for 5
    int scores[NumTurns];
};
// 关于宏的常用误区，就是用来写函数，如常见的最大最小
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
// 宏的表达式中是必须有括号的，负责可能会出错
// 且对于如下的操作有误
int a = 5, b = 0;
CALL_WITH_MAX( ++a, b);   // a 被加了两次
CALL_WITH_MAX( ++a, b+10);   // a 被加了一次
//这很明显是有问题的
实际上可以借助模板进行处理
template<typename T>
inline void callWithMax(const T& a, const T& b)
{
	f(a > b ? a : b);  // f是某一个函数
}
// 宏函数一个作用就是跳过函数跳转带来的性能损失，对于其他宏函数，我们可以使用inline进行替代


```

### Item 3: Use const whenever possible

C++ const 允许指定一个语义约束，编译器会强制实施这个约束，允许程序员告诉编译器某值是保持不变的。如果在编程中确实有某个值保持不变，就应该明确使用const，这样可以获得编译器的帮助。

const修饰指针变量时：（顶层与底层const）

　　(1)只有一个const，如果const位于*左侧，表示指针所指数据是常量，不能通过解引用修改该数据；指针本身是变量，可以指向其他的内存单元。

　　(2)只有一个const，如果const位于*右侧，表示指针本身是常量，不能指向其他内存地址；指针所指的数据可以通过解引用修改。

　　(3)两个const，*左右各一个，表示指针和指针所指数据都不能修改。

(2)值传递

 如果函数返回值采用“值传递方式”，由于函数会把返回值复制到外部临时的存储单元中，加const 修饰没有任何价值。所以，**对于值传递来说，加const没有太多意义。**

#### const 强大地方在于，const可以修饰参数，返回值，函数本身。

1. 函数返回一个const，通常可以降低一些错误。如

```c++
Class Rational;表示一个有理数
const Rational operator*( const Rational &lhs, const Rational &rhs);
Rational a,b,c;
(a*b) = c;//这是一个没有意义的操作。所以operator*必须返回一个const
```

const成员函数。两个理由使用const函数。

1. 可以知道哪个函数可以改变class资源，哪个不可以
2. 可以操作const对象。
3. 最重要的是两个函数如果只是函数const不同，可以被重载

```c++
class TextBlock {
    public:
        const char& operator[](std::size_t position) const
        { return text[position]; }
        char& operator[](std::size_t position)
        { return text[position]; }
    
    private:
   	 std::string text;
};
TextBlock tb("Hello");  
std::cout << tb[0];  //调用char& operator[](std::size_t position)
const TextBlock ctb("World");
std::cout << ctb[0];  //调用const char& operator[](std::size_t position) const
tb[0] = 'x';
ctb[0] = 'x'; // error ，因为ctb返回的是一个const，所以无法进行修改
 
```

对于如下const函数设计，是一个失败的

```c++
class CTextBlock {
public:
    char& operator[](std::size_t position) const
    { return pText[position]; }
private:
    char *pText;
};
const CTextBlock cctb("Hello");
char *pc = &cctb[0];
*pc = 'J';
// cctb是一个常对象，但是最后的值缺被修改了
```

此外注意non-constoperator[]的返回值一定是&。



* 避免non-const与const的重复

```c++
class TextBlock {
public:
...
const char& operator[](std::size_t position) const
{
... // do bounds checking  
... // log access data
... // verify data integrity
return text[position];
}
char& operator[](std::size_t position)
{
... // do bounds checking
... // log access data
... // verify data integrity
return text[position];
}
private:
std::string text;
};
//上面俩那个函数除了函数本身属性不同和返回值不同，其他一样，这就是代码重复。可以改为
class TextBlock {
public:
...
const char& operator[](std::size_t position) const
// same as before
{
...
...
...
return text[position];
}
char& operator[](std::size_t position)
{
	return const_cast <char&>(static_cast <const TextBlock&>(*this)[position]);
}
//注意到non-const调用了const。有两层转换，
//第一层将（*this）转为const对象，这样就可以调用const char& operator[](std::size_t position) const，然后将const char& operator[](std::size_t position) const的返回值去除const属性，即const_cast转换。
// 不可以用const调用non-const函数，因为non-const可能会对函数进行修改。
```



### Item 4: Make sure that objects are initialized before they're used

c++ 中局部变量会被初始化，其他变量不一定，可能是随机值，可能默认初始化，也可能没有定义。所以编程人员应该养成初始化的习惯

对应于成员变量初始化

```c++
class ABEntry {
public:
    ABEntry(const std::string& name, const std::string& address,
    const std::list<PhoneNumber>& phones);
private:
    std::string theName;
    std::string theAddress;
    std::list<PhoneNumber> thePhones;
    int num TimesConsulted;
};
ABEntry::ABEntry(const std::string& name, const std::string& address,
const std::list<PhoneNumber>& phones)
{
    theName = name;
     // these are all assignments,
    theAddress = address;
     // not initializations
    thePhones = phones
    numTimesConsulted = 0;
}
// 上面并不是一个很好的案例，因为其使用是赋值，（先初始化，再赋值）
// 作为c++编程人员应该写为
ABEntry::ABEntry(const std::string& name, const std::string& address,
const std::list<PhoneNumber>& phones)
: theName(name),theAddress(address),thePhones(phones),numTimesConsulted(0)
{}
// 因为成员的初始化是在进入构造函数之前完成的，函数体内是赋值。所以上面的构造函数直接进行的初始化，这样减少了程序的工作量（省去赋值操作，而赋值底层就是将原来的值抹掉然后指向新的值）
```

验证如下：

**第一种： 使用初始化列表。**

```c++
class Animal {
public:
    Animal() {
        std::cout << "Animal() is called" << std::endl;
    }

    Animal(const Animal &) {
        std::cout << "Animal (const Animal &) is called" << std::endl;
    }

    Animal &operator=(const Animal &) {
        std::cout << "Animal & operator=(const Animal &) is called" << std::endl;
        return *this;
    }

    ~Animal() {
        std::cout << "~Animal() is called" << std::endl;
    }
};

class Dog {
public:
    Dog(const Animal &animal) : __animal(animal) {
        std::cout << "Dog(const Animal &animal) is called" << std::endl;
    }

    ~Dog() {
        std::cout << "~Dog() is called" << std::endl;
    }

private:
    Animal __animal;
};

int main() {
    Animal animal;
    std::cout << std::endl;
    Dog d(animal);
    std::cout << std::endl;
    return 0;
}
```

运行结果：

```c++
Animal() is called

Animal (const Animal &) is called
Dog(const Animal &animal) is called

~Dog() is called
~Animal() is called
~Animal() is called
```

**第二种：构造函数赋值来初始化对象。**

构造函数修改如下：

```c++
Dog(const Animal &animal) {
    __animal = animal;
    std::cout << "Dog(const Animal &animal) is called" << std::endl;
}
```

此时输出结果：

```c++
Animal() is called

Animal() is called
Animal & operator=(const Animal &) is called
Dog(const Animal &animal) is called

~Dog() is called
~Animal() is called
~Animal() is called
```

通过上述我们得出如下结论：

- **类中包含其他自定义的class或者struct，采用初始化列表，实际上就是创建对象同时并初始化**
- **而采用类中赋值方式，等价于先定义对象，再进行赋值，一般会先调用默认构造，在调用=操作符重载函数。**

特别是引用数据成员，必须用初始化列表初始化，而不能通过赋值初始化！

继承关系：在继承关系中，子类必须初始化父类的构造函数。

```c++
class Animal {
public:
    Animal(int age) {
        std::cout << "Animal(int age) is called" << std::endl;
    }

    Animal(const Animal & animal) {
        std::cout << "Animal (const Animal &) is called" << std::endl;
    }

    Animal &operator=(const Animal & amimal) {
        std::cout << "Animal & operator=(const Animal &) is called" << std::endl;
        return *this;
    }

    ~Animal() {
        std::cout << "~Animal() is called" << std::endl;
    }
};

class Dog : Animal {
public:
    Dog(int age) : Animal(age) {
        std::cout << "Dog(int age) is called" << std::endl;
    }

    ~Dog() {
        std::cout << "~Dog() is called" << std::endl;
    }

};
```

初始化顺序，一般要和声明一致；

```c++
class X{
    int i;
    int j;
   Public:
    X(int val):j(val),i(j){}
}
// 报错，因为表面上是先用cal初始化j，然后用j初始化i，实际上先用j初始化i，此时i未定义！
```

跨文件编译的初始化顺序问题

静态变量：此时的静态指全局，在class外，或者class内被static的值。

`file1.cpp`

```c++
class FileSystem {
public:
	std::size_t numDisks() const;
};
extern FileSystem tfs;

```

`file2.cpp`

```c++
class Directory {
public:
    Directory::Directory( params )
    {
        std::size_t disks = tfs.numDisks();
    }
};
```

在这样的一个函数中，显然是两个不一样的文件，需要tfs的初始化在早与Directory，但是这是两个文件，所有初始化顺序没法保证。于是改为下面

​	

`file1.cpp`

```c++
class FileSystem {
	public:
		std::size_t numDisks() const;
};
FileSystem& tfs(){
    static FileSystem fs;
    return fs;
}
```

`file2.cpp`

```c++
class Directory {
	public:
        Directory::Directory( params ){
            std::size_t disks = tfs().numDisks(); // 这里不一样
        }
};
Directory& tempDir(){
    static Directory td;
    return td;
}

```

上面类似于单利设计模式；

### Item 5: Know what functions C++ silently writes and calls

首先编译器会默认生成构造函数，拷贝构造函数，拷贝赋值构造函数，析构函数，且是满足一定条件下的。

```c++
class Empty{};
//等同于
class Empty {
public:
    Empty() { ... }
    Empty(const Empty& rhs) { ... }
    ~Empty() { ... }
    Empty& operator=(const Empty& rhs) { ... }
};

template<typename T>
class NamedObject {
    public:
    NamedObject(const char *name, const T& value);
    NamedObject(const std::string& name, const T& value);
    private:
    std::string nameValue;
    T objectValue;
};
NamedObject<int> no1("Smallest Prime Number", 2);  
NamedObject<int> no2(no1);  // 默认拷贝构造函数
//但是
template<typename T>
class NamedObject {
    public:
    NamedObject(std::string& name, const T& value);
    private:
    std::string& nameValue;  // 引用
	const T objectValue;
};

std::string newDog("Persephone");
std::string oldDog("Satch");
NamedObject<int> p(newDog, 2);
NamedObject<int> s(oldDog, 36);
p = s;  //报错
因为nameValue是一个引用，无法进行=时的赋值，此时需要自定义赋值构造函数
```

### Item 6: Explicitly disallow the use of compiler-generated functions you do not want

```c++
class HomeForSale { ... };
HomeForSale h1;
HomeForSale h2;

HomeForSale h3(h1);
h1 = h2;
// 对于不想被赋值构造，也不想默认生成赋值，我们可以自定义一个构造函数，并设置为private，但是成员函数和友元函数可以调用，于是为了禁止他们使用，可以仅仅声明，而不去定义，这样当别的使用时，会有链接错误。


事实上为了简单会设计这样一个类
class Uncopyable {
    protected:
    Uncopyable() {}
    ~Uncopyable() {}

    private:
    Uncopyable(const Uncopyable&);
    Uncopyable& operator=(const Uncopyable&);
};
//然后让我们想要禁止构造的类继承这个，因为子类构造必须构造父类，此时父类没办法构造，所以子类也没办法构造。
```

对于c++11的语法可以只用`delete`

如：

```c++
HomeForSale() = delete;
```

### Item 7: Declare destructors virtual in polymorphic base classes

##### 如果子类的指针对象指向父类，并由父类delete，则是未定义行为，或者无法完成子类的析构

```c++
class Animal {
public:
    Animal(int age) {
        std::cout << "Animal(int age) is called" << std::endl;
    }
     ~Animal(){
    std::cout << "~Animal() is called" << std::endl;
    }
};

class Dog : public Animal {
public:
    Dog(int age) : Animal(age) {
        std::cout << "Dog(int age) is called" << std::endl;
    }
    ~Dog() {
        std::cout << "~Dog() is called" << std::endl;
    }
};

int main() {
    Animal *a = new Dog(1);
    delete a;
    return 0;
}
```

```shell
Animal(int age) is called
Dog(int age) is called
~Animal() is called
```

只有将子类虚构函数定义为virtual时才可以

如果在  ~Animal()前面加virtual，则结果为

```shell
Animal(int age) is called
Dog(int age) is called
~Dog() is called
~Animal() is called
```

* 结论： 如果一个类要用作多态的父类，应该将父类虚构函数定义为virtual

  

由于虚函数的实现中会有一个虚函数表(内存成本)，所以不应该盲目使用虚函数。



### Item 8: Prevent exceptions from leaving destructors

抛出异常时，将暂停当前函数的执行，开始查找匹配的catch子句。首先检查`throw`本身是否在`try`块内部，如果是，检查与该`try`相关的`catch`子句，看是否可以处理该异常。如果不能处理，就退出当前函数，并且释放当前函数的内存并销毁局部对象，继续到上层的调用函数中查找，直到找到一个可以处理该异常的`catch`。这个过程称为栈展开。如果不能找到就会调用terminate函数，这个函数会立即退出程序，此时甚至都没有对变量进行回收。所以应该防止terminate的调用。

有两种情况下会调用析构函数:
 1.在正常情况下删除一个对象，例如对象超出了作用域或被显式地`delete`
 2.异常传递的堆栈辗转开解（stack-unwinding）过程中，由异常处理系统删除一个对象。

```c++
class DBConnection {
    public:
    static DBConnection create();
        void close();  如果这个函数一定要最后调用，
}

// 	可以将它作为一个析构函数
class DBConn { // class to manage DBConnection
    public: // objects
    ~DBConn()
    {
        db.close();
    }
    private:
        DBConnection db;
};
/*
但是如果析构函数中调用异常，就会传播异常，使异常离开这个析构函数，这样会带来难以处理的问题。
于是可以使用try块
1.能够在异常传递的堆栈辗转开解（stack-unwinding）过程中，防止terminate被调用；
2.能帮助确保析构函数总能完成我们希望它能做的所有事情；
*/
DBConn::~DBConn()
{
    try { db.close(); }
    catch (...) {
   // 只要抛出异常就中置
    std::abort();
}
}


// 为了转移异常，重新设计异常处理
class DBConn {
    public:
    void close() // new function for
    { // client use
        db.close();
        closed = true;
    }
    ~DBConn()
    {
        if (!closed) {
            try {
            	db.close();
    			}
    		catch (...) 
    		{
    
    		}
    }
    private:
    DBConnection db;
    bool closed;
};
    
这里将可能发生异常的函数变为该成员函数，然后析构函数调用这个函数，就保证了函数异常是发生在
```

** 析构函数最好不要抛出异常，如果一个被析构函数调用的函数可能会抛出异常，则析构函数应该使用try块捕捉该函数

** 如果客户需要对某个在运行时抛出的异常做出反应，则应该提供一个函数，执行该操作。

[![DdUHE9.png](https://s3.ax1x.com/2020/11/25/DdUHE9.png)](https://imgchr.com/i/DdUHE9)



#### 这里有好几点不清晰，异常到底在处理什么？ 如客户端自己调用了其成员函数close（）后出错。那么析构函数也不会执行的。 ###

---

### Item 9: Never call virtual functions during construction or destruction

```c++
class Transaction {
public:
    Transaction();
// base class for all
// transactions
    virtual void logTransaction() ;
};
void Transaction::logTransaction() {

}
Transaction::Transaction()
{
    logTransaction();
}
class BuyTransaction: public Transaction {
public:
    virtual void logTransaction() const;
};

BuyTransaction b;
// 对于上面实例，创建b时会调用BuyTransaction的构造函数，而Transaction的构造函数中有一个虚函数logTransaction，此时就是调用基类的logTransaction，而不是父类的。很明显此时子类并没有完成初始化，所以并不能调用子类的logTransaction。

// 对于如下例子，即纯虚函数，首先Transaction会有一个调用了纯虚函数的非虚函数，这会在运行时出错。出于考量，基类定义纯虚函数，就是需要子类去调用的，父类的纯虚函数并没有多少意义，但是在父类构造阶段又无法调用子类，所以出错。
// TODO 思考为什么会这样
#include <iostream>
#include "main.h"
class Transaction {
public:
    Transaction()
    { init(); }
// call to non-virtual...
    virtual void logTransaction() const = 0;
private:
    void init()
    {
        logTransaction();
    }
};
void Transaction::logTransaction()const {}
class BuyTransaction : public Transaction{
    virtual void logTransaction() const{}

};


```

* Note： 尽量避免在构造和虚构函数中直接或者间接调用虚函数。
### Item 10: Have assig7nment operators return a reference to *this

```c++
如果我们需要对Widget类型重载=操作符号，应该最好返回引用
Widget& operator=(const Widget& rhs)
 // return type is a reference to
{
 // the current class
return *this;
 // return the left-hand object
}
};
// 这种操作符对于+=，-=,<,等等都是有效的。
// 此类主要是为了因对一些极端要求，如
(a = b ) = c;
// 此时 a= b 后的值要作为左值，就是说a=b的操作一定是一个左值，只能需要=的返回是一个左值。
// 这种要求对于重载<< ,>> 尤其重要。
// 因为stream，无法copy，只能引用，所以只能返回引用，否则出现`>>a>>b`就是错误
```

### Item 11: Handle assignment to self in operator=

```c++
class Bitmap {  };
class Widget {
private:
Bitmap
};

Widget&
Widget::operator=(const Widget& rhs)
 // unsafe impl. of operator=
{
delete pb;
 // stop using current bitmap
pb = new Bitmap(*rhs.pb);
 // start using a copy of rhs's bitmap
}
// 对于上面的自我赋值，如果rhs和db指向同一个内容，那么delete pb;后就会delete rhs，此时再进行赋值，就是不是期望的。；
// 改进1
Widget& Widget::operator=(const Widget& rhs)
{
if (this == &rhs) return *this;
delete pb;
pb = new Bitmap(*rhs.pb);
}
//这个改进了，具有自我赋值安全性。但假设在new Bitmap(*rhs.pb);中出现异常，则pd指向了一个delete后的空间，这样在后面处理或者delete中也是一个灾难
// 改进2
Widget& Widget::operator=(const Widget& rhs)
{
Bitmap *pOrig = pb;  // 先保留原始的指针
 // remember original pb
pb = new Bitmap(*rhs.pb);
 // make pb point to a copy of *pb
delete pOrig;
 // delete the original pb
}
// 这样即使 new Bitmap(*rhs.pb);出错，但是原来的指针内容也没有丢
// 考虑性能问题，可以改进为，
class Widget {
...
void swap(Widget& rhs); // 可以在Item 20看到
...
};
Widget& Widget::operator=( Widget rhs)
 // rhs is a copy of the object
{
 // passed in — note pass by val
swap(rhs);
 // swap *this's data with
// the copy's
return *this;
}

```

* Note: 要留心自我赋值的内容

### Item 12: Copy all parts of an object

* 应该拷贝所有值

```c++
class Date { ... };
 // for dates in time
void logCall(const std::string& funcName);
 // make a log entry
class Customer {
    public:
    Customer(const Customer& rhs);
    Customer& operator=(const Customer& rhs);
    private:
    std::string name;
    Date lastTransaction;
};
Customer::Customer(const Customer& rhs)
: name(rhs.name)
{
	logCall("Customer copy constructor");
}
Customer& Customer::operator=(const Customer& rhs)
{
logCall("Customer copy assignment operator");
name = rhs.name;
 return *this;
}
// 上面的拷贝构造函数并没有复制Date类型值
```

* 包括父类

考虑以上类发生继承时

```c++
class PriorityCustomer: public Customer {
 // a derived class
    public:
        PriorityCustomer(const PriorityCustomer& rhs);
        PriorityCustomer& operator=(const PriorityCustomer& rhs);
    private:
        int priority;
};
PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs)
: priority(rhs.priority)
{
logCall("PriorityCustomer copy constructor");
}
PriorityCustomer&
PriorityCustomer::operator=(const PriorityCustomer& rhs)
{
logCall("PriorityCustomer copy assignment operator");
priority = rhs.priority;
return *this;
}
// 当拷贝子类时，应该自己编写函数拷贝父类。
// 写对于父类的处理，只能通过其构造函数，因为数据一般都是私有，无法直接访问
// 如下
PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs)
:
 Customer(rhs),
 // invoke base class copy ctor
priority(rhs.priority)
{
logCall("PriorityCustomer copy constructor");
}
PriorityCustomer&
PriorityCustomer::operator=(const PriorityCustomer& rhs)
{
logCall("PriorityCustomer copy assignment operator");
Customer::operator=(rhs); //应该调用父类复制函数
 // assign base class parts
priority = rhs.priority;
return *this;
}

```



* 复制所有的成员值。包括其父类的成员。

* 不要在拷贝构造函数中调用构造函数，因为此时构造函数不一定存在

* 有时候拷贝构造与赋值构造很相似，但不建议在赋值构造中调用拷贝构造。可以再提供一个private函数如init, 然后在两个构造函数中使用（将相似部分写进init）

  ### Item 13: Use objects to manage resources.

  ```c++
  class Investment { ... };
  Investment* createInvestment();
  // return ptr to dynamically allocated
  // object in the Investment hierarchy;
  // the caller must delete it
  // (parameters omitted for simplicity)
  void f()
  {
  Investment *pInv = createInvestment(); // call factory function
  ... // use pInv
  delete pInv; // release object
  }
  //因为很可能...中有异常或者有return语句，使得函数没有执行delete而导致内存泄露
  ```
  
  此时的一个方法是：把资源放进对象中，当对象作用域消失时会自动调用其析构函数。
  
  ```c++
  void f()
  {
  ...
  std::tr1::shared_ptr<Investment>
  pInv(createInvestment());
  ...
  }
  ```
  
  

## Item 14: Think carefully about copying behavior in resource-managing classes.

```c++
void lock(Mutex *pm); // lock mutex pointed to by pm
void unlock(Mutex *pm); // unlock the mutex

// 建立一个class来管理Mutex
class Lock {
    public:
    explicit Lock(Mutex *pm)
    : mutexPtr(pm)
    { lock(mutexPtr); } // acquire resource
    ~Lock() { unlock(mutexPtr); } // release resource
    private:
    Mutex *mutexPtr;
};

//我们会这样使用
Mutex m;
...

{
    // create block to define critical section
    Lock ml(&m);   // lock the mutex
    ... // perform critical section operations
} // automatically unlock mutex at end of block
//但是当发生Mutex资源复制时候,如
Lock ml1(&m); // lock m
Lock ml2(ml1); // copy ml1 to ml2—what should happen here?

方法1：
    禁止复制行为，如private copying操作
方法2：   
    
class Lock {
  public:
    explicit Lock(Mutex *pm) // init shared_ptr with the Mutex
    : mutexPtr(pm, unlock) // to point to and the unlock func as the deleter 
    { 
    lock(mutexPtr.get());  // see Item 15 for info on "get"
    }
  private:
    std::tr1::shared_ptr<Mutex> mutexPtr;  // use shared_ptr instead of raw pointer
};
//这里不需要定义析构函数，因为析构函数会自动调用非静态成员的析构函数，于是mutexPtr的析构函数会调用unlock方法

```

## Item 15: Provide access to raw resources in resource-managing classes.

一些API使用原始的接口（指针）而不是资源管理类，所以应该提供对原始资源访问的接口。如shared_ptr中的get一样。

```c++
//如下FontHandle作为原始资源，Font对其进行了封装
FontHandle getFont();  // from C API—params omitted for simplicity
void releaseFont(FontHandle fh); // from the same C API RAII class
class Font {
   public:
    explicit Font(FontHandle fh) // acquire resource;
    : f(fh) // use pass-by-value, because the C API doesnewed
    {} 
    FontHandle get() const { return f; } // explicit conversion function
    ~Font() { releaseFont(f); } // release resource
   private:
    FontHandle f; // the raw font resource
};
//访问如下
void changeFontSize(FontHandle f, int newSize);
// from the C API
Font f(getFont());
int newFontSize;
changeFontSize(f.get() , newFontSize);
// 这是显式调用get方法，还有隐式的。但是显式更加安全，只是需要手动调用get方法
```

## Item 16: Use the same form in corresponding uses of new and delete .

```c++
std::string *stringPtr1 = new std::string;
std::string *stringPtr2 = new std::string[100];
delete stringPtr1;
delete [] stringPtr2;
```

TODO 

shared_ptr p(new int[10]); 默认调用delete删除器，而不是delete [] ，所有会资源泄露。事实上资源管理类中，应使用vector，string等数组，而非原始数组。

## Item 17: Store newed objects in smart pointers in standalone statements.

```c++
//以独立语句将newed的对象放入智能指针里面
//对于如下的两个函数
int priority();
void processWidget(std::tr1::shared_ptr<Widget> pw, int priority);
// 其有可能的一个调用方法是，但是这样有可能会造成资源浪费。
processWidget( std::shared_ptr<Widget>( new Widget ) , priority());
//在调用processWidget之前会准备实参，于是会先进行
Call priority.
Execute "new Widget".
Call the shared_ptr constructor
//这个顺序只能保证   Execute "new Widget".在Call the shared_ptr constructor的前面，其他不能保证
// 于是有可能会出现
1. Execute "new Widget".
2. Call priority.
3. Call the tr1::shared_ptr constructor.
//但是假设Call priority.异常，此时"new Widget".并没有被资源管理类进行包裹，于是资源浪费。
//于是可以改成
std::shared_ptr<Widget> pw(new Widget); // store newed object in a smart pointer in a  standalone statement
processWidget( pw , priority()); // this call won't leak    
```



### Item 18: Make interfaces easy to use correctly and hard to use incorrectly

要开发一个容易使用并且不易出错的接口，就要想客户会犯什么样的错，假设设计一个需要表示日期的class

```c++
class Date {
public:
Date(int month, int day, int year);
// 客户可能犯的错误：
//   1>，错误的参数顺序传入。
//   2>，非法的月份和日期    
    struct Day {
        explicit Day(int d):val(m) {} 
        int val;
    }
    struct Month {
        explicit Month(int m):val(d) {}
        int val;
    }
    struct Year {
        explicit Year(int y):val(d) {}
        int val;    
    }
    //可以将年月日包装为一个类。
    //此时日期构造函数为
    Date d(Month(3), Day(30), Year(1995));
    //同理也可以包装月份如下
class Month {
    public:
    static Month Jan() { return Month(1); } // functions returning all valid
    static Month Feb() { return Month(2); } // Month values; see below for
    ... // why these are functions, not
    static Month Dec() { return Month(12); } // objects
    ... // other member functions
    private:
    explicit Month(int m);
    // prevent creation of new
    // Month values
    ...
    // month-specific data
};
    
    
    
```

预防客户做错还有一个方法：限制客户做什么。通常可以通过const修饰完成

此外任何接口要求客户必须做什么，就有可能是设计错误。如

```c++
Investment* createInvestment(); // from Item 13; parameters omitted for simplicit
//这个指针必须被释放，那么有可能客户没有delete，或者删除多次。所以可以改为
std::shared_ptr<Investment> createInvestment();

```

### Item 19: Treat class design as type design

精心设计class

设计一个class，应该考虑以下

* 如何被创建和销毁

* 对象初始化和赋值有什么差别

* 如果对象构造的时候以值传递，该如何

* class的和法值，即数值检查

* class需要继承某一个class吗

* class与其他class之间有没有转换呢，可以写non-explicit参数的构造函数

* 这个class有什么函数和操作符

* 函数的private或者public等属性

* class的一般性怎么样，即真正需要的是class还是template

* 是否真的需要一个class，比如需要一个子类（考虑是否在父类上添加函数可以完成新需求）

### Item 20: Prefer pass-by-reference-to- const to pass-by-value

  考虑下面代码

```c++
class Person {
	public:
    Person(); // parameters omitted for simplicity
    virtual ~Person(); // see Item 7 for why this is virtual
    ...
    private:
    std::string name;
    std::string address;
};
class Student: public Person {
    public:
    Student();
    // parameters again omitted
    ~Student();
    ...
    private:
    std::string schoolName;
    std::string schoolAddress;
};
bool validateStudent( Student s);
Student plato;
bool platoIsOK = validateStudent(plato);
//当上面函数调用发生时，plato传递给validateStudent参数s（构造函数被调用），参数返回时又进行Student的析构。同时Student的两个成员函数也被构造，同时其父类及其的两个成员函数也被构造，至此一个Person拷贝构造，一个Student拷贝构造，四个string拷贝构造，最后对应其析构，这成本太高。可以改为：
bool validateStudent( const Student & s);
//引用传递也可以避免在多态中被切割，即将子类用值传递给父类对象，子类特性会被切割掉，只剩下父类特性。
// c++ 中内置类型，stl迭代器的传值和传递引用成本都差不多，
```

### Item 21: Don't try to return a reference when you must return an object

考虑下面例子，

```c++
class Rational {  //有理数
    public:
    Rational(int numerator = 0,
    int denominator = 1); // see Item 24 for why this ctor isn't declared explicit
    ...
    private:
    int n, d; // numerator and denominator
    friend const Rational operator*(const Rational& lhs, const Rational& rhs); // see Item 3 for why the return type is const
};

Rational a(1, 2); // a = 1/2
Rational b(3, 5); // b = 3/5
Rational c = a * b; // c should be 3/10
//1
const Rational& operator*(const Rational& lhs, const Rational& rhs)
{
    Rational result(lhs.n * rhs.n, lhs.d * rhs.d);
    return result;
}// 这里返回的是一个局部的引用，肯定是有问题的
//2
const Rational& operator*(const Rational& lhs, const Rational& rhs)
{
    Rational *result = new Rational(lhs.n * rhs.n, lhs.d * rhs.d);
    return *result;
} //上面的代码需要手动释放内存，且对于下面的代码，分配了两次内存，如何合理的释放。
Rational w, x, y, z;
w = x * y * z;
//3
const Rational& operator*(const Rational& lhs, const Rational& rhs)
{
    static Rational result;
    result = ... ;
    return result;
}
bool operator==(const Rational& lhs, const Rational& rhs);
Rational a, b, c, d;
...
if ((a * b) == (c * d))
{
    do whatever's appropriate when the products are equal;
} else
{
	do whatever's appropriate when they're not;
}
// 展开为if (operator==( operator*(a, b), operator*(c, d) ))，这是结果始终为true？
//所以结论是返回引用就是一个不合理的设计。正确应该返回value
inline const Rational operator*(const Rational& lhs, const Rational& rhs)
{
	return Rational(lhs.n * rhs.n, lhs.d * rhs.d);
}
```

### Item 22: Declare data members private

这是oop语言的通用，不仅可以进行精准权限设置，也可以封装。

```c++
class AccessLevels {
public:
    ...
        int getReadOnly() const { return readOnly; }
    void setReadWrite(int value) { readWrite = value; }
    int getReadWrite() const { return readWrite; }
    void setWriteOnly(int value) { writeOnly = value; }
    private:
    int noAccess; // no access to this int
    int readOnly; // read-only access to this int
    int readWrite; // read-write access to this int
    int writeOnly; // write-only access to this int
};
```

private以为着封装，引用private的数据改变后对代码影响比较小，只需要改变对应成员函数就可以了。public意味着没有封装，当public数据改变后，所以使用这个数据的代码都需要修改。

### Item 23: Prefer non-member non-friend functions to member functions

对于如下实例

```c++
class WebBrowser {
    public:
    ...
    void clearCache();
    void clearHistory();
    void removeCookies();
	void clearEverything();// calls clearCache, clearHistory, and removeCookies
};
//其中clearEverything也可以实现如下
void clearBrowser(WebBrowser& wb)
{
    wb.clearCache();
    wb.clearHistory();
    wb.removeCookies();
}
// 对于考虑点，面对对象需要数据及其相应操作函数应该绑定在一起，这样看应该是成员函数。
// 但是对于封装来说成员函数的封装性就比非成员函数的封装性低，这样看应该选择非成员函数。
//如果一些数据被封装，就比可见。越多的数据被封装，越少的人看到他，就有越大的弹性去改变他。
// 如果要在一个member函数，一个non-friend，non-member函数之间选择，应该选择non-friend，non-member函数，因为member函数可以访问private数据，enums，typedfs等，这样就降低了封装性。
//在意封装性而改的non-member并不意味这它不可以是另一个类的成员函数，事实上可以另clearBrowser为某个工具类的成员。现代c++的作法是让这个函数与这个class处于一个namespace中，如；
namespace WebBrowserStuff {
class WebBrowser { ... };
void clearBrowser(WebBrowser& wb);
...
}
```

### Item 24: Declare non-member functions when type conversions should apply to all parameters

考虑下面，

```c++
class Rational {
public:
Rational(int numerator = 0,
int denominator = 1);
// ctor is deliberately not explicit;
// allows implicit int-to-Rational
// conversions
int numerator() const; // accessors for numerator and
int denominator() const; // denominator — see Item 22
private:
...
};
```

对上面类添加运算函数，以支持*。

```c++
class Rational {
public:
...
const Rational operator*(const Rational& rhs) const;
};
Rational oneEighth(1, 8);
Rational oneHalf(1, 2);
Rational result = oneHalf * oneEighth; // fine
result = result * oneEighth; // fine
result = oneHalf * 2; // fine
result = 2 * oneHalf; // error!
```

如果要支持混合运算，就需要改为non-member函数

```c++
class Rational {
...
// contains no operator*
};
// 对于这个函数来说，不需要作为Rational的friend函数，因为Rational的public函数接口完全可以。
const Rational operator*(const Rational& lhs, const Rational& rhs)
{
	return Rational(lhs.numerator() * rhs.numerator(), lhs.denominator() * rhs.denominator());
}
```

### Item 25: Consider support for a non-throwing swap

```c++

namespace std {
    template<typename T> // typical implementation of std::swap;
    void swap(T& a, T& b) // swaps a's and b's values
    {
        T temp(a);
        a = b;
        b = temp;
    }
}
```

上面是最简单的std::swap, 当其对象为指针时，如pImpl时。

```c++
class WidgetImpl { // class for Widget data;
    public: // details are unimportant
    ...
    private:
    int a, b, c; // possibly lots of data —
    std::vector<double> v; // expensive to copy!
        ...
};
class Widget { // class using the pimpl idiom
    public:
    Widget(const Widget& rhs);
    Widget& operator=(const Widget& rhs) 
    // to copy a Widget, copy its WidgetImpl object. For details on implementing
    { 
    ...
    *pImpl = *(rhs.pImpl); // operator= in general, see Items 10, 11, and 12.
    ... 
    }
    ...
    private:
    WidgetImpl *pImpl;
};
// ----------------------------------------------------------------------------
//事实上，当我们对Widget对象进行swap的时候，只需要交换其WidgetImpl指针就可以了，于是，可以改swap
class Widget { // same as above, except for the addition of the swap mem func
    public:  
    ...
    void swap(Widget& other)
    {
        std::swap(pImpl, other.pImpl); // to swap Widgets, swap their pImpl pointers
    }
    ...
};
//下面 template<>表明他是std::swap的一个全特化版本，而<Widget>表示这一特化版本系针对“T是Widget设计的”,通常我们不能够改变std空间的任何东西，但是可以为标准templates如swap制造特化版本。
namespace std {
    template<> // revised specialization of  std::swap
    void swap<Widget>(Widget& a, Widget& b)
    {
        a.swap(b); // to swap Widgets, call their swap member function
    }
}
// ----------------------------------------------------------------------------
```

### Item 26: Postpone variable definitions as long as possible.

假设下面例子

```c++
void encrypt(std::string& s);
std::string encryptPassword(const std::string& password)
{
    using namespace std;
    string encrypted;
    if (password.length() < MinimumPasswordLength) {
    	throw logic_error("Password is too short");
    }
    
 	encrypted = encrypt(password);
    return encrypted;
}
// 延后encrypted的构造，并且降低一次构造函数。
std::string encryptPassword(const std::string& password)
{
    ... // check length
    std::string encrypted(password); 
    encrypt(encrypted);
    return encrypted;
}
```

### Item 27: Minimize casting.

考虑下面代码，

```c++
class Window {
    public:
    virtual void onResize() { ... }
    ...
};
class SpecialWindow: public Window {
    public:
    virtual void onResize() {
        static_cast<Window>(*this) .onResize();
    ... // do SpecialWindow specific stuff

    }
    ...
};
//上面代码初衷是将*this转换为Window，然后调用其onResize函数，然后再执行SpecialWindow的onResize专有的方法,但是事实上static_cast<Window>(*this)会得到(*this)的一个副本，然后在这个副本上调用onResize。假设这个onResize函数改变了父类的资源，但是因为是在副本上改变的，即不会传递到下面。所以这样的转化就是有问题的。事实上，子类调用父类需要如下，
virtual void onResize() {
     Window::onResize();
    ... // do SpecialWindow specific stuff

    }
```

应该尽量少使用dynamic_cast，因为这个成本是比较大的，之所以需要dynamic_cast，通常是一个“你认为的子类”需要调用子类方法，但是实际上你只有指向父类的指针或者引用。但是可以通过下面两点避免。

```c++
// 1. 使用容器储存指向父类的指针
class Window { ... };
class SpecialWindow: public Window {
public:
void blink();  // 只有子类有这个方法
...
};
typedef std::vector<std::shared_ptr<Window> > VPW;

VPW winPtrs;
for (VPW::iterator iter = winPtrs.begin(); iter != winPtrs.end(); ++iter) {
	if (SpecialWindow *psw = dynamic_cast <SpecialWindow*>(iter->get()))
		psw->blink();
}
// 事实上应该改为
typedef std::vector<std::shared_ptr< SpecialWindow> > VPSW ;
VPSW winPtrs;
for ( VPSW ::iterator iter = winPtrs.begin();  iter != winPtrs.end();  ++iter)
	(* iter ) ->blink();
// 但是这样就使得每个子类都需要对应的容器。此时可以改为
// ----------------------------------------------------------
// 2. 下面代码里面父类有一个空的blink方法，然后容器里面是父类类型
class Window {
public:
virtual void blink() {} // default impl is no-op;

};

class SpecialWindow: public Window {
public:
virtual void blink() { ... }; // in this class, blink
... // does something
};
typedef std::vector<std::shared_ptr<Window> > VPW; VPW winPtrs;

for (VPW::iterator iter = winPtrs.begin(); iter != winPtrs.end(); ++iter)
	(*iter)->blink();
// --------------------------------------------------
// 一定要避免的是一连串的dynamic_cast
class Window { ... };

// derived classes are defined here
typedef std::vector<std::tr1::shared_ptr<Window> > VPW;
VPW winPtrs;
for (VPW::iterator iter = winPtrs.begin(); iter != winPtrs.end(); ++iter)
{
if (SpecialWindow1 *psw1 = dynamic_cast <SpecialWindow1*>(iter->get())) { ... }
else if (SpecialWindow2 *psw2 = dynamic_cast <SpecialWindow2*>(iter->get())) { ... }
else if (SpecialWindow3 *psw3 = dynamic_cast <SpecialWindow3*>(iter->get())) { ... }
    ...
}
```

*note* 

1. 尽量避免转型，尤其是dynamic_cast

2. 如果转型是必须的，尽量将转型隐藏在函数后面

3. 使用c++ style 风格

### Item 28: Avoid returning "handles" to object internals.

用一个简单例子表示

```c++
class Student
{
private:
    int ID;
    string name;
public:
    string& GetName()
    {
        return name;
    }
};

int main()
{  
    Student s;
    s.GetName() = "Jerry";
    cout << s.GetName() << endl;
}
//如果给GetName加上const，
// ---------------------------------------------------------------
 const string& GetName()
 {
     return name;
 }
//但是还是会存在问题
const string& fun()
{
    return Student().GetName();
}

int main()
{
    string name = fun(); //name指向一个不存的对象的成员变量
}
// 因为name指向Student().GetName()的引用，而fun函数结束Student()就会被析构，所以此时的name是一个悬空的引用。所以应该避免引用或者指针指向class的内部。当然vectoe[]之类的还是必须的

```

### Item29: Strive for exception-safe code.

努力做到异常安全

```c++
class PrettyMenu {
    public:
    ...
    void changeBackground(std::istream& imgSrc); // change background image
    private:
    Mutex mutex; // mutex for this object
    Image *bgImage; // current background image
    int imageChanges; // # of times image has been changed
};
void PrettyMenu::changeBackground(std::istream& imgSrc)
{
    lock(&mutex); // acquire mutex (as in Item 14)
    delete bgImage; // get rid of old background
    ++imageChanges; // update image change count
    bgImage = new Image(imgSrc); // install new background
    unlock(&mutex); // release mutex
}
//考虑上面例子，如果new Image(imgSrc)发生异常，则会导致如法解锁，此时可以放置于Lock类中，实际上就是c++的std::lock_guard，
//------------------------------------------------------------------------------
class PrettyMenu {
std::shared_ptr<Image> bgImage;
};
void PrettyMenu::changeBackground(std::istream& imgSrc)
{
    Lock ml(&mutex);
    bgImage.reset(new Image(imgSrc));
    ++imageChanges;
}
// 上面代码改变的 ++imageChanges;的位置,使用了Lock类，也使用了智能指针，就不用删除旧图像了
```

带异常安全性的函数会提供三个保证之一：

1. 基本承诺：如果异常被抛出，程序内的任何事物仍然保持在有效状态下。没有任何对象或者数据结构会因此被破坏。比如上例中本次更换背景图失败，不会导致相关的数据发生破坏。

2. 强烈保证：在基本承诺的基础上，保证成功就是完全成功，失败也能回到之前的状态，不存在介于成功或失败之间的状态。

3. 不抛出异常：承诺这个代码在任何情况下都不会抛出异常，但这只适用于简单的语句。

强烈保证有一种实现方法，那就是copy and swap。原则就是：在修改这个对象之前，先创建它的一个副本，然后对这个副本进行操作，如果操作发生异常，那么异常只发生在这个副本之上，并不会影响对象本身，如果操作没有发生异常，再在最后进行一次swap。

```c++
struct PMImpl {
	std::shared_ptr<Image> bgImage; // Impl."; see below for
	int imageChanges; // why it's a struct
};
class PrettyMenu {
        ...
    private:
    Mutex mutex;
    std::tr1::shared_ptr<PMImpl> pImpl;
};
void PrettyMenu::changeBackground(std::istream& imgSrc)
{
    Lock ml(&mutex); // acquire the mutex
    std::shared_ptr<PMImpl> pNew(new PMImpl(*pImpl));
    pNew->bgImage.reset(new Image(imgSrc));
    ++pNew->imageChanges;
    std::swap(pImpl, pNew);
}

```

copy-and-swap策略关键在于“修改对象数据的副本，然后在一个不抛异常的函数中将修改后的数据和原件置换”。它确实提供了强异常安全保障，但代价是时间和空间，因为必须为每一个即将被改动的对象造出副本。另外，这种强异常安全保障，也会在下面的情况下遇到麻烦：

```c++
 void someFunc()
 {
     f1();
     f2();
 }
```

f1()和f2()都是强异常安全的，但万一f1()没有抛异常，但f2()抛了异常呢？是的，数据会回到f2()执行之前的状态，但程序员可能想要的是数据回复到f1()执行之前。要解决这个问题就需要将f1与f2内容进行融合，确定都没有问题了，才进行一次大的swap，这样的代价都是需要改变函数的结构，破坏了函数的模块性。如果不想这么做，只能放弃这个copy-and-swap方法，将强异常安全保障回退成基本保障。

类似于木桶效应，代码是强异常安全的，还是基本异常安全的，还是没有异常安全，取决于最低层次的那个模块。换言之，哪怕只有一个地方没有考虑到异常安全，整个代码都不是异常安全的。

最后总结一下：

1. 异常安全函数是指即使发生异常也不会泄漏资源或者允许任何数据结构败坏。这样的函数区分为三种可能的保证：基本型、强烈型、不抛异常型。

2. 强烈保证往往可以通过copy-and-swap实现出来，但“强烈保证”并非对所有函数都可以实现或具备现实意义。

3. 异常安全保证通常最高只等于其所调用之各个函数的异常安全保证中的最弱者。