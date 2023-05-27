---
layout:     post
title:      "Rocket Core源代码笔记——控制流"
subtitle:   "Rocket Core的控制流"
date:       2018-11-21 12:00:00
author:     "Readm"
header-img: "img/riscv.jpg"
tags:
    - 技术
    - RISCV
    - Rocket Core
    - Chisel
---

# Control Flow in Rocket Core

Today, we talk about the control flow instructions in Rocket Core, mainly the jumps. I'm not familiar with the CSR and other part, so we only talk the normal jump instructions. This part differs in different version, so please refer to your version.

## Jump or In Order?

Normally, the decode stage use the output of `ibuf` as the result of fetch stage. However, the `ibuf` read the instructions from `icache` (instruction cache). See [Overview of Rocker Pipeline](/docs/rocket-core.pdf). When there is no jump, `ibuf` will fetch the next instruction orderly from the `imem`.

If a jump comes, (or other situations that the pipeline do not want next instruction) a `kill` is send to the `ibuf`, which means clean instruction in the buffer, and read the new respond from the `imem`. The pc of the new instruction (where jump to) is in the request of the `imem`.

```
  io.imem.req.bits.pc :=
    Mux(wb_xcpt || csr.io.eret, csr.io.evec, // exception or [m|s]ret
    Mux(replay_wb,              wb_reg_pc,   // replay
                                mem_npc))    // flush or branch misprediction
```


Here, inclueds all the possible of control flow alter. We are not talking about the exception/[m|s]ret/replay. The normal jumps, will take the last line, i.e., the mem_npc. Remember that the call/ret are special jumps.

## Jump? When and Where?

Keys to anaylyze when and where to jump, are `take_pc` and `io.imem.req.bit.pc`. You can glance at [Overview](/docs/rocket-core.pdf) to get a general idea. 


### take_pc

As you can see in the [Overview](/docs/rocket-core.pdf), the `take_pc` leads to two results: one is enable the `imem.req`, another is kill the `ibuf`. That's comprehensible. If there is a jump, we need to read the new instructions and abandon the instruction buffer.

What will enable `take_pc`, `take_pc` = `take_pc_mem_wb` in current version. In the history, there was a `take_pc_id`, this will handle the jal instructions in the decode stage. But now, it was handled in frontend---faster. So now, `take_pc` means `take_pc_mem` or `take_pc_wb`.

If you trace from the `take_pc_mem`, you will find it will handle the 1) branch was taken 2) jal 3) jalr. The details differ when use BTB or not. So we do not dive into the code.

From `take_pc_wb := replay_wb || wb_xcpt || csr.io.eret || wb_reg_flush_pipe` we found that the `take_pc_wb` comes from 1) replay 2) exceptions 3) eret 4) flush pipe. Each of them is complicated, so we will disscuss them in the future.


### imem.req.bit.pc

As we mentioned before, the `imem.req.bits.pc` comes from `mem_npc` in jumps. `mem_npc` comes from two case: 

+ If it's a `jalr`, `mem_pc` equals `mem_reg_wdata`, which comes from the ALU.
+ If it's a branch/`jal`, `mem_pc` equals `mem_br_target`, which is `mem_reg_pc` added an offset.

This is reasonable: `jalr` uses the address in the register and use ALU, while `jal` and branch use the current pc and an offset.

### Others

For replay/exceptions, I may discuss them in the future. But CSR is not in my plan.