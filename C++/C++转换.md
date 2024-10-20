## 一、dynamic_cast

用法：

```c++
dynamic_cast<type-id> ( expression ) 
```

dynamic_cast（动态转换），允许在运行时刻进行类型转换，从而使程序能够在一个类层次结构安全地转换类型。
dynamic_cast 提供了两种转换方式，把基类指针转换成派生类指针，或者把指向基类的左值转换成派生类的引用。

实现机制是RTTI

## 二、reinterpret_cast

用法：

```c++
reinterpret_cast<type-id> (expression)
```

type-id 必须是一个指针、引用、算术类型、函数指针或者成员指针。它可以把一个指针转换成一个整数，
也可以把一个整数转换成一个指针（先把一个指针转换成一个整数，在把该整数转换成原类型的指针，还可以得到原先的指针值）

## 三、static_cast

用法：

```c++
static_cast<type-id>(expression) 
```

该运算符把 expression 转换为 type-id 类型，但没有运行时类型检查来保证转换的安全性。它主要有如下几种用法：

① 用于类层次结构中基类和子类之间指针或引用的转换。进行上行转换（把子类的指针或引用转换成基类表示）是安全的；进行下行转换（把基类指针或引用转换成子类表示）时，由于没有动态类型检查，所以是不安全的。
② 用于基本数据类型之间的转换，如把 int 转换成 char，把 int 转换成 enum。这种转换的安全性也要开发人员来保证。
③ 把空指针转换成目标类型的空指针。
④ 把任何类型的表达式转换成void类?
注意 static_cast 不能转换掉 expression 的 const、volitale、或者 __unaligned 属性。

## 四、const_cast

用法：

```c++
const_cast<type-id>(const expression) 
```

该运算符把 const expression 转换为 无const的type-id 类型