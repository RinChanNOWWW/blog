---
title: 简单的汇编学习笔记与总结
date: 2018-11-08 20:18:25
tags: 汇编
categories: 
  - 课程学习
---

<font color=#DC143C>**根据CSAPP第三章内容与课堂讲义总结，默认是x86-64系统，C语言。这里讨论的都是整数。**</font>
<font color=#DC143C>**因为我比较菜，写的比较垃圾，争取自己复习的时候可以看懂。**</font>

<!-- more -->

## 数据格式
| C声明 | Intel数据类型 | 汇编代码后缀 | 大小（字节） |
| :-----: | :-----: | :-----: | :-----: |
| char | 字节 | b | 1 |
| short | 字 | w | 2 |
| int | 双字 | l | 4 |
| long | 四字 | q | 8 |
| char* | 四字 | q | 8 |
| float | 单精度 | s | 4 |
| double | 双精度 | l | 8 |

-------------

## 整数寄存器
![](https://cdn.jsdelivr.net/gh/RinChanNOWWW/jsdelivrp-cdn@master/blog/images/learning-asm/64bit-register.png)

![](https://cdn.jsdelivr.net/gh/RinChanNOWWW/jsdelivrp-cdn@master/blog/images/learning-asm/32bit-register.png)

-------------

## 操作数
### 类型
- 1. 立即数：
直接表示常数值。例子：`$577`, `0x1F`。
- 2. 寄存器：
某个寄存器里的内容。例子: `%rax`, `%ecx`。
- 3. 内存引用：
根据计算出来的地址访问内存中的位置。例子：`(%rax)`, `(0x100)`。

### 引用内存的多种形式
- 1. (R) : Mem[Reg[R]] : 直接访问。如：`(%rax)`。
- 2. D(R) : Mem[Reg[R] + D] : 访问原始地址加上偏移量后的地址。如：`8(%rbp)`， 访问的是 `%rbp + 8` 地址的值。
- 3. D(Rb, Ri, s) : Mem(Reg[Rb] + Reg[Ri] \* s) : 比例变址寻址。如：`4(%rax, %rdx, 4)`，访问的是 `%rax + %rdx  * 4 + 4`地址的值。

--------------

## 基本的汇编指令
### 数据传送
<font color=#DC143C>MOV类：</font>MOV Src, Dst ：把Src上面的数据传送到Dst上

| 指令 | 描述 | 
| :-----: | :-----: |
| movb | 传送字节 |
| movw | 传送字 |
| movl | 传送双字 |
| movq | 传送四字 |
| movabsq | 传送绝对四字 |    

**注意事项**
- 常规的movq指令只能以表示为32位补码数字的立即数作为源操作数，然后把这个值扩展到64位再放到目的位置，
**而movabsq指令能够以任意64位立即数值作为源操作数，并且只能以寄存器作为目的。**
- 大多数情况中，MOV指令只会更新目的操作数指定的寄存器字节或内存位置，高位不变。**唯一的例外是movl指令以寄存器作为目的时，它会把寄存器的高位4字节设置为0**
- 传输**不能**从内存到内存

<font color=#DC143C>MOVZ和MOVS类：</font>MOVZ/MOVS Src, Dst ：将较小的源值复制到较大的目的时使用，分别是零扩展（剩余填充0）和符号扩展（剩余填充符号位）。

零扩展：

| 指令 | 描述 | 
| :-----: | :-----: |
| movzbw | 将做了零扩展的字节传送到字 |
| mozbl | 将做了零扩展的字节传送到双字 |
| movzwl | 将做了零扩展的字传送到双字 |
| movzbq | 将做了零扩展的字节传送到四字 |
| movzwq | 将做了零扩展的s字传送绝对四字 |   

符号扩展：

| 指令 | 描述 | 
| :-----: | :-----: |
| movsbw | 将做了符号扩展的字节传送到字 |
| mosbl | 将做了符号扩展的字节传送到双字 |
| movswl | 将做了符号扩展的字传送到双字 |
| movsbq | 将做了符号扩展的字节传送到四字 |
| movswq | 将做了符号扩展的字传送绝对四字 |   
| movslq | 将做了符号扩展的双字传送到四字 |
| cltq | 把%eax符号扩展到%rax |

**注意：cltq指令只能作用于寄存器%eax和%rax**
数据传输的例子：
1. 第一个例子

```x86asm
movabsq $0x0011223344556677, %rax   # %rax = 0x0011223344556677
movb    $-1, %al                    # %rax = 0x00112233445566FF
movw    $-1, %ax                    # %rax = 0x001122334455FFFF
movl    $-1, %eax                   # %rax = 0x00000000FFFFFFFF
movq    $-1, %rax                   # %rax = 0xFFFFFFFFFFFFFFFF
```

2.第二个例子

```x86asm
movabsq $0x0011223344556677, %rax   # %rax = 0x0011223344556677
movb    $0xAA, %dl                  # %dl  = 0xAA
movb %dl, %al                       # %rax = 0x00112233445566AA
movsbq %dl, %rax                    # %rax = 0xFFFFFFFFFFFFFFAA
movzbq %dl, %rax                    # %rax = 0x00000000000000AA
```

### 压入和弹出栈数据
- push src：把数据压入栈中
- pop dst：抽出栈顶数据
**注意：** 以上操作是通过寄存器%rsp中所存地址指向栈顶。抽出栈顶数据后，原来的数据保持在原来的内存位置中，直到被覆盖。
这里的相关内容还没细讲，以后再补吧。

### 加载有效地址
<font color=#DC143C>leaq指令：</font> leaq Src, Dst：直接将有效地址（即：把<font color=#FF8C00>括号内的值</font>，不读入对应内存的数据）写入到目的。
**leaq可以简洁地描述普通的算术操作**
例如：

```x86asm
leaq 7(%rdi, %rsi, 4), %rax         # 设%rdi总存数据x，%rsi中存数据y，则这条指令是将 x+4y+7 存入%rax中
```
### 其他算术和逻辑操作
这里先简单的给出这些操作：

| 指令 | 描述 |
| ----- | ----- |
| inc D | D = D + 1 |
| dec D | D = D - 1 |
| neg D | D = -D |
| not D | D = ~D |
| add S,D | D = D + S |
| sub S,D | D = D - S |
| imul S,D | D = D \* S |
| xor S,D | D = D ^ S |
| or S,D | D = D &#124; S |
| and S,D | D = D & S |
| sal k,D | D = D << k |
| shl k,D | D = D << k |
| sar k,D | D = D >><sub>算术</sub>k |
| shr k,D | D = D >><sub>逻辑</sub>k |

(注意到sal和shl是一样的，因为左移不会涉及符号位)

#### 移位操作
移位操作对w位长的数据值进行操作，移位量是有%cl寄存器的低m位决定的，这里2<sup>m</sup>=w，剩余高位会被忽略。
**所以，例如当寄存器%cl是十六进制值为0xFF(11111111)时，指令salb会移7位(111，二进制3位)，salw会移15位(1111，二进制4位)，sall会移31位(11111，二进制5位)，salq会移63位(111111，二进制5位)。** 这些位数也是对应指令能移动的最高位数。

这里来举个例子：一个32位的int数1，移动n=34位，计算1<<n，因为(34)<sub>10</sub>=(100010)<sub>2</sub> 取前五位00010，即2。所以1<<34等价于1<<2=4。其实就是让n mod 32(2<sup>m</sup>)。

所以习题3.60中
```x86asm
salq %cl, %rdx          # %ecx中的值为n，%rdx中的值是mask
```
所以这条语句的作用是 `mask = mask << n`，并不用截取%rcx的前八位~~(n & 0xFF)~~，直接移动n为即可，因为salq最多移63位(11111)，n太大了也会被截成64以下（只取n二进制下的前6位）。

#### 特殊算术操作
- 1.一个操作数
从上面的表我们可以看到，乘（mul和imul）和除（div和idiv）都是二元操作。这样的操作是在同位数数据采用的，比如：如果你用
`imulq S,D`指令，则代表你计算的内容是一个64位数乘一个64位数并得到一个64位数。我们在前面的学习中便知道，w位数乘w位数会先得到一个2w位数，然后再截取前w位得到最后的结果。但如果你想得到就是那个2w位数，以w=64位例，计算机会将其处理未这样的汇编指令：`imulq S`。这条指令的效果是：R[%rdx]:R[%rax]<--S/*R[%rax]，即把S中的数与%rax中的数做补码乘法后，**将乘积的高64位存在%rdx中，低64位存在%rax中。** 当操作为无符号操作`mulq S`时同理。
这里有一个两个无符号64位数乘法得到无符号128位数的例子：
C语言代码为：

```c
void store_uprod(uint128_t *dest, uint64_t x, uint64_t y){
    *dest = x * (uint128_t)y;
}
```

他的汇编代码为：

```x86asm
store_uprod:
    movq %rsi, %rax                 
    mulq %rdx
    movq %rax, (%rdi)
    movq %rdx, 8(%rdi)              # 小端机器
    ret
```

以64位为例，对于除法`idivl S`，他会把R[%rdx]:R[%rax]作为**被除数**（128位），S为除数，将结果的商存在%rax中，余存在%rdx中。如果被除数是64位，则%rdx中全为0（无符号位）或全为%rax的符号位（有符号运算），这两个操作可以用`cqto`（R[%rdx]:R[%rax]<--R[%rax]完成。

对于64位以下的操作mulb / mulw / mull，一元乘除操作也是同理，**另一个源操作数会隐含在R[%al] / R[%ax] / R[%eax]中，结果存在R[%ax] / R[%dx]:R[%ax] / R[%edx]:R[%eax]中**，除法同理。
还需要注意一点的是有符号乘法，若要取2w位应该采用“布斯乘法”，也就是习题2.75让我们推导的计算两个w位补码运算结果的高w位（一共2w位）的方法：(x, y表示有符号数，x',y'表示与其二进制表示相同的无符号数)

>  (x' \* y')<sub>高w位</sub> = (x \* y)<sub>高w位</sub> + x \* y的符号位 + y \* x的符号位

此公式的推导思路源于教材公式2.18。
例子：
无符号数0xB4 (180) 乘 无符号数0x11 (17) 结果为0xBF4 (3060)。
有符号数0xB4(-76) 乘 有符号数0x11 (17) 结果为0xFAF4 (-1292) ，并非~~(0xBF4)~~。(0xB=(0xFA+0x11) mod 64)
- 2.三个操作数
指令： MUL Imm, Src, Reg
功能：将Src和立即数Imm相乘，结果存在Reg中。
例子：R[%eax] = 0xB4, R[%ebx] = 0x11, M[0xF8] = 0xA0，执行指令`imull $-16, (%eax, %ebx, 4), %eax`的效果：
R[%eax]<-- (-16) x M[R[%eax] + R[%ebx] x 4] = (-16) x M[0xB4 + 0x11 << 2] = (-16) x M[0xF8] = (-16) x 0xA0 = 0xFFFFFF60 << 4（做一个补码操作去除负号）= 0xFFFFF600 = -2560

#### 整数乘除指令总结
- 乘法
    - 一个操作数
    若给出一个操作数Src，则另一个源操作数隐含在R[%al] / R[%ax] / R[%eax]中，将Src和前述寄存器（累加器accumulate）中内容相乘，结果存放在R[%ax]（16位）/ R[%dx]:R[%ax]（32位）/ R[%edx]:R[%eax]（64位）中。
    - 两个操作数
    MUL Src, Dst : Dst<--Dst MUL Src
    - 三个操作数
    MUL Imm, Src, Reg : Reg<--Imm MUL Src
- 除法
    - 除数为8位，则16位被除数在R[%ax]中，商送回R[%al]，余数在R[%ah]
    - 除数为16位，则32位被除数在R[%dx]:R[%ax]中，商送回R[%ax]，余数在R[%dx]
    - 除数为32位，则64位被除数在R[%edx]:R[%eax]中，商送回R[%eax]，余数在R[%edx] 
    - 除数为64位，则128位被除数在R[%rdx]:R[%rax]中，商送回R[%rax]，余数在R[%rdx]

### 控制指令
在下面介绍

### 其他操作
输入输出指令(IN, OUT)和标志传送指令(PUSHF, POPF)等还没细讲，以后再补。

---------------

## 控制
### 条件码
除了整数寄存器，CPU还维护着一组**单个位**的条件码寄存器，他们描述了**最近**的算术或逻辑操作的属性。可以检测这些寄存器来执行条**条件分支指令**。
最常用的条件码有：
- **CF**：进位标志。最近的操作使最高位产生了进位（加分有进位（carry），减法有借位（borrow））。可以用来检查**无符号**操作的溢出。
- **ZF**：零标志。最近的操作得出结果为0。（所有位上数字都是0）
- **SF**：符号标志。最近的操作得到的结果为负数。
- **OF**：溢出标志。最近的操作导致一个补码溢出——正溢出或负溢出。(如y......+y......=z......，或y.......-z......=z...... 「其中z=~y」等)

### 定点算术运算和逻辑运算对条件码的影响
- ADD：影响OF, ZF, SF, CF。
- SUB：影响OF, ZF, SF, CF。有借位，即减数>被减数，则CF=1。两个数符号相反但结果符号与减数相同，则OF=1
- INC：影响OF, ZF, SF。**注意：不会影响CF**，也就是说不会产生进位信息
- DEC：影响OF, ZF, SF。**注意：同INC**
- NEG：影响OF, ZF, SF,  CF。相当于用0减操作数或者取反+1，OF变化同减法（所以只有当操作数为100...0时，OF才会变为1）
- CMP：影响OF, ZF, SF, CF。
- MUL：只影响OF, CF。乘积高一半为0，则CF=OF=0，否则是1。
- IMUL：只影响OF, CF。乘积高一半为低一半的符号扩展，则CF=OF=0，否则是1。
- DIV, IDIV：不影响上述条件码。
- AND, OR, XOR, TEST：**都会使OF和CF变为0**，ZF和SF根据结果设置。
- **NOT：不影响标志**
- SHL, SHR, SAL, SAR, ROL, ROR：CF=移入的数值，ZF和SF根据结果设置，如果最高位变化，则OF=1，否则为0。

[参考链接](http://abcdxyzk.github.io/blog/2012/12/20/assembly-cmd-flags/)

例子：R[%ax]=0xFFFA, R[%bx]=0xFFF0，执行指令（Intel格式）`add ax bx`：
R[%ax]<--R[%ax] + R[%bx] = 0xFFFA + 0xFFF0 = 0xFFEA, R[%bx]中内容不变，CF = 1, OF = 0, ZF = 0, SF = 1。
对于上述例子，**若是无符号整数运算，则CF=1说明结果溢出。若是有符号整数运算，OF=0说明结果没有溢出。**

### 比较和控制指令
**这两种指令不修改任何寄存器的值，只设置条件码**
- 1. CMP (cmpb, cmpw, cmpl, cmpq)
    CMP S1, S2：就是计算**S2 - S1**，以设置条件码得以看出比较的结果。
    - CF = 1: 发生了进位或借位（这里做减法一般是借位，借位了就表明S2 < S1）
    - ZF = 1: S1 = S2
    - SF = 1: S2 - S1 < 0（补码运算意义上的）
    - OF = 1: (a > 0 && b < 0 && (a - b) < 0) || (a < 0 && b > 0 && (a - b) > 0)
- 2. TEST (testb, testw, testl, testq)
    TEST S1, S2：就是计算**S1 & S2**，以设置条件码。
    - ZF = 1: S1 & S2 = 0
    - SF = 1: S1 & S2 < 0（补码运算意义上的）
    经常使用这个指令测试一个数是不是负数：`testq %rax, %rax`

### 设置指令
SET类的指令可以将一个字节的值设置为条件码的某种组合，这种指令的目的操作数是**低位单字节寄存器之一或一个字节的内存位置**（如%al），一般是配合比较和测试指令使用，下面列出常用的SET类指令：

| 指令 | 同义名 | 效果 | 设置条件 |
| ----- | ----- | ----- | ----- |
| sete D | setz | D <-- ZF | 相等/零 |
| setne D | setnz | D <-- ~ZF | 不等/非零 |
| sets D |  | D <-- SF | 负数 |
| setns D |  | D <-- ~SF | 非负数 |
| setg D | setnle | D <-- ~(SF ^ OF) & ~ZF | 有符号> (greater)|
| setge D | setnl | D <-- ~(SF ^ OF) | 有符号 >= |
| setl D | setnge | D <-- SF ^ OF | 有符号< |
| setle D | setng | D <-- (SF ^ OF) &#124; ZF | 有符号<= |
| seta D | setnbe | D <-- ~CF & ~ZF | 无符号> (above) |
| setae D | setnb | D <-- ~CF | 无符号>= |
| setb D | setnae | D <-- CF | 无符号< (below) |
| setbe D | setna | D <-- CF &#124; ZF | 无符号<= |

下面是一个例子：
设C函数：

```c
int compare(long x, long y)
{
    return x > y;
}
```

其汇编代码为：

```x86asm
compare:                        # x in %rdi, y in %rsi
    cmpq %rsi, %rdi             # compute x - y
    setg %al                    # 大于则设置为1，否则为0
    movzbl %al, %eax            # 这项操作是使 %eax(and %rax) 上其他位的数据全部清空为0，保证返回数据只为%al上的数据
    ret
```

### 跳转指令
等讲了再补

## 未完待续
未完待续……
