Assembly Directives (Header)

```asm
.syntax unified
.cpu cortex-m3
.fpu softvfp
.thumb
```

Tells assembler how to compile:

.syntax unified: Use unified ARM/Thumb assembly syntax
.cpu cortex-m3: Target Cortex-M3 instruction set
.fpu softvfp: Software floating point (no FPU hardware)
.thumb: Use Thumb instruction set (required for Cortex-M3)

Global Symbols
asm.global g_pfnVectors
.global Default_Handler
Makes these symbols visible to linker (can be referenced from other files).
Linker Symbol References
asm.word _sidata
.word _sdata
.word _edata
.word _sbss
.word _ebss
Declares symbols defined in linker script - these are just references here, actual addresses come from linker. These .word directives reserve space in the binary and will be filled with the actual addresses by the linker.
Boot RAM Constant
asm.equ  BootRAM, 0xF108F85F
.equ = assembly constant. This magic value is used at the end of the vector table for booting from RAM (special STM32 feature).
Reset_Handler Function
asm.section .text.Reset_Handler
.weak Reset_Handler
.type Reset_Handler, %function
Reset_Handler:
Function declaration:

.section .text.Reset_Handler: Put this code in its own section
.weak: Weak symbol - can be overridden by another definition
.type Reset_Handler, %function: Tell debugger this is a function

Step 1: System Init
asmbl  SystemInit
bl = branch with link (function call). Calls SystemInit() - typically sets up clocks. You'd provide this function elsewhere (or stub it out).
Step 2: Copy .data Section
asmldr r0, =_sdata          /* Destination: RAM start */
ldr r1, =_edata          /* Destination: RAM end */
ldr r2, =_sidata         /* Source: Flash location */
movs r3, #0              /* Offset counter = 0 */
b LoopCopyDataInit       /* Jump to loop check */

CopyDataInit:
  ldr r4, [r2, r3]       /* Load from flash[offset] */
  str r4, [r0, r3]       /* Store to RAM[offset] */
  adds r3, r3, #4        /* offset += 4 bytes */

LoopCopyDataInit:
  adds r4, r0, r3        /* r4 = current RAM address */
  cmp r4, r1             /* Compare with end address */
  bcc CopyDataInit       /* If less, continue copying */
Copies initialized data from flash to RAM:

Uses registers r0-r4 as workspace
r3 tracks offset (increments by 4 bytes each iteration)
bcc = branch if carry clear (unsigned less than)

Why the offset approach? Allows copying word-by-word efficiently.
Step 3: Zero .bss Section
asmldr r2, =_sbss           /* Start of bss */
ldr r4, =_ebss           /* End of bss */
movs r3, #0              /* Value to write (zero) */
b LoopFillZerobss        /* Jump to loop check */

FillZerobss:
  str  r3, [r2]          /* Write zero to [r2] */
  adds r2, r2, #4        /* r2 += 4 bytes */

LoopFillZerobss:
  cmp r2, r4             /* Compare current with end */
  bcc FillZerobss        /* If less, continue */
Zeros uninitialized variables:

r2 increments through memory
Writes r3 (which is 0) to each location
Simpler than .data copy (no source to read from)

Step 4: Call C++ Constructors
asmbl __libc_init_array
Calls global C++ constructors (from .init_array in linker script). For pure C, this does nothing but doesn't hurt.
Step 5: Jump to main
asmbl main
bx lr

bl main: Call your main function
bx lr: Branch to link register (return)

Note: main() should never return in embedded, but if it does, this returns to... nowhere. Better to add infinite loop here.
Default_Handler
asm.section .text.Default_Handler,"ax",%progbits
Default_Handler:
Infinite_Loop:
  b Infinite_Loop
  .size Default_Handler, .-Default_Handler
Fallback for unhandled interrupts:

.section .text.Default_Handler,"ax",%progbits: Put in separate section, allocatable + executable
b Infinite_Loop: Branch to self (infinite loop)
.size: Tell debugger the function size

If an interrupt fires without a handler, CPU jumps here and hangs. Debugger can catch this.
Vector Table
asm.section .isr_vector,"a",%progbits
.type g_pfnVectors, %object
.size g_pfnVectors, .-g_pfnVectors

g_pfnVectors:
  .word _estack
  .word Reset_Handler
  .word NMI_Handler
  .word HardFault_Handler
  /* ... many more ... */
The actual interrupt vector table:

.section .isr_vector: Goes in .isr_vector section (linker places this at 0x08000000)
.type g_pfnVectors, %object: This is data, not code
.word: Each entry is 32-bit address

First 16 Entries (ARM Standard)

_estack: Initial stack pointer (not a function!)
Reset_Handler: What runs on reset
NMI_Handler: Non-maskable interrupt
HardFault_Handler: CPU fault handler
5-15. Other ARM core exceptions (MemManage, BusFault, SVC, etc.)

Entries 7-9 and 13 are 0 (reserved by ARM).
Entries 16+ (STM32 Specific)
asm.word WWDG_IRQHandler
.word PVD_IRQHandler
/* ... */
.word USART1_IRQHandler
/* ... */
STM32F103 peripheral interrupts - order and count come from reference manual. F103xB has ~60 interrupts total.
Last Entry
asm.word BootRAM
Special STM32 boot mode feature - allows booting from RAM instead of flash.
Weak Aliases
asm.weak NMI_Handler
.thumb_set NMI_Handler,Default_Handler

.weak HardFault_Handler
.thumb_set HardFault_Handler,Default_Handler

/* ... repeated for every interrupt ... */
Makes all handlers point to Default_Handler by default:

.weak: Symbol can be overridden
.thumb_set: Set this symbol equal to another (alias)

This means: If you don't implement USART1_IRQHandler, it automatically uses Default_Handler (infinite loop). If you DO implement it, your version overrides the weak symbol.
Key Concepts
Why assembly?

Need precise control before C runtime is ready
Must manipulate registers directly
Vector table must be exact format

Can you write startup in C?

Vector table: no (must be assembly or complex C)
Reset_Handler: yes, but need inline assembly for some parts
Most people use assembly for startup, C for everything else

What would you write yourself?
For learning, you'd write a minimal version:

Simpler vector table (just core + few peripherals you use)
Skip SystemInit call (or provide empty stub)
Skip __libc_init_array (C++ support)
Keep .data copy, .bss zeroing, call to main

