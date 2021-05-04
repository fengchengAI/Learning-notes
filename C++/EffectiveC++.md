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

### Item 30: Understand the ins and outs of inlining.****

​	inline是内联的关键字，它可以建议编译器将函数的每一个调用都用函数本体替换。这是一种以空间换时间的做法。把每一次调用都用本体替换，无疑会使代码膨胀，(降低指令缓存的命中率)。但可以节省函数调用的成本，因为函数调用需要将之前的参数以堆栈的形式保存起来，调用结束后又要从堆栈中恢复那些参数。

​	但注意inline只是对编译器的一个建议，编译器并不表示一定会采纳，比如当一个函数内部包含对自身的递归调用时，inline就会被编译器所忽略。对于虚函数的inline，编译器也会将之忽略掉，因为内联（代码展开）发生在编译期，而虚函数的行为是在运行期决定的，所以编译器忽略掉对虚函数的inline。对于函数指针，当一个函数指针指向一个inline函数的时候，通过函数指针的调用也有可能不会被编译器处理成内联。

对于如下代码

```c++
inline void f() {...}
void (*pf)() = f;
f(); // this call will be inlined, because it's a "normal" call
pf();// this call probably won't be, because it's through a function pointer
```

​	另一方面，即使有些函数没有inline关键字，编译器也会将之内联，用本体替换调用代码，比如直接写在class内部的成员函数，如下

```c++
class Person
{
private:
    int age;
public:
    int getAge() const
    {
        return age;
    }
    void setAge(const int o_age);
};
void Person::setAge(const int o_age)
{
    age = o_age;
}
```

这里getAge()尽管没有inline关键字，但因为是直接写在class里面的，所以编译器将之处理成内联的；setAge()是在类内声明、类外定义的，编译器就不会将之处理成内联的了。

构造函数和析构函数虽然“看”似简单，但编译器会在背后做很多事情。因为c++对于“对象被创建和销毁时发生了什么事”做了一些保证。如果在构造期间发生异常，则会销毁已经构造好的数据。比如一个空的构造函数里面会由编译器写上对所有成员函数的初始化，如果将之inline，将会导致大批量的代码复制，所以不对构造函数和析构函数inline为好。如下所示：

```c++
class Base {
    public:
    private:
    std::string bm1, bm2;
};
class Derived: public Base {
    public:
    Derived() {}
    private:
    std::string dm1, dm2, dm3;
};
//实际上编译器会添加以下代码（例如Derived的构造函数，即使用户的Derived是一个空的函数体）
Derived::Derived()
{
   
    Base::Base();
    try { dm1.std::string::string(); }  //构造第一个参数
    catch (...) {
    Base::~Base();
    throw; // propagate the exception
    }
    try { dm2.std::string::string(); } //构造第二个参数
    catch(...) { // 销毁前一个
    dm1.std::string::~string();      // destroy dm1,
    Base::~Base(); // destroy base class part, and
    throw; // propagate the exception
    }
    try { dm3.std::string::string(); } //构造第三个参数
    catch(...) {  // 销毁前两个
    dm2.std::string::~string();   
    dm1.std::string::~string();
    Base::~Base();
    throw;
    }
}

```



​	要慎用inline，是因为一旦编译器真的将之inline了，那么这个inline函数一旦被修改，整个程序都需要重新编译，而如果这个函数不是inline的，那么只要重新连接就好。另外，一些调试器对inline函数的支持也是有限的。

​	作者认为，“一开始先不要将任何函数声明为inline”，经测试，确实发现某个函数的inline要比不对之inline的性能提升很多，才对之inline。在大多数情况下，inline并不是程序的瓶颈，真正的精力应该放在改善一些算法的修缮，以及反复调用的代码研究上，它们往往才是耗时的瓶颈所在。

### Item31: Minimize compilation dependencies between files.

在说这一条款之前，先要了解一下C/C++的编译知识，假设有三个类ComplexClass,  SimpleClass1和SimpleClass2，采用头文件将类的声明与类的实现分开，这样共对应于6个文件，分别是ComplexClass.h，ComplexClass.cpp，SimpleClass1.h，SimpleClass1.cpp，SimpleClass2.h，SimpleClass2.cpp。

ComplexClass复合两个BaseClass，SimpleClass1与SimpleClass2之间是独立的，ComplexClass的.h是这样写的：

