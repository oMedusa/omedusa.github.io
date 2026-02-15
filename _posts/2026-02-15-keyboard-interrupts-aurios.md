---
layout: post
title: "interrupts & keyboard: making AuriOS actually interactive"
date: 2026-02-15
tags: [osdev, interrupts, idt, keyboard]
author: Medusa
---

so i spent the last few days getting keyboard input working in AuriOS. turns out you can't just `scanf()` when there's no OS underneath you. who knew?

here's how i made my kernel actually listen to keypresses without polling like a caveman.

## the problem: how does hardware talk to the CPU?

when you press a key, the keyboard controller needs to tell the CPU "hey, someone just pressed 'a'". but the CPU is busy doing CPU things. that's where **interrupts** come in.

interrupts are basically the hardware's way of saying "stop what you're doing, i need attention NOW". Think of it like having to write code on paper for an exam, annoying but necessary.

## the IDT: a phonebook for interrupts

the **Interrupt Descriptor Table (IDT)** is just a big array that tells the CPU where to jump when an interrupt happens. each entry in the table is 8 bytes and looks like this:

```c
struct idt_entry {
    uint16_t base_low;      // lower 16 bits of handler address
    uint16_t selector;      // kernel code segment (0x08)
    uint8_t always0;        // reserved, always 0
    uint8_t flags;          // type and attributes (0x8E = present, ring 0, 32-bit interrupt gate)
    uint16_t base_high;     // upper 16 bits of handler address
} __attribute__((packed));
```

the CPU needs to know where this table is, so we give it a pointer:

```c
struct idt_ptr {
    uint16_t limit;    // size of IDT - 1
    uint32_t base;     // address of IDT
} __attribute__((packed));
```

my IDT has 256 entries because x86 supports 256 different interrupts:
- 0-31: CPU exceptions (division by zero, page faults, etc.)
- 32-47: hardware interrupts (remapped from IRQ 0-15)
- 48-255: available for software interrupts

setting up an entry is straightforward:

```c
void idt_set_gate(uint8_t num, uint32_t base, uint16_t selector, uint8_t flags)
{
    idt[num].base_low = base & 0xFFFF;
    idt[num].base_high = (base >> 16) & 0xFFFF;
    idt[num].selector = selector;  // 0x08 = kernel code segment
    idt[num].always0 = 0;
    idt[num].flags = flags;        // 0x8E = present, ring 0, 32-bit
}
```

then i just fill the table with all my interrupt handlers and load it with `lidt`:

```nasm
idt_flush:
    mov eax, [esp+4]    ; get pointer to IDT
    lidt [eax]          ; load it
    ret
```

## ISRs: the actual interrupt handlers

when an interrupt fires, the CPU needs assembly stubs to save the current state. i have 32 CPU exception handlers (ISR 0-31) and 16 hardware interrupt handlers (IRQ 0-15, mapped to ISR 32-47).

each stub looks like this:

```nasm
isr0:
    push byte 0        ; dummy error code (some exceptions push one, some don't)
    push byte 0        ; interrupt number
    jmp isr_common     ; jump to common handler
```

the common handler saves all registers and calls my C handler:

```nasm
isr_common:
    pusha              ; push all general purpose registers
    
    mov ax, ds         ; save data segment
    push eax
    
    mov ax, 0x10       ; load kernel data segment
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    
    call isr_handler   ; call C handler
    
    pop eax            ; restore segments
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    
    popa               ; restore registers
    add esp, 8         ; clean up error code and int number
    iret               ; return from interrupt
```

my C handler just displays exceptions for now:

```c
void isr_handler(registers_t *regs)
{
    if (regs->int_no < 32) {
        terminal_writestring("EXCEPTION: ");
        terminal_writestring(exception_messages[regs->int_no]);
        terminal_writestring("\n");
        
        for (;;);  // halt on exception
    }
}
```

## the PIC: why remapping matters

here's where it gets annoying. the **Programmable Interrupt Controller (PIC)** is what sends hardware interrupts to the CPU. but by default, it sends them to interrupts 0-15... which overlap with CPU exceptions. not great.

so i remap them to 32-47 instead:

```c
void pic_remap(void)
{
    // tell both PICs to initialize
    outb(0x20, 0x11);  // PIC1 command
    outb(0xA0, 0x11);  // PIC2 command
    
    // set offsets: PIC1 starts at 32, PIC2 starts at 40
    outb(0x21, 0x20);  // PIC1 data
    outb(0xA1, 0x28);  // PIC2 data
    
    // tell them how they're wired together
    outb(0x21, 0x04);
    outb(0xA1, 0x02);
    
    // set to 8086 mode
    outb(0x21, 0x01);
    outb(0xA1, 0x01);
    
    // mask all interrupts initially
    outb(0x21, 0xFF);
    outb(0xA1, 0xFF);
}
```

