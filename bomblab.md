# bomblab实验报告
## 刘金成 2023302111097
## phase 0
这个是我自己加的一些小tips和前置知识
1. **记得每次run之前都要给炸弹打断点 ！！！**

2.  一些常用的gdb调试命令：
```
gdb bomb
获取帮助
help
设置断点(可以在函数也可以在某个地址)
break explode_bomb
break phase_1
break *0x4015d9
在断点停下后继续执行(continue)注意不要在炸弹断点用
c
开始运行
run
检查汇编 会给出对应的代码的汇编
disas 
打印指定寄存器
print $rsp
每步执行
stepi
检查寄存器或某个地址,打印指定位数(中间时是指定参数，可以是单位或者格式)
x/32xb $rsp
```

3. 在过了几个phase之后可以把答案存在文件里用gdb的设置参数功能省去每次输入的麻烦:
```
(gdb) set args ans.txt
```
4. 在编辑3的答案文档时会用到vim的一些基本操作，如三种模式的切换，如何复制粘贴保存等等，google解决
5. 常用 `sscanf` 来读取字符串，第二个参数（即`%esi`中的）是格式化字符串，可以通过打印它确认key的格式
6. 每个人的bomb都不一样，本文档仅供参考，照抄会"炸"得很惨

## phase 1
### 汇编代码：
用objdump得到整个bomb的汇编代码

找到phase_1
```
0000000000400ef0 <phase_1>:
  400ef0:       48 83 ec 08             sub    $0x8,%rsp
  400ef4:       be 30 25 40 00          mov    $0x402530,%esi
  400ef9:       e8 30 04 00 00          callq  40132e <strings_not_equal>
  400efe:       85 c0                   test   %eax,%eax
  400f00:       74 05                   je     400f07 <phase_1+0x17>
  400f02:       e8 d2 06 00 00          callq  4015d9 <explode_bomb>
  400f07:       48 83 c4 08             add    $0x8,%rsp
  400f0b:       c3                      retq
```
### 思路：
```
0000000000400ef0 <phase_1>:
  400ef0:       48 83 ec 08             sub    $0x8,%rsp
  400ef4:       be 30 25 40 00          mov    $0x402530,%esi
  400ef9:       e8 30 04 00 00          callq  40132e <strings_not_equal>
```
注意到 `0x400ef4` 处的`mov`指令将 `0x402530`的值转移到`%esi`中，作为第二个参数传入函数 `strings_not_equal`（位于`40132e`）

查看该函数的汇编代码可知是比较传入的字符串是否不相等并返回结果到`%eax`中，再test %eax 是否为0，若ZF为零则执行`je`跳过炸弹（两字符串相等），不为零则爆炸，炸弹在`4015d9`

由此推测需要输入的key即在`0x402530`处，下面使用gdb进行调试:
```
(gdb) break *0x4015d9
Breakpoint 1 at 0x4015d9
(gdb) break phase_1
Breakpoint 2 at 0x400ef0
```
先给炸弹和phase_1处打断点
```
(gdb) run
Starting program: /headless/Desktop/bomb2023302111097/bomb 
Welcome to my fiendish little bomb. You have 6 phases with
which to blow yourself up. Have a nice day!
a

Breakpoint 2, 0x0000000000400ef0 in phase_1 ()
```
随便输入什么在炸弹前停下，此时已经运行了函数 `strings_not_equal`，查看`0x402530`处有什么
```
(gdb) print (char*)0x402530
$1 = 0x402530 "You can Russia from land here in Alaska."
```
可知key为 `You can Russia from land here in Alaska.`

### 思考：
比较简单
要运行经过函数 `strings_not_equal`目标字符串才被加载，才能尝试print目标字符串

## phase 2
### 汇编代码：
```
0000000000400f0c <phase_2>:
  400f0c:       55                      push   %rbp
  400f0d:       53                      push   %rbx
  400f0e:       48 83 ec 28             sub    $0x28,%rsp
  400f12:       48 89 e6                mov    %rsp,%rsi
  400f15:       e8 f5 06 00 00          callq  40160f <read_six_numbers>
  400f1a:       83 3c 24 00             cmpl   $0x0,(%rsp)
  400f1e:       75 07                   jne    400f27 <phase_2+0x1b>
  400f20:       83 7c 24 04 01          cmpl   $0x1,0x4(%rsp)
  400f25:       74 21                   je     400f48 <phase_2+0x3c>
  400f27:       e8 ad 06 00 00          callq  4015d9 <explode_bomb>
  400f2c:       eb 1a                   jmp    400f48 <phase_2+0x3c>
  400f2e:       8b 43 f8                mov    -0x8(%rbx),%eax
  400f31:       03 43 fc                add    -0x4(%rbx),%eax
  400f34:       39 03                   cmp    %eax,(%rbx)
  400f36:       74 05                   je     400f3d <phase_2+0x31>
  400f38:       e8 9c 06 00 00          callq  4015d9 <explode_bomb>
  400f3d:       48 83 c3 04             add    $0x4,%rbx
  400f41:       48 39 eb                cmp    %rbp,%rbx
  400f44:       75 e8                   jne    400f2e <phase_2+0x22>
  400f46:       eb 0c                   jmp    400f54 <phase_2+0x48>
  400f48:       48 8d 5c 24 08          lea    0x8(%rsp),%rbx
  400f4d:       48 8d 6c 24 18          lea    0x18(%rsp),%rbp
  400f52:       eb da                   jmp    400f2e <phase_2+0x22>
  400f54:       48 83 c4 28             add    $0x28,%rsp
  400f58:       5b                      pop    %rbx
  400f59:       5d                      pop    %rbp
  400f5a:       c3                      retq
```
### 思路：
先看开头
```
  400f0e:       48 83 ec 28             sub    $0x28,%rsp     
  400f12:       48 89 e6                mov    %rsp,%rsi      
  400f15:       e8 f5 06 00 00          callq  40160f <read_six_numbers>   
```
先分配了40 字节的栈空间，栈指针赋值给 %rsi 寄存器，然后调用 `read_six_numbers `函数（在`40160f`），phase2的key应该是六个数

再看看函数`read_six_numbers `的汇编似乎就是读六个数然后存到栈上，用gdb确认一下，在`read_six_numbers`的return打一个断点（`400f5a`）
```
Phase 1 defused. How about the next one?
1 2 3 4 5 6

Breakpoint 2, 0x0000000000401650 in read_six_numbers ()
(gdb) x /32xb $rsp
0x7fffffffe738:	0x1a	0x0f	0x40	0x00	0x00	0x00	0x00	0x00
0x7fffffffe740:	0x01	0x00	0x00	0x00	0x02	0x00	0x00	0x00
0x7fffffffe748:	0x03	0x00	0x00	0x00	0x04	0x00	0x00	0x00
0x7fffffffe750:	0x05	0x00	0x00	0x00	0x06	0x00	0x00	0x00
(gdb) 
```
看到按顺序存了 1 2 3 4 5 6，说明猜想没错，再看下面的
```
  400f1a:       83 3c 24 00             cmpl   $0x0,(%rsp)  ; 比较栈顶元素和0是否相等
  400f1e:       75 07                   jne    400f27 <phase_2+0x1b>   不相等爆炸
  400f20:       83 7c 24 04 01          cmpl   $0x1,0x4(%rsp)   比较第二个数和1是否相等
  400f25:       74 21                   je     400f48 <phase_2+0x3c>   相等跳转到 400f48
```
栈的组织结构如图，`%rsp`是栈顶指针，`0x4(%rsp)`把第一个数加上4移到下一个数，可见第一个数是0，第二个数是1
```
//low address
%rsp+0    0
%rsp+4    1
%rsp+8    ?
%rsp+12   ?
%rsp+16   ?
%rsp+20   ?
//high address
```
跳到 `loop_start`以后
```
  400f48:       48 8d 5c 24 08          lea    0x8(%rsp),%rbx   %rbx第三个数
  400f4d:       48 8d 6c 24 18          lea    0x18(%rsp),%rbp   %rbp栈帧结尾
  400f52:       eb da                   jmp    400f2e <phase_2+0x22>   跳到loop_start
```
```
loop_start:
  400f2e:       8b 43 f8                mov    -0x8(%rbx),%eax   将当前元素的前面第二个数移入 %eax
  400f31:       03 43 fc                add    -0x4(%rbx),%eax   将当前元素的前一个数和前第二个数相加
  400f34:       39 03                   cmp    %eax,(%rbx)       比较eax和当前元素(rbx)是否相等
  400f36:       74 05                   je     400f3d <phase_2+0x31>   如果相等，跳到400f3d//不炸
  400f38:       e8 9c 06 00 00          callq  4015d9 <explode_bomb>   
  400f3d:       48 83 c3 04             add    $0x4,%rbx        移动到下一个元素
  400f41:       48 39 eb                cmp    %rbp,%rbx        结束？
  400f44:       75 e8                   jne    400f2e <phase_2+0x22>   不相等400f2e//circle
  400f46:       eb 0c                   jmp    400f54 <phase_2+0x48>   400f54//accept
```
可见`%rbx`是一个游标，一开始在第三个元素也就是`%rsp+8`处，比较前两个元素相加的值（存在`%eax`里）和游标处的值是否相等，如果相等就把游标往后移一个元素，判断要不要结束再重复上面的步骤，所以可以得到递推：`D[n]=D[n-1]+D[n-2]` ，故key为 0 1 1 2 3 5 这六个数

