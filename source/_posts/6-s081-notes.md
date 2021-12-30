---
title: 6.S081 笔记
date: 2021-11-16 15:24:10
tags:
	- 操作系统
    - C
    - 公开课
categories:
    - 课程学习
---

记录一下学习 MIT 6.S081 与 xv6 book 时遇到的一些问题。

<!-- more -->

## LEC 1: Introduction

1. 为什么系统调用 `exec` 不直接和 `fork` 做成一块。因为要考虑到管道这种操作的情况，管道需要把前一个程序的标准输出定向到后一个程序的标准输入（使用 close、dup 等操作），由于子进程的文件描述符表和父进程独立，这样先 `fork` 再在子进程中 `exec` 就可以直接不用管后续了。如果做成了 `forkexec` 这种，如果遇到执行失败返回了原本的程序，需要再恢复文件描述符的对应，这样的操作比较繁琐，可扩展性差。

## LEC 4: Page Table

1. 在 RISC-V 中内存管理单元（MMU）并不会保存完整的页表，它会通过寄存器 SATP 找到当前运行的进程的页表，SATP 中存放的是页表在内存中的地址（实际物理地址），只有 SATP 设置了 RISC-V CPU 才会启用 MMU。当不同进程进行切换时，也会更新 SATP。
2. 三级页表中的 PPN 都是实际的物理地址。

3. 为什么硬件（RISC-V）实现了 MMU 的功能，我们还是要在内核代码中实现 walk 这样对虚拟内存翻译的功能？我的理解是，硬件自己实现的 MMU 可以帮助硬件在执行机器代码时可以通过 xv6 写入的页表进行映射，比如 C 语言代码编写的一些非系统调用的操作，比如 C 语言代码中的直接通过某地址访问内存，这个就是虚拟地址，而且就可以靠硬件自己的 MMU 了。而实现 walk 是因为内核需要实现页表的可编程，可以直接通过访问物理内存来实现一些操作（比如一些系统调用），比如直接从指定物理地址提取数据或向指定物理地址写入数据，要不然只能通过机器指令（汇编）获取物理内存中的内容。而且后续课程会提到的 Page Fault 等操作也需要靠这样可编程的页表来操作。另外值得提的一点是，内核在初始化时会将自己的页表构造为和物理地址一对一的映射（direct-mapping），所以内核程序执行时访问的内存地址就是实际的物理地址。

## LEC 6: Isolation & system call entry/exit

1. 这一章主要就是程序陷入 trap 的整个过程。以系统调用为例。系统调用都是通过 `ecall` 这个 RISC 指令进入，`ecall` 会触发一种 trap，并会设置好 `STVEC` （也就中断向量）、`SEPC` 等寄存器。在 xv6 里中断向量会指向 `trampoline.S` 的 `uservec` 函数，这里会进行一波 trap 之前的处理。然后接下来就是 `uservec` -> `usertrap` -> `usertrapret` -> `userret`，从进入中断向量，到中断处理，再到返回用户代码。中断处理的逻辑都在 `usertrap`，在后续的 lab 中基本都是在这里面进行操作。

## LEC 8: Page faults

1. 这一章的内容相对简单，主要是讲了现代操作系统利用 page fault 可以做一些什么样的操作。基本都是利用 page fault 做一些内存相关的 lazy 的操作。也就是对于内存的分配，不是一开始就分配好的，而是通过 page fault 的触发来现分配内存，这样可以节约创建进程的开销（但是会带来对内存页写的开销）。主要思想就是，分配了内存地址范围之后，并不分配实际内存，当访存失败触发 page fault 时再分配内存。COW fork，demand paging 等都依靠这种方式实现。
2. 在 COW 的 Lab 中有一些小细节需要注意，这个地方害得我 cowtest 中的 file 一直遇到问题（见下面代码块）。还有一个小的注意点是，在 `copyonwrite` 中，内存页 copy 之后需要对原物理地址页调用一次 `kfree` 来减去引用计数（或者单独写一个函数来控制引用计数，我这里是融合到了其他函数中），不然会出现内存未被正确清除干净的问题。

    ```c
    int
    copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
    {
      uint64 n, va0, pa0;
    
      while(len > 0){
        va0 = PGROUNDDOWN(dstva);
        pa0 = walkaddr(pagetable, va0);
        if(pa0 == 0)
          return -1;
    	
        // 这里不能直接使用 PA2PTE(pa0),
        // 因为这样 pte 不带标志位，必须要重新 walk
        pte_t* pte = walk(pagetable, va0, 0); 
        if (*pte & PTE_COW) {
          if (copyonwrite(pagetable, va0) != 0) {
            return -1;
          }
          // 这里也必须重新 walk，因为 copyonwrite 中可能重新分配了地址
          pa0 = walkaddr(pagetable, va0);
        }
    
        n = PGSIZE - (dstva - va0);
        if(n > len)
          n = len;
        memmove((void *)(pa0 + (dstva - va0)), src, n);
    
        len -= n;
        src += n;
        dstva = va0 + PGSIZE;
      }
      return 0;
    }
    ```

## LEC 11: Thread switching

1. 做 lab 的时候有一个问题稍微困扰了一下我。在 `thread_create` 的时候使用 `t->ra = (uint64) func` 将线程中要运行的函数的地址保存在了 `t->ra` 中，这样在 `thread_switch` 结束的时候我可以返回到正确的位置。我的疑惑在于要怎么保存线程运行到的地方呢，比如 `thread_a` 执行到某一个行的时候进行了 `yield` 让出了 CPU，那回到 `thread_a` 后要怎么回到 `yield` 的位置呢，因为 `pc` 并没有保存，`ra` 中存的又是 `func` 的初始位置，这岂不是每次都回 `thread_a` 的时候都要从 `func` 的一开始执行？这里其实是我想错了，因为 `ra` 并不是一成不变的，当执行 `yield` 等函数的时候，会在线程的栈中（也就是 `t->stack` 中）记录好各种信息，退出函数的时候会从 `t->stack` 中拿到返回地址等信息，返回地址会被打入 `ra` 中。所以以 `thread_a` 为例，在下一次 `thread_switch` 的时候，存到 `thread_a` 上下文信息中的 `ra` 已经不是最开始的 `func` 的地址了，而是执行 `thread_switch` 时的 `ra`。进程切换、线程切换就这里比较绕，必须以汇编的思维来思考每一个函数、每一个指令的执行逻辑，必要时可以以 gdb 为辅助一步一步观察来理解整个过程。
2. lab 的 `Using threads` 部分有一个小点，就是用一个全局大锁锁住所有操作没法通过 `ph_fast` 测试，优化方法为给每一个 bucket 一个锁，分别锁自己的，这样才能通过 `ph_fast` 测试。