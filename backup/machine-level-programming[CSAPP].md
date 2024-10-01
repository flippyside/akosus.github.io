# basic

## history of intel processors and architectures

Intel X86 Processors：

- dominate laptop/desktop/server market
- CISC(complex instruction set computer)
- evolution
  - 8086: 16-bit
  - 386(i386)(80386)(IA32) 32-bit
  - pentium4E(x86-64): 64-bit
  - core 2: multi-core
  - core i7: four cores

Architecture(ISA: instruction set architecture): the parts of a processor design that needs to understand or write assembly/machine code

machine code: byte-level programs, can be executed by processor
assembly code: a text representation of machine code

programmer-visible state:

- PC: program counter
  - address of next instruction
  - RIP in x86-64
- Register File: heavily used program data
- Condition Codes:
  - store status information about most
  - used for conditional branching
- Memory
  - byte addressable array
  - code and user data
  - stack to support procedures

```c
long plus(long x, long y);
void sumstore(long x, long y, long *dest){
    long t = plus(x, y);
    *dest = t;
}
```

```asm
sumstore:
  pushq %rbx
  movq %rdx, %rbx
  call plus
  movq %rax, (%rbx)
  popq %rbx
  ret
```

## assembly characteristics

Data type:

- integer data of 1,2,4, 8 byte
- floating point data of 4, 8, 10 byte
- code: byte sequences encoding series of instructions
- no aggregate types such as arrays or structures

Operation:

- perform arithmetic function on register or memory data
- transfer data between memory and register
- transfer control

```
# C code
*dest = t;

# Assembly
movq %rax, (%rbx) # move 8-byte value to memory
# operands:
#  t:     register %rax
#  dest:  register %rbx
#  *dest:  memory  M[%rbx]

# object code
0x40059e: 48 89 03
# 3-byte instuction
# stored at address 0x40059e
```

![Alt text](assets/ch3/image-1.png)

```
%r: 64bit
%e: 32bit
```

x8664将寄存器增加到%r，相比i386增加了一倍

![Alt text](assets/ch3/image-2.png)

寄存器的名字在很早之前反映了他们的特定功能，现在没有这种对应了。除了esp之外：%esp: 栈指针

# 访问信息

## MOV

```
movq Src Dest
```

DEST ← SRC;

- the SRC in movq only can be 32 bit. It sign-extend it to 64 bit, and put it into dest.
- `movabsq`(传送绝对的四字)'s SRC can be 64 bit.

operand types:

- immediate(立即数): constant integer data
  - e.g. $0x400 $-533
  - encoded with 1, 2 or 4 bytes
- register(寄存器): one of 16 integer register
  - e.g. %rax %r13
- memory(内存引用): 8 consecutive bytes of memory at address given by register
  - e.g. (%rax)

movq 的所有可能组合：不能mem-mem

![Alt text](assets/ch3/image-3.png)

simple memory addressing modes

- Normal: (R) `Mem[Reg[R]]`
  - register R specifies **memory address**
  - pointer dereferencing in C
- Displacement(偏移值): D(R) `Mem[Reg[R]+D]`
  - register R specifies the **start** of memory region
  - constant displacement D specifies offset

examples for normal:

```c
void swap(long *xp, long *yp){
    long t0 = *xp;
    long t1 = *yp;
    *xp = t1;
    *yp = t0;
}
```

```asm
swap:
    movq (%rdi), %rax
    movq (%rsi), %rdx
    movq %rdx, (%rdi)
    movq %rax, (%rsi)
    ret
```

rdi：xp，即指向存储【指针xp指向的值】的内存地址
rsi：yp，即指向存储【指针yp指向的值】的内存地址
rax：t0
rdx：t1

![Alt text](assets/lec5/image.png)

![Alt text](assets/lec5/image-1.png)
![Alt text](assets/lec5/image-2.png)
![Alt text](assets/lec5/image-3.png)

MOVZ & MOVS:

- movz zero-extend DEST
- movs sign-extend DEST

```
movzbw SRC, DEST
```

### address computation

存储器操作数形式：

`偏移量(基址寄存器，变址寄存器，比例因子)`

`D(Rb, Ri, S)` -> `Mem[Reg[Rb] + S * Reg[Ri] + D]`
D: constant displacement, 1,2,4 bytes
Rb: base register, any of 16 integer registers
Ri: index register, any, expect %rsp
S: scale, 1,2,4,8

```
100   (%ebx,      %esi,     4)
偏移量 基址寄存器，变址寄存器，比例因子
```

address computation examples:

