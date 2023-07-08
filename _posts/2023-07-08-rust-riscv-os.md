---
layout: post
title: Things I learned making a simple RISC-V OS with Rust
---
<!-- vim: set textwidth=80 colorcolumn=80: -->

Recently I've been reading a lot about RISC-V and wanted to use it for something
to learn how it works. I decided to go ballistic and
[make an operating system][rust-riscv-os]. I already delved into OSdev before,
but it was with i386, and in C. RISC-V seems much simpler, and since that time I
fell in love with Rust. I did some initial exploration in C following the
[RISC-V Bare Bones][riscv-bare] and [RISC-V Meaty Skeleton][riscv-meaty]
tutorials in the [OSDev wiki][os-dev], and then started following the
[Adventures of OS][adventures-of-os] tutorial, which uses Rust. Here are a few
things I learned along the way.

## SMP doesn't need to be initialized

One of the first things I wanted to know about how RISC-V systems work is how
[SMP][smp] works. I thought you had to initialize the other harts[^1], as you
[apparently have to do in x86][smp-x86], and I wasted an entire morning
searching how to do that in [the spec][riscv-spec] and in the internet, but
turns out they all just start at the same time running in the same address, at
least in the QEMU virt machine.

If you don't want the different harts to do redundant or conflicting work, as
they all run the same instructions in the beginning, you have to park[^2] them.
According to section 3.1.5 of the volume 2 of [the spec][riscv-spec]:

> The `mhartid` CSR is an MXLEN-bit read-only register containing the integer ID
> of the hardware thread running the code. This register must be readable in any
> implementation. Hart IDs might not necessarily be numbered contiguously in a
> multiprocessor system, but at least one hart must have a hart ID of zero. Hart
> IDs must be unique within the execution environment.

So the only hart we can be certain we can identify is the one with `mhartid =
0`. We can then park all the harts with `mhartid != 0` and use the one with
`mhartid = 0` as a bootstrap hart to setup the system.

```text
_start:
    # Park all harts except the one with mhartid = 0
    csrr t0, mhartid
    bnez t0, 1f
    
    # Bootstrapping
    # ...
    
1:
    # Threads with mhartid != 0 go here
    # The wfi instruction is a hint that the hart should sleep and wait for an
    # interrupt. However, a valid implementation can just be a nop, so we have
    # to put this in a loop just in case. In QEMU the wfi instruction actually
    # puts the hart to sleep so it saves a lot of cpu.
    wfi
    j 1b
```

Later we could set the interrupts in a way that could allow harts to communicate
and we have SMP. But that seems very complicated so I'm staying single-threaded
for now.

## Device trees

I knew that to do IO in embedded systems like an operating system kernel you had
to do memory mapped IO, which is basically reading and writing magic values to
magic places in the address space to do magic things, but I always wondered how
you were actually supposed to know _what_ magic values and places you had to
use. I knew you could simply look at the documentation of the device you are
using, but even then you don't exactly know where in the address space it is
connected. You would have to look at the documentation of the machine you are
using[^3] to know how the manufacturer connected things up.