```c++
#ifndef COMPLESS_CLASS_H
#define COMPLESS_CLASS_H

#include “SimpleClass1.h”
#include “SimpleClass2.h”

class ComplexClass
{
    SimpleClass1 xx;
    SimpleClass2 xxx;
};

#endif /* COMPLESS _CLASS_H */
```

我们来考虑以下几种情况：

***Case 1：\***

现在SimpleClass1.h发生了变化，比如添加了一个新的成员变量，那么没有疑问，SimpleClass1.cpp要重编，SimpleClass2因为与SimpleClass1是独立的，所以SimpleClass2是不需要重编的。

那么现在的问题是，ComplexClass需要重编吗？

答案是“是”，因为ComplexClass的头文件里面包含了SimpleClass1.h（使用了SimpleClass1作为成员对象的类），而且所有使用ComplexClass类的对象的文件，都需要重新编译！

如果把ComplexClass里面的#include  “SimpleClass1.h”给去掉，当然就不会重编ComplexClass了，但问题是也不能通过编译了，因为ComplexClass里面声明了SimpleClass1的对象xx。那如果把#include “SimpleClass1.h”换成类的声明class SimpleClass1，会怎么样呢？能通过编译吗？

答案是“否”，因为编译器需要知道ComplexClass成员变量SimpleClass1对象的大小(必须读取class定义才能知道对象大小)，而这些信息仅由class  SimpleClass1声明是不够的，但如果SimpleClass1作为一个函数的形参，或者是函数返回值，用class  SimpleClass1声明就够了。如：

```c++
 // ComplexClass.h
 class SimpleClass1;
 …
 SimpleClass1 GetSimpleClass1() const;
 …
```

但如果换成指针呢？像这样：

```c++
// ComplexClass.h
#include “SimpleClass2.h”

class SimpleClass1;

class ComplexClass:
{
    SimpleClass1* xx;
    SimpleClass2 xxx;
};
```

这样能通过编译吗？

 

答案是“是”，因为编译器视所有指针为一个字长（在32位机器上是4字节），因此class  SimpleClass1的声明是够用了。但如果要想使用SimpleClass1的方法，还是要包含SimpleClass1.h，但那是ComplexClass.cpp做的，因为ComplexClass.h只负责类变量和方法的声明。

 

那么还有一个问题，如果使用SimpleClass1*代替SimpleClass1后，SimpleClass1.h变了，ComplexClass需要重编吗？

先看Case2。

 

***Case 2：***

回到最初的假定上（成员变量不是指针），现在SimpleClass1.cpp发生了变化，比如改变了一个成员函数的实现逻辑（换了一种排序算法等），但SimpleClass1.h没有变，那么SimpleClass1一定会重编，SimpleClass2因为独立性不需要重编，那么现在的问题是，ComplexClass需要重编吗？

 

答案是“否”，因为编译器重编的条件是发现一个变量的类型或者大小跟之前的不一样了，但现在SimpleClass1的接口并没有任务变化，只是改变了实现的细节，所以编译器不会重编。

 

***Case 3：***

结合Case1和Case2，现在我们来看看下面的做法：

```c++

// ComplexClass.h
#include “SimpleClass2.h”

class SimpleClass1;

class ComplexClass
{
    SimpleClass1* xx;
    SimpleClass2 xxx;
};
```

```c++
// ComplexClass.cpp

void ComplexClass::Fun()
{
    SimpleClass1->FunMethod();
}
```

请问上面的ComplexClass.cpp能通过编译吗？

 

答案是“否”，因为这里用到了SimpleClass1的具体的方法，所以需要包含SimpleClass1的头文件，但这个包含的行为已经从ComplexClass里面拿掉了（换成了class SimpleClass1），所以不能通过编译。

 

如果解决这个问题呢？其实很简单，只要在ComplexClass.cpp里面加上#include  “SimpleClass1.h”就可以了。换言之，我们其实做的就是将ComplexClass.h的#include  “SimpleClass1.h”移至了ComplexClass1.cpp里面，而在原位置放置class SimpleClass1。

 

这样做是为了什么？假设这时候SimpleClass1.h发生了变化，会有怎样的结果呢？

SimpleClass1自身一定会重编，SimpleClass2当然还是不用重编的，ComplexClass.cpp因为包含了SimpleClass1.h，所以需要重编，但换来的好处就是所有用到ComplexClass的其他地方，它们所在的文件不用重编了！因为ComplexClass的头文件没有变化，接口没有改变！

 

