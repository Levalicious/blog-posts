# Introduction
Allocating memory on the heap is expensive. Especially, in functions that are called frequently it should be avoided for performance reasons. However, there are cases where it is not possible to get rid of all heap allocations. In these cases there are four options:

1. Live with the heap allocation(s).
1. Statically allocate a block of memory on the stack, which is hopefully big enough.
1. Cache the block of allocated heap memory and only reallocate it if the amount of required memory is bigger than the size of the cached block of memory.
1. Write a custom memory allocator.

The first and the second option are obviously a very bad choice. The third one looks promising, but it turns out to be impractical in a lot of cases as it requires an execution context (state) to store a handle to the cached memory in. Classic functions don't have a state by definition. Classes have a state when instantiated, but there are plenty of situations where storing cached memory in class objects is impractical (e.g. for temporary objects). Writing a custom allocator works, but requires a lot of additional work.

# Dynamically allocating memory on the stack
A less well known, fifth possibility is to dynamically allocate memory on the stack. Yes, that's possible and sometimes a very convenient way to optimize heap allocations. Especially, for small dynamic arrays like look-up tables. However, dynamic stack allocations come along with a number of drawbacks one should be aware of in order to decide if they are the right tool in some situation or not. Facts that are important to know are listed in the following.

* Memory, dynamically allocated on the stack, is automatically freed when the calling function exits; not when the scope the memory was allocated in is left. This can have severe consequences on stack allocations performed within a scope, where one would assume that the memory is freed when the scope is left. Examples for that are dynamic stack allocations in
    * functions that are potentially inlined.
    * loops, because memory usage accumulates with each iteration.
* The maximum amount of allocatable stack memory is relatively small. The exact size depends on the linker settings or thread creation parameters. On Windows (Visual C++) the default stack size is 1 MB and on Linux (GCC) it is 8 MB.
* Depending on how much stack memory already got allocated by previously called "parent" functions, there is more or less stack memory left for allocation. This is especially important for dynamic stack allocations performed in recursive functions.
* Using dynamic stack allocations might effect portability, as they are not specified by the ANSI-C standard. However, most compilers do support them.
* No constructors and destructors are called and no `delete` or `free` is required.
*Dynamic stack allocations cannot be performed directly inside a function call's argument list, because the allocated stack space would appear between the parameters passed to that function.
* Generally, when allocating more than 1 *page* (usually 4 KB) on the stack, *stack probing* is required to assure correct stack growth (more on that later).

# How to do it?
In Visual C++ there are two functions to dynamically allocate memory on the stack: [`_alloca`](http://msdn.microsoft.com/en-us/library/wb1s57t5.aspx) and [`_malloca`](http://msdn.microsoft.com/en-us/library/5471dc8s.aspx). `_malloca` should be preferably used over `_alloca` for one reason: `_malloca` allocates the requested number of bytes on the heap instead of the stack if a certain size threshold (`_ALLOCA_S_THRESHOLD`) is exceeded. For that reason, memory allocated with `_malloca` has to be "deallocated" with [`_freea`](http://msdn.microsoft.com/en-us/library/k8984a8h.aspx), to free any memory which might have been allocated on the heap. In debug builds `_malloca` allocates all memory on the heap. GCC provides the [`__builtin_alloca`](https://www.kernel.org/doc/man-pages/online/pages/man3/alloca.3.html) built-in function, which can be used conveniently by including `alloca.h` and calling `alloca`. An example is provided below.

``` cpp
void allocStackDynamic(size_t size)
{
    assert(size > 0);
    char *mem = static_cast<char *>(_malloca(size));
    memset(mem, 0, size);
    _freea(mem);
}
```

# How does Windows manage stack memory?
An interesting question is how actually the compiler performs dynamic stack allocations. To answer that question, first, we have to make a little excursion into the operating system and see how it manages stack memory internally. I will concentrate here on Windows.

A certain amount of each process' virtual memory is used by its threads as stack memory. Windows initially only *reserves* stack memory for a thread and *commits* it on demand, in order to avoid unnecessarily wasting *page-file backed* virtual memory. Reserved memory is a portion of a process' virtual memory, which is not mapped to physical memory yet and hence, is not backed by the page-file. Committed memory in contrast is mapped to physical memory and page-file backed. The amount of reserved and committed memory can be specified by changing the PE header (`SizeOfStackReserve`, `SizeOfStackCommit` fields in the [`IMAGE_OPTIONAL_HEADER`](http://msdn.microsoft.com/en-us/library/windows/desktop/ms680339(v=vs.85).aspx) structure), or directly for a specific thread when calling [`CreateThread`](http://msdn.microsoft.com/en-us/library/windows/desktop/ms682453(v=vs.85).aspx) (`dwStackSize` parameter and the `STACK_SIZE_PARAM_IS_A_RESERVATION` flag). By default, for a new thread 1 MB of memory is reserved for the stack and 4 KB of this memory is initially committed.

In order to know when to grow the amount of committed stack memory, a *guard page* is placed after the last committed page. As the stack grows over time a new page will be touched when accessing memory. That page will be the guard page. When touching a guard page an exception is raised. In that case a new page of already reserved stack memory is committed and the guard page is moved one page further behind the just committed page. In the following figure the situation before and after allocating some memory on the stack are depicted.

![Memory layout: before and after stack growth](http://geidav.files.wordpress.com/2013/03/memory_layout_stack_growth.png)

The stack growth granularity is one page and usually, a page has a size of 4 KB. So what happens, if a function allocates a block of memory on the stack bigger than 4 KB? Consider the following piece of code.

``` cpp
void allocStackStatic()
{
    char mem[5000]; // stack grows downwards:
    mem[0] = 0;     // => &mem[0] < &mem[4999] < previous SP
}
```

Here, 5000 bytes are allocated on the stack and it is written to the byte being furthest away from the previous stack pointer (SP). When executing the example above it can happen, that depending on the current stack situation the guard page that comes after the last committed page is not touched, but the page lying behind the guard page. In that case no new page of stack memory would be committed and the calling process would be terminated, because accessing any page after the guard page causes an access violation. In the following figure two allocations are shown. The first one correctly touching the guard page, the second one missing the guard page and causing an access violation.

![Stack growth: good vs. problematic allocation](http://geidav.files.wordpress.com/2013/03/stack_growth_allocs.png)

Therefore, the compiler has to make sure that the stack grows correctly also for allocations bigger than 4 KB. This can be achieved by gradually touching, or *probing*, memory in 4 KB steps directly after allocation, effectively avoiding ever accessing memory behind the guard page. Sometimes, the memory access patterns of the calling function already touches memory in a way, that no additional probing would be required. Though, the compiler is usually not capable of figuring that out.

## Looking over the compiler's shoulder
So how does the compiler (Visual C++ in my case) implement stack probing? In case of static stack allocations the compiler can detect problematic (too large) stack frames, because their size is already known at compile time. For functions containing such stack frames the compiler emits additional stack probing code, which manifests in a call to the `_chkstk` function actually probing the stack. When performing dynamic stack allocations the `_alloca_probe16` function is called. It allocates a given amount of memory on the stack and calls `_chkstk`afterwards for probing. Implementation-wise memory probing is just a simple `for`-loop gradually touching memory at an address, which is decremented (because stack grows downwards) by the system's page size.

In Visual C++ the `/Gs` compiler flag can be used to specify the maximum stack frame size before the compiler generates stack probing code. To suppress the generation of any stack probing code, a very high threshold value can be specified: `/Gs99999999`. A threshold value of 0 will result in a call to `_chkstk` in every function, independent of its stack frame size.