[[Twitter Posting]]
My friend has had the most heartbreaking experience of his lifetime with someone who he thought was his girlfriend. 

Man got played like a PS4 in Covid y'all, it's insane what happened to him. 

He was talking to this girl for about 8 months, and he really thought that he's got her undivided attention, and secretly, he fell in love with her. He really thought that he has all of her heart, soul and spirit, but then - doomsday arrived for him. 

It turns out, this girl had him in genjutsu of the HIGHEST order. She was talking to not one, not two, but THREE lads at the same damn time. And all those thought that they were the only ones who had her - JUST like multiple processes believe they have some control and access over the entire memory space, while in reality, it's all just virtual memory space that is mapped to physical memory that's managed by something called to MEMORY MANAGEMENT UNIT. 

Now that you are here, let's see how this plays out in the first place - 

Whenever you execute a program on your computer, the operating system creates a _process_ to manage its execution. This process needs memory to store its instructions, variables, stack, heap, and all the other data it needs to function.

Now, traditionally, you might imagine that a program simply gets handed a chunk of physical RAM and operates directly on it. But there's a problem with that approach.

What happens when multiple processes are running simultaneously? How do we prevent one process from accidentally overwriting another process's data? What happens when a program needs more memory than is currently available in physical RAM?

We need a mechanism that allows programs to operate independently of the actual physical memory available to them. Enter **Virtual Memory**.

The fundamental idea behind virtual memory is simple: instead of having every process directly access physical memory addresses, we give each process its own _virtual address space_.

A virtual address space is the range of memory addresses that a process can use. When a program accesses a variable at a particular address, that address is generally a virtual address, not a direct reference to a physical location in RAM.

This means that two different processes can use the same virtual address without necessarily accessing the same physical memory location.

For example, imagine two processes, A and B.

Process A accesses virtual address `0x1000`. Process B also accesses virtual address `0x1000`.

Despite using the same address, the two processes can access completely different physical memory locations.

How?

Because the virtual addresses are translated into physical addresses through a mechanism managed by the operating system and supported by the processor's hardware.

And that's where things get interesting.

## The Memory Management Unit (MMU)

The **Memory Management Unit (MMU)** is a hardware component responsible for translating virtual addresses into physical addresses and enforcing memory-access permissions.

When a process attempts to access memory, the processor generates a virtual address. The MMU uses the relevant address-translation information to determine the corresponding physical address, provided the access is valid.

Let's consider our earlier example.

|Process|Virtual address|Physical address|
|---|---|---|
|A|`0x1000`|`0x8000`|
|B|`0x1000`|`0xC000`|

Both processes access the same virtual address, `0x1000`, but the MMU translates it into different physical addresses depending on the active address-space mappings.

The processes don't need to know where their data resides in physical RAM. They operate using virtual addresses, while the underlying hardware and operating system handle the translation.

But where does the MMU get this mapping information from?

**Page tables.**

## Page Tables and Memory Mapping

Managing a separate mapping for every individual byte of memory would be incredibly inefficient. Instead, most modern virtual-memory systems divide virtual address spaces into fixed-size blocks called _pages_.

Physical memory is divided into corresponding blocks called _frames_.

A page table contains the information needed to map virtual pages to physical frames, along with attributes such as access permissions and other status information.

For example, suppose a system uses 4 KiB pages. A virtual address can be divided into two parts:

- **Virtual page number:** Identifies the virtual page being accessed.
    
- **Page offset:** Identifies the byte within that page.
    

The page number is used to find the corresponding physical frame. The offset remains unchanged because the byte's position within the page doesn't change during translation.

Let's walk through a simplified example.

Suppose a process accesses virtual address `0x1234`, and the system uses 4 KiB pages.

Since 4 KiB equals `0x1000` bytes, the virtual address can be separated into:

- Virtual page number: `0x1`
    
- Offset within the page: `0x234`
    

Suppose virtual page `0x1` maps to physical frame `0x8`.

The physical address is then:

`0x8000 + 0x234 = 0x8234`

So, virtual address `0x1234` translates to physical address `0x8234`.

This is a simplified illustration of address translation. Actual page-table structures and translation procedures vary between processor architectures.

The important takeaway is that **the virtual page determines the physical frame, while the offset identifies the location within that frame**.

## The Translation Lookaside Buffer (TLB)

At this point, you might be wondering: if every memory access requires the MMU to consult page tables, wouldn't that introduce additional overhead?

Absolutely.

Memory accesses are already performance-critical, and repeatedly walking through multiple levels of page tables would be expensive.

This is where the **Translation Lookaside Buffer (TLB)** comes in.

The TLB is a small, fast hardware cache that stores recently used address translations.

When the processor accesses a virtual address, the MMU checks whether the required translation is available in the TLB.