总结一下，对于C++类而言，如果它的头文件变了，那么所有这个类的对象所在的文件都要重编，但如果它的实现文件（cpp文件）变了，而头文件没有变（对外的接口不变），那么所有这个类的对象所在的文件都不会因之而重编。

因此，避免大量依赖性编译的解决方案就是：在头文件中用class声明外来类，用指针或引用代替变量的声明；在cpp文件中包含外来类的头文件。

于是进入正文。对于如下的案例



```c++
#include <string>
#include "date.h"
#include "address.h"
class Person {
    public:
    Person(const std::string& name, const Date& birthday,
    const Address& addr);
    std::string name() const;
    std::string birthDate() const;
    std::string address() const;
    ...
    private:
    std::string theName;
    Date theBirthDate;
    Address theAddress;
};

```

 

两种方法解决文件间编译依赖，

1. 第一种是采用Handler Classes（用指针指向真正实现的方法），一句话，就是.h里面不包含类的自定义头文件，用“class  类名”的声明方式进行代替（也要把相应的成员变量替换成指针或引用的形式），在.cpp文件里面包含类的自定义头文件去实现具体的方法。改造之后的程序看起来是这样子的：

```c++
// Person.h
#include <string>
using namespace std;

class PersonImp;

class Person
{
private:
    //string Name;
    //MyDate Birthday;
    //MyAddress Address;
    PersonImp* MemberImp;

public:
    string GetName() const;
    string GetBirthday() const;
    string GetAddress() const;
    // follows functions
    // ...
};
```



```c++
// Person.cpp
#include "PersonImp.h"
#include "Person.h"

string Person::GetName() const
{
    return MemberImp->GetName();
}
string Person::GetBirthday() const
{
    return MemberImp->GetName();
}
string Person::GetAddress() const
{
    return MemberImp->GetAddress();
}
```

```c++
// PersonImp.h
#ifndef PersonImp_H
#define PersonImp_H

#include <string>
#include "MyAddress.h"
#include "MyDate.h"
using namespace std;

class PersonImp
{
private:
    string Name;
    MyAddress Address;
    MyDate Birthday;

public:
    string GetName() const
    {
        return Name;
    }

    string GetAddress() const
    {
        return Address.ToString();
    }

    string GetBirthday() const
    {
        return Birthday.ToString();
    }
};

#endif /* PersonImp_H */
```

这样，客户只需要使用Person.h的头文件就可以了，因为在Person.h头文件中，只声明了PersonImp*指针，而在Person.cpp中包含了PersonImp的定义，我们自己修改接口的时候，可以修改PersonIm.h就可以了

2. 下面来谈谈书中的第二部分，用Interface  Classes来降低编译的依赖。从上面也可以看出，避免重编的诀窍就是保持头文件（接口）不变化，而保持接口不变化的诀窍就是不在里面声明编译器需要知道大小的变量，Handler Classes的处理就是把变量换成变量的地址（指针），头文件只有class  xxx的声明，而在cpp里面才包含xxx的头文件。Interface  Classes则是利用继承关系和多态的特性，在父类里面只包含成员方法（成员函数），而没有成员变量，像这样：

```c++
// Person.h
#include <string>
using namespace std;

class MyAddress;
class MyDate;
class RealPerson;

class Person
{
public:
    virtual string GetName() const = 0;
    virtual string GetBirthday() const = 0;
    virtual string GetAddress() const = 0;
    virtual ~Person(){}
};
```

而这些方法的实现放在其子类中，像这样：

```c++
// RealPerson.h
#include "Person.h"
#include "MyAddress.h"
#include "MyDate.h"

class RealPerson: public Person
{
private:
    string Name;
    MyAddress Address;
    MyDate Birthday;
public:
    RealPerson(string name, const MyAddress& addr, const MyDate& date):Name(name), Address(addr), Birthday(date){}
    virtual string GetName() const;
    virtual string GetAddress() const;
    virtual string GetBirthday() const;
};
```

在RealPerson.cpp里面去实现GetName()等方法。从这里我们可以看到，只有子类里面才有成员变量，也就是说，如果Address的头文件变化了，那么子类一定会重编，所有用到子类头文件的文件也要重编，所以为了防止重编，应该尽量少用子类的对象。利用多态特性，我们可以使用父类的指针，像这样Person* p = new RealPerson(xxx)，然后p->GetName()实际上是调用了子类的GetName()方法。

