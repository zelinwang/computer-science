# C++ 内存分配(new，operator new)详解

## 一、new 操作符的执行过程

　　**a、内存分配**
　　　　调用相应的operator new( )分配内存
　　**b、构造函数**
　　　　调用相应的构造函数　　

## 二、new的内部实现机制


我们都知道new分为两步完成，但是new是怎样完成这两步的呢？开始我以为是在new内部封装了一个两个函数，一个用来分配内存，另一个用来调用相应的构造函数。

但事实上并非如此，根本就没有一个“隐藏”的new函数，编译器会自动将new改写成一个内存分配函数，一个构造函数。
这里其实还有一个更本质的问题需要说明：new的本质是关键字，编译器在背后的操作可能与我们的想象大不相同。能够重载的是函数，是运算符，关键字是不能重载的（有的博客中说operator new可以重载，new不能重载就是这个原因，更本质的原因可以概括为不能重载关键字），这也是为什么运算符重载前面必须要有operator的原因。+(int a)，new(int a)都是错的。
简而言之，new在编译器的操作下会变成两部分代码，一部分分配内存，一部分调用构造函数。

这其实是可以做实验验证的：
就在上述代码中加入new T并打上断点，再F10一步一步地调试，发现先进入类中的operator new，执行完后并没有到new T的下一行，而是依然停留在new这一行，继续F10进入构造函数。就是说new T这一行代码停留了两次。这意味着编译器确实将new T这一行代码翻译成了两次函数调用。

第二种验证方式就是直接使用反汇编查看代码（推荐）：
　　　　　　
可以看到的确有两个call，第一个call T::operator new( )。第二个call T::T( )。注意：真正的汇编中没有具体的函数名，只有相应的地址。显示函数名是VS的一个比较便利的功能。

## 三、new运算符和operator new()

new：指我们在C++里通常用到的运算符，比如A* a = new A;  对于new来说，有new和::new之分，前者位于std::operator new()：指对new的重载形式，它是一个函数，并不是运算符。对于operator new来说，分为全局重载和类重载，全局重载是void* ::operator new(size_t size)，在类中重载形式 void* A::operator new(size_t size)。还要注意的是这里的operator new()完成的操作一般只是分配内存，事实上系统默认的全局::operator new(size_t size)也只是调用malloc分配内存，并且返回一个void*指针。而构造函数的调用(如果需要)是在new运算符中完成的

## 四、new和operator new之间的关系

A* a = new A；我们知道这里分为两步：1.分配内存，2.调用A()构造对象。事实上，分配内存这一操作就是由operator new(size_t)来完成的，如果类A重载了operator new，那么将调用A::operator new(size_t )，如果没有重载，就调用::operator new(size_t )，全局new操作符由C++默认提供。因此前面的两步也就是：1.调用operator new 2.调用构造函数。

（1）new ：不能被重载，其行为总是一致的。它先调用operator new分配内存，然后调用构造函数初始化那段内存。

new 操作符的执行过程：

1. 调用operator new分配内存 ；

2. 调用构造函数生成类对象；

3. 返回相应指针。

（2）operator new：要实现不同的内存分配行为，应该重载operator new，而不是new。

operator new就像operator + 一样，是可以重载的。如果类中没有重载operator new，那么调用的就是全局的::operator new来完成堆的分配。同理，operator new[]、operator delete、operator delete[]也是可以重载的。



## 五、如何让类只能建立在栈上(重载new函数设为私有)

只有使用new运算符，对象才会建立在堆上，因此，只要禁用new运算符就可以实现类对象只能建立在栈上。将operator new()设为私有即可。

## 六、placement new，将对象建立在指定的空间

通过placement new可将对象产生在指定的内存空间。

```c++
class A
{
public:
	A(){}
	~A(){}
private:
	int a;
}
char * buff = new char[sizeof(A)];
A * a = new(buff) A;

char _buff[sizeof(A)];
A * _a = new(_buff) A;
```

经过测试，buff的空间地址和a的空间地址是一致的。而且，对象不仅可以建立在堆空间，也可以建立在栈空间。