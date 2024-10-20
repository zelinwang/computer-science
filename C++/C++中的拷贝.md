# 拷贝

浅拷贝和深拷贝最根本的区别在于是否真正获取一个对象的复制实体，而不是“引用”。浅拷贝和深拷贝一般在拷贝构造函数和赋值运算符重载函数中涉及到。在对象拷贝过程中，如果没有自定义拷贝构造函数，系统会提供一个缺省的拷贝构造函数，缺省的拷贝构造函数对于基本类型的成员变量，**按字节复制**，对于类类型的成员变量，**调用其相应类型的拷贝构造函数**。在拷贝的过程中会发生两种拷贝，**浅拷贝**和**深拷贝**。浅拷贝只复制指向某个对象的指针，而不复制对象本身，新旧对象还是共享同一块内存。但深拷贝会另外创造一个一模一样的对象，新对象跟原对象不共享内存，修改新对象不会改到原对象。

## 一）浅拷贝

浅拷贝就是对象的数据成员之间的简单赋值，如你设计了一个没有类而没有提供它的复制构造函数，当用该类的一个对象去给令一个对象赋值时所执行的过程就是浅拷贝。

```
int * a = 100;
int * b = a;//浅拷贝（位拷贝）
```

## 二）深拷贝

深拷贝指的就是当拷贝对象中有对其他资源（如堆、文件、系统等）的引用时（引用可以是指针或引用）时，对象另开辟一块新的资源，而不是对拷贝对象中其他资源的指针或引用进行单纯的赋值。

```
int * a = 100;
int * b = new int;
*b = *a//深拷贝
```

## 三）拷贝构造函数

​		如果一个类型没有实现拷贝构造函数，编译器会提供默认拷贝构造器函数。对于类的成员属性来说，则只是提供简单位拷贝，所以对于有指针的，且需要复制的类来讲，最好自定义拷贝构造函数。

```
class A
{
public:
	A():_val{new int}
	A(const A&a):_val(a.getVal()){}//浅拷贝
	
	int * getVal() const {return _val;}
private:
	int *_val;
}

int main()
{
	A a;
	A b(a);
	return 0;
}
```

​		在构造对象b时候， 其实b成员变量_val指向的a的_val的地址，当a和b都析构的时候，对同一块内存空间进行两次释放会出现问题，所以需要讲拷贝构造函数写成深拷贝，或者对析构函数进行定义

```
class A
{
public:
	A():_val{new int}
	~A()
	{
		if(_val)
		{
			delete _val;
			_val = nullptr;//保证不会对空指针进行第二次析构
		}
	}
	A(const A&a):_val(new int){*_val = *(a.getVal());}//深拷贝
	
	int * getVal() const {return _val;}
private:
	int *_val;
}
```

## 三）C++ 中如何避免拷贝

1)在新标准下，将拷贝构造函数和拷贝赋值运算符定义为删除函数(delete function)

```
class A
{
public:
	A():_val{new int}
	A(const A&a) = delete;
	A & operator(const A&a) = delete;
	int * getVal() const {return _val;}
private:
	int *_val;
}
```

2)将拷贝构造函数和拷贝赋值运算符声明为私有的且不予实现

```
class A
{
public:
	A():_val{new int}
	int * getVal() const {return _val;}
private:
	A(const A&a);
	A & operator(const A&a);
private:
	int *_val;
}
```

