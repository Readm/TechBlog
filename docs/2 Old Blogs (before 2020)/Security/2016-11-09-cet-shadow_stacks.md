---
layout:     post
title:      "Intel官方手册：CET技术预览-影子栈"
subtitle:   "翻译自Control-flow Enforcement Technology Preview中第二章，影子栈。"
date:       2016-11-09 12:00:00
author:     "Readm"
header-img: "img/post-bg-2015.jpg"
tags:
    - 技术
    - 安全
    - 影子栈
    - CET
    - CFI
    - Shadow Stacks
---

# 2 影子栈

影子栈是控制流转移指令专用的第二个栈。这个栈区别于数据栈。这个栈并不是用于存储数据的，因此其不能够被软件直接地写入。对影子栈的写入被限制为控制流转移指令和影子栈操作指令。影子栈特性可以分别地在用户态（CPL==3）和特权态（CPL<3）启用。

影子栈操作只在开启页保护模式下启用。影子栈在virtual 8086模式下不能使用。

## 2.1 影子栈指针SSP和其操作和Address Size属性

当CET被开启并且处理器支持影子栈时时，处理器支持一个新的寄存器，影子栈寄存器SSP。SSP不能直接地在指令中被编码为源操作数，目标操作数或者内存操作。SSP指向当前影子栈顶。

SSP保存线性地址，并通过FAR RET，IRET和ROTORS指令加载到寄存器中。SSP必须装载到一个32位对齐的线性地址。

影子堆栈的宽度在32位/兼容模式下为32位，在64位模式下为64位。影子堆栈的地址大小属性在32位/兼容模式下为32位，在64位模式下为64位。

## 2.2 术语

当启用影子堆栈时，控制传输指令/流和影子堆栈管理指令会执行加载/存储到影子堆栈。 来自控制传送指令和影子堆栈管理指令的这种加载/存储被称为shadow_stack_load和shadow_stack_store，以区别于由诸如MOV，XSAVES等的其它指令执行的加载/存储。


用于指令操作的伪代码使用符号ShadowStackEnabled（CPL）作为在CPL处是否启用阴影堆栈的测试。 该术语返回TRUE或FALSE指示，如下所示：

```
ShadowStackEnabled(CPL):
    IF CR4.CET = 1 AND CR0.PE = 1 AND EFLAGS.VM = 0
        IF CPL = 3
            THEN
                (* Obtain the shadow stack enable from MSR used to enable feature for CPL = 3 *)
                SHADOW_STACK_ENABLED = IA32_U_CET.SH_STK_EN;
            ELSE
                (* Obtain the shadow stack enable from MSR used to enable feature for CPL < 3 *)
                SHADOW_STACK_ENABLED = IA32_S_CET.SH_STK_EN;
            FI;
        IF SHADOW_STACK_ENABLED = 1
            THEN
                return TRUE;
            ELSE
                return FALSE;
        FI;
    ELSE
        (* Shadow stacks not enabled in real mode and virtual-8086 mode or if the master CET feature enable in CR4 is disabled *)
        return FALSE;
    ENDIF
```

## 2.3 影子栈启用下的Near CALL和RET的行为

当启用影子堆栈时，Near CALL压入返回地址到数据堆栈和影子堆栈上；Near RET 从影子堆栈和数据堆栈弹出返回地址。 如果指定了可选的“n”操作数，则数据堆栈指针（ESP / RSP）可选地进一步增加“n”个字节，但是影子堆栈指针（SSP）不递增。如果从两个堆栈弹出的返回地址不相同，那么处理器会导致#CP（near-ret）异常。

## 2.4 Far CALL和RET

CALL指令可以用于调用位于与当前代码段不同的段中的过程，或者调用处于不同特权级别的段。

对于Far CALL，处理器压入影子堆栈上的CS，LIP（返回地址的线性地址）和SSP，并在远RET上从影子堆栈中弹出SSP，LIP和CS。如果CS和LIP不匹配通过从数据栈弹出CS和EIP确定的返回地址，则处理器导致#CP（FAR-RET / IRET）异常。


Far CALL到更高权限级别的影子堆栈行为如下：

+ 当Far CALL发起于CPL3时，返回地址不会压入到特权影子堆栈。同样，从特权级级（CPL <3）到CPL3的Far RET不对返回地址进行任何验证。 在CPL3 - > CPL <3转换时，用户空间SSP保存到MSR - IA32_PL3_SSP，并在CPL <3 - > CPL3转换从该MSR恢复。
+ 在特权级间CALL时，CALL指令执行栈交换。特权级的数据栈位置在当前TSS中。同样地，影子堆栈也以这种方式被切换。根据目标特权级别，从以下MSR之一获得用于特权程序的SSP
    + IA32_PL2_SSP为切换到ring2
    + IA32_PL1_SSP为切换到ring1
    + IA32_PL0_SSP为切换到ring0