但这样还有一个问题，就是new RealPerson()这句话一写，就需要RealPerson的构造函数，那么RealPerson的头文件就要暴露了，这样可不行。还是只能用Person的方法，所以我们在Person.h里面加上这个方法：

```c++
 // Person.h
 static Person* CreatePerson(string name, const MyAddress& addr, const MyDate& date);
```

注意这个方法是静态的（没有虚特性），它被父类和所有子类共有，可以在子类中去实现它：

```c++
// RealPerson.cpp
#include “Person.h”
Person* Person::CreatePerson(string name, const MyAddress& addr, const MyDate& date)
{
    return new RealPerson(name, addr, date);
}
```

这样在客户端代码里面，可以这样写：

```c++
// Main.h
 class MyAddress;
 class MyDate;
 void ProcessPerson(const string& name, const MyAddress& addr, const MyDate& date);
```

```c++
// Main.cpp
#include "Person.h"
#include “MyAddress.h”;
#include “MyDate.h”;

void ProcessPerson(const string& name, const MyAddress& addr, const MyDate& date)
{
    Person* p = Person::CreatePerson(name, addr, date);
…
}
```

请记住：

**1. 支持“编译依存性最小化”的一般构想是：相依于声明式，不要相依于定义式，基于此构想的两个手段是Handler classes和Interface classes。**

**2. 程序库头文件应该以“完全且仅有声明式”的形式存在，这种做法不论是否涉及templates都适用。**

### Item 32: Make sure public inheritance models "is-a."

这一条款是说的是公有继承的逻辑，如果使用继承，而且继承是公有继承的话，一定要确保子类是一种父类（is-a关系）。这种逻辑可能与生活中的常理不相符，比如企鹅是生蛋的，所有企鹅是鸟类的一种，直观来看，我们可以用公有继承描述：

```c++
class Bird
{
public:
    virtual void fly(){cout << "it can fly." << endl;}
};

class Penguin: public Bird
{
    // fly()被继承过来了，可以覆写一个企鹅的fly()方法，也可以直接用父类的
};

int main()
{
    Penguin p;
    p.fly(); // 问题是企鹅并不会飞！
}
```



但问题就来了，虽然企鹅是鸟，但鸟会飞的技能并不适用于企鹅，该怎么解决这个问题呢？方法一，在Penguin的fly()方法里面抛出异常，一旦调用了p.fly()，那么就会在运行时捕捉到这个异常。这个方法不怎么好，因为它要在运行时才发现问题。

方法二，去掉Bird的fly()方法，在中间加上一层FlyingBird类（有fly()方法）与NotFlyingBird类（没有fly()方法），然后让企鹅继承与NotFlyingBird类。这个方法也不好，因为会使注意力分散，继承的层次加深也会使代码难懂和难以维护。

方法三，保留所有Bird一定会有的共性（比如生蛋和孵化），去掉Bird的fly()方法，只在其他可以飞的鸟的子类里面单独写这个方法。这是一种比较好的选择，因为根本没有定义fly()方法，所以Penguin对象调用fly()会在编译期报错。

在艰难选择方法三之后，我们回过头来思考，就是在所有public继承的背后，一定要保证父类的所有特性子类都可以满足（父类能飞，子类一定可以飞），抽象起来说，就是在可以使用父类的地方，都一定可以使用子类去替换。

任何父类可以出现的地方，子类一定可以替代这个父类，只有当替换使软件功能不受影响时，父类才可以真正被复用。通俗地说，是“**子类可以扩展父类的功能，但不能改变父类原有的功能**”。

**“public继承”意味着is-a。适用于base classes身上的每一件事情一定也适用于derived classes身上，因为每一个derived class对象也都是一个base class对象。**



### Item 33: Avoid hiding inherited names

避免遮掩继承来的名字

名称的遮掩可以分成变量的遮掩与函数的遮掩两类，本质都是名字的查找方式导致的，当编译器要去查找一个名字时，它一旦找到一个相符的名字，就不会再往下去找了，因此遮掩本质上是优先查找哪个名字的问题。

而查找是分作用域的，虽然本条款的命名是打着“继承”的旗子来说的，但我觉得其实与继承并不是很有关系，关键是作用域。

