# Understanding Page Tables in Linux
The goal of this recitation is to provide a high-level overview of x86/arm64 paging as
well as the data structures and functions the Linux kernel uses to manipulate
page tables.

Before we dive into the details, here's a quick refresher on what a page table
does and how it does it. Page tables allow the operating system to virtualize
the entire address space for each process by providing a mapping between virtual
addresses and physical addresses. What is a virtual address? Every address that
application code references is a virtual address. For the most part this is true
of kernel code as well, except for code that executes before paging is enabled
(think bootloader). Once paging is enabled in the CPU it cannot be disabled.
Every address referenced by an application is translated transparently by the
hardware via the page table.

A virtual address is usually different than the physical address it maps to, but
it is also possible for virtual addresses to be "identity" mapped to their
corresponding physical address or mapped to no address at all in cases where the
kernel swapped out a page or never allocated one in the first place.

## The shape of a page table
Recall that the structure of the page table is rigidly specified by the CPU
architecture. This is necessary since the CPU hardware directly traverses the
page table to transparently map virtual addresses to physical addresses. The
hardware does this by using the virtual address as a set of indices into the
page table.

<div align='center'>
    <img src='./x86_address_structure.svg'/>
</div>

[Source](https://os.phil-opp.com/page-tables)

This diagram shows how the bits of a 64 bit virtual address specify the indices
into a 4-level x86 page table (you can expect something similar in arm64).
With 4-level paging only bits 0 through 47 are used. Bits 48 through 63 are sign
extension bits that must match bit 47; this prevents clever developers from
stuffing extra information into addresses that might interfere with future addressing
schemes, like 5-level page tables.

As you can see, there are 9 bits specifying the index into each page table and a
12 bit offset into the physical page frame. Since 2^9 = 512, each level of the
page table has 512 64 bit entries, which fits perfectly in one 4096 byte frame.
The last 12 bits allow the virtual address to specify a specific byte offset
within the page frame.

<div align='center'>
    <img src='./X86_Paging_64bit.svg'/>
</div>

For clarity, we're using the naming scheme in the diagram (P4, P3,...), which
differs slightly from the names used in Linux. Linux also implements 5-level
paging, which we describe in the next section. For reference, here is how the
names above mapped to Linux before it added 5-level paging:

```
Diagram: P4  -> P3  -> P2  -> P1  -> page frame
Linux:   PGD -> PUD -> PMD -> PTE -> page frame
```

Each entry in the P4 table is the address of a *different* P3 table such that
each page table can have up to 512 different P3 tables. In turn, each entry in a
P3 table points to a different P2 table such that there are 512 * 512 = 262,144
P2 tables. Since most of the virtual address space is unused for a given
process, the kernel does not allocate frames for most of the intermediary tables
comprising the page table.

I've been using the word frame since each page table entry (in any table, P4,
P3, etc) is the address of a physical 4096 byte memory frame (except for huge
pages). These addresses cannot be virtual addresses since the hardware accesses
them directly to translate virtual addresses to physical addresses. Furthermore,
since frames are page aligned, the last 12 bits of a frame address are all
zeros. The hardware takes advantage of this by using these bits to store
information about the frame in its page table entry.

<div align='center'>
<img src='./Page_table.png'/>
</div>

