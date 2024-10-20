## 一、内存分配的原理

## 1、概念

从操作系统角度来看，进程动态分配内存有两种方式，分别由两个系统调用完成：brk和mmap（不考虑共享内存）。在标准C库中，提供了malloc/free函数分配释放内存，这两个函数底层是由brk，mmap，munmap这些系统调用实现的。

- **brk：**将数据段(.data)的最高地址指针_edata往高地址推；
- **mmap：**在进程的虚拟地址空间中（堆和栈中间，称为文件映射区域的地方）找一块空闲的虚拟内存。

## 2、流程

- 当开辟的空间**小于 128K** 时，调用 brk()函数，malloc 的底层实现是系统调用函数 brk())，其主要移动指针 _enddata(此时的 _enddata 指的是 Linux 地址空间中堆段的末尾地址，不是数据段的末尾地址)
- 当开辟的空间**大于 128K** 时，mmap()系统调用函数来在虚拟地址空间中（堆和栈中间，称为“文件映射区域”的地方）找一块空间来开辟。

这两种方式分配的都是虚拟内存，没有分配物理内存。在第一次访问已分配的虚拟地址空间的时候，发生缺页中断，操作系统负责分配物理内存，然后建立虚拟内存和物理内存之间的映射关系。

### 情况一、小于128k

malloc小于128k的内存，使用brk分配内存，将_edata往高地址推(只分配虚拟空间，不对应物理内存(因此没有初始化)，第一次读/写数据时，引起内核缺页中断，内核才分配对应的物理内存，然后虚拟地址空间建立映射关系)

![](E:\Code\复习心得\res\picture\内存分配01.jpg)

1、进程启动的时候，其（虚拟）内存空间的初始布局如图 1-(1)所示。

```c++
其中，mmap内存映射文件是在堆和栈的中间（例如libc-2.2.93.so，其它数据文件等），为了简单起见，省略了内存映射文件。
_edata指针（glibc里面定义）指向数据段的最高地址。 
```

2、进程调用A=malloc(30K)以后，内存空间如图1-(2)：

malloc函数会调用brk系统调用，将_edata指针往高地址推30K，就完成虚拟内存分配。

你可能会问：只要把_edata+30K就完成内存分配了？

事实是这样的，_edata+30K只是完成虚拟地址的分配，A这块内存现在还是没有物理页与之对应的，等到进程第一次读写A这块内存的时候，发生缺页中断，这个时候，内核才分配A这块内存对应的物理页。也就是说，如果用malloc分配了A这块内容，然后从来不访问它，那么，A对应的物理页是不会被分配的。 

3、进程调用B=malloc(40K)以后，内存空间如图1-(3)。 

### 情况二、大于128k

malloc大于128k的内存，使用mmap分配内存，在堆和栈之间找一块空闲内存分配(对应独立内存，而且初始化为0)，如图：

![](E:\Code\复习心得\res\picture\内存分配02.jpg)

4、进程调用C=malloc(200K)以后，内存空间如图2-(4)；

默认情况下，malloc函数分配内存，如果请求内存大于128K（可由M_MMAP_THRESHOLD选项调节），那就不是去推_edata指针了，而是利用mmap系统调用，从堆和栈的中间分配一块虚拟内存。这样子做主要是因为brk分配的内存需要等到高地址内存释放以后才能释放（例如，在B释放之前，A是不可能释放的，这就是内存碎片产生的原因，什么时候紧缩看下面），而mmap分配的内存可以单独释放。当然，还有其它的好处，也有坏处，再具体下去，有兴趣的同学可以去看glibc里面malloc的代码了。 

5、进程调用D=malloc(100K)以后，内存空间如图2-(5)；

6、进程调用free(C)以后，C对应的虚拟内存和物理内存一起释放2-(6)；

![](E:\Code\复习心得\res\picture\内存分配03.jpg)

7、进程调用free(B)以后，如图3-(7)；