```
%rdx 0xf000
%rcx 0x0100

0x8(%rdx)   0xf000 + 0x8 = 0xf008
(%rdx, %rcx)   0xf000 + 0x100 = 0xf100
(%rdx, %rcx, 4)   0xf000 + 4 * 0x100 = 0xf400
0x80(, %rdx, 2)   2 * 0xf000 + 0x80 = 0x1e080
```

### pop&push

the address of the top of the stack is the **LOWest**. the stack pointer **%rsp** stores the address of the top.

```
pushq S
R[%rsp] <- R[%rsp] - 8
M[R[%rsp]] <- S

popq D
D <- M[R[%rsp]]
R[%rsp] <- R[%rsp] + 8
```

popq %rax is equal to:

```
movq (%rsp), %rax
addq $8, %rsp
```

# arithmetic & logical operations

there is one-operand and two-operand instr.

the 2-operand is just like `+=` `-=` in C.

## leaq(load effective address)

```
leaq src, dst
```

- src is address mode expression, set dst to address denoted by expression.
- that is, compute the address(x + k * y) and load it to the register

e.g.

if rdi is x:

```asm
leaq 7(%rdi, %rdi, 4), %rax    # set the %rax to 7 + x + x * 4 
```

```c
long mul12(long x){ 
    return x * 12; 
}

leaq (%rdi, %rdi, 2), %rax   # t <- x + 2x
salq $2, %rax  # shift to left, 2bit
```

# control

rip: instruction pointer. in i386, it's eip.

## processor state

information about currently executing program:

- temporary data
- location of runtime stack
- location of current code control point
- status of recent tests (CF, SF...)

> status register, flag register, or condition code register (CCR) is a collection of status flag bits for a processor.

### condition code

CF is for unsigned number, OF is for signed number

positive overflow: opsitive + opsitive = negative | negative + negative = opsitive

```
cmpq src2 src1
```

computing src1 - src2 without setting destination. Just set condition code.

```
testq src2, src1
```

- computing src1 & src2 without setting destination. Just set condition code.
- useful to have one of the operands be a mask

#### read condition code:

SET instructions:

- set low-order byte of destination to 0 or 1 based on combinations of condition codes
- does not alter remaining 7 bytes

![Alt text](assets/machine-level-programming/image.png)

e.g.

```
int gt(long x, long y){
  return x > y;
}

asm:

cmpq %rsi, %rdi  # compare x, y
setg %al         # set when >
movzbl %al, %eax # zero rest of %rax
ret
```

movzbl: move with zero extension byte to long(零扩展至32bit)
由于x86-64中所有改动了32bit的指令都会将高32位清零，所以不仅仅是eax的高3个字节清零，rax的高7个字节都全部变0。

## Jump

> if the condition is satisfied, jump to a destination with label.

```c
long absdiff(long x, long y){
    long result;
    if(x > y) result = x - y;
    else result = y - x;
    return result;
}

asm:

    cmpq  %rsi, %rdi
    jle   .L4
    movq  %rdi, %rax
    subq  %rsi, %rax
    ret
.L4:
    movq  %rsi, %rax
    subq  %rdi, %rax
    ret
```

we can express it with **goto** code: jump to positon designated by label

```c
long absdiff(long x, long y){
    long result;
    int ntest = x <= y;
    if(ntest) goto Else;
    result = y - x;
    goto Done;
Else:
    result = x - y
Done:
    return result;
}
```

jump的编码: PC相对寻址(PC-relative)

- 将 目标指令的地址 和 紧跟在跳转指令后面的指令的地址 之间的差作为编码。也就是说，jump的编码(03) + 紧跟在jump后面的指令的地址(5) = 目标的地址(8)
- 执行PC-relative时，程序计数器的值时jump指令后面的那条指令的地址

```
3: eb 03      jmp 8 <lopp+0x8>
5: 48 d1 f8   sar %rax
8: 48 85 c0   test %rax, %rax
```

### Conditional Move cmov(数据的条件转移)

if computations are easy to do, do both and pick one according to the condition. do not use jump:

```
    movq  %rdi, %rax
    subq  %rsi, %rax # x-y
    movq  %rsi, %rdx
    subq  %rdi, %rdx # y-x
    cmpq  %rsi, %rdi # x:y
    cmovle %rdx, %rax # if x<=y
```

```
v = test ? then : else
```

用条件控制转移的标准方法:

```
    if(!test)
        goto false
    v = then
    goto done
false:
    v = else  
done
```

用条件传送的方法：then 和 else都会求值