用gdb尝试一下，在释放栈前打断点（`400f54`，breakpoint 2)，输入推出的key
```
Phase 1 defused. How about the next one?
0 1 1 2 3 5

Breakpoint 2, 0x0000000000400f54 in phase_2 ()
(gdb) x /32xb $rsp
0x7fffffffe740:	0x00	0x00	0x00	0x00	0x01	0x00	0x00	0x00
0x7fffffffe748:	0x01	0x00	0x00	0x00	0x02	0x00	0x00	0x00
0x7fffffffe750:	0x03	0x00	0x00	0x00	0x05	0x00	0x00	0x00
0x7fffffffe758:	0xb1	0x14	0x40	0x00	0x00	0x00	0x00	0x00
(gdb) x /8xb $rbx
0x7fffffffe758:	0xb1	0x14	0x40	0x00	0x00	0x00	0x00	0x00
```
可见游标已经移动到了栈尾说明前面的数匹配上了（其实能在breakpoint 2停下而不是在炸弹前停下已经说明匹配正确了），phase2通过

### 思考：
了解了栈和栈指针，思考这六个数的储存结构
使用gdb打印寄存器中的值和寄存器作为指针后面一些位置的值
有一点疑惑是为什么判停是和 rsp+18比较而不是+20

## phase 3
### 汇编代码：
```
0000000000400f5b <phase_3>:
  400f5b:       48 83 ec 18             sub    $0x18,%rsp
  400f5f:       48 8d 4c 24 08          lea    0x8(%rsp),%rcx
  400f64:       48 8d 54 24 0c          lea    0xc(%rsp),%rdx
  400f69:       be 01 28 40 00          mov    $0x402801,%esi
  400f6e:       b8 00 00 00 00          mov    $0x0,%eax
  400f73:       e8 b8 fc ff ff          callq  400c30 <__isoc99_sscanf@plt>
  400f78:       83 f8 01                cmp    $0x1,%eax
  400f7b:       7f 05                   jg     400f82 <phase_3+0x27>
  400f7d:       e8 57 06 00 00          callq  4015d9 <explode_bomb>
  400f82:       83 7c 24 0c 07          cmpl   $0x7,0xc(%rsp)
  400f87:       77 66                   ja     400fef <phase_3+0x94>
  400f89:       8b 44 24 0c             mov    0xc(%rsp),%eax
  400f8d:       ff 24 c5 90 25 40 00    jmpq   *0x402590(,%rax,8)
  400f94:       b8 00 00 00 00          mov    $0x0,%eax
  400f99:       eb 05                   jmp    400fa0 <phase_3+0x45>
  400f9b:       b8 e8 02 00 00          mov    $0x2e8,%eax
  400fa0:       2d 7a 01 00 00          sub    $0x17a,%eax
  400fa5:       eb 05                   jmp    400fac <phase_3+0x51>
  400fa7:       b8 00 00 00 00          mov    $0x0,%eax
  400fac:       05 46 02 00 00          add    $0x246,%eax
  400fb1:       eb 05                   jmp    400fb8 <phase_3+0x5d>
  400fb3:       b8 00 00 00 00          mov    $0x0,%eax
  400fb8:       2d b9 02 00 00          sub    $0x2b9,%eax
  400fbd:       eb 05                   jmp    400fc4 <phase_3+0x69>
  400fbf:       b8 00 00 00 00          mov    $0x0,%eax
  400fc4:       05 b9 02 00 00          add    $0x2b9,%eax
  400fc9:       eb 05                   jmp    400fd0 <phase_3+0x75>
  400fcb:       b8 00 00 00 00          mov    $0x0,%eax
  400fd0:       2d b9 02 00 00          sub    $0x2b9,%eax
  400fd5:       eb 05                   jmp    400fdc <phase_3+0x81>
  400fd7:       b8 00 00 00 00          mov    $0x0,%eax
  400fdc:       05 b9 02 00 00          add    $0x2b9,%eax
  400fe1:       eb 05                   jmp    400fe8 <phase_3+0x8d>
  400fe3:       b8 00 00 00 00          mov    $0x0,%eax
  400fe8:       2d b9 02 00 00          sub    $0x2b9,%eax
  400fed:       eb 0a                   jmp    400ff9 <phase_3+0x9e>
  400fef:       e8 e5 05 00 00          callq  4015d9 <explode_bomb>
  400ff4:       b8 00 00 00 00          mov    $0x0,%eax
  400ff9:       83 7c 24 0c 05          cmpl   $0x5,0xc(%rsp)
  400ffe:       7f 06                   jg     401006 <phase_3+0xab>
  401000:       3b 44 24 08             cmp    0x8(%rsp),%eax
  401004:       74 05                   je     40100b <phase_3+0xb0>
  401006:       e8 ce 05 00 00          callq  4015d9 <explode_bomb>
  40100b:       48 83 c4 18             add    $0x18,%rsp
  40100f:       c3                      retq

```
### 思路：
先看要读什么数据
 ```
  400f69:       be 01 28 40 00          mov    $0x402801,%esi
  400f6e:       b8 00 00 00 00          mov    $0x0,%eax
  400f73:       e8 b8 fc ff ff          callq  400c30 <__isoc99_sscanf@plt> 
  400f78:       83 f8 01                cmp    $0x1,%eax              sscanf的返回值是否大于1
  400f7b:       7f 05                   jg     400f82 <phase_3+0x27>  大于1跳0x400f82
  400f7d:       e8 57 06 00 00          callq  4015d9 <explode_bomb>  boom
 ```
 可见这个phase需要读入超过一个数放在栈中
 看看读入格式
```
(gdb) x /s 0x402801
0x402801:	"%d %d"
```
所以读两个数，分别为`0xc(%rsp)` 和`0x8(%rsp)`
```
 400f82:       83 7c 24 0c 07          cmpl   $0x7,0xc(%rsp)       
 400f87:       77 66                   ja     400fef <phase_3+0x94>  第一个数大于7boom
 400f89:       8b 44 24 0c             mov    0xc(%rsp),%eax        
 400f8d:       ff 24 c5 90 25 40 00    jmpq   *0x402590(,%rax,8)    
```
`400f8d`处是一条跳转指令，有跳转表，应该是switch函数，根据输入的第一个数跳转而且第一个数不能大于7
跳转表位于 `0x402590`处，用gdb打印出来是
```
Breakpoint 1, 0x00000000004015d9 in explode_bomb ()
(gdb) x/8a 0x402590
0x402590:	0x400f9b <phase_3+64>	0x400f94 <phase_3+57>
0x4025a0:	0x400fa7 <phase_3+76>	0x400fb3 <phase_3+88>
0x4025b0:	0x400fbf <phase_3+100>	0x400fcb <phase_3+112>
0x4025c0:	0x400fd7 <phase_3+124>	0x400fe3 <phase_3+136>
```
分别对应第一个数从0到7的八种情况，继续翻译下面的case语句：
```
400f94:    b8 00 00 00 00          mov    $0x0,%eax              eax=0
400f99:    eb 05                   jmp    400fa0 <phase_3+0x45>    跳0x400fa0

400f9b:    b8 e8 02 00 00          mov    $0x2e8,%eax          eax=0x2e8
400fa0:    2d 7a 01 00 00          sub    $0x17a,%eax           eax-0x17a
400fa5:    eb 05                   jmp    400fac <phase_3+0x51>  跳0x400fac

400fa7:    b8 00 00 00 00          mov    $0x0,%eax            eax=0
400fac:    05 46 02 00 00          add    $0x246,%eax            eax+0x246
400fb1:    eb 05                   jmp    400fb8 <phase_3+0x5d>  跳0x400fb8

400fb3:    b8 00 00 00 00          mov    $0x0,%eax             eax=0
400fb8:    2d b9 02 00 00          sub    $0x2b9,%eax           eax-0x2b9
400fbd:    eb 05                   jmp    400fc4 <phase_3+0x69>  跳0x400fc4

400fbf:    b8 00 00 00 00          mov    $0x0,%eax            eax=0
400fc4:    05 b9 02 00 00          add    $0x2b9,%eax           eax+0x2b9
400fc9:    eb 05                   jmp    400fd0 <phase_3+0x75>  跳0x400fd0

400fcb:    b8 00 00 00 00          mov    $0x0,%eax             eax=0
400fd0:    2d b9 02 00 00          sub    $0x2b9,%eax           eax-0x2b9
400fd5:    eb 05                   jmp    400fdc <phase_3+0x81>  跳0x400fdc

400fd7:    b8 00 00 00 00          mov    $0x0,%eax           eax=0
400fdc:    05 b9 02 00 00          add    $0x2b9,%eax           eax+0x2b9
400fe1:    eb 05                   jmp    400fe8 <phase_3+0x8d>  跳0x400fe8

400fe3:    b8 00 00 00 00          mov    $0x0,%eax           eax=0
400fe8:    2d b9 02 00 00          sub    $0x2b9,%eax          eax-0x2b9
400fed:    eb 0a                   jmp    400ff9 <phase_3+0x9e> ; 跳0x400ff9
```
注意到这堆case都没有break，为了方便计算先试试第一个数填7，从`400fe3`开始跑，`%eax` = 0-0x2b9 = -697，然后跳到`400ff9`
```
400ff9:    83 7c 24 0c 05          cmpl   $0x5,0xc(%rsp)       比较第一个数和5
400ffe:    7f 06                   jg     401006 <phase_3+0xab>  如果大于则跳0x401006 boom
```
然后就是比较比较`%eax`是否和第二个数相等
```
401000:    3b 44 24 08             cmp    0x8(%rsp),%eax 
401004:    74 05                   je     40100b <phase_3+0xb0>  相等则accept
```
在`400ff9`发现第一个数还不能大于5，那说明填7不行，所以尝试填5，从`400fcb`开始，`%eax` = 0-0x2b9+0x2b9-0x2b9 = -697
所以phase3的key可以是 5 -697，用gdb调试通过了，当然第一个数不同就会有不同的解，我懒得一个个算了

 ### 思考：
 switch语句根据跳转表跳转在练习里出现过，简简单单
 
