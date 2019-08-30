#### **一.new** 

new operator就是new操作符，不能被重载，假如A是一个类，那么A * a=new A;实际上执行如下3个过程。

1. 调用operator new分配内存，operator new (sizeof(A)) 

2. 调用构造函数生成类对象，A::A()   

3. 返回相应指针

事实上，分配内存这一操作就是由operator new(size_t)来完成的，如果类A重载了operator new，那么将调

用A::operator new(size_t )，否则调用全局::operator new(size_t )，后者由C++默认提供。

#### 二.operator new

operator new是函数，分为三种形式（前2种不调用构造函数，这点区别于new operator）：

1. void* operator new (std::size_t size) throw (std::bad_alloc); 
   分配size个字节的存储空间，并将对象类型进行内存对齐。如果成功，返回一个非空的指针指向首地址。失败抛出bad_alloc异常。

2. void* operator new (std::size_t size, const std::nothrow_t& nothrow_constant) throw(); 
   在分配失败时不抛出异常，它返回一个NULL指针。

3. void* operator new (std::size_t size, void* ptr) throw(); 

   placement new版本，它本质上是对operator new的重载，定义于#include <new>中。它不分配内存，调用合适的构造函数在ptr所指的地方构造一个对象，之后返回实参指针ptr。

 A* a = new A; //调用第一种
 new (p)A(); //调用第三种
 下面是重载operator new的一个例子：

```c++
#include <iostream>
#include <string>
using namespace std;
 
class X
{
public:
	X()
	{
		cout << "X's constructor" << endl;
	}
	~X()
	{
		cout << "X's destructor" << endl;
	}
 
	void* operator new(size_t size, string str)
	{
		cout << "operator new size " << size << " with string " << str << endl;
		return ::operator new(size);
	}
 
 
	void operator delete(void* pointer)
	{
		cout << "operator delete" << endl;
		::operator delete(pointer);
	}
private:
	int num;
};
 
int main()
{
	X *p = new("A new class") X;
	delete p;
	getchar();
	return 0;
}
```

**![img](http://img.voidcn.com/vcimg/000/014/869/406_9e8_b0f.jpg)**

#### 三.placement new

一般来说，使用new申请空间时，是从系统的“堆”（heap）中分配空间。申请所得的空间的位置是根据当时的内存的实际使用情况决定的。但是，在某些特殊情况下，可能需要在已分配的特定内存创建对象，这就是所谓的“定位放置new”（placement new）操作。 定位放置new操作的语法形式不同于普通的new操作。例如，一般都用如下语句A* p=new A;申请空间，而定位放置new操作则使用如下语句A* p=new (ptr)A;申请空间，其中ptr就是程序员指定的内存首地址。考察如下程序。

```c++
#include <iostream>
using namespace std;
 
class A
{
public:
	A()
	{
		cout << "A's constructor" << endl;
	}
    
	~A()
	{
		cout << "A's destructor" << endl;
	}
	
	void show()
	{
		cout << "num:" << num << endl;
	}
	
private:
	int num;
};
 
int main()
{
	char mem[100];
	mem[0] = 'A';
	mem[1] = '\0';
	mem[2] = '\0';
	mem[3] = '\0';
	cout << (void*)mem << endl;
	A* p = new (mem)A;
	cout << p << endl;
	p->show();
	p->~A();
	getchar();
}
```

阅读以上程序，注意以下几点。  

1. 用定位放置new操作，既可以在栈(stack)上生成对象，也可以在堆（heap）上生成对象。如本例就是在栈上生成一个对象。 

2. 使用语句A* p=new (mem) A;定位生成对象时，指针p和数组名mem指向同一片存储区。所以，与其说定位放置new操作是申请空间，还不如说是利用已经请好的空间，真正的申请空间的工作是在此之前完成的。 
3. 使用语句A *p=new (mem) A;定位生成对象时，会自动调用类A的构造函数，但是由于对象的空间不会自动释放（对象实际上是借用别人的空间），所以必须显示的调用类的析构函数，如本例中的p->~A()。 
4. 如果有这样一个场景，我们需要大量的申请一块类似的内存空间，然后又释放掉，比如在在一个server中对于客户端的请求，每个客户端的每一次上行数据我们都需要为此申请一块内存，当我们处理完请求给客户端下行回复时释放掉该内存，表面上看者符合c++的内存管理要求，没有什么错误，但是仔细想想很不合理，为什么我们每个请求都要重新申请一块内存呢，要知道每一次内从的申请，系统都要在内存中找到一块合适大小的连续的内存空间，这个过程是很慢的（相对而言)，极端情况下，如果当前系统中有大量的内存碎片，并且我们申请的空间很大，甚至有可能失败。为什么我们不能共用一块我们事先准备好的内存呢？可以的，我们可以使用placement new来构造对象，那么就会在我们指定的内存空间中构造对象。 
   下面是一个在堆上生成对象的例子。

```C++
#include <iostream>
using namespace std;
 
class B
{
public:
    B()
    {
        cout<<"B's constructor"<<endl;
    }
 
    ~B()
    {
        cout<<"B's destructor"<<endl;
    }
 
    void SetNum(int n)
    {
        num = n;
    }
 
    int GetNum()
    {
        return num;
    }
 
 
private:
    int num;
};
 
int main()
{
	char* mem = new char[10 * sizeof(B)];
	cout << (void*)mem << endl;
	B *p = new(mem)B;
	cout << p << endl;
	p->SetNum(10);
	cout << p->GetNum() << endl;
	p->~B();
	delete[]mem;
	getchar();
}
```

