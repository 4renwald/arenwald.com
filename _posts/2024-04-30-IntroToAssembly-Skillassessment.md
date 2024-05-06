---
title: Intro to Assembly language - Skills Assessment
date: 2024-04-30 21:34:00 +/-TTTT
categories: [Writeups]
tags: [assembly, binary, shellcoding]
---
<style>
b {font-size:87.5%;color: #0398fc;word-wrap:break-word}
</style>

# Intro to Assembly language - Skills Assessment
This post is about the end of module assessment given for an introduction to assembly language course. Those are the solutions I came up with, don't mind sending me suggestions for improvement since i'm new to this. Also, this post is mostly for me and to keep track of my progress over time.

## Task 1
For the first task, we're given a binary called <b>loaded_shellcode</b>. We have to dissassemble it, modify the assembly code to decode the shellcode loaded in it, then execute it to get the flag. The decoding key is stored in <b>rbx</b> (Called Saved)

To dissassemble the <b>.text</b> section :  
  
`objdump -M intel --no-show-raw-insn --no-addresses -d loaded_shellcode`
  
```nasm
<_start>:
        movabs rax,0xa284ee5c7cde4bd7
        push   rax
        movabs rax,0x935add110510849a
        push   rax
        movabs rax,0x10b29a9dab697500
        push   rax
        movabs rax,0x200ce3eb0d96459a
        push   rax
        movabs rax,0xe64c30e305108462
        push   rax
        movabs rax,0x69cd355c7c3e0c51
        push   rax
        movabs rax,0x65659a2584a185d6
        push   rax
        movabs rax,0x69ff00506c6c5000
        push   rax
        movabs rax,0x3127e434aa505681
        push   rax
        movabs rax,0x6af2a5571e69ff48
        push   rax
        movabs rax,0x6d179aaff20709e6
        push   rax
        movabs rax,0x9ae3f152315bf1c9
        push   rax
        movabs rax,0x373ab4bb0900179a
        push   rax
        movabs rax,0x69751244059aa2a3
        push   rax
        movabs rbx,0x2144d2144d2144d2
```
  
The encoded shellcode is loaded by initializing the register <b>rax</b> with a value, then pushing it into the stack. This process is repeated <b>14</b> times. At the end, the decoding key is set into the Callee Saved register <b>rbx</b>.

The quickest/easiest approach would be to <b>pop</b> the values from the stack, <b>xor</b> them with the key in <b>rbx</b> and loop these steps 14 times. After that, load the program in a debugger and take note of the decoded value at each step. I wanted to make it just a little more fun by making a procedure that will do these steps, but will also print the entire decoded shellcode in my terminal, ready to be executed as is.
  
The approach I came up with is:
- <b>pop</b> the current stack pointer <b>rsp</b> into a register not used (<b>rdx</b> in my case)  
- <b>xor</b> it with the value in <b>rbx</b>
- print the value in <b>rdx</b> <b>libc</b> functions <b>printf</b> and <b>fflush</b>
- loop these steps 14 times

The format specifier used for <b>printf</b>:   
`outFormat db  "%016llx", 0x00`

- <b>0</b> : to pad the output with zeroes intead of spaces if minimum width is not met
- <b>16</b> : field width specifier of 16 characters, will be padded to the left with zeros.
- <b>ll</b> : length modifier long long int.
- <b>x</b> : lowercase hexadecimal integer
- <b>0x00</b> is the string terminator in <b>printf</b>

This is necessary because for example, one of the values is: 14831ff40b70148 instead of <b>0</b>14831ff40b70148 which would break the shellcode if I didn't pad the extra <b>0</b>.

Also, to be able to print all the values on the same line, I need to call <b>fflush</b> to flush all streams or else, I'd have to print on a new line instead.

Once this is all put together : 

```nasm
global _start
extern printf, fflush               ; Import external libc functions printf and fflush

section .data
    outFormat db  "%016llx", 0x00   ; Set the format specifier for printf

section .text
_start:
    mov rax,0xa284ee5c7cde4bd7
    push   rax
    mov rax,0x935add110510849a
    push   rax
    mov rax,0x10b29a9dab697500
    push   rax
    mov rax,0x200ce3eb0d96459a
    push   rax
    mov rax,0xe64c30e305108462
    push   rax
    mov rax,0x69cd355c7c3e0c51
    push   rax
    mov rax,0x65659a2584a185d6
    push   rax
    mov rax,0x69ff00506c6c5000
    push   rax
    mov rax,0x3127e434aa505681
    push   rax
    mov rax,0x6af2a5571e69ff48
    push   rax
    mov rax,0x6d179aaff20709e6
    push   rax
    mov rax,0x9ae3f152315bf1c9
    push   rax
    mov rax,0x373ab4bb0900179a
    push   rax
    mov rax,0x69751244059aa2a3
    push   rax
    mov rbx,0x2144d2144d2144d2

    mov rcx, 14                     ; Set the Loop Counter to 14

printDecoded:                        ; Start of new procedure printDecode

    pop rdx                         ; Pop the current stack pointer to rdx
    xor rdx, rbx                    ; Decode rdx using the key in rbx

    push rcx                        ; Push registers to stack before calling the printf function
    push rdx                        
    push rbx

    mov rdi, outFormat              ; Set the first printf argument (format specifier)
    mov rsi, rdx                    ; Set the second printf argument (value to print)
    call printf                     ; printf(outFormat, rdx)
    
    xor  rdi, rdi                   ; Setting rdi to zero
    call fflush                     ; Flush all streams

    pop rbx                         ; Restore registers from stack
    pop rdx
    pop rcx

    loop printDecode                ; Loop this procedure until rcx reaches 0

exit:                               ; Exit procedure
    xor rax, rax
    add al, 60
    xor dil, dil
    syscall
```

Assemble the code, do dynamic linking with <b>libc</b> and execute it using :   
`nasm -f elf64 flag.s &&  ld flag.o -o flag -lc --dynamic-linker /lib64/ld-linux-x86-64.so.2 && ./flag`

Result : 
```shell
nasm -f elf64 flag.s &&  ld flag.o -o flag -lc --dynamic-linker /lib64/ld-linux-x86-64.so.2 && ./flag
4831c05048bbe671167e66af44215348bba723467c7ab51b4c5348bbbf264d344bb677435348bb9a10633620e771125348bbd244214d14d244214831c980c1044889e748311f4883c708e2f74831c0b0014831ff40b7014831f64889e64831d2b21e0f054831c04883c03c4831ff0f05
```
{: .nolineno }

To execute the shellcode, I'll use the <b>pwntools</b> library in <b>python</b>:
```python
from pwn import *
context(os="linux", arch="amd64", log_level="error")
run_shellcode(unhex('SHELLCODE')).interactive()
```
{: .nolineno }
I can then execute the shellcode, which will print the flag :

```shell
python loader.py '4831c05048bbe671167e66af44215348bba723467c7ab51b4c5348bbbf264d344bb677435348bb9a10633620e771125348bbd244214d14d244214831c980c1044889e748311f4883c708e2f74831c0b0014831ff40b7014831f64889e64831d2b21e0f054831c04883c03c4831ff0f05'
HTB{4553mbly_d3bugg1ng_m4573r}$
```
{: .nolineno }
## Task 2
### Context