## phase 4
### 汇编代码：
```
0000000000401010 <func4>:
  401010:       41 54                   push   %r12
  401012:       55                      push   %rbp
  401013:       53                      push   %rbx
  401014:       89 fb                   mov    %edi,%ebx
  401016:       85 ff                   test   %edi,%edi
  401018:       7e 24                   jle    40103e <func4+0x2e>
  40101a:       89 f5                   mov    %esi,%ebp
  40101c:       89 f0                   mov    %esi,%eax
  40101e:       83 ff 01                cmp    $0x1,%edi
  401021:       74 20                   je     401043 <func4+0x33>
  401023:       8d 7f ff                lea    -0x1(%rdi),%edi
  401026:       e8 e5 ff ff ff          callq  401010 <func4>
  40102b:       44 8d 24 28             lea    (%rax,%rbp,1),%r12d
  40102f:       8d 7b fe                lea    -0x2(%rbx),%edi
  401032:       89 ee                   mov    %ebp,%esi
  401034:       e8 d7 ff ff ff          callq  401010 <func4>
  401039:       44 01 e0                add    %r12d,%eax
  40103c:       eb 05                   jmp    401043 <func4+0x33>
  40103e:       b8 00 00 00 00          mov    $0x0,%eax
  401043:       5b                      pop    %rbx
  401044:       5d                      pop    %rbp
  401045:       41 5c                   pop    %r12
  401047:       c3                      retq

0000000000401048 <phase_4>:
  401048:       48 83 ec 18             sub    $0x18,%rsp
  40104c:       48 8d 4c 24 0c          lea    0xc(%rsp),%rcx
  401051:       48 8d 54 24 08          lea    0x8(%rsp),%rdx
  401056:       be 01 28 40 00          mov    $0x402801,%esi
  40105b:       b8 00 00 00 00          mov    $0x0,%eax
  401060:       e8 cb fb ff ff          callq  400c30 <__isoc99_sscanf@plt>
  401065:       83 f8 02                cmp    $0x2,%eax
  401068:       75 0c                   jne    401076 <phase_4+0x2e>
  40106a:       8b 44 24 0c             mov    0xc(%rsp),%eax
  40106e:       83 e8 02                sub    $0x2,%eax
  401071:       83 f8 02                cmp    $0x2,%eax
  401074:       76 05                   jbe    40107b <phase_4+0x33>
  401076:       e8 5e 05 00 00          callq  4015d9 <explode_bomb>
  40107b:       8b 74 24 0c             mov    0xc(%rsp),%esi
  40107f:       bf 05 00 00 00          mov    $0x5,%edi
  401084:       e8 87 ff ff ff          callq  401010 <func4>
  401089:       3b 44 24 08             cmp    0x8(%rsp),%eax
  40108d:       74 05                   je     401094 <phase_4+0x4c>
  40108f:       e8 45 05 00 00          callq  4015d9 <explode_bomb>
  401094:       48 83 c4 18             add    $0x18,%rsp
  401098:       c3                      retq
```
### 思路：
看到在phase_4前面有一个func4，再看到phase_4有调用，就知道这个phase应该是关于函数调用的
```
 401056:       be 01 28 40 00          mov    $0x402801,%esi      
 40105b:       b8 00 00 00 00          mov    $0x0,%eax           
 401060:       e8 cb fb ff ff          callq  400c30 <__isoc99_sscanf@plt> 
 401065:       83 f8 02                cmp    $0x2,%eax          返回值是否为2
 401068:       75 0c                   jne    401076 <phase_4+0x2e> 不是则爆炸
```
`0x402801`这个地址的东西在phase3看过了，可知也是读入两个数 `0x8(%rsp)`和`0xc(%rsp)`（熟悉）
```
Halfway there!
1 2

Breakpoint 2, 0x0000000000401089 in phase_4 ()
(gdb) x /16xb $rsp
0x7fffffffe760:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x7fffffffe768:	0x01	0x00	0x00	0x00	0x02	0x00	0x00	0x00
```
看看位置，第一个数是 `0x8(%rsp)`，第二个是`0xc(%rsp)`
```
 40106a:       8b 44 24 0c             mov    0xc(%rsp),%eax       eax=第二个数
 40106e:       83 e8 02                sub    $0x2,%eax           eax-2
 401071:       83 f8 02                cmp    $0x2,%eax            比较eax和2
 401074:       76 05                   jbe    40107b <phase_4+0x33>  <=2跳转, 否则boom
 401076:       e8 5e 05 00 00          callq  4015d9 <explode_bomb>  
 40107b:       8b 74 24 0c             mov    0xc(%rsp),%esi       esi=第二个数
 40107f:       bf 05 00 00 00          mov    $0x5,%edi            edi=5
 401084:       e8 87 ff ff ff          callq  401010 <func4>       调用 func4
 401089:       3b 44 24 08            cmp    0x8(%rsp),%eax       
 40108d:       74 05                   je     401094 <phase_4+0x4c>  
 40108f:       e8 45 05 00 00          callq  4015d9 <explode_bomb>
 ```
 由此可得第二个数要<=4（不知道为什么，减了2以后小于零也是不行的，所以>=2 maybe），然后把和5第二个数作为参数传入func4（func4接受`%edi`，`%esi`两个参数），即执行 `func4(5,D2)`
 然后比较第一个数D1和func4的返回值（存在`%eax`里），相等就accept否则炸弹爆炸
 
 所以，目标是求出一组D1D2使得`D1=func4(5,D2)`
 下面分析func4，一个递归函数，先翻译大概的伪代码
```
0000000000401010 <func4>:

func(edi,esi)

  401010:       41 54                   push   %r12
  401012:       55                      push   %rbp
  401013:       53                      push   %rbx

  401014:       89 fb                   mov    %edi,%ebx

ebx=edi

  401016:       85 ff                   test   %edi,%edi
  401018:       7e 24                   jle    40103e <func4+0x2e>

if edi<=0,return 0

  40101a:       89 f5                   mov    %esi,%ebp
  40101c:       89 f0                   mov    %esi,%eax

ebp=esi
eax=esi

  40101e:       83 ff 01                cmp    $0x1,%edi
  401021:       74 20                   je     401043 <func4+0x33>

if edi=1,return esi 

  401023:       8d 7f ff                lea    -0x1(%rdi),%edi
  401026:       e8 e5 ff ff ff          callq  401010 <func4>

edi--
func(edi,esi)

  40102b:       44 8d 24 28             lea    (%rax,%rbp,1),%r12d

r12d=eax+ebp

  40102f:       8d 7b fe                lea    -0x2(%rbx),%edi

edi=ebx-2

  401032:       89 ee                   mov    %ebp,%esi

esi=ebp

  401034:       e8 d7 ff ff ff          callq  401010 <func4>

func(edi,esi)

  401039:       44 01 e0                add    %r12d,%eax

eax+=r12d

  40103c:       eb 05                   jmp    401043 <func4+0x33>

return eax

  40103e:       b8 00 00 00 00          mov    $0x0,%eax

eax=0

  401043:       5b                      pop    %rbx
  401044:       5d                      pop    %rbp
  401045:       41 5c                   pop    %r12
  401047:       c3                      retq
 ```
 整理一下的得到的c代码就是（本人逆向工程水平太低，连蒙带猜）
```c
int func4(int x, int y) {
   if (x == 0) return 0;
   if (x == 1) return y;
   return func4(x - 1, y) + func4(x - 2, y) + y;
}
```
在ide里跑了一下，猜测一组可能的key是 24 2（一开始12 1 过不了）运行就通过了这个phase

