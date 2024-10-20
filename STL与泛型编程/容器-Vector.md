emplace_back和push_back的区别

一、本质

C++11及之前，emplace_back是模板函数，而push_back是成员函数，C++11之后，push_back实际上是调用emplace_back，且emplace_back在C++11之前返回值是void，而在C++14返回的是引用。

二、效率

C++11中push_back传递的参数是引用，在实际执行过程中，需要执行一次拷贝动作。

emplace_back使用的万能引用，在插入点构造，相比push_back能更好地避免内存的拷贝与移动，使容器插入元素的性能得到进一步提升。

三、注意

```c++
struct TS
{
    TS(int a) : _m(a) { }
    int _m;
};

class B
{
public:
    B(int a) : _m(a){}
private:
    int _m;
};

int main()
{
    std::vector<TS> vts;
    vts.push_back({0});            //正确
    vts.emplace_back({2});         //错误
    vts.emplace_back<TS>({1});     //正确

    std::vector<B> VCB;
    VCB.push_back({0});//正确
    VCB.emplace_back({2});//错误
    VCB.emplace_back<B>({1});//正确
}

```

emplace_back在使用的时候，如果容器成员是结构，需要指定模板参数，否则编译错误。