## 一、大小

### 一）空类的大小

空类的大小是一个字节，这是被编译器插进去的一个char，使这个class的实体（object）在内存中有自己的位置。

### 二）含有虚函数的空类大小是多少

大小为4字节，因为含有虚函数表的指针

## 二、空类产生的函数

一个空类在编译后会产生五种函数（6个函数）：

1. 默认构造函数
2. 析构函数
3. 拷贝构造函数 
4. 赋值运算符（operator=）
5. 取址运算符（operator&）（一对，一个非const的，一个const的）

```c++
class Empty
{
public:
	Empty();                           // 缺省构造函数
	Empty( const Empty& );             // 拷贝构造函数
	~Empty();                          // 析构函数
	Empty& operator=( const Empty& );  // 赋值运算符
	Empty* operator&();                // 取址运算符
	const Empty* operator&() const;    // 取址运算符 const
};
```

  

 