**第二种方法：**
有点取巧，这是正常推导过了以后才想到的，其实没必要理解func4的原理，可以让bomb帮我们算出来
在phase_4最后比较func4返回值和D1是否相等的地方打一个断点，即
`0x401089`的cmp语句处
```
(gdb) break *0x401089
Breakpoint 2 at 0x401089
```
 输入随便一个D1和一个合法的D2 ,等它在第二个断点停下，这时已经跑过了func4，也就是说`%eax`里面存的值就是我们想要的
打印`%eax`
```
Halfway there!
1 2  

Breakpoint 2, 0x0000000000401089 in phase_4 ()
(gdb) print $eax
$1 = 24
```
可见 24 2是一对可行的key

### 思考：
这个phase卡了挺久，主要是逆向能力差，一开始推出来的函数错了，在翻译递归函数碰到很多困难，主要在于换来换去的临时寄存器资源
读入两个数，和phase3一样，但是为什么顺序却是反的（打印了栈里的内容才发现的）

## phase 5
### 汇编代码：
```
0000000000401099 <phase_5>:
  401099:       53                      push   %rbx
  40109a:       48 83 ec 10             sub    $0x10,%rsp
  40109e:       48 89 fb                mov    %rdi,%rbx
  4010a1:       e8 6b 02 00 00          callq  401311 <string_length>
  4010a6:       83 f8 06                cmp    $0x6,%eax
  4010a9:       74 3f                   je     4010ea <phase_5+0x51>
  4010ab:       e8 29 05 00 00          callq  4015d9 <explode_bomb>
  4010b0:       eb 38                   jmp    4010ea <phase_5+0x51>
  4010b2:       0f b6 14 03             movzbl (%rbx,%rax,1),%edx
  4010b6:       83 e2 0f                and    $0xf,%edx
  4010b9:       0f b6 92 d0 25 40 00    movzbl 0x4025d0(%rdx),%edx
  4010c0:       88 14 04                mov    %dl,(%rsp,%rax,1)
  4010c3:       48 83 c0 01             add    $0x1,%rax
  4010c7:       48 83 f8 06             cmp    $0x6,%rax
  4010cb:       75 e5                   jne    4010b2 <phase_5+0x19>
  4010cd:       c6 44 24 06 00          movb   $0x0,0x6(%rsp)
  4010d2:       be 86 25 40 00          mov    $0x402586,%esi
  4010d7:       48 89 e7                mov    %rsp,%rdi
  4010da:       e8 4f 02 00 00          callq  40132e <strings_not_equal>
  4010df:       85 c0                   test   %eax,%eax
  4010e1:       74 0f                   je     4010f2 <phase_5+0x59>
  4010e3:       e8 f1 04 00 00          callq  4015d9 <explode_bomb>
  4010e8:       eb 08                   jmp    4010f2 <phase_5+0x59>
  4010ea:       b8 00 00 00 00          mov    $0x0,%eax
  4010ef:       90                      nop
  4010f0:       eb c0                   jmp    4010b2 <phase_5+0x19>
  4010f2:       48 83 c4 10             add    $0x10,%rsp
  4010f6:       5b                      pop    %rbx
  4010f7:       c3                      retq
```
### 思路：
```
  401099:       53                      push   %rbx
  40109a:       48 83 ec 10             sub    $0x10,%rsp
  40109e:       48 89 fb                mov    %rdi,%rbx rbx=输入的字符串
  4010a1:       e8 6b 02 00 00          callq  401311 <string_length>
  4010a6:       83 f8 06                cmp    $0x6,%eax
  4010a9:       74 3f                   je     4010ea <phase_5+0x51> 长度是6
  4010ab:       e8 29 05 00 00          callq  4015d9 <explode_bomb>
  4010b0:       eb 38                   jmp    4010ea <phase_5+0x51>
 ```
 phase 5是读入一个字符串，长度为6，再继续看下去
 ```
  4010ea:       b8 00 00 00 00          mov    $0x0,%eax
  4010ef:       90                      nop
  4010f0:       eb c0                   jmp    4010b2 <phase_5+0x19>
 ```
 这里把eax置零，然后跳到前面的 `4010b2`
 ```
  4010b2:       0f b6 14 03             movzbl (%rbx,%rax,1),%edx edx=rbx+rax,进行了扩展
  4010b6:       83 e2 0f                and    $0xf,%edx  取低四位
  4010b9:       0f b6 92 d0 25 40 00    movzbl 0x4025d0(%rdx),%edx
  4010c0:       88 14 04                mov    %dl,(%rsp,%rax,1)
  4010c3:       48 83 c0 01             add    $0x1,%rax
  4010c7:       48 83 f8 06             cmp    $0x6,%rax
  4010cb:       75 e5                   jne    4010b2 <phase_5+0x19>
 ```
 是一个循环，对字符串进行了一些看起来很混乱的操作，下面慢慢分析一下
 先看循环退出条件
 ```
  4010c7:       48 83 f8 06             cmp    $0x6,%rax
  4010cb:       75 e5                   jne    4010b2 
 ```
 退出条件是`%rax`里面的值等于6，`%rax`已经在`4010ea`置为0了，看字符串长度也是6，可以猜想rax是一个游标，目的是对字符串的每一位操作，看回循环里
 ```
  4010b2:       0f b6 14 03             movzbl (%rbx,%rax,1),%edx edx=rbx+rax,进行了扩展
  4010b6:       83 e2 0f                and    $0xf,%edx  取低四位
```
还看到`4010c3`有rax++，是遍历没错

`%rdx`是`%rbx`对应的地址的第`%rax`个字节，与`0xf`进行位与运算，即取`%rdx`的低四位，`%rdx`的值从0～15共16种可能
（好像字符串hash啊）

继续看下去
```
  4010b9:       0f b6 92 d0 25 40 00    movzbl 0x4025d0(%rdx),%edx
  4010c0:       88 14 04                mov    %dl,(%rsp,%rax,1)
  ```
  在`4010b9`处，把`0x4025d0`加上`%rdx`得到的地址里面的值放回`%rdx`
  `4010c0`处是把得到的新值进栈
  （%dl是%rdx的低8位，为什么要用它而不是%rdx或者%edx呢我也不知道）
  
 所以，这个过程就是把原字符串加密，对应表是`0x4025d0`后面16位范围里的值，用gdb看看
```
(gdb) x /s 0x4025d0
0x4025d0 <array.3159>:	"maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?"
```
**maduiersnfotvbyl**对应着16个hash后的数字，下面可以印一张对应表
```c
char blo [17] = "maduiersnfotvbyl";
int main() {
    for (int i='a';i<'a'+26;i++) {
        int hashc = i & 0xf;
        cout<<(char)i<<"-"<<blo[hashc]<<endl;
    }
}
```
```
a-a b-d c-u d-i e-e f-r g-s h-n i-f
j-o k-t l-v m-b n-y o-l p-m q-a r-d
s-u t-i u-e v-r w-s x-n y-f z-o
```
下面的代码就是匹配了
```
  4010cd:       c6 44 24 06 00          movb   $0x0,0x6(%rsp)
  4010d2:       be 86 25 40 00          mov    $0x402586,%esi
  4010d7:       48 89 e7                mov    %rsp,%rdi
  4010da:       e8 4f 02 00 00          callq  40132e <strings_not_equal>
  4010df:       85 c0                   test   %eax,%eax
  4010e1:       74 0f                   je     4010f2 <phase_5+0x59>
  4010e3:       e8 f1 04 00 00          callq  4015d9 <explode_bomb>
  4010e8:       eb 08                   jmp    4010f2 <phase_5+0x59>
```
 `4010cd`先在加密后字符串后面加0，用phase1里面出现过的`strings_not_equal`函数，和位于`0x402586`的字符串匹配
 
破解方法就出来了：只要把`402586`处的答案字符串按照表格倒推就能求出key，用gdb看看`402586`处有什么
```
(gdb) x /s 0x402586
0x402586:	"bruins"
```
查表得到对应的key为 mfcdhg，通过了
当然也会有其它答案，因为一个hash值可以对应多个字母

### 思考：
难度骤降，因为这个加密太明显，更复杂的加密就没办法通过直接推密文的方法破解了

