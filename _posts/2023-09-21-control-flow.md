---
layout: post
title: "control flow"
category: compute system
tags: [x86,kernel,virt]
---
{% include JB/setup %}


## control flow

control flow表示执行的指令序列：

> In computer science, **control flow** (or **flow of control**) is the order in which individual statements, instructions or function calls of an imperative program are executed or evaluated. 



通常情况下，处理器执行机器级指令是连续的，也就是control flow是"平滑的"。但是下面场景下，指令的执行并不是连续的：

1. 同步的jump等指令，我们称之为control transfer。
2. 异步的Exceptional Control Flow(异常控制流、ECF)。

注意：intel SDM中的control transfer概念与我们这里的定义有些不同，它认为control flow不仅是同步的，还是"主动的"。



下面讨论这些流程中的硬件机制和软件规范。

## control transfer

### jump

jump(跳转)指令分成两类：条件跳转(jcc)、非条件跳转(jmp)

#### jcc

jcc(Jump if Condition Is Met)指令，只有满足条件时才会跳转。条件有很多种，所以具体的jcc指令也有很多种。比如:

| Opcode | Instruction | Op/En | 64-Bit Mode | Compat/Leg Mode | Description                          |
| ------ | ----------- | ----- | ----------- | --------------- | ------------------------------------ |
| 77 cb  | JA rel8     | D     | Valid       | Valid           | Jump short if above (CF=0 and ZF=0). |

**所有的跳转指令都是short jump**

#### jmp

jump指令分成下面四类:

> •Near jump—A jump to an instruction within the current code segment (the segment currently pointed to by the
> CS register), sometimes referred to as an intrasegment jump.
> •Short jump—A near jump where the jump range is limited to –128 to +127 from the current EIP value.
> •Far jump—A jump to an instruction located in a different segment than the current code segment but at the
> same privilege level, sometimes referred to as an intersegment jump.
> •Task switch—A jump to an instruction located in a different task.

**task switch和task gate有关，所以只存在于保护模式。**

**near jump就是通常所说的相对寻址跳转指令，可以认为它是一个特殊的short jump：**

| Opcode | Instruction | Op/En | 64-Bit Mode | Compat/Leg Mode | Description                                                  |
| ------ | ----------- | ----- | ----------- | --------------- | ------------------------------------------------------------ |
| EB cb  | JMP rel8    | D     | Valid       | Valid           | Jump short, RIP = RIP + 8-bit displacement sign extended to 64-bits |

### call

call(调用)指令分成下面四类：

> •Near Call — A call to a procedure in the current code segment (the segment currently pointed to by the CS
> register), sometimes referred to as an intra-segment call.
> •Far Call — A call to a procedure located in a different segment than the current code segment, sometimes
> referred to as an inter-segment call.
> •Inter-privilege-level far call — A far call to a procedure in a segment at a different privilege level than that
> of the currently executing program or procedure.
> •Task switch — A call to a procedure located in a different task.

**后两者与trap gate、task gate有关，所以只存在于保护模式(64位模式下已经取消了trap gate、task gate)。**

通常情况下，call和ret指令是成对出现的(当然也可以不是)。

ret指令一共有四个：

| Opcode | Instruction | Op/En | 64-Bit Mode | Compat/Leg Mode | Description                                                  |
| ------ | ----------- | ----- | ----------- | --------------- | ------------------------------------------------------------ |
| C3     | RET         | NP    | Valid       | Valid           | Near return to calling procedure.                            |
| CB     | RET         | NP    | Valid       | Valid           | Far return to calling procedure.                             |
| C2 iw  | RET imm16   | I     | Valid       | Valid           | Near return to calling procedure and pop imm16 bytes from stack. |
| CA iw  | RET imm16   | I     | Valid       | Valid           | Far return to calling procedure and pop imm16 bytes from stack. |

**RET imm16就是通常所说的retn指令。**

