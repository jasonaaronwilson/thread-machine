# Thread Machine

This is a a computer architecture design for a machine designed to run a large number of threads per core. We also explore making *all* instructions operate on 512bits, one cache line, at a time.

The basic idea is that the state of a thread (even registers) are maintained so that a thread is mostly a 64bit address to a root cache-line and an address space identifier.

## SIMD

The basic idea of SIMD is that a single instruction operates on multiple units of data in parallel. Currently AVX-512 can operate on 64 8bit values in parallel! A 64bit mask allows only a portion of the bytes to be "writen" so it's still possible to operate on a single byte at a time if this is needed.

AVX-512 may sound like it isn't suitable for small area implementations, however by "pumping" say 64bits at a time into a 64bit ALU, a small area implementation is still possible (if the mask only operates on a subset of a cache-line, then pumping only the useful operation into the ALU would save time when running basic single threaded like code.

## Threads and Registers

The design uses "memory mapped" registers (an old idea actually). Let's start with a design where registers are mapped to a single cache-line. Running threads need a PC (program counter) and an ASI (which will be some subset of 64bits). Some small amount of state could be stored in the unused portion of the ASI. This can take up the first two 64bits "slots" of a cache-line. That leaves 6 64bit slots to hold register data. Registers are normally the same width as the ALU data-path and our logical ALU width is 512bits which don't fit into 64bits. The solution is to say that these 6 64bit slots to hold either 64bit addresses or 64bit "masks" (often the mask will come directly from the instruction itself). One of the remaining slots would be used as a "stack" pointer by convention leaving room for 5 addresses.

## Sample Instructions

Let's say we want to compile this C program fragment.

```
uint64_t a = 100;
uint64_t b = 200;
uint64_t c = a + b;
```

We will allocate a, b, and c to an area on the stack.

What is weird is that we don't really have loads and stores. Instead instructions eventually computes the address of the cacheline ...