## phase 6
### 汇编代码：
```
00000000004010f8 <phase_6>:
  4010f8:       41 56                   push   %r14
  4010fa:       41 55                   push   %r13
  4010fc:       41 54                   push   %r12
  4010fe:       55                      push   %rbp
  4010ff:       53                      push   %rbx
  401100:       48 83 ec 50             sub    $0x50,%rsp
  401104:       4c 8d 6c 24 30          lea    0x30(%rsp),%r13
  401109:       4c 89 ee                mov    %r13,%rsi
  40110c:       e8 fe 04 00 00          callq  40160f <read_six_numbers>
  401111:       4d 89 ee                mov    %r13,%r14
  401114:       41 bc 00 00 00 00       mov    $0x0,%r12d
  40111a:       4c 89 ed                mov    %r13,%rbp
  40111d:       41 8b 45 00             mov    0x0(%r13),%eax
  401121:       83 e8 01                sub    $0x1,%eax
  401124:       83 f8 05                cmp    $0x5,%eax
  401127:       76 05                   jbe    40112e <phase_6+0x36>
  401129:       e8 ab 04 00 00          callq  4015d9 <explode_bomb>
  40112e:       41 83 c4 01             add    $0x1,%r12d
  401132:       41 83 fc 06             cmp    $0x6,%r12d
  401136:       74 22                   je     40115a <phase_6+0x62>
  401138:       44 89 e3                mov    %r12d,%ebx
  40113b:       48 63 c3                movslq %ebx,%rax
  40113e:       8b 44 84 30             mov    0x30(%rsp,%rax,4),%eax
  401142:       39 45 00                cmp    %eax,0x0(%rbp)
  401145:       75 05                   jne    40114c <phase_6+0x54>
  401147:       e8 8d 04 00 00          callq  4015d9 <explode_bomb>
  40114c:       83 c3 01                add    $0x1,%ebx
  40114f:       83 fb 05                cmp    $0x5,%ebx
  401152:       7e e7                   jle    40113b <phase_6+0x43>
  401154:       49 83 c5 04             add    $0x4,%r13
  401158:       eb c0                   jmp    40111a <phase_6+0x22>
  40115a:       48 8d 74 24 48          lea    0x48(%rsp),%rsi
  40115f:       4c 89 f0                mov    %r14,%rax
  401162:       b9 07 00 00 00          mov    $0x7,%ecx
  401167:       89 ca                   mov    %ecx,%edx
  401169:       2b 10                   sub    (%rax),%edx
  40116b:       89 10                   mov    %edx,(%rax)
  40116d:       48 83 c0 04             add    $0x4,%rax
  401171:       48 39 f0                cmp    %rsi,%rax
  401174:       75 f1                   jne    401167 <phase_6+0x6f>
  401176:       be 00 00 00 00          mov    $0x0,%esi
  40117b:       eb 20                   jmp    40119d <phase_6+0xa5>
  40117d:       48 8b 52 08             mov    0x8(%rdx),%rdx
  401181:       83 c0 01                add    $0x1,%eax
  401184:       39 c8                   cmp    %ecx,%eax
  401186:       75 f5                   jne    40117d <phase_6+0x85>
  401188:       eb 05                   jmp    40118f <phase_6+0x97>
  40118a:       ba f0 42 60 00          mov    $0x6042f0,%edx
  40118f:       48 89 14 74             mov    %rdx,(%rsp,%rsi,2)
  401193:       48 83 c6 04             add    $0x4,%rsi
  401197:       48 83 fe 18             cmp    $0x18,%rsi
  40119b:       74 15                   je     4011b2 <phase_6+0xba>
  40119d:       8b 4c 34 30             mov    0x30(%rsp,%rsi,1),%ecx
  4011a1:       83 f9 01                cmp    $0x1,%ecx
  4011a4:       7e e4                   jle    40118a <phase_6+0x92>
  4011a6:       b8 01 00 00 00          mov    $0x1,%eax
  4011ab:       ba f0 42 60 00          mov    $0x6042f0,%edx
  4011b0:       eb cb                   jmp    40117d <phase_6+0x85>
  4011b2:       48 8b 1c 24             mov    (%rsp),%rbx
  4011b6:       48 8d 44 24 08          lea    0x8(%rsp),%rax
  4011bb:       48 8d 74 24 30          lea    0x30(%rsp),%rsi
  4011c0:       48 89 d9                mov    %rbx,%rcx
  4011c3:       48 8b 10                mov    (%rax),%rdx
  4011c6:       48 89 51 08             mov    %rdx,0x8(%rcx)
  4011ca:       48 83 c0 08             add    $0x8,%rax
  4011ce:       48 39 f0                cmp    %rsi,%rax
  4011d1:       74 05                   je     4011d8 <phase_6+0xe0>
  4011d3:       48 89 d1                mov    %rdx,%rcx
  4011d6:       eb eb                   jmp    4011c3 <phase_6+0xcb>
  4011d8:       48 c7 42 08 00 00 00    movq   $0x0,0x8(%rdx)
  4011df:       00
  4011e0:       bd 05 00 00 00          mov    $0x5,%ebp
  4011e5:       48 8b 43 08             mov    0x8(%rbx),%rax
  4011e9:       8b 00                   mov    (%rax),%eax
  4011eb:       39 03                   cmp    %eax,(%rbx)
  4011ed:       7d 05                   jge    4011f4 <phase_6+0xfc>
  4011ef:       e8 e5 03 00 00          callq  4015d9 <explode_bomb>
  4011f4:       48 8b 5b 08             mov    0x8(%rbx),%rbx
  4011f8:       83 ed 01                sub    $0x1,%ebp
  4011f4:       48 8b 5b 08             mov    0x8(%rbx),%rbx
  4011f8:       83 ed 01                sub    $0x1,%ebp
  4011fb:       75 e8                   jne    4011e5 <phase_6+0xed>
  4011fd:       48 83 c4 50             add    $0x50,%rsp
  401201:       5b                      pop    %rbx
  401202:       5d                      pop    %rbp
  401203:       41 5c                   pop    %r12
  401205:       41 5d                   pop    %r13
  401207:       41 5e                   pop    %r14
  401209:       c3                      retq
```
### 思路：
phase6的汇编代码太长了，一段段看
```
  401100:       48 83 ec 50             sub    $0x50,%rsp
  401104:       4c 8d 6c 24 30          lea    0x30(%rsp),%r13
  401109:       4c 89 ee                mov    %r13,%rsi
  401111:       4d 89 ee                mov    %r13,%r14
```
`0x30(%rsp)`是我们输入数据的起始地址
这里是在开辟栈空间，将输入数据起始地址赋给`%rsi`，`%r13`和`%r14`
 ```
  40110c:       e8 fe 04 00 00          callq  40160f<read_six_numbers> 
```
然后把`%rsi`作为参数传给`<read_six_numbers>`

 下面这堆东西是一个循环
```
  401114:       41 bc 00 00 00 00       mov    $0x0,%r12d

  40111a:       4c 89 ed                mov    %r13,%rbp
  40111d:       41 8b 45 00             mov    0x0(%r13),%eax
  401121:       83 e8 01                sub    $0x1,%eax
  401124:       83 f8 05                cmp    $0x5,%eax
  401127:       76 05                   jbe    40112e <phase_6+0x36>
  401129:       e8 ab 04 00 00          callq  4015d9 <explode_bomb>
  
  40112e:       41 83 c4 01             add    $0x1,%r12d
  401132:       41 83 fc 06             cmp    $0x6,%r12d
  401136:       74 22                   je     40115a <phase_6+0x62>
  
  401138:       44 89 e3                mov    %r12d,%ebx
  40113b:       48 63 c3                movslq %ebx,%rax
  40113e:       8b 44 84 30             mov    0x30(%rsp,%rax,4),%eax
  401142:       39 45 00                cmp    %eax,0x0(%rbp)
  401145:       75 05                   jne    40114c <phase_6+0x54>
  401147:       e8 8d 04 00 00          callq  4015d9 <explode_bomb>
  
  40114c:       83 c3 01                add    $0x1,%ebx
  40114f:       83 fb 05                cmp    $0x5,%ebx
  401152:       7e e7                   jle    40113b <phase_6+0x43>
  
  401154:       49 83 c5 04             add    $0x4,%r13
  401158:       eb c0                   jmp    40111a <phase_6+0x22>
```
详细分析一下这个循环：

看开头，在`401114`处，熟悉的置零又出现了，被置零，猜想`%r12d`是游标，在` 40112e`果然就找到了对`%r12d`++

但同时我们看到`40114c`也有一个add指令，对`%ebx`++，如果它也是游标那可能是两层循环，找找它的 Init stmt 在哪：`401138`处，令`%ebx`等于`%r12d`，可见`%ebx`是内层循环游标，`%r12d`是外层循环的游标

再找 Loop cond ，一般就在add后面

对于`%r12d`，等于6的时候退出，为<6
```
  401132:       41 83 fc 06             cmp    $0x6,%r12d
  401136:       74 22                   je     40115a <phase_6+0x62>
 ```
 
 对于`%ebx`，小于等于5的时候继续循环，也为 <6
 ```
  40114f:       83 fb 05                cmp    $0x5,%ebx
  401152:       7e e7                   jle    40113b <phase_6+0x43>
 ```
注意到`401138`处给`%ebx`赋值已经是++后的了，现在可以写出双层循环的框架:
```c
/%r12d i,%ebx j/
for(int i=0;i<6;i++){
	for(int j=i+1;j<6;j++){
	
	}
}
```
看看游标是用来调用什么的
```
  40113b:       48 63 c3                movslq %ebx,%rax
  40113e:       8b 44 84 30             mov    0x30(%rsp,%rax,4),%eax
```
这几句读取 `0x30 + %rsp + %rax * 4` 处的数据，赋值给 `%eax`（`%rax`就是`%ebx`，做了一个扩展），`0x30 + %rsp`是读入六个数字的开头，所以%eax是一个调用游标

