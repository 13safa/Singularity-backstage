# CSAPP 第三章 Note

> 程序就是状态机。——jyy, on 2022 NJU OS Lecture 01

这句话道出了程序的本质. 实际上, 如今的计算机本质上就是通用[图灵机](https://plato.stanford.edu/entries/turing-machine/), 而我们研究汇编语言就是研究程序在机器层面上的运行.

程序的进行依赖于`rip`寄存器, 它储存的值是当前运行的指令地址. 程序的进行实际上就是一条指令一条指令的移动(你可以在`gdb`里随时监控`rip`). 这和图灵机读写控制的方式本质上是一致的!

> 书中出示的汇编指令是`AT&T`格式, 为`gcc`和`gdb`编译选项默认.
> 大部分情况下两种格式是互通的. 唯一值得注意的是读取顺序, 两者**源操作数**和**目的操作数**的顺序是**反的**. 例如下面的语句表示栈指针减8:
>
>```asm
>subq     $8,%rsp             # ATT flavor
>sub      rsp,8               # Intel flavor
>```

## 访问值和访问地址

`%rsp` 表示 `rsp` 寄存器储存的值, 通常来说栈指针保存的是一个地址. 对于更一般的寄存器, 例如`rax`寄存器, 是可以保存值的, 而且通常是返回值. 我们用 $r_a$ 表示寄存器`a` 存储的值.
内存本质上是一个很长的数组, **内存引用**表示 $M[r_a]$, 例如 `(%rsp)`表示`*rsp`, 也就是`rsp` 寄存器储存地址上的值.

`lea`和`mov`指令间的区别: 前者常用于简单的算术运算, 例如:

```ASM
/* i in %rsi */
leal    (%eax,%eax,2), %eax
```

实际上进行的操作是: 源操作数计算为`S = M[%eax + 2 * %eax] = M[3 * %eax]`, 然后求出 `&S`(`S`的有效地址), 然后将这个值复制给 `D`.
如果对`%rax`解引用, 例如`gdb`指令 `p *(int*) $rax`, 可以得到`3i`.

`mov`指令可以干两件事: 把内存地址复制给另一个寄存器, 或者将一个值复制给寄存器.

```C
int x = a[i];
```

```ASM
/* x in %eax, a in %rdx, i in %rcx */
movl    (%rdx,%rcx,4),%eax
```

源操作数`S = M[a + 4*i]`, `a`是`int`数组的起始地址. 将内存地址`S`上的值赋值给寄存器`rax`. 此时 `p $rax` 就是`a[i]`.

> 数组和结构本质上就是人为规定的一段内存.
> 上面的例子显示了数组`int a[N]`可以用`*(a+i)`访问`a[i]`的本质. 对于下面的结构体:
>
> ```C
> typedef struct node {
>     long val;
>     node* next; 
> } node;
> ```
>
> 假设`node`类型`node1`的指针为`ptr`, 那么`*ptr`是什么? `*(ptr+8)`代表什么? `**(ptr+8)` 又是什么?
> `mov    0x8(%rdx),%rdx` 假设`rdx`是`ptr`, 这句指令代表什么?

区别下面的指令:

```ASM
   mov    0x20(%rsp),%rbx 
   lea    0x28(%rsp),%rax 
```

其中`rbx`的值是内存地址`$rsp+32`上储存的值, `rax`现在是一个指针, 它的值是`$rsp+40`, 也就是一个内存地址.

## 控制语句

跳转命令`jmp`等价于C中的`goto`.
`if`语句对应条件跳转指令.
循环控制语句可以通过`if`和`goto`所模拟.

```C
long absdiff_se(long x, long y) {
    long res;
    if (x < y) {
        res = y - x;
    }
    else {
        res = x - y;
    }
    return res;
}
```

```ASM
/* x in %rdi, y in %rsi */
absdiff_se:
   cmpq %rsi, %rdi              ; x - y              
   jge .L6                      ; if >= goto .L6
   movq %rsi, %rax              ; res = y
   subq %rdi, %rax              ; res -= x
   ret                          ; return
.L6:                         
   movq %rdi, %rax              ; res = x
   subq %rsi, %rax              ; res -= y
   ret                          ; return
```

判断条件(`if`括号内的语句)基于一组条件码寄存器, 通过`test`或`cmp`指令设置.

> **Q:** 这里发生了什么?
> **A:** `cmq`等同为不修改源操作数的`sub`指令(`a-b`), `test`类似`and`指令(`a&b`), 程序根据结果设置`OF`和`SF`以及其他的条件码寄存器(默认为0).
>
>假设结果小于`0`, 那么`SF`会被设置为`1`; 还会判断是否发生数据类型的溢出, 假设溢出, `OF`会被设置为`1`.
>
>`jge`会先对条件码进行位运算, 这里:`~(SF ^ OF)`, 如果结果为`1`, 那么跳转, 否则程序继续执行下去.

在实际阅读汇编代码的过程中, 可以忽略这部分细节, 明白程序按照条件跳转即可.

`-O1`级别以上的优化还会出现条件移动指令. 此处从略.

`do while` 作为最不常用的循环, 模拟方式却是最简单的. 它的流程就和它本身一致.

```C
loop:
    body
    t = test_expr;
    if(t)
        goto loop;
```

和`while`语句的区别是, `while`语句在进入之前会进行一次条件判断.

```C
goto test;
loop:
    body
test:
    t = test_expr;
    if(t)
        goto loop;
```

至于最常用的`for`语句, 它有等价的`while`表述, 此处从略.

做完了要不要试试手? 要不要拆个[二进制炸弹](http://csapp.cs.cmu.edu/3e/bomb.tar)试试手:) 在Linux下输入下面的指令:

```bash
wget http://csapp.cs.cmu.edu/3e/bomb.tar
tar xvf bomb.tar
cd bomb
```

## 函数调用的过程

函数调用的过程满足**栈**的前进后出(First in, Last out)原则.

> **Q:** 怎么画函数栈帧的示意图? 是栈顶放最上面, 还是内存高位的一端放在最上面?
> **A:** 看个人习惯. [UP?](http://www.stackgrowsup.com/) 还是 [DOWN?](http://www.stackgrowsdown.com/) 只要你看别人的示意图不会被绕晕就可以.

当过程P执行过程Q时, 会把返回地址, 也就是`call`指令的下一句(记作A)压入栈中(PC++), Q返回时就会从A开始继续执行. 我们认为A是P栈帧的一部分. 对应的 `ret` 会将A从栈中弹出, 然后将`rip`设置为A.

调用函数的时候怎么传入参数? 一般来说, 如果还有空闲的寄存器, 程序在调用过程Q之前, 在过程P, 会先将参数移动到对应的寄存器, 例如第一个参数会移动到指定寄存器`rdi`上(`rsi`和`rdx`以此类推), 然后在过程Q中调用这几个寄存器, "共享"过程中, 实际上就实现了参数的传递.

在函数调用的一开始, 会为过程Q分配栈空间. 因为程序运行地址是往低址增长, 所以扩大栈需要减小栈指针`rsp`(指向栈顶元素). 将栈指针减小一个适当的值可以为没有初始值的数据在栈上分配空间.

通常来说`push`指令(将参数压入栈)就可以满足需要. 压栈实际上就是复制一个参数. 某些情况下, 还需要通过主动利用`sub`指令来分配栈帧.
在结束调用时, 对应的需要增加栈指针, 对应指令`pop`, 本质上是`add`一个特定的值.

> **Q1:** `push`指令到底是什么意思?
> **A:** `push`指令实际上可以通过更简单的指令模拟出来: `pushq %rbp` 等价于 `subq $8,%rsp`然后 `movq %rbp,(%rsp)`. 区别在于`pushq`指令占的字节数更少. 同理可以知道`pop`的含义.
>
> **Q2:** 还有什么时候需要用到栈帧?
> **A:** 上面提到了, 如果寄存器不足够放置所有的本地数据, 就需要用栈去储存. 此外, 对一个临时变量取地址`&`, 局部变量是数组或结构, 同样需要分配栈帧. 栈帧的具体分配可以查阅书.

传递参数存在保护机制. 方才我们提到参数传入靠的是`push`和`pop`, 实际是参数的复制. 在此之外, 我们还有一组特殊的寄存器: `rbx`, `rbp`和 `r12 ~ r15`寄存器被称为**被调用者保存**寄存器. 在P调用Q的过程中, Q必须保护这些寄存器的值, 使得当Q返回P的时候与调用Q时是一样的.

```C
long P(long x, long y)
{
    long u = Q(y);
    long v = Q(x);
    return u + v;
}
```

第四行`sub $8,%rsp`是为了内存对齐. 第五行把`x`储存到了`rbp`寄存器上, 这样就不需要担心`rdi`寄存器后来被修改导致`x`丢失. 同样`rbx`储存了`Q(y)`的值, 这样就不用担心`rax`寄存器修改导致`Q(y)`的丢失.

```ASM
long P(long x, long y)
x in %rdi, y in %rsi
1     P:
2       pushq %rbp                Save %rbp
3       pushq %rbx                Save %rbx
4       subq $8, %rsp             Align stack frame
5       movq %rdi, %rbp           Save x
6       movq %rsi, %rdi           Move y to first argument
7       call Q                    Call Q(y)
8       movq %rax, %rbx           Save result
9       movq %rbp, %rdi           Move x to first argument
10      call Q                    Call Q(x)
11      addq %rbx, %rax           Add saved Q(y) to Q(x)
12      addq $8, %rsp             Deallocate last part of stack
13      popq %rbx                 Restore %rbx
14      popq %rbp                 Restore %rbp
15      ret
```

理解了函数调用的过程, 实际上你就可以实现将任何递归程序改写成非递归! 比方说, 写一个非递归版的[汉诺塔程序](demonstration/hanoi.c).

此外, 关于栈帧有一个有趣的话题: 内存越界和缓冲区溢出. 主要是这样一个库函数`gets()`, 它可以无视字符串长度向字符数组写入, 引起内存越界, 小则修改栈上的值, 大则修改返回地址, 使得函数调用到奇怪的地方!

有关这部分, CSAPP提供了[attack lab](http://csapp.cs.cmu.edu/3e/target1.tar), 让你模拟为一名hacker去攻击程序! 很有意思.