now IRQ0 (timer) is at interrupt 32, IRQ1 (keyboard) is at interrupt 33, etc.

## handling hardware interrupts

hardware interrupts (IRQs) work differently than exceptions. i can register custom handlers for each one:

```c
static void (*irq_handlers[16])(registers_t *) = { 0 };

void irq_register_handler(int irq, void (*handler)(registers_t *))
{
    irq_handlers[irq] = handler;
}
```

when an IRQ fires, i acknowledge it by sending EOI (End Of Interrupt) to the PIC, then call the registered handler:

```c
void irq_handler(registers_t *regs)
{
    // send EOI to slave PIC if interrupt came from IRQ8-15
    if (regs->int_no >= 40)
        outb(0xA0, 0x20);
    
    // always send EOI to master PIC
    outb(0x20, 0x20);
    
    // call the registered handler
    if (irq_handlers[regs->int_no - 32] != 0)
        irq_handlers[regs->int_no - 32](regs);
}
```

## finally: the keyboard driver

the keyboard controller sends IRQ1 every time a key is pressed or released. i just need to read the scancode from port 0x60 and translate it to ASCII.

here's my keyboard callback:

```c
void keyboard_callback(registers_t *regs)
{
    uint8_t scancode = inb(0x60);  // read scancode
    
    // handle shift keys
    if (scancode == 0x2A || scancode == 0x36) {
        shift_pressed = 1;
        return;
    }
    if (scancode == 0xAA || scancode == 0xB6) {
        shift_pressed = 0;
        return;
    }
    
    // ignore key release events (bit 7 set)
    if (scancode & 0x80)
        return;
    
    // handle backspace
    if (scancode == 0x0E) {
        shell_handle_key('\b');
        return;
    }
    
    // convert scancode to ASCII
    char c = shift_pressed ? scancode_to_ascii_shift[scancode] 
                           : scancode_to_ascii[scancode];
    
    if (c)
        shell_handle_key(c);
}
```

i use two lookup tables for scancode → ASCII conversion (one for normal, one for shift). not elegant but it works.

to enable the keyboard interrupt, i unmask IRQ1 in the PIC:

```c
void keyboard_init(void)
{
    irq_register_handler(1, keyboard_callback);
    
    uint8_t mask = inb(0x21);  // read current mask
    mask &= ~(0x02);           // clear bit 1 (IRQ1)
    outb(0x21, mask);          // write it back
}
```

## putting it all together

initialization order matters:

1. setup IDT with all handlers
2. remap PIC to avoid conflicts
3. load IDT into CPU with `lidt`
4. enable interrupts with `sti`
5. initialize keyboard driver
6. start getting keypresses

now when you press a key:
1. keyboard controller sends IRQ1
2. PIC converts it to interrupt 33
3. CPU looks up entry 33 in IDT
4. jumps to my IRQ1 handler (in assembly)
5. assembly stub calls `irq_handler` (in C)
6. `irq_handler` sends EOI and calls my keyboard callback
7. keyboard callback reads scancode and converts to ASCII
8. character gets passed to shell

and that's how i can type commands in AuriOS.

## things that broke along the way

- forgot to remap PIC first → triple fault because IRQ0 overlapped with division by zero
- forgot to send EOI → only got one interrupt then everything froze
- forgot to check bit 7 on scancodes → got duplicate characters on key release
- wrong flags in IDT entries → general protection fault on first interrupt

## what's next

the scancode tables are incomplete (no arrow keys, F-keys, etc). also the keyboard driver is super basic — no key repeat, no special key combos, no keyboard LEDs.

but hey, i can type in my OS now. that's pretty cool.

```bash
kernel@auri-os~$ about

        X                 
       XXX                kernel@auri-os
      XXXXX               
     X XXXXX              Kernel: AuriKernel
    XXX XXXXX             Version: 0.2
   XXXXX XXXXX            Release: 2-14-26
  XXXXXX  XXXXX           
 XXXXXX    XXXXX          
XXXXXX      XXXXX         

Type 'help' for available commands
```

> "On everything i made, i don't feel proud, but i cryied on it..." — a Shadow in the night. 

check out the full code on [github](https://github.com/Auri-OS/AuriOS).