举例子说明这个问题会比较好理解。

```c++
//例1：普通变量遮掩
int i = 3;

int main()
{
    int i = 4;
    cout << i << endl; // 输出4
}
```

这是一个局部变量遮掩全局变量的例子，编译器在查找名字时，优先查找的是局部变量名，找到了就不会再找，所以不会有warning，不会有error，只会是这个结果。

```c++
//例2：成员变量遮掩
class Base
{
public:
    int x;
    Base(int _x):x(_x){}
};

class Derived: public Base
{
public:
    int x;
    Derived(int _x):Base(_x),x(_x + 1){}
};

int main()
{
    Derived d(3);
    cout << d.x << endl; //输出4
}
```



因为定义的是子类的对象，所以会优先查找子类独有的作用域，这里已经找到了x，所以不会再查找父类的作用域，因此输出的是4，如果子类里没有另行声明x成员变量，那么才会去查找父类的作用域。那么这种情况下如果想要访问父类的x，怎么办呢？

可以在子类里面添加一个方法：

```c++
int GetBaseX() {return Base::x;}
```

利用Base::x，可以使查找指定为父类的作用域，这样就能返回父类的x的值了。

```c++
//例3：函数的遮掩
class Base
{
public:
    void CommonFunction(){cout << "Base::CommonFunction()" << endl;}
    void virtual VirtualFunction(){cout << "Base::VirturalFunction()" << endl;}
    void virtual PureVirtualFunction() = 0;
};

class Derived: public Base
{
public:
    void CommonFunction(){cout << "Derived::CommonFunction()" << endl;}
    void virtual VirtualFunction(){cout << "Derived::VirturalFunction()" << endl;}
    void virtual PureVirtualFunction(){cout << "Derived::PureVirtualFunction()" << endl;}
};

int main()
{
    Derived d;
    d.CommonFunction(); // Derived::CommonFunction()
    d.VirtualFunction(); // Derived::VirtualFunction()
    d.PureVirtualFunction(); // Derived::PureVirtualFunction()
    return 0;
}
```

与变量遮掩类似，函数名的查找也是先从子类独有的作用域开始查找的，一旦找到，就不再继续找下去了。这里无论是普通函数，虚函数，还是纯虚函数，结果都是输出子类的函数调用。

下面我们来一个难点的例子。

```c++
//例4：重载函数的遮掩
class Base
{
public:
    void CommonFunction(){cout << "Base::CommonFunction()" << endl;}
    void virtual VirtualFunction(){cout << "Base::VirturalFunction()" << endl;}
    void virtual VirtualFunction(int x){cout << "Base::VirtualFunction() With Parms" << endl;}
    void virtual PureVirtualFunction() = 0;
};

class Derived: public Base
{
public:
    void CommonFunction(){cout << "Derived::CommonFunction()" << endl;}
    void virtual VirtualFunction(){cout << "Derived::VirturalFunction()" << endl;}
    void virtual PureVirtualFunction(){cout << "Derived::PureVirtualFunction()" << endl;}
};

int main()
{
    Derived d;
    d.VirtualFunction(3); // ?
    return 0;
}
```

很多人都会认为输出的是**Base::VirtualFunction() With Parms**，实际上这段代码却是编译不过的。因为编译器在查找名字时，并没有“为重载而走的很远”，C++的确是支持重载的，编译器在发现函数重载时，会去寻找相同函数名中最为匹配的一个函数（从形参个数，形参类型两个方面考虑，与返回值没有关系），如果大家的匹配程度都差不多，那么编译器会报歧义的错。

但以上法则成立的条件是这些函数位于相同的作用域中，而这里是不同的域！编译器先查找子类独有的域，一旦发现了完全相同的函数名，它就已经不再往父类中找了！在核查函数参数时，发现了没有带整型形参，所以直接报编译错了。

如果去掉子类的VirualFunction()，那么才会找到父类的VirtualFunction(int)。

提醒一下，千万不要被前面的Virtual关键字所误导，你可以试一个普通函数，结果是一样的，只要子类中有同名函数，不管形参是什么，编译器都不会再往父类作用域里面找了。

 

好，如果现在你非要访问父类里面的方法，那也可以，书上给出了两种作法，一种是采用using声明，另一种是定义转交函数。