```
v = then
ve = else
t = test
if(!t) v = ve
```

不适用条件转移的情形：

- 有副作用（全局变量）
- 可能有非法操作（解引用空指针）
- 需要大量计算

## Loop

### do-while

count number of 1 in x(popcount)

```c
long pcount(unsigned long x){
  long result = 0;
  do{
    result += x & 0x1;
    x >>= 1;
  }while(x);
  return result;
}
```

goto:

```c
long pcount(unsigned long x){
  long result = 0;
  loop:
    result += x & 0x1;
    x >>= 1;
    if(x) goto loop;
  return result;
}
```

asm: "jump to middle"

```
  movl $0, %eax  # result = 0;
.L2:             # loop:
  movq %rdi, %rdx
  andl $1, %edx  # t = x & 1
  addq %rdx, %rax # result += t
  shrq %rdi      # x >>= 1
  jne .L2        # if(x) goto loop
  rep; ret
```

### while

```c
long pcount(unsigned long x){
  long result = 0;
  while(x){
    result += x & 0x1;
    x >>= 1;
  }
  return result;
}
```

goto:

```c
long pcount(unsigned long x){
  long result = 0;
  goto test;  <----
  loop:
    result += x & 0x1;
    x >>= 1;
  test:       <----
    if(x) goto loop;
  return result;
}
```

another version: use initial-test + do-while:

```c
long pcount(unsigned long x){
  long result = 0;
  if(!x) goto done;  <----
  loop:
    result += x & 0x1;
    x >>= 1;
    if(x) goto loop;
  done:              <----
    return result;
}
```

### for

```
for(init; test; update)
    body
          |
          |
         \ /
          '
init;
while(test){
    body
    update
}
```

In -O1, Initial test can be optimized away:
![Alt text](assets/machine-level-programming/image-1.png)

## switch

jump table

![Alt text](assets/machine-level-programming/image-2.png)
![Alt text](assets/machine-level-programming/image-4.png)

```c
long switch(long x, long y, long z){
  long w = 1;
  switch(x){  // 1~6, no 4
    case 1: break;
    case 2: w = y/z;
    case 3: w += z; break;
    case 5: 
    case 6: break;
    default;
  }
  return w;
}
```

![Alt text](assets/machine-level-programming/image-3.png)

- case 4 is treated as default case(.L8).
- case 5 6 is using the same label .L7, because there is no break in case 5.

asm

```asm
switch:
  movq %rdx, %rcx
  cmpq $6, %rdi         # x:6
  ja   .L8              # default
  jmp  *.L4(, %rdi, 8)  # goto *JTab[x]
```

`ja`: jump above. the number is treated as **unsigned**.

- if x > 6, it will jump;
- if x < 0, the unsigned x will be a large positive number larger than 6, it will jump too.

n

```
.L5:
  movq %rsi, %rax
  cqto
  idivq %rcx      # y/z
  jmp .L6         # goto merge
.L9:
  movl $1, %eax   # w = 1
.L6:
  addq %rcx, %rax # w += z
  ret
```

# procedure

## calling conventions

### passing control

ABI(application binary interface)应用程序二进制接口

![Alt text](assets/machine-level-programming/image-5.png)

stack:

- the top is in the bottom.
- push, `stack pointer--`

![Alt text](assets/machine-level-programming/image-6.png)

`pushq SRC`:

- fetch operand at SRC
- %rsp -= 8
- write operand at address given by %rsp

`popq DEST`:

- read value at address given by %rsp
- %rsp += 8
- store value at DEST(must be a register)

Procedure control flow: use **stack** to support procedure call and return

- procedure call: call label
  - **push** return address into stack
  - jump to label
- return address:
  - address of the **next** instruction after call
- procedure return: ret
  - **pop** address from stack
  - jump to address

`callq 400550 <mul2>`: push the return address into stack and set the PC to the new value.

`retq`: pop the address off the stack(rsp++), and set the PC to that address.

### passing data

argument type: integer, pointer

![Alt text](assets/machine-level-programming/image-7.png)

stack frame:

contents:

- return information
- local storage
- temporary space

management:

- space **allocated** when enter procedure
  - set-up code
  - includes push by `call` instruction
- **deallocated** when return
  - finish code
  - includes pop by `ret` instruction

![Alt text](assets/machine-level-programming/image-8.png)

- Current stack frame
  - argument build: parameters for function about to call
  - local variables
  - saved register content
  - old frame pointer(optional)
- Caller Stack Frame
  - return address(pushed by `call`)
  - arguments for this call

