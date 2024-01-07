# Thread Machine

This is a a computer architecture design to run a large number of
threads per core. Additionally we are also explore making *all* or
*most* instructions operate on 512bits, one cache line, at a time,
i.e., this is inherently also a SIMD architecture. The combination of
lots of thread parallelism plus lots of SIMD instruction is hopefully
something suitable to run AI tasks and various simulations. It won't
make much of today's mostly single threaded software run any faster
(it might even be slower), but it seems fun to take a look at
something very different from a learning / hobby perspective.

The basic idea is that the state of a thread (even registers) are
maintained so that a thread is mostly a 64bit address to a root
cache-line and an address space identifier.

## SIMD

SIMD allows a single instruction operates on multiple units of data in
parallel. For example, currently AVX-512 can operate on 64 8bit values
in parallel! A 64bit mask can be used to allow only some of those
bytes to be "writen" so it's still possible to operate on a single
byte at a time if this is needed.

AVX-512 may sound like it isn't suitable for small area
implementations, however by "pumping" say 64bits at a time into a
64bit ALU, a small area implementation is still possible.

## Threads and Registers

The design uses "memory mapped" registers (an old idea
actually). Let's start with a design where registers are mapped to a
single cache-line. Running threads need a PC (program counter) and an
ASI (which will be some subset of 64bits). Some small amount of state
could be stored in the unused portion of the ASI. This can take up the
first two 64bits "slots" of a cache-line. That leaves 6 64bit slots to
hold register data. Registers are normally the same width as the ALU
data-path and our main ALU width is 512bits which don't fit into
64bits... The solution is to say that these 6 64bit slots hold either
64bit addresses or 64bit "masks" (often the mask will come directly
from the instruction itself). Most real data will be read from and
written back to memory (via a cache of course). One of the remaining
slots would be used as a "stack" pointer by convention leaving room
for 5 addresses. Our stack pointer will be required to be 512 bit
aligned to simplify things further. (In fact the lower bits of all
registers will likely be ignored or trapped on.)

## Sample Instructions

Let's say we want to compile this C program fragment.

```
uint64_t a = 100;
uint64_t b = 200;
uint64_t c = a + b;
```

We will allocate a, b, and c to an area on the stack, probably right
next to each-other!

What is weird is that we don't really want loads and stores
anymore. For a two input, one output ALU, what we really want is 3
address and potentially some masks and shifts.

First let's write what the instruction might look like in a
hypothetical 64bit world with some kind of very CISCy instructions:

```
  add.64 sp[128], sp[0], sp[64] 
```

We can convert that into a more complicated instruction that might
look something like this:

```
  (add.64 (adderress sp (info-dest)) (sp (src1-info)) (sp (src2-info)) 
```

Where (info-dest) still needs to be figured out. If we shift src1 to
the output position and shift the result to where it needs to go then
use a mask that only computes what is needed to save power and of
course only write back the right bytes, then we have accomplished an
ADD.

If this actually a SIMD code snippet, you probably wouldn't balk at
any of this, after all you getting lots of work out of the
instruction. The only reason we are doing this for regular code is to
see just how bad it would be.

Using a very tightly integrated L1 cache, I don't see why the SIMD
case can't be made just as fast as when using register files. I'm
probably wrong, but perhaps at the hardare level a cache and a
register file actually end up being somewhat similar...

## Threads

We propose that each core have N threads and have a hardware thread
scheduler. The simplest hardware implementation would simply go "round
robin" to "ready" threads. When an instruction is decoded, we see if
we have the input and output cacheline. If yes, we execute the
instruction and proceed, advance the PC, and also advance to the next
ready thread. If no, we send a message to "fetch" up to three
cachelines, set our status to waiting for 3 ACKs, mark the thread as
not ready, and proceed to the next ready thread.

There is no guarantee that by the time we get the 3rd ACK that the
data for the 1st two are still in the cache! When we attempt to retry
the instruction until is succeeds.

## Operating System Threads

The proposal is to have the hardware thread scheduler be quite stupid
and low-level. The operating system can map user threads to hardware
threads on the same core or elsewhere and support a crazy number of
"virtual" threads by periodically removing some threads from a core
and replacing them with ready threads that aren't currently in a
hardware thread bank.

## Memory Hierachy + TLB misses / page-faults

Memory hierachies from a core's perspective normally start with small
but fast virtually addressed L1 cache, then a slower L2 cache (could
be virtually or phsycially addressed) a big shared L3 cache (usually
physically addressed) and of course DRAM.

Eventually a virtual address must be converted into to a physical
address and this is typically done with a TLB. When a TLB miss
occurrs, then the instruction must generally assume that the memory is
not mapped so we can't just use a store buffer and delay the TLB
lookup for the store address, it must be in the TLB (sort of - we
probably can make this assumption as long as we can roll back the
state a little bit...)

The great thing about a TLB miss is that we can handle it in regular
code in a privledge thread running like all of the others. Wether
going directly to a traditional page table or first looking at a nice
large hashtable, we can probably implement many formats for how page
tables work in software. If we want to completely use physical
addressing, the hardware thread that resolves the TLB miss can put the
appropriate magic in the TLB with a few instructions and then let the
machine know that this hardware thread has ended (so it can be reused)
as well as letting it know which thread id to mark ready (may have
already been removed from the hardware by a different thread running
in the OS).

What's great about this architecture is that hardware and software are
working together to make hardware thread scheduling practical. Since
the hardware already needs to deal with a variety of asynchrnous
events that might already delay and retry executing an instruction.

Having many hardware threads is that we can do useful work while
stalled waiting for memory. This doesn't make single threaded programs
any faster but may lead to big speedups in multi-threaded workloads.

## branches

Unconditional branches are kind of easy to imagine how to handle - we
simply modify the pc and continue executing.

there are a bunch of ways to do conditional branches, though we can
probably just have "branch if zero/not-zero (cacheline+mask)"

indirect branches are pretty similar really.

most architectures also have "call" instructions though it is not
clear if we need that here. the caller can be responsible for putting
it's return address on the stack before executing one of the other
branch types. this adds more instructions but might other things
simpler. We still haven't used our scratch space yet...

## Messages

The messages used above are not architecturally visible state. However
it would be cool to be able to send messages to other threads...

## Some Random Ideas

* an implementation can be pipelined across threads or within the same
  thread
* obviously we can execute more than one instruction before moving to
  the next thread. probably a small counter of instructions executed
  and/or switching whenever we execute a branch instruction is the way
  to go.