绝大部分使用call&ret的场景是程序中的函数调用，其使用near call&near return。这种场景下涉及[calling conventions](https://en.wikipedia.org/wiki/X86_calling_conventions)，常见的规范有stdcall、cdecl、fastcall。GNU C使用cdecl，下面重点讨论。

#### cdecl

cdecl规定了在函数调用过程中栈和通用寄存器的使用规范。我们这里只讨论X86_64位模式下的规范：

- 优先使用RDI、RSI、RCDX、RCX、R8、R9这6个寄存器保存传递的参数，超出的参数使用栈进行传递。

- 使用RAX保存返回结果。

- 被调用者可以任意使用保存传参的寄存器和RAX寄存器，若使用其他寄存器，那么需要先保存，在返回时还原。

- 如果使用栈传递参数，那么参数是从右向左压入栈中。

- 在函数返回时，被调用者完成栈的平衡(保存传参的栈属于调用者的栈)。栈的平衡策略使用rbp，所以在rbp在GNU C中被称为

  帧指针(栈帧指针)，具体的策略如下：

  ```
  <function>
  push %rbp
  ...
  ...
  leaveq  //= mov %rbp,%rsp;pop %rbp
  retq
  ```

下面简单看一个示例

```
//src.c
int f1(int a1, int a2, int a3, int a4,
        int a5, int a6, int a7, int a8){
    return a1+a2+a3+a4+a5+a6+a7+a8;
}

int main(){
    int result=0;
    result = f1(1, 2, 3, 4, 5, 6, 7, 8);
    return result;
}


//assembly
0000000000000000 <f1>:
   0:	f3 0f 1e fa          	endbr64 
   4:	55                   	push   %rbp
   5:	48 89 e5             	mov    %rsp,%rbp
   8:	89 7d fc             	mov    %edi,-0x4(%rbp)
   b:	89 75 f8             	mov    %esi,-0x8(%rbp)
   e:	89 55 f4             	mov    %edx,-0xc(%rbp)
  11:	89 4d f0             	mov    %ecx,-0x10(%rbp)
  14:	44 89 45 ec          	mov    %r8d,-0x14(%rbp)
  18:	44 89 4d e8          	mov    %r9d,-0x18(%rbp)
  1c:	8b 55 fc             	mov    -0x4(%rbp),%edx
  1f:	8b 45 f8             	mov    -0x8(%rbp),%eax
  22:	01 c2                	add    %eax,%edx
  24:	8b 45 f4             	mov    -0xc(%rbp),%eax
  27:	01 c2                	add    %eax,%edx
  29:	8b 45 f0             	mov    -0x10(%rbp),%eax
  2c:	01 c2                	add    %eax,%edx
  2e:	8b 45 ec             	mov    -0x14(%rbp),%eax
  31:	01 c2                	add    %eax,%edx
  33:	8b 45 e8             	mov    -0x18(%rbp),%eax
  36:	01 c2                	add    %eax,%edx
  38:	8b 45 10             	mov    0x10(%rbp),%eax
  3b:	01 c2                	add    %eax,%edx
  3d:	8b 45 18             	mov    0x18(%rbp),%eax
  40:	01 d0                	add    %edx,%eax
  42:	5d                   	pop    %rbp
  43:	c3                   	ret    

0000000000000044 <main>:
  44:	f3 0f 1e fa          	endbr64 
  48:	55                   	push   %rbp
  49:	48 89 e5             	mov    %rsp,%rbp
  4c:	48 83 ec 10          	sub    $0x10,%rsp
  50:	c7 45 fc 00 00 00 00 	movl   $0x0,-0x4(%rbp)
  57:	6a 08                	push   $0x8
  59:	6a 07                	push   $0x7
  5b:	41 b9 06 00 00 00    	mov    $0x6,%r9d
  61:	41 b8 05 00 00 00    	mov    $0x5,%r8d
  67:	b9 04 00 00 00       	mov    $0x4,%ecx
  6c:	ba 03 00 00 00       	mov    $0x3,%edx
  71:	be 02 00 00 00       	mov    $0x2,%esi
  76:	bf 01 00 00 00       	mov    $0x1,%edi
  7b:	e8 00 00 00 00       	call   80 <main+0x3c>
  80:	48 83 c4 10          	add    $0x10,%rsp
  84:	89 45 fc             	mov    %eax,-0x4(%rbp)
  87:	8b 45 fc             	mov    -0x4(%rbp),%eax
  8a:	c9                   	leave  
  8b:	c3                   	ret   
```

**注意：这里使用的gcc版本比较高，直接使用mov写栈(效率高)，所以也就没有leaveq指令。**



### 异常/软件中断

在intel SDM volume1 6.4中，这样对中断&异常定义。

> The processor provides two mechanisms for interrupting program execution, interrupts and exceptions:
> •An interrupt is an asynchronous event that is typically triggered by an I/O device.
> •An exception is a synchronous event that is generated when the processor detects one or more predefined
> conditions while executing an instruction. The IA-32 architecture specifies three classes of exceptions: faults,
> traps, and aborts.

中断和异常的区别仅仅在于是否同步。

但在intel SM volume3 6.3中，又把同步的软件中断认为是中断的一种，这其实是前后矛盾的。

> The processor receives interrupts from two sources:
> •External (hardware generated) interrupts.
> •Software-generated interrupts.

异常和软件中断都是同步的，两者的区别在与软件中断是程序"主动"发起的，软中断命令包括：INT n 、INTO 、BOUND。

#### 中断(硬件)机制

在硬件层面上，异常、硬件中断、软件中断的机制是一样的。下面所述的"中断"指的是这种硬件机制。

每个中断都会有一个中断号，每个中断号对应了IDT中的一个中断描述符。异常、硬件中断、软件中断对应不同类型的中断描述符。

中断发生后，control flow会跳转到中断描述符中指定的位置，同时栈也会发生变化：

<img src="https://raw.githubusercontent.com/Reventon1993/pictures/master/typora/control_flow_pic1.png" alt="control_flow_pic1" style="zoom: 33%;" />

- error code只有部分异常中才有。且#PG(页异常)的error code非常特殊。
- 如果需要切换栈，新栈的信息来自TSS。有两种场景需要切换栈：特权级发生变化、中断描述符使用IST栈。

#### iret

iret是中断返回命令。64位模式下，一般使用下面指令：

| Opcode     | Instruction | Op/En | 64-Bit Mode | Compat/Leg Mode | Description                             |
| ---------- | ----------- | ----- | ----------- | --------------- | --------------------------------------- |
| REX.W + CF | IRETQ       | NP    | Valid       | N.E.            | Interrupt return (64-bit operand size). |

#### 中断处理程序

只有软件中断中才涉及到参数的传递，目前在x86_64位系统中已经很少使用软件中断。所以这里也不深入探讨。

除了参数传递外，中断处理程序需要完成寄存器的保存和恢复、栈的平衡。下面是一个中断处理程序的示例：

```
	pushq   $0   //if no error code , push fake error code  
    pushq   %rax
    movq    %es,    %rax
    pushq   %rax
    movq    %ds,    %rax
    pushq   %rax
    pushq   %rbp;       
    pushq   %rdi;       
    pushq   %rsi;       
    pushq   %rdx;       
    pushq   %rcx;       
    pushq   %rbx;       
    pushq   %r8;        
    pushq   %r9;        
    pushq   %r10;       
    pushq   %r11;       
    pushq   %r12;       
    pushq   %r13;       
    pushq   %r14;       
    pushq   %r15;
	...
	...
	popq    %r15;       
    popq    %r14;       
    popq    %r13;       
    popq    %r12;       
    popq    %r11;       
    popq    %r10;       
    popq    %r9;        
    popq    %r8;        
    popq    %rbx;       
    popq    %rcx;       
    popq    %rdx;       
    popq    %rsi;       
    popq    %rdi;       
    popq    %rbp;       
    popq    %rax;       
    movq    %rax,   %ds;    
    popq    %rax;       
    movq    %rax,   %es;    
    popq    %rax;       
    addq    $0x10,  %rsp;   
    iretq;
```



### sysenter

早期os中的系统调用是通过INT实现，现在改用了更高效的sysenter指令。

#### syscall vs sysenter

https://wiki.osdev.org/SYSENTER

考虑到兼容性，在64位模式下，使用syscall/sysret指令。但是我们还是使用sysenter/sysexit论述。

#### sysenter

完成特权级3到特权级0的控制转移(control transfer)，转移过程就是重载CS、SS、RIP、RSP。

- IA32_SYSENTER_EIP，64位的MSR寄存器(0x176)，保存转移后的RIP地址。

- IA32_SYSENTER_ESP，64的位MSR寄存器(0x175)，保存转移后的RSP地址。

- IA32_SYSENTER_CS，32位的MSR寄存器(0x174)。IA32_SYSENTER_CS[15:0]保存特权级0的代码段选择子，特权级0的栈段选择子

  为代码段选择子+8。

#### sysexit

完成特权级0到特权级3的控制转移，转移过程就是重载CS、SS、RIP、RSP。

- RDX保存转移后的RIP地址

- RCX保存转移后的RSP地址。

- IA32_SYSENTER_CS，IA32_SYSENTER_CS[15:0]+32保存特权级3的代码段选择子，特权级3的栈段选择子

  为代码段选择子+8。

根据IA32_SYSENTER_CS的描述可知，内核的代码段、数据段、用户的代码段和内核段在GDT表的位置是有要求的。下面是一个满足要求的示例：

```
GDT_Table:
    .quad   0x0000000000000000          /*0 NULL descriptor             00*/
    .quad   0x0020980000000000          /*1 KERNEL  Code    64-bit  Segment 08*/
    .quad   0x0000920000000000          /*2 KERNEL  Data    64-bit  Segment 10*/
    .quad   0x0000000000000000          /*3 USER    Code    32-bit  Segment 18*/
    .quad   0x0000000000000000          /*4 USER    Data    32-bit  Segment 20*/
    .quad   0x0020f80000000000          /*5 USER    Code    64-bit  Segment 28*/
    .quad   0x0000f20000000000          /*6 USER    Data    64-bit  Segment 30*/
    .quad   0x00cf9a000000ffff          /*7 KERNEL  Code    32-bit  Segment 38*/
    .quad   0x00cf92000000ffff          /*8 KERNEL  Data    32-bit  Segment 40*/
    .fill   10,8,0                  /*10 ~ 11 TSS (jmp one segment <9>) in long-mode 128-bit 50*/
GDT_END:
```



#### 系统调用

sysenter命令有统一的处理程序入口。该处理程序通过系统调用号来区分系统调用，系统调用号保存在RAX中。

下面是一个参考linux的系统调用实现，我们着重关注数据的传递。

内核态实现的系统调用：

```
wrmsr(0x174,KERNEL_CS);
wrmsr(0x175,current->thread->rsp0);
wrmsr(0x176,(unsigned long)system_call);


ENTRY(system_call)
    sti 
    subq    $0x38,  %rsp     
    cld;                      
    pushq   %rax;                   
    movq    %es,    %rax;               
    pushq   %rax;                   
    movq    %ds,    %rax;               
    pushq   %rax;                   
    xorq    %rax,   %rax;               
    pushq   %rbp;                   
    pushq   %rdi;                   
    pushq   %rsi;                   
    pushq   %rdx;                   
    pushq   %rcx;                
    pushq   %rbx;                   
    pushq   %r8;                    
    pushq   %r9;                    
    pushq   %r10;                
    pushq   %r11;                
    pushq   %r12;                   
    pushq   %r13;                
    pushq   %r14;                   
    pushq   %r15;                   
    movq    $0x10,  %rdx;               
    movq    %rdx,   %ds; 
    movq    %rax,   %es
    movq    RAX(%rsp),  %rax    //RAX is the offset of rax in stack
    leaq    system_call_table(%rip),    %rbx
    callq   *(%rbx,%rax,8)
    movq    %rax,   RAX(%rsp)
ENTRY(ret_system_call)
    movq    %rax,   0x80(%rsp)
    popq    %r15
    popq    %r14
    popq    %r13
    popq    %r12
    popq    %r11
    popq    %r10
    popq    %r9
    popq    %r8
    popq    %rbx
    popq    %rcx
    popq    %rdx
    popq    %rsi
    popq    %rdi
    popq    %rbp
    popq    %rax
    movq    %rax,   %ds
    popq    %rax
    movq    %rax,   %es
    popq    %rax
    addq    $0x38,  %rsp
    xchgq   %rdx,   %r10
    xchgq   %rcx,   %r11
	sti
    .byte   0x48
    sysexit


system_call_t system_call_table[MAX_SYSTEM_CALL_NR] = 
{
    [0] = sys_open,
    [1 ... MAX_SYSTEM_CALL_NR-1] = no_system_call
};


unsigned long sys_open(char *filename,int flags)
{
      ...
      ...
      return fd;
}
```

用户程序使用系统调用：

```
#define SYSFUNC_DEF(name)   _SYSFUNC_DEF_(name,__NR_##name)
#define _SYSFUNC_DEF_(name,nr)  __SYSFUNC_DEF__(name,nr)
#define __SYSFUNC_DEF__(name,nr)    \
__asm__ (       \
".global "#name"    \n\t"   \
".type  "#name",    @function \n\t" \
#name":     \n\t"   \
"pushq   %rbp   \n\t"   \
"movq    %rsp,  %rbp    \n\t"   \
"movq   $"#nr", %rax    \n\t"   \
"jmp    LABEL_SYSCALL   \n\t"   \
);

__asm__ (
"LABEL_SYSCALL: \n\t"
"pushq  %r10    \n\t"
"pushq  %r11    \n\t"
"leaq   sysexit_return_address(%rip),   %r10    \n\t"
"movq   %rsp,   %r11        \n\t"
"sysenter           \n\t"
"sysexit_return_address:    \n\t"
"xchgq  %rdx,   %r10    \n\t"
"xchgq  %rcx,   %r11    \n\t"
"popq   %r11    \n\t"
"popq   %r10    \n\t"
"cmpq   $-0x1000,   %rax    \n\t"
"jb LABEL_SYSCALL_RET   \n\t"
"movq   %rax,   errno(%rip) \n\t"
"orq    $-1,    %rax    \n\t"
"LABEL_SYSCALL_RET: \n\t"
"leaveq \n\t"
"retq   \n\t"
);

#define	__NR_open	2
SYSFUNC_DEF(open)

fd = open(path,0);
```

下面解析fd = open(path,0)完整流程

1. 根据cdecl规范，path存在RDI、0存在RSI，调用(call)open函数。
2. open函数把系统调用号0存于RAX，push R10、R11。把系统调用返回地址存于R10、返回的栈地址存于R11。
3. sysenter进入内核态。
4. 同一的系统调用入口函数保存通用寄存器、段寄存器于栈中。重载数据段寄存器为内核数据段。根据RAX中的系统调用号，调用(call)具体的处理函数，这里的处理函数为sys_open。
5. 用户态的参数一直保存在RDI和RSI寄存器中，sys_open能正常获得入参，最终把返回值存于RAX，返回到系统调用入口函数。
6. 系统调用入口函数把RAX的值覆盖保存在栈中的rax值。恢复保存在栈中的寄存器值。交换RDX和R10、RCX和R11。这样RDX、RCX就存入了系统调用返回地址和返回的栈地址。
7. sysexit返回到用户态。
8. 用户态再次交换RDX和R10、RCX和R11，恢复各自的值。然后pop R11、R10。返回到open函数。返回值已经存于RAX。



## Exceptional Control Flow

### 硬件中断

硬件中断和异常的机制是一样的。只是硬件中断使用的是中断门，而异常使用陷入门。这两者区别仅仅在与中断门会主动进行CLI(关闭硬件中断)。

关闭硬件中断会导致后面的已经中断不能得到及时处理。为此，硬件中断的中断处理程序会使用软中断(OS概念)、tasklet等机制缩短硬件中断关闭的时间。

这里对硬件中断就不再过多描述。



## VM Entry&VM Exit

TODO





