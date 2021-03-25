# Lab4 - traps

## RISC-V assembly (easy)

`git checkout traps`

`make clean`

To answer the following question, first do `make fs.img`.

### 1. Which registers contain arguments to functions? For example, which register holds 13 in main's call to printf?

```asm6502
000000000000001c <main>:

void main(void) {
  1c:	1141                	addi	sp,sp,-16
  1e:	e406                	sd	ra,8(sp)
  20:	e022                	sd	s0,0(sp)
  22:	0800                	addi	s0,sp,16
  printf("%d %d\n", f(8)+1, 13);
  24:	4635                	li	a2,13
  26:	45b1                	li	a1,12
```

> since 12 = f(8)+1 = 8+3+1, it may be optimized by compiler and a2 holds 13.
> 
> a1 and a2 holds the arguments to printf(), and a0 holds the argument to g(x).

```asm6502
int g(int x) {
   0:	1141                	addi	sp,sp,-16
   2:	e422                	sd	s0,8(sp)
   4:	0800                	addi	s0,sp,16
  return x+3;
}
   6:	250d                	addiw	a0,a0,3
   8:	6422                	ld	s0,8(sp)
   a:	0141                	addi	sp,sp,16
   c:	8082                	ret

000000000000000e <f>:

int f(int x) {
   e:	1141                	addi	sp,sp,-16
  10:	e422                	sd	s0,8(sp)
  12:	0800                	addi	s0,sp,16
  return g(x);
}
  14:	250d                	addiw	a0,a0,3
  16:	6422                	ld	s0,8(sp)
  18:	0141                	addi	sp,sp,16
  1a:	8082                	ret

```

#### 2. Where is the call to function f in the assembly code for main? Where is the call to g? (Hint: the compiler may inline functions.)

```asm6502
int g(int x) {
   0:	1141                	addi	sp,sp,-16
   2:	e422                	sd	s0,8(sp)
   4:	0800                	addi	s0,sp,16
  return x+3;
}
   6:	250d                	addiw	a0,a0,3
   8:	6422                	ld	s0,8(sp)
   a:	0141                	addi	sp,sp,16
   c:	8082                	ret

000000000000000e <f>:

int f(int x) {
   e:	1141                	addi	sp,sp,-16
  10:	e422                	sd	s0,8(sp)
  12:	0800                	addi	s0,sp,16
  return g(x);
}
  14:	250d                	addiw	a0,a0,3
  16:	6422                	ld	s0,8(sp)
  18:	0141                	addi	sp,sp,16
  1a:	8082                	ret

000000000000001c <main>:

void main(void) {
  1c:	1141                	addi	sp,sp,-16
  1e:	e406                	sd	ra,8(sp)
  20:	e022                	sd	s0,0(sp)
  22:	0800                	addi	s0,sp,16
  printf("%d %d\n", f(8)+1, 13);
  24:	4635                	li	a2,13
  26:	45b1                	li	a1,12
  28:	00000517          	auipc	a0,0x0
  2c:	7b050513          	addi	a0,a0,1968 # 7d8 <malloc+0xea>
  30:	00000097          	auipc	ra,0x0
  34:	600080e7          	jalr	1536(ra) # 630 <printf>
  exit(0);
  38:	4501                	li	a0,0
  3a:	00000097          	auipc	ra,0x0
  3e:	27e080e7          	jalr	638(ra) # 2b8 <exit>
```

> As we can see, there's no corresponding invoking for f and g in main(). Maybe it's already optimzed by compiler to give a single 12 to a1.

#### 3. At what address is the function printf located?

> By searching we can find:

```asm6502
0000000000000630 <printf>:

void
printf(const char *fmt, ...)
{
 630:	711d                	addi	sp,sp,-96
 632:	ec06                	sd	ra,24(sp)
```

> So it's located at 0x630.

#### 4. What value is in the register ra just after the jalr to printf in main?

> AUIPC (add upper immediate to pc) is used to build pc-relative addresses and uses the U-type format. AUIPC forms a 32-bit offset from the U-immediate, filling in the lowest 12 bits with zeros, adds this offset **to the address of the AUIPC instruction**, then places the result in register rd.

> The indirect jump instruction JALR (jump and link register) uses the I-type encoding. The target address is obtained by adding the sign-extended 12-bit I-immediate to the register rs1, then setting the least-significant bit of the result to zero. The address of the instruction following the jump (pc+4) is written to register rd. Register x0 can be used as the destination if the result is not required.

```asm6502
 652:	85aa                	mv	a1,a0
 654:	4505                	li	a0,1
 656:	00000097          	auipc	ra,0x0
 65a:	dce080e7          	jalr	-562(ra) # 424 <vprintf>
}
```

> `auipc ra 0x0` made ra the current address, namely 0x656, jalr made ra add the offset(10-digit) to approach the specified function. Here is 0x424.