```c
%rdi: argument p
%rsi: argument val, y
%rax: x, return value

long incr(long *p, long val){
  long x = *p;
  long y = x + val;
  return x;
}

incr:
  movq  (%rdi)  %rax
  addq  %rax,  %rsi
  movq  %rsi,  (%rdi)
  ret
```

```
long call_incr(){
  long v1 = 15213;
  long v2 = incr(&v1, 3000);
  return v1 + v2;
}

call_incr:
  subq  $16, %rsp       # allocate
  movq  $15213, 8(%rsp) # push in stack
  movl  $3000, %esi
  leaq  8(%rsp), %rdi   # 把15213的地址传入rdi
  call  incr            # 调用函数
  addq  8(%rsp), %rax
  addq  $16, %rsp       # deallocate
  ret
```

register saving conventions

- caller saved: caller saves temporary values in its frame before the call
- callee saved: callee saves temporary values in its frame before using, and restores them before returning to caller

## recursion

```c
/* recursive popcount */
long pcount_r(unsigned long x){
  if(x == 0) return 0;
  else
    return (x & 1) + pcount_r(x >> 1);
}

rdi: x
rax: return value

pcount_r:
  movl    $0, %eax   # assume x is 0 and set the return value(eax) 0
  testq   %rdi, %rdi # x:0
  je      .L6        # jump if equal
  pushq   %rbx       # 暂存rbx内的值以便之后复原
  movq    %rdi, %rbx # 把x放到rbx
  andl    $1, %ebx   # x & 1
  shrq    %rdi       # x >> 1
  call    pcount_r
  addq    %rbx, %rax # rax: 递归结果
  popq    %rbx       # 复原rbx之前的值
.L6:
  rep; ret
```

- stack frame: each function call has private storage
  - saved registers & local variables
  - saved return pointer
- register saving conventions prevent one function call from corrupting other's data
- stack discipline follows call/return pattern
  - if P calls Q, then Q returns before P


# Data

## array

### array accessing

e.g.
```c
typedef int zip_dig[5]
int get_digit(zip_dig z, int digit){
  return z[digit];
}
```

```bash
%rdi = z                  # rdi: starting address (z)
%rsi = digit              # rsi: array index (digit)
movl(%rdi, %rsi, 4), %eax # eax: z[digit], digit = z + 4*digit
```

### array loop

```c
void incr(zip_dig z){
  size_t i;
  for(int i = 0; i < LEN; i++){
    z[i]++;
  }
}
```

```bash
  movl  $0, %eax  # i = 0
  jmp   .L3       # goto middle
.L4:              # loop
  addl  $1, (%rdi, %rax, 4)   # z[i]++
  addq  $1, %rax  # i++
.L3:              # middle
  cmpq  $4, %rax  # i:4
  jbe   .L4       # if <=, goto loop
  ret
```


## multidimensional(nested) arrays

arrangement: row-major ordering

`int A[R][C]`;
```
 A  ... A    A  ... A  ...  A  ... A
[0]    [0]  [1]    [1]    [R-1]  [R-1]
[0]   [C-1] [0]   [C-1]    [0]   [C-1]
```

every column's starting address: `A + i * C * ele_size`

![alt text](assets/machine-level-programming/image-9.png)

### multidimensional array accessing

1. get the ith column

```c
#define PCOUNT 4
zip_dig pgh[PCOUNT] =
 {{1, 5, 2, 0, 6},
 {1, 5, 2, 1, 3 },
 {1, 5, 2, 1, 7 },
 {1, 5, 2, 2, 1 }}; 

int *get_pgh_zip(int index)
{
 return pgh[index];
}
```

```c
# %rdi = index
leaq (%rdi,%rdi,4), %rax      # 5 * index
leaq pgh(,%rax,4),%rax        # pgh + (20 * index)
```

2. 

![alt text](assets/machine-level-programming/image-10.png)

```c
int get_pgh_digit (int index, int dig)
{
 return pgh[index][dig];
} 
```

```c
leaq (%rdi,%rdi,4), %rax      # 5*index
addl %rax, %rsi               # 5*index+dig
movl pgh(,%rsi,4), %eax       # M[pgh + 4*(5*index+dig)]
```

### multi-level array
![alt text](assets/machine-level-programming/image-11.png)
```c
zip_dig cmu = { 1, 5, 2, 1, 3 };
zip_dig mit = { 0, 2, 1, 3, 9 };
zip_dig ucb = { 9, 4, 7, 2, 0 };
#define UCOUNT 3
int *univ[UCOUNT] = {mit, cmu, ucb}; 

int get_univ_digit
 (size_t index, size_t digit)
{
 return univ[index][digit];
} 
```

