.. title: What I have learned from resurrecting my operating system
.. slug: what-i-have-learned-from-resurrecting-my-operating-system
.. date: 2017-06-28 15:30:37 UTC+03:00
.. tags: osdev, c, qemu, assembly
.. category:
.. link:
.. description:
.. type: text

Back at school in 2011 when I was still studying software engineering,
I implemented a `simple operating system <https://github.com/povilasb/simple-os>`_
for i386 architecture.  When I wrote it, I did not have too much experience
with makefiles, automated tests, builds, etc. Thus the project does not look
too nice.
For a long time I was eager to refactor some parts, port to C++, maybe use
llvm and some of nice C++ features to better organize the code.
Fortunately, I wish I could use C++11 or above, maybe even implement 64bit
support and run it on arm architecture :)

I just spent a week figuring out how to compile and run the OS.
And eventually I've learned some new things. Some of it I knew, but forgot,
some if it was completely new to me :)

QEMU
====

When I was implementing the OS, I used `bochs
<https://en.wikipedia.org/wiki/Bochs>`_ to test it.
But this time `QEMU <http://www.qemu.org/>`_ seemed more attractive to me.

I can run QEMU with floppy disk image::

    $ qemu-system-i386 -fda sos.img

Alternatively I can tell QEMU to emulate usb disk::

    $ qemu-system-i386 -usb -usbdevice disk:sos.img
    $ qemu-system-i386 -usb -usbdevice disk:/dev/sdb

I've learned that if I press `ctrl+alt+shift+2` when OS is running inside QEMU,
it will open console which allows me to control the emulated machine.

I can dump physical memory at given address::

    xp /10xb 0x7E00

* 10 - number of items to display
* x - print numbers in hex format
* b - item size is one byte

I can scroll console with `ctrl+page up/down`.

Assembly
========

My simple OS has a small handcrafter bootloader written in assembly language.
I use `nasm <https://en.wikipedia.org/wiki/Netwide_Assembler>`_ to produce
machine code.

I can produce raw binaries with nasm::

    $ nasm -f bin boot.asm -o boot.bin

To disassemble I use `objdump`::

    $ objdump -D -b binary -m i386 -M intel boot.bin

where:

* `-D` means disassemble all segments
* `-b binary` specifies file format
* `-m i386` specifies architecture
* `-M intel` specifies assembly language syntax

Also, it was new to me that nasm has `local labels
<https://www.tortall.net/projects/yasm/manual/html/nasm-local-label.html>`_.
Meaning you can have multiple labels with the same name.
Local labels must go after regular labels and start with a period:

.. code-block:: asm

    label1  ; some code
    .loop   ; some more code
            jne .loop
            ret
    label2  ; some code
    .loop   ; some more code
            jne .loop
            ret

OS loading
==========

When computer starts, BIOS loads the first sector (512 bytes) of a bootable
device.
My bootloader (at least first stage) has to fit into those 512 bytes.
Then it is able to read some more data: other parts of bootloader or kernel.

BIOS provides basic functionality to read some data from floppy or USB disk.
In real mode BIOS emulates USB as floppy. Thus reading from USB is identical
to reading from floppy.

There are two functions to read data from disk:

* `int 0x13, ah=0x2 <https://en.wikipedia.org/wiki/INT_13H#INT_13h_AH.3D02h:_Read_Sectors_From_Drive>`_
* `int 0x13, ah=0x42 <https://en.wikipedia.org/wiki/INT_13H#INT_13h_AH.3D42h:_Extended_Read_Sectors_From_Drive>`_

The difference is that `ah=0x2` is older and uses CHS (Cylinder, Head, Sector)
addressing. Whereas `ah=0x42` uses linear addressing.
Personally, I prefer `ah=0x42`, because it's more intuitive to me.
And it's use looks more or less like this:

.. code-block:: asm

    boot_disk db 0x80 ; disk number
    dapack:
            db 0x10
            db 0
    .count: dw 0 ; int 13 resets this to # of blocks actually read/written
    .buf:   dw 0 ; memory buffer destination address
    .seg:   dw 0 ; in memory page zero
    .addr:  dq 1 ; skip 1st disk sector which is bootloader, which is loaded by BIOS

    mov ax, 127
    mov [dapack.count], ax
    mov ax, 0x7E00
    mov [dapack.buf], ax

    mov dl, [boot_disk]
    mov si, dapack
    mov ah, 0x42
    int 0x13

Memory paging
=============

When I was finally able to boot the OS, all I saw was page faults.
I had completely forgotten how i386 paging works, except that page size
usually is 4096 bytes :)

MMU
---

In protected mode MMU (Memory Management Unit) translates the virtual address
the running process is trying to access into physical address. MMU consults
CR3 register and two tables: page directory and page table::

                                    Page Tables                  Pages
                                  +--------------+           +--------------+
                                  |              |---------> |              |
                                  +--------------+           |              |
                                  |              |-----+     |              |
                                  +--------------+     |     |              |
                           +----> |              |     |     |              |
      Page Directory       |      +--------------+     |     +--------------+
     +--------------+      |                           |
     |              |------+      +--------------+     |     +--------------+
     +--------------+             |              |     +---->|              |
     |              |------+      +--------------+           |              |
     +--------------+      |      |              |           |              |
     |              |      |      +--------------+           |              |
     +--------------+      +----> |              |           |              |
                                  +--------------+           +--------------+

I had completely forgotten about control registers.
Basically `CR3 <https://en.wikipedia.org/wiki/Control_register#CR3>`_ is
a 32bit register that holds the address of page directory.

MMU algorithm::

    page_directory_addr = CR3
    page_table_addr = page_directory_addr[virtual_addr[31:22]]
    page_addr = page_table[virtual_addr[21:12]]
    physical_addr = page_addr[virtual_addr[11:0]]

Context swithing
----------------

When it was pretty much clear how paging works, it was easier to investigate
page faults.
After playing around for some time, it was clear that context switching
was causing the page faults. So I started to refresh my knowledge how pages
are refreshed for running processes.

Kernel holds physical addresses of every page a process owns. Virtual memory
for every process starts a 0x0. So what task scheduler does, when it switches
the processes, it alters the `Page Tables` entries to point to new pages.
Turns out everything was ok with this procedure.

The BUG was that, when new pages were linked, MMU would not start using
them immediately because of `TLB
<https://en.wikipedia.org/wiki/Translation_lookaside_buffer>`_.
I did not know about TLB before, but basically it's a cache of virtual to
physical address mappings.
Flushing TLB is pretty easy. I only have to update CR3 register:

.. code-block:: c

    void
    set_page_directory(PageDirectory *page_dir)
    {
        asm volatile ("mov %0, %%eax;"
            "mov %%eax, %%cr3"
            : : "r" (page_dir)
            : "eax"
            );
    }

So, I updated OS task scheduler to update CR3 every time context switch
happened. And that fixed the page faults.
After that the OS ran smoothly, just like 6 years ago :)

Summary
=======

I'm happy I spent 5 days on this. I've learned some QEMU, nasm, objdump
features, refreshed my knowledge of i386 boot process and memory management.
And now I have a working environment that empowers me to play with real
devices, misc CPU instructions, test some OS design ideas, etc.
