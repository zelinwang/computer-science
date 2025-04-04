## STL相关问题

### 一、vector的底层（存储）机制

vector就是一个动态数组，里面有一个指针指向一片连续的内存空间，当空间不够装下数据时，会自动申请另一片更大的空间（一般是增加当前容量的50%或100%），然后把原来的数据拷贝过去，接着释放原来的那片空间；当释放或者删除里面的数据时，其存储空间不释放，仅仅是清空了里面的数据。

### 二、vector的自增长机制

当已经分配的空间不够装下数据时，分配双倍于当前容量的存储区，把当前的值拷贝到新分配的内存中，并释放原来的内存。
对vector的任何操作，一旦引起空间重新配置，指向原vector的所有迭代器就都失效了。

### 三、list的底层（存储）机制

list使用双向链表存储数据，结点的地址在内存中不一定连续，每次插入或者删除一个元素，就配置或者释放一个元素空间。

### 四、什么情况下用vector，什么情况下用list

vector可以随机存储元素（即可以通过公式直接计算出元素地址，而不需要挨个查找），访问的时间复杂度是O(1)，但是非尾部插入删除数据时，效率很低，适合对象简单，对象数量变化不大，随机访问频繁。
list不支持随机存储，适用于对象大，对象数量变化频繁，插入和删除频繁的场景。插入和删除的时间复杂度是O(1)。

### 五、list自带排序函数的排序原理

通过归并算法实现，将前2个元素合并，再将后2个元素合并，然后合并这2个子序列成为4个元素序列的子序列，重复这一过程，得到8个，16个。。。子序列，最后得到的就是排序后的序列。
时间复杂度O(log n)

### 六、deque的底层机制

deque动态地以分段连续空间组合而成，随时可以增加一段新的连续空间并链接起来，不提供空间保留功能。
注意：除非必要，我们尽可能选择使用vector而非deque，因为deque的迭代器比vector迭代器复杂很多。对duque的排序，为了提高效率，可先将deque复制到一个vector上排序，然后再复制到deque。
deque采用一块map（不是STL的map容器）作为主控，其为一小块连续空间，其中每个元素都是指针，指向另一段较大的连续空间（缓冲区）。

deque的迭代器包含4个内容：

1. cur：迭代器当前所指元素
2. first：此迭代器所指的缓冲区的头。

3. last：缓冲区尾。

4. node：指向管控中心。


### 七、deque与vector的区别

1. vector是单向开口的连续线性空间，deque是双向开口的连续线程空间。
2. deque没有提供空间保留功能，vector则要提供空间保留功能。
3. deque也提供随机访问迭代器，但是其迭代器比vector复杂很多

### 八、map底层机制，查找复杂度，能不能边遍历边插入

红黑树，自平衡的二叉搜索树。自动排序效果不错。通过map的迭代器不能修改key，只能修改value值。
查找复杂度：O(logN)
不可以，map不像vector，它在容器进行erase操作后不会返回后一个元素的迭代器，不能遍历往后插删。

### 九、vector插入删除和list有什么区别

vector插入删除数据，需要对现有数据复制移动，如果vector存储的对象很大或者构造函数很复杂，则开销很大，如果是简单的小数据，效率优于list
list插入和删除数据，需要对现有数据进行遍历，但要在首部插入数据，效率很高。

### 十、hashtable如何避免地址冲突

#### 一）开放定址法

这种方法也称**再散列法**，基本思想是一旦发生了冲突，就去寻找下一个空的散列地址，只要散列表足够大，空的散列地址总能找到，并将记录存入。开放地址法又有三种算法支持

1. **线性探测法**

   如果计算出哈希的位置为H，但是H占用，那么就会通过H+1，H+2，H+3...的方法来查找空闲的地址，用来存放数据

2. **二次探测法**

   如果计算出哈希的位置为H，但是H占用，那么就会通过H+pow(1,2)，H+pow(2,2)，H+pow(3,2)...的方法来查找空闲的地址，用来存放数据

3. **随机探测法**

   具体实现时，应建立一个伪随机数发生器，（如i=(i+p) % m），并给定一个随机数做起点。

**优点：** 

- 记录更容易进行序列化（serialize）操作 
- 如果记录总数可以预知，可以创建完美哈希函数，此时处理数据的效率是非常高的

**缺点：**

