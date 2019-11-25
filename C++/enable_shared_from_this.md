## enable_shared_from_this和shared_from_this机制

当你做这样的代码操作时：

```cpp
shared_ptr<int> ptr1(new int);
shared_ptr<int> ptr2(ptr1);
cout<<ptr1.use_count()<<endl;
cout<<ptr2.use_count()<<endl;
1234
```

这段代码没有任何问题，**ptr1和ptr2管理了同一个资源，引用计数打印出来的都是2，出函数作用域依次析构，最终new int资源只释放一次，逻辑正确**！这是因为shared_ptr ptr2(ptr1)调用了shared_ptr的拷贝构造函数（源码可以自己查看下），只是做了资源的引用计数的改变，没有额外分配其它资源，如下图所示：

但是当你做如下代码操作时：

```cpp
int *p = new int;
shared_ptr<int> ptr1(p);
shared_ptr<int> ptr2(p);
cout<<ptr1.use_count()<<endl;
cout<<ptr2.use_count()<<endl;
12345
```

这段代码就有问题了，因为shared_ptr ptr1( p )和shared_ptr ptr2( p )**都调用了shared_ptr的构造函数**，在它的构造函数中，**都重新开辟了引用计数的资源**，导致ptr1和ptr2都记录了一次new int的引用计数，都是1，析构的时候它俩都去释放内存资源，导致释放逻辑错误，如下图所示：



## 代码清单1

```cpp
#include <iostream>
using namespace std;
// 智能指针测试类
class A
{
public:
	A():mptr(new int) 
	{
		cout << "A()" << endl;
	}
	~A()
	{
		cout << "~A()" << endl;
		delete mptr; 
		mptr = nullptr;
	}
	 
	// A类提供了一个成员方法，返回指向自身对象的shared_ptr智能指针。
	shared_ptr<A> getSharedPtr() 
	{ 
		/*注意：不能直接返回this，在多线程环境下，根本无法获知this指针指向
		的对象的生存状态，通过shared_ptr和weak_ptr可以解决多线程访问共享		
		对象的线程安全问题*/
		return shared_ptr<A>(this); 
	}
private:
	int *mptr;
};
int main()
{
	shared_ptr<A> ptr1(new A());
	shared_ptr<A> ptr2 = ptr1->getSharedPtr();

	/* 按原先的想法，上面两个智能指针管理的是同一个A对象资源，但是这里打印都是1
	导致出main函数A对象析构两次，析构逻辑有问题*/
	cout << ptr1.use_count() << endl; 
	cout << ptr2.use_count() << endl;

	return 0;
}
```

**代码运行打印如下**：

```
A()
1
1
~A()
~A()
```

为什么两次析构,因为shared_ptr<A> ptr2 = ptr1->getSharedPtr();中返回的return shared_ptr<A>(this); 还是一个构造函数,所以ptr1和ptr2虽然指向内容一样,但是引用数都是1





解决以上办法的方法就是**继承enable_shared_from_this类**，然后通过**调用从基类继承来的shared_from_this()方法**返回指向同一个资源对象的智能指针shared_ptr。即



## 修改代码清单1

```cpp
#include <iostream>
using namespace std;
// 智能指针测试类，继承enable_shared_from_this类
class A : public enable_shared_from_this<A>
{
public:
	A() :mptr(new int)
	{
		cout << "A()" << endl;
	}
	~A()
	{
		cout << "~A()" << endl;
		delete mptr;
		mptr = nullptr;
	}

	// A类提供了一个成员方法，返回指向自身对象的shared_ptr智能指针
	shared_ptr<A> getSharedPtr()
	{
		/*通过调用基类的shared_from_this方法得到一个指向当前对象的
		智能指针*/
		return shared_from_this();
	}
private:
	int *mptr;
};
int main()
{
	shared_ptr<A> ptr1(new A());
	shared_ptr<A> ptr2 = ptr1->getSharedPtr();

	// 引用计数打印为2
	cout << ptr1.use_count() << endl;
	cout << ptr2.use_count() << endl;

	return 0;
}

代码打印如下：

A()
2
2
~A()

```

也就是说，如果一个类继承了enable_shared_from_this，那么它产生的对象就会从基类enable_shared_from_this继承一个成员变量_Wptr，**当定义第一个智能指针对象的时候shared_ptr< A > ptr1(new A())，调用shared_ptr的普通构造函数，就会初始化A对象的成员变量_Wptr，作为观察A对象资源的一个弱智能指针观察者das ** 