```c++
//例5：采用using声明，使查找范围扩大至父类指定的函数：
class Base
{
public:
    void CommonFunction(){cout << "Base::CommonFunction()" << endl;}
    void virtual VirtualFunction(){cout << "Base::VirturalFunction()" << endl;}
    void virtual VirtualFunction(int x){cout << "Base::VirtualFunction() With Parms" << endl;}
    void virtual PureVirtualFunction() = 0;
};

class Derived: public Base
{
public:
    using Base::VirtualFunction; // 第一级查找也要包括Base::VirtualFunction
    void CommonFunction(){cout << "Derived::CommonFunction()" << endl;}
    void virtual VirtualFunction(){cout << "Derived::VirturalFunction()" << endl;}
    void virtual PureVirtualFunction(){cout << "Derived::PureVirtualFunction()" << endl;}
};

int main()
{
    Derived d;
    d.VirtualFunction(3); // 这样没问题了，编译器会把父类作用域里面的函数名VirtualFunciton也纳入第一批查找范围，这样就能发现其实是父类的函数与main中的调用匹配得更好（因为有一个形参），这样会输出Base::VirtualFunction() With Parms
    return 0;
}
```



用了using，实际上是告诉编译器，把父类的那个函数也纳入第一批查找范围里面，这样就能发现匹配得更好的重载函数了。



```c++
//例6：使用转交函数强制指定父类的作用域
class Base
{
public:
    void CommonFunction(){cout << "Base::CommonFunction()" << endl;}
    void virtual VirtualFunction(){cout << "Base::VirturalFunction()" << endl;}
    void virtual VirtualFunction(int x){cout << "Base::VirtualFunction() With Parms" << endl;}
    void virtual PureVirtualFunction() = 0;
};

class Derived: public Base
{
public:
    using Base::VirtualFunction;
    void CommonFunction(){cout << "Derived::CommonFunction()" << endl;}
    void virtual VirtualFunction(){cout << "Derived::VirturalFunction()" << endl;}
    void virtual PureVirtualFunction(int x){cout << "Derived::PureVirtualFunction()" << endl;}
    void virtual VirtualFunction(int x){Base::VirtualFunction(x)};
};

int main()
{
    Derived d;
    d.VirtualFunction(3); // 输出Base::VirtualFunction() With Parms
    return 0;
}
```

采用这种做法，需要在子类中再定义一个带int参的同名函数，在这个函数里面用Base进行作用域的指定，从而调用到父类的同名函数。

快结尾了，这里还是声明一下，在不同作用域内定义相同的名字，无论是发生在变量还是函数身上，都是非常无聊也是不好的做法。除了考试试卷上，还是不要把名字遮掩问题带到任何地方，命个不同的名字真的有那么难吗？

 

最后总结一下：

**1. derived classses内的名称会遮掩base classes内的名称。在public继承下从来没有人希望如此。**

**2. 为了让被遮掩的名称再见天日，可使用using声明式或转交函数（forwarding functions）。**





### Item 34: Differentiate between inheritance of interface and inheritance of implementation

区分接口继承和实现继承

这个条款书上内容说的篇幅比较多，但其实思想并不复杂。只要能理解三句话即可，第一句话是：纯虚函数只继承接口；第二句话是：虚函数既继承接口，也提供了一份默认实现；第三句话是：普通函数既继承接口，也强制继承实现。这里假定讨论的成员函数都是public的。

 

这里回顾一下这三类函数，如下：



```c++
class BaseClass
{
public:
    void virtual PureVirtualFunction() = 0; // 纯虚函数
    void virtual ImpureVirtualFunction(); // 虚函数
    void CommonFunciton(); // 普通函数
};
```



纯虚函数有一个“等于0”的声明，具体实现一般放在派生中（但基类也可以有具体实现），所在的类（称之为虚基类）是不能定义对象的，派生类中仍然也可以不实现这个纯虚函数，交由派生类的派生类实现，总之直到有一个派生类将之实现，才可以由这个派生类定义出它的对象。

虚函数则必须有实现，否则会报链接错误。虚函数可以在基类和多个派生类中提供不同的版本，利用多态性质，在程序运行时动态决定执行哪一个版本的虚函数（机制是编译器生成的虚表）。virtual关键字在基类中必须显式指明，在派生类中不必指明，即使不写，也会被编译器认可为virtual函数，virtual函数存在的类可以定义实例对象。