- 存储记录的数目不能超过桶数组的长度，如果超过就需要扩容，而扩容会导致某次操作的时间成本飙升，这在实时或者交互式应用中可能会是一个严重的缺陷
- 使用探测序列，有可能其计算的时间成本过高，导致哈希表的处理性能降低 
- 由于记录是存放在桶数组中的，而桶数组必然存在空槽，所以当记录本身尺寸（size）很大并且记录总数规模很大时，空槽占用的空间会导致明显的内存浪费
- 删除记录时，比较麻烦。比如需要删除记录a，记录b是在a之后插入桶数组的，但是和记录a有冲突，是通过探测序列再次跳转找到的地址，所以如果直接删除a，a的位置变为空槽，而空槽是查询记录失败的终止条件，这样会导致记录b在a的位置重新插入数据前不可见，所以不能直接删除a，而是设置删除标记。这就需要额外的空间和操作。

#### 二）再哈希法

同时构造多个不同的哈希函数，若第一个哈希函数计算的地址产生冲突后，会使用第二个哈希函数，直至没有冲突为止。

#### 三）开链法

在每一个表格元素中维护一个list，哈希函数分配一个list，将冲突的数据添加到list中，若list的长度超过8，就会将list变为红黑树，降低访问时间

**优点：**

- 对于记录总数频繁可变的情况，处理的比较好（也就是避免了动态调整的开销）
- 由于记录存储在结点中，而结点是动态分配，不会造成内存的浪费，所以尤其适合那种记录本身尺寸（size）很大的情况，因为此时指针的开销可以忽略不计了
- 删除记录时，比较方便，直接通过指针操作即可

**缺点：**

- 存储的记录是随机分布在内存中的，这样在查询记录时，相比结构紧凑的数据类型（比如数组），哈希表的跳转访问会带来额外的时间开销 
- 如果所有的 key-value 对是可以提前预知，并之后不会发生变化时（即不允许插入和删除），可以人为创建一个不会产生冲突的完美哈希函数（perfect hash function），此时封闭散列的性能将远高于开放散列
- 由于使用指针，记录不容易进行序列化（serialize）操作

#### 四）建立公共溢出区

这种方法的基本思想是：将哈希表分为基本表和溢出表两部分，凡是和基本表发生冲突的元素，一律填入溢出表

### 十一、hashtable、hash_set、hash_map的区别

　　hash_set以hashtable为底层，不具有排序功能，能快速查找，其键值就是实值。（set以红黑树为底层，具有排序功能）
　　hash_map以hashtable为底层，没有自动排序功能，能快速查找，每一个元素同时拥有一个实值和键值。（map以红黑树为底层，有排序功能）

### 十二、unordered_map与map的区别、什么时候用它们两个？

**构造函数：**unordered_map需要hash function 和等于函数，而 map需要比较函数（大小或小于）
**存储结构：**unordered_map以hashtable为底层，而map以红黑树为底层。
**总的来说**，unordered_map查找速度比map快，而且查找速度基本和数据量大小无关，属于常数级别。而map的查找速度是logN级别。但不一定常数就比log小，而且hash_map还有hash function耗时。
如果考虑效率，特别当元素达到一定数量级时，用hash_map；考虑内存，或者元素数量较少时，用map

选用map还是hash_map，关键是看关键字查询操作次数，以及你所需要保证的是查询总体时间还是单个查询的时间。如果是要很多次操作，要求其整体效率，那么使用hash_map，平均处理时间短。如果是少数次的操作，使用 hash_map可能造成不确定的O(N)，那么使用平均处理时间相对较慢、单次处理时间恒定的map，考虑整体稳定性应该要高于整体效率，因为前提在操作次数较少。如果在一次流程中，使用hash_map的少数操作产生一个最坏情况O(N)，那么hash_map的优势也因此丧尽了。当你需要低内存占用率，当你希望序列是有序的，当你需要稳定的表现时用map；当有一个很好的哈希函数和对内存占用率没有限制时使用unordered_map。

### 十三、红黑树有什么性质？

1. 每个节点都是红色或黑色
2. 根节点为黑色
3. 叶节点为黑色的NULL结点。
4. 如果结点为红，其子节点必须为黑
5. 任一节点到NULL的任何路径，所含黑结点数必须相同

### 十四、map和set的3个问题

1. 为何map和set的插入删除效率比其他序列高
   因为不需要内存拷贝和移动
2. 为何map和set每次insert之后，以前保存的iterator不会失效？
   因为插入操作只是结点指针换来换去，结点内存没有改变，而iterator就像指向结点的指针，内存没变，指向内存的指针也不会变。
3. 当数据元素增多时，（从10000到20000），map、set查找速度会怎样变化？
   红黑树用二分查找法，时间复杂度为logN，所以查找次数从log100000=14变为log20000=15,多了1次而已。

### 十五、为什么vector插入操作可能导致迭代器失效

