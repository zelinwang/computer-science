## 一、类的元数据

程序调用moc生成对应的moc_xxx.cpp文件，在文件中生成了类的元对象信息

```c++
const QMetaObject Object::staticMetaObject = {
    { &QObject::staticMetaObject, qt_meta_stringdata_Object,
      qt_meta_data_Object, qt_static_metacall, nullptr, nulllptr }
};
```

QObject::staticMetaObject表示基类的元数据

qt_meta_stringdata_Object的=表示元数据的签名标记，类型为QByteArrayData，

qt_meta_data_Object表示元数据的索引数组的指针，类型为uint[]，

qt_static_metacall表示一个指向xxx::qt_static_metacall 的函数指针， xxxx内存布局已经包含了静态成员staticMetaObject和qt_static_metacall成员函数。  

```c++
const QMetaObject *xxx::metaObject() const
{
	return QObject::d_ptr->metaObject ? QObject::d_ptr->dynamicMetaObject : &staticMetaObject;
}

```

 QObject::d_ptr->metaObject仅供动态元对象（QML对象）使用，所以一般而言，虚函数 metaObject() 仅返回类的 staticMetaObject。 

## 二、元数据表

在生成的moc_xxx.cpp文件中会有对象的元数据以及内省表

### 1、内省表

类型为uint[],

```
static const uint qt_meta_data_Object[] = {}

内省表是一个 uint 数组，分为五个部分：第一部分content，即内容，分为9行。第一行revision，指MOC生成代码的版本号（Qt4 是6，Qt5则是7）。第二个classname，即类名，该值是一个索引，指向字符串表的某一个位置（本例中就是第0位）。
```

## 三、SIGNAL与SLOT宏

Qt的信号和槽是通过SIGNAL和SLOT宏将信号和槽的定义转换为字符串，实际上这个宏是通过文本替换将信号和槽以及编码，头文件信息，行号等拼接在一起，文件和行号是通过qFlagLocation（）进行获取，并且将槽编码为1，信号编码为2，。经过预编译后的文件xxx.i中信号和槽的格式如下

信号 "2"  "signal_text(class)" "\0" "xxx.h" "." "line"

槽     "1"  "slot_text(class)" "\0" "xxx.h" "." "line"

## 四、信号的实现

通过moc生成的moc_xxx.cpp文件中实现了信号，创建了一个指向参数的指针的数组，如

```
void class::signal_method(argv _t)
{
    void *_a[] = { 0, const_cast<void*>(reinterpret_cast<const void*>(&_t)) };
    QMetaObject::activate(this, &staticMetaObject, 0, _a);
}
```

并且将数组传递给QMetaObject::activate，数组_a的第一个数据为0，表示返回值为void。传给activate的第三个参数是指信号的索引，表示该信号在类中的索引。

## 五、槽的调用

 利用槽函数在qt_static_metacall 函数的索引位置来调用槽函数： 

## 六、元对象中的索引

在每一个QMetaObject对象中，槽、信号以及其它的对象可调用函数都会分配一个从0开始的索引。索引是有顺序的，信号在第一位，槽在第二位，最后是其它函数。这个索引在内部被称为相对索引，不包含父对象的索引位。
为了实现包含在继承链中其它函数的索引，在相对索引的基础上添加一个偏移量，得到绝对索引。绝对索引是在公开API中使用的索引，由QMetaObject::indexOf(Signal, Slot, Method) 类似的函数返回。
连接机制使用以信号为索引的向量。但是在向量中，所有的槽也会占有一定空间，通常在一个对象中，槽的数量要比信号多。所以从 Qt 4.6开始，使用的是一种仅包含信号索引的新的内部实现。



## 七、信号和槽的连接

开始连接时，Qt所要做的第一件事是找出所需要的信号和槽的索引。Qt会去查找元对象的字符串表来找出相应的索引。然后，创建一个 QObjectPrivate::Connection 对象，将其添加到内部的链表中。
由于允许多个槽连接到同一个信号，需要为每一个信号添加一个已连接的槽的列表。每一个连接都必须包含接收对象和槽的索引。在接收对象销毁的时候，相应的连接也能够被自动销毁。所以每一个接收对象都需要知道谁连接到它自己，以便能够清理连接。 

每一个QObject对象都有一个连接链表容器QObjectConnectionListVector *connectionLists：将每一个信号与一个 QObjectPrivate::Connection 的链表关联起来。
QObject::connect函数的实现如下： 



QObject::connect函数的主要功能是在接受者对象的元对象中将槽函数的相对索引找到，在接受者对象的元对象中将槽函数的相对索引找到，最后调用QMetaObjectPrivate::connect将信号与槽进行连接。QObject及其派生类对象的元对象在创建时就有一个QObjectConnectionListVector连接链表容器，QObject::connect的作用就是将新的连接加入到信号发送者附属的元对象的连接链表容器的相应信号的连接链表中（一个信号可能连接多个槽函数）。 

## 八、元对象系统对Qt进行了扩展

元对象系统提供了的两个技术，元对象和内省

经过moc编译的qt类，会生成对应的moc_XXX.cpp文件

该文件中包含了静态成员函数staticMetaObject和 qt对象 的元对象信息，以及内省