But this poses a interesting problem: how do you support multiple machines? Do
you need to have a separate linker script for each machine? I don't think Linux
does that. And apparently [they don't][linux-devicetree], through something
called a [device tree][devicetree].

A device tree is a standard data structure that describes the hardware of a
machine. The system provides the bootloader some way to read the device tree,
that way your kernel or embedded application can be aware of the hardware
available and in which addressed they are connected.
[As we will see later](#qemu-virt-prelude), QEMU virt passes the device tree in a
register at boot.

There are a few formats for a device tree, the two we will be looking at are the
binary one with extension .dtb, and the text one with extension .dts. We can
dump the device tree of a QEMU machine with the `-machine dumpdtb=filename.dtb`
option:

```text
$ qemu-system-riscv64 -machine virt,dumpdtb=virt.dtb
qemu-system-riscv64: info: dtb dumped to virt.dtb. Exiting.
```

We can then convert it to the text version with the program `dtc`:

```text
$ dtc virt.dtb -o virt.dts
```

And now we have a description of the hardware of the QEMU virt machine. If you
open this file you will see things like the following:

```text
memory@80000000 {
    device_type = "memory";
    reg = <0x00 0x80000000 0x00 0x8000000>;
};
```

This tells us that the device `memory` is mapped at address `0x8000_0000`. This
is where the RAM starts in this machine. Another one:

```text
serial@10000000 {
    interrupts = <0x0a>;
    interrupt-parent = <0x03>;
    clock-frequency = "\08@";
    reg = <0x00 0x10000000 0x00 0x100>;
    compatible = "ns16550a";
};
```

This tells us that the device `serial` is mapped at address `0x1000_0000`, and
that it is compatible with the `ns16550a` UART device. There's much more info in
these, I recommend reading [this][devicetree-usage] if you are interested in
understanding this format.

A last one that confused me for a while:

```text
poweroff {
    value = <0x5555>;
    offset = <0x00>;
    regmap = <0x04>;
    compatible = "syscon-poweroff";
};
```

This is clearly the device you use for shutting down the system. But differently
from the others above, this one doesn't have an address. It took me a while to
figure this out, but this isn't the actual device, this is just a specification
of the magic value (and offset) you need to send the device for the system to
shutdown. Well, _but where is the device???_ Turns out you have to look at the
`regmap` field and find the device that has the corresponding `phandle` field.
In this case is this one:

```text
test@100000 {
    phandle = <0x04>;
    reg = <0x00 0x100000 0x00 0x1000>;
    compatible = "sifive,test1\0sifive,test0\0syscon";
};
```

It being named `test` is a bit suspicious. A device for something as common as
shutting down your system shouldn't be named like that! However the
`compatible` field contains `syscon`, which is also cited in the `poweroff`
entry. So I tested the following:

```text
poweroff:
    # Store the value 0x5555 into the address of the test device
    li t0, 0x5555
    li t1, 0x100000
    sw t0, (t1)
```

And it worked! Weird flex but ok.

## UART

The protocol for serial communication is called [UART][uart]. It is very simple,
but it took me some time to understand it[^4]. In the QEMU virt machine, if you pass
`-serial mon:stdio` as an argument then your terminal is connected to the serial
device we saw above at address `0x1000_0000`. The basic
functionality of UART is very simple: to receive you read from this address, to
transmit you write to this address.

```rust
const uart_ptr = 0x1000_0000 as *mut u8;

fn uart_receive() -> u8 {
    unsafe {
        uart_ptr.read_volatile()
    }
}

fn uart_transmit(u8: byte) {
    unsafe {
        uart_ptr.write_volatile(byte);
    }
}
```

### Pooling

A hurdle I had with UART is when to know that a new character arrived. From
tests I did, if I run a loop with the function `uart_receive()` above I will end
up infinitely getting the last character sent through the serial. There are 2
ways of dealing with this. One is interrupts, which I didn't implement yet. The
other one is to pool the UART device to check if a new byte arrived. Something
like:

```rust
fn uart_receive() -> u8 {
    unsafe {
        loop {
            // We need to check the first bit of the byte at offset 5 of the
            // UART. Offset 5 is the Line Status Register. It's first bit
            // signals if there is data available.
            if (uart_ptr.add(5).read_volatile & 1) == 1 {
                break;
            }
        }
        uart_ptr.read_volatile()
    }
}
```

This isn't very good, it's a busy loop and makes my computer go hot after some
time with QEMU open even though it's not doing anything. I think we can't do any
better without interrupts though.

### Backspace sent as delete

A quirk I'm not sure if is my terminal, tmux or some other thing's fault, but
that [some other people also have][backspace-stack], is that when I press
backspace (byte `0x08`) in the terminal, the UART inside QEMU receives delete
(byte `0x7F`).

## Timer interrupts

One thing that gave me a lot of headache is that my code wasn't reaching my
kernel main function in Rust. I tried using gdb to debug my kernel but
failed[^5]. My previous attempt in C booted just fine, and if I used its boot
code with my Rust project it also worked, so the problem was there. This was the
boot code of the C project:

```text
_start:
    # Some code to set stack, global pointer and clear bss section
    #...
    
    la t0, kmain
    csrw mepc, t0

    # Jump to kernel!
    tail kmain
```

This was the code for the Rust project:

```text
_start:
    # Some code to set stack, global pointer and clear bss section
    #...
    
    # Here I set mstatus to return to M mode with interrupts enabled later
    # when I call mret
    li t0, (0b11 << 11) | (1 << 7) | (1 << 3)
    csrw mstatus, t0

    # Put the address of kmain in the mepc, which is the address we will jump
    # when mret is called
    la t1, kmain
    csrw mepc, t1

    # Set up the trap handler
    la t2, asm_trap_vector
    csrw mtvec, t2

    # Enable interrupts
    # bit 3: software interrupts 
    # bit 7: timer interrupts
    # bit 11: external interrupts
    li t3, (1 << 3) | (1 << 7) | (1 << 11)
    csrw mie, t3
    
    mret
    
asm_trap_vector:
    # No interrupt handling for now
    mret
```

In the Rust version we use `mret` instead of `tail`. That is necessary for
enabling interrupts which I will need later.

After sprinkling this code with snippets of assembly sending letters through
UART, I discovered the problem was with the line setting the `mie` to enable
interrupts. Specifically it was the `mtie` bit (the bit 7), for timer
interrupts. After sprinkling the code with a assembly functions that dumps `mip`
(a CSR register indicating what interrupts are pending) to UART, I confirmed
that there was always a timer interrupt when reaching this line of code. When
enabling timer interrupts in `mie` the hart trapped instantly, and went to the
`asm_trap_vector` handler. Here I dumped `mcause` to UART and confirmed it was a
timer interrupt that caused the trap[^6]. It instantly calls `mret`, which
returns to the place where the interrupt was activated. But as I didn't handle
the interrupt, it was still pending in `mip`, so before even stepping a
instruction it trapped to `asm_trap_vector` again, in a infinite loop.

Turns out that if you don't handle interrupts they are not handled I guess. As I
don't need timer interrupts for now, I will keep them disabled until later:

```text
    # Enable interrupts
    # bit 3: software interrupts 
    # bit 11: external interrupts
    li t3, (1 << 3) | (1 << 11)
    csrw mie, t3
```

Now it gets to the Rust code. UART debugging ftw.

## QEMU virt prelude

While attempting to debug the timer interrupt thing with gdb I discovered
something interesting. When you run QEMU with the option `-s` it listens for gdb
in port 1234 of localhost, and with option `-S` it waits for a command of gdb
(or the QEMU monitor) before starting the CPU. We can then connect from gdb with
`target extended-remote localhost:1234` and debug the whole system. This means I
could see where the machine boots, which to my surprise isn't at `0x8000_0000`
at the start of the RAM where the start of my kernel is loaded, but at `0x1000`,
after the first 8 bytes of the address space[^7]. This is what is there:

```text
(gdb) x/6i $pc
=> 0x1000:      auipc   t0,0x0
   0x1004:      add     a2,t0,40
   0x1008:      csrr    a0,mhartid
   0x100c:      ld      a1,32(t0)
   0x1010:      ld      t0,24(t0)
   0x1014:      jr      t0
(gdb) x/g $pc+24
0x1018: 0x0000000080000000
(gdb) x/g $pc+32
0x1020: 0x0000000087e00000
(gdb) x/g $pc+40
0x1028: 0x000000004942534f
```

The code loads the `mhartid` into the `a0` register, some values into `t0`, `a1`
and `a2`, and then jumps to `t0`. The last 3 commands show the value of `t0`
(that we jump to), the value of `a1` and the value that `a2` points to after
this initialization. Let's see what each of them mean.

The value of `t0` that we jump to is easy, it's `0x8000_0000`, which is the
start of RAM, where we put the kernel. Makes sense.

The value of `a1` is more interesting. It's an address at the end of RAM[^8]. If
we follow it we see a bunch of data:

```text
(gdb) x/16w 0x87e00000
0x87e00000:     0xedfe0dd0      0xe6100000      0x38000000      0x340f0000
0x87e00010:     0x28000000      0x11000000      0x10000000      0x00000000
0x87e00020:     0xb2010000      0xfc0e0000      0x00000000      0x00000000
0x87e00030:     0x00000000      0x00000000      0x01000000      0x00000000
```

The first 4 bytes are the magic number for the header of the device tree binary
format! From [the specification][devicetree-spec]:

> magic
>
> This field shall contain the value `0xd00dfeed` (big-endian).

`0xedfe0dd0` is `0xd00dfeed` in little-endian! The system gives it's device tree
for us! This means we could make a kernel that parses it and adjusts itself to
the hardware available! Very cool.

I read somewhere that passing the `mhartid` in `a0` at boot is almost standard
in the world of RISC-V. I wonder if passing a device tree in `a1` also is.

`a2` for me is still a mystery. It points to the value `0x4942_534f`, which
seems to be a pointer to near the end of the `pci` device. Reading it just gives
bytes filled with `0xff`.

```text
(gdb) x/16w 0x4942534f
0x4942534f:     0xffffffff      0xffffffff      0xffffffff      0xffffffff
0x4942535f:     0xffffffff      0xffffffff      0xffffffff      0xffffffff
0x4942536f:     0xffffffff      0xffffffff      0xffffffff      0xffffffff
0x4942537f:     0xffffffff      0xffffffff      0xffffffff      0xffffffff
```

[^1]: A hart is short for "hardware thread". One CPU core might have more than
    one hardware thread.

[^2]: Parking a hart here means making it wait for an interrupt to do work.

[^3]: Or, in the case of the QEMU virt machine I was using, I suppose
    [reading the source code][qemu-virt], as I wasn't able to find it's
    documentation.

[^4]: [This][uart-tutorial] helped a lot. Especially the example code.

[^5]: It froze in some specific places for reasons beyond my comprehension. I
    suspect it has something to do with I not having implemented interrupts yet.

[^6]: For some time I was really confused because it seemed like a software
    interrupt. But then I noticed that I was only showing the highest 63 bits of
    `mcause`. For software interrupts it ends in `...0011`, for timer interrupts
    it ends in `...0111`.

[^7]: I checked and they are not readable, probably to avoid null pointer
    dereferences or something like this.

[^8]: From the device tree you can derive what is the size of the memory. I
    didn't explain this here, but it has to do with the `reg` field in the
    `memory` device.

[rust-riscv-os]: https://github.com/bakaq/rust-riscv-os
[riscv-bare]: https://wiki.osdev.org/RISC-V_Bare_Bones
[riscv-meaty]: https://wiki.osdev.org/RISC-V_Meaty_Skeleton_with_QEMU_virt_board
[os-dev]: https://wiki.osdev.org
[adventures-of-os]: https://osblog.stephenmarz.com/index.html
[smp]: https://en.wikipedia.org/wiki/Symmetric_multiprocessing
[smp-x86]: https://wiki.osdev.org/SMP
[riscv-spec]: https://github.com/riscv/riscv-isa-manual
[qemu-virt]: https://github.com/qemu/qemu/blob/stable-8.0/hw/riscv/virt.c#L77-L98
[linux-devicetree]: https://en.wikipedia.org/wiki/Devicetree#Linux
[devicetree]: https://www.devicetree.org/
[uart]: https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter
[uart-tutorial]: https://wiki.osdev.org/Serial_Ports
[backspace-stack]: https://stackoverflow.com/questions/70214434/uart-driver-for-qemu-receiving-delete-byte-instead-of-backspace
[devicetree-spec]: https://www.devicetree.org/specifications/
[devicetree-usage]: https://elinux.org/Device_Tree_Usage
