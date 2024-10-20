## 前言

PE文件是portable File Format（可移植文件）的简写，我们比较熟悉的DLL和exe文件都是PE文件。了解PE文件格式有助于加深对操作系统的理解，掌握可执行文件的数据结构机器运行机制，对于逆向破解，加壳等安全方面方面的工作极其重要。以下将详细介绍PE文件的格式。

## 基本概念

PE文件使用的是一个平面地址空间，所有代码和数据都被合并在一起，组成一个很大的组织结构。文件的内容分割为不同的区块（Setion，又称区段，节等），区段中包含代码数据，各个区块按照页边界来对齐，区块没有限制大小，是一个连续的结构。每块都有他自己在内存中的属性，比如：这个块是否可读可写，或者只读等等。

认识PE文件不是作为单一内存映射文件被装入内存是很重要的，windows加载器（PE加载器）便利PE文件并决定文件的哪个部分被映射，这种映射方式是将文件较高的偏移位置映射到较高的内存地址中。当磁盘的数据结构中寻找一些内容，那么几乎能在被装入到内存映射文件中找到相同的信息。但是数据之间的位置可能改变，其某项的偏移地址可能区别于原始的偏移位置，不管怎么样，所表现出来的信息都允许从磁盘文件到内存偏移的转换，如下图：

<img src="E:\Code\复习心得\res\picture\PE介绍.jpg" style="zoom:200%;" />

PE文件头以下的地址无论在内存映射中还是在磁盘映射中都是一样的，当内存分页和磁盘分页一致时无需进行地址转换，只有当磁盘分页和内存分页不一样时才要进行地址转化，这点很重要，拿到PE文件是首先查看分页是否一致。

## 重要概念

下面要介绍几个重要概念，分别是基地址（ImageBase）,相对虚拟地址（Relative Virtual Address），文件偏移地址（File Offset）

### 1）基地址

定义：当PE文件通过Windows加载器被装入内存后，内存中的版本被称作模块（Module）。映射文件的起始地址被称作模块句柄（hMoudule），可以通过模块句柄访问其他的数据结构。这个初始内存弟子就是基地址。

内存中的模块代表着进程从这个可执行文件中所需要的代码，数据，资源，输入表，输出表以及其他有用的数据结构所使用的内存都放在一个连续的内存块中，编程人员只要知道装载程序文件映像到内存的基地址即可。在32位系统中可以直接调用GetModuleHandle以取得指向DLL的指针，通过指针访问DLL module的内容，例如：

HMODULE GetmoduleHandle（LPCTSRT lpModuleName）；

当调用该函数时，传递一个可执行文件或者DLL文件名字字符串。如果系统找到该文件，则返回该可执行文件的或者DLL文件映像加载到的基地址。也可以调用GetModuleHandle,传递NULL参数，则返回调用的可执行文件的基地址。

### 2）相对虚拟地址

在可执行文件中，有相当多的地方需要指定内存的地址。例如：引用全局变量时，需要指定它的地址。PE文件尽管有一个首选的载入地址（基地址），但是他们可以载入到进程空间的任意地方，所以不能依赖与PE的载入点。由于这个原因，必须有一个方法来指定一个地址而不是依赖于PE载入点。

为了在PE文件中避免有确定的内存地址，出现了相对虚拟地址(Relative Virtual Addres,简称RVA)的概念。RVA只是内存中的一个简单的相对于PE文件装入地址的偏移地址，它是一个“相对”地址，或者称位“偏移量”地址。例如：假设一个EXE文件从地址40000h处载入，并且它的代码区块开始于4010000h，代码区的RVA将是：

目标地址401000h-----转入地址400000h则RVA=1000h。

将RVA地址转换成真实地址，只需简单的翻转这个过程：将实际装入地址加上RVA即可得到实际的内存地址。顺便一提，在PE用语里，实际的内存地址被称作虚拟地址(Vritual Address，简称VA)，另外也可以把虚拟地址想象为加上首选装入地址的RVA。不要忘了前面提到的装入地址等同于模块句柄，它们之间的关系如下：

虚拟地址(VA)=基地址(ImageBase)+相对虚拟地址(RVA)

### 3）文件偏移地址

当PE文件存储在磁盘上时，某个数据的位置相对于文件头的偏移量也称文件偏移地址（FileOffset）或者物理地址（RAW Offset）。文件偏移地址从PE文件的第一个字节开始计数，起始为零。用十六进制工具比如：winhex，hexworkshop都可以查看。注意这个物理地址和虚拟地址的区别，物理地址是文件在磁盘上相对于文件头的地址，而虚拟地址是PE可执行程序加载在内存中的地址。

