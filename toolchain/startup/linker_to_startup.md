What the Linker Script Does
The linker script is a blueprint that:

Defines memory regions (where things CAN go)
Defines sections (how to organize code/data)
Creates symbols (_estack, _sdata, etc.)
Tells the linker where to place your compiled code

But it doesn't execute anything. It's just instructions for the linking process.
What Startup Code Does
Startup code is actual executable code that:

Uses the symbols the linker created
Runs on the actual hardware
Performs initialization before main()

The Connection

1. Linker Script Creates Symbols

```ld
ld_estack = ORIGIN(RAM) + LENGTH(RAM);    /* Creates symbol */
_sdata = .;                              /* Creates symbol */
_edata = .;                              /* Creates symbol */
```

These are just addresses - they don't do anything by themselves.

2. Startup Code Uses Those Symbols

```c
extern uint32_t _estack;   /* Reference linker symbol */
extern uint32_t _sdata;    /* Reference linker symbol */
extern uint32_t _edata;    /* Reference linker symbol */

// Startup code uses them:
uint32_t *src = &_sidata;
uint32_t *dst = &_sdata;
while (dst < &_edata) {
    *dst++ = *src++;       /* Copy data from flash to RAM */
}
```

The linker calculated the addresses, startup code uses them to actually perform operations.

3. Vector Table Placement

Linker script says:

```ld
.isr_vector :
{
    KEEP(*(.isr_vector))
} >FLASH
```

"Put anything in section `.isr_vector` at the start of FLASH"

Startup code provides:

```c
__attribute__((section(".isr_vector")))
uint32_t vector_table[] = {
    (uint32_t)&_estack,      /* Uses _estack from linker */
    (uint32_t)Reset_Handler, /* Address of reset function */
    // ... more handlers
};
```

The linker sees this array is in .isr_vector section and places it at 0x08000000 in FLASH.

The Flow
Compile time:

Compiler compiles startup code → startup.o
Compiler compiles your code → main.o

Link time:
3. Linker reads linker script
4. Linker sees .isr_vector section in startup.o
5. Linker places it at start of FLASH
6. Linker calculates all the _estack, _sdata, etc. addresses
7. Linker resolves references in startup code to those symbols
8. Linker produces firmware.elf
Runtime (on hardware):
9. CPU powers up, reads address 0x08000000
10. Finds vector table (placed there by linker)
11. Loads stack pointer from entry 0 (which contains _estack value)
12. Jumps to Reset_Handler (entry 1)
13. Reset_Handler uses _sdata, _edata, etc. to initialize memory
14. Reset_Handler calls main()

What's Missing Between Them

The linker script declares the structure.
The startup code implements the behavior.

Neither works without the other:

Linker script without startup code → no vector table, no initialization
Startup code without linker script → symbols undefined, don't know where to put things