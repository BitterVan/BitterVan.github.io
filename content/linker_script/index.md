+++
title = "Linker Script Segments"
date = 2022-04-06
[taxonomies]
tags = ["Compilers", "Operating System"]
categories = ["Research"]
+++

Linker is under the control of linker script, thus it can put the codes and variables at the correct place in a logical address space. There are 4 basic segments:

* `.text` holding the programs.
* `.data` holding initialized data.
* `.rodata` holding read-only data.
* `.bss` acronym for "Block Starting Symbols", holding uninitialize data in a program.

<!-- more -->

Here shows a linker script used in building rCore of THU:

```linker script
OUTPUT_ARCH(riscv)
ENTRY(_start)
BASE_ADDRESS = 0x80200000;

SECTIONS
{
	. = BASE_ADDRESS;
	skernel = .;

	stext = .;
	.text : {
		*(.text.entry)
		*(.text .text.*)
	}
	. = ALIGN(4K);
	etext = .;

	srodata = .;
	.rodata : {
		*(.rodata .rodata.*)
		*(.srodata .srodata.*)
	}
	. = ALIGN(4K);
	erodata = .;

	sdata = .;
	.data : {
		*(.data .data.*)
		*(.sdata .sdata.*)
	}
	. = ALIGN(4K);
	edata = .;

	.bss : {
		*(.bss.stack)
		sbss = .;
		*(.bss .bss.*)
		*(.sbss .sbss.*)
	}
	. = ALIGN(4K);
	ebss = .;
	ekernel = .;

	/DISCARD/ : {
		*(.eh_frame)
	}
}
```

In the script, instruction with `. = <some_address>` or `<some_address> = .` is quite helpful interacting with current address `.`. 

And instruction `ALIGN(SIZE)` calculates the next address that is aligned to the given block size. Here we use page size to ensure the segments are scattered in different pages, since the priviledge control of hardware is usually with a granularity of page size.

The extra tags, like `edata`, `ebss` defined here can be accessed as tags in other source code files like this

```rust
fn clear_bss() {
	extern "C" {
		fn sbss();
		fn ebss();
	}
	(sbss as usize..ebss as usize).for_each(|a| {
		unsafe { (a as *mut u8).write_volatile(0) }
	});
}
```