

### 指针与数组

C++指针是最重要的一个概念：

**数组指针（也称行指针）**

```c++
int (*p)[n];
```
 ()优先级高，首先说明p是一个指针，指向一个整型的一维数组，这个一维数组的长度是n，也可以说是p的步长。也就是说执行p+1时，p要跨过n个整型数据的长度。所以数组指针也称指向一维数组的指针，亦称行指针。如：

```c++
int a[m][n] = {...};
int (*p)[n]; //数组指针
p = a;
```

这里的n一定要和a的第二个维度一样，因为p相当于一个行指针，p+1是要指向下一行，即跳过一行的元素，就是n列．以下是相等的．

```c++
a==a[0]==&a[0][0]==p==p[0]; //都是a的首元素地址；
p[1]==a[1]==&a[1][0];
```

`注意`这里的p[1]是一个单纯的地址，不能通过一维的指针下标来访问．只能通过*访问具体的单值，而不是n长度的值．

对于一维数组;

```c++
int a[n];
int *p = a;
```

则以下都是相等的；

```c++
a[i] == p[i] == *(a+i) == *(p+i)//都是具体的值[]
```



**指针数组**
```c++
int *p[n];
```
`注意`;p!=p[0]实际上实验结果显示，p的值是p[n-1]的下一个地址(p[n-1]是p的最后一个值)

 []优先级高，先与p结合成为一个数组，再由int \*说明这是一个整型指针数组，它有n个指针类型的数组元素。这里执行p+1时，则p指向下一个数组元素，这样赋值是错误的：p=a（即不能直接对p进行赋值）；因为p是个不可知的表示，`只存在`   p[0]、p[1]、p[2]...p[n-1],而且它们分别是指针变量可以用来存放变量地址。但可以这样 \*p=a; 这里\*p表示指针数组第一个元素的值，a的首地址的值。
 如要将二维数组赋给一指针数组:

```c++
nt *p[3];
int a[3][4];
p++; //该语句表示p数组指向下一个数组元素。注：此数组每一个元素都是一个指针
for(i=0;i<3;i++)
p[i]=a[i]
//这里int *p[3] 表示一个一维数组内存放着三个指针变量，分别是p[0]、p[1]、p[2]
```



#### 多线程