普通函数则是将接口与实现都继承下来了，如果在派生类中重定义普通函数，将会出现名称的遮盖（见条款33），事实上，也是极不推荐在派生类中覆盖基类的普通函数的，如果真的要这样做，请一定要考虑是否该把基类的这个函数声明为虚函数或者纯虚函数。

 

下面是三类成员函数的应用：



```c++
class BaseClass
{
public:
    void virtual PureVirtualFunction() = 0; // 纯虚函数
    void virtual ImpureVirtualFunction(); // 虚函数
    void CommonFunciton(); // 普通函数
};
void BaseClass::PureVirtualFunction()
{
    cout << "Base PureVirtualFunction" << endl;
}
void BaseClass::ImpureVirtualFunction()
{
    cout << "Base ImpureVirtualFunciton" << endl;
}

class DerivedClass1: public BaseClass
{
    void PureVirtualFunction()
    {
        cout << "DerivedClass1 PureVirturalFunction Called" << endl;
    }
};

class DerivedClass2: public BaseClass
{
    void PureVirtualFunction()
    {
        cout << "DerivedClass2 PureVirturalFunction Called" << endl;
    }
};

int main()
{
    BaseClass *b1 = new DerivedClass1();
    BaseClass *b2 = new DerivedClass2();
    b1->PureVirtualFunction(); // 调用的是DerivedClass1版本的PureVirtualFunction
    b2->PureVirtualFunction(); // 调用的是DerivedClass2版本析PureVirtualFunction
    b1->BaseClass::PureVirtualFunction(); // 当然也可以调用BaseClass版本的PureVirtualFucntion
    return 0;
}
```



书上提倡用纯虚函数去替代虚函数，因为虚函数提供了一个默认的实现，如果派生类的想要的行为与这个虚函数不一致，而又恰好忘记去覆盖虚函数，就会出现问题。但纯虚函数不会，因为它从语法上限定派生类必须要去实现它，否则将无法定义派生类的对象。

同时，因为纯虚函数也是可以有默认实现的（但是它从语法上强调派生类必须重定义之，否则不能定义对象），所以完全可以替换虚函数。

 

普通函数所代表的意义是不变性凌驾与特异性，所以它绝不该在派生类中被重新定义。

 

在设计类成员函数时，一般既不要将所有函数都声明为non-virtual（普通函数），这会使得没有余裕空间进行特化工作；也一般不要将所有函数都声明为virtual（虚函数或纯虚函数），因为一般会有一些成员函数是基类就可以决定下来的，而被所有派生类所共用的。这个设计法则并不绝对，要视实际情况来定。

 

最后总结一下：

**1. 接口继承和实现继承不同。在public继承之下，derived class总是继承base class的接口；**

**2. pure virtual函数只具体指定接口继承；**

**3. impure virtual函数具体指定接口继承和缺省实现继承；**

**4. non-virutal函数具体指定接口继承以及强制性实现继承。**



## Item 35:



## Item 36: Never redefine an inherited non-virtual function

这个条款的内容很简单，见下面的示例：



```c++
class BaseClass
{
public:
    void NonVirtualFunction()
    {
        cout << "BaseClass::NonVirtualFunction" << endl;
    }
};

class DerivedClass: public BaseClass
{
public:
    void NonVirtualFunction()
    {
        cout << "DerivedClass::NonVirtualFunction" << endl;
    }
};

int main()
{
    DerivedClass d;
    BaseClass* bp = &d;
    DerivedClass* dp = &d;
    bp->NonVirtualFunction(); // 输出BaseClass::NonVirtualFunction
    dp->NonVirtualFunction(); // 输出DerivedClass::NonVirtualFunction
}
```

从输出结果可以看到一个有趣的现象，那就是两者都是通过相同的对象d调用成员函数NonVirutalFunction，但显示结果却不相同，这会给读者带来困惑。（如果NonVirtualFunction函数是虚函数，则调用结果都是DerivedClass::NonVirtualFunction, 既动态绑定）

现在这个现象的原因是在于BaseClass:NonVirutalFunction与DerivedClass:NonVirtualFunction都是静态绑定，(静态绑定就是这个值最后赋值的对象，动态绑定就是这个值创建时候的类型，如：Base *b = new Derive(), b的动态类型为Derive，静态类型为Base)所以调用的non-virtual函数都是各自定义的版本。

 

回顾下之前的条款，如果是public继承的话，那么：

