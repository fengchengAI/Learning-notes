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
std::cout << tb[0];
const TextBlock ctb("World");
std::cout << ctb[0];
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
Customer::operator=(rhs);
 // assign base class parts
priority = rhs.priority;
return *this;
}

```

* 有时候拷贝构造与赋值构造很相似，但不建议在赋值构造中调用拷贝构造。可以再提供一个private函数如init, 然后在两个构造函数中使用（将相似部分写进init）

  ### 上面这个解释是，在赋值构造中，拷贝构造不一定存在？？？ Why

  