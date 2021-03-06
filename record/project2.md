1. 按照`Hint`填完所有函数后，出现了两个`Exception 13`，调试发现，这个是在`Switch_To_User_Context`函数中，调用了`Switch_To_Address_Space(kthread->userContext);`后导致，原来系统开始阶段，对内核态进程掉调用这个函数后，内核态进程的`userContext`是空，因此导致了空指针错误，在这里具体解决只需要加上一个条件判断就好。
2. 具体了解了一下，关于中断和异常的知识，整理如下：
    1. 中断分广义和狭义两种，指外部硬件设备发出的请求是狭义终端，CPU 内部计算出现的错误如除0、溢出等等叫作异常，异常和狭义的中断都是广义的中断
    2. 中断号是由CPU产生，然后根据中断号，在IDTR指向的中断表中找到处理函数，并进入。中断号中的前32位是固定的，其中13号终端也就是之前总是碰到的错误，叫作`General Protection Fault`， 主要是指内存访问错误，很多情况下是由于段寄存器的错误。一些中断发生后，CPU还会产生一个`error code`，13号中断的`error code`含义如下：若是异常原因是和段相关的，则`error code`为段选择子索引，否则为0。[来源](https://wiki.osdev.org/Exceptions)
3. 又重新读了一遍自己的代码，以及把项目其他的代码大概看了一遍，发现了很多问题，很多错误是由于低级的指针错误，大概修改了一下，发现异常已经转变成了12号异常，即堆栈异常。
4. 一开始以为是`Setup_User_Thread`中入栈顺序错误，然后试着改了一下，发现的确有错误，就是在压通用寄存器入栈时候，不用压sp入栈，只需要其他七个寄存器。不过写完后才发现，原来寄存器入栈和出栈的顺序就在`lowlevel.asm`的`RestoreRegister`函数中。以及`hacking.pdf`中讲了入栈的顺序。
5. 改正了寄存器的顺序后，发现异常仍然存在。发现了脚本文件中`eipToFunction`可以根据出错时候`eip`寄存器的值，定位到当时的函数。基本原理是，在编译内核时候，生成了一个`kernel.syms`文件，加载了内核文件中所有全局变量和函数的虚拟地址。不过此处的异常显示`eip`为`0x1000`,根据脚本找不到对应的函数。猜测应该是加载了`shell`程序后发生的错误，因此没有在内核文件中记载地址。
6. 看了一个教程，发现我在生成`User_Context`时候，还有一些错误。其中最关键的是，该结构体的`argBlockAddr`和`stackPointerAddr（栈顶指针）`指针指向的地址本因该都是虚拟地址，但我错误的将其赋为线性地址，于是导致栈顶越界，导致12号异常。修改之后，没有异常了。
7. 显示还需要实现系统调用。填完`Sys_PrintString`。填写`Sys_Spawn`时候，发现总有断言错误。查看教程发现，每次`Spawn`之前都要打开中断，之后要关闭中断。同样填完剩下的系统调用。
8. 填完之后，运行b程序，发现每次运行之后都没有办法退出，即运行完之后，只有运行的结果，但是不会显示新的一行命令提示符。这是以为在`Start_User_Thread`时候，在`detach`参数处，我选择了`false`，改为`true`之后可以退出了。
9. 输入按键12个以上按键，会突然抛出`GPF`异常，并显示异常发生在`Load_LDT`函数内。判断因该是在`Wait_For_Key`中。
10. 找不到原因，看了下教程，发现是在`Switch_To_User_Context`中，设置堆栈指针时候出错了。我的代码中设置的堆栈指针是`thread->esp`,正确的方式应该是`thread->stackPage + PAGE_SIZE`。
11. 做到这里，基本上没有太明显的BUG了。但是我还有很多疑问：
    1. 创建核态进程时候，(在`kthread.c::Create_Thread`)，分别为进程本身和进程的堆栈分配了一个页面大小的内存。在生成`User_Context`时候(`userseg.c::Load_User_Program`)，我为读取的elf文件中的每个段（这里的段是指elf文件的段，并不是代码段或数据段）分配了一段内存，同时为参数块和堆栈也都分配了内存。问题就是，前者的堆栈和后者分配的堆栈有什么关系？有什么区别？为什么有两个堆栈？以及进程本身的那一页内存有何用处？
        1. 区别1：前者是所有内核态进程都有的，后者是用户态进程独有的。我们知道，在这个系统里面，所有的用户态进程都是由内核态进程搭配上一个自己的`User_Context`构成的，用来划分自己的虚拟空间(或者叫逻辑空间),而内核态的所有进程有相同的代码段和数据段，分别由`GDT`中1、2号描述符所描述。
        2. 通过搜索得知，进程一般有两个堆栈，即内核栈和用户栈，分别在内核态和用户态时使用

    2. 在加载elf文件时候，对于所有的段，并没有区分类型，而是直接将所有段并在一起（按照elf文件规定的逻辑地址，并不代表相连），成为一个大的地址空间，而生成的代码段和数据段都是这个一整个地址空间。换句话说，我们生成的代码段和数据段从地址到界限都是一样的，他们都指向了同一片内存区域。但是我们知道，在elf文件中有专门的`.text`,`.data`段，分别是代码和数据。为什么这里是这样的？是否后面会有变化？否则似乎分段意义不大？

    3. 在写8086汇编时候，我们在每一个程序的前面都定义了数据段、堆栈段、代码段，但是加载elf文件的时候，好像没有见到堆栈段。那我们在汇编代码中定义的堆栈段是怎么起作用的？