1) 适用于BaseClass的行为一定适用于DerivedClass，因为每一个DerivedClass对象都是一个BaseClass对象；

2) 如果BaseClass里面有非虚函数，那么DerivedClass一定是既继承了接口，也继承了实现；

3) 子类里面的同名函数会掩盖父类的同名函数，这是由于搜索法则导致的。

 

如果DerivedClass重定义一个non-virtual函数，那么会违反上面列出的法则。以第一条为例，如果子类真的要重定义这个函数，那么说明父类的这个函数不能满足子类的要求，这就与每一个子类都是父类的原则矛盾了。

 

可以总结一下了，无论哪一个观点，结论都相同：



**任何情况下都不该重新定义一个继承而来的non-virtual函数。**

### Item 37: Never redefine a function's inherited default parameter value



先看下面的例子：

```c++
enum MyColor
{
    RED,
    GREEN,
    BLUE,
};

class Shape
{
public:
    void virtual Draw(MyColor color = RED) const = 0;
};

class Rectangle: public Shape
{
public:
    void Draw(MyColor color = GREEN) const
    {
        cout << "default color = " << color << endl;
    }
};

class Triangle : public Shape
{
public:
    void Draw(MyColor color = BLUE) const
    {
        cout << "default color = " << color << endl;
    }
};


int main()
{
    Shape *sr = new Rectangle();
    Shape *st = new Triangle();
    cout << "sr->Draw() = "; // ？
    sr->Draw();
    cout << "st->Draw() = "; // ？
    st->Draw();

    delete sr;
    delete st;
}
```



问号所在处的输出是什么？

要回答这个问题，需要回顾一下虚函数的知识，如果父类中存在有虚函数，那么编译器便会为之生成虚表与虚指针，在程序运行时，根据虚指针的指向，来决定调用哪个虚函数，这称之与动态绑定，与之相对的是静态绑定，静态绑定在编译期就决定了。

实现动态绑定的代价是比较大的，所以编译器在函数参数这部分，并没有采用动态绑定的方式，也就是说，默认的形参是静态绑定的，它是编译期就决定下来了。

 

我们看下这两行代码，分析一下：

```c++
 Shape *sr = new Rectangle();
 Shape *st = new Triangle();
```

sr的静态类型是Shape*，动态类型才是Rectangle*，类似地，st的静态类型是Shape*，动态类型是Triangle*。这里没有带参数，所以使用的是默认的形参，即为静态的Shape::Draw()里面的缺省值RED，所以两个问题所在处的输出值都是0。

正因为编译器并没有对形参采用动态绑定，所以如果对继承而来的虚函数使用不同的缺省值，将会给读者带来极大的困惑，试想一下下面两行代码：

```c++
 Shape *sr = new Rectangle(); // 默认值是RED
 Rectangle *rr = new Rectangle(); // 默认值是GREEN
```

如果一定要为虚函数采用默认值，那么只要在父类中设定就可以了，可以借用条款35所说的NVI方法，像下面这样：



```c++
class Shape
{
public:
    void DrawShape(MyColor color = RED)
    {
        Draw(color);
    }
private:
    virtual void Draw(MyColor color) const = 0
    {
        cout << "Shape::Draw" << endl;
    }
};

class Rectangle: public Shape
{
private:
    void Draw(MyColor color) const
    {
        cout << "Rectangle::Draw" << endl;
    }
};

class Triangle : public Shape
{
private:
    void Draw(MyColor color) const
    {
        cout << "Triangle::Draw" << endl;
    }
};


int main()
{
    Shape *sr = new Rectangle();
    Shape *st = new Triangle();
    cout << "sr->DrawRectangle() = "; // Rectangle::Draw
    sr->DrawShape();
    cout << "st->DrawTriangle() = "; // Triangle::Draw
    st->DrawShape();
    delete sr;
    delete st;
}
```



因为前面条款已经约定non-virtual函数不会被覆写，所以这样就不用担心在子类中出现定义不同缺省形参值的问题了。

最后总结一下：

**绝对不要重新定义一个继承而来的缺省参数值，因为缺省参数值都是静态绑定，而virtual函数——你唯一应该覆写的东西——却是动态绑定。**



### Item 38: Model "has-a" or "is-implemented-in-terms-of" through composition


### Item 39: Use private inheritance judiciously

### Item 40: Use multiple inheritance judiciously