+ 从ring2到ring1，ring2到ring0或从ring1到ring0的远程调用被认为是影子栈的“相同特权级”传输。 因此，在针对新的特权级定位SSP之后的这种Far CALL将调用过程的CS，LIP和SSP压入到被调用过程的影子堆栈上。 同样，Far RET将验证来自影子栈的CS和LIP与从数据栈获得的CS和EIP确定的返回地址匹配。

在跨特权远CALL上，CET在创建要用于这些传输的影子堆栈时，验证由特权用户设置的“特权影子堆栈令牌（supervisor shadow stack token）”。 特权影子栈令牌是一个64位值，结构如下：

+ Bit 63:3 – “supervisor shadow stack token”的线性地址。
+ Bit 2 - 保留。必须置零。
+ Bit 1 - 保留。必须置零。
+ Bit 0 - Busy Bit。如果为0，则次影子栈不被任何逻辑处理器激活。

下图说明了一个特权影子堆栈，位于其“特权影子堆栈令牌”基址上：

```
+--------------------+          +--------------------------+
|IA32_PLx_SSP = 0xFF8|  ----->  |  <Next push saves here>  |
+--------------------+          +--------------------------+
                                |0xFF8 | (EFER.LMA &  CS.L)|
                                +--------------------------+
```

IA32_PLx_SSP中指定的地址需要为8字节对齐。在切换到编码到IA32_PLx_SSP中的特权影子堆栈之前，处理器执行以下检查。这些步骤以原子方式进行。

1. 使用shadow_stack_load从IA32_PLx_SSP中指定的地址加载管理器影子堆栈令牌。
2. Busy Bit必须为0.
3. 在MSR中编码的地址必须与“特权影子堆栈令牌”中的地址匹配。
4. 如果检查2和3成功，则使用shadow_stack_store设置令牌中的忙位，并将SSP切换到IA32_PLx_SSP中指定的值。
5. 如果检查2或3失败，则Busy bit不置位，并产生#GP（0）异常

在Far RET中，指令清除影子堆栈令牌中的忙位如下。 这些步骤也以原子方式执行：

1. 使用shadow_stack_load从SSP加载内核影子堆栈令牌。
2. 检查Busy bit是否为1.
3. 检查编码为“特权影子堆栈令牌”的地址是否与SSP匹配。
4. 如果检查2和3成功，然后使用影子堆栈存储清除令牌中的忙位，否则继续，而不修改SSP指向的影子堆栈的内容。（译者注：而不产生异常？）

这里描述的操作也适用于当通过IDT中的中断/陷阱门调用中断或异常处理程序时执行的远程调用。 同样，IRET指令的行为类似于Far RET指令。

## 2.5 在64位模式下调用中断/异常处理程序时栈的切换

64位模式操作提供了称为中断堆栈表（IST）的栈切换机制，其中当不存在涉及的特权改变时，64位IDT描述符可用于指定64位TSS中的7个数据堆栈指针中的一个作为调用的一部分。 如果指定的IST索引为0，并且没有涉及的特权更改，则会对同一堆栈发生栈切换。

为了支持这种栈切换机制，影子栈特征提供了MSR，IA32_INTERRUPT_SSP_TABLE，以编写7个影子栈指针的表的线性地址。当指定了非零IST值并且没有涉及作为调用的一部分的特权更改时，MSR指向使用IST索引索引的内存中的64字节表。

```
                                    +----------------------------+
                                    |     IST7 SSP      |Offset 7|
                                    +-------------------+--------+
                                    |     IST6 SSP      |Offset 6|
                                    +-------------------+--------+
                                    |     IST5 SSP      |Offset 5|
                                    +-------------------+--------+
                                    |     IST4 SSP      |Offset 4|
                                    +-------------------+--------+
                                    |     IST3 SSP      |Offset 3|
                                    +-------------------+--------+
                                    |     IST2 SSP      |Offset 2|
                                    +-------------------+--------+
                                    |     IST1 SSP      |Offset 1|
+------------------------+          +-------------------+--------+
|IA32_INTERRUPT_SSP_TABLE|  ----->  |Not used. abailable|Offset 0|
+------------------------+          +----------------------------+
```

## 2.6 任务切换下的影子栈

任务切换可以通过以下方式来调用：

+ JMP和CALL到GDT中的TSS描述符
+ JMP或CALL到GDT或当前LDT中的任务门描述符
+ 中断或异常向量指向IDT中的任务门描述符

启用影子栈后，新任务必须与32位TSS相关联，且不能处于虚拟8086模式。用于新任务的32位SSP位于32位TSS中的偏移104处。因此，新任务的TSS必须至少为108字节。此SSP需要为8字节对齐，并且需要指向“特权影子堆栈令牌”（尽管任务可能在CPL3）。

在由CALL指令发起的嵌套任务切换中，旧任务的SSP不保存到旧任务TSS。相反，旧任务的SSP连同旧任务的CS和LIP一起被压入到新任务的影子堆栈上。同样，在由IRET发起的非嵌套任务切换中，新任务的SSP从旧任务的影子栈中恢复。旧任务的影子堆栈上的CS和LIP与由新任务的CS和EIP确定的返回地址匹配。如果匹配失败，则报告#CP（FAR-RET / IRET）异常。