[Source](https://wiki.osdev.org/Paging)

This diagram shows the bit flags for a page table entry, which in our diagram
above corresponds to an entry in the P1 table. A similar but slightly different
set of flags exists for the intermediary entries as well.

## Working with page tables in Linux
In this section we'll take a look at the data structures and functions the Linux
kernel defines to manage page tables.

### The fifth level of Dante's page table
We just finished discussing 4-level page tables, but Linux actually implements
5-level paging and exposes a 5-level paging interface, even when the kernel is
built with 5-level paging disabled. Luckily 5-level paging is similar to the
4-level scheme we discussed, is simply adds another level of indirection and
uses 9 previously unused bits of the 64 bit virtual address to index it.

Here's what the page table hierarchy looks like in Linux:

```
4-Level Paging (circa 2016)
Diagram Above
    P4  -> P3  -> P2  -> P1  -> page frame
Linux
    PGD -> PUD -> PMD -> PTE -> page frame
    
5-Level Paging (current)
Linux
    PGD -> P4D -> PUD -> PMD -> PTE -> page frame
```

What does this mean for us in light of the fact that our kernel config specifies
4 level paging?

```
CONFIG_PGTABLE_LEVELS=4
```

If we look at the output from an example program that reports the physical
addresses of the process' page frame and of the intermediate page tables,
it shows that the `p4d_paddr` and `pud_paddr` are identical.

```
[405] inspect_physical_address():
paddr: 0x115a3c069
pf_paddr: 0x115a3c000
pte_paddr: 0x10d2c7000
pmd_paddr: 0x10d623000
pud_paddr: 0x10c215000
p4d_paddr: 0x10c215000
pgd_paddr: 0x10c90a000
dirty: no
refcount: 57
```

Digging into `arch/x86/include/asm/pgtable_types.h`, we see the following:

```C
#if CONFIG_PGTABLE_LEVELS > 4
typedef struct { p4dval_t p4d; } p4d_t;

static inline p4d_t native_make_p4d(pudval_t val)
{
	return (p4d_t) { val };
}

static inline p4dval_t native_p4d_val(p4d_t p4d)
{
	return p4d.p4d;
}
#else
#include <asm-generic/pgtable-nop4d.h>

static inline p4d_t native_make_p4d(pudval_t val)
{
	return (p4d_t) { .pgd = native_make_pgd((pgdval_t)val) };
}

static inline p4dval_t native_p4d_val(p4d_t p4d)
{
	return native_pgd_val(p4d.pgd);
}
#endif
```
[x86 Source](https://elixir.bootlin.com/linux/v5.10.57/source/arch/x86/include/asm/pgtable_types.h#L332)

Interesting. Looking at `pgtable-nop4d.h` we find that `p4d_t` is defined as
```
typedef struct { pgd_t pgd; } p4d_t;
```

With 4-level paging the p4d folds into the pgd. p4d entries, which are
represented by `p4d_t`, essentially become a type alias for `pgd_t`. The kernel
does this so that it has a standard 5-level page table interface to program
against regardless of how many levels of page tables actually exist.

As of writing, arm64 (for linux 5.10.57) directly includes `pgtable-nop4d.h`.

To summarize, with 4-level paging there are no "real" p4d tables. Instead, pgd
entries contain the addresses of pud tables, and the kernel "pretends" the p4d
exists by making it appear that the p4d is a mirror copy of the pgd.

If you read on in `arch/x86/include/asm/pgtable_types.h` you'll see that the kernel uses the same
scheme for 3 and 2 level page table configurations as well. arm64 follows a similar scheme in `arch/arm64/include/asm/pgtable-types.h`

NOTE that you cannot make use of this equivalence directly. Your Farfetch'd
implementation must work correctly for any page table configuration and
therefore must use the macros defined by the kernel.

### Data structures, functions, and macros
In this section we'll take a step back and discuss the data structures
and functions the kernel uses to manage page tables in more detail.

To encapsulate memory management information for each task, `struct task_struct` contains
a `struct mm_struct*`.

```C
struct mm_struct                *mm;
struct mm_struct                *active_mm;
```

We won't go into the details of `active_mm`, which is used for kernel threads that do not
have their own `struct_mm`. Check out `context_switch()` in core.c if you want to read
more.

Looking in `struct mm_struct`, we find the member `pgd_t *pgd;`. This is a
pointer to the first entry in the pgd for this `mm_struct`. Do you think that
this is a virtual address or a physical address? Remember that all memory
references are translated from virtual addresses to physical addresses by the
CPU, so any address the kernel code uses directly must be a virtual address.

However, it's easy to recover the physical address since all kernel addresses
are linearly mapped to physical addresses. `virt_to_phys` recovers the physical
address using this linear mapping.


Section 3.3 in Gordman's [chapter on page table
management](https://www.kernel.org/doc/gorman/html/understand/understand006.html)
provides a good overview of the functions / macros used to navigate the page table.

A common source of confusion arises from a misunderstanding of what macros like `pud_offset`
return. 

```C
/* Find an entry in the third-level page table.. */
// From include/linux/pgtable.h. Note that this definition is shared between x86 and arm64.
static inline pud_t *pud_offset(p4d_t *p4d, unsigned long address)
{
	return (pud_t *)p4d_page_vaddr(*p4d) + pud_index(address);
}

// x86: arch/x86/include/asm/pgtable_types.h
// arm64: arch/arm64/include/asm/pgtable-types.h
typedef struct { pudval_t pud; } pud_t;

// x86: arch/x86/include/asm/pgtable_64_types.h
typedef unsigned long   pudval_t;

// arm64: arch/arm64/include/asm/pgtable-types.h
typedef u64 pudval_t;
```

We touched on this briefly above. A `pud_t` is just a struct containing an
unsigned long, which means it compiles down to an unsigned long. Recall from our
earlier discussion that each entry is the address of a physical page frame and
that the last 12 bits are reserved for flags since each page frame is page
aligned. The macros that Gordman discusses, like `pte_none()` and
`pmd_present()`, check these flag bits to determine information about the entry.

If you want to recover the actual value of the entry, the type casting macros
Gordman discussed in section 3.2 are useful. Although the Gordman reading is x86-specific,
arm64 defines similar, if not indentical, macros. Keep in mind that if you want the
physical address the entry points to you'll need to bitwise-and the value with the
correct mask.

x86 and arm64 either define functions/macros for the mask so you can
manually perform the bitwise-and or define function/macros that outright do the correct
bitwise-and for you. Either way, recovering the physical address the entry points to is possible in
both x86 and arm64, it just may look slightly different depending which architecture you're on and
which function/macros you choose.

### Page dirty and refcount
Recall from before that a flag in the page table entry indicates whether the
page frame is dirty or not. Do not read the flag directly; the kernel provides
a macro for this purpose.

You will find section 3.4 of Gordman useful for figuring out how to retrieve
the refcount of a page frame. Hint: every physical frame has a `struct page` in
the kernel, which is defined
[here](https://elixir.bootlin.com/linux/v5.10.57/source/include/linux/mm_types.h#L70).
Be sure to use the correct kernel functions / macros to access any information
in `struct page`.

