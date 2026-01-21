---
title: Optimizing register renaming
data: 2026-01-20
categories: [FPGA]
tags: []
---

# Story

A little over a year ago, I was discovering hardware and FPGAs for the first time.
So I started my first project in Bluespec System Verilog with the ambition of doing a RISC-V core, I didn't imagine how long it would take.
Ultimately, I went much further than I initially expected, since that CPU was out of order.
I decided to name the CPU [DOoOM](https://github.com/RemyCiterin/DOoOM) for "DOoOM Out-of-Order Machine" with the idea of running DOOM on it, and even if it still have some bugs that I haven't had time to solve, it is capable of running DOOM on an ECP5-85F FPGA using the [ULX3S](https://radiona.org/ulx3s/) board.

# Overall architecture

![dooom pipeline](/assets/img/dooom/dooom.svg)
_DOoOM pipeline architecture in it's final version_

The final version is an out-of-order RISC-V CPU implementing rv32imf_zicsr_zicbom meanings that it support:
- Integer multiplication and division
- Floating point operations
- System instruction like `csrr`, `csrw`...
- Explicit cache managment, even though this part is still buggy but used to run DOOM

# Implementation of the register renaming stage

## First try

The main goal of register renaming is to maintain the invariant that two instructions after the renaming stage can't have the same destination register.
With this invariant, it's possible to write-back the results of the instructions in any order because there can be no write conflicts.

The first method that I implemented is to use the reorder buffer, a circular buffer that you increment each time you dispatch a new instruction to the issue queues.
This circular buffer is also used to detect mispredictions later in the pipeline, and to flush it if everything unexpected happend.
As example with those lines of assembly:

```asm
li t0, 42  ; regs[t0] <- 42
mv t1, t0  ; regs[t1] <- regs[t1]
li t0, 43  ; regs[t0] <- 43
mv t1, t0  ; regs[t2] <- regs[t0]
```

this set of events can happends in the CPU:

|------|--------------------------------|--------|------------|-----------------|--------------------|
|Cycle | Event                          | ROB ID | Micro OP   | Operands        | Alias table        |
|------|--------------------------------|--------|------------|-----------------|--------------------|
| `0`  | `li t0, 42` enter the pipeline | `0`    | `r0 <- 42` | `{}`            | `{t0: r0}`         |
| `1`  | `mv t1, t0` enter the pipeline | `1`    | `r1 <- r0` | `{unknown(r0)}` | `{t0: r0, t1: t1}` |
| `2`  | `li t0, 43` enter the pipeline | `2`    | `r2 <- 43` | `{}`            | `{t0: r2, t1: t1}` |
| `3`  | `li t0, 42` complete           |        |            |                 | `{t0: r2, t1: t1}` |
| `4`  | `mv t1, t0` complete           |        |            |                 | `{t0: r2}`         |
| `5`  | `mv t0, t1` enter the pipeline | `3`    | `r3 <- t1` | `{known(42)}`   | `{t0: r3}`         |
|------|--------------------------------|--------|------------|-----------------|--------------------|

Of course, in practice, one instruction could enter the pipeline while another is leaving it at the same cycle.
And it's also necessary to empty some data structures each time the pipeline is flushed.
But I will ignore those problems for the rest of this post.

Here we maintain an alias table that gives, for every register, the ID in the reorder buffer of the previous instruction that can write to the register, if one exists.
So we can use the values in this table to know if we can read the operands of the new instruction from the register file, or if we have to wait for another operation to complete.

Those informations are then entered into a queue (named the issue queue), and the instruction waits for all its operands to be ready.
This method results in a very clean interfaces in my implementation in Bluespec:

```bsc
interface Rob;
  // Enter a new instruction and return it's ID
  method ActionValue#(RobId) enter(MicroOp microOp);

  // Write-back an operation from the execution unit
  method Action writeBack(Addr newPc, Data result, Bool flush, ...);

  // Remove an instruction from the circular buffer
  method Action deq;

  ...
endinterface

interface IssueQueue;
  // Push a new instruction to the issue queue
  method Action enter(MicroOp microOp, Vector#(2, DataOrRobId) operands);

  // Inform the issue queue that a register was written, and propagate the data
  method Action wakeup(RobId id, Data data);

  // Dequeue an instruction with all it's operands ready from the queue
  method ActionValue#(IssueQueue2Exec) issue;
endinterface
```

This was working well until I looked at the clock frequency of my CPU, because even if I was able to commit instructions at a very high throughput, I was struggling to run it at more than 20 MHz.
In particular, I saw in the outputs of nextpnr that the arrival of data from the memory unit to the issue queues was part of the critical path, so I switched to using another method.

## Second try

Seeing that my method was not working well, I did some research and in particular found this [graph](https://docs.boom-core.org/en/latest/sections/rename-stage.html) in the [BOOM](https://boom-core.org/) documentation (a RISC-V out-of-order superscalar CPU).
This graph explains that there are several types of renaming, implicit renaming which I have used so far, and explicit renaming.
Then, I found more explanations by reading the chapter dedicated to register renaming in [Modern Processor Design](https://github.com/savitham1/books/blob/master/Modern%20Processor%20Design%20-%20Fundamentals%20of%20Superscalar%20Processors.pdf).

This new method requires two changes in the architecture:
- First I don't transmit directly the data from the execution units to the issue queue; instead, I just inform it that the data is ready to be read in the register file at the next cycle. Then I read all the operands at the output of the issue queues. This adds one stage in the pipeline, reducing the performance of my CPU on the benchmarks but allowing it to run at a higher clock speed.
- Then I switched to an explicit physical register file, instead of having registers in the architectural register file and the reorder buffer. Doing so, we need some kind of free-list to keep track of which register is available.


In short, we now have four stages to consider instead of three:
- the entry/dispatching of an instruction into the reorder buffer and issue queues (in order)
- the reading of registers (out of order)
- the writing of registers (out of order)
- the completion/commit (in order)


Using an explicit register file is necessary here, otherwise one could observe some bugs.
Indeed, since registers are read out of order (after leaving the issue queues), the physical registers associated to an instruction must remain valid until another instruction writing to the same architectural register complete. At that point, all instructions that read the previous physical register are known to have also been committed too.
But using the reorder buffer, the physical registers were only valid until the instruction left the circular buffer (at commit/completion stage), but could be reallocated from that instant.

For example, this sequence of events is possible by using explicit renaming with the reorder buffer as a register file of size 2:

```asm
dispatch: li t0, 42      ; micro op: r0 <- 42
dispatch: mv t1, t0      ; micro op: r1 <- r0
write-back: li t0, 42    ; rob[r0] <- 42
complete: li t0, 42      ; free(r0)
dispatch: li t3, 43      ; micro op: r0 <- 43 (r0 is re-allocated)
write-back: li t3, 43    ; rob[r0] <- 43
register-read: mv t1, t0 ; read 43 into r0 (ERROR)
```

But with a dedicated physical register file and a free-list this is not possible, as the register allocated for `t0` will only be freed when an instruction that also writes to `t0` is committed, and at that moment `mv t1, t0` will have already left the reorder buffer, because dispatch/stages are done in order.

This optimization, with some others, improved the maximum clock frequency of my CPU from around 18 MHz to 27 MHz on my ECP5.
Allowing me to play DOOM without overclocking my FPGA!
