程序运行中对类对象进行判断，发现对象大小就是成员变量大小+虚函数指针大小，那对象是怎么获取成员函数的地址呢？

## C++函数编译

### 名字编码

C++和C语言的编译方式不同。C语言中的函数在编译时名字不变，或者只是简单的加一个下划线 `_`（不同的编译器有不同的实现），例如，func() 编译后为 func() 或 _func()。

在C++编译时会根据它所在的命名空间，所属的类、以及参数列表（也叫参数签名）等信息进行重新命名，形成新的函数名。这个新函数名只有编译器知道，对用户不可见。对函数重新命名的过程叫做**名字编码（Name Mangling）**，是通过特殊算法实现。

> 1. **Name Mangling算法是可逆的**，即可以通过现有的函数名计算出新的函数名称，也可以通过新的函数名称逆向推演原有的函数名称。
> 2. **Name Mangling算法保证函数名称的唯一性**，只要函数所在的名称、所属的类、包含的参数列表有一个不同，最后产生的新函数也是不一样的。

### 调用约定

关于 *C/C++* 函数调用约定，大多数时候并不会影响程序逻辑。

```
VC 中默认调用是 __cdecl 方式，Windows API 使用 __stdcall 调用方式，在 DLL 导出函数中，为了跟 Windows API 保持一致，建议使用 __stdcall 方式。
调用约定跟堆栈清除密切相关。如果写一个汇编函数，给 C/C++ 调用，在_cdecl 方式下，则汇编函数无需清除堆栈，在 __stdcall 方式下，汇编函数需要在返回（RET）之前恢复堆栈。

C 语言有__cdecl、__stdcall、__fastcall、naked、__pascal。
C++ 语言有__cdecl、__stdcall、__fastcall、naked、__pascal、__thiscall，比 C 语言多出一种 __thiscall 调用方式。
```

下面详细介绍如上六种调用方式：

1. __cdecl：_cdecl调用约定又称为 C 调用约定，是 C/C++ 语言缺省的调用约定。参数按照从右至左的方式入栈，函数本身不清理栈，此工作由调用者负责，返回值在EAX中。由于由调用者清理栈，所以允许可变参数函数存在，如int sprintf(char* buffer,const char* format,...);。
2. __stdcall：很多时候被称为 pascal 调用约定。pascal 语言是早期很常见的一种教学用计算机程序设计语言，其语法严谨。参数按照从右至左的方式入栈，函数自身清理堆栈，返回值在EAX中。

3. __fastcall：特点就是快，因为它通过 CPU 寄存器来传递参数。他用 ECX 和 EDX 传送前两个双字（DWORD）或更小的参数，剩下的参数按照从右至左的方式入栈，函数自身清理堆栈，返回值在 EAX 中。

4. naked： 是一个很少见的调用约定，一般不建议使用。编译器不会给这种函数增加初始化和清理代码，更特殊的是，你不能用return返回返回值，只能用插入汇编返回结果，此调用约定必须跟 __declspec 同时使用。例如定义一个求和程序，如__declspec(naked) int  add(int a,int b);。

5. _pascal：这是 pascal 语言的调用约定，跟 _stdcall 一样，参数按照从右至左的方式入栈，函数自身清理堆栈，返回值在EAX中。VC 中已经废弃了这种调用方式，因此在写 VC 程序时，建议使用 __stdcall 代替。

6. _thiscall ：这是 C++ 语言特有的一种调用方式，用于类成员函数的调用约定。如果参数确定，this 指针存放于 ECX 寄存器，函数自身清理堆栈；如果参数不确定，this指针在所有参数入栈后再入栈，调用者清理栈。__thiscall 不是关键字，程序员不能使用。参数按照从右至左的方式入栈。
   

## 成员函数调用

成员函数最终会被编译成与对象没有关系的普通函数。C++规定，编译成员函数时需要额外添加一个参数，就是对象指针。通过指针访问成员变量。

这样通过传递对象指针就完成了成员函数和成员变量的关联。这与我们从表明上看到的刚好相反，通过对象调用成员函数时，不是通过对象找函数，而是通过函数找对象。





[C/C++程序编译过程详解_Alex-L的博客-CSDN博客](https://blog.csdn.net/qq_40309341/article/details/113428490?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromBaidu~Rate-1.pc_relevant_paycolumn_v3&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromBaidu~Rate-1.pc_relevant_paycolumn_v3&utm_relevant_index=2)