![](./1.png)

> Return address is in ra and it's 0x38.

#### 5. Run the following code.

```
unsigned int i = 0x00646c72;
printf("H%x Wo%s", 57616, &i);
```

What is the output? [Here's an ASCII table](http://web.cs.mun.ca/~michael/c/ascii-table.html) that maps bytes to characters.

The output depends on that fact that the RISC-V is little-endian. If the RISC-V were instead big-endian what would you set `i` to in order to yield the same output? Would you need to change `57616` to a different value?

[Here's a description of little- and big-endian](http://www.webopedia.com/TERM/b/big_endian.html) and [a more whimsical description](http://www.networksorcery.com/enp/ien/ien137.txt).

![](./2.png)

> To convert 57616, we can already see that its 16-digit form is E110, so the former part is HE110, quite similar to HELLO.

![](./3.png)

> Due to little-endian, 0x00646c72 get reversed and they are r(0x72), l(0x6C), d(0x64), '\0'(0x00). Putting them together, it's `HE110, World\0`.

#### 6. In the following code, what is going to be printed after `'y='`? (note: the answer is not a specific value.) Why does this happen?

```
printf("x=%d y=%d", 3);
```

> if we rewrite the main() and `make fs.img`.

```c
void main(void) {
  printf("%d %d %d %d %d", 1,2,3,4,5);
  exit(0);
}
```

> It would be like this.

```asm6502
000000000000001c <main>:

void main(void) {
  1c:	1141                	addi	sp,sp,-16
  1e:	e406                	sd	ra,8(sp)
  20:	e022                	sd	s0,0(sp)
  22:	0800                	addi	s0,sp,16
  printf("%d %d %d %d %d", 1,2,3,4,5);
  24:	4795                	li	a5,5
  26:	4711                	li	a4,4
  28:	468d                	li	a3,3
  2a:	4609                	li	a2,2
  2c:	4585                	li	a1,1
  2e:	00000517          	auipc	a0,0x0
  32:	7aa50513          	addi	a0,a0,1962 # 7d8 <malloc+0xe4>
  36:	00000097          	auipc	ra,0x0
  3a:	600080e7          	jalr	1536(ra) # 636 <printf>
  exit(0);
  3e:	4501                	li	a0,0
  40:	00000097          	auipc	ra,0x0
  44:	27e080e7          	jalr	638(ra) # 2be <exit>
```

> What happened in printf:

```asm6502
void
printf(const char *fmt, ...)
{
 636:	711d                	addi	sp,sp,-96
 638:	ec06                	sd	ra,24(sp)
 63a:	e822                	sd	s0,16(sp)
 63c:	1000                	addi	s0,sp,32
 63e:	e40c                	sd	a1,8(s0)
 640:	e810                	sd	a2,16(s0)
 642:	ec14                	sd	a3,24(s0)
 644:	f018                	sd	a4,32(s0)
 646:	f41c                	sd	a5,40(s0)
 648:	03043823          	sd	a6,48(s0)
 64c:	03143c23          	sd	a7,56(s0)
  va_list ap;

  va_start(ap, fmt);
 650:	00840613          	addi	a2,s0,8
 654:	fec43423          	sd	a2,-24(s0)
  vprintf(1, fmt, ap);
 658:	85aa                	mv	a1,a0
 65a:	4505                	li	a0,1
 65c:	00000097          	auipc	ra,0x0
 660:	dce080e7          	jalr	-562(ra) # 42a <vprintf>
}	
```

> Now we look into the vprintf:

```asm6502
void
vprintf(int fd, const char *fmt, va_list ap)
{
  char *s;
  int c, i, state;

  state = 0;
  for(i = 0; fmt[i]; i++){
 448:	0005c903          	lbu	s2,0(a1)
 44c:	18090f63          	beqz	s2,5ea <vprintf+0x1c0>
 450:	8aaa                	mv	s5,a0
 452:	8b32                	mv	s6,a2
```

> a2 is placed at s6.

```asm6502
        printint(fd, va_arg(ap, int), 10, 1);
 4e4:	008b0913          	addi	s2,s6,8
 4e8:	4685                	li	a3,1
 4ea:	4629                	li	a2,10
 4ec:	000b2583          	lw	a1,0(s6)
 4f0:	8556                	mv	a0,s5
 4f2:	00000097          	auipc	ra,0x0
 4f6:	e8e080e7          	jalr	-370(ra) # 380 <printint>
 4fa:	8b4a                	mv	s6,s2
```

> So the first %d is the value of s6 and it extracts 4 bytes.

> When that is over, s6 add itself with 8 to point to the next value.
> 
> So printf("x=%d y=%d", 3); when 3 is printed and the next %d will to be reaching the first address of stack.
> 
> It will print the content of a2, but no one could determine what the previous value of register a2. So the result is undefined.