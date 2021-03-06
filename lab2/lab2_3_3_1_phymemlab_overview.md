### 实验执行流程概述

本次实验主要完成 ucore 内核对物理内存的管理工作。参考 ucore 总控函数 kern_init 的代码，可以清楚地看到在调用完成物理内存初始化的 pmm_init 函数之前和之后，是已有 lab1 实验的工作，好像没啥修改。其实不然，ucore 有两个方面的扩展。首先，bootloader 的工作有增加，在 bootloader 中，完成了对物理内存资源的探测工作（可进一步参阅附录 A 和附录 B），让 ucore
kernel 在后续执行中能够基于 bootloader 探测出的物理内存情况进行物理内存管理初始化工作。其次，bootloader 不像 lab1 那样，直接调用 kern_init 函数，而是先调用位于 lab2/kern/init/entry.S 中的 kern_entry 函数。kern_entry 函数的主要任务是为执行 kern_init 建立一个良好的 C 语言运行环境（设置堆栈），而且临时建立了一个段映射关系，为之后建立分页机制的过程做一个准备（细节在 3.5 小节有进一步阐述）。完成这些工作后，才调用 kern_init 函数。

kern_init 函数在完成一些输出并对 lab1 实验结果的检查后，将进入物理内存管理初始化的工作，即调用 pmm_init 函数完成物理内存的管理，这也是我们 lab2 的内容。接着是执行中断和异常相关的初始化工作，即调用 pic_init 函数和 idt_init 函数等，这些工作与 lab1 的中断异常初始化工作的内容是相同的。

为了完成物理内存管理，这里首先需要探测可用的物理内存资源；了解到物理内存位于什么地方，有多大之后，就以固定页面大小来划分整个物理内存空间，并准备以此为最小内存分配单位来管理整个物理内存，管理在内核运行过程中每页内存，设定其可用状态（free 的，used 的，还是 reserved 的），这其实就对应了我们在课本上讲到的连续内存分配概念和原理的具体实现；接着 ucore
kernel 就要建立页表，
启动分页机制，让 CPU 的 MMU 把预先建立好的页表中的页表项读入到 TLB 中，根据页表项描述的虚拟页（Page）与物理页帧（Page
Frame）的对应关系完成 CPU 对内存的读、写和执行操作。这一部分其实就对应了我们在课本上讲到内存映射、页表、多级页表等概念和原理的具体实现。

在代码分析上，建议根据执行流程来直接看源代码，并可采用 GDB 源码调试的手段来动态地分析 ucore 的执行过程。内存管理相关的总体控制函数是 pmm_init 函数，它完成的主要工作包括：

1. 初始化物理内存页管理器框架 pmm_manager；
2. 建立空闲的 page 链表，这样就可以分配以页（4KB）为单位的空闲内存了；
3. 检查物理内存页分配算法；
4. 为确保切换到分页机制后，代码能够正常执行，先建立一个临时二级页表；
5. 建立一一映射关系的二级页表；
6. 使能分页机制；
7. 从新设置全局段描述符表；
8. 取消临时二级页表；
9. 检查页表建立是否正确；
10. 通过自映射机制完成页表的打印输出（这部分是扩展知识）

另外，主要注意的相关代码内容包括：

- boot/bootasm.S 中探测内存部分（从 probe_memory 到 finish_probe 的代码）；
- 管理每个物理页的 Page 数据结构（在 mm/memlayout.h 中），这个数据结构也是实现连续物理内存分配算法的关键数据结构，可通过此数据结构来完成空闲块的链接和信息存储，而基于这个数据结构的管理物理页数组起始地址就是全局变量 pages，具体初始化此数组的函数位于 page_init 函数中；
- 用于实现连续物理内存分配算法的物理内存页管理器框架 pmm_manager，这个数据结构定义了实现内存分配算法的关键函数指针，而同学需要完成这些函数的具体实现；
- 设定二级页表和建立页表项以完成虚实地址映射关系，这与硬件相关，且用到不少内联函数，源代码相对难懂一些。具体完成页表和页表项建立的重要函数是 boot_map_segment 函数，而 get_pte 函数是完成虚实映射关键的关键。
