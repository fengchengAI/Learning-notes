# 带引用计数的智能指针shared_ptr、weak_ptr

这里主要介绍shared_ptr和weak_ptr两个智能指针，**什么是带引用计数的智能指针**？当允许多个智能指针指向同一个资源的时候，**每一个智能指针都会给资源的引用计数加1，当一个智能指针析构时，同样会使资源的引用计数减1，这样最后一个智能指针把资源的引用计数从1减到0时，就说明该资源可以释放了**，由最后一个智能指针的析构函数来处理资源的释放问题，这就是引用计数的概念。

要对资源的引用个数进行计数，那么大家知道，**对于整数的++或者- -操作，它并不是线程安全的操作，因此shared_ptr和weak_ptr底层的引用计数已经通过CAS操作，保证了引用计数加减的原子特性，因此shared_ptr和weak_ptr本身就是线程安全的带引用计数的智能指针**。

# 智能指针的交叉引用（循环引用）问题

请看下面的这个代码示例：

```cpp
#include <iostream>
#include <memory>
using namespace std;

class B; // 前置声明类B
class A
{
public:
	A() { cout << "A()" << endl; }
	~A() { cout << "~A()" << endl; }
	shared_ptr<B> _ptrb; // 指向B对象的智能指针
};
class B
{
public:
	B() { cout << "B()" << endl; }
	~B() { cout << "~B()" << endl; }
	shared_ptr<A> _ptra; // 指向A对象的智能指针
};
int main()
{
	shared_ptr<A> ptra(new A());// ptra指向A对象，A的引用计数为1
	shared_ptr<B> ptrb(new B());// ptrb指向B对象，B的引用计数为1
	ptra->_ptrb = ptrb;// A对象的成员变量_ptrb也指向B对象，B的引用计数为2
	ptrb->_ptra = ptra;// B对象的成员变量_ptra也指向A对象，A的引用计数为2

	cout << ptra.use_count() << endl; // 打印A的引用计数结果:2
	cout << ptrb.use_count() << endl; // 打印B的引用计数结果:2

	/*
	出main函数作用域，ptra和ptrb两个局部对象析构，分别给A对象和
	B对象的引用计数从2减到1，达不到释放A和B的条件（释放的条件是
	A和B的引用计数为0），因此造成两个new出来的A和B对象无法释放，
	导致内存泄露，这个问题就是“强智能指针的交叉引用(循环引用)问题”
	*/
	return 0;
}

```

代码打印结果：
 A()
 B()
 2
 2
 可以看到，A和B对象并没有进行析构，通过上面的代码示例，能够看出来“交叉引用”的问题所在，就是对象无法析构，资源无法释放，那怎么解决这个问题呢？请注意强弱智能指针的一个重要应用规则：**定义对象时，用强智能指针shared_ptr，在其它地方引用对象时，使用弱智能指针weak_ptr**。

**弱智能指针weak_ptr区别于shared_ptr之处在于**：

1. weak_ptr不会改变资源的引用计数，只是一个观察者的角色，通过观察shared_ptr来判定资源是否存在
2. weak_ptr持有的引用计数，不是资源的引用计数，而是同一个资源的观察者的计数
3. weak_ptr没有提供常用的指针操作，无法直接访问资源，需要先通过lock方法提升为shared_ptr强智能指针，才能访问资源（因为weak_ptr不知道所指向得share_ptr是否有效，lock会判断share_ptr是否有效）
4. 只要最后以一个指向对象的share_ptr被销毁，对象资源就会被释放，不管是否有weak_ptr指向该对象

那么上面的代码怎么修改，**也就是如何解决带引用计数的智能指针的交叉引用问题**，代码如下：

```cpp
#include <iostream>
#include <memory>
using namespace std;

class B; // 前置声明类B
class A
{
public:
	A() { cout << "A()" << endl; }
	~A() { cout << "~A()" << endl; }
	weak_ptr<B> _ptrb; // 指向B对象的弱智能指针。引用对象时，用弱智能指针
};
class B
{
public:
	B() { cout << "B()" << endl; }
	~B() { cout << "~B()" << endl; }
	weak_ptr<A> _ptra; // 指向A对象的弱智能指针。引用对象时，用弱智能指针
};
int main()
{
    // 定义对象时，用强智能指针
	shared_ptr<A> ptra(new A());// ptra指向A对象，A的引用计数为1
	shared_ptr<B> ptrb(new B());// ptrb指向B对象，B的引用计数为1
	// A对象的成员变量_ptrb也指向B对象，B的引用计数为1，因为是弱智能指针，引用计数没有改变
	ptra->_ptrb = ptrb;
	// B对象的成员变量_ptra也指向A对象，A的引用计数为1，因为是弱智能指针，引用计数没有改变
	ptrb->_ptra = ptra;

	cout << ptra.use_count() << endl; // 打印结果:1
	cout << ptrb.use_count() << endl; // 打印结果:1

	/*
	出main函数作用域，ptra和ptrb两个局部对象析构，分别给A对象和
	B对象的引用计数从1减到0，达到释放A和B的条件，因此new出来的A和B对象
	被析构掉，解决了“强智能指针的交叉引用(循环引用)问题”
	*/
	return 0;
}

```

代码打印如下：
 A()
 B()
 1
 1
 ~B()
 ~A()
 可以看到，A和B对象正常析构，问题解决！