B对应的虚拟内存和物理内存都没有释放，因为只有一个_edata指针，如果往回推，那么D这块内存怎么办呢？当然，B这块内存，是可以重用的，如果这个时候再来一个40K的请求，那么malloc很可能就把B这块内存返回回去了。 

8、进程调用free(D)以后，如图3-(8)；

把B和D连接起来，变成一块140K的空闲内存。

**合并空闲块**：在释放内存块后，如果不进行合并，那么相邻的空闲内存块还是相当于两个内存块，会形成一种假碎片。所以当释放内存后，我们需要将两个相邻的内存块进行合并。

9、默认情况下：

当最高地址空间的空闲内存超过128K（可由M_TRIM_THRESHOLD选项调节）时，执行**内存紧缩操作（trim）**。在上一个步骤free的时候，发现最高地址空闲内存超过128K，于是内存紧缩，变成图3-(9)所示。

## 2）如何进行物理分配

当一个进程发生缺页中断的时候，进程会进入内核态，执行以下操作：

1）检查要访问的虚拟地址是否合法

2）查找/分配一个物理页

3）填充物理页内容（读取磁盘，或者直接置0，或者什么都不做）

4）建立映射关系（虚拟地址到物理地址的映射关系）

5）重复执行发生缺页中断的那条指令

如果第3步，需要读取磁盘，那么这次缺页就是 majfit(major fault：大错误),否则就是 minflt(minor fault：小错误)

## 二、malloc的实现方案

1）malloc 函数的实质是它有一个将可用的内存块连接为一个长长的列表的所谓空闲链表。

2）调用 malloc（）函数时，它沿着连接表寻找一个大到足以满足用户请求所需要的内存块。 然后，将该内存块一分为二（一块的大小与用户申请的大小相等，另一块的大小就是剩下来的字节）。 接下来，将分配给用户的那块内存存储区域传给用户，并将剩下的那块（如果有的话）返回到链表上。

3）调用 free 函数时，它将用户释放的内存块连接到空闲链表上。

4）到最后，空闲链会被切成很多的小内存片段，如果这时用户申请一个大的内存片段， 那么空闲链表上可能没有可以满足用户要求的片段了。于是，malloc（）函数请求延时，并开始在空闲链表上搜索各内存片段，对它们进行内存整理，将相邻的小空闲块合并成较大的内存块。

### 搜索空闲块常见的算法

搜索空闲块最常见的算法有：首次适配，下一次适配，最佳适配。

1. **首次适配**：从头部搜索空闲链，选择第一个合适的内存块，找到足够大的内存块就分配，这种方法会产生很多的内存碎片。

   优点是将大的空闲块保留在链表的后面，缺点是在靠近链表起始处留下了小空间碎片，增大了查找大空间的时间。

2. **下一次适配**：和首次适配相似，但是从上次查询结束的地方开始搜索。

   优点是比首次适配速度快一些，缺点是空间利用率比首次适配低。

3. **最佳适配**：对堆进行彻底的搜索，从头开始，遍历所有块，使用数据区大于size且差值最小的块作为此次分配的块。

   优点是对存储器的利用率高，缺点是时间消耗大。

## 三、malloc分配空间

malloc()到底从哪里得到了内存空间？答案是从堆里面获得空间。也就是说函数返回的指针是指向堆里面的一块内存。操作系统中有一个记录空闲内存地址的链表。当操作系统收到程序的申请时，就会遍历该链表，然后就寻找第一个空间大于所申请空间的堆结点，然后就将该结点从空闲结点链表中删除，并将该结点的空间分配给程序。

malloc()在运行期动态分配分配内存,free()释放由其分配的内存。malloc()在分配用户传入的大小的时候，还分配的一个相关的用于管理的额外内存，不过，用户是看不到的。所以**实际大小 = 管理空间 + 用户空间**。

## 管理空间

