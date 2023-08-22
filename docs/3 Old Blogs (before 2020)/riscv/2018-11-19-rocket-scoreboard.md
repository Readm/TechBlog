---
layout:     post
title:      "Rocket Core源代码笔记——Scoreboard"
subtitle:   "Rocket Core的计分板"
date:       2018-11-19 12:00:00
author:     "Readm"
header-img: "img/riscv.jpg"
tags:
    - 技术
    - RISCV
    - Rocket Core
    - Chisel
---

# Scoreboard in Rocket Core

Today we talk about the [Scoreboard](https://en.wikipedia.org/wiki/Scoreboarding) in Rocket Core. In fact, the Score Board in Rocke Core is very simple: 32 bit to record the status of each register. The class Scoreboard defined a class of this simple score board and the two instance is `sboard` for normal registers and `fp_sboard` for float point registers.

## How does it work?

The class Scoreboard is a little bit convoluted. In fact, you could understand it as a array of bit, with a size of 32. Each bit stands for one register. For example, the `sboard.set(True, 5)` will set the 5th bit to "busy". Similarly, `sboard.clear(True, 5)` will set the 5th bit to "not busy". `sboard.read(5)` will return whether the 5th register is busy or not. The function `readBypassed(rd)` seems to return the result of next cycle. But not used in current version.

## When to update the scoreboard?

To discuss when to update the scoreboard, we should know the two types of the "write into the registers": 

A) In general pipeline, (let's say `add`), when the instrucion come into the write back stage, the result of the `add` is **ready** for the writing. So the value be written to the register in the write back stage. In this senario, the score board will **not** be used. 

B) Another type of "write into registers" is, the "ll" in the code (which means "long lantency"). The "ll" write includes:

+ Multiplier instructions
+ Memory instructions and cache miss
+ ROCC instructions

In the pipeline of these instrucions, the value cannot get ready before the writeback stage. Consequently, the register should be marked as "busy" to avoid hazards. Here, we need to use scoreboard. You can see the logic from these code:

```
val wb_set_sboard = wb_ctrl.div || wb_dcache_miss || wb_ctrl.rocc
...
sboard.set(wb_set_sboard && wb_wen, wb_waddr)
```

See? if the instruction is div/cache miss/rocc, and writeback is enabled, the `sboard` is updated.

When to clear it? See `sboard.clear(ll_wen, ll_waddr)`. Here, the "ll" include the three cases above. You can read the writeback stage to find it out.

## When to read the scoreboard?

Now, we have figured out how the scoreboard various. But where do we use it? This is quite simple. The only usage of `sboard.read` is:

```
val id_sboard_hazard = checkHazards(hazard_targets, rd => sboard.read(rd) && !id_sboard_clear_bypass(rd))
```

It uses `checkHazards` to check the possible hazards. The `hazard_target` is a seq  of registers used (rx1/rx2/rd). And the express after the comma is to check whether the sboard of the register is "not busy" (in the next cycle, so need `!id_sboard_clear_bypass`).

The `id_sboard_hazard` will eventually lead to `ctrl_stalld` and the pipeline will stall.

## Finally

So far, we finished the analysis of scoreboard. As for the scoreboard of float point, I didn't focus on it, but it should be similar.