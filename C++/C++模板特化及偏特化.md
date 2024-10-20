如果一个模板类拥有一个以上的template参数，可以对其某个（或者多个，但并非全部）进行特化工作。也就是将某些template指定具体的类型。所谓的偏特化提供零一分template定义，但其本质上还是templatized。



偏特化是实现traits的重要支持。

```c++
template <class T>
struct iterator_traits{
	typedef typename T::value_type value_type;
}
```

如果类T有定义自己的value_type，那自然是可以获得T::value_type，如果是原生指针就不可以了，需要使用特化来实现

```c++
template <class T>
struct iterator_traits<int>{
	typedef typename int value_type;
}
```

