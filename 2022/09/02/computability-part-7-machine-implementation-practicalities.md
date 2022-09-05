# Computability Part 7: Machine Implementation Practicalities

In the [previous
post](https://vladris.com/blog/2022/07/31/computability-part-6-von-neumann-architecture.html)
we covered the von Neumann architecture and even built a small VM
implementing the different components. Such naïve implementation does
make for a very inefficient machine though. In this post, we'll dive a
bit deeper into machine architectures (virtual and physical) and discuss
some of the implementation details. We'll talk about processing:
register and stack-based; we'll talk about memory: word size, byte and
word addressing; finally, we'll talk about I/O: port and memory mapped.
Note these are all machines that conform to the von Neumann
architecture, with the same high-level components. We're just double
clicking to the next level of implementation details.

## Register machines

The VM we implemented in our previous post simply operated directly over
the memory. This works for a toy example, but moving data from memory to
the CPU and back is costly. That's why modern CPUs employ multiple
layers of caching (we won't cover these in this post), and rely on a
set of *registers* to perform operations.

Registers can store a number of bits (the *word size*, more on it below)
and operations are performed using registers. For example, to add two
numbers, the machine would load one number into register `R0`, the
second number into register `R1`, add the values stored in registers
`R0` and `R1`, then finally save the result back to memory:

``` text
mov r0 @<memory address 1> # Move the value from memory address 1 to r0
mov r1 @<memory address 2> # Move the value from memory address 2 to r1
add r0 r1 # Add the values storing the result in r0
mov @<memory address 3> r0 # Move the value from r0 to memory address 3
```

Some register are used for general computation. These are called
*general-purpose registers*. Other register have specialized purposes.
For example, the program counter which keeps track of the instruction to
be executed is usually implemented as an `IP` (instruction pointer) or
`PC` (program counter) register.

The original 8088 Intel processor had 14 registers. Modern Intel
processors have significantly more registers[^1], though many of them
are special-purpose. ARM processors have 17 registers[^2], 13 of which
are general purpose.

Let's emulate a simple CPU with 4 general purpose registers and a
program counter register to get the feel of it. We will only implement
`mov` (move) and `add` instructions for this example. Our implementation
will check the 16th bit of an argument to determine whether it refers to
a register (if `0`) or to a memory location (if `1`).

``` python
class CPU:
    def __init__(self, memory):
        self.memory = memory
        self.registers = [0, 0, 0, 0, 0] # r0, r1, r2, r3, pc

    def run(self):
        while self.registers[4] < len(self.memory):
            instr, arg1, arg2 = self.memory[
                self.registers[4]:self.registers[4] + 3]
            self.process(instr, arg1, arg2)
            self.registers[4] += 3

    def get_at(self, arg):
        # 16th bit tells us whether this refers to a register or memory
        if arg & (1 << 15): # Memory address
            return self.memory[arg ^ (1 << 15)]
        else: # Register
            return self.registers[arg]

    def set_at(self, arg, value):
        # 16th bit tells us whether this refers to a register or memory
        if arg & (1 << 15): # Memory address
            self.memory[arg ^ (1 << 15)] = value
        else: # Register
            self.registers[arg] = value

    def process(self, instr, arg1, arg2):
        match instr:
            case 0: # mov
                self.set_at(arg1, self.get_at(arg2))
            case 1: # add
                self.set_at(arg1, self.get_at(arg1) + self.get_at(arg2))
```

Here is how it would run a small program that adds two numbers and
stores the result:

``` python
program = [
    0, 0, 15 | (1 << 15), # mov r0 @15
    0, 1, 16 | (1 << 15), # mov r1 @16
    1, 0, 1,              # add r0 r1
    0, 17 | (1 << 15), 0, # mov @17 r0
    0, 4, 18 | (1 << 15), # mov pc @18 - this ends execution
    40,                   # this is @15
    2,                    # this is @16
    0,                    # this is @17
    10000                 # this is @18
]

## Load program into memory
memory = [0] * 10000
memory = program + memory[len(program):]

print(memory[17]) # Should print 0

CPU(memory).run()

print(memory[17]) # Should print 42
```

We're doing a bunch of stuff "by hand", like loading the program into
memory and not using an assembler to implement the program. That's
because we're only focusing on the register-based processing. You can
update the assembler in the previous post to target this VM as an
exercise.

## Stack machines

An alternative to registers is to use a stack for storage. While
hardware stack machines are not unheard of, register machines easily
outperform them so most CPUs you interact with are register-based. That
said, stack machines are a popular choice for virtual machines - they
are easier to implement and port to different systems and the stack
keeps the data being processed close together which helps with
performance when running the VM on a physical machine. A few examples:
JVM (the Java virtual machine), the CLR (the .NET virtual machine),
CPython's VM (the VM for the reference Python implementation) are all
stack-based.

The example we used above of adding two numbers would look like this on
a stack machine: push the first number onto the stack, push the second
number onto the stack, add the numbers (which would pop the two numbers
from the stack and replace them with their sum), then pop the value from
the stack and store it in memory.

``` text
push @<memory address 1> # Push a value from memory address 1
push @<memory address 2> # Push a value from memory address 2
add # Add the top two values
pop @<memory address 3> # Pop the top of the stack and store at memory address 3
```

Another advantage of stack machines is in general the instructions tend
to be shorter. As you can see above, for most instructions that move
data around, we don't need to specify both a source and a destination
since the stack is implied.

Let's emulate a simple stack VM with only `push`, `add`, and `pop`
instructions, plus a `jmp` (jump) instruction so we can use the same
mechanism to terminate:

``` python
class CPU:
    def __init__(self, memory):
        self.memory = memory
        self.stack, self.pc = [], 0

    def run(self):
        while self.pc < len(self.memory):
            instr, arg = self.memory[self.pc:self.pc + 2]
            self.process(instr, arg)
            self.pc += 2

    def process(self, instr, arg):
        match instr:
            case 0: # push
                self.stack.append(self.memory[arg])
            case 1: # pop
                self.memory[arg] = self.stack.pop()
            case 2: # jmp
                self.pc = self.stack.pop()
            case 3: # add
                self.stack.append(self.stack.pop() + self.stack.pop())
```

Here is how it would run a small program that adds two numbers and
stores the result:

``` python
program = [
    0, 12, # push @12
    0, 13, # push @13
    3, 0,  # add
    1, 14, # pop @14
    0, 15, # push @15
    2, 0,  # jmp
    40,    # this is @12
    2,     # this is @13
    0,     # this is @14
    10000, # this is @15
]

## Load program into memory
memory = [0] * 10000
memory = program + memory[len(program):]

print(memory[14]) # Should print 0

CPU(memory).run()

print(memory[14]) # Should print 42
```

Contrast the implementation with the register-based one: the latter VM
only needs 1 argument for the instructions we implemented and the
program is slightly shorter.

So far we focused on how data is processed. Let's also look at the
different ways of referencing data.

## Word size

We've been using Python for our toy implementations. Python supports
arbitrarily large integers, so a list of numbers in Python (the way we
implemented our memory) doesn't imply much in terms of bits and bytes.
Bits and bytes do become important for physical machines and serious VMs
implemented in languages closer to the metal.

First, let's talk about *word size*. A *word* is the fixed-size unit of
computation for a CPU. It's size is the number of bits. For example, a
16-bit processor has a word-size of 16-bits.

Applied to registers, this would mean that a machine register can hold
at most 16 bits (a value between 0 and 65535). Operations within the
value range are blazingly fast, as they run natively. If we need to
process larger values, we need to do extra work to chunk the values into
words and process these in turn. For example we can split a 32-bit value
into two 16-bit values, process them separately, then concatenate the
result. This obviously impacts performance. The point being that we are
not necessarily limited to the word size, but processing larger values
becomes much costlier.

Applied to memory addresses, this would mean how pointers are
represented and what range of values can be addressed. For example, if
the word size is 16 bits, then a pointer can point to any one of 65536
distinct memory locations.

An architecture can use the same word size for both registers and
pointers, or different word sizes for different concerns. Commonly, a
single word size is used (and, potentially, fractions or multiples of it
for special concerns), that's why it's common to refer to a processor
as a 32-bit processor, 64-bit processor etc.

## Byte and word addressing

An implication of word size applied to memory addressing is how the
machine accesses memory. Some architectures allow *byte addressing*,
which means a pointer points to a specific byte in memory, while others
support only *word addressing*, which means a pointer points to a word
in memory.

This is another important decision when designing a computer. If we want
to be able to address individual bytes, a 16 bit pointer can refer to
any of 65536 bytes. That is 64 Kb. If our memory is larger than that, a
pointer won't be able to address higher locations.

On the other hand, if we make our memory word-addressable, for our
16-bit example, a pointer can refer to any of 65536 16-bit words. 16
bits are 2 bytes, so our memory's upper limit is 131072 bytes (65536 x
2), which is 128 Kb. We can now refer to higher memory addresses, but we
can't address individual bytes as before - address 0 is no longer the
byte at 0, is the whole 2-byte word (since address 1 refers to the next
2 bytes and so on).

This difference becomes even more dramatic for higher word sizes. A
32-bit pointer can address 4294967296 bytes (up to 4 Gb of memory).
Alternately, with word addressing, the same pointer can cover 16 Gb.

On the flip side, word-addressing is less efficient when the unit of
processing is smaller. Let's take text editing as an example. Say we
want to update a one byte character, like a UTF-8 encoded common
character like `a`. If we can refer to it directly, we can load,
process, and update its memory location using a pointer. If, on the
other hand, this character is part of a larger word, we would have to
process the whole word to extract the character we care about (masking
bits we don't need to process), apply the update to the whole word, and
write this word back to memory.

So depending on the scenario, byte or word addressing might make things
faster or slower. Byte addressing is great for text processing -
document authoring, HTML, writing code etc. Word addressing unlocks
larger memory sizes and is great for crunching numbers - math, graphics
etc.

Another important design decision is how to handle I/O.

## Port-mapped I/O

One way to connect I/O to the system is through specific CPU
instructions. For example, the CPU might have an `inp` instruction used
to consume input and an `out` instruction used to send output. Programs
can use these instructions to perform I/O. This is called *port-mapped
I/O*, as I/O is achieved by connecting devices to the CPU via dedicated
ports.

For example, let's extend our stack machine with an `out` instruction
(also connecting an output to it):

``` python
class CPU:
    def __init__(self, memory, out):
        self.memory, self.out = memory, out
        self.stack, self.pc = [], 0

    def run(self):
        while self.pc < len(self.memory):
            instr, arg = self.memory[self.pc:self.pc + 2]
            self.process(instr, arg)
            self.pc += 2

    def process(self, instr, arg):
        match instr:
            case 0: # push
                self.stack.append(self.memory[arg])
            case 1: # pop
                self.memory[arg] = self.stack.pop()
            case 2: # jmp
                self.pc = self.stack.pop()
            case 3: # add
                self.stack.append(self.stack.pop() + self.stack.pop())
            case 4: # out
                self.out(self.stack.pop())
```

Here is a program that prints "Hello":

``` python
program = [
    0, 24, # push @24
    0, 25, # push @25
    0, 26, # push @26
    0, 27, # push @27
    0, 28, # push @28
    4, 0,  # out
    4, 0,  # out
    4, 0,  # out
    4, 0,  # out
    4, 0,  # out
    0, 29, # push @29
    2, 0,  # jmp
    111,   # this is @24
    108,   # this is @25
    108,   # this is @26
    101,   # this is @27
    72,    # this is @28
    10000, # this is @29
]

## Load program into memory
memory = [0] * 10000
memory = program + memory[len(program):]

def out(val):
    print(chr(val), end='')

CPU(memory, out).run()
```

## Memory-mapped I/O

An alternative to port-mapped I/O is *memory-mapped I/O*. In this case,
a certain address range of memory is used for I/O operations. That is,
from the CPU's perspective, memory and I/O are addressed identically.
But depending on the address range, data might reside in memory or it
might actually come from/go to an I/O device.

Let's enhance our memory implementation (which so far was just an
array) to support mapped I/O. In this case, any values written at
address 1000 will be instead printed on screen:

``` python
class MappedMemory:
    def __init__(self, program):
        # MappedMemory wraps a list
        self.memory = [0] * 10000
        self.memory = program + self.memory[len(program):]

    def __len__(self):
        # Use underlying list's __len__
        return self.memory.__len__()

    def __getitem__(self, key):
        # Index in wrapped list
        return self.memory[key]

    def __setitem__(self, key, value):
        # If key is 1000, print
        if key == 1000:
            print(chr(value), end='')
        # Otherwise set in underlying list
        else:
            self.memory[key] = value
```

And here is the corresponding program that prints "Hello" (using the
stack CPU without the `out` instruction and connected output):

``` python
program = [
    0, 24, # push @24
    0, 25, # push @25
    0, 26, # push @26
    0, 27, # push @27
    0, 28, # push @28
    1, 1000, # pop @1000
    1, 1000, # pop @1000
    1, 1000, # pop @1000
    1, 1000, # pop @1000
    1, 1000, # pop @1000
    0, 29,   # push @29
    2, 0,    # jmp
    111,     # this is @24
    108,     # this is @25
    108,     # this is @26
    101,     # this is @27
    72,      # this is @28
    10000,   # this is @29
]

## Load program into memory
memory = MappedMemory(program)

CPU(memory).run()
```

Note in this program we repeatedly "set" the value at address 1000
which is mapped to our output device (`print()`).

## Summary

In this post we discussed some of the implementation details of machines
and virtual machines:

-   Register machines, which are high-performance designs for physical
    machines.
-   Stack machines, which are simple, portable, and great alternatives
    for VMs.
-   Word size, as the unit of data used by the machine for different
    purposes.
-   Word-addressable memory, which is great for computation intensive
    scenarios.
-   Byte-addressable memory, which is best for text-processing
    scenarios.
-   Port-mapped I/O, where special CPU instructions are used for input
    and ouptut.
-   Memory-mapped I/O, where reserved address ranges of memory are used
    for I/O and the CPU can access I/O just like it does memory.

### Bonus

A few years back I implemented a toy VM with 7 registers, 16 op codes,
128 KB of memory, and port-mapped I/O in 121 lines of C++. It comes with
an assembler, examples, and, of course, a Brainfuck interpreter. Linking
it here for reference: [Pixie](https://github.com/vladris/pixie).

[^1]: See [this SO question](https://reverseengineering.stackexchange.com/questions/19693/how-many-registers-does-an-x86-64-cpu-actually-have>).

[^2]: See the [ARM documentation](https://developer.arm.com/documentation/dui0473/c/overview-of-the-arm-architecture/arm-registers?lang=en>).
