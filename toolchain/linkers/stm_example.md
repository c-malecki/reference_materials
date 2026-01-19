references STM32F103XX_FLASH.ld

This is a much more complete linker script. Let's compare it to the minimal one and explain the additions.
Same Core Concepts
Both have:

MEMORY block defining FLASH/RAM
_estack symbol
.isr_vector, .text, .rodata, .data, .bss sections

Key Additions in CubeMX Script
Entry Point
ldENTRY(Reset_Handler)
Tells debugger where execution starts. Not strictly required for functionality, but helpful for debugging tools.
Minimum Heap/Stack Sizes
ld_Min_Heap_Size = 0x200;      /* 512 bytes */
_Min_Stack_Size = 0x400;     /* 1024 bytes */
Used later to verify you have enough RAM. If your code exceeds available RAM, linker will error.
Alignment
ld. = ALIGN(4);
. = current location counter
Ensures data is 4-byte aligned. ARM Cortex-M performs best with aligned access. Some instructions require alignment.
Additional .text Sections
ld*(.glue_7)         /* glue arm to thumb code */
*(.glue_7t)        /* glue thumb to arm code */
Compatibility shims for mixing ARM/Thumb code. Cortex-M3 only uses Thumb, so these are typically empty but included for completeness.
ld*(.eh_frame)
KEEP (*(.init))
KEEP (*(.fini))

.eh_frame: Exception handling info (C++ exceptions, usually not used in embedded)
.init/.fini: Constructor/destructor support (C++ global objects)

ARM Exception Tables
ld.ARM.extab (READONLY) :
{
    *(.ARM.extab* .gnu.linkonce.armextab.*)
} >FLASH

.ARM (READONLY) :
{
    __exidx_start = .;
    *(.ARM.exidx*)
    __exidx_end = .;
} >FLASH
Exception unwinding tables - used for C++ exceptions and stack unwinding in debuggers. Rarely needed in bare-metal C, but harmless to include.
C++ Constructor/Destructor Arrays
ld.preinit_array (READONLY) :
{
    PROVIDE_HIDDEN (__preinit_array_start = .);
    KEEP (*(.preinit_array*))
    PROVIDE_HIDDEN (__preinit_array_end = .);
} >FLASH
Similar for .init_array and .fini_array.
Purpose: C++ global objects need constructors called before main(). These arrays store function pointers to call. Startup code iterates through them.
PROVIDE_HIDDEN: Creates symbols only if referenced, keeps them internal.
RamFunc Support
ld.data :
{
    *(.RamFunc)        /* .RamFunc sections */
    *(.RamFunc*)
} >RAM AT> FLASH
Functions that run from RAM - copied from flash to RAM at startup, then executed from RAM. Faster execution for time-critical code. You'd mark functions with __attribute__((section(".RamFunc"))).
Thread-Local Storage (TLS)
ld.tdata : ALIGN(4)
{
    *(.tdata .tdata.* .gnu.linkonce.td.*)
    _edata = .;
} >RAM AT> FLASH

.tbss (NOLOAD) : ALIGN(4)
{
    *(.tbss .tbss.*)
} >RAM
Per-thread variables - each thread gets its own copy. Requires OS/RTOS support. Not used in bare-metal single-threaded code.
Lots of PROVIDE symbols for TLS management.
Heap/Stack Check Section
ld._user_heap_stack (NOLOAD) :
{
    . = ALIGN(8);
    PROVIDE ( end = . );
    PROVIDE ( _end = . );
    . = . + _Min_Heap_Size;
    . = . + _Min_Stack_Size;
    . = ALIGN(8);
} >RAM
Reserves space for minimum heap + stack. If this overflows RAM, linker errors with "region RAM overflowed".
(NOLOAD): Doesn't allocate actual memory, just checks if space exists.
Discard Unused Library Code
ld/DISCARD/ :
{
    libc.a:* ( * )
    libm.a:* ( * )
    libgcc.a:* ( * )
}
Removes standard library code - since you used -nostdlib, this prevents accidentally linking any standard library symbols.
What You Actually Need
For bare-metal learning, the minimal script is better:

Easier to understand
Contains only essentials
No C++ or RTOS overhead

Add later if needed:

Heap/stack size checks
RamFunc support for critical functions
C++ support if using C++