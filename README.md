# TIS-100
An attempt to re-create the [TIS-100 Tessellated Intelligence System](http://www.zachtronics.com/tis-100/) (as faithfully as possible) in VHDL

The end goal is to create a template for [PYNQ](https://github.com/Xilinx/PYNQ) [Overlays](http://pynq.readthedocs.io/en/v2.1/pynq_overlays.html) that can execute programs that would work in the game and reach the same results.

This requires four main things:

1. VHDL models of the different T** nodes described in the TIS-100 Manual and the overall system (clock distribution, memory buses, etc.)
2. Some sort of Architecture Description Language (ADL) to make it easy to compose them into clusters that resemble the ones in the game
3. An "assembler" that takes the language accepted by the game and transforms it into actual instructions, ready to transfer to the FPGA
4. An Overlay package that wraps all this up nicely and allows users to run their own code on their very own, real-world TIS-100

## System Design

The overall system parameters define the framework for the individual nodes' functionality and inform things like the width and design of data connections and internal buses.

We'll go through a few aspects of that system design, using the **T21 Basic Execution Node** as a reference.
All other available nodes (with the exception of the Visualization Module) are much simpler and can be thought of as variations of this one.

### Clock
All the nodes in a TIS-100 follow the same clock. For easy debugging, its speed is adjustable, from *stopped* to, uh... *fast*.
Nothing to worry about here - maybe we can use SW1 and SW0 on the PYNQ board to select a speed, and one of the push buttons to step (we should probably also enable stepping via the Overlay driver, so we can do it from Python).

### Arithmetic Range
Next, let's see how many bits we need!

All data on the TIS-100 is numeric in nature.
To be precise, every peace of data is an integer in the interval [-999, 999].
That's 1998 numbers in total, so the nearest power of 2 is 2^11 = 2048.
That means we need to use 11 bits to represent our numbers, since you can represent 2^n numbers using n bits.

Conveniently, that leaves 50 numbers we cannot use for math, which comes in useful when designing the ISA.
Let's do what all the cool kids do and use [two's complement](https://en.wikipedia.org/wiki/Two%27s_complement) for negative numbers.

> **Architecture Note:**
>
> The TIS-100 VHDL System will use 11-bit signed (two's complement) integer numbers

With that decided, let's look at the minimum and maximum numbers our system can use in 11-bit binary:

```
 999: 01111100111
-999: 10000011001
```

> **Architecture Note:**
>
> The addition and subtraction operations do not overflow, but saturate at the interval limits.
> We'll need to keep this in mind when designing the ALU!

Due to the way two's complement numbers work, this means we can nab the block in between those two for our own sordid purposes.
Since there aren't that many special values, I'll just take the block from `10000000000` to `10000010000` - easy to recognize, and 16 "special" numbers are more than we need!

> **Architecture Note:**
>
> Registers and ports are numbered starting from `10000000000` when expressed as 11-bit numbers.

### Registers & Ports
Now, what special numbers might we need?
The first thing that comes to mind are the ACC and NIL registers of the T21 node.
They can be used in the place of literal numbers with any instruction, so we need a way to encode them.
(NIL is special, but for now, let's treat it like a register)

There's also the four Ports (plus two "Pseudo-Ports") that every T21 node has:

* UP
* DOWN
* LEFT
* RIGHT

* ANY
* LAST

That gets us up to 8 registers (and register-y things) total.
Yay!
We have half of our special numbers left!

Anyhow...
Each port is a synchronous (with some exceptions, depending on the node), bidirectional (again, with exceptions) data connection between two end points.
So of course, each one needs 11 data wires (one for each bit, duh).

But what else?

To get a definitive answer, we also need to consider the "Pseudo-Ports", LAST and ANY.

ANY writes to whatever port first indicates it wants to read, or reads from whichever port receives data first.
LAST works together with ANY, but is pretty easy:
It just means "the last thing ANY picked", so we need to save that in a special, hidden 2-bit (there's only four options) register.
(The manual specifies that the behavior of LAST with no preceding ANY is undefined, so this is fine. But the reference implementation stalls when using LAST without ANY, so we may want to use another bit to show whether LAST has been initialized)

Without spending too much time on premature optimization, let's add two extra wires to the Port interconnect: WRITE and READ.

> **Architecture Note:**
>
> The physical interface at a Port consists of 11 DATA wires, one READ and one WRITE wire.

With those, ANY can just choose the port whose WRITE or READ wire is being pulled high by a neighbor.
A quick experiment shows that in case of multiple ports receiving data in the same cycle, the order of precedence is

**LEFT, RIGHT, UP, DOWN**

for reading, and

**UP, LEFT, RIGHT, DOWN**

for writing.

The READ and WRITE wires are also used to determine whether to continue execution.
Let's assume that we're able to do a read from or write to a port in one cycle - the manual says timing and performance may vary between hardware, but we'll stick to the reference implementation for the sake of simplicity.

Let's look at an example:
```
           +---------------+       +-------+
READ   +---+               +-------+       +

                   +-------+       +-------+
WRITE  +-----------+       +-------+       +

           +---+   +---+   +---+   +---+
CLOCK  +---+   +---+   +---+   +---+   +---+
           T1      T2      T3      T4

```

1. At T1, one of the two nodes connected to this port pulls the READ line high.
2. At T2, the second node writes a piece of data and pulls the WRITE line high.
3. At T3, the reading node sees the high WRITE line and reads the data. It can then continue to the next instruction.
   The writing node checks the READ line. It's high, so it knows the data has been read and continues.
   So the next instructions (another read and write, this time both at the same time!) are executed at T4.

Given that Port transfers wait for the other side to acknowledge (by pulling either READ or WRITE high), the process will deadlock whenever both sides of a Port try to read or write at the same time.
The manual says that "a hardware fault will occur", but no such thing happens in the reference implementation.
The two nodes will simply wait forever.

Given that potential to deadlock, a RESET pin is crucial for the VHDL implementation!

> **Architecture Note:**
>
> Each Node has a high-active RESET pin in addition to its CLOCK input and any Ports.

## ISA

Unlike all the other nodes featured in the game, the T21 node is programmable.
Luckily for us, the instruction set is very simple:
There's only 13 instructions (we'll get to what they are later).

That means we need 4 bits in our instruction word to encode the opcode.
Let's see how much we need for the operands!

Let's break it down into categories:

### Moving Data

* **NOP**

  A bit of a cheat, I know. But I guess this moves no data to nowhere.
  Easy!
* **MOV** src: {literal, port, ACC, NIL}, dst: {register, port, ACC, NIL}

  A bit more interesting!
  We figured out earlier that our literals are 11 bits long, and that with those 11 bits, we can also encode the registers and ports.
  But what about *dst*? Since we only have 8 special targets, we can get away with only using 3 bits for that!

  When moving to ACC or NIL, this takes one cycle.
  Writes to a Port block until the other side moves out of that port.
  **Note:** A Port write will block for at least one cycle, even if another node is already waiting on the other side - you need to write to the Port *before* the other side can read!

  *Reads* from a Port will block until data is available, but without the one cycle minimum that writes incur.
* **SWP**

  This swaps ACC with the special register BAK - no operands, no problem!
* **SAV**

  This *copies* ACC into BAK - again, no operands!


### Arithmetic

* **ADD** src: {literal, port, ACC, NIL}

  Add the operand to ACC.
  Again, when reading from a Port, this may block.
  ACC saturates at 999.

* **SUB** src: {literal, port, ACC, NIL}

  Subtract the operand from ACC (will block if reading from a Port).
  ACC saturates at -999.

* **NEG**

  Negate the value in ACC (0 remains 0)


### Jumps

Jumps are going to be interesting, for two reasons that are intertwined:

1. It's not possible to jump out of the program
2. The length of the program has an upper bound, but can vary

This means relative jumps (see **JRO** below) in particular need to be somewhat supervised.

> **Architecture Note:**
>
> We need a way to figure out the length of the currently loaded program at run time.
> See "Extra Instructions" below for ideas on how to do this.
> We'll call that value *programLength*.

Given the length of the program, we can now ensure that the PC is never less than 0 or more than *programLength*.

* **JMP** target: literal
  Jump to the specified address in the program, i.e. set the Program Counter (PC) to *target*.

* **JEZ** target: literal
  Jump to *target* if ACC is zero.

* **JNZ** target: literal
  Jump to *target* if ACC is *not* zero.

* **JGZ** target: literal
  Jump to *target* if ACC is strictly greater than zero.

* **JLZ** target: literal
  Jump to *target* if ACC is strictly less than zero.

* **JRO** offset: literal
  Jump to PC + *offset*.
  If *offset* is zero, this will trap the machine in an infinite loop!

  Note that this, too, is bounded at *programLength* and does **not** wrap, which means the program

  ```
  MOV 1 ACC
  JRO ACC
  ```

  Is an infinite loop!

  ACC is 1, the JRO instruction tries to jump ahead by 1, but since there is no instruction after JRO to jump to, it gets "stuck".


### Extra Instructions

This section is primarily dedicated to figuring out how to find the length of the current program.

Two alternatives come to mind:

* **LEN** programLength: literal

  Must be the first instruction in any T21 program.
  This tells the CPU how long the program is and thereby sets the upper bound for jumps.
  Easier from a hardware perspective (read *programLength* into a register, then use it from there), but there's a few downsides:
    1. The assembler now needs to count instructions, making it more complex
    2. All programs are one instruction longer than their "original" TIS-100 equivalent.
    3. How do we count the space the LEN instruction is at?
       For programs to work as quickly as on a "real" T21, it would need to be "-1" and only
       executed once after a reset.

* **VOID**
  Denotes unused program space.
  Since the T21 has only 15 instructions' worth of program memory, it's feasible to detect how many of those are used by using a specialized memory matrix.

  This is spectacularly wasteful in terms of hardware used vs. using a BRAM on an FPGA, but it avoids the issues with LEN above.

  A VOID instruction must never be executed. If it is, the T21 will halt.
  So, just as it would be the assembler's responsibility to calculate the correct length if we used the LEN instruction above, with VOID it would be the assembler's responsibility to ensure the program is contiguous in memory.

  Then again, that's what it would have to do anyway.
  No point in allowing the assembler to mix in illegal instructions, is there?


### Instruction format

For those not keeping track in their head:
The longest instruction we have is MOV, with its two parameters - one literal/register/port (11 bits), one register/port (3 bits).

So how many bits do we need in total?
Behold!
```
    4 bits for the opcode
   /
  /    11 bits for the first operand (if any)
 /    /           3 bits for the second operand (if any)
/    /           /
0000 00000000000 000

18 bits total
```

## Loading a program

The next question is:

How do we even load a program into a T21 node?
Let's go with a pretty easy way:

> **Architecture Note:**
>
> Each T21 node has an 18-bit wide DATA port and DEN (data enable) pin.
> If DEN is high, the values from DATA are written to program memory on rising clock edge.
>
> An internal counter increments after every clock cycle, and is reset when RESET is pulled high.

This approach allows us to connect the DEN pins of all nodes to a multiplexer and their DATA ports to a single bus.

The interface to the outside world might then be shared memory, with some logic to deal with the resetting, read from the shared memory block and manage the node address (i.e. the input to the DEN mux), incrementing it after reading a VOID instruction or something.
