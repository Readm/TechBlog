---
layout:     post
title:      "Rocket Core源代码笔记——旁路"
subtitle:   "Rocket Core的旁路简介"
date:       2018-03-15 12:00:00
author:     "Readm"
header-img: "img/riscv.jpg"
tags:
    - 技术
    - RISCV
    - Rocket Core
    - Chisel
---

*Note:为了可能的参与[rocket-chip-read](https://github.com/cnrv/rocket-chip-read)，之后的文章尽量直接使用英文，减少工作量。*

# Bypass in Rocket Core

Today we start with a simple part of Rocket Core: bypass.

Bypass in rocket core is faily simple, since the Rocket is an in-order pipelined core. The bypass module only detect bypass opportunites and bypass the correct values befor the execute stage.

## If not bypass

Before we discuss about the bypass, we first review the data path without bypass.

Assuming there is no data hazard, the data in the registers will be loaded by `rf.read`. In `RocketCore.scala`, two registers are read together as follows:

```
val id_rs = id_raddr.map(rf.read _)
```

*Note: the id_raddr only contains id_raddr1 and id_raddr2, without id_raddr3.*

Here, `id_rs` is a sequence contains the values of the two source registersa. If there is no bypass in this instruction, the values will goto `ex_rs(2)` after bypass. After that the value will be sent to ALU or DIV.

## Bypass opportunites

How to know whether there are bypass opportunites in the pipeline or not? The answer is "the stage will write a register". `bypass_sources` lists all possible bypass opportunites and the corresponding stages.

```
  val bypass_sources = IndexedSeq(
    (Bool(true), UInt(0), UInt(0)), // treat reading x0 as a bypass
    (ex_reg_valid && ex_ctrl.wxd, ex_waddr, mem_reg_wdata),
    (mem_reg_valid && mem_ctrl.wxd && !mem_ctrl.mem, mem_waddr, wb_reg_wdata),
    (mem_reg_valid && mem_ctrl.wxd, mem_waddr, dcache_bypass_data))
  val id_bypass_src = id_raddr.map(raddr => bypass_sources.map(s => s._1 && s._2 === raddr))
  ```

 In each item, 
 + first argument(eg:`ex_reg_valid && ex_ctrl.wxd`), represents whether the source is valid;
 + second argument(eg:`ex_waddr`), is the tag indicating the target register of this source;
 + last argument(eg:`mem_reg_wdata`), is the location of the value in the next cycle.

For example, the 2nd item means: when `ex_reg_valid && ex_ctrl.wxd`, this source can be used in bypass, the register is indicated by `ex_waddr`, and in the next cycyle, the value is in `mem_reg_wdata`.

The first item in `bypass_sources` is quiet easy: Always(when true), the x0 can be bypassed as 0.

In `id_bypass_src`, each read addr in id_raddr will decide which bypass source to use by comparing the `raddr` and `waddr`. 

## Bypass Datapath

Now we know which register can be bypassed, the next aim is to mark it and pass the value into execute stage. *Note: In the code, you need to read not only the beginning of the execute stage, but also code around `do_bypass` and `bypass_src`*

First, for each read addr, `do_bypass` decide whether the register can do bypass, `bypass_src` decide which value is used to do bypass. Both values are from `id_bypass_src`.

```
    val do_bypass = id_bypass_src(i).reduce(_||_)
    val bypass_src = PriorityEncoder(id_bypass_src(i))
```

In addition, each origin value in `id_rs` is devided into two parts: `ex_reg_rs_msb` and `ex_reg_rs_lsb`. And:
+ When do not bypass: correct values are `Cat(ex_reg_rs_msb, ex_reg_rs_lsb)`, which is consistent with it in `id_rs`;
+ When do bypass: correct values are selected by `ex_reg_rs_lsb` from `bypass_mux`, which is a list of the last arguments of each item in `bypass_sources`. In this case `ex_reg_rs_msb` is useless.

## Finally

Finally, **the correct values of the source register is in the `ex_rs(2)`**.

For example, you are adding an instruction reading registers and do not plan to use ALU or DIV, where are the correct values in the pipeline?

The answer is `ex_rs`, not the `id_rs`, since the `id_rs` are the values in the RegFile, without considering the bypass. 