If it is, the translation can be retrieved quickly. This is called a _TLB hit_.

If the translation isn't available, it's a _TLB miss_. The processor must obtain the translation through the architecture's page-table-walking mechanism or another applicable mechanism. Once retrieved, the translation may be cached in the TLB for future use.

A TLB miss is not the same thing as a page fault.

A TLB miss means the translation wasn't found in the TLB. A page fault means the attempted memory access requires operating-system intervention, for example because a page isn't currently mapped in a usable way or because the access violates the relevant permissions.

The distinction matters because a TLB miss can be resolved entirely through a page-table walk without requiring the operating system to intervene.

## Memory Protection and Process Isolation

Virtual memory isn't just about making memory easier to manage. It's also one of the mechanisms that allows an operating system to isolate processes from one another.

Page-table entries can specify whether a page is readable, writable, executable, accessible from user mode, or subject to other architecture-specific restrictions.

When a process attempts an access that violates the applicable permissions, the processor can raise an exception, allowing the operating system to handle it.

For example, a process attempting to write to a read-only page may trigger a protection fault.

Similarly, a process generally cannot access another process's private memory simply by guessing its virtual address. Its address space has its own mappings and permissions, and the hardware enforces those restrictions.

This is an essential part of building a stable, secure operating system. A fault in one application shouldn't automatically allow it to corrupt the memory of every other application running on the machine.

Of course, isolation isn't absolute in every circumstance. Operating systems can deliberately establish shared-memory mappings, and privileged software has access to capabilities unavailable to ordinary user processes. The exact protections also depend on the processor architecture and system configuration.

## Does Virtual Memory mean that every process gets its own RAM?

No. This is an important distinction.

Virtual memory does not mean that every process receives a dedicated chunk of physical RAM equal to the size of its virtual address space.

A process can have a large virtual address space without all of it being backed by physical memory at any given moment.

Some pages may be mapped to physical RAM. Others may be backed by executable files, shared libraries, memory-mapped files, or swap space. Some regions may simply be reserved as part of the process's address space without having physical pages allocated to them yet.

Physical pages can also be shared between processes where appropriate.

For example, multiple processes running the same executable may share physical pages containing read-only program instructions. Each process can have those pages mapped into its own virtual address space without requiring a separate physical copy of the instructions for every process.

Writable memory can be handled differently, including through mechanisms such as _copy-on-write_, where physical pages can initially be shared and separate copies created when a process modifies them.

This is one of the reasons virtual memory is so powerful: it separates the address space a program works with from the physical resources currently used to back it.

## What happens when a page isn't in RAM?

Let's say a process attempts to access a virtual page that isn't currently mapped to a usable physical page.

The processor may raise a **page fault**, transferring control to the operating system.

A page fault isn't necessarily an error. It is a mechanism through which the operating system can respond to certain memory accesses that require additional work.

Depending on the situation, the OS might:

- Allocate a new physical page to satisfy a valid memory access.
    
- Load a page of data from an executable or memory-mapped file.
    
- Retrieve a page from swap space, if applicable.
    
- Establish or update a mapping for a valid access.
    
- Reject the access if it violates the process's address-space rules or memory permissions.
    

Once a recoverable fault has been handled, the processor can retry the instruction that triggered it.

However, if the access is invalid or cannot be resolved, the operating system may terminate the process or report an appropriate error.

This mechanism allows the operating system to manage physical memory dynamically instead of requiring every potentially used page to be loaded into RAM before a process begins execution.

It also introduces trade-offs. Fetching data from storage is much slower than accessing RAM, and excessive paging can severely degrade system performance.

Virtual memory is therefore not a magical way to create unlimited physical memory. It is an abstraction and management mechanism that helps the operating system use available resources effectively.

## Bringing it all together

Virtual memory is the result of cooperation between the operating system and the processor's memory-management hardware.

The operating system manages process address spaces, establishes mappings, allocates memory, and handles page faults. The MMU translates virtual addresses into physical addresses and enforces hardware-supported access permissions. The TLB accelerates address translation by caching recently used mappings.

Together, these mechanisms provide several important benefits:

- **Isolation:** Processes can operate in separate virtual address spaces.
    
- **Protection:** Memory permissions restrict which accesses are allowed.
    
- **Flexibility:** Virtual memory can be mapped to physical RAM, files, shared regions, and other backing resources.
    
- **Efficient resource utilisation:** Physical memory can be allocated as needed and shared where appropriate.
    
- **A simpler programming model:** Applications generally don't need to manage the physical placement of every object they access.
    

The next time you write a program and access a variable, remember that the address involved is usually a virtual address. The processor and operating system work together to determine how that access relates to the underlying physical memory.

And that's the beauty of abstraction in computer architecture: you get a relatively simple interface to work with, while an extraordinary amount of complexity remains hidden underneath.