```
/**内存控制块数据结构，用于管理所有的内存块
* is_available: 标志着该块是否可用。1表示可用，0表示不可用
* size: 该块的大小
**/
struct mem_control_block {
    int is_available;
    int size;
};
```

## 用户空间

堆中的内存块总是成块分配的，并不是申请多少字节，就拿出多少个字节的内存来提供使用。堆中内存块的大小通常与内存对齐有关8Byte(for 32bit system)或16Byte(for 64bit system)。

因此，在64位系统下，当(申请内存大小+sizeof(struct mem_control_block) )% 16 == 0的时候，刚好可以完成一次满额的分配，但是当其!=0的时候，就会多分配内存块。

## 四、free

看完malloc的实现，也会很容易的想到free()函数的实现思路，只要将内存管理块is_available设置为可用就可以了。这样下次调用malloc()函数的时候就可以将该内存块作为可分配块再次进行分配了。

## 五、代码

### malloc实现

```
/**内存控制块数据结构，用于管理所有的内存块
* is_available: 标志着该块是否可用。1表示可用，0表示不可用
* size: 该块的大小
**/
struct mem_control_block {
    int is_available;
    int size;
};

/**在实现malloc时要用到linux下的全局变量
*managed_memory_start：该指针指向进程的堆底，也就是堆中的第一个内存块
*last_valid_address：该指针指向进程的堆顶，也就是堆中最后一个内存块的末地址
**/
void *managed_memory_start;
void *last_valid_address;

/**malloc()功能是动态的分配一块满足参数要求的内存块
*numbytes：该参数表明要申请多大的内存空间
*返回值：函数执行结束后将返回满足参数要求的内存块首地址，要是没有分配成功则返回NULL
**/
void *malloc(size_t numbytes) {
    //游标，指向当前的内存块
    void *current_location;
    //保存当前内存块的内存控制结构
    struct mem_control_block *current_location_mcb;
    //保存满足条件的内存块的地址用于函数返回
    void *memory_location;
    memory_location = NULL;
    //计算内存块的实际大小，也就是函数参数指定的大小+内存控制块的大小
    numbytes = numbytes + sizeof(struct mem_control_block);
    //利用全局变量得到堆中的第一个内存块的地址
    current_location = managed_memory_start;

    //对堆中的内存块进行遍历，找合适的内存块
    while (current_location != last_valid_address) //检查是否遍历到堆顶了
    {
        //取得当前内存块的内存控制结构
        current_location_mcb = (struct mem_control_block*)current_location;
        //判断该块是否可用
        if (current_location_mcb->is_available)
            //检查该块大小是否满足
            if (current_location_mcb->size >= numbytes)
            {
                //满足的块将其标志为不可用
                current_location_mcb->is_available = 0;
                //得到该块的地址，结束遍历
                memory_location = current_location;
                break;
            }
        //取得下一个内存块
        current_location = current_location + current_location_mcb->size;
    }

    //在堆中已有的内存块中没有找到满足条件的内存块时执行下面的函数
    if (!memory_location)
    {
        //向操作系统申请新的内存块
        if (sbrk(numbytes) == -1)
            return NULL;//申请失败，说明系统没有可用内存
        memory_location = last_valid_address;
        last_valid_address = last_valid_address + numbytes;
        current_location_mcb = (struct mem_control_block)memory_location;
        current_location_mcb->is_available = 0;
        current_location_mcb->size = numbytes;
    }
    //到此已经得到所要的内存块，现在要做的是越过内存控制块返回内存块的首地址
    memory_location = memory_location + sizeof(struct mem_control_block);
    return memory_location;
}
```

### free实现

```
/**free()功能是将参数指向的内存块进行释放
*firstbyte：要释放的内存块首地址
*返回值：空
**/
void free(void *firstbyte)
{
    struct mem_control_block *mcb;
    //取得该块的内存控制块的首地址
    mcb = firstbyte - sizeof(struct mem_control_block);
    //将该块标志设为可用
    mcb->is_available = 1;
    return;
}
```