这个时候还是没找到`r12d`参与的调用，但注意到大循环内的结尾（即每结束一个小循环）会把`%r13`+=4（`401154`处），在大循环的开头会把`%r13`赋给`%rbp`，考虑真正的调用游标是`%r13`（`%rbp`），`r12d`只是用来计次数的（不过我们可以把二者合并成一个游标）

翻译两个炸弹爆炸的if语句，就可以还原整个循环了
```c
int a[6]; // 我们输入的数组
for(int i=0;i<6;i++){
	if(a[i]<1||a[i]>6)explode_bomb();
	for (int j=i+1;j<6;j++){
		if (a[i]==a[j])explode_bomb();
	}
}
```
可知我们输入的六个数一定是1到6这六个（调试也得此结论正确）

再往下看
```
  40115a:       48 8d 74 24 48          lea    0x48(%rsp),%rsi
  40115f:       4c 89 f0                mov    %r14,%rax
  401162:       b9 07 00 00 00          mov    $0x7,%ecx
  
  401167:       89 ca                   mov    %ecx,%edx
  401169:       2b 10                   sub    (%rax),%edx
  40116b:       89 10                   mov    %edx,(%rax)
  40116d:       48 83 c0 04             add    $0x4,%rax
  
  401171:       48 39 f0                cmp    %rsi,%rax
  401174:       75 f1                   jne    401167 <phase_6+0x6f>
  ```
又发现一个循环，不过这是一个简单的单层循环，就直接读好了：
把输入数据起始地址赋给`%rax`做游标，每次循环+4移到下一个数 ，移到末尾结束
`%ecx`=7，在`401169`处，计算`7-%rax`的值，存在`%edx`里，再把`%edx`赋回`%rax`，即 x=7-x

所以这个循环的作用就是把整个数组加个密

然后
```
  401176:       be 00 00 00 00          mov    $0x0,%esi
  40117b:       eb 20                   jmp    40119d <phase_6+0xa5>
```
把`%esi`置零（又是一个游标），跳到后面一堆代码中间(`40119d`)
接下来的部分就很复杂了，有很多个跳转和条件跳转

首先，注意到
```
 40117d:       48 8b 52 08             mov    0x8(%rdx),%rdx
```
`%rdx`自增8，以及
```
  40118a:       ba f0 42 60 00          mov    $0x6042f0,%edx
```
可以知道， `%rdx`存了一个地址，应该是一个指针，而且这个地址属于全局地址（60开头），指向的东西大小是8Byte，用gdb看看这里面有什么
```
(gdb) x /96xb 0x6042f0
0x6042f0 <node1>:	0xa2	0x01	0x00	0x00	0x01	0x00	0x00	0x00
0x6042f8 <node1+8>:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x604300 <node2>:	0x8b	0x03	0x00	0x00	0x02	0x00	0x00	0x00
0x604308 <node2+8>:	0xf0	0x42	0x60	0x00	0x00	0x00	0x00	0x00
0x604310 <node3>:	0x99	0x01	0x00	0x00	0x03	0x00	0x00	0x00
0x604318 <node3+8>:	0x00	0x43	0x60	0x00	0x00	0x00	0x00	0x00
0x604320 <node4>:	0x13	0x01	0x00	0x00	0x04	0x00	0x00	0x00
0x604328 <node4+8>:	0x10	0x43	0x60	0x00	0x00	0x00	0x00	0x00
0x604330 <node5>:	0x5b	0x02	0x00	0x00	0x05	0x00	0x00	0x00
0x604338 <node5+8>:	0x20	0x43	0x60	0x00	0x00	0x00	0x00	0x00
0x604340 <node6>:	0x84	0x00	0x00	0x00	0x06	0x00	0x00	0x00
0x604348 <node6+8>:	0x30	0x43	0x60	0x00	0x00	0x00	0x00    0x00
```
尝试了几次8的倍数，把有名字的结构都打印出来了，看到命名是node，猜想是某种链式结构的节点， 每个node有16Byte也就是两个Word，换个单位再打印试试
```
(gdb) x /24w 0x6042f0
0x6042f0 <node1>:	0x000001a2	0x00000001	0x00000000	0x00000000
0x604300 <node2>:	0x0000038b	0x00000002	0x006042f0	0x00000000
0x604310 <node3>:	0x00000199	0x00000003	0x00604300	0x00000000
0x604320 <node4>:	0x00000113	0x00000004	0x00604310	0x00000000
0x604330 <node5>:	0x0000025b	0x00000005	0x00604320	0x00000000
0x604340 <node6>:	0x00000084	0x00000006	0x00604330	0x00000000
```
现在能看出来这是一个单链表了

接着看代码，我们输入的数究竟有什么用处？
之前说到把输入的数加密以后就到了`40119d`
 ```
  40117d:       48 8b 52 08             mov    0x8(%rdx),%rdx
  401181:       83 c0 01                add    $0x1,%eax
  401184:       39 c8                   cmp    %ecx,%eax
  401186:       75 f5                   jne    40117d <phase_6+0x85>
  401188:       eb 05                   jmp    40118f <phase_6+0x97>
  
  40118a:       ba f0 42 60 00          mov    $0x6042f0,%edx
  
  40118f:       48 89 14 74             mov    %rdx,(%rsp,%rsi,2)
  401193:       48 83 c6 04             add    $0x4,%rsi
  401197:       48 83 fe 18             cmp    $0x18,%rsi
  40119b:       74 15                   je     4011b2 <phase_6+0xba>
  
  40119d:       8b 4c 34 30             mov    0x30(%rsp,%rsi,1),%ecx
  4011a1:       83 f9 01                cmp    $0x1,%ecx
  4011a4:       7e e4                   jle    40118a <phase_6+0x92>
  4011a6:       b8 01 00 00 00          mov    $0x1,%eax
  4011ab:       ba f0 42 60 00          mov    $0x6042f0,%edx
  4011b0:       eb cb                   jmp    40117d <phase_6+0x85>
 ```
 这也是一个双层循环
`%rsi`在`401176`置零，是大循环的游标，`0x30(%rsp,%rsi,1)`就是数组第一个数
设为x，把它存进 `%ecx`，和1比较，如果不是1就进内层循环

小循环的游标是`%eax`，一开始是1，每次自增1，出口在`401184`，所以Loop cond是<=x
首先把指向第一个node的指针赋给%rdx，然后开始进行沿链表进行单向移动，每次小循环移动一步，直至到达第x个node，然后将这个node的指针存到从栈顶开始的一片空间

 在存指针的地方（`40118f`到`40119b`），我们也看到了大循环的出口：
 大循环游标`%rsi`每次自增4，把第x个node的指针存到`%rsp+%rsi*2`的地方，六个node全部存进去的时候大循环结束

下面举一个例子说明推出来的代码，假设内存中的链表节点序号如图
```
 1->2->3->4->5->6
```
输入的数是 5 3 1 6 4 2 ，加密的数组是 2 4 6 1 3 5
第一个数2，指向链表第2个节点的指针进栈
第二个数4，指向链表第4个节点的指针进栈
...
最后的栈如图
```
 rsp+0    2
 rsp+8    4
 rsp+16   6
 rsp+24   1
 rsp+32   3
 rsp+40   5
```

下一段就是按照栈中的顺序，把链表连起来，一个循环直接读
```
  4011b2:       48 8b 1c 24             mov    (%rsp),%rbx
  4011b6:       48 8d 44 24 08          lea    0x8(%rsp),%rax
  4011bb:       48 8d 74 24 30          lea    0x30(%rsp),%rsi
  4011c0:       48 89 d9                mov    %rbx,%rcx
  4011c3:       48 8b 10                mov    (%rax),%rdx
  4011c6:       48 89 51 08             mov    %rdx,0x8(%rcx)
  4011ca:       48 83 c0 08             add    $0x8,%rax
  4011ce:       48 39 f0                cmp    %rsi,%rax
  4011d1:       74 05                   je     4011d8 <phase_6+0xe0>
  4011d3:       48 89 d1                mov    %rdx,%rcx
  4011d6:       eb eb                   jmp    4011c3 <phase_6+0xcb>
  4011d8:       48 c7 42 08 00 00 00    movq   $0x0,0x8(%rdx)
```
然后又是一个循环
```
  4011e0:       bd 05 00 00 00          mov    $0x5,%ebp
  4011e5:       48 8b 43 08             mov    0x8(%rbx),%rax
  4011e9:       8b 00                   mov    (%rax),%eax
  4011eb:       39 03                   cmp    %eax,(%rbx)
  4011ed:       7d 05                   jge    4011f4 <phase_6+0xfc>
  4011ef:       e8 e5 03 00 00          callq  4015d9 <explode_bomb>
  4011f4:       48 8b 5b 08             mov    0x8(%rbx),%rbx
  4011f8:       83 ed 01                sub    $0x1,%ebp
  4011f4:       48 8b 5b 08             mov    0x8(%rbx),%rbx
  4011f8:       83 ed 01                sub    $0x1,%ebp
  4011fb:       75 e8                   jne    4011e5 <phase_6+0xed>
  ```
  循环遍历链表，`%rbx`是当前节点，`4011e5`和`4011e9`把当前节点的下一个节点的值存进`%eax`
  `4011eb`比较`%rbx`指向的节点的值与`%eax`的值，大于等于的时候就跳过炸弹，否则爆炸
  
  所以到这一步的链表节点的值必须按照从大到小的顺序排列，可以倒推答案了
 把`0x6042f0` 里链表节点按值排列得到`2->5->1->3->4->6`