引自[StormZhu](https://www.jianshu.com/u/a549acfa2f33)



`std::thread`类的构造函数是使用**可变参数模板**实现的，也就是说，可以传递任意个参数，第一个参数是线程的入口**函数**，而后面的若干个参数是该函数的**参数**。

第一参数的类型并不是`c`语言中的函数指针（`c`语言传递函数都是使用函数指针），在`c++11`中，增加了**可调用对象(Callable Objects)**的概念，总的来说，可调用对象可以是以下几种情况：

- 函数指针
- 重载了`operator()`运算符的类对象，即仿函数
- `lambda`表达式（匿名函数）
- `std::function`

线程之间只能move,不能copy,且只能给空线程move,如刚声明的线程.

```cpp
std::thread t3(function_3, 1, "hello"); //即是function_3中的int和string参数是引用传递,也是无法修改1,"hello"的值,事实上线程内也是一个副本
std::thread t1(f, std::ref(m));  // 需要使用std::ref()才能传值
t1.join();//每个线程都必须制定线程启动方式,join
t1.detch();
```



## 竞争条件

并发代码中最常见的错误之一就是**竞争条件(race condition)**。而其中最常见的就是**数据竞争(data race)**，从整体上来看，所有线程之间共享数据的问题，都是修改数据导致的，如果所有的共享数据都是只读的，就不会发生问题。

而`c++`中常见的`cout`就是一个共享资源，如果在多个线程同时执行`cout`，你会发发现很奇怪的问题：

对于cout<<A<<B<<endl;

如果两个线程访问上面语句,很容易发生线程1输出一半,线程2开始输出.



## 使用互斥元保护共享数据mutex

解决办法就是要对`cout`这个共享资源进行保护。在`c++`中，可以使用互斥锁`std::mutex`进行资源保护，头文件是`#include <mutex>`，共有两种操作：**锁定(lock)**与**解锁(unlock)**。将`cout`重新封装成一个线程安全的函数：



```cpp
#include <iostream>
#include <thread>
#include <string>
#include <mutex>
using namespace std;

std::mutex mu;
// 使用锁保护
void shared_print(string msg, int id) {
    mu.lock(); // 上锁
    cout << msg << id << endl;
    mu.unlock(); // 解锁
}

void function_1() {
    for(int i=0; i>-100; i--)
        shared_print(string("From t1: "), i);
}

int main()
{
    std::thread t1(function_1);

    for(int i=0; i<100; i++)
        shared_print(string("From main: "), i);
    t1.join();
    return 0;
}
```

修改完之后，运行可以发现打印没有问题了。但是还有一个隐藏着的问题，如果`mu.lock()`和`mu.unlock()`之间的语句发生了异常，会发生什么？`unlock()`语句没有机会执行！导致导致`mu`一直处于锁着的状态，其他使用`shared_print()`函数的线程就会阻塞。

解决这个问题也很简单，使用`c++`中常见的`RAII`技术，即**获取资源即初始化(Resource Acquisition Is Initialization)**技术，这是`c++`中管理资源的常用方式。简单的说就是在类的构造函数中创建资源，在析构函数中释放资源，因为就算发生了异常，`c++`也能保证类的析构函数能够执行。我们不需要自己写个类包装`mutex`，`c++`库已经提供了`std::lock_guard`类模板，使用方法如下：

```cpp
void shared_print(string msg, int id) {
    //构造的时候帮忙上锁，析构的时候释放锁
    std::lock_guard<std::mutex> guard(mu);
    //mu.lock(); // 上锁
    cout << msg << id << endl;
    //mu.unlock(); // 解锁
}
```

## 为保护共享数据精心组织代码

上面的`std::mutex`互斥元是个全局变量，他是为`shared_print()`准备的，这个时候，我们最好将他们绑定在一起，比如说，可以封装成一个类。由于`cout`是个全局共享的变量，没法完全封装，就算你封装了，外面还是能够使用`cout`，并且不用通过锁。下面使用文件流举例：



```cpp
#include <iostream>
#include <thread>
#include <string>
#include <mutex>
#include <fstream>
using namespace std;

std::mutex mu;
class LogFile {
    std::mutex m_mutex;
    ofstream f;
public:
    LogFile() {
        f.open("log.txt");
    }
    ~LogFile() {
        f.close();
    }
    void shared_print(string msg, int id) {
        std::lock_guard<std::mutex> guard(mu);
        f << msg << id << endl;
    }
};

void function_1(LogFile& log) {
    for(int i=0; i>-100; i--)
        log.shared_print(string("From t1: "), i);
}

int main()
{
    LogFile log;
    std::thread t1(function_1, std::ref(log));

    for(int i=0; i<100; i++)
        log.shared_print(string("From main: "), i);

    t1.join();
    return 0;
}
```



上面的`LogFile`类封装了一个`mutex`和一个`ofstream`对象，然后`shared_print`函数在`mutex`的保护下，是线程安全的。使用的时候，先定义一个`LogFile`的实例`log`，主线程中直接使用，子线程中通过引用传递过去（也可以使用单例来实现）,这样就能保证资源被互斥锁保护着，外面没办法使用但是使用资源。

但是这个时候还是得小心了！用互斥元保护数据并不只是像上面那样保护每个函数，就能够完全的保证线程安全，如果将资源的指针或者引用不小心传递出来了，所有的保护都白费了！要记住一下两点：

不要提供函数让用户获取资源。

不要资源传递给用户的函数。

```cpp
std::mutex mu;
class LogFile {
    std::mutex m_mutex;
    ofstream f;
public:
    LogFile() {
        f.open("log.txt");
    }
    ~LogFile() {
        f.close();
    }
    void shared_print(string msg, int id) {
        std::lock_guard<std::mutex> guard(mu);
        f << msg << id << endl;
    }
    // Never return f to the outside world
    ofstream& getStream() {
        return f;  //never do this !!!
    }
       // Never pass f as an argument to user provided function
    void process(void fun(ostream&)) {
        fun(f);
    }
};
```



如果我们想要从线程中返回异步任务结果，一般需要依靠**全局变量**；从安全角度看，有些不妥,所以需要一系列加锁等操作；为此C++11提供了std::future类模板，future对象提供访问异步操作结果的机制，很轻松解决从异步任务中返回结果。



## 死锁

如果你将某个`mutex`上锁了，却一直不释放，另一个线程访问该锁保护的资源的时候，就会发生死锁，这种情况下使用`lock_guard`可以保证析构的时候能够释放锁，然而，当一个操作需要使用两个互斥元的时候，仅仅使用`lock_guard`并不能保证不会发生死锁，

```cpp
#include <iostream>
#include <thread>
#include <string>
#include <mutex>
#include <fstream>
using namespace std;

class LogFile {
    std::mutex _mu;
    std::mutex _mu2;
    ofstream f;
public:
    LogFile() {
        f.open("log.txt");
    }
    ~LogFile() {
        f.close();
    }
    void shared_print(string msg, int id) {
        std::lock_guard<std::mutex> guard(_mu);
        std::lock_guard<std::mutex> guard2(_mu2);
        f << msg << id << endl;
        cout << msg << id << endl;
    }
    void shared_print2(string msg, int id) {
        std::lock_guard<std::mutex> guard(_mu2);
        std::lock_guard<std::mutex> guard2(_mu);
        f << msg << id << endl;
        cout << msg << id << endl;
    }
};

void function_1(LogFile& log) {
    for(int i=0; i>-100; i--)
        log.shared_print2(string("From t1: "), i);
}

int main()
{
    LogFile log;
    std::thread t1(function_1, std::ref(log));

    for(int i=0; i<100; i++)
        log.shared_print(string("From main: "), i);

    t1.join();
    return 0;
}
```



运行之后，你会发现程序会卡住，这就是发生死锁了。程序运行可能会发生类似下面的情况：

```cpp
Thread A              Thread B
_mu.lock()          _mu2.lock()
   //死锁               //死锁
_mu2.lock()         _mu.lock()
```

这种错误大都是加锁顺序问题,可以使用`c++`标准库中提供了`std::lock()`函数，能够保证将多个互斥锁同时上锁，同时，`lock_guard`也需要做修改，因为互斥锁已经被上锁了，那么`lock_guard`构造的时候不应该上锁，只是需要在析构的时候释放锁就行了，使用`std::adopt_lock`表示无需上锁：

```cpp
#include <iostream>
#include <thread>
#include <string>
#include <mutex>
#include <fstream>
using namespace std;

class LogFile {
    std::mutex _mu;
    std::mutex _mu2;
    ofstream f;
public:
    LogFile() {
        f.open("log.txt");
    }
    ~LogFile() {
        f.close();
    }
    void shared_print(string msg, int id) {
        std::lock(_mu, _mu2);
        std::lock_guard<std::mutex> guard(_mu, std::adopt_lock);
        std::lock_guard<std::mutex> guard2(_mu2, std::adopt_lock);
        f << msg << id << endl;
        cout << msg << id << endl;
    }
    void shared_print2(string msg, int id) {
        std::lock(_mu, _mu2);
        std::lock_guard<std::mutex> guard(_mu2, std::adopt_lock);
        std::lock_guard<std::mutex> guard2(_mu, std::adopt_lock);
        f << msg << id << endl;
        cout << msg << id << endl;
    }
};

void function_1(LogFile& log) {
    for(int i=0; i>-100; i--)
        log.shared_print2(string("From t1: "), i);
}

int main()
{
    LogFile log;
    std::thread t1(function_1, std::ref(log));

    for(int i=0; i<100; i++)
        log.shared_print(string("From main: "), i);

    t1.join();
    return 0;
}
```

### 细粒度加锁

互斥锁保证了线程间的同步，但是却将并行操作变成了串行操作，这对性能有很大的影响，所以我们要尽可能的**减小锁定的区域**，也就是使用**细粒度锁**。

这一点`lock_guard`做的不好，不够灵活，`lock_guard`只能保证在析构的时候执行解锁操作，`lock_guard`本身并没有提供加锁和解锁的接口，但是有些时候会有这种需求。看下面的例子。

```cpp
class LogFile {
    std::mutex _mu;
    ofstream f;
public:
    LogFile() {
        f.open("log.txt");
    }
    ~LogFile() {
        f.close();
    }
    void shared_print(string msg, int id) {
        {
            std::lock_guard<std::mutex> guard(_mu);
            //do something 1
        }
        //do something 2
        {
            std::lock_guard<std::mutex> guard(_mu);
            // do something 3
            f << msg << id << endl;
            cout << msg << id << endl;
        }
    }

};
```

上面的代码中，一个函数内部有两段代码需要进行保护，这个时候使用`lock_guard`就需要创建两个局部对象来管理同一个互斥锁（其实也可以只创建一个，但是锁的力度太大，效率不行），修改方法是使用`unique_lock`。它提供了`lock()`和`unlock()`接口，能记录现在处于上锁还是没上锁状态，在析构的时候，会根据当前状态来决定是否要进行解锁（`lock_guard`就一定会解锁）。上面的代码修改如下：

```cpp
class LogFile {
    std::mutex _mu;
    ofstream f;
public:
    LogFile() {
        f.open("log.txt");
    }
    ~LogFile() {
        f.close();
    }
    void shared_print(string msg, int id) {

        std::unique_lock<std::mutex> guard(_mu);
        //do something 1
        guard.unlock(); //临时解锁

        //do something 2

        guard.lock(); //继续上锁
        // do something 3
        f << msg << id << endl;
        cout << msg << id << endl;
        // 结束时析构guard会临时解锁
        // 这句话可要可不要，不写，析构的时候也会自动执行
        // guard.ulock();
    }

};
```

上面的代码可以看到，在无需加锁的操作时，可以先临时释放锁，然后需要继续保护的时候，可以继续上锁，这样就无需重复的实例化`lock_guard`对象，还能减少锁的区域。同样，可以使用`std::defer_lock`设置初始化的时候不进行默认的上锁操作：

```cpp
void shared_print(string msg, int id) {
    std::unique_lock<std::mutex> guard(_mu, std::defer_lock);
    //do something 1

    guard.lock();
    // do something protected
    guard.unlock(); //临时解锁

    //do something 2

    guard.lock(); //继续上锁
    // do something 3
    f << msg << id << endl;
    cout << msg << id << endl;
    // 结束时析构guard会临时解锁
}
```

这样使用起来就比`lock_guard`更加灵活！然后这也是有代价的，因为它内部需要维护锁的状态，所以效率要比`lock_guard`低一点，在`lock_guard`能解决问题的时候，就是用`lock_guard`，反之，使用`unique_lock`。

##  条件变量

互斥锁`std::mutex`是一种最常见的线程间同步的手段，但是在有些情况下不太高效。

假设想实现一个简单的消费者生产者模型，一个线程往队列中放入数据，一个线程往队列中取数据，取数据前需要判断一下队列中确实有数据，由于这个队列是线程间共享的，所以，需要使用互斥锁进行保护，一个线程在往队列添加数据的时候，另一个线程不能取，反之亦然。用互斥锁实现如下：

```cpp
#include <iostream>
#include <deque>
#include <thread>
#include <mutex>

std::deque<int> q;
std::mutex mu;

void function_1() {
    int count = 10;
    while (count > 0) {
        std::unique_lock<std::mutex> locker(mu);
        q.push_front(count);
        locker.unlock();
        std::this_thread::sleep_for(std::chrono::seconds(1));
        count--;
    }
}

void function_2() {
    int data = 0;
    while ( data != 1) {
        std::unique_lock<std::mutex> locker(mu);
        if (!q.empty()) {
            data = q.back();
            q.pop_back();
            locker.unlock();
            std::cout << "t2 got a value from t1: " << data << std::endl;
        } else {
            locker.unlock();
        }
    }
}
int main() {
    std::thread t1(function_1);
    std::thread t2(function_2);
    t1.join();
    t2.join();
    return 0;
}

//输出结果
//t2 got a value from t1: 10
//t2 got a value from t1: 9
//t2 got a value from t1: 8
//t2 got a value from t1: 7
//t2 got a value from t1: 6
//t2 got a value from t1: 5
//t2 got a value from t1: 4
//t2 got a value from t1: 3
//t2 got a value from t1: 2
//t2 got a value from t1: 1
```

可以看到，互斥锁其实可以完成这个任务，但是却存在着性能问题。

首先，`function_1`函数是生产者，在生产过程中，`std::this_thread::sleep_for(std::chrono::seconds(1));`表示延时`1s`，所以这个生产的过程是很慢的；`function_2`函数是消费者，存在着一个`while`循环，只有在接收到表示结束的数据的时候，才会停止，每次循环内部，都是先加锁，判断队列不空，然后就取出一个数，最后解锁。所以说，在`1s`内，做了很多无用功！

这就引出了条件变量（condition variable）,`c++11`中提供了`#include <condition_variable>`头文件，其中的`std::condition_variable`可以和`std::mutex`结合一起使用，其中有两个重要的接口，`notify_one()`和`wait()`，`wait()`可以让线程陷入**休眠状态**，在消费者生产者模型中，如果生产者发现队列中没有东西，就可以让自己休眠，但是不能一直不干活啊，`notify_one()`就是**唤醒**处于`wait`中的**其中一个条件变量**（可能当时有很多条件变量都处于`wait`状态）。那什么时刻使用`notify_one()`比较好呢，当然是在生产者往队列中放数据的时候了，队列中有数据，就可以赶紧叫醒等待中的线程起来干活了。

使用条件变量修改后如下：

```cpp
#include <iostream>
#include <deque>
#include <thread>
#include <mutex>
#include <condition_variable>

std::deque<int> q;
std::mutex mu;
std::condition_variable cond;

void function_1() {
    int count = 10;
    while (count > 0) {
        std::unique_lock<std::mutex> locker(mu);
        q.push_front(count);
        locker.unlock();
        cond.notify_one();  // Notify one waiting thread, if there is one.
        std::this_thread::sleep_for(std::chrono::seconds(1));
        count--;
    }
}

void function_2() {
    int data = 0;
    while ( data != 1) {
        std::unique_lock<std::mutex> locker(mu);
        while(q.empty())
            cond.wait(locker); // Unlock mu and wait to be notified
        data = q.back();
        q.pop_back();
        locker.unlock();
        std::cout << "t2 got a value from t1: " << data << std::endl;
    }
}
int main() {
    std::thread t1(function_1);
    std::thread t2(function_2);
    t1.join();
    t2.join();
    return 0;
}
```

上面的代码有三个注意事项：

1. 在`function_2`中，在判断队列是否为空的时候，使用的是`while(q.empty())`，而不是`if(q.empty())`，这是因为`wait()`从阻塞到返回，不一定就是由于`notify_one()`函数造成的，还有可能由于系统的不确定原因唤醒（可能和条件变量的实现机制有关），这个的时机和频率都是不确定的，被称作**伪唤醒**，如果在错误的时候被唤醒了，执行后面的语句就会错误，所以需要再次判断队列是否为空，如果还是为空，就继续`wait()`阻塞。
2. 在管理互斥锁的时候，使用的是`std::unique_lock`而不是`std::lock_guard`，而且事实上也不能使用`std::lock_guard`，这需要先解释下`wait()`函数所做的事情。可以看到，在`wait()`函数之前，使用互斥锁保护了，如果`wait`的时候什么都没做，岂不是一直持有互斥锁？那生产者也会一直卡住，不能够将数据放入队列中了。所以，**`wait()`函数会先调用互斥锁的`unlock()`函数，然后再将自己睡眠，在被唤醒后，又会继续持有锁，保护后面的队列操作。**而`lock_guard`没有`lock`和`unlock`接口，而`unique_lock`提供了。这就是必须使用`unique_lock`的原因。
3. 使用细粒度锁，尽量减小锁的范围，在`notify_one()`的时候，不需要处于互斥锁的保护范围内，所以在唤醒条件变量之前可以将锁`unlock()`。

还可以将`cond.wait(locker);`换一种写法，`wait()`的第二个参数可以传入一个函数表示检查条件，这里使用`lambda`函数最为简单，如果这个函数返回的是`true`，`wait()`函数不会阻塞会直接返回，如果这个函数返回的是`false`，`wait()`函数就会阻塞着等待唤醒，如果被伪唤醒，会继续判断函数返回值。



# 1. std::promise

std::promise是一个模板类: `template<typename R> class promise`。其泛型参数R为std::promise对象保存的值的类型，R可以是void类型。std::promise保存的值可被与之关联的std::future读取，读取操作可以发生在其它线程。std::promise**允许move**语义(右值构造，右值赋值)，但**不允许拷贝**(拷贝构造、赋值)，std::future亦然。std::promise和std::future合作共同实现了多线程间通信。

## 1.1 设置std::promise的值

通过成员函数set_value可以设置std::promise中保存的值，该值最终会被与之关联的std::future::get读取到。**需要注意的是：set_value只能被调用一次，多次调用会抛出std::future_error异常**。事实上std::promise::set_xxx函数会改变std::promise的状态为ready，再次调用时发现状态已要是reday了，则抛出异常。



```c++
#include <iostream> // std::cout, std::endl
#include <thread>   // std::thread
#include <string>   // std::string
#include <future>   // std::promise, std::future
#include <chrono>   // seconds
using namespace std::chrono;

void read(std::future<std::string> *future) {
    // future会一直阻塞，直到有值到来
    std::cout << future->get() << std::endl;
}

int main() {
    // promise 相当于生产者
    std::promise<std::string> promise;
    // future 相当于消费者, 右值构造
    std::future<std::string> future = promise.get_future();
    // 另一线程中通过future来读取promise的值
    std::thread thread(read, &future);
    // 让read等一会儿:)
    std::this_thread::sleep_for(seconds(1));
    // 
    promise.set_value("hello future");
    // 等待线程执行完成
    thread.join();

    return 0;
}
```

与std::promise关联的std::future是通过std::promise::get_future获取到的，自己构造出来的无效。**一个std::promise实例只能与一个std::future关联共享状态**，当在同一个std::promise上反复调用get_future会抛出future_error异常。
 共享状态。在std::promise构造时，std::promise对象会与共享状态关联起来，这个共享状态可以存储一个R类型的值或者一个由std::exception派生出来的异常值。通过std::promise::get_future调用获得的std::future与std::promise共享相同的共享状态



### std::async

```cpp
//防止数据竞争
std::mutex g_mtu;

void calA(int& sum)
{
    this_thread::sleep_for(std::chrono::milliseconds(1));
    //多线程加锁保护
    lock_guard<mutex> lk(g_mtu);
    sum += 1;
}

void calB(int& sum)
{
    this_thread::sleep_for(std::chrono::milliseconds(2));
    lock_guard<mutex> lk(g_mtu);
    sum += 2;
}

void calC(int& sum)
{
    this_thread::sleep_for(std::chrono::milliseconds(3));
    lock_guard<mutex> lk(g_mtu);
    sum += 3;
}

//case2
void mian()
{
    int sum = 0;
    
	//启动三个线程，进行运算
    thread t1(calA, ref(sum));
    thread t2(calB, ref(sum)); 
    thread t3(calC, ref(sum)); 
	//等待计算完成
    t1.join();
    t2.join();
    t3.join();
    //计算完成
    cout << "test_threads_time result is :" <<  sum << "  ";
        
}
```

异步调用

```cpp
//case3
int calculateA()
{
    //延迟1ms
    this_thread::sleep_for(std::chrono::milliseconds(1));
    return 1;
}

//计算B函数
int calculateB()
{
    //延迟2ms
    this_thread::sleep_for(std::chrono::milliseconds(2));
    return 2;
}

//计算C函数
int calculateC()
{
    //延迟3ms
    this_thread::sleep_for(std::chrono::milliseconds(3));
    return 3;
}
void main()
{
	int sum = 0;
	
	//异步调用
	future<int> f1 = std::async(calculateA); //而std::async只是一个函数模板
	future<int> f2 = std::async(calculateB);
	future<int> f3 = std::async(calculateC);
	//get()函数会阻塞，直到结果返回
	sum = f1.get() + f2.get() + f3.get();
	cout << "test_feture_time result is :" <<  sum << "  ";
   
}
```

async相比thread的不同：

- std::thread是一个类模板，而std::async只是一个函数模板
- std::async返回std::future对象，让异步操作创建者访问异步结果.
- 调用std::thread总启动一个新线程，且在新线程中立即执行f
- 调用std::async不一定会启动一个新线程，并且可以决定是否立即执行f、延迟执行、不执行

async的policy参数

- 设置为launch::async,  将在新线程中立即执行f
- 设置为launch::deferred，async的f将延期执行直到future对象被查询(get/wait)，并且f将会在查询future对象的线程中执行，这个线程甚至不一定是调用async的线程；如果future对象从未被查询，f将永远不会执行。
- 设置为launch::deferred |std::launch::async,与实现有关，可能立即异步执行，也可能延迟执行。



相比多线程版本的计算过程，async异步调用整个程序更加简洁，没有明显引入线程，锁，缓冲区等概念.async异步调用函数通过启动一个新线程或者复用一个它认为合适的已有线程.”简单性”是async/future设计中最重视的一个方面；future一般也可以和线程一起使用，不过不要使用async()来启动类似I/O操作，操作互斥体（mutex），多任务交互操作等复杂任务。





###  std::future

  future是**类模板**，future对象里面含有线程执行的结果，可以通过调用函数get()来获取结果。但是，get()函数的设计包含**移动**语义，即只能调用一次，第二次调用时会报异常。故可以使用 std::shared_future







原子操作

### 

```cpp
#include<iostream>
#include<mutex>
#include <atomic>
using namespace std;
atomic<int> g_count = 0;    //全局变量
void myThread()
{
	for (int i = 0; i < 1000000; i++)
	{
		g_count++;
	}
}
int main()
{
	thread t1(myThread);
	thread t2(myThread);
	t1.join();
	t2.join();
	cout << "g_count = " << g_count << endl;
	return 0;

}
```

  原子操作(atomic)比互斥量(mutex)加锁效率高。

  原子操作(atomic)只能针对一个变量加锁，而互斥量(mutex)可以对代码段加锁。

  原子操作(atomic)支持的运算符有：++ , -- , += , -= , &=, |= , ^= 。

  原子操作(atomic)不支持拷贝构造函数和赋值运算符，可以分别用load()和store()以原子方式读取和写入。