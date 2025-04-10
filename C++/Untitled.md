

## 左值和右值的定义

在C语言中，我们常常会提起左值、右值这样的称呼。而在编译程序时，编译器有时也会在报出的错误信息中会包含左值、右值的说法。不过左值、右值通常不是通过一个严谨的定义而为人所知的，大多数时候左右值的定义与其判别方法是一体的。一个最为典型的判别方法就是，在赋值表达式中，出现在等号左边的就是“左值”，而在等号右边的，则称为“右值”。比如：

```c++
a = b + c;
```

在这个赋值表达式中，a就是一个左值，而b+c则是一个右值。这种识别左值、右值的方法在C++中依然有效。不过C++中还有一个被广泛认同的说法，那就是可以取地址的、有名字的就是左值，反之，不能取地址的、没有名字的就是右值。那么这个加法赋值表达式中，&a是允许的操作，但&(b+ c)这样的操作则不会通过编译。因此 a是一个左值，(b+c)是一个右值。

这些判别方法通常都非常有效。更为细致地，在C++11中，右值是由两个概念构成的，一个是将亡值(xvalue，eXpiring Value )，另一个则是纯右值( prvalue，Pure Rvalue )。
其中纯右值就是C++98标准中右值的概念，讲的是用于辨识临时变量和一些不跟对象关联的值。比如非引用返回的函数返回的临时变量值就是一个纯右值。一些运算表达式，比如1＋3产生的临时变量值，也是纯右值。而不跟对象关联的字面量值，比如:2、’c’、true，也是纯右值。此外，类型转换函数的返回值、lambda表达式等，也都是右值。
而将亡值则是C++11新增的跟右值引用相关的表达式，这样表达式通常Markdown Preview Enhance是将要被移动的对象（移为他用)，比如返回右值引用T&&的函数返回值、std:move的返回值或者转换为T&&的类型转换函数的返回值。而剩余的，可以标识函数、对象的值都属于左值。在C++11的程序中，所有的值必属于左值、将亡值、纯右值三者之一。

## 右值引用与左值引用

在C++11中，右值引用就是对一个右值进行引用的类型。事实上，由于右值通常不具有名字，我们也只能通过引用的方式找到它的存在。通常情况下，我们只能是从右值表达式获得其引用。比如:

```c++
T&& a = ReturnRvalue();
```

这个表达式中，假设ReturnRvalue返回一个右值，我们就声明了一个名为a的右值引用，其值等于ReturnRvalue函数返回的临时变量的值。
为了区别于C++98中的引用类型，我们称C++98中的引用为“左值引用”( lvaluereference )。右值引用和左值引用都是属于引用类型。无论是声明一个左值引用还是右值引用，都必须立即进行初始化。而其原因可以理解为是引用类型本身自己并不拥有所绑定对象的内存，只是该对象的一个别名。左值引用是具名变量值的别名，而右值引用则是不具名(匿名）变量的别名。
在上面的例子中，ReturnRvalue函数返回的右值在表达式语句结束后，其生命也就终结了（通常也称其具有表达式生命期)，而通过右值引用的声明，该右值又“重获新生”，其生命期将与右值引用类型变量a的生命期一样。只要a还“活着”，该右值临时量将会一直“存活”下去。右值引用的原理是在返回值后，在栈空间中重新开辟了一份空间来存储临时变量，并且将栈地址传递给右值引用，造成了临时变量一直存在的假象。所以相比于以下语句的声明方式：

```
T b = ReturnRvalue();
```

我们刚才的右值引用变量声明，就会少一次对象的析构及一次对象的构造。因为a是右值引用，直接绑定了ReturnRvalue()返回的临时量，而b只是由临时值构造而成的，而临时量在表达式结束后会析构因应就会多一次析构和构造的开销。
不过值得指出的是，能够声明右值引用a的前提是ReturnRvalue返回的是一个右值。通常情况下，右值引用是不能够绑定到任何的左值的。比如下面的表达式就是无法通过编译的。

```
int c;
int && d = c ;
```

相对地，在C++98标准中就已经出现的左值引用是否可以绑定到右值（由右值进行初始化）呢?比如:

```
T & e = ReturnRvalue();
const T & f = ReturnRvalue();
```

这样的语句是否能够通过编译呢?这里的答案是：e的初始化会导致编译时错误，而f则不会。出现这样的状况的原因是，在常量左值引用在C++98标准中开始就是个“万能”的引用类型。它可以接受非常量左值、常量左值、右值对其进行初始化。而且在使用右值对其初始化的时候，常量左值引用还可以像右值引用一样将右值的生命期延长。不过相比于右值引用所引用的右值，常量左值所引用的右值在它的“余生”中只能是只读的。相对地，非常量左值只能接受非常量左值对其进行初始化。
为了语义的完整，C++11中还存在着常量右值引用，比如通过以下代码可以声明一个常量右值引用。

```
const T && crvalueref = ReturnRvalue();
```

但是，一来右值引用主要就是为了移动语义，而移动语义需要右值是可以被修改的，那么常量右值引用在移动语义中就没有用武之处﹔二来如果要引用右值且让右值不可以更改，常量左值引用往往就足够了。因此在现在的情况下，常量右值引用还没有任何用处。
下表列出了在C++11中各种引用类型可以引用的值的类型。值得注意的是，只要能够绑定右值的引用类型，都能够延长右值的生命期。

![](E:\Code\复习心得\res\picture\左值右值.jpg)