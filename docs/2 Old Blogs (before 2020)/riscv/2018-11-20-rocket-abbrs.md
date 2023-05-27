---
layout:     post
title:      "Rocket Core源代码笔记——缩写"
subtitle:   "Rocket Core的命名中的缩写"
date:       2018-11-20 12:00:00
author:     "Readm"
header-img: "img/riscv.jpg"
tags:
    - 技术
    - RISCV
    - Rocket Core
    - Chisel
---


# Note: Abbreviation in Rocke Core

Understanding naming is important to get started. Here we introduce some common abbreviations.  The normal abbreviations like inst=instruction are not included.

+ ibuf: Instrucion Buffer (Fetch)
+ id: Instruction Decode
+ ex: Execute
+ mem: Memory Access
+ wb: Write Back
+ ctrl: Control
+ raddr1/2/3, rs1/2/3: Read Address (register number)
+ waddr, rd: Write Address (register number)
+ rf: Register File
+ imem: Instruction Cache
+ dmem: Date Cache
+ *d: * Decode (killd = kill decode)
+ *x: * Execute (killx = kill execute)
+ *m: * Memory (killm = kill memory)
+ ren: Read Enable
+ op1/2: Operand
+ xcpt: Exception
+ br: Branch
+ cfi: Control Flow Instruction
+ npc: next PC
+ ll: Long Lantency
+ icache/dcache: instruction/data cache

The lines in the decoded bits will also help, see `IDecode.scala`. `*_ctrl` contains the signals of all the bits. For example, `ex_ctrl.wxd` is True means the instruction in the execute stage is an instruction which will write a register.

