---
layout:     post
title:      "如何在编译器中保留特定寄存器"
subtitle:   "在LLVM中保留一个特殊寄存器作为专用寄存器，需要哪些改动。"
date:       2018-04-11 12:00:00
author:     "Readm"
header-img: "img/tech.jpg"
tags:
    - 技术
    - 编译器
    - LLVM
    - GLIBC
---

最近的工作中，设计中需要一个专用的寄存器，仅供我自己使用。在现有编译器和库中当然没有预留，这需要我们更改编译器和库。

## 目的

在目前的X86寄存器中，选取若干寄存器作为专用寄存器，保留起来不被其他功能使用。

## 前置知识

+ 调用惯例
+ X86寄存器结构
+ 寄存器分配算法

### 调用惯例

在不同的操作系统中有不同的二进制接口（ABI），最主要就是调用惯例。调用惯例中规定了在函数调用中，哪些寄存器用于传递参数，哪些寄存器保持不变(callee saved registers/Preserved Registers)，哪些寄存器会被改变(caller saved registers/Scratch Registers)。这些信息对我们很重要，因为**传递参数的寄存器是无法预留的，除非你改动整个系统的ABI**。

对于其他寄存器，是否保持不变就看你的需求了：这取决于你如何计划库对你的寄存器的操作。

一般，你的编译器只编译你的源文件，而库文件直接使用共享库。这意味着你的程序会调用到共享库，如果你选定了Preserved Register，那么库文件就不会改动你的寄存器（这可以使你直接做到共享库的兼容而不需要重新编译库）；反之，你的寄存器会被库文件破坏。

但值得注意的是：并不总是从你的源文件调用库再返回的。类似于：

```
your_fun0() --call--> your_fun1() --call--> lib_fun0() --call--> lib_fun1()
```

一些情况下，也会出现交叉的调用：

```
your_fun0() --call--> lib_fun0() --call--> your_fun1() --call--> lib_fun1()
```

因为库不改动你的寄存器只是对于调用者说的，所以对于your_fun0来说，lib_fun0()会保持所有Preserved Registers，但在其调用your_func1()时，Preserved Registers仍然可能是被它更改的（在他退出时恢复之前）。

所以，**最保险的做法还是无论是否是Scratch Registers，都重新编译库来保证库不使用它们。**

查询这些信息，可以参考[Call Conventions](http://www.agner.org/optimize/calling_conventions.pdf) [(本站备份)](/docs/calling_conventions.pdf)。

### X86寄存器结构

X86的寄存器结构比较复杂，不同CPU也不同。这里只是提醒如果你的数据较大，通用寄存器可能难以保存，不妨尝试向量寄存器（XMM，YMM等）这在[Cryptographically enforced control flow integrity](https://arxiv.org/abs/1408.1451)中有过使用。

但向量寄存器的缺点在于，难于学习和使用，因为版本不同和用途各异，正确的使用是需要时间学习的。**混用不同版本或者作用域的指令可能得到正确的值，但极大影响性能。**

### 寄存器分配算法

编译器中（LLVM）在编译的前端使用虚拟寄存器来表示变量，只有在后端时才将虚拟寄存器映射到物理寄存器中。我们知道这个过程就足够了，我们需要的是在这个映射过程中，将我们需要的寄存器标记为保留寄存器，不被分配就可以了。

在[文档](https://llvm.org/docs/WritingAnLLVMBackend.html#defining-a-register-class)这一节的最后提到，只要将寄存器标记为reserved，一般情况下就不会被使用到，在我们的观察中也基本是这样的。即使使用到，也会马上帮你恢复回来。

## LLVM中预留

假设你通过上面的知识选定好了使用哪个寄存器来预留，我们就可以开始在LLVM中预留寄存器了。

在LLVM中预留寄存器非常简单，在文件`/lib/Target/X86/X86RegisterInfo.cpp`中：

```
BitVector X86RegisterInfo::getReservedRegs(const MachineFunction &MF) const {
    ......
}
```

这里有各种Reserved的寄存器。包括简单的

```
  // Mark the segment registers as reserved.
  Reserved.set(X86::CS);
  Reserved.set(X86::SS);
  Reserved.set(X86::DS);
  Reserved.set(X86::ES);
  Reserved.set(X86::FS);
  Reserved.set(X86::GS);
```

和稍微复杂的：

```
if (!Is64Bit) {
    // These 8-bit registers are part of the x86-64 extension even though their
    // super-registers are old 32-bits.
    Reserved.set(X86::SIL);
    Reserved.set(X86::DIL);
    Reserved.set(X86::BPL);
    Reserved.set(X86::SPL);

    for (unsigned n = 0; n != 8; ++n) {
      // R8, R9, ...
      for (MCRegAliasIterator AI(X86::R8 + n, this, true); AI.isValid(); ++AI)
        Reserved.set(*AI);

      // XMM8, XMM9, ...
      for (MCRegAliasIterator AI(X86::XMM8 + n, this, true); AI.isValid(); ++AI)
        Reserved.set(*AI);
    }
  }
```

这里 MCRegAliasIterator 的部分实际上是将有别名的寄存器都标记为Reserved，请注意这个问题，如果你的寄存器也有别名，也要采用类似的处理。

这里我们也看到，LLVM在编译X86_32的程序时，仅存在于64位的寄存器reserve起来了。

## LLVM中使用

在LLVM中直接使用这些寄存器（不然你预留干嘛），正常的方法应该用Read_register和Write_register的内联函数来做，但尴尬的是我一直没有成功，应该是一些低级错误。

所以我一直使用的是嵌入汇编的方法，这样并不好，会增加不必要的指令。

## 在Glibc库中预留

之后我们来考虑如何避免共享库使用这些寄存器。方法就是重新编译共享库并且使用这些共享库来运行我们的程序。

为什么是重新编译Glibc？因为它是最常用的，至少，SPEC中使用的都在Glibc中。这对于写论文是足够有效的了，如果你的程序还需要其他的库，那就也重新编译那些库。

编译的方法非常简单，因为这些库基本都使用gcc编译，而gcc中排除特定寄存器太简单了，例如`-ffixed-xmm15`就可以将XMM15寄存器排除到编译中。其他的寄存器也是一样的，类似于`-ffixed-r12 -ffixed-r13 -ffixed-r14`。当然，用于传参的寄存器是不可能避免的。

如何重新编译和使用Glibc请参照[官网](http://www.gnu.org/software/libc/started.html)。这里描述了如何下载和重新编译Glibc的库。我建议使用非安装的方式来使用，避免你的系统中其他程序被你的新的共享库搞崩。

在Glibc中添加前面的编译选项有两处：

+ 在源码文件夹中`Makeconfig` line324:
    + default_cflages中添加上述参数
+ 在Build文件夹中`configure.make` line 109:
    + CFLAGS中添加上述参数

这样再Make后的库就是满足要求的库了。


## 总结

到这里，我们就完全预留好了需要的寄存器，如果你没有使用这些寄存器，在库中和你编译的二进制文件中都不会使用到这些寄存器。

为什么不使用更加优美的方式呢，例如在LLVM中设置一个全局变量并且绑定一个寄存器？

答案是老子不会，这可能涉及到函数间的寄存器分配，我尝试了一下并没有什么头绪。有人这样实现的话可以告诉我膜拜一下:)