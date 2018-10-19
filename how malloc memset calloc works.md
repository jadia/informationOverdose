### The short version: 
Always use `calloc()` instead of `malloc()+memset()`. In most cases, they will be the same. In some cases, `calloc()` will do less work because it can skip `memset()` entirely. In other cases, `calloc()` can even cheat and not allocate any memory! However, `malloc()+memset()` will always do the full amount of work.

Understanding this requires a short tour of the memory system.

## Quick tour of memory

There are four main parts here: your program, the standard library, the kernel, and the page tables.  You already know your program, so...

Memory allocators like `malloc()` and `calloc()` are mostly there to take small allocations (anything from 1 byte to 100s of KB) and group them into larger pools of memory.  For example, if you allocate 16 bytes, `malloc()` will first try to get 16 bytes out of one of its pools, and then ask for more memory from the kernel when the pool runs dry.  However, since the program you're asking about is allocating for a large amount of memory at once, `malloc()` and `calloc()` will just ask for that memory directly from the kernel.  The threshold for this behavior depends on your system, but I've seen 1 MiB used as the threshold.

The kernel is responsible for allocating actual RAM to each process and making sure that processes don't interfere with the memory of other processes.  This is called *memory protection,* it has been dirt common since the 1990s, and it's the reason why one program can crash without bringing down the whole system.  So when a program needs more memory, it can't just take the memory, but instead it asks for the memory from the kernel using a system call like `mmap()` or `sbrk()`.  The kernel will give RAM to each process by modifying the page table.

The page table maps memory addresses to actual physical RAM.  Your process's addresses, 0x00000000 to 0xFFFFFFFF on a 32-bit system, aren't real memory but instead are addresses in *virtual memory.*  The processor divides these addresses into 4 KiB pages, and each page can be assigned to a different piece of physical RAM by modifying the page table.  Only the kernel is permitted to modify the page table.

## How it doesn't work

Here's how allocating 256 MiB does *not* work:

1. Your process calls `calloc()` and asks for 256 MiB.

2. The standard library calls `mmap()` and asks for 256 MiB.

3. The kernel finds 256 MiB of unused RAM and gives it to your process by modifying the page table.

4. The standard library zeroes the RAM with `memset()` and returns from `calloc()`.

5. Your process eventually exits, and the kernel reclaims the RAM so it can be used by another process.

## How it actually works

The above process would work, but it just doesn't happen this way.  There are three major differences.

* When your process gets new memory from the kernel, that memory was probably used by some other process previously.  This is a security risk.  What if that memory has passwords, encryption keys, or secret salsa recipes?  To keep sensitive data from leaking, the kernel always scrubs memory before giving it to a process.  We might as well scrub the memory by zeroing it, and if new memory is zeroed we might as well make it a guarantee, so `mmap()` guarantees that the new memory it returns is always zeroed.

* There are a lot of programs out there that allocate memory but don't use the memory right away.  Some times memory is allocated but never used.  The kernel knows this and is lazy.  When you allocate new memory, the kernel doesn't touch the page table at all and doesn't give any RAM to your process.  Instead, it finds some address space in your process, makes a note of what is supposed to go there, and makes a promise that it will put RAM there if your program ever actually uses it.  When your program tries to read or write from those addresses, the processor triggers a *page fault* and the kernel steps in assign RAM to those addresses and resumes your program.  If you never use the memory, the page fault never happens and your program never actually gets the RAM.

* Some processes allocate memory and then read from it without modifying it.  This means that a lot of pages in memory across different processes may be filled with pristine zeroes returned from `mmap()`.  Since these pages are all the same, the kernel makes all these virtual addresses point a single shared 4 KiB page of memory filled with zeroes.  If you try to write to that memory, the processor triggers another page fault and the kernel steps in to give you a fresh page of zeroes that isn't shared with any other programs.

The final process looks more like this:

1. Your process calls `calloc()` and asks for 256 MiB.

2. The standard library calls `mmap()` and asks for 256 MiB.

3. The kernel finds 256 MiB of unused *address space,* makes a note about what that address space is now used for, and returns.

4. The standard library knows that the result of `mmap()` is always filled with zeroes (or *will be* once it actually gets some RAM), so it doesn't touch the memory, so there is no page fault, and the RAM is never given to your process.

5. Your process eventually exits, and the kernel doesn't need to reclaim the RAM because it was never allocated in the first place.

If you use `memset()` to zero the page, `memset()` will trigger the page fault, cause the RAM to get allocated, and then zero it even though it is already filled with zeroes.  This is an enormous amount of extra work, and explains why `calloc()` is faster than `malloc()` and `memset()`.  If end up using the memory anyway, `calloc()` is still faster than `malloc()` and `memset()` but the difference is not quite so ridiculous.

---

## This doesn't always work

Not all systems have paged virtual memory, so not all systems can use these optimizations.  This applies to very old processors like the 80286 as well as embedded processors which are just too small for a sophisticated memory management unit.

This also won't always work with smaller allocations.  With smaller allocations, `calloc()` gets memory from a shared pool instead of going directly to the kernel.  In general, the shared pool might have junk data stored in it from old memory that was used and freed with `free()`, so `calloc()` could take that memory and call `memset()` to clear it out.  Common implementations will track which parts of the shared pool are pristine and still filled with zeroes, but not all implementations do this.


Ref: [https://stackoverflow.com/questions/2688466/why-mallocmemset-is-slower-than-calloc](https://stackoverflow.com/questions/2688466/why-mallocmemset-is-slower-than-calloc)
