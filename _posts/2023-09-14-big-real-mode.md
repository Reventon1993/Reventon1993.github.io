---
layout: post
title: "big real mode"
category: x86
tags: [x86]
---
{% include JB/setup %}


## big real mode

特殊的实模式(CR0.PE=0)，此模式下，数据寻址可以大于1MB。同时他依旧可以使用实模式的bios中断服务，这就是该模式的意义。

进入该模式步骤:置位CR0.PE进入32位模式(保护模式)，然后重载数据段寄存器，获取大于1MB的数据寻址能力。然后复位CR0.PE，退出到实模式。

因为段寄存器的缓存机制(重载才会清除缓存)，依旧保存大于1MB的数据寻址能力。

具体代码如下：

    [SECTION gdt]  
    LABEL_GDT:      dd  0,0 
    LABEL_DESC_CODE32:  dd  0x0000FFFF,0x00CF9A00
    LABEL_DESC_DATA32:  dd  0x0000FFFF,0x00CF9200
    GdtLen  equ $ - LABEL_GDT
    GdtPtr  dw  GdtLen - 1 
        dd  LABEL_GDT
    SelectorCode32  equ LABEL_DESC_CODE32 - LABEL_GDT
    SelectorData32  equ LABEL_DESC_DATA32 - LABEL_GDT
    
    cli 
    db  0x66
    lgdt    [GdtPtr]
    
    mov eax,    cr0
    or  eax,    1
    mov cr0,    eax  
    mov ax, SelectorData32
    mov fs, ax
    mov eax,    cr0
    and al, 11111110b
    mov cr0,    eax
      
    sti