vector动态增加大小时，并不是在原空间后增加新的空间，而是以原大小的两倍在另外配置一个较大的新空间，然后将内容拷贝过来，并释放原来的空间，由于操作改变了空间，所以迭代器失效。
但vector一般底层是用数组实现的，仔细考虑数组的特性，得出结论：

1. insert时，假设insert位置为p，分2种情况：

   a)容器还有空余空间，不重新分配内存，p前的迭代器有效，p后的失效。
   b)重分内存，p之后的迭代器失效

2. erase时，假设erase位置为p，则p之前的迭代器都有效，并且p指向下一个元素位置，（如果之前p在尾巴上，则p指向无效尾end），p之后的迭代器都无效。

### 十六、hashtable和hashmap的区别

hashmap以hashtable为底层。有以下不同：

1. hashtable是Dictionary的子类，而hashmap是Map接口的一个实现类
2. hashtable中的方法是同步的，而hashmap的方法不同步

### 十七、STL底层数据结构实现

1. **vector：**数组，支持快速随机访问
2. **list：**双向链表，支持快速增删
3. **deque：**双端队列，一个中央控制器和多个缓冲区，支持首尾（中间不能）快速增删，支持随机访问
4. **stack：**底层用deque或者list实现，不用vector的原因是扩容耗时
5. **queue ：**底层用deque或者list实现，不用vector的原因是扩容耗时
6. **priority_queue：**优先队列，一般以vector为底层容器，heap为处理规则来管理底层容器实现
7. **set：**红黑树 有序，不重复
8. **multiset ：**红黑树 有序，可重复
9. **map：**红黑树 有序，不重复
10. **multimpap：**红黑树 有序，可重复
11. **unordered_set：**hash表 无序，不重复
12. **unordered_map：**hash表 ，无序，不重复
13. **hashtable：**底层结构为vector

### 十八、STL提供哪六大组件？

1. 容器
   就是各种数据结构
   序列式容器：array、vector、heap、priority_queue、list、slist、deque、stack、queue
   关联式容器：set、map、multiset、multimap、hashtable、hash_set、hash_map、hash_multiset、hash_multimap
2. 算法
   各种常见算法：sort、search、copy、erase等，值得学习的有，sort、next_permutation、partition、merge sort 等
3. 迭代器
   容器与算法之间的胶合剂，是所谓的“泛型指针”。共有五种类型，迭代器是一种将operator*，operator->,operator++,operator–-等指针相关操作予以重载的class template。所有的STL容器多附带有自己专属的迭代器。只有容器设计者才知道如何设计迭代器，源生指针也是一种迭代器。是设计模式中的一种。
4. 仿函数
   行为类函数，可作为算法的某种策略，从实现角度看，仿函数是一种重载了operator()的class或class template。一般函数指针可视为狭义的仿函数。
5. 配接器
   一种用来修饰容器或者仿函数或迭代器接口的东西。比如queue和stack，看着像容器，其实就是deque包了一层皮。
6. 配置器
   复制空间配置与管理，从实现角度看，配置器就是实现了动态空间配置、空间管理、空间释放的classtemplate。

### 十九、auto_ptr可以做vector的元素吗？为什么？

不能。因为STL的标准容器规定它所容纳的元素必须是可以**拷贝构造**和可被**转移赋值**的。而auto_ptr不能，可以用shared_ptr智能指针代替。

### 二十、简单说一下next_permutation和partition的实现？

1.next_permutation(下一个排列）
　　首先，从最尾端开始往前寻找2个相邻元素，令第一个元素为i，第二个元素为I，且满足i< I，找到这样一组相邻元素后，再从尾端开始向前检验，找出第一个大于i的元素j，将i和j对调，再将ii之后的所有元素颠倒排列，此即所求“下一个”排列组合。
2.partion：
　　令头端迭代器first向尾部移动，尾部迭代器last向头部移动。当first所指的值大于或等于枢轴时就停下来，当last所指的值小于或等于枢轴时也停下来，然后检验两个迭代器是否交错。如果first仍然在last左边，就将连着元素互换，然后各自调整一个位置（向中央逼近），再继续进行相同的行为。如果发现两个迭代器叫错了，表示整个序列已经调整完毕。

### 二十一、不允许有遍历行为的容器有哪些？（不提供迭代器）

1.queue ，除了头部外，没有其他方法存取deque的其他元素。
2.stack(底层以deque实现),除了最顶端外，没有任何方法可以存取stack的其他元素。
3.heap，所有元素都必须遵循特别的排序规则，不提供遍历功能。

### 二十一、优先队列(priority_queue)

优先队列不是按照普通对象先进先出原FIFO则进行数据操作，其中的元素有优先级属性，优先级高的元素先出队。