```c
salq   $2, %rsi             # 4*digit
addq   univ(,%rdi,8), %rsi  # p = univ[index] + 4*digit
movl   (%rsi), %eax         # return *p
ret
```

Element	access: `Mem[Mem[univ+8*index]+4*digit] `
do 2 memory	reads	
- First	get	pointer	to row array	
- Then access element	within array	

- multi-level array has two memory access
- nested array has one memory access


### nxn matrix access

because n is a unknown number, so imulq instr is used.

```c
int var_ele(size_t n, int a[n][n], size_t i, size_t j)
{
 return a[i][j];
}

# n in %rdi, a in %rsi, i in %rdx, j in %rcx
imulq %rdx, %rdi            # n*i
leaq (%rsi,%rdi,4), %rax    # a + 4*n*i
movl (%rax,%rcx,4), %eax    # a + 4*n*i + 4*j
ret   
```



## structures
![alt text](assets/machine-level-programming/image-12.png)
- a block of memory for every element
- memory ordered according to declaration
- Compiler determines overall	size +	positions	of fields

linked list:

```c
struct rec {
 int a[4];
 int i;
 struct rec *next;
}; 
void set_val(struct rec *r, int val)
{
  while (r) {
    int i = r->i;
    r->a[i] = val;
    r = r->next;
  }
} 
```

```
rdi: r
rsi: val
.L11:                       # loop:
  movslq 16(%rdi), %rax     # i = M[r+16]
  movl %esi, (%rdi,%rax,4)  # M[r+4*i] = val
  movq 24(%rdi), %rdi       # r = M[r+24]
  testq %rdi, %rdi          # Test r
  jne .L11                  # if !=0  goto loop
```

Alignment of struct:
- the largest data type requires K bytes, then address must be multiple of K
- 

![alt text](assets/machine-level-programming/image-13.png)

save spaces in alignment: put large data types first
![alt text](assets/machine-level-programming/image-14.png)



## pointer

the pointer just be allocated the memory of **this pointer(8 bytes)**, not its type that is point to.
```c
                size of it
int A1[3]           12
int *A2[3]          8
```

Cmp: compiles Y/N
Bad: possible bad pointer reference Y/N
Size: value returned by sizeof

`int A1[3]`: 数组，元素为int类型
`int *A2[3]`: 数组，元素为`int*`类型（指向int类型的指针）
`int (*A3)[3]`: 指针，指向数组
`int (*A4[3])`: 数组，元素为`int*`类型（指向int类型的指针）。和A2一模一样

```c
int A1[3] = {1, 2, 3};

int a, b, c;
int *A2[3] = {&a, &b, &c};

int arr[3] = {1, 2, 3};
int (*A3)[3] = &arr;  // A3 是指向 arr 数组的指针

int a = 1, b = 2, c = 3;
int *A4[3] = {&a, &b, &c};  // 和 A2 的效果类似
```


![alt text](assets/machine-level-programming/image-16.png)

`int A1[3][5]`: 二维数组，元素是int类型
`int *A2[3][5]`: 二维数组，元素为`int*`类型（指向int类型的指针）
`int (*A3)[3][5]`: 指针，指向 3x5 大小二维数组。
`int *(A4[3][5])`: 与 A2 等价
`int (*A5[3])[5]`: 数组，元素是指向int类型数组的指针。

```c
int A1[3][5] = {
    {1, 2, 3, 4, 5},
    {6, 7, 8, 9, 10},
    {11, 12, 13, 14, 15}
};

int a = 1, b = 2, c = 3;
int *A2[3][5] = {
    {&a, &b, &c, nullptr, nullptr},
    {nullptr, nullptr, nullptr, nullptr, nullptr},
    {nullptr, nullptr, nullptr, nullptr, nullptr}
};

int (*A3)[3][5] = &A1; 

int a = 1;
int *(A4[3][5]) = {
    {&a, nullptr, nullptr, nullptr, nullptr},
    {nullptr, nullptr, nullptr, nullptr, nullptr},
    {nullptr, nullptr, nullptr, nullptr, nullptr}
};

int arr1[5] = {1, 2, 3, 4, 5};
int arr2[5] = {6, 7, 8, 9, 10};
int arr3[5] = {11, 12, 13, 14, 15};

int (*A5[3])[5] = {&arr1, &arr2, &arr3}; 
```

![alt text](assets/machine-level-programming/image-17.png)
![alt text](assets/machine-level-programming/image-18.png)