# 多线程访问共享对象问题

线程A和线程B访问一个共享的对象，如果线程A正在析构这个对象的时候，线程B又要调用该共享对象的成员方法，此时可能线程A已经把对象析构完了，线程B再去访问该对象，就会发生不可预期的错误。

先看如下代码：

```cpp
#include <iostream>
#include <thread>
using namespace std;

class Test
{
public:
	// 构造Test对象，_ptr指向一块int堆内存，初始值是20
	Test() :_ptr(new int(20)) 
	{
		cout << "Test()" << endl;
	}
	// 析构Test对象，释放_ptr指向的堆内存
	~Test()
	{
		delete _ptr;
		_ptr = nullptr;
		cout << "~Test()" << endl;
	}
	// 该show会在另外一个线程中被执行
	void show()
	{
		cout << *_ptr << endl;
	}
private:
	int *volatile _ptr;
};
void threadProc(Test *p)
{
	// 睡眠两秒，此时main主线程已经把Test对象给delete析构掉了
	std::this_thread::sleep_for(std::chrono::seconds(2));
	/* 
	此时当前线程访问了main线程已经析构的共享对象，结果未知，隐含bug。
	此时通过p指针想访问Test对象，需要判断Test对象是否存活，如果Test对象
	存活，调用show方法没有问题；如果Test对象已经析构，调用show有问题！
	*/
	p->show();
}
int main()
{
	// 在堆上定义共享对象
	Test *p = new Test();
	// 使用C++11的线程类，开启一个新线程，并传入共享对象的地址p
	std::thread t1(threadProc, p);
	// 在main线程中析构Test共享对象
	delete p;
	// 等待子线程运行结束
	t1.join();
	return 0;
}

```

运行上面的代码，发现在main主线程已经delete析构Test对象以后，子线程threadProc再去访问Test对象的show方法，无法打印出*_ptr的值20。可以用shared_ptr和weak_ptr来解决多线程访问共享对象的线程安全问题，上面代码修改如下：

```cpp
#include <iostream>
#include <thread>
#include <memory>
using namespace std;

class Test
{
public:
	// 构造Test对象，_ptr指向一块int堆内存，初始值是20
	Test() :_ptr(new int(20)) 
	{
		cout << "Test()" << endl;
	}
	// 析构Test对象，释放_ptr指向的堆内存
	~Test()
	{
		delete _ptr;
		_ptr = nullptr;
		cout << "~Test()" << endl;
	}
	// 该show会在另外一个线程中被执行
	void show()
	{
		cout << *_ptr << endl;
	}
	int *volatile _ptr;
};
void threadProc(weak_ptr<Test> pw) // 通过弱智能指针观察强智能指针
{
	// 睡眠两秒
	std::this_thread::sleep_for(std::chrono::seconds(2));
	/* 
	如果想访问对象的方法，先通过pw的lock方法进行提升操作，把weak_ptr提升
	为shared_ptr强智能指针，提升过程中，是通过检测它所观察的强智能指针保存
	的Test对象的引用计数，来判定Test对象是否存活，ps如果为nullptr，说明Test对象
	已经析构，不能再访问；如果ps!=nullptr，则可以正常访问Test对象的方法。
	*/
	shared_ptr<Test> ps = pw.lock();
	if (ps != nullptr)
	{
		ps->show();
	}
}
int main()
{
	// 在堆上定义共享对象
	shared_ptr<Test> p(new Test);
	// 使用C++11的线程，开启一个新线程，并传入共享对象的弱智能指针
	std::thread t1(threadProc, weak_ptr<Test>(p));
	// 在main线程中析构Test共享对象
	// 等待子线程运行结束
	t1.join();
	return 0;
}

```

运行上面的代码，show方法可以打印出20，**因为main线程调用了t1.join()方法等待子线程结束，此时pw通过lock提升为ps成功**，见上面代码示例。

这里主要是因为shared_ptr是线程安全的.

如果设置t1为分离线程，让main主线程结束，p智能指针析构，进而把Test对象析构，此时show方法已经不会被调用，**因为在threadProc方法中，pw提升到ps时，lock方法判定Test对象已经析构，提升失败**！main函数代码可以如下修改测试：

```cpp
int main()
{
	// 在堆上定义共享对象
	shared_ptr<Test> p(new Test);
	// 使用C++11的线程，开启一个新线程，并传入共享对象的弱智能指针
	std::thread t1(threadProc, weak_ptr<Test>(p));
	// 在main线程中析构Test共享对象
	// 设置子线程分离
	t1.detach();
	return 0;
}

```

该main函数运行后，最终的threadProc中，show方法不会被执行到。**以上是在多线程中访问共享对象时，对shared_ptr和weak_ptr的一个典型应用**。

`注意`
首先用weak_ptr传入形参,然后用lock得道shared_ptr实际上和直接传入shared_ptr是一样的,因为lock获得的shared_ptr也会use_count++.

但是当t1.detach();时就不一样了,在线程里面sleep后,用weak_ptr传入形参的use_count并没有加1,在主线程结束时,use_count只有1,当离开main的作用域后use_count--, 即会调用析构函数.但是当用shared_ptr传递时,当离开main的作用域后use_count--后为1,所以没有调用析构函数.

