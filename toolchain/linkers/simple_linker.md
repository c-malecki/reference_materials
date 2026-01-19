# Linker script explained by Claude

```ld
MEMORY
{
    FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 128K
    RAM (rwx)  : ORIGIN = 0x20000000, LENGTH = 20K
}

_estack = ORIGIN(RAM) + LENGTH(RAM);

SECTIONS
{
    .text :
    {
        KEEP(*(.isr_vector))
        *(.text*)
        *(.rodata*)
        _etext = .;
    } > FLASH

    .data :
    {
        _sdata = .;
        *(.data*)
        _edata = .;
    } > RAM AT > FLASH

    _sidata = LOADADDR(.data);

    .bss :
    {
        _sbss = .;
        *(.bss*)
        *(COMMON)
        _ebss = .;
    } > RAM
}
```

## MEMORY Block

```ld
MEMORY
{
    FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 128K
    RAM (rwx)  : ORIGIN = 0x20000000, LENGTH = 20K
}
```

**Defines physical memory regions:**

- `FLASH (rx)`: Read + execute permissions. Code goes here.
- `RAM (rwx)`: Read + write + execute. Data goes here.
- `ORIGIN`: Starting address from datasheet
- `LENGTH`: Size of region (K = kilobytes)

The linker uses this to know what memory exists and where to place things.

## Stack Pointer

```ld
_estack = ORIGIN(RAM) + LENGTH(RAM);
```

**Creates a symbol for top of stack:**

- `ORIGIN(RAM)` = `0x20000000`
- `LENGTH(RAM)` = `0x5000` (20KB)
- `_estack` = `0x20005000` (end of RAM)

ARM stacks grow downward, so stack starts at the top of RAM and grows down. Your startup code will use `_estack` as the initial stack pointer.

## SECTIONS Block

This tells the linker how to arrange your code/data.

**.text Section (Code)**

```ld
.text :
{
    KEEP(*(.isr_vector))
    *(.text*)
    *(.rodata*)
    _etext = .;
} > FLASH
```

**What goes in .text:**

- `KEEP(*(.isr_vector))`: Vector table (interrupt handlers). KEEP means never remove it even if unused.
- `*(.text*)`: All code from all files (functions)
- `*(.rodata*)`: Read-only data (const variables, string literals)
- `_etext = .;`: Symbol marking end of .text

`> FLASH`**: Place this section in FLASH memory**

**Why this order?** The vector table MUST be first because the CPU reads it at address `0x08000000` on reset.

**.data Section (Initialized Variables)**

```ld
.data :
{
    _sdata = .;
    *(.data*)
    _edata = .;
} > RAM AT > FLASH
```

**Initialized global/static variables:**

- `_sdata`: Start of .data in RAM
- `*(.data*)`: All initialized variables
- `_edata`: End of .data in RAM

`> RAM AT > FLASH`**: This is crucial**

- **Run location (VMA)**: RAM - where variables live during execution
- **Load location (LMA)**: FLASH - where initial values are stored

**Why? Flash is non-volatile, RAM is volatile. Initial values must be stored in flash, then copied to RAM at startup.**

**Load Address Symbol**

```ld
_sidata = LOADADDR(.data);
```

**Address in flash where .data initial values are stored**

Your startup code will copy from `_sidata` (flash) to `_sdata` (RAM) for `_edata - _sdata` bytes.

**.bss Section (Uninitialized Variables)**

```ld
.bss :
{
    _sbss = .;
    *(.bss*)
    *(COMMON)
    _ebss = .;
} > RAM
```

**Zero-initialized variables:**
- `_sbss`: Start of .bss in RAM
- `*(.bss*)`: Uninitialized variables
- `*(COMMON)`: Common symbols (old C compatibility)
- `_ebss`: End of .bss

**`> RAM`** only - not `AT > FLASH`

**Why?** Variables initialized to zero don't need to store zeros in flash. Startup code just zeros this region.

## Memory Layout After Linking

```
FLASH (0x08000000):
├── Vector table (.isr_vector)
├── Code (.text)
├── Read-only data (.rodata)
└── Initial values for .data (_sidata)

RAM (0x20000000):
├── Initialized variables (.data) ← copied from flash
├── Zero-initialized variables (.bss) ← zeroed at startup
└── Stack (grows down from _estack)
```

**Startup Code Will Use These Symbols**

- `_estack`: Set initial stack pointer
- `_sidata`, `_sdata`, `_edata`: Copy .data from flash to RAM
- `_sbss`, `_ebss`: Zero out .bss in RAM

Then call `main()`.