## 几个重要头部信息介绍

接下来介绍MS-DOS头部信息，PE文件头信息及几个重要字段。

### 1）MS-DOS头部

每个PE文件是以一个DOS程序开始的，有了它，一旦程序在DOS下执行，DOS就能辨别出这是个有效的执行体，然后运行紧随MZ header（后面会介绍）之后的DOS stub(DOS块)。DOS stub实际上是一个有效的EXE，在不支持PE文件格式的操作系统中，它将简单显示一个错误提示，类似于字符串“This Program cannot be run in MS-DOS”。用户通常对DOS stub 不感兴趣，因为大多数情况下他们由汇编器自动生成。平常把DOS stub和DOS MZ头部合称为DOS文件头。

PE文件的第一个字节起始于一个传统的MS-DOS头部，被称作IMAGE_DOS_HEADER。其IMAGE_DOS_HEADER的结构如下（左边的数字是到文件头的偏移量）：

```
IMAGE_DOS_HEADER STRUCT 
{
+0h WORD e_magic       // Magic DOS signature MZ(4Dh 5Ah)      ->DOS可执行文件标记 
+2h WORD  e_cblp       // Bytes on last page of file
+4h WORD  e_cp         // Pages in file
+6h WORD  e_crlc       // Relocations
+8h WORD  e_cparhdr    // Size of header in paragraphs
+0ah WORD  e_minalloc  // Minimun extra paragraphs needs
+0ch WORD  e_maxalloc  // Maximun extra paragraphs needs
+0eh WORD  e_ss        // intial(relative)SS value             ->DOS代码的初始化堆栈SS
+10h WORD  e_sp        // intial SP value                      ->DOS代码的初始化堆栈指针SP 
+12h WORD  e_csum      // Checksum
+14h WORD  e_ip        // intial IP value                      ->DOS代码的初始化指令入口[指针IP] 
+16h WORD  e_cs        // intial(relative)CS value             ->DOS代码的初始堆栈入口 
+18h WORD  e_lfarlc    // File Address of relocation table
+1ah WORD  e_ovno      // Overlay number
+1ch WORD  e_res[4]    // Reserved words
+24h WORD  e_oemid     // OEM identifier(for e_oeminfo)
+26h WORD  e_oeminfo   // OEM information;e_oemid specific
+29h WORD  e_res2[10]  // Reserved words
+3ch DWORD e_lfanew    // Offset to start of PE header         ->指向PE文件头 
} IMAGE_DOS_HEADER ENDS
 
```

这个结构中有两字段很重要，一个是e_magic,一个是e_lfanew。e_magic（一个字大小）字段需要被设置为5A4Dh这个也是PE程序载入的重要标志，这个值非常有意思，他们对应的字符分别位Z和M，是为了纪念MS-DOS的最初创建者Mark Zbikowski而专门设置的，由于在hex编辑器中显示是由低位到高位故显示为4D5Ah，刚好是创建者的名字缩写。另一个字段是e_lfanew。这个字段表示的是真正的PE文件头部相对偏移地址（RVA），它指出了真正PE头部文件偏移位置。它占用四个字节，位于文件开始偏移的3ch字节中。

### 2）PE文件头文件

相对于MS-DOS头文件，PE头文件PEheader要复杂的多，下面将详细讲解其中的几个字段。

紧跟着DOS头文件下面的就是peheader。PEheader是PE相关结构NT映像头（IMAGE_NT_HEADER）的简称，其中包含许多PE装载器用到的重要字段。执行体在支持PE文件结构的操作系统执行时，PE装载器将IMAGE_DOS_HEADER结构中的e_fanew字段找到PEheader的起始偏移量，加上基址得到PE文件头的指针：

PNTHeader=IMAGBase+dosHeader->e_lfanewr(其实就是去字段e_lfanew的值)。

下面来讨论IMAGE_NT_HEADER的结构，它是由三个字段组成（左边的数字是PE文件头的偏移量）

```
IMAGE_NT_HEADER STRUCT
{
+0h Signature      DWORD               //PE文件标志
+4h FileHeader     IMAGE_FILE_HEADER   //文件头初始偏移地址
+18 optionalHeader IMAGE_OPTION_HEADER //另一个重要头部初始偏移地址
} IMAGE_NT_HEADER ENDS
```

