# Tutorial for Converting Venus RISC-V Assembly to Real Chip Kendryte 210 (K210) RISC-V Assembly
> [Venus RISC-V Assembly](https://github.com/kvakil/venus) is the assembly that can run on Venus RISC-V simulator.  

## Objective for the Tutorial
This tutorial is designed to help you convert Venus RISC-V Assembly to real chip Kendryte 210 (K210) RISC-V Assembly. Finally, you should be able to run RISC-V Assembly that you modified from Venus on a real chip K210.  
However, if you are interested in writing a compiler with RISC-V Assembly as the target language or writing RISC-V Assembly by hand, you may also glance over this tutorial.  

## Advices
Please ensure you are `comfortable` with (in descending order of importance) Linux shell, RISC-V Assembly, GDB, GCC, GNU Toolchain, C/C++ Memory Allocation, a little bit of Operating System, CMake.  
Since Venus is usually used for education purpose, this tutorial will assume that you have undergraduate level knowledge of Computer Architecture and Organization & Principles of Compiler.  
> Notice that this tutorial is more like experience than a detailed step-by-step guide, so you need to do a lot of debugging and development work yourself, and basic knowledge and patience are very important to success.  
  
## Hardware Preparation  
When I first decided to port Venus assembly to real chip's assembly, I first investigated the available RISC-V development boards. Then, I decided to choose the development board that has the potential to run Linux (now (2020.12), the performance of RISC-V chip on most development boards is too low to run Linux), but the price is not too high. Thus, `Maix Bit` was chosen finally.   
If you are willing to use the same development board as me, please read and complete [this tutorial](https://github.com/qingpeng9802/build-maix-bit-k210-bare-metal-debug-dev-env) to build a bare metal debugging and development environment. Notice that you need a `debugger` to complete the tutorial. The cost of the development board and debugger is about $20.  
  
> You may notice that Maix Bit is 64-bit architecture, while Venus simulates 32-bit instructions, which means that we need some extra work to deal with the OFFSET problem, that is, 4->8.  
(In fact, the cost-performance ratio of the 32-bit development board is not as good as K210 so I still choose K210 here.)

TL;DR:  
1. Sipeed Maix Bit is about $12.9
2. Sipeed RV Debugger is about $7.6  
   & complete [this tutorial](https://github.com/qingpeng9802/build-maix-bit-k210-bare-metal-debug-dev-env)   

## Workflow  
```
[OpenOCD is connecting with debugger]------------------------|
                                                             |---|
[minicom/serial communication is connecting with debugger]---|   |
                printf()/scanf()                                 |
                                                                 |
              modify                                             |
(Kendryte Standalone SDK Project Test:                           |
 RISC-V Assembly file: Test/test.s)                              |
                |                                                |
                | cmake PROJ=Test                                |
                V                                                |
1. (test) for debugging on chip & 2. (test.bin)                  |
                |                    for flashing to chip,       |
                |                    NOT used here               |
                |-------------------------------------------->>[GDB] 
                    as file input
```
0. Keep OpenOCD and minicom alive.  
1. Any RISC-V code changes in `Test/test.s` in SDK.  
2. Then, `cmake` again once you changed `Test/test.s`.  
3. The build result `test` is used by GDB.  
4. Check the output and change the RISC-V code by Step 1.  

## Tips before Starting Porting
If you have any questions about the shape of RISC-V assembly to be generated, the best reference is the code in `k210asms` of [this project](https://github.com/qingpeng9802/minijava-to-k210-riscv-compiler).  
  
> Good start point of GNU Assembler: [GNU Assembler Examples](https://cs.lmu.edu/~ray/notes/gasexamples/) (although it is x86-64 asm) & [The GNU Assembler](https://sourceware.org/binutils/docs/as/)  
  
### GDB Tips
Ensure you read [GDB Cheat Sheet](https://gist.github.com/rkubik/b96c23bd8ed58333de37f2b8cd052c30) first.  
#### Common Workflow:  
If you doubt the value of the registers is correct, you can run your Venus assembly on 
https://chocopy.org/venus.html and compare the register values in Venus with the register values in GDB.  
`i r (info registers)`  
  
If you need `pc`, `current line`, `sp`, by https://sourceware.org/gdb/current/onlinedocs/gdb/Registers.html  
`p/x $pc`  
`x/i $pc`  
`set $sp += 4`  
  
If you need to print frame,  
`info frame`  
print stack,  
`backtrace`  

If you need to print heap, by https://ftp.gnu.org/old-gnu/Manuals/gdb-5.1.1/html_chapter/gdb_9.html#SEC56  
`x/xh addr` print hex with length double(64-bit)  
`x/uh addr` print unsigned decimal with length double(64-bit)  
`x/dh addr` print signed decimal with length double(64-bit)  
  
It may be a bit annoying to find the line causing crash. By my experience, you need to step into manually, otherwise the program will fall into the `trap_entry()` in `bsp` of K210 after crush.  
However, I notice that the IDE [PlatformIO](https://docs.platformio.org/en/latest/boards/kendryte210/sipeed-maix-bit.html) for embedded systems is very complete and mature. Maybe `PlatformIO`'s debugging and development experience is great and worth a try. Although I did not try it, I hope to see a `PlatformIO` experience for this porting project :)  

Or consider using [Kendryte IDE](https://github.com/kendryte/kendryte-ide) (not sure about the quality of the beta version).  

## Port Venus RISC-V Assembly to Kendryte 210 (K210) RISC-V Assembly
Here, we will focus on how to run let RISC-V code in `Test/test.s` can run on K210.  
Always check [RISC-V Reference Card](https://www.cl.cam.ac.uk/teaching/1617/ECAD+Arch/files/docs/RISCVGreenCardv8-20151013.pdf)  

### 32-bit->64-bit | OFFSET: 4->8
First, we will deal with the conversion issue of 32->64.  
I will assume that you are in a compiler project, guiding you how to modify the program for outputting RISC-V code. In fact, my compiler project is based on UCLA CS132 RISC-V version.      

1. If you hardcoded the `OFFSET` of `fp`/`sp` to `4` in the memory allocation part (object, array) of your program. Add a `const int OFFSET=8`, and change all these `4` to `OFFSET`.  
If you have used `OFFSET` of `fp`/`sp` in your program, just change it to `8`.  
    > Be careful with the locations and interface functions for calculating offset.  
2. In most cases, you have some hardcoded offset in strings. For example: `-4(fp)` or `8(sp)`, and they should be double to `-8(fp)` or `16(sp)`.  
3. If you have a program to generate `IR` before current program in your compiler project.  
The memory allocation part (object, array) of your IR generating program should also have a offset of `4`. The offsets in your IR generating program should be modified to `8` like in Step 1.  
4. [RISC-V allows mixing different length instrucions](https://stackoverflow.com/questions/56874101/how-does-risc-v-variable-length-of-instruction-work-in-detail). That is, we can both use `lw t0, 8(fp)` and `ld t0, 8(fp)` in 64-bit.  
Then, it may cause
    ```
    ----------16(fp)
    ---------- 8(fp) value: 0x0000000000800168

    t0: 0xffffffffffffffff
    ==========================================
    After lw t0, 8(fp),
    t0: 0xffffffff00800168

    After ld t0, 8(fp),
    t0: 0x0000000000800168
    ```
    (This can be solved by add `li t0, 0` before `lw t0, 8(fp)`, but elegance is lost)  
    Although this feature is a way to save space, for the simplicity of our implement, we will NOT use `lw` and `sw` here.  
    All `lw` and `sw` should be replaced by `ld` and `sd`. If you use `exact match` first and then use `global replacement`, there should be no problem.
5. If you have some strings with `.asciz`, `string`, etc., change its correspoding `.align 2` to `.align 3`.  

If you see `core dump: misaligned load`, check this part again.  
  
### Assembly Support Difference between Venus and K210  
See [RISC-V Assembly Programmer's Manual](https://github.com/riscv/riscv-asm-manual/blob/master/riscv-asm.md)  
Notice that many directives in Venus are not supported officially and many official directives are not supported by K210.  

1. `.equiv` and `.equ` are not supported by K210, just replace all constant definitions. Then, delete them all.  
2. For the simplicity of our implement, `errors/exceptions`/`exit`, will not be implemented. Just delete all related part.  
3. At the beginning of program, if you have something like:
    ```
     .text
    -  jal Main
    -  li a0, @exit
    -  ecall
    ```
    just change to 
    ```
    .text
    ```
4. Make sure the main function call `main`, not `Main`. Also, add a infinite loop before `main` function's last line `jr ra`. This can prevent `pc` from falling out of the boundary of the program, which is convenient for debugging and burning (press `RESET` button on Maix Bit can rerun the program easily). The end of `main`:  
    ```asm
      something
    + .inf_loop:
    +   j .inf_loop
        jr ra
    ```
5. At the beginning of every functions (include `main`), it should be
    ```
    .align 1
    .globl your_function_name
      your_function_name:
    ```
  
### Add Util Function: `.print_int` & `.alloc`  
Delete old Util Functions.  
Do not use `ecall` for K210.  
Add this part to the end of your program:  
Note the parameter register and the return register in comment.  
  
> Notice that the function's prefix is `.`, this is by the convention of the product of `riscv64-unknown-elf-gcc -S`.  
  
```asm
.align 1
.globl .print_int
# need save a0, a1 before call
# a1: num -> void
.print_int:
  sd fp, -16(sp)          # Store old fp
  mv fp, sp               # Set new fp
  addi sp, sp, -16
  sd ra, -8(fp)
  la a0, .rep_int
  call printf
  ld t6, _impure_ptr
  ld t6, 16(a7)
  mv a0, t6
  call fflush
  ld ra, -8(fp)           # Restore ra register
  ld fp, -16(fp)          # Restore old fp
  addi sp, sp, 16
  jr ra

.align 1
.globl .alloc
# need save a0, a1 before call
# a0: num -> a0: pointer
.alloc:
  sd fp, -16(sp)          # Store old fp
  mv fp, sp               # Set new fp
  addi sp, sp, -16
  sd ra, -8(fp)
  # int size is 4, but we are in 64-bit, use 8
  li a1, 8
  call calloc
  ld ra, -8(fp)           # Restore ra register
  ld fp, -16(fp)          # Restore old fp
  addi sp, sp, 16
  jr ra

.section	.rodata
.rep_int:
  .string "%d\n"
  .align 3

```
  
#### How to Call `.print_int`:  
Since the function `call`ed in the Util Function may destroy all `caller-save registers`, we need to `save and restore` all `a reg` and `t reg`.  
!!! Notice that I did not use `a0, a1, t6` in register allocation so I did not save them.  
!!! You should save and restore all caller-save registers you used.  
This is just an example:  
```asm
  addi sp, sp, -96
  sd a2, 0(sp)
  sd a3, 8(sp)
  sd a4, 16(sp)
  sd a5, 24(sp)
  sd a6, 32(sp)
  sd a7, 40(sp)
  sd t0, 48(sp)
  sd t1, 56(sp)
  sd t2, 64(sp)
  sd t3, 72(sp)
  sd t4, 80(sp)
  sd t5, 88(sp)
  jal .print_int
  ld a2, 0(sp)
  ld a3, 8(sp)
  ld a4, 16(sp)
  ld a5, 24(sp)
  ld a6, 32(sp)
  ld a7, 40(sp)
  ld t0, 48(sp)
  ld t1, 56(sp)
  ld t2, 64(sp)
  ld t3, 72(sp)
  ld t4, 80(sp)
  ld t5, 88(sp)
  addi sp, sp, 96
```
  
#### How to Call `.alloc`:  
Since the function `call`ed in the Util Function may destroy all `caller-save registers`, we need to `save and restore` all `a reg` and `t reg`.  
!!! Notice that I did not use `a0, a1, t6` in register allocation so I did not save them.  
!!! You should save and restore all caller-save registers you used.  
  
> For Java Compiler, in [Java spec](https://docs.oracle.com/javase/specs/jls/se7/html/jls-4.html#jls-4.12.5), the default values of `new int[]` is `0`. Thus, `calloc()` is used here instead of `malloc()`.  
  
> Noticed that there is NOT pointer free step, if you would like to free the pointer, you need to create a Util Function by yourself.  
This is just an example:  
```asm
  addi sp, sp, -96
  sd a2, 0(sp)
  sd a3, 8(sp)
  sd a4, 16(sp)
  sd a5, 24(sp)
  sd a6, 32(sp)
  sd a7, 40(sp)
  sd t0, 48(sp)
  sd t1, 56(sp)
  sd t2, 64(sp)
  sd t3, 72(sp)
  sd t4, 80(sp)
  sd t5, 88(sp)
  jal .alloc
  ld a2, 0(sp)
  ld a3, 8(sp)
  ld a4, 16(sp)
  ld a5, 24(sp)
  ld a6, 32(sp)
  ld a7, 40(sp)
  ld t0, 48(sp)
  ld t1, 56(sp)
  ld t2, 64(sp)
  ld t3, 72(sp)
  ld t4, 80(sp)
  ld t5, 88(sp)
  addi sp, sp, 96
```
  
### Design your Util Function  
1. Write a minimal program similar to `Hello_World` in C.  
Then, add the C function to your program you would like to run in RISC-V.  
Do not write your program in C++ or the compiled product can become very complicated and difficult to understand.  
2. Use `riscv64-unknown-elf-gcc -S` to compile your C program.  
Then, you will get the product of compilation, that is, RISC-V assembly.  
3. By looking for the relationship between the source code in C and RISC-V assembly, we can infer the assembly fragment we need.  
Note the parameter register, the return register, and `lw` `ld` `sw` `sd`.  
  
## About  
This project's idea is mainly derived from the RISC-V version of UCLA CS132 Compiler Construction. The target language of the compiler project is Venus RISC-V assembly. This project can verify that the RISC-V assembly output by the modified compiler can run on the RISC-V chip K210.  
  
## License  
Copyright (C) 2020 Qingpeng Li  
This work is licensed under a [Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0) License](https://creativecommons.org/licenses/by-nc-nd/4.0/).  
