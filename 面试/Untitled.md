假设一个class A; A *p = new A; free(A);这里不用delete，而是用new，可能会发生什么情况 

  答：由于free并不会去调用析构函数，因此析构函数中做的一些额外操作，比如指针指向的堆里分配的内存，就无法释放（还有继承中父类的析构、包含其他类的其他类的析构也不会调到） 

  考官答：除了上面的，还有new中不一定能调用的是malloc分配的内存；（尤其是某些debug的版本），因此直接使用free可能会有问题； 另外new返回的指针不一定就是malloc的指针，比如new出来的对象的大小是3，编译器做对齐，可能会将指针往后移动一位，返回之后的那个位置；（C++11 新引入操作符alignof， 对齐描述符alignas，基本对齐值 alignof(std::max_align_t)）

作者：yxluffy
链接：https://www.nowcoder.com/discuss/459282?source_id=discuss_experience_nctrack&channel=-1
来源：牛客网



追问：上面说到用于函数外的变量，改变external、internal属性，这个具体编译器如何实现，比如一个文件中有一个external的变量，还有一个internal的变量，别的文件怎么知道用的是哪个？ 

  追答：猜测，[c++]()中有namespace，可能是external的话在一个全局的namespace中，每个文件有每个文件的一个namespace，如果是internal的话，就放在文件中的namespace下； 其他文件访问的时候，如果不带namespace，就直接去全局的namespace下找到这个符号 

  面试官追答：也可以这么用；实际上是给这些符号加一些限制名称；



函数调用栈分布、ELF结构、手写递归[排序](https://www.nowcoder.com/jump/super-jump/word?word=排序)





作者：Lyon
链接：https://www.nowcoder.com/discuss/427797?source_id=discuss_experience_nctrack&channel=-1
来源：牛客网



\1. [红黑树]()最差旋转几次 

 \2. tcp udp有什么不同 

 \3. 关系型数据库是什么 

 \4. 一些[c++]()的基本知识 

 \5. [c++]()11的特性有哪些 

 \6. 挑了一个你没说到的问你知不知道(我没提到的constexpr) 

 \7. 做一道dp的题(选数问题)、一道智力题 

 \8. 你说到啥就问啥。比如问你工作中做到的一个东西，扯到怎么实现的，说到map，问map是什么，说到[红黑树]()，问[红黑树]()最差情况是啥样 

 \9. 手撕两道题，一个镜像反转[二叉树]()，一个实现smart point。 

 \10. 问到mfc里怎么实现的消息传递，message map怎么实现的。 

 然后手撕一个log分析的代码(其实是让写思路)。



写了一个[平衡二叉树](https://www.nowcoder.com/jump/super-jump/word?word=平衡二叉树)转双向[链表](https://www.nowcoder.com/jump/super-jump/word?word=链表)和一道[动态规划](https://www.nowcoder.com/jump/super-jump/word?word=动态规划)。

switch的case里为何不能定义变量

\5. windows创建完成队列的参数及意义， 

 \6. 对比windows完成队列和linux epoll的区别， 

 \7. 是linux的信号和线程的关系。 

 \8. qq的对话框怎么实现；qt中某个具体函数参数

 第一个问的比较高级的c++的知识，涉及到了stl库中map的实现，有默认值参数的同名虚函数是否也能实现多态等等。