还原，得到key为 5 2 6 4 3 1

### 思考：
phase6很难，但没有想象中难，更多是考验综合能力，必须逆向工程还原出接近完整的代码逻辑
巩固了读循环语句的能力，两重循坏一堆游标跳来跳去也很考验耐心

## Secret_Phase
做完phase6发现才95分，剩下5分在Secret_Phase里
在用objdump反汇编整个bomb时，注意到phase6后面还有两段代码
```
000000000040120a <fun7>:
  40120a:       48 83 ec 08             sub    $0x8,%rsp
  40120e:       48 85 ff                test   %rdi,%rdi
  401211:       74 2b                   je     40123e <fun7+0x34>
  401213:       8b 17                   mov    (%rdi),%edx
  401215:       39 f2                   cmp    %esi,%edx
  401217:       7e 0d                   jle    401226 <fun7+0x1c>
  401219:       48 8b 7f 08             mov    0x8(%rdi),%rdi
  40121d:       e8 e8 ff ff ff          callq  40120a <fun7>
  401222:       01 c0                   add    %eax,%eax
  401224:       eb 1d                   jmp    401243 <fun7+0x39>
  401226:       b8 00 00 00 00          mov    $0x0,%eax
  40122b:       39 f2                   cmp    %esi,%edx
  40122d:       74 14                   je     401243 <fun7+0x39>
  40122f:       48 8b 7f 10             mov    0x10(%rdi),%rdi
  401233:       e8 d2 ff ff ff          callq  40120a <fun7>
  401238:       8d 44 00 01             lea    0x1(%rax,%rax,1),%eax
  40123c:       eb 05                   jmp    401243 <fun7+0x39>
  40123e:       b8 ff ff ff ff          mov    $0xffffffff,%eax
  401243:       48 83 c4 08             add    $0x8,%rsp
  401247:       c3                      retq

0000000000401248 <secret_phase>:
  401248:       53                      push   %rbx
  401249:       e8 03 04 00 00          callq  401651 <read_line>
  40124e:       ba 0a 00 00 00          mov    $0xa,%edx
  401253:       be 00 00 00 00          mov    $0x0,%esi
  401258:       48 89 c7                mov    %rax,%rdi
  40125b:       e8 a0 f9 ff ff          callq  400c00 <strtol@plt>
  401260:       48 89 c3                mov    %rax,%rbx
  401263:       8d 40 ff                lea    -0x1(%rax),%eax
  401266:       3d e8 03 00 00          cmp    $0x3e8,%eax
  40126b:       76 05                   jbe    401272 <secret_phase+0x2a>
  40126d:       e8 67 03 00 00          callq  4015d9 <explode_bomb>
  401272:       89 de                   mov    %ebx,%esi
  401274:       bf 10 41 60 00          mov    $0x604110,%edi
  401279:       e8 8c ff ff ff          callq  40120a <fun7>
  40127e:       83 f8 02                cmp    $0x2,%eax
  401281:       74 05                   je     401288 <secret_phase+0x40>
  401283:       e8 51 03 00 00          callq  4015d9 <explode_bomb>
  401281:       74 05                   je     401288 <secret_phase+0x40>
  401283:       e8 51 03 00 00          callq  4015d9 <explode_bomb>
  401288:       bf 60 25 40 00          mov    $0x402560,%edi
  40128d:       e8 ae f8 ff ff          callq  400b40 <puts@plt>
  401292:       e8 e0 04 00 00          callq  401777 <phase_defused>
  401297:       5b                      pop    %rbx
  401298:       c3                      retq
  401299:       0f 1f 80 00 00 00 00    nopl   0x0(%rax)
```
### 思路：
我们现在拿到了Secret_Phase的代码，但是注意在 bomb.c里面是看不到直接的隐藏关调用的，还需要寻找一个入口
```c
(bomb.c)
	...
	input = read_line();
    phase_6(input);
    phase_defused();

    /* Wow, they got it!  But isn't something... missing?  Perhaps
     * something they overlooked?  Mua ha ha ha ha! */
```
我们在`phase_defused`上打个断点看看汇编
```
Congratulations! You've defused the bomb!
Your instructor has been notified and will verify your solution.
[Inferior 1 (process 492) exited normally]
(gdb) disas phase_defused
Dump of assembler code for function phase_defused:
   0x0000000000401777 <+0>:	sub    $0x68,%rsp
   0x000000000040177b <+4>:	mov    $0x1,%edi
   0x0000000000401780 <+9>:	callq  0x4014ba <send_msg>
   0x0000000000401785 <+14>:	cmpl   $0x6,0x203010(%rip)        # 0x60479c <num_input_strings>
   0x000000000040178c <+21>:	jne    0x4017fb <phase_defused+132>
   0x000000000040178e <+23>:	lea    0x10(%rsp),%r8
   0x0000000000401793 <+28>:	lea    0x8(%rsp),%rcx
   0x0000000000401798 <+33>:	lea    0xc(%rsp),%rdx
   0x000000000040179d <+38>:	mov    $0x40284b,%esi
   0x00000000004017a2 <+43>:	mov    $0x6048b0,%edi
   0x00000000004017a7 <+48>:	mov    $0x0,%eax
   0x00000000004017ac <+53>:	callq  0x400c30 <__isoc99_sscanf@plt>
   0x00000000004017b1 <+58>:	cmp    $0x3,%eax
   0x00000000004017b4 <+61>:	jne    0x4017e7 <phase_defused+112>
   0x00000000004017b6 <+63>:	mov    $0x402854,%esi
   0x00000000004017bb <+68>:	lea    0x10(%rsp),%rdi
   0x00000000004017c0 <+73>:	callq  0x40132e <strings_not_equal>
   0x00000000004017c5 <+78>:	test   %eax,%eax
   0x00000000004017c7 <+80>:	jne    0x4017e7 <phase_defused+112>
   0x00000000004017c9 <+82>:	mov    $0x4026a0,%edi
   0x00000000004017ce <+87>:	callq  0x400b40 <puts@plt>
   0x00000000004017d3 <+92>:	mov    $0x4026c8,%edi
   0x00000000004017d8 <+97>:	callq  0x400b40 <puts@plt>
   0x00000000004017dd <+102>:	mov    $0x0,%eax
   
   0x00000000004017e2 <+107>:	callq  0x401248 <secret_phase>
   
   0x00000000004017e7 <+112>:	mov    $0x402700,%edi
   0x00000000004017ec <+117>:	callq  0x400b40 <puts@plt>
   0x00000000004017f1 <+122>:	mov    $0x402730,%edi
   0x00000000004017f6 <+127>:	callq  0x400b40 <puts@plt>
   0x00000000004017fb <+132>:	add    $0x68,%rsp
   0x00000000004017ff <+136>:	retq   
End of assembler dump.
```
看 `4017e2`，终于找到了 （csapp极小概率事件，bgm播放）
```
  0x00000000004017e2 <+107>:	callq  0x401248 <secret_phase>
```
检查整个函数，发现没有任何一条跳转语句直接跳到`4017e2`和前一个地址，所以进入方法是满足在`4017e2`前不跳转
```
   0x0000000000401785 <+14>:	cmpl   $0x6,0x203010(%rip)        # 0x60479c <num_input_strings>
   0x000000000040178c <+21>:	jne    0x4017fb <phase_defused+132>
   ```
 由此可知在整个程序中必须已经读入六个字符串，即通过了前面的所有phase
```
   0x000000000040179d <+38>:	mov    $0x40284b,%esi
   0x00000000004017a2 <+43>:	mov    $0x6048b0,%edi
   0x00000000004017a7 <+48>:	mov    $0x0,%eax
   0x00000000004017ac <+53>:	callq  0x400c30 <__isoc99_sscanf@plt>
   0x00000000004017b1 <+58>:	cmp    $0x3,%eax
   0x00000000004017b4 <+61>:	jne    0x4017e7 <phase_defused+112>
```
由此可知需要读入三个参数

打印一下`0x40284b`里的格式串
```
(gdb) x /s 0x40284b
0x40284b:	"%d %d %s"
```
这个格式是我们在前面从来没有见过的，考虑前面有两个整型，应该是 phase3 或者 phase4
再看看这个字符串应该是什么东西
```
   0x00000000004017b6 <+63>:	mov    $0x402854,%esi
   0x00000000004017bb <+68>:	lea    0x10(%rsp),%rdi
   0x00000000004017c0 <+73>:	callq  0x40132e <strings_not_equal>
```
打印`0x402854`的值
```
(gdb) x /s 0x402854
0x402854:	"DrEvil"
```
在 secret_phase 里面某个地方比如 `401249`打断点，调试得到phase4答案后面加DrEvil进去了
```
Curses, you've found the secret phase!
But finding it and solving it are quite different...

Breakpoint 2, 0x0000000000401249 in secret_phase ()
```
现在看 secret_phase 代码
```
0000000000401248 <secret_phase>:
  401248:       53                      push   %rbx
  401249:       e8 03 04 00 00          callq  401651 <read_line>
  40124e:       ba 0a 00 00 00          mov    $0xa,%edx
  401253:       be 00 00 00 00          mov    $0x0,%esi
  401258:       48 89 c7                mov    %rax,%rdi
  40125b:       e8 a0 f9 ff ff          callq  400c00 <strtol@plt>
  401260:       48 89 c3                mov    %rax,%rbx
  401263:       8d 40 ff                lea    -0x1(%rax),%eax
  401266:       3d e8 03 00 00          cmp    $0x3e8,%eax
  40126b:       76 05                   jbe    401272 <secret_phase+0x2a>
  40126d:       e8 67 03 00 00          callq  4015d9 <explode_bomb>
```
要输入一个数字，strtol函数是将输入的字符串转换为long int，我们输入的数大小需要在1到0x3e9之间
```
  401272:       89 de                   mov    %ebx,%esi
  401274:       bf 10 41 60 00          mov    $0x604110,%edi
  401279:       e8 8c ff ff ff          callq  40120a <fun7>
```
第一个参数在`0x604110`，第二个数数是输入的数 
```
  40127e:       83 f8 02                cmp    $0x2,%eax
  401281:       74 05                   je     401288 <secret_phase+0x40>
  401283:       e8 51 03 00 00          callq  4015d9 <explode_bomb>
```
fun7的返回值和2比较，不等就爆炸

分析fun7，看到`401219`处，又出现了链式结构，尝试打印第一个传进来的参数`0x604110`附近有什么
```
(gdb) x /128xw 0x604110
0x604110 <n1>:	0x00000024	0x00000000	0x00604130	0x00000000
0x604120 <n1+16>:	0x00604150	0x00000000	0x00000000	0x00000000
0x604130 <n21>:	0x00000008	0x00000000	0x006041b0	0x00000000
0x604140 <n21+16>:	0x00604170	0x00000000	0x00000000	0x00000000
0x604150 <n22>:	0x00000032	0x00000000	0x00604190	0x00000000
0x604160 <n22+16>:	0x006041d0	0x00000000	0x00000000	0x00000000
0x604170 <n32>:	0x00000016	0x00000000	0x00604290	0x00000000
0x604180 <n32+16>:	0x00604250	0x00000000	0x00000000	0x00000000
0x604190 <n33>:	0x0000002d	0x00000000	0x006041f0	0x00000000
0x6041a0 <n33+16>:	0x006042b0	0x00000000	0x00000000	0x00000000
0x6041b0 <n31>:	0x00000006	0x00000000	0x00604210	0x00000000
0x6041c0 <n31+16>:	0x00604270	0x00000000	0x00000000	0x00000000
0x6041d0 <n34>:	0x0000006b	0x00000000	0x00604230	0x00000000
0x6041e0 <n34+16>:	0x006042d0	0x00000000	0x00000000	0x00000000
0x6041f0 <n45>:	0x00000028	0x00000000	0x00000000	0x00000000
0x604200 <n45+16>:	0x00000000	0x00000000	0x00000000	0x00000000
0x604210 <n41>:	0x00000001	0x00000000	0x00000000	0x00000000
0x604220 <n41+16>:	0x00000000	0x00000000	0x00000000	0x00000000
0x604230 <n47>:	0x00000063	0x00000000	0x00000000	0x00000000
0x604240 <n47+16>:	0x00000000	0x00000000	0x00000000	0x00000000
0x604250 <n44>:	0x00000023	0x00000000	0x00000000	0x00000000
0x604260 <n44+16>:	0x00000000	0x00000000	0x00000000	0x00000000
0x604270 <n42>:	0x00000007	0x00000000	0x00000000	0x00000000
0x604280 <n42+16>:	0x00000000	0x00000000	0x00000000	0x00000000
0x604290 <n43>:	0x00000014	0x00000000	0x00000000	0x00000000
0x6042a0 <n43+16>:	0x00000000	0x00000000	0x00000000	0x00000000
0x6042b0 <n46>:	0x0000002f	0x00000000	0x00000000	0x00000000
0x6042c0 <n46+16>:	0x00000000	0x00000000	0x00000000	0x00000000
0x6042d0 <n48>:	0x000003e9	0x00000000	0x00000000	0x00000000
0x6042e0 <n48+16>:	0x00000000	0x00000000	0x00000000	0x00000000
0x6042f0 <node1>:	0x000001a2	0x00000001	0x00604310	0x00000000
0x604300 <node2>:	0x0000038b	0x00000002	0x00604330	0x00000000
```
果然也是某种node（话说phase6链表的node在下面，靠这么近）
拎一个出来分析一下
```
0x604110 <n1>:	0x00000024	0x00000000	0x00604130	0x00000000
0x604120 <n1+16>:	0x00604150	0x00000000	0x00000000	0x00000000
```
显然，前面存的是一个value，后面存的两个地址，一个数据域两个指针域，猜想是一棵二叉树的节点
尝试画出这棵树（16进制）
```
              24
            /    \
          /        \
        /            \
      8               32
    /    \          /    \
   /      \        /      \
  6       16      2d      6b
 / \     /  \    /  \    /   \
1   7   14  23  28  2f  63  3e9
```
转成十进制是
```
              36
            /    \
          /        \
        /            \
      8               50
    /    \          /    \
   /      \        /      \
  6       22      45      107
 / \     /  \    /  \    /   \
1   7   20  35  40  47  99   1001
```
发现是BST
（其实node的名字就是它的层数和位置）
下面仔细看fun7的代码
```
000000000040120a <fun7>:
  40120a:       48 83 ec 08             sub    $0x8,%rsp
  40120e:       48 85 ff                test   %rdi,%rdi
  401211:       74 2b                   je     40123e <fun7+0x34>
  
  401213:       8b 17                   mov    (%rdi),%edx
  401215:       39 f2                   cmp    %esi,%edx
  401217:       7e 0d                   jle    401226 <fun7+0x1c>
  
  401219:       48 8b 7f 08             mov    0x8(%rdi),%rdi
  40121d:       e8 e8 ff ff ff          callq  40120a <fun7>
  
  401222:       01 c0                   add    %eax,%eax
  401224:       eb 1d                   jmp    401243 <fun7+0x39>
  
  401226:       b8 00 00 00 00          mov    $0x0,%eax
  40122b:       39 f2                   cmp    %esi,%edx
  40122d:       74 14                   je     401243 <fun7+0x39>
  
  40122f:       48 8b 7f 10             mov    0x10(%rdi),%rdi
  401233:       e8 d2 ff ff ff          callq  40120a <fun7>
  
  401238:       8d 44 00 01             lea    0x1(%rax,%rax,1),%eax
  40123c:       eb 05                   jmp    401243 <fun7+0x39>
  
  40123e:       b8 ff ff ff ff          mov    $0xffffffff,%eax
  
  401243:       48 83 c4 08             add    $0x8,%rsp
  401247:       c3                      retq
```
是一个递归函数，递归访问的方法是改变`%rdi`，即正在操作的node指针，也是fun7的第一个参数
`0x8(%rdi)`是左儿子，`0x10(%rdi)`是右儿子
翻译成c语言：
```c
int fun7(node* p, int v)  {  
	if (p == NULL)return -1;  
	else if (v < p->data)return 2 * fun7(p->left, v);  
	else if (v == p->data)return 0;  
	else return 2 * fun7(p->right, v) + 1;  
}
```
调用时传入的节点是根节点，考虑如何凑出2

想到一个三层的递归
`2*(2*(return 0)+1)`可以凑出2（因为只能考虑2*1=2）
即根节点先走left再走right到达一个value相等的节点

即key为22，通过！

**至此，满分无失误解除炸弹**

结算画面：
```
22
Wow! You've defused the secret stage!
GET /csapp/?userid=2023302111097&userpwd=6B6WSYiFBBJQ0sPmqJJw&lab=whu24&result=372514681%3AALFVHNNL4HIQ4%3Adefused%3A7%3A22&submit=submit HTTP/1.0


Congratulations! You've defused the bomb!
Your instructor has been notified and will verify your solution.
[Inferior 1 (process 782) exited normally]
(gdb) 
```
### 思考：
有了phase6处理指针变换的经验，secret_phase的难度略微下降（但只有一点）

## 实验总结
整个流程走下来，已经能较为熟练使用gdb调试了，阅读汇编代码的能力也有明显的提高

总耗时15h左右，应该算是比较慢的

在phase4还原代码的递归结构和寻找secret_phase入口时得到了SkyZheng的帮